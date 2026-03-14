# Core Open Brain MCP Server — Deep Dive

This document covers the technical internals of the core Open Brain MCP server: what it
is, how it is structured, how it authenticates, what each tool does at the code level, and
how it fits into the broader system.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Role in the Architecture](#2-role-in-the-architecture)
3. [Transport: HTTP](#3-transport-http)
4. [Authentication](#4-authentication)
5. [The Four Tools](#5-the-four-tools)
6. [Parallel Processing in capture_thought](#6-parallel-processing-in-capture_thought)
7. [Dependencies](#7-dependencies)
8. [Deployment](#8-deployment)
9. [How It Connects to Other Components](#9-how-it-connects-to-other-components)
10. [Source File Reference](#10-source-file-reference)

---

## 1. Overview

The core Open Brain MCP server is the central hub of the entire system. It is a Deno
TypeScript program deployed as a Supabase Edge Function and accessed over HTTP. Any
MCP-compatible AI client — Claude Desktop, ChatGPT, Claude Code, Cursor — can connect to
it by POSTing JSON-RPC messages to its URL.

It exposes four tools:

| Tool | What it does |
|------|-------------|
| `capture_thought` | Writes a new thought: embeds it, extracts metadata, inserts into `thoughts` |
| `search_thoughts` | Reads via semantic vector search using cosine similarity |
| `list_thoughts` | Reads via chronological pagination (no embedding required) |
| `stats` | Returns aggregate statistics about what is stored |

This is the only MCP server in Open Brain that runs in the cloud. All six extension
servers run locally on the user's machine via stdio. The core server is what makes Open
Brain reachable from any device, anywhere — a laptop, a phone via ChatGPT web, a CI
runner via Claude Code.

**Deployment path:** `supabase/functions/open-brain-mcp/index.ts`

**Live URL pattern:**

```
https://YOUR_PROJECT_REF.supabase.co/functions/v1/open-brain-mcp
```

---

## 2. Role in the Architecture

Open Brain has seven MCP servers total: one core server and six extension servers. This
diagram shows where the core server sits relative to everything else.

```
+---------------------------------------------------------------------+
|                        AI CLIENTS                                   |
|                                                                     |
|   Claude Desktop   ChatGPT (web)   Claude Code   Cursor   others   |
+----------+-----------+----------+------+----------+-------+---------+
           |           |          |      |          |
           |           |    HTTP (this server)      |  stdio (ext servers, local only)
           |           |          |      |          |
           +-----+-----+----------+      +----------+
                 |                              |
                 v                              v
    +-------------------------+    +-----------------------------+
    |  open-brain-mcp         |    |  Extension servers          |
    |  (Supabase Edge Fn)     |    |  household-knowledge        |
    |  Deno runtime           |    |  home-maintenance           |
    |  CLOUD                  |    |  family-calendar            |
    |                         |    |  meal-planning              |
    |  capture_thought        |    |  professional-crm           |
    |  search_thoughts        |    |  job-hunt                   |
    |  list_thoughts          |    |  (TypeScript, Node.js)      |
    |  stats                  |    |  LOCAL MACHINE              |
    +----------+--------------+    +-------------+---------------+
               |                                 |
               |  OpenRouter API calls           | Supabase JS client
               v                                 v
    +--------------------+        +------------------------------+
    |  OpenRouter        |        |  Supabase PostgreSQL         |
    |  (openrouter.ai)   |        |                              |
    |                    |        |  thoughts table (pgvector)   |
    |  text-embedding    +------->+  HNSW index                  |
    |    -3-small        |        |  match_thoughts() function   |
    |  gpt-4o-mini       |        |                              |
    +--------------------+        |  + extension tables          |
                                  |    (household_*, family_*,   |
                                  |     maintenance_*, etc.)     |
                                  +------------------------------+
```

**The core server is the only component that calls OpenRouter.** Extension servers
talk directly to PostgreSQL via the Supabase JS client — they do no embedding and no LLM
calls. Only the core server requires an `OPENROUTER_API_KEY`.

**The core server is the only server that runs in the cloud.** Extension servers run
as Node.js processes launched by the AI client on the user's local machine. They are
only reachable on that machine. The core server is a permanent HTTPS endpoint that
is reachable from anywhere.

---

## 3. Transport: HTTP

The MCP protocol supports two transport mechanisms. The core server uses HTTP. Extension
servers use stdio. The distinction matters.

```
Core server (HTTP transport)
+------------------+          HTTPS POST          +------------------+
|                  |                               |                  |
|  Claude Desktop  |  /functions/v1/open-brain-mcp |  Supabase Edge   |
|  (your laptop)   | ----------------------------> |  Function        |
|                  |  body: JSON-RPC message        |  (cloud)         |
|                  | <---------------------------- |                  |
|                  |  body: JSON-RPC response       |                  |
+------------------+                               +------------------+

Extension servers (stdio transport)
+------------------+    stdin/stdout pipe    +------------------+
|                  |                          |                  |
|  Claude Desktop  |  writes JSON-RPC ------> |  Node.js process |
|  (your laptop)   |  to stdin                |  (your laptop)   |
|                  |  reads JSON-RPC  <------  |                  |
|                  |  from stdout              |                  |
+------------------+                          +------------------+
```

**Why HTTP for the core server?** The core server needs to be reachable from any device
the user works from. Claude Desktop on a laptop, ChatGPT in a web browser, Claude Code
running in a CI pipeline — all of these can make HTTPS requests. None of them can spawn
a local process on a remote machine. HTTP makes the server a permanent public endpoint.

**Why stdio for extension servers?** Extension servers only need to work from the AI
client on the user's machine. stdio is simpler: no URL, no port, no TLS, no firewall
rules. Claude Desktop launches the process, pipes messages through stdin/stdout, and the
whole thing disappears when the session ends.

AI clients that support remote MCP servers (Claude Desktop, ChatGPT) connect to the core
server by URL. Clients that only support local stdio servers (some versions of Cursor,
VS Code Copilot) can bridge to the core server via `mcp-remote`:

```json
{
  "mcpServers": {
    "open-brain": {
      "command": "npx",
      "args": [
        "mcp-remote",
        "https://YOUR_PROJECT_REF.supabase.co/functions/v1/open-brain-mcp",
        "--header",
        "x-brain-key:${BRAIN_KEY}"
      ],
      "env": {
        "BRAIN_KEY": "your-access-key"
      }
    }
  }
}
```

> Note: no space after the colon in `x-brain-key:${BRAIN_KEY}`. Some clients have a
> bug where spaces inside args strings get mangled.

---

## 4. Authentication

The core server is a public HTTPS endpoint. Anyone who knows the URL can send requests to
it. The `MCP_ACCESS_KEY` mechanism closes this gap.

### How it works

Every incoming request is checked for a valid access key before any tool logic runs. The
key can arrive two ways:

**Option A — URL query parameter** (simplest, works everywhere):

```
https://YOUR_PROJECT_REF.supabase.co/functions/v1/open-brain-mcp?key=a3f8b2c1d4e5...
```

This is what you paste into Claude Desktop's "Remote MCP server URL" field. The key is
embedded in the URL itself, so no extra configuration is needed.

**Option B — HTTP header** (cleaner for programmatic access):

```bash
curl -H "x-brain-key: a3f8b2c1d4e5..." \
  https://YOUR_PROJECT_REF.supabase.co/functions/v1/open-brain-mcp
```

Claude Code uses this form:

```bash
claude mcp add --transport http open-brain \
  https://YOUR_PROJECT_REF.supabase.co/functions/v1/open-brain-mcp \
  --header "x-brain-key: your-access-key"
```

The server checks the query parameter first, then falls back to the header. If neither is
present or the value does not match `MCP_ACCESS_KEY`, the server returns a `401
Unauthorized` response and no tool code runs.

### Generating the key

The key is a 64-character hex string generated once during setup:

```bash
# Mac/Linux
openssl rand -hex 32

# Windows PowerShell
-join ((1..32) | ForEach-Object { '{0:x2}' -f (Get-Random -Maximum 256) })
```

Output looks like: `a3f8b2c1d4e56789abcdef01234567890123456789abcdef0123456789abcdef`

### Storing the key

The key is stored as a Supabase secret so it is available to the Edge Function as an
environment variable:

```bash
supabase secrets set MCP_ACCESS_KEY=your-generated-key-here
```

Supabase secrets are encrypted at rest and injected as environment variables at runtime.
They are never visible in the dashboard after the initial set. The Edge Function reads the
key at startup via `Deno.env.get("MCP_ACCESS_KEY")`.

### What this protects against

The key protects against two things: random internet traffic hitting the public URL, and
access to your stored thoughts if someone discovers your Supabase project ref. The project
ref appears in your URL and is not a secret. The access key is the actual secret.

The key does not protect against someone who has obtained the key — treat it the same as
an API key. If it leaks, rotate it with `supabase secrets set MCP_ACCESS_KEY=new-key`.

---

## 5. The Four Tools

### 5.1 `capture_thought`

The primary write path. Receives a string, runs embedding and metadata extraction in
parallel, inserts one row into the `thoughts` table.

**Input:**

| Parameter | Type   | Required | Description            |
|-----------|--------|----------|------------------------|
| `content` | string | Yes      | The text to store      |

**Data flow:**

```
AI client calls capture_thought({content: "Sarah wants to start consulting"})
  |
  v
[Server receives content string]
  |
  +---> getEmbedding(content)          ---> OpenRouter /embeddings
  |     model: text-embedding-3-small       returns float[1536]
  |
  +---> extractMetadata(content)       ---> OpenRouter /chat/completions
        model: gpt-4o-mini                  returns JSON object
        (runs in parallel with embedding)
  |
  v
[Promise.all resolves — both calls complete]
  |
  v
supabase.from("thoughts").insert({
  content:   "Sarah wants to start consulting",
  embedding: [0.023, -0.041, ...],    // 1536 floats
  metadata:  {
    type: "person_note",
    topics: ["consulting", "career"],
    people: ["Sarah"],
    action_items: []
  }
})
  |
  v
Returns:
{
  "success": true,
  "id": "uuid",
  "content": "Sarah wants to start consulting",
  "metadata": { "type": "person_note", "topics": ["consulting", "career"], ... }
}
```

The embedding is what enables semantic search — it encodes the meaning of the text as a
position in 1536-dimensional vector space. The metadata is what enables filtered queries —
it is structured data the AI can use to narrow results by type, topic, or person.

The `getEmbedding` function calls the OpenRouter embeddings endpoint:

```typescript
async function getEmbedding(text: string): Promise<number[]> {
  const response = await fetch("https://openrouter.ai/api/v1/embeddings", {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${OPENROUTER_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      model: "openai/text-embedding-3-small",
      input: text,
    }),
  });
  const data = await response.json();
  return data.data[0].embedding; // number[] with 1536 elements
}
```

The `extractMetadata` function calls the OpenRouter chat completions endpoint with a
system prompt that defines the output schema:

```typescript
async function extractMetadata(text: string): Promise<Record<string, unknown>> {
  const response = await fetch("https://openrouter.ai/api/v1/chat/completions", {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${OPENROUTER_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      model: "openai/gpt-4o-mini",
      response_format: { type: "json_object" },
      messages: [
        {
          role: "system",
          content: `Extract metadata from the user's captured thought. Return JSON with:
- "people": array of people mentioned
- "action_items": array of implied to-dos
- "topics": array of 1-3 short topic tags
- "type": one of "observation", "task", "idea", "reference", "person_note"
Only extract what's explicitly there.`
        },
        { role: "user", content: text },
      ],
    }),
  });
  const data = await response.json();
  return JSON.parse(data.choices[0].message.content);
}
```

`response_format: { type: "json_object" }` forces `gpt-4o-mini` to return valid JSON.
Without this, the model may wrap its response in prose.

---

### 5.2 `search_thoughts`

The primary read path. Converts a query string into a vector embedding, then calls the
`match_thoughts()` PostgreSQL function to find stored thoughts with similar semantic
meaning.

**Input:**

| Parameter   | Type   | Required | Default | Description                                    |
|-------------|--------|----------|---------|------------------------------------------------|
| `query`     | string | Yes      | —       | The search query (will be embedded)            |
| `threshold` | float  | No       | `0.7`   | Minimum cosine similarity score (0.0–1.0)      |
| `count`     | int    | No       | `10`    | Maximum results to return                      |
| `filter`    | JSONB  | No       | `{}`    | Metadata containment filter                    |

**Data flow:**

```
AI client calls search_thoughts({query: "career changes"})
  |
  v
getEmbedding("career changes")  ---> OpenRouter /embeddings
  returns float[1536] query vector
  |
  v
supabase.rpc("match_thoughts", {
  query_embedding: [0.019, -0.038, ...],
  match_threshold: 0.7,
  match_count: 10,
  filter: {}
})
  |
  v  [PostgreSQL executes match_thoughts() function]
     SELECT t.id, t.content, t.metadata,
            1 - (t.embedding <=> query_embedding) AS similarity,
            t.created_at
     FROM thoughts t
     WHERE 1 - (t.embedding <=> query_embedding) > 0.7
       AND ('{}'::jsonb = '{}'::jsonb OR t.metadata @> '{}')
     ORDER BY t.embedding <=> query_embedding
     LIMIT 10
  |
  v
Returns:
{
  "success": true,
  "count": 2,
  "results": [
    {
      "id": "uuid",
      "content": "Sarah wants to start consulting",
      "metadata": { "type": "person_note", "people": ["Sarah"] },
      "similarity": 0.89,
      "created_at": "2026-03-10T14:22:00Z"
    }
  ]
}
```

The `<=>` operator in PostgreSQL is pgvector's cosine distance operator. It returns a
value between 0 (identical vectors) and 2 (opposite vectors). The expression `1 -
(embedding <=> query_embedding)` converts this to cosine similarity: 1.0 means identical,
0.0 means orthogonal.

The `threshold` parameter filters results before returning them. The default of `0.7`
means only thoughts with at least 70% cosine similarity to the query are returned. Lower
values (e.g., `0.3`) return more results with weaker matches. Higher values (e.g., `0.85`)
return only strong matches.

The `filter` parameter uses PostgreSQL's JSONB containment operator `@>`. To find only
thoughts tagged as tasks:

```json
{ "type": "task" }
```

This translates to `t.metadata @> '{"type": "task"}'` in the SQL, which returns only rows
where the metadata object contains that key-value pair. Both the HNSW vector index and the
GIN metadata index are used in this query.

---

### 5.3 `list_thoughts`

Paginated chronological listing of stored thoughts. No embedding is generated — this is a
plain SQL query ordered by `created_at DESC`.

**Input:**

| Parameter | Type | Required | Default | Description                              |
|-----------|------|----------|---------|------------------------------------------|
| `count`   | int  | No       | `10`    | Number of thoughts to return             |
| `offset`  | int  | No       | `0`     | Number of thoughts to skip (pagination)  |

**Data flow:**

```
AI client calls list_thoughts({count: 5, offset: 0})
  |
  v
supabase
  .from("thoughts")
  .select("id, content, metadata, created_at")
  .order("created_at", { ascending: false })
  .limit(5)
  .range(0, 4)
  |
  v
Returns:
{
  "success": true,
  "count": 5,
  "thoughts": [
    {
      "id": "uuid",
      "content": "Decided to move launch to March 15 ...",
      "metadata": { "type": "observation", "topics": ["launch", "QA"] },
      "created_at": "2026-03-13T09:00:00Z"
    },
    ...
  ]
}
```

Use `offset` for pagination: `list_thoughts({count: 10, offset: 0})` gets the 10 most
recent thoughts, `list_thoughts({count: 10, offset: 10})` gets the next 10.

`list_thoughts` is not a substitute for `search_thoughts`. It returns thoughts in time
order regardless of relevance. Use it for "show me what I captured recently" or "give me a
weekly review of my captures." Use `search_thoughts` for "find what I captured about X."

---

### 5.4 `stats`

Returns an aggregate overview of the `thoughts` table. No input parameters.

**Data flow:**

```
AI client calls stats()
  |
  v
Multiple aggregate queries run against the thoughts table:
  - COUNT(*) for total
  - MIN(created_at) for oldest
  - MAX(created_at) for newest
  - Aggregation on metadata->>'type' for type distribution
  - Aggregation on metadata->'topics' for top topics
  - Aggregation on metadata->'people' for top people
  |
  v
Returns:
{
  "success": true,
  "total_thoughts": 247,
  "oldest": "2025-11-01T08:00:00Z",
  "newest": "2026-03-13T09:00:00Z",
  "top_topics": ["consulting", "product", "family", "hiring"],
  "top_people": ["Sarah", "Marcus", "Jen"],
  "type_distribution": {
    "idea": 42,
    "task": 78,
    "person_note": 55,
    "reference": 31,
    "observation": 41
  }
}
```

`stats` is typically the first tool an AI calls in a new session — it gives the AI a
quick orientation to what is stored before it decides whether to search or browse. It is
also useful for prompts like "give me a summary of what I've been thinking about this
month."

---

## 6. Parallel Processing in `capture_thought`

The two external API calls in `capture_thought` — embedding generation and metadata
extraction — are independent. Neither result depends on the other. Running them
sequentially would waste roughly half the total latency.

The server uses `Promise.all` to run both calls in parallel:

```typescript
const [embedding, metadata] = await Promise.all([
  getEmbedding(content),
  extractMetadata(content),
]);
```

`Promise.all` takes an array of promises and resolves when all of them complete, returning
their results in the same order. If `getEmbedding` takes 200ms and `extractMetadata` takes
300ms:

```
Sequential:    [getEmbedding 200ms] --> [extractMetadata 300ms] --> total: ~500ms
Parallel:      [getEmbedding 200ms]
               [extractMetadata 300ms]                           --> total: ~300ms
```

The total time is determined by the slower of the two calls, not the sum of both. For the
typical latency range of these models (100–400ms each), parallelization cuts capture
latency by 30–50%.

This pattern only works because the two calls are genuinely independent — `extractMetadata`
does not need the embedding result, and `getEmbedding` does not need the metadata result.
Both take the same `content` string as input and produce separate outputs that are then
combined for the database insert.

If `Promise.all` rejects (either call fails), the error propagates to the tool handler,
which returns an error response to the AI client. The partial result (one success, one
failure) is discarded — no partial row is inserted. This is the correct behavior: a row
without an embedding cannot be searched, and a row without metadata is incomplete.

---

## 7. Dependencies

The server's dependencies are declared in
`supabase/functions/open-brain-mcp/deno.json`:

```json
{
  "imports": {
    "@hono/mcp": "npm:@hono/mcp@0.1.1",
    "@modelcontextprotocol/sdk": "npm:@modelcontextprotocol/sdk@1.24.3",
    "hono": "npm:hono@4.9.2",
    "zod": "npm:zod@4.1.13",
    "@supabase/supabase-js": "npm:@supabase/supabase-js@2.47.10"
  }
}
```

This is a Deno import map — the equivalent of `package.json` for Deno projects. Deno does
not use `node_modules`; it resolves imports at runtime from the URLs or npm specifiers
declared here.

**`hono`** — a lightweight TypeScript web framework built for edge runtimes (Deno, Bun,
Cloudflare Workers). The core server uses Hono to handle the HTTP layer: routing, request
parsing, response formatting. Hono is well-suited to Edge Functions because it has minimal
overhead and no Node.js-specific dependencies.

**`@hono/mcp`** — a Hono middleware that implements the MCP HTTP transport spec. It
handles the JSON-RPC framing, the `tools/list` and `tools/call` method routing, and the
Streamable HTTP transport that the MCP spec requires for server-sent events and stateful
sessions. Using `@hono/mcp` means the server does not need to implement the transport
layer manually — it registers tools, and the middleware handles the wire protocol.

**`@modelcontextprotocol/sdk`** — the official TypeScript SDK for MCP. Provides the type
definitions, request/response schemas, and server infrastructure. The extension servers use
this SDK directly with `StdioServerTransport`; the core server uses it via `@hono/mcp`
which wraps the SDK's server interface for HTTP delivery.

**`zod`** — a TypeScript schema validation library. Used to validate and parse incoming
tool arguments at runtime. Zod schemas define the expected shape of each tool's input and
produce typed, validated objects that the handler functions can use safely. If the AI
client sends a malformed argument (wrong type, missing required field), Zod catches it
before the handler runs.

**`@supabase/supabase-js`** — the official Supabase JavaScript client library. Used inside
the Edge Function to communicate with the PostgreSQL database via the PostgREST REST API.
Provides the query builder (`.from()`, `.select()`, `.insert()`, `.rpc()`) and manages the
`Authorization` and `apikey` headers automatically when initialized with the service role
key.

The Deno runtime inside Supabase Edge Functions resolves `npm:` specifiers natively — no
build step, no `npm install`. When the function is deployed, Supabase handles dependency
resolution.

---

## 8. Deployment

Deploying the core server is a single CLI command after writing the source files.

### Prerequisites

Install the Supabase CLI and link your project:

```bash
# Mac with Homebrew
brew install supabase/tap/supabase

# Windows with Scoop
scoop bucket add supabase https://github.com/supabase/scoop-bucket.git
scoop install supabase

# Linux or Mac without Homebrew
npm install -g supabase
```

```bash
supabase login
supabase link --project-ref YOUR_PROJECT_REF
```

### Set secrets

The Edge Function requires two secrets beyond the auto-injected Supabase variables:

```bash
supabase secrets set OPENROUTER_API_KEY=your-openrouter-key-here
supabase secrets set MCP_ACCESS_KEY=your-generated-key-here
```

`SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` are injected automatically into every Edge
Function at runtime — you do not need to set them.

### Create the function scaffold

```bash
supabase functions new open-brain-mcp
```

This creates `supabase/functions/open-brain-mcp/index.ts` with a placeholder. Replace the
contents with the full server code and create the `deno.json` dependency file alongside it.

### Deploy

```bash
supabase functions deploy open-brain-mcp --no-verify-jwt
```

**Why `--no-verify-jwt`?** By default, Supabase Edge Functions require every incoming
request to carry a valid Supabase JWT in the `Authorization` header. This works well for
functions called by browser clients using Supabase Auth. But MCP clients are not browser
clients — they are AI assistants that know nothing about Supabase JWTs. The `--no-verify-jwt`
flag disables the default JWT gate at the platform level. The server then handles its own
authentication using the `MCP_ACCESS_KEY` mechanism described in Section 4.

After deployment, the server is live at:

```
https://YOUR_PROJECT_REF.supabase.co/functions/v1/open-brain-mcp
```

Append `?key=your-access-key` for the connection URL you give to AI clients:

```
https://YOUR_PROJECT_REF.supabase.co/functions/v1/open-brain-mcp?key=your-access-key
```

There is no npm install, no TypeScript compilation step, no build artifact to manage.
Supabase compiles and deploys the Deno TypeScript source directly. Re-deploying after a
code change is the same single command.

---

## 9. How It Connects to Other Components

```
+-----------------------------------------------------------------------+
|                         AI CLIENTS                                    |
|                                                                       |
|  Claude Desktop    ChatGPT (web)    Claude Code    Cursor (mcp-remote)|
+----+-------------+--------+----------+----------+------+--------------+
     |             |        |          |          |
     |             |        |          |          |
     |         HTTP (JSON-RPC over HTTPS)         |
     |             |        |          |          |
     +------+------+--------+----------+----------+
            |
            v
+---------------------------------------------+
|  open-brain-mcp Edge Function               |
|  (Supabase, Deno runtime)                   |
|                                             |
|  1. Authenticate (MCP_ACCESS_KEY check)     |
|  2. Route JSON-RPC method                   |
|  3. Execute tool handler                    |
+----+-----------------------+-----------------+
     |                       |
     | fetch() HTTPS         | @supabase/supabase-js
     | (OpenRouter API)      | (PostgREST HTTPS)
     |                       |
     v                       v
+--------------------+  +-----------------------------------+
|  OpenRouter API    |  |  Supabase PostgreSQL              |
|                    |  |                                   |
|  /v1/embeddings    |  |  thoughts table                   |
|  text-embedding    |  |  - id, content, embedding,        |
|  -3-small          |  |    metadata, created_at           |
|                    |  |                                   |
|  /v1/chat/         |  |  HNSW index (vector search)       |
|  completions       |  |  GIN index (metadata filter)      |
|  gpt-4o-mini       |  |  BTREE index (date ordering)      |
|                    |  |                                   |
+--------------------+  |  match_thoughts() function        |
                        |  (cosine similarity search)       |
                        +-----------------------------------+
```

**The request path for `capture_thought`:**

1. AI client POSTs JSON-RPC to the Edge Function URL with `?key=` or `x-brain-key` header
2. Hono middleware authenticates the key
3. `@hono/mcp` routes the `tools/call` method to the `capture_thought` handler
4. Handler calls `getEmbedding()` and `extractMetadata()` in parallel via `Promise.all`
5. Both calls hit OpenRouter's API over HTTPS; responses return to the Edge Function
6. Handler calls `supabase.from("thoughts").insert(...)` — this hits the PostgREST API
7. PostgreSQL inserts the row; the HNSW and GIN indexes are updated automatically
8. Handler returns a JSON result; Hono serializes it as a JSON-RPC response
9. AI client receives the response and confirms to the user that the thought was saved

**The request path for `search_thoughts`:**

1–3. Same as above (auth + routing)
4. Handler calls `getEmbedding(query)` — one OpenRouter API call
5. Handler calls `supabase.rpc("match_thoughts", {...})` — this calls the PostgreSQL
   function via PostgREST
6. PostgreSQL runs the `match_thoughts()` function, using the HNSW index to efficiently
   find nearest vectors and the GIN index to apply any metadata filter
7. Results return through PostgREST to the Supabase client to the handler
8. Handler serializes results as JSON-RPC response
9. AI client receives the matching thoughts and presents them to the user

**What the Edge Function never does:** It never stores credentials on disk, never keeps
state between requests (Edge Functions are stateless), and never exposes the PostgreSQL
connection string or service role key to the AI client. All secrets stay inside the
Supabase secrets store.

---

## 10. Source File Reference

The core server source lives in the Supabase functions directory:

| File | Purpose |
|------|---------|
| `supabase/functions/open-brain-mcp/index.ts` | Full server implementation |
| `supabase/functions/open-brain-mcp/deno.json` | Dependency import map |

The setup guide at `docs/01-getting-started.md` covers the deployment workflow with
specific line references:

| Lines | Content |
|-------|---------|
| 246–259 | Step 5: generating the `MCP_ACCESS_KEY` |
| 261–265 | Storing the key as a Supabase secret |
| 269–365 | Step 6: full deployment walkthrough |
| 328–337 | The `deno.json` dependency declaration |
| 341 | Reference to the full server source code |
| 346 | The deploy command with `--no-verify-jwt` |
| 349–363 | Constructing the MCP Connection URL |
| 369–443 | Step 7: connecting AI clients (Claude Desktop, ChatGPT, Claude Code, mcp-remote) |

The architecture documents that explain the broader context:

| File | What it covers |
|------|---------------|
| `docs/HOW_OPEN_BRAIN_WORKS.md` | Five-layer architecture, MCP protocol details, transport comparison |
| `docs/ARCHITECTURE.md` | Component map, data flow diagrams, security boundaries |
| `docs/API_REFERENCE.md` | Complete tool schemas, output formats, filter examples |

The `match_thoughts()` PostgreSQL function called by `search_thoughts` is defined in
`docs/01-getting-started.md:164-194` (Step 2, "Create the Search Function"). The `thoughts`
table schema including the three indexes is at `docs/01-getting-started.md:124-157`.
