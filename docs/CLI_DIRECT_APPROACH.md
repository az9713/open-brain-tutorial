# CLI-Direct Approach to Open Brain

How to use Open Brain from CLI-based AI tools without MCP — using shell commands,
a lightweight `ob` CLI tool, and explicit activation controls.

**Audience:** Developers using Claude Code, OpenAI Codex, Gemini CLI, or similar
terminal-based AI assistants. Assumes comfort with shell commands and environment
variables.

---

## Table of Contents

1. [The Problem This Solves](#1-the-problem-this-solves)
2. [Architecture Comparison](#2-architecture-comparison)
3. [Why the Supabase CLI Is NOT the Answer](#3-why-the-supabase-cli-is-not-the-answer)
4. [Three Direct Access Methods](#4-three-direct-access-methods)
5. [The Embedding Pipeline Gap](#5-the-embedding-pipeline-gap)
6. [The Activation Problem](#6-the-activation-problem)
7. [The `ob` CLI Tool Design](#7-the-ob-cli-tool-design)
8. [Configuration for Each CLI AI Tool](#8-configuration-for-each-cli-ai-tool)
9. [Comparison Matrix: MCP vs CLI-Direct](#9-comparison-matrix-mcp-vs-cli-direct)
10. [Migration and Coexistence](#10-migration-and-coexistence)
11. [Troubleshooting](#11-troubleshooting)
- [Appendix A: Complete `ob` Script](#appendix-a-complete-ob-script)
- [Appendix B: PostgreSQL Auto-Embedding Trigger SQL](#appendix-b-postgresql-auto-embedding-trigger-sql)

---

## 1. The Problem This Solves

MCP (Model Context Protocol) exists because GUI-based AI clients — Claude Desktop,
ChatGPT web, mobile apps — cannot execute code. They can only call pre-registered
tools through a standardized protocol. MCP gives these clients a way to interact with
external systems like databases and APIs.

CLI-based AI tools are different. Claude Code, OpenAI Codex, and Gemini CLI run in
a terminal. They can execute shell commands, run scripts, and call APIs directly using
`curl`, `psql`, `node`, or `python`. For these tools, MCP is an unnecessary
intermediary — an extra layer of protocol, deployment, and infrastructure between the
AI and the database.

### Side-by-Side: What Each Client Type Needs

| Capability | GUI Client (Claude Desktop, ChatGPT) | CLI Client (Claude Code, Codex, Gemini CLI) |
|---|---|---|
| Can execute shell commands | No | **Yes** |
| Can call APIs via `curl` | No | **Yes** |
| Can run inline scripts | No | **Yes** |
| Can connect to databases | No | **Yes** (via `psql`, client libraries) |
| Needs pre-registered tools | **Yes** — MCP or native tool use | No — can construct calls dynamically |
| Needs a protocol layer | **Yes** — MCP or function calling | No — shell is the protocol |
| Needs a deployed server | **Yes** — Edge Function or local server | No — can call APIs directly |

### The Implication

For a developer who primarily uses Claude Code or another CLI AI tool, the standard
Open Brain setup — deploying an Edge Function, configuring MCP transport, managing
JSON-RPC framing — solves a problem they don't have. The AI can just run `curl` to
the Supabase REST API, or execute a script that generates embeddings and inserts rows.

This document describes how to set that up cleanly, with proper activation controls
so Open Brain only engages when you want it to.

---

## 2. Architecture Comparison

### The Current MCP Path

This is how Open Brain works today. Every AI client — GUI or CLI — talks to the
database through the MCP Edge Function.

```
+------------------+          JSON-RPC/HTTPS          +--------------------+
|                  |                                  |                    |
|   AI Client      |  POST (tools/call)               |   open-brain-mcp   |
|   (any)          | -------------------------------->|   Edge Function    |
|                  |                                  |   (Supabase/Deno)  |
|                  |  JSON-RPC response               |                    |
|                  | <--------------------------------|                    |
+------------------+                                  +---+------------+---+
                                                          |            |
                                                          | fetch()    | supabase-js
                                                          v            v
                                                   +----------+  +-----------+
                                                   | OpenRouter|  | PostgreSQL|
                                                   | (embed +  |  | thoughts  |
                                                   |  metadata)|  | table     |
                                                   +----------+  +-----------+
```

**Five components in the request path:** AI client → MCP protocol → Edge Function →
OpenRouter → PostgreSQL.

### The CLI-Direct Path

CLI AI tools skip MCP entirely. They execute shell commands that interact with the
database and embedding API directly.

```
+------------------+
|                  |
|   CLI AI Tool    |   executes shell commands
|   (Claude Code,  |
|    Codex, etc.)  |
|                  |
+---+---------+----+
    |         |
    | ob capture "thought text"
    |         |
    v         v
+---+---------+----+
|                  |
|   ob CLI         |   bash script (curl + jq)
|   (local)        |
|                  |
+---+---------+----+
    |         |
    | curl    | curl
    v         v
+----------+  +-----------+
| OpenRouter|  | Supabase  |
| (embed)   |  | REST API  |
+----------+  +-----------+
                    |
                    v
              +-----------+
              | PostgreSQL|
              | thoughts  |
              | table     |
              +-----------+
```

**Three components in the request path:** AI tool → `ob` CLI → OpenRouter + PostgreSQL.

No Edge Function. No MCP protocol. No JSON-RPC. No Supabase secrets management. No
deployment step.

### Both Paths Coexisting

The CLI-Direct approach does not replace MCP — it runs alongside it. Both paths read
and write the same `thoughts` table.

```
+----------------+     +----------------+     +----------------+
| Claude Desktop |     | ChatGPT (web)  |     | Claude Code    |
| (GUI)          |     | (GUI)          |     | (CLI)          |
+-------+--------+     +-------+--------+     +-------+--------+
        |                       |                       |
        |  MCP (JSON-RPC)       |  MCP (JSON-RPC)       |  shell commands
        |                       |                       |
        v                       v                       v
+-------+--------+     +-------+--------+     +--------+-------+
|  open-brain-mcp|     |  open-brain-mcp|     |  ob CLI        |
|  Edge Function |     |  Edge Function |     |  (bash script) |
+-------+--------+     +-------+--------+     +---+--------+---+
        |                       |                  |        |
        +----------+------------+                  |        |
                   |                               |        |
                   v                               v        v
        +----------+----------+           +--------+--+ +---+-------+
        |  PostgreSQL         |           | OpenRouter | | Supabase  |
        |  thoughts table     |           | API        | | REST API  |
        |  (shared data)      |           +------------+ +-----+-----+
        +---------------------+                                |
                ^                                              |
                |                                              |
                +----------------------------------------------+
                         same thoughts table
```

A thought captured via Claude Desktop over MCP and a thought captured via Claude Code
using `ob capture` end up in the same table, with the same embedding, searchable by
either path.

---

## 3. Why the Supabase CLI Is NOT the Answer

Supabase provides a CLI tool (`supabase`). It might seem like the obvious choice for
CLI-direct database access. It is not — it solves a different problem entirely.

### What the Supabase CLI Does vs What Open Brain Needs

| Task | `supabase` CLI | What Open Brain needs |
|---|---|---|
| Create and manage projects | Yes | Not at runtime |
| Deploy Edge Functions | Yes | Not at runtime |
| Run database migrations | Yes | Not at runtime |
| Set secrets/env vars | Yes | Not at runtime |
| Link to a remote project | Yes | Not at runtime |
| **Insert a row into `thoughts`** | **No** | **Yes — this is the core operation** |
| **Query `thoughts` by semantic similarity** | **No** | **Yes — this is the core query** |
| **Generate embeddings via OpenRouter** | **No** | **Yes — required for search** |
| **Call `match_thoughts()` RPC function** | **No** | **Yes — this is semantic search** |
| Read table data | No (use dashboard or REST API) | Yes |
| Run arbitrary SQL | Only via `supabase db` locally | Need remote access |

The Supabase CLI is an **admin and deployment** tool. It manages infrastructure:
creating projects, deploying functions, running migrations, setting secrets. It does
not provide runtime data access — you cannot use it to insert a thought, run a search,
or call an RPC function against your remote database.

For runtime data access, you use either:
- The **Supabase REST API** (PostgREST) — auto-generated HTTP endpoints for every table
- **`psql`** — direct PostgreSQL connection
- **Client libraries** (`supabase-js`, `supabase-py`) — wrappers around the REST API

The `ob` CLI tool described in this document wraps the Supabase REST API with `curl`,
adding the embedding pipeline that the REST API alone cannot provide.

---

## 4. Three Direct Access Methods

Any CLI AI tool can interact with Supabase using these methods. Each has different
trade-offs.

### 4.1 `curl` to Supabase REST API

Supabase auto-generates a REST API (PostgREST) for every table. The `thoughts` table
has endpoints available immediately after creation.

**List recent thoughts:**

```bash
curl -s \
  "https://YOUR_PROJECT_REF.supabase.co/rest/v1/thoughts?order=created_at.desc&limit=5&select=id,content,metadata,created_at" \
  -H "apikey: YOUR_ANON_KEY" \
  -H "Authorization: Bearer YOUR_ANON_KEY" | jq .
```

**Insert a thought (without embedding — plain text only):**

```bash
curl -s -X POST \
  "https://YOUR_PROJECT_REF.supabase.co/rest/v1/thoughts" \
  -H "apikey: YOUR_SERVICE_ROLE_KEY" \
  -H "Authorization: Bearer YOUR_SERVICE_ROLE_KEY" \
  -H "Content-Type: application/json" \
  -H "Prefer: return=representation" \
  -d '{"content": "Sarah wants to start a consulting business", "metadata": {}}' | jq .
```

**Semantic search via RPC (requires a pre-computed embedding vector):**

```bash
curl -s -X POST \
  "https://YOUR_PROJECT_REF.supabase.co/rest/v1/rpc/match_thoughts" \
  -H "apikey: YOUR_SERVICE_ROLE_KEY" \
  -H "Authorization: Bearer YOUR_SERVICE_ROLE_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query_embedding": [0.023, -0.041, ...1536 floats...],
    "match_threshold": 0.7,
    "match_count": 10,
    "filter": {}
  }' | jq .
```

**Limitations:**
- Inserting without an embedding means the thought is not searchable semantically
- The REST API cannot generate embeddings — that requires calling OpenRouter separately
- You need to chain two API calls (OpenRouter for embedding, then Supabase for insert)

### 4.2 `psql` Direct PostgreSQL Connection

If you have direct PostgreSQL access (Supabase provides a connection string in the
dashboard under Settings → Database), you can use `psql` for full SQL access.

**Connection:**

```bash
psql "postgresql://postgres.[YOUR_PROJECT_REF]:[YOUR_DB_PASSWORD]@aws-0-[REGION].pooler.supabase.com:6543/postgres"
```

**List recent thoughts:**

```sql
SELECT id, content, metadata, created_at
FROM thoughts
ORDER BY created_at DESC
LIMIT 5;
```

**Insert a thought (without embedding):**

```sql
INSERT INTO thoughts (content, metadata)
VALUES (
  'Sarah wants to start a consulting business',
  '{"type": "person_note", "topics": ["consulting", "career"], "people": ["Sarah"]}'::jsonb
);
```

**Semantic search (requires a pre-computed embedding):**

```sql
SELECT id, content, metadata,
       1 - (embedding <=> '[0.023, -0.041, ...]'::vector) AS similarity,
       created_at
FROM thoughts
WHERE 1 - (embedding <=> '[0.023, -0.041, ...]'::vector) > 0.7
ORDER BY embedding <=> '[0.023, -0.041, ...]'::vector
LIMIT 10;
```

**Advantages:**
- Full SQL power — joins, aggregates, window functions, anything PostgreSQL supports
- No REST API overhead
- Can run migrations and schema changes directly

**Limitations:**
- Requires the database password (more sensitive than the anon key)
- Connection string must be stored securely
- Still cannot generate embeddings — that requires an external API call
- `psql` may not be installed on all systems

### 4.3 Inline Script Execution

CLI AI tools can write and execute scripts directly. This is the most flexible
approach — the AI generates a complete script that handles the full pipeline.

**Node.js (using `fetch`, no dependencies beyond Node 18+):**

```javascript
// capture.mjs — run with: node capture.mjs "your thought text"
const SUPABASE_URL = process.env.OB_SUPABASE_URL;
const SUPABASE_KEY = process.env.OB_SUPABASE_KEY;
const OPENROUTER_KEY = process.env.OB_OPENROUTER_KEY;
const content = process.argv[2];

// Generate embedding
const embedRes = await fetch("https://openrouter.ai/api/v1/embeddings", {
  method: "POST",
  headers: {
    "Authorization": `Bearer ${OPENROUTER_KEY}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({ model: "openai/text-embedding-3-small", input: content }),
});
const embedData = await embedRes.json();
const embedding = embedData.data[0].embedding;

// Insert into thoughts table
const insertRes = await fetch(`${SUPABASE_URL}/rest/v1/thoughts`, {
  method: "POST",
  headers: {
    "apikey": SUPABASE_KEY,
    "Authorization": `Bearer ${SUPABASE_KEY}`,
    "Content-Type": "application/json",
    "Prefer": "return=representation",
  },
  body: JSON.stringify({ content, embedding, metadata: {} }),
});
const row = await insertRes.json();
console.log(JSON.stringify(row, null, 2));
```

**Python (using `requests`):**

```python
# capture.py — run with: python capture.py "your thought text"
import os, sys, json, requests

SUPABASE_URL = os.environ["OB_SUPABASE_URL"]
SUPABASE_KEY = os.environ["OB_SUPABASE_KEY"]
OPENROUTER_KEY = os.environ["OB_OPENROUTER_KEY"]
content = sys.argv[1]

# Generate embedding
embed_res = requests.post(
    "https://openrouter.ai/api/v1/embeddings",
    headers={"Authorization": f"Bearer {OPENROUTER_KEY}", "Content-Type": "application/json"},
    json={"model": "openai/text-embedding-3-small", "input": content},
)
embedding = embed_res.json()["data"][0]["embedding"]

# Insert into thoughts table
insert_res = requests.post(
    f"{SUPABASE_URL}/rest/v1/thoughts",
    headers={
        "apikey": SUPABASE_KEY,
        "Authorization": f"Bearer {SUPABASE_KEY}",
        "Content-Type": "application/json",
        "Prefer": "return=representation",
    },
    json={"content": content, "embedding": embedding, "metadata": {}},
)
print(json.dumps(insert_res.json(), indent=2))
```

**Advantages:**
- Full pipeline in one script — embedding + insert in sequence
- The AI can generate these on-the-fly, tailored to the task
- No external tool installation beyond the language runtime

**Limitations:**
- Requires Node.js 18+ or Python with `requests`
- Each execution spawns a process — slightly slower than a compiled CLI tool
- The AI must know the script pattern (solvable with CLAUDE.md instructions)

---

## 5. The Embedding Pipeline Gap

This is the core technical challenge of the CLI-Direct approach. The MCP Edge Function
does two things that the Supabase REST API alone cannot:

1. **Generates an embedding** by calling OpenRouter's `text-embedding-3-small` model
2. **Extracts metadata** by calling OpenRouter's `gpt-4o-mini` model

Without the Edge Function, who generates the embedding? Without an embedding, semantic
search does not work — the `match_thoughts()` function compares query vectors against
stored vectors, and a row with no embedding vector is invisible to it.

There are four ways to solve this.

### 5.1 Two Sequential `curl` Calls

The simplest approach: make two API calls in sequence.

**Step 1 — Get the embedding from OpenRouter:**

```bash
EMBEDDING=$(curl -s https://openrouter.ai/api/v1/embeddings \
  -H "Authorization: Bearer $OB_OPENROUTER_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"model\": \"openai/text-embedding-3-small\", \"input\": \"$THOUGHT_TEXT\"}" \
  | jq -c '.data[0].embedding')
```

**Step 2 — Insert into Supabase with the embedding:**

```bash
curl -s -X POST "$OB_SUPABASE_URL/rest/v1/thoughts" \
  -H "apikey: $OB_SUPABASE_KEY" \
  -H "Authorization: Bearer $OB_SUPABASE_KEY" \
  -H "Content-Type: application/json" \
  -H "Prefer: return=representation" \
  -d "{\"content\": \"$THOUGHT_TEXT\", \"embedding\": $EMBEDDING, \"metadata\": {}}" \
  | jq .
```

**Pros:** No dependencies beyond `curl` and `jq`. Completely transparent.
**Cons:** Two network round trips. No metadata extraction. The AI must know to chain
these calls.

### 5.2 Shell Script Wrapping Both Calls

Wrap the two-call pattern into a reusable shell function or script. This is what the
`ob` CLI tool (Section 7) does — it is a bash script that orchestrates `curl` calls.

```bash
#!/bin/bash
# ob-capture.sh — usage: ob-capture.sh "your thought text"
set -euo pipefail

CONTENT="$1"

# Get embedding
EMBEDDING=$(curl -sf https://openrouter.ai/api/v1/embeddings \
  -H "Authorization: Bearer $OB_OPENROUTER_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg c "$CONTENT" '{model: "openai/text-embedding-3-small", input: $c}')" \
  | jq -c '.data[0].embedding')

# Extract metadata (optional — remove this block to skip)
METADATA=$(curl -sf https://openrouter.ai/api/v1/chat/completions \
  -H "Authorization: Bearer $OB_OPENROUTER_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg c "$CONTENT" '{
    model: "openai/gpt-4o-mini",
    response_format: {type: "json_object"},
    messages: [
      {role: "system", content: "Extract metadata from the user'\''s captured thought. Return JSON with: \"people\" (array), \"action_items\" (array), \"topics\" (array of 1-3 tags), \"type\" (one of observation, task, idea, reference, person_note). Only extract what'\''s explicitly there."},
      {role: "user", content: $c}
    ]
  }')" \
  | jq -c '.choices[0].message.content | fromjson')

# Insert into Supabase
curl -sf -X POST "$OB_SUPABASE_URL/rest/v1/thoughts" \
  -H "apikey: $OB_SUPABASE_KEY" \
  -H "Authorization: Bearer $OB_SUPABASE_KEY" \
  -H "Content-Type: application/json" \
  -H "Prefer: return=representation" \
  -d "$(jq -n --arg c "$CONTENT" --argjson e "$EMBEDDING" --argjson m "$METADATA" \
    '{content: $c, embedding: $e, metadata: $m}')" \
  | jq .
```

**Pros:** Full pipeline including metadata extraction. Reusable. No runtime dependencies
beyond `curl` and `jq`.
**Cons:** Shell quoting is fragile for content with special characters. `jq -n` with
`--arg` handles this safely, but the script is more complex than it first appears.

### 5.3 `curl` Directly to the Edge Function

If you already have the MCP Edge Function deployed, you can call it directly with a
JSON-RPC payload — bypassing MCP client libraries entirely.

```bash
curl -s -X POST \
  "https://YOUR_PROJECT_REF.supabase.co/functions/v1/open-brain-mcp?key=YOUR_ACCESS_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/call",
    "params": {
      "name": "capture_thought",
      "arguments": {
        "content": "Sarah wants to start a consulting business"
      }
    }
  }' | jq .
```

**For search:**

```bash
curl -s -X POST \
  "https://YOUR_PROJECT_REF.supabase.co/functions/v1/open-brain-mcp?key=YOUR_ACCESS_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/call",
    "params": {
      "name": "search_thoughts",
      "arguments": {
        "query": "career changes",
        "threshold": 0.7
      }
    }
  }' | jq '.result.content[0].text | fromjson'
```

**Pros:** Uses the existing Edge Function — no new infrastructure. Gets embedding
and metadata extraction for free. One network call handles everything.
**Cons:** Requires the Edge Function to already be deployed. Still goes through the
MCP protocol layer (JSON-RPC framing). Depends on the Edge Function being available.

This is a pragmatic hybrid: you skip the MCP client, but you keep the MCP server as
a convenient embedding pipeline. It is the fastest way to get CLI-Direct working if
you have already completed the standard Open Brain setup.

### 5.4 PostgreSQL Trigger for Auto-Embedding (Cleanest)

The cleanest long-term solution: a PostgreSQL trigger that automatically generates
embeddings when a row is inserted. The client inserts a thought with just `content`,
and the database handles embedding generation asynchronously using `pg_net` (a Supabase
extension for making HTTP calls from within PostgreSQL).

With this trigger in place, any insert — from `curl`, `psql`, the MCP server, or any
other client — automatically gets an embedding. The embedding pipeline moves from the
application layer into the database layer.

```
Client (any)                  PostgreSQL                    OpenRouter
    |                              |                             |
    |  INSERT INTO thoughts        |                             |
    |  (content, metadata)         |                             |
    |----------------------------->|                             |
    |                              |                             |
    |  row inserted (no embedding) |                             |
    |                              |                             |
    |                              |  AFTER INSERT trigger fires  |
    |                              |                             |
    |                              |  pg_net.http_post()          |
    |                              |---------------------------->|
    |                              |                             |
    |                              |  embedding vector returned   |
    |                              |<----------------------------|
    |                              |                             |
    |                              |  UPDATE thoughts             |
    |                              |  SET embedding = vector      |
    |                              |  WHERE id = NEW.id           |
    |                              |                             |
    |  (embedding available        |                             |
    |   within ~500ms)             |                             |
```

**Pros:**
- Embeds every thought automatically, regardless of how it was inserted
- The client only needs to do a simple INSERT — no embedding logic required
- Works with `curl`, `psql`, `supabase-js`, MCP, Slack webhooks, anything
- Single source of truth for embedding logic

**Cons:**
- Requires `pg_net` extension (available on Supabase, not all PostgreSQL hosts)
- Embedding is asynchronous — there is a brief window after insert where the thought
  exists but is not yet searchable
- Requires a Supabase Edge Function or external endpoint to receive the `pg_net` call
  and return the embedding (pg_net makes HTTP calls, but OpenRouter returns the
  embedding in a response body that must be extracted and written back)
- More complex to debug than a client-side pipeline

The full trigger SQL is in [Appendix B](#appendix-b-postgresql-auto-embedding-trigger-sql).

---

## 6. The Activation Problem

CLI AI tools have access to your shell environment. If you set `OB_SUPABASE_URL` and
`OB_OPENROUTER_KEY` as environment variables, the AI can see them. But having the
credentials available does not mean the user wants Open Brain active in every
conversation.

You might be debugging a Python script and don't want the AI deciding to capture
stray thoughts. You might be in a work project where personal memory capture is
inappropriate. The AI needs a clear signal: "Open Brain is available AND the user
wants it active right now."

### Options Evaluated

| Approach | How it works | Verdict |
|---|---|---|
| **Per-project CLAUDE.md** | Add Open Brain instructions to the project's `.claude/CLAUDE.md` | Only works in projects where you opt in. Good for project-level control. |
| **Global prompt** | Add instructions to `~/.claude/CLAUDE.md` | Always active. Too aggressive — you don't want memory capture in every context. |
| **Keyword trigger** | AI activates when user says "remember this" or "search my brain" | Fragile. The AI might miss triggers or false-positive on similar phrases. |
| **Dedicated CLI tool** | `ob` command — AI executes it only when instructed | Explicit. The user or the AI runs `ob capture "..."` or `ob search "..."`. Clear intent signal. |
| **Environment variable flag** | `OB_ACTIVE=true` — AI checks before using Open Brain | Programmatic. Can be toggled per-session with `export OB_ACTIVE=true`. |

### Recommended: `ob` CLI + CLAUDE.md Instruction

The recommended approach combines two mechanisms:

1. **The `ob` CLI tool** — a bash script that wraps the embedding pipeline and
   Supabase REST API calls into simple commands (`ob capture`, `ob search`,
   `ob recent`, `ob stats`)

2. **A CLAUDE.md instruction** — tells the AI that the `ob` tool exists but should
   only be used when the user explicitly asks

The CLAUDE.md instruction:

```markdown
## Open Brain (Personal Memory)

You have access to the `ob` CLI tool for personal memory management.
Commands: `ob capture "text"`, `ob search "query"`, `ob recent`, `ob stats`.

**IMPORTANT:** Only use `ob` when the user explicitly asks to capture, search,
or review their personal memories. Do NOT proactively capture thoughts, summarize
conversations into memories, or search the knowledge base unless asked.
```

This creates a clear activation pattern:
- The AI knows the tool exists (from CLAUDE.md)
- The AI will not use it unless asked (from the instruction)
- The user triggers it with natural language: "remember this", "search my brain
  for...", "capture this thought"
- The AI translates the intent into an `ob` command and executes it

---

## 7. The `ob` CLI Tool Design

### 7.1 Command Reference

```
ob capture "thought text"     — Save a thought with embedding + metadata
ob search  "query text"       — Semantic search across all thoughts
ob recent  [count]            — List recent thoughts (default: 10)
ob stats                      — Show aggregate statistics
ob version                    — Print version and config status
```

#### `ob capture`

Saves a thought to the `thoughts` table with a vector embedding and extracted metadata.

```bash
$ ob capture "Sarah mentioned she wants to start a consulting business focused on UX"

{
  "success": true,
  "id": "a3f8b2c1-d4e5-6789-abcd-ef0123456789",
  "content": "Sarah mentioned she wants to start a consulting business focused on UX",
  "metadata": {
    "type": "person_note",
    "topics": ["consulting", "UX", "career"],
    "people": ["Sarah"],
    "action_items": []
  }
}
```

Pipeline:
1. Calls OpenRouter `text-embedding-3-small` to generate a 1536-float vector
2. Calls OpenRouter `gpt-4o-mini` to extract metadata (type, topics, people, action items)
3. Inserts the row into `thoughts` with content, embedding, and metadata

The two OpenRouter calls run sequentially in the bash implementation (parallelization
would require background processes and adds complexity for minimal gain in a CLI tool).

#### `ob search`

Performs semantic search — finds thoughts by meaning, not keywords.

```bash
$ ob search "career planning"

Found 3 results:

[0.89] Sarah mentioned she wants to start a consulting business focused on UX
       Topics: consulting, UX, career | People: Sarah
       Captured: 2026-03-10T14:22:00Z

[0.82] Need to update my resume with the Q4 project results
       Topics: resume, career | Type: task
       Captured: 2026-03-08T09:15:00Z

[0.74] Marcus asked if I'd be interested in the VP Engineering opening at his company
       Topics: job opportunity, leadership | People: Marcus
       Captured: 2026-03-05T16:45:00Z
```

Pipeline:
1. Calls OpenRouter to embed the query string
2. Calls `match_thoughts()` via Supabase RPC with the query vector
3. Formats results with similarity scores and metadata

Optional flags:
- `ob search "query" --threshold 0.5` — lower threshold for broader results
- `ob search "query" --count 20` — return more results
- `ob search "query" --json` — output raw JSON instead of formatted text

#### `ob recent`

Lists the most recently captured thoughts in reverse chronological order.

```bash
$ ob recent 5

5 most recent thoughts:

1. Sarah mentioned she wants to start a consulting business focused on UX
   Topics: consulting, UX, career | People: Sarah
   Captured: 2026-03-10T14:22:00Z

2. Decided to push the product launch to March 15 to allow more QA time
   Topics: launch, QA | Type: observation
   Captured: 2026-03-09T11:30:00Z

...
```

Pipeline: Single REST API call to `GET /rest/v1/thoughts?order=created_at.desc&limit=N`.

#### `ob stats`

Returns aggregate statistics about stored thoughts.

```bash
$ ob stats

Open Brain Statistics:
  Total thoughts: 247
  Oldest: 2025-11-01
  Newest: 2026-03-13
  Top topics: consulting, product, family, hiring, AI
  Top people: Sarah, Marcus, Jen
  Type distribution:
    task: 78
    person_note: 55
    idea: 42
    observation: 41
    reference: 31
```

Pipeline: Multiple aggregate queries against the `thoughts` table via the REST API.

### 7.2 Environment Variables

All environment variables use the `OB_` prefix to avoid collisions with other tools.

| Variable | Required | Description |
|---|---|---|
| `OB_SUPABASE_URL` | Yes | Supabase project URL (e.g., `https://xxxx.supabase.co`) |
| `OB_SUPABASE_KEY` | Yes | Supabase service role key (full access) |
| `OB_OPENROUTER_KEY` | Yes | OpenRouter API key (for embeddings and metadata) |
| `OB_THRESHOLD` | No | Default similarity threshold (default: `0.7`) |
| `OB_COUNT` | No | Default result count (default: `10`) |

**Where to set them:**

```bash
# In ~/.bashrc, ~/.zshrc, or ~/.profile
export OB_SUPABASE_URL="https://your-project-ref.supabase.co"
export OB_SUPABASE_KEY="eyJhbGciOiJIUzI1NiIs..."
export OB_OPENROUTER_KEY="sk-or-v1-..."
```

**Security note:** The `OB_SUPABASE_KEY` is the service role key, which has full
database access. Do not commit it to any repository. Do not share it. If it leaks,
rotate it immediately in the Supabase dashboard under Settings → API.

### 7.3 Implementation: Bash Script

The `ob` CLI is implemented as a single bash script with no dependencies beyond
`curl` and `jq`. Both are available on macOS, Linux, and Windows (via Git Bash,
WSL, or MSYS2).

**Why bash?**
- Zero runtime dependencies — no Node.js, no Python, no package manager
- Works in any environment where the CLI AI tool runs
- The AI can read and understand the script instantly
- Easy to modify — it is just `curl` calls with `jq` formatting

**Why not a compiled binary?**
- A binary requires a build toolchain and cross-compilation for each platform
- The script does nothing computationally intensive — it makes HTTP calls
- Transparency matters: the user and the AI can inspect exactly what the script does

The complete script is in [Appendix A](#appendix-a-complete-ob-script).

### 7.4 Installation

**Step 1 — Download the script:**

```bash
# Create a directory for the script (if it doesn't exist)
mkdir -p ~/.local/bin

# Copy the script (from Appendix A or the repo)
cp ob ~/.local/bin/ob
chmod +x ~/.local/bin/ob
```

**Step 2 — Add to PATH (if `~/.local/bin` is not already in PATH):**

```bash
# Add to ~/.bashrc or ~/.zshrc
export PATH="$HOME/.local/bin:$PATH"
```

**Step 3 — Set environment variables:**

```bash
# Add to ~/.bashrc or ~/.zshrc
export OB_SUPABASE_URL="https://your-project-ref.supabase.co"
export OB_SUPABASE_KEY="your-service-role-key"
export OB_OPENROUTER_KEY="your-openrouter-key"
```

**Step 4 — Verify:**

```bash
$ ob version

ob v1.0.0
Supabase URL: https://xxxx.supabase.co (configured)
OpenRouter:   configured
```

```bash
$ ob stats

Open Brain Statistics:
  Total thoughts: 247
  ...
```

If `ob stats` returns results, you are connected and ready.

---

## 8. Configuration for Each CLI AI Tool

### 8.1 Claude Code

Claude Code reads instructions from `CLAUDE.md` files. The most flexible approach is
a global instruction file combined with per-project opt-in.

**Global instruction (`~/.claude/CLAUDE.md`):**

```markdown
## Open Brain (Personal Memory)

You have access to the `ob` CLI tool for personal memory management.
Commands: `ob capture "text"`, `ob search "query"`, `ob recent [count]`, `ob stats`.

**Rules:**
- Only use `ob` when I explicitly ask to capture, search, or review memories
- Do NOT proactively capture thoughts or search the knowledge base
- When I say "remember this" or "capture this", use `ob capture`
- When I say "search my brain" or "what do I know about", use `ob search`
- Always show me the result of `ob` commands
```

**Per-project opt-in (project's `CLAUDE.md`):**

If you want Open Brain active for a specific project, add to that project's
`CLAUDE.md`:

```markdown
## Open Brain

Open Brain is active for this project. You may proactively suggest capturing
important decisions, architectural choices, and meeting notes using `ob capture`.
Search the knowledge base with `ob search` when context from past decisions
would be relevant.
```

### 8.2 OpenAI Codex

Codex reads from `.codex.md` in the project root or `~/.codex/instructions.md`
globally.

**`~/.codex/instructions.md`:**

```markdown
## Open Brain (Personal Memory)

You have access to the `ob` CLI tool for personal memory management.
Run shell commands: `ob capture "text"`, `ob search "query"`, `ob recent`, `ob stats`.

Only use when I explicitly ask. Do not proactively capture or search.
```

### 8.3 Gemini CLI

Gemini CLI reads from `GEMINI.md` in the project root or `~/.gemini/instructions.md`.

**`~/.gemini/instructions.md`:**

```markdown
## Open Brain (Personal Memory)

You have access to the `ob` CLI tool for personal memory management.
Run shell commands: `ob capture "text"`, `ob search "query"`, `ob recent`, `ob stats`.

Only use when I explicitly ask. Do not proactively capture or search.
```

### 8.4 Universal Instruction Template

For any CLI AI tool that reads a system instruction or project configuration file:

```markdown
## Open Brain (Personal Memory)

The `ob` CLI tool provides personal memory management via a Supabase-hosted
PostgreSQL database with vector search.

Available commands (execute as shell commands):
- `ob capture "text"` — Save a thought with semantic embedding
- `ob search "query"` — Find related thoughts by meaning
- `ob recent [N]` — List N most recent thoughts (default 10)
- `ob stats` — Show knowledge base statistics

Activation rules:
1. Only use when the user explicitly requests memory operations
2. Do NOT capture thoughts proactively or search without being asked
3. Show the full output of every `ob` command to the user
4. For captures, confirm what was saved
5. For searches, present results with similarity scores
```

---

## 9. Comparison Matrix: MCP vs CLI-Direct

| Dimension | MCP Approach | CLI-Direct Approach |
|---|---|---|
| **Works with GUI clients** | Yes (Claude Desktop, ChatGPT) | No |
| **Works with CLI clients** | Yes (via `claude mcp add`) | Yes (via `ob` commands) |
| **Requires Edge Function** | Yes | No (unless using method 5.3 or 5.4) |
| **Requires deployment** | Yes (`supabase functions deploy`) | No (copy script to PATH) |
| **Embedding pipeline** | Built into Edge Function | Built into `ob` script |
| **Metadata extraction** | Built into Edge Function | Built into `ob` script |
| **Dependencies** | Supabase CLI, Deno, MCP SDK, Hono, Zod | `curl`, `jq` |
| **Setup time** | 30–45 minutes | 5–10 minutes |
| **Protocol overhead** | JSON-RPC framing, MCP transport | None — raw HTTP |
| **Multi-client consistency** | One server, any client | One script, CLI clients only |
| **Debugging** | Edge Function logs in Supabase dashboard | Direct `curl` output in terminal |
| **Credential management** | Supabase secrets store (encrypted) | Shell environment variables |
| **Offline capability** | No (Edge Function is cloud) | No (still needs Supabase + OpenRouter) |
| **Cost** | Same API costs | Same API costs |
| **Data compatibility** | Same `thoughts` table | Same `thoughts` table |

### When to Use Which

| Scenario | Recommendation |
|---|---|
| You only use CLI AI tools (Claude Code, Codex) | **CLI-Direct** — skip MCP entirely |
| You use both CLI and GUI clients | **Both** — MCP for GUI, CLI-Direct for CLI |
| You already have MCP deployed and it works | **Keep MCP** — add `ob` as a convenience |
| You want the simplest possible setup | **CLI-Direct** — fewer moving parts |
| You want to share access with non-technical users | **MCP** — they can't use CLI tools |
| You are building on top of Open Brain for others | **MCP** — it is the standardized interface |

---

## 10. Migration and Coexistence

### There Is No Migration

The CLI-Direct approach does not replace anything. It does not require changes to
existing infrastructure. Both paths write to the same `thoughts` table with the same
schema:

```sql
thoughts (
  id          uuid default gen_random_uuid() primary key,
  content     text not null,
  embedding   vector(1536),
  metadata    jsonb default '{}'::jsonb,
  created_at  timestamptz default now(),
  updated_at  timestamptz default now()
)
```

A thought captured via `ob capture` is indistinguishable from a thought captured via
the MCP `capture_thought` tool. Both generate embeddings using `text-embedding-3-small`
via OpenRouter. Both extract metadata using `gpt-4o-mini`. Both insert the result into
the same table with the same three indexes (HNSW for vector search, GIN for metadata
filtering, BTREE for date ordering).

### What Coexistence Looks Like

```
Monday morning, laptop:
  Claude Desktop → MCP → "capture: team standup decisions" → thoughts table

Monday afternoon, terminal:
  Claude Code → ob capture "architecture decision: use event sourcing for audit log"
  → thoughts table

Tuesday, phone:
  ChatGPT (web) → MCP → "search: what did we decide about the audit log?"
  → finds Monday afternoon's capture via semantic search

Tuesday evening, terminal:
  Claude Code → ob search "audit log decisions"
  → finds the same capture
```

Every entry in the table is searchable by every path. There is no path-specific data,
no format differences, no compatibility layer needed.

### Adding CLI-Direct to an Existing Setup

If you already have Open Brain running with MCP:

1. Install the `ob` script (Section 7.4)
2. Set the `OB_` environment variables using your existing credentials:
   - `OB_SUPABASE_URL` — same as your Supabase project URL
   - `OB_SUPABASE_KEY` — same as your `SUPABASE_SERVICE_ROLE_KEY`
   - `OB_OPENROUTER_KEY` — same as your `OPENROUTER_API_KEY`
3. Add the CLAUDE.md instruction (Section 8.1)
4. Verify with `ob stats` — should show the same thought count as the MCP `stats` tool

Total time: under 5 minutes.

### Starting Fresh with CLI-Direct Only

If you do not want MCP at all and are starting from scratch:

1. Create a Supabase project (Steps 1–2 of `docs/01-getting-started.md`)
2. Run the SQL to create the `thoughts` table, `match_thoughts()` function, and indexes
3. Get an OpenRouter API key (Step 4 of `docs/01-getting-started.md`)
4. Install `ob` and set environment variables
5. Skip Steps 5–7 (Edge Function deployment and MCP configuration)

You save roughly 20 minutes by skipping the Edge Function deployment, Supabase CLI
installation, secret management, and MCP client configuration.

---

## 11. Troubleshooting

### `ob capture` fails with "Could not resolve host"

The `OB_SUPABASE_URL` environment variable is not set or is malformed.

```bash
echo $OB_SUPABASE_URL
# Should print: https://your-project-ref.supabase.co
```

### `ob capture` returns 401 Unauthorized

The `OB_SUPABASE_KEY` is wrong or expired. Verify it matches the service role key in
the Supabase dashboard under Settings → API → Service role key (not the anon key).

### `ob search` returns no results but thoughts exist

1. **Threshold too high:** Try `ob search "query" --threshold 0.3` for broader matches
2. **No embeddings:** Thoughts inserted without embeddings (e.g., via raw SQL or REST
   without the embedding step) are invisible to semantic search. Check:
   ```bash
   curl -s "$OB_SUPABASE_URL/rest/v1/thoughts?embedding=is.null&select=id,content" \
     -H "apikey: $OB_SUPABASE_KEY" \
     -H "Authorization: Bearer $OB_SUPABASE_KEY" | jq .
   ```
   If this returns rows, those thoughts have no embedding. Re-capture them with
   `ob capture` or run a backfill script.

### `ob` command not found

The script is not in your PATH. Verify:

```bash
which ob
# Should print a path like /home/user/.local/bin/ob

# If not found:
ls -la ~/.local/bin/ob
# Should show the script exists and is executable (-rwxr-xr-x)
```

### OpenRouter returns 402 Payment Required

Your OpenRouter account has insufficient credits. Add credits at
https://openrouter.ai/credits. Embedding calls cost approximately $0.00002 per
thought (1536-dimensional embedding of a few sentences). Metadata extraction costs
approximately $0.0001 per thought (a short `gpt-4o-mini` chat completion).

### `jq` not found

Install `jq`:

```bash
# macOS
brew install jq

# Ubuntu/Debian
sudo apt install jq

# Windows (Git Bash / MSYS2)
# jq is included with Git for Windows. If missing:
curl -L -o /usr/bin/jq.exe https://github.com/jqlang/jq/releases/latest/download/jq-windows-amd64.exe
```

### Thoughts captured via `ob` don't appear in MCP search (or vice versa)

This should not happen — both paths use the same table. If it does:

1. Verify both are pointing to the same Supabase project (compare URLs)
2. Check that the `ob` insertion actually succeeded: `ob recent 1` should show the
   latest capture
3. Check for RLS issues: the service role key bypasses RLS, but if you are using
   the anon key somewhere, RLS policies may filter results

### Slow response from `ob capture`

`ob capture` makes two sequential API calls to OpenRouter and one to Supabase. Typical
total time is 500ms–2s depending on network latency and API response times. If it
consistently takes more than 5 seconds:

1. Test OpenRouter directly: `curl -w "\n%{time_total}s\n" https://openrouter.ai/api/v1/models`
2. Test Supabase directly: `curl -w "\n%{time_total}s\n" "$OB_SUPABASE_URL/rest/v1/thoughts?limit=1" -H "apikey: $OB_SUPABASE_KEY" -H "Authorization: Bearer $OB_SUPABASE_KEY"`
3. If one service is slow, the issue is with that service, not with `ob`

---

## Appendix A: Complete `ob` Script

Copy this entire script to `~/.local/bin/ob` and run `chmod +x ~/.local/bin/ob`.

```bash
#!/usr/bin/env bash
#
# ob — Open Brain CLI
# A lightweight command-line interface for Open Brain personal memory.
# Dependencies: curl, jq
# License: FSL-1.1-MIT (same as Open Brain)
#
set -euo pipefail

OB_VERSION="1.0.0"

# ─── Configuration ────────────────────────────────────────────────────────────

: "${OB_SUPABASE_URL:=}"
: "${OB_SUPABASE_KEY:=}"
: "${OB_OPENROUTER_KEY:=}"
: "${OB_THRESHOLD:=0.7}"
: "${OB_COUNT:=10}"

OPENROUTER_BASE="https://openrouter.ai/api/v1"

# ─── Helpers ──────────────────────────────────────────────────────────────────

die() { echo "ob: error: $*" >&2; exit 1; }

check_deps() {
  command -v curl >/dev/null 2>&1 || die "curl is required but not installed"
  command -v jq   >/dev/null 2>&1 || die "jq is required but not installed"
}

check_config() {
  [[ -n "$OB_SUPABASE_URL" ]]  || die "OB_SUPABASE_URL is not set"
  [[ -n "$OB_SUPABASE_KEY" ]]  || die "OB_SUPABASE_KEY is not set"
  [[ -n "$OB_OPENROUTER_KEY" ]] || die "OB_OPENROUTER_KEY is not set"
}

supabase_headers() {
  echo -H "apikey: $OB_SUPABASE_KEY" -H "Authorization: Bearer $OB_SUPABASE_KEY"
}

# ─── API Functions ────────────────────────────────────────────────────────────

get_embedding() {
  local text="$1"
  curl -sf "$OPENROUTER_BASE/embeddings" \
    -H "Authorization: Bearer $OB_OPENROUTER_KEY" \
    -H "Content-Type: application/json" \
    -d "$(jq -n --arg t "$text" '{model: "openai/text-embedding-3-small", input: $t}')" \
    | jq -c '.data[0].embedding'
}

extract_metadata() {
  local text="$1"
  local system_prompt='Extract metadata from the user'\''s captured thought. Return JSON with: "people" (array of people mentioned, empty if none), "action_items" (array of implied to-dos, empty if none), "topics" (array of 1-3 short topic tags, always at least one), "type" (one of "observation", "task", "idea", "reference", "person_note"). Only extract what'\''s explicitly there.'

  curl -sf "$OPENROUTER_BASE/chat/completions" \
    -H "Authorization: Bearer $OB_OPENROUTER_KEY" \
    -H "Content-Type: application/json" \
    -d "$(jq -n --arg t "$text" --arg s "$system_prompt" '{
      model: "openai/gpt-4o-mini",
      response_format: {type: "json_object"},
      messages: [
        {role: "system", content: $s},
        {role: "user", content: $t}
      ]
    }')" \
    | jq -c '.choices[0].message.content | fromjson'
}

# ─── Commands ─────────────────────────────────────────────────────────────────

cmd_capture() {
  local content="${1:?Usage: ob capture \"thought text\"}"

  echo "Generating embedding..." >&2
  local embedding
  embedding=$(get_embedding "$content") || die "Failed to generate embedding"

  echo "Extracting metadata..." >&2
  local metadata
  metadata=$(extract_metadata "$content") || metadata='{"topics":["uncategorized"],"type":"observation"}'

  echo "Saving thought..." >&2
  local result
  result=$(curl -sf -X POST "$OB_SUPABASE_URL/rest/v1/thoughts" \
    -H "apikey: $OB_SUPABASE_KEY" \
    -H "Authorization: Bearer $OB_SUPABASE_KEY" \
    -H "Content-Type: application/json" \
    -H "Prefer: return=representation" \
    -d "$(jq -n \
      --arg c "$content" \
      --argjson e "$embedding" \
      --argjson m "$metadata" \
      '{content: $c, embedding: $e, metadata: $m}')" \
  ) || die "Failed to insert thought"

  echo "$result" | jq '{
    success: true,
    id: .[0].id,
    content: .[0].content,
    metadata: .[0].metadata,
    created_at: .[0].created_at
  }'
}

cmd_search() {
  local query="${1:?Usage: ob search \"query text\"}"
  local threshold="${OB_THRESHOLD}"
  local count="${OB_COUNT}"
  local json_output=false

  shift
  while [[ $# -gt 0 ]]; do
    case "$1" in
      --threshold) threshold="$2"; shift 2 ;;
      --count)     count="$2"; shift 2 ;;
      --json)      json_output=true; shift ;;
      *) die "Unknown flag: $1" ;;
    esac
  done

  local embedding
  embedding=$(get_embedding "$query") || die "Failed to generate query embedding"

  local results
  results=$(curl -sf -X POST "$OB_SUPABASE_URL/rest/v1/rpc/match_thoughts" \
    -H "apikey: $OB_SUPABASE_KEY" \
    -H "Authorization: Bearer $OB_SUPABASE_KEY" \
    -H "Content-Type: application/json" \
    -d "$(jq -n \
      --argjson e "$embedding" \
      --argjson t "$threshold" \
      --argjson c "$count" \
      '{query_embedding: $e, match_threshold: $t, match_count: $c, filter: {}}')" \
  ) || die "Failed to search thoughts"

  if [[ "$json_output" == "true" ]]; then
    echo "$results" | jq .
    return
  fi

  local result_count
  result_count=$(echo "$results" | jq 'length')

  if [[ "$result_count" -eq 0 ]]; then
    echo "No results found for: $query"
    return
  fi

  echo "Found $result_count results:"
  echo ""

  echo "$results" | jq -r '.[] | "[" + (.similarity * 100 | round / 100 | tostring) + "] " + .content + "\n       Topics: " + ((.metadata.topics // []) | join(", ")) + (if (.metadata.people // []) | length > 0 then " | People: " + ((.metadata.people // []) | join(", ")) else "" end) + (if .metadata.type then " | Type: " + .metadata.type else "" end) + "\n       Captured: " + .created_at + "\n"'
}

cmd_recent() {
  local count="${1:-$OB_COUNT}"

  local results
  results=$(curl -sf \
    "$OB_SUPABASE_URL/rest/v1/thoughts?order=created_at.desc&limit=$count&select=id,content,metadata,created_at" \
    -H "apikey: $OB_SUPABASE_KEY" \
    -H "Authorization: Bearer $OB_SUPABASE_KEY" \
  ) || die "Failed to list thoughts"

  local result_count
  result_count=$(echo "$results" | jq 'length')

  echo "$result_count most recent thoughts:"
  echo ""

  echo "$results" | jq -r 'to_entries[] | "\(.key + 1). \(.value.content)\n   Topics: \((.value.metadata.topics // []) | join(", "))" + (if (.value.metadata.people // []) | length > 0 then " | People: \((.value.metadata.people // []) | join(", "))" else "" end) + "\n   Captured: \(.value.created_at)\n"'
}

cmd_stats() {
  # Total count
  local total
  total=$(curl -sf \
    "$OB_SUPABASE_URL/rest/v1/thoughts?select=id" \
    -H "apikey: $OB_SUPABASE_KEY" \
    -H "Authorization: Bearer $OB_SUPABASE_KEY" \
    -H "Prefer: count=exact" \
    -H "Range: 0-0" \
    -I 2>/dev/null | grep -i content-range | sed 's/.*\///' | tr -d '\r\n') || total="unknown"

  # Recent thoughts for date range and aggregation
  local all_thoughts
  all_thoughts=$(curl -sf \
    "$OB_SUPABASE_URL/rest/v1/thoughts?select=metadata,created_at&order=created_at.asc" \
    -H "apikey: $OB_SUPABASE_KEY" \
    -H "Authorization: Bearer $OB_SUPABASE_KEY" \
  ) || die "Failed to fetch thoughts"

  local oldest newest
  oldest=$(echo "$all_thoughts" | jq -r 'first.created_at // "N/A"' | cut -d'T' -f1)
  newest=$(echo "$all_thoughts" | jq -r 'last.created_at // "N/A"' | cut -d'T' -f1)

  local top_topics
  top_topics=$(echo "$all_thoughts" | jq -r '[.[].metadata.topics // [] | .[]] | group_by(.) | map({topic: .[0], count: length}) | sort_by(-.count) | .[0:5] | .[].topic')

  local top_people
  top_people=$(echo "$all_thoughts" | jq -r '[.[].metadata.people // [] | .[]] | group_by(.) | map({person: .[0], count: length}) | sort_by(-.count) | .[0:5] | .[].person')

  local type_dist
  type_dist=$(echo "$all_thoughts" | jq -r '[.[].metadata.type // "unknown"] | group_by(.) | map("    " + .[0] + ": " + (length | tostring)) | .[]')

  echo "Open Brain Statistics:"
  echo "  Total thoughts: $total"
  echo "  Oldest: $oldest"
  echo "  Newest: $newest"
  echo "  Top topics: $(echo "$top_topics" | tr '\n' ', ' | sed 's/,$//' | sed 's/,/, /g')"
  echo "  Top people: $(echo "$top_people" | tr '\n' ', ' | sed 's/,$//' | sed 's/,/, /g')"
  echo "  Type distribution:"
  echo "$type_dist"
}

cmd_version() {
  echo "ob v$OB_VERSION"
  if [[ -n "$OB_SUPABASE_URL" ]]; then
    echo "Supabase URL: $(echo "$OB_SUPABASE_URL" | sed 's|https://\([^.]*\)\..*|\1|')... (configured)"
  else
    echo "Supabase URL: not configured"
  fi
  if [[ -n "$OB_OPENROUTER_KEY" ]]; then
    echo "OpenRouter:   configured"
  else
    echo "OpenRouter:   not configured"
  fi
}

# ─── Main ─────────────────────────────────────────────────────────────────────

main() {
  check_deps

  local cmd="${1:-help}"
  shift 2>/dev/null || true

  case "$cmd" in
    capture)  check_config; cmd_capture "$@" ;;
    search)   check_config; cmd_search "$@" ;;
    recent)   check_config; cmd_recent "$@" ;;
    stats)    check_config; cmd_stats ;;
    version)  cmd_version ;;
    help|--help|-h)
      echo "ob — Open Brain CLI v$OB_VERSION"
      echo ""
      echo "Usage:"
      echo "  ob capture \"thought text\"        Save a thought with embedding + metadata"
      echo "  ob search  \"query\" [--threshold N] [--count N] [--json]"
      echo "                                    Semantic search across thoughts"
      echo "  ob recent  [count]                List recent thoughts (default: 10)"
      echo "  ob stats                          Show knowledge base statistics"
      echo "  ob version                        Print version and config status"
      echo ""
      echo "Environment variables:"
      echo "  OB_SUPABASE_URL    Supabase project URL (required)"
      echo "  OB_SUPABASE_KEY    Supabase service role key (required)"
      echo "  OB_OPENROUTER_KEY  OpenRouter API key (required)"
      echo "  OB_THRESHOLD       Default similarity threshold (default: 0.7)"
      echo "  OB_COUNT           Default result count (default: 10)"
      ;;
    *)
      die "Unknown command: $cmd (run 'ob help' for usage)"
      ;;
  esac
}

main "$@"
```

---

## Appendix B: PostgreSQL Auto-Embedding Trigger SQL

This trigger automatically generates embeddings for new thoughts using `pg_net`, a
Supabase extension that makes HTTP calls from within PostgreSQL. With this trigger,
any insert into the `thoughts` table — from any client, any path — gets an embedding
automatically.

**Prerequisites:**
- `pg_net` extension enabled (Supabase dashboard → Database → Extensions → search
  `pg_net` → Enable)
- A small Edge Function or external endpoint that accepts a thought ID, generates
  the embedding, and writes it back

### The Embedding Webhook Function

Deploy this as a Supabase Edge Function (`supabase/functions/embed-thought/index.ts`):

```typescript
import { createClient } from "@supabase/supabase-js";

const SUPABASE_URL = Deno.env.get("SUPABASE_URL")!;
const SUPABASE_SERVICE_ROLE_KEY = Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!;
const OPENROUTER_API_KEY = Deno.env.get("OPENROUTER_API_KEY")!;

const supabase = createClient(SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY);

Deno.serve(async (req) => {
  const { id, content } = await req.json();

  // Generate embedding
  const embedRes = await fetch("https://openrouter.ai/api/v1/embeddings", {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${OPENROUTER_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      model: "openai/text-embedding-3-small",
      input: content,
    }),
  });
  const embedData = await embedRes.json();
  const embedding = embedData.data[0].embedding;

  // Update the thought with the embedding
  const { error } = await supabase
    .from("thoughts")
    .update({ embedding })
    .eq("id", id);

  if (error) {
    return new Response(JSON.stringify({ error: error.message }), { status: 500 });
  }

  return new Response(JSON.stringify({ success: true, id }), { status: 200 });
});
```

Deploy: `supabase functions deploy embed-thought --no-verify-jwt`

### The Database Trigger

```sql
-- Enable pg_net if not already enabled
create extension if not exists pg_net with schema extensions;

-- Create the trigger function
create or replace function trigger_embed_thought()
returns trigger
language plpgsql
security definer
as $$
begin
  -- Only fire if embedding is null (avoid infinite loops on update)
  if NEW.embedding is null then
    perform extensions.http_post(
      url := current_setting('app.settings.supabase_url') || '/functions/v1/embed-thought',
      body := json_build_object('id', NEW.id, 'content', NEW.content)::text,
      headers := json_build_object(
        'Content-Type', 'application/json',
        'Authorization', 'Bearer ' || current_setting('app.settings.service_role_key')
      )::jsonb
    );
  end if;
  return NEW;
end;
$$;

-- Attach the trigger to the thoughts table
create trigger thoughts_auto_embed
  after insert on thoughts
  for each row
  execute function trigger_embed_thought();

-- Set the configuration values (run once)
-- Replace with your actual values
alter database postgres set app.settings.supabase_url = 'https://YOUR_PROJECT_REF.supabase.co';
alter database postgres set app.settings.service_role_key = 'your-service-role-key';
```

### How It Works

1. A client inserts a row into `thoughts` with `content` and optionally `metadata`
   — no embedding required
2. The `thoughts_auto_embed` trigger fires after the insert
3. The trigger function checks if `embedding` is null (to avoid loops)
4. If null, it makes an HTTP POST to the `embed-thought` Edge Function via `pg_net`
5. The Edge Function calls OpenRouter to generate the embedding
6. The Edge Function updates the `thoughts` row with the embedding vector
7. The thought is now searchable via `match_thoughts()`

**Timing:** The embedding is typically available within 500ms–1s after insert. During
this window, the thought exists in the table but is not searchable. For most use cases,
this latency is imperceptible.

**Important:** The `alter database` commands store credentials in PostgreSQL
configuration. These are accessible to anyone with database admin access. For a
personal single-user setup, this is acceptable. For shared environments, consider
using Supabase Vault instead.

### Adding Metadata Extraction to the Trigger

To also extract metadata automatically, modify the `embed-thought` Edge Function to
call `gpt-4o-mini` for metadata extraction (same pattern as the `extractMetadata`
function in the core MCP server — see `docs/mcp-servers/00-core-mcp-server.md`
Section 5.1 for the implementation) and update both `embedding` and `metadata` in a
single update call.

---

*This document describes the CLI-Direct approach to Open Brain as of the date this
file was committed. It is complementary to the MCP-based approach described in
`docs/01-getting-started.md` — both paths are fully supported and interoperable.*
