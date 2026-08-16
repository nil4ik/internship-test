## Part 2 - Code Review

1. API key is hardcoded
- What's wrong: `API_KEY = "sk-prod-abc123xyz"` is hardcoded.
- Why it matters: anyone with repo access can see it, and it stays in git history even if you remove it later. Same key is used everywhere too, so one leak affects everything.
- Fix: pull it from an env variable instead (`os.environ.get("API_KEY")`), keep it out of git via `.gitignore`.

2. SQL injection (search_documents + save_answer)
- What's wrong: both functions build queries by just gluing strings together (`"...LIKE '%" + question + "%'"`).
- Why it matters: someone types `' OR '1'='1` and now they control the query, they could read or wreck data they shouldn't touch.
- Fix: use parameterized queries, e.g. `cursor.execute("... LIKE ?", (f"%{question}%",))`.

3. No error handling in ask_llm
- What's wrong: `response.json()["response"]` just assumes the API call worked.
- Why it matters: if the API is down, slow, or returns something unexpected, the script just crashes.
- Fix: try/except around the request, set a timeout, check status code before touching the response.

4. ask_llm gets called twice for no reason
- What's wrong: it's called once to print the answer, once to save it, same question, same docs.
- Why it matters: doubles the API cost/latency for nothing, and if the model isn't deterministic the printed answer and the saved one might not even match.
- Fix: call it once, save the result in a variable, reuse it.


## Part 3 — Short Written Questions

**Q1.**
With `LIKE '%...%'`, Postgres can't use an index because the `%` on both sides means it doesn't know where a match starts, so it has to scan the entire table row by row.
At 1M rows this gets slow and CPU-heavy, and under concurrent traffic it can start timing out.

I'd switch to Postgres full-text search (`tsvector`/`tsquery` with a GIN index) or `pg_trgm` for substring search, so lookups use an index instead of a full scan.

**Q2.**
Dumping every found document into the prompt doesn't scale: if there are many documents, the combined text can exceed the model's context window.
Even when it fits, it's expensive and slow since you're paying for tokens that are mostly irrelevant to the actual question.

A basic RAG setup fixes this by chunking documents into small pieces, embedding each chunk, and at query time retrieving only the top-k most relevant chunks, so the prompt stays small and focused.

*(I hadn't worked with RAG before this test, I looked into how it works and this is my understanding of it.)*

**Q3.**

1. The API is slow or down - handled with a timeout and retry with backoff.
2. The API returns an error - handled by checking `status_code` and reacting differently per case instead of assuming success.
3. The response comes back in an unexpected shape (missing key, bad JSON) - handled with try/except around `response.json()` and a fallback message instead of letting the script crash.

**Q4 (bonus).**
I'd use two tables:

1. `users` (id, external_id, channel, created_at) - who's messaging.
2. `messages` (id, user_id, role, content, created_at) - the actual conversation, where `role` marks whether it was the user or the assistant.

Keeping user info in its own table avoids duplicating it on every message, and `messages` can be filtered by `user_id` and sorted by time to rebuild the conversation history.



**Time spent:** 1 hour (approx.)