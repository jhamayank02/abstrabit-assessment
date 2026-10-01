# Multi-Workspace Document Assistant (RAG & Tool Calling)

A multi-workspace document assistant with tenant isolation, grounded retrieval, and OpenAI Agents SDK tool calling.

## What It Does

- Sign in with a demo account or register a new account.
- Create and switch between multiple workspaces.
- Upload text documents into the active workspace.
- Store every workspace's chunks in one shared `chunks` table with `workspace_id` on every row.
- Retrieve only chunks from the active workspace before answering.
- Cite source document and chunk number when an answer is supported.
- Say `I don't know based on this workspace's documents.` when no active-workspace document supports the answer.
- Use the OpenAI Agents SDK so the model can decide when to call two validated tools:
  - `save_task`: saves a task into the active workspace.
  - `send_notification`: records a channel notification result in the active workspace audit log.
- Show documents, chat history, saved tasks, and tool-call audit log behind login.

The default data store uses SQLite through Node's built-in `node:sqlite`. Chat and tool decisions require an OpenAI API key in `OPENAI_API_KEY`.

## Run Locally

Requires Node.js 24+.

```bash
npm install
$env:OPENAI_API_KEY="sk-your-openai-api-key"
npm start
```

Open `http://localhost:3000`.

## Environment

See `.env.example`.

```env
PORT=3000
OPENAI_API_KEY=sk-your-openai-api-key
OPENAI_MODEL=gpt-4.1-mini
DATABASE_PATH=assistant.db
# DATABASE_URL=postgresql://postgres:postgres@localhost:5432/assistant
```

`DATABASE_URL` is only for deployments that already provide free PostgreSQL. Without it, the app runs with local SQLite.

## Demo Login

- Email: `demo@example.com`
- Password: `demo123`

## Test The Required Flows

1. Select `Company Handbook` and ask: `What is the product roadmap?`
   Expected: the assistant says it does not know from this workspace's documents.

2. Select `Product Notes` and ask: `What is the product roadmap?`
   Expected: answer cites `roadmap.txt (chunk 1)`.

3. Send: `Create task: review launch plan`
   Expected: a task appears in the active workspace and `save_task` appears in the audit log.

4. Send: `Send notification #dev-alerts: migration complete`
   Expected: `send_notification` appears in the audit log with channel `#dev-alerts`.

5. Upload the same text file twice into one workspace.
   Expected: first upload ingests, second upload reports the document already exists.

## Notes

The assignment prompt is preserved in `requirements.txt` and `REQUIREMENTS.md`. AI collaboration notes are in `AI_NOTES.md`.
