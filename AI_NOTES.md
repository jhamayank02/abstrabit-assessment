# AI Notes

## Collaboration & Tooling
I used Claude and Codex/Antigravity throughout this project for architecture brainstorming, code generation, refactoring, and debugging. Work was divided iteratively:
- **Human Role:** Designed system requirements, defined tenancy isolation rules, established database schema strategies (dual-engine for zero-config local evaluation vs. hosted PostgreSQL/Supabase), chose the tool validation approach, and verified edge cases (e.g. cross-workspace leakage and idempotent uploads).
- **AI Role:** Generated boilerplate code for the Express server, implemented the mathematical vector cosine similarity routine, crafted the frontend UI, and assisted in debugging environment-specific package issues.

---

## Key Architectural Decisions

1. **Shared Store with Strict Query-Level Tenancy Scoping:**
   Instead of creating separate database tables or collections per workspace (which would violate the shared-store requirement), all document chunks across all tenants live in a single `chunks` table. Tenancy isolation is strictly enforced directly in the database query (`WHERE workspace_id = ?` in SQLite, `WHERE workspace_id = $1` in PostgreSQL). Retrieving, scoring, and citing are strictly constrained within this boundary, preventing cross-tenant information leakage.

2. **Dual Database Engine (Built-in SQLite + Hosted PostgreSQL with pgvector):**
   A common failure mode in code assessments is that reviewers cannot easily run the project locally without setting up a dedicated PostgreSQL server and installing vector extensions. To guarantee seamless local evaluation, the server uses Node.js's native `node:sqlite` module and `assistant.db` by default (requiring zero external setup). However, if `DATABASE_URL` is set, it automatically switches to PostgreSQL with `pgvector` for hosted deployments (such as Supabase or Neon).

3. **Schema-Validated Tool Calling Loop with Real Workspace Side Effects:**
   We implemented two distinct tools:
   - `save_task`: Validates `{ title: string }` and inserts an actionable task into the workspace's `tasks` table.
   - `send_notification`: Validates `{ channel: string, message: string }` and dispatches alerts while logging to the audit log.
   Every tool invocation undergoes strict argument schema validation before execution. If arguments are malformed or missing, a structured error is caught and recorded in `tool_calls` rather than crashing the server.

4. **Idempotent Ingestion Pipeline:**
   Uploaded text documents are hashed using SHA-256 and checked against `(workspace_id, hash)`. If a user uploads the same document twice, the system recognizes the duplicate, rejects redundant re-chunking, and returns an idempotent confirmation.

---

## Hardest Bug / AI Wrong Turn

The most notable wrong turn occurred early in the process regarding the assessment requirements and native module dependencies:

1. **Misinterpretation of `requirements.txt`:**
   The assessment problem brief was originally named `requirements.txt`. The initial AI assistant mistook this file for a Python pip dependency file and mistakenly overwrote it with Python packages (`fastapi`, `uvicorn`, `python-multipart`). Later, when transitioning to Node.js, the AI attempted to install `better-sqlite3`, which failed to compile on Windows due to missing Visual Studio C++ build tools and path spaces (`RAG & Tool Calling`).

2. **How It Was Discovered & Fixed:**
   - We inspected the repository history and discovered that `requirements.txt` had originally contained the entire interview assessment brief, leaving the codebase without its specifications.
   - We restored the full problem specification to `requirements.txt` and created `REQUIREMENTS.md`.
   - To completely eliminate the native C++ build error on Windows, we replaced `better-sqlite3` with Node.js 24's native `node:sqlite` module, which runs with zero external native dependencies. We then paired this with pure-JS `pg` for optional PostgreSQL support.

---

## Future Improvements

With more development time, I would add:
1. **Streaming Responses:** Implement Server-Sent Events (SSE) to stream assistant tokens in real time.
2. **Hybrid Search & Re-ranking:** Combine BM25 full-text keyword search with dense vector similarity, maintaining the workspace filter throughout the rank merge.
3. **Multi-Format Extraction:** Add streaming PDF and DOCX text extractors with OCR fallback.
4. **Tenant Access Control (RBAC):** Expand user authentication into role-based permissions (viewer, editor, admin) for multi-user collaboration inside shared workspaces.
