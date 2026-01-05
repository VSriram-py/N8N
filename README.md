# N8N

> ⚡ Detailed n8n agent templates and setup instructions for Live Agent Studio

## Overview ✅
This repository contains several n8n workflow templates that demonstrate patterns for building AI agents and RAG (Retrieval-Augmented Generation) pipelines used with Live Agent Studio. Each JSON file under the repo is an export of an n8n workflow: import them into n8n to try them.

Included templates:
- **`Base_Agent.json`** — Minimal example: webhook → input parsing → store messages (Supabase) → respond
- **`Agent_Node_Agent.json`** — Example using the dedicated `Agent` node and Anthropic LLM
- **`Agentic_RAG_AI_Agent.json`** — Full RAG template: Google Drive ingestion, chunking, embeddings (OpenAI), Supabase vector store / pgvector, and lookup SQL
- **`MCP_Agent.json`** — Multi-Capability (MCP) agent demo using tool calls (Brave search, Convex)

---

## Key Components & Behavior 🔧
- **Webhook endpoints**: All workflows expose a POST webhook that expects authenticated requests. Webhook path and id are set in each workflow (import to view or modify paths).
- **Authentication**: Header authentication is used by default; the sample header payload in `pinData` expects `Authorization: Bearer YOUR BEARER TOKEN`. Response nodes set a `X-n8n-Signature` header for response validation.
- **Input fields**: Standard fields used across templates:
  - `query` (string): user text
  - `user_id` (string): user identifier
  - `request_id` (string): request tracking id
  - `session_id` (string): session identifier (used by chat memory nodes)
- **Datastores**:
  - **Supabase**: used to store messages and vector metadata (vector store uses JSONB for metadata).
  - **Postgres (pgvector)**: used for chat memory and optional local vector store. See SQL snippet in the RAG template for table creation and `match_documents` SQL helper.
- **Models & Embeddings**:
  - OpenAI chat and embedding models are used in the RAG template (e.g. `text-embedding-3-small`).
  - Anthropic models appear in `Agent_Node_Agent.json` (e.g. `claude-3-5-haiku-20241022`).
  - Embedding dimension noted in RAG SQL: `vector(1536)` (OpenAI default in example).
- **Agent tools**: MCP template demonstrates how to expose external tools to the agent (list-tools + execute-tool pattern).

---

## Workflow-specific notes ✳️
### Base Agent (`Base_Agent.json`)
- Minimal, production-ready skeleton:
  - Webhook path: `9a6c4630-b422-4d42-b894-81ecfe881ffe`
  - Uses Supabase to add user and AI messages to a `messages` table.
  - Sticky notes inside the workflow explain where to add custom agent logic.

### Agent Node Sample (`Agent_Node_Agent.json`)
- Uses the `Agent` node in n8n (manages conversation state internally / compatible with Live Agent Studio).
- Key nodes:
  - Webhook: path `d9fec84b-86f0-4230-9fd4-c1cb392ff8b5` with header authentication
  - `Anthropic Chat Model` node (credentials: **Anthropic account**) and a `Postgres Chat Memory` node for session persistence.
- The sample includes a response header `X-n8n-Signature` (value set in workflow) for verifying responses.

### Agentic RAG AI Agent (`Agentic_RAG_AI_Agent.json`) — RAG Template
- Full ingestion pipeline:
  - Google Drive trigger to detect new/updated files
  - Download & convert (docs → text)
  - Extract / chunk text
  - Create embeddings with OpenAI and insert vectors into Supabase
  - Query strategies: RAG lookups, SQL-based numeric analysis, or full-document retrieval
- Important snippets shipped in the workflow:
  - SQL to enable `pgvector`, create `documents` table, and `match_documents` function (used for fast similarity search).
  - Table creation and `pgvector` enablement are recommended to run before ingesting documents.
- Webhook (chat trigger / REST) paths exist for chat integrations — set your n8n URL and header auth credentials before going live.

### MCP Agent (`MCP_Agent.json`)
- Demonstrates the **tool-based** agent pattern:
  - `List <toolset> Tools` nodes return tool schemas to the agent
  - `Execute <toolset> Tool` nodes accept an exact tool name and JSON params from the agent
- Works with external providers (Brave Search, Convex) via configured `mcpClientApi` credentials.
- Model used: OpenAI (example `gpt-4o-mini`) configured in the workflow.

---

## Getting Started (Quick) 🚀
1. Import the JSON workflow you want to use into n8n (Settings → Import from file).
2. Add credentials and set them to match the credential selectors in the workflow:
   - `OpenAi account` / `Anthropic account` / `Prod Supabase account` / `Prod Postgres account` / `Google Drive account` / `mcpClientApi` entries
3. Configure Header Auth credential for your webhook (or switch to a different auth method as needed).
4. (RAG) Run the SQL/table-creation nodes or execute the provided SQL to set up `pgvector` and `documents` table. Example SQL (from the template):

```sql
-- Enable the pgvector extension to work with embedding vectors
create extension vector;

-- Create a table to store your documents
create table documents (
  id bigserial primary key,
  content text,
  metadata jsonb,
  embedding vector(1536)
);

-- Create a function to search for documents
create function match_documents (
  query_embedding vector(1536),
  match_count int default null,
  filter jsonb DEFAULT '{}'
) returns table (
  id bigint,
  content text,
  metadata jsonb,
  similarity float
)
language plpgsql
as $$
begin
  return query
  select
    id,
    content,
    metadata,
    1 - (documents.embedding <=> query_embedding) as similarity
  from documents
  where metadata @> filter
  order by documents.embedding <=> query_embedding
  limit match_count;
end;
$$;
```

5. Set the Google Drive folder IDs or other file source parameters before enabling ingestion.
6. Send a test POST request to your webhook URL using the sample JSON body from the workflow `pinData` to verify end-to-end behavior.

---

## Example Webhook Payload (from workflow pinData) 🧾
```json
{
  "query": "Supabase",
  "user_id": "google-oauth2|116467443974012389959",
  "request_id": "f98asdyf987yasd0f987asdf8",
  "session_id": "google-oauth2|116467443974012389959~2~8dfbddbe603d"
}
```

> Tip: use `Authorization: Bearer <YOUR_TOKEN>` and ensure the header auth credential in n8n matches the token.

---

## Customization & Tips 💡
- Swap models easily via the model selection within the workflow nodes; test with smaller models during development.
- Adjust `contextWindowLength` in `Postgres Chat Memory` nodes to change how much conversation history you persist to memory.
- For large corpora, tune chunk sizes and embedding batch sizes for efficiency.
- Use the sticky notes embedded in the exported workflows as local documentation — they already contain helpful setup and usage notes.

---

## License & Author
Author: Cole Medin (workflows include author sticky notes)

If you want a deeper, step-by-step setup or a tailored README for a particular deployment environment, open an issue or request a README customization.
