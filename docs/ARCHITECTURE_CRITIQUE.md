# Open Brain Architecture Critique

An honest technical evaluation of what the system gets right, what adds unnecessary
friction, and how to simplify it for your use case.

**Audience:** Developers comfortable with databases, APIs, and systems design, but not
necessarily familiar with MCP, Supabase, or vector databases. Every concept is explained
from first principles.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [The Architecture As-Is](#2-the-architecture-as-is)
3. [Can You Skip MCP?](#3-can-you-skip-mcp)
4. [What's Genuinely Well-Designed](#4-whats-genuinely-well-designed)
5. [What Could Be Improved](#5-what-could-be-improved)
6. [Recommendations for Different Users](#6-recommendations-for-different-users)

---

## 1. Executive Summary

Open Brain's architecture is sound for what it delivers, but the complexity is unevenly
distributed. The running system is simple: one database table, one SQL function, one API
key. The perceived complexity comes almost entirely from setup friction spread across
multiple browser tabs, command-line tools, and third-party services that have to be
coordinated manually. MCP — the protocol that connects AI clients to Open Brain — adds
genuine value if you use multiple AI clients, but if you use only one, it introduces a
standardization layer you don't need, and native tool use with a plain REST API would be
simpler. The core insight of the system (vector embeddings in PostgreSQL, searched through
a serverless function) is excellent; the surrounding infrastructure choices that made sense
for multi-client reach have calcified into assumptions that not every user needs to inherit.

---

## 2. The Architecture As-Is

### The Five Layers

Open Brain has five identifiable layers, each built on top of the previous one. The diagram
below shows how they connect at runtime.

```
+-----------------------------------------------------------------------------+
|  LAYER 5: AI CLIENTS                                                        |
|                                                                             |
|   Claude Desktop    ChatGPT (web)    Claude Code    Cursor    Windsurf      |
|        |                 |               |             |          |         |
|        |     MCP over HTTPS (core)       |   MCP over stdio (extensions)   |
|        |                 |               |             |          |         |
+--------+-----------------+---------------+-------------+----------+---------+
         |                 |               |
         |    HTTPS POST (JSON-RPC)        |  (extension servers run locally,
         |                 |               |   not shown in this path)
         +--------+--------+
                  |
+------------------v----------------------------------------------------------+
|  LAYER 4: MCP PROTOCOL LAYER                                                |
|                                                                             |
|   open-brain-mcp Edge Function                                              |
|   Deployed on Supabase (Deno runtime, cloud)                                |
|                                                                             |
|   Exposes four tools:                                                       |
|     capture_thought   — write a thought with embedding + metadata           |
|     search_thoughts   — semantic search by vector similarity                |
|     list_thoughts     — chronological browse, no embedding needed           |
|     stats             — aggregate counts, top topics, top people            |
|                                                                             |
+-----+----------------------------------+------------------------------------+
      |                                  |
      | HTTPS (fetch)                    | HTTPS (supabase-js / PostgREST)
      |                                  |
+-----v--------------+     +------------v---------------------------------------+
|  LAYER 3:          |     |  LAYER 1 + 2: DATABASE                             |
|  OPENROUTER        |     |                                                    |
|  AI GATEWAY        |     |  Supabase (managed PostgreSQL hosting)             |
|                    |     |  + pgvector extension                              |
|  text-embedding    |     |                                                    |
|  -3-small          +---->+  thoughts table:                                   |
|  (embeddings)      |     |    id          uuid, primary key                   |
|                    |     |    content     text, the raw thought                |
|  gpt-4o-mini       |     |    embedding   vector(1536), 1536 floats            |
|  (metadata         |     |    metadata    jsonb, structured tags               |
|   extraction)      |     |    created_at  timestamptz                          |
|                    |     |    updated_at  timestamptz                          |
+--------------------+     |                                                    |
                           |  match_thoughts() SQL function:                    |
                           |    takes a query vector + threshold                |
                           |    returns rows ranked by cosine similarity        |
                           |                                                    |
                           |  Indexes:                                          |
                           |    HNSW index on embedding (fast vector search)   |
                           |    GIN index on metadata (fast JSON filtering)     |
                           |    BTREE index on created_at (fast date queries)   |
                           +----------------------------------------------------+
```

### What Each Layer Does

**Layer 1: PostgreSQL + pgvector**

PostgreSQL is a relational database — think Excel with transactions, concurrency, and
SQL. pgvector is an extension that adds one new data type (`vector`) and several new
operators, including `<=>` for cosine distance. Without pgvector, you would need a
separate vector database service (Pinecone, Weaviate, Qdrant) alongside your relational
database. With pgvector, everything lives in one place.

A vector is an array of floating-point numbers that encodes the meaning of a piece of
text as a position in high-dimensional space. Texts with similar meaning end up near each
other in that space. Searching for semantically related content becomes a geometry problem:
find the stored vectors closest to the query vector.

The `thoughts` table is the entire data model. There is no complex schema — one table
handles everything the core system needs.

**Layer 2: Supabase hosting**

Supabase runs PostgreSQL for you and wraps it in:
- A web dashboard for inspecting tables and running SQL
- A REST API (PostgREST) that auto-generates endpoints for every table
- An authentication system (not used in the default Open Brain setup)
- Edge Functions — serverless compute that runs Deno TypeScript close to your database
- A secrets store for environment variables

You could run a self-hosted PostgreSQL instance and get the same database behavior, but
you would lose the managed infrastructure and the Edge Functions hosting.

**Layer 3: OpenRouter AI gateway**

OpenRouter is a unified API that gives you access to many AI models (OpenAI, Anthropic,
Google, others) through a single account and API key. Open Brain uses it for two things:
generating embeddings (via `text-embedding-3-small`) and extracting metadata from thought
content (via `gpt-4o-mini`).

Using OpenRouter rather than OpenAI directly means you only maintain one billing
relationship, and you can swap models by changing a string in the Edge Function code.

**Layer 4: MCP protocol layer**

MCP stands for Model Context Protocol. It is a standardized protocol for AI chatbots to
call external functions. Think of it as a menu at a restaurant: the AI reads the list of
available tools and their descriptions, decides which ones to call, and sends structured
requests. The server handles the calls and returns results.

The core MCP server is deployed as a Supabase Edge Function — a serverless function that
runs on demand. It is a permanent HTTPS endpoint reachable from any device or AI client
that supports remote MCP connections.

The extension servers (household knowledge, home maintenance, family calendar, meal
planning, professional CRM, job hunt) are a different deployment model: they run as
Node.js processes on your local machine, launched by Claude Desktop over stdin/stdout.
They are not cloud-hosted.

**Layer 5: AI clients**

Claude Desktop, ChatGPT, Claude Code, Cursor, and similar tools. These are the user-facing
interfaces. They connect to the MCP server(s) and call tools based on the user's
conversational prompts. The AI decides which tool to call and with what arguments — the
user does not write code.

### Component-by-Component Ratings

| Component | Complexity | Value | Simpler alternative? |
|-----------|-----------|-------|---------------------|
| PostgreSQL + pgvector | Low | Critical | No — this is already the simplest vector DB option |
| Supabase hosting | Low | High | Self-hosted Postgres, but more ops work |
| Embedding pipeline (text-embedding-3-small) | Medium | Critical | No — embeddings are what make semantic search work |
| Metadata extraction (gpt-4o-mini) | Low | Nice-to-have | Could be skipped entirely; semantic search still works via embeddings alone |
| MCP protocol | Medium | High if multi-AI, Low if single-AI | Native function calling + REST API |
| Edge Function deployment | Medium | Medium | Local server, but then you need a machine running 24/7 |
| Extension MCP servers (stdio) | Low each | Domain-specific | Same logic as REST handlers; MCP is just the transport |
| RLS policies | Medium | Only matters for multi-user | Skip entirely for single-user setups |
| Shared MCP server | High | Only matters if sharing data | Skip unless you need household access |

**Complexity ratings explained:**

- **Low**: A junior developer can implement and debug it in under an hour.
- **Medium**: Requires understanding a non-trivial concept (edge functions, JSON-RPC
  protocol, async parallelism) but is well-documented and not inherently tricky.
- **High**: Requires coordinating multiple systems, multiple credential sets, and
  understanding security boundaries across all of them.

**Value ratings explained:**

- **Critical**: The system does not work without it.
- **High**: The system is significantly less useful without it.
- **Medium**: Adds real capability but the core use case works without it.
- **Nice-to-have**: A genuine improvement but not load-bearing.

---

## 3. Can You Skip MCP?

The short answer is yes. For some users, skipping MCP entirely would result in a simpler,
equally functional system. Whether that trade is worth it depends on how many AI clients
you use.

### What MCP Actually Does

MCP (Model Context Protocol) is a standardized way for AI assistants to call external
functions. The word "standardized" is the key: it defines a common format — how tools are
described, how arguments are passed, how results come back — so that any MCP-compatible
AI client can use any MCP-compatible server without custom integration code.

Think of it as a restaurant menu. The kitchen (the MCP server) decides what dishes are
available and writes them on the menu (the tools list). Any customer (any AI client) can
read the menu and order what they need. The kitchen does not need to know in advance which
customer will walk in.

At the wire level, MCP uses JSON-RPC over HTTP. JSON-RPC is a protocol where the client
sends a JSON object containing a method name and parameters, and the server returns a JSON
object with the result. It is not exotic — it is a thin standardization layer on top of
plain HTTPS.

```
What the AI client sends:
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "capture_thought",
    "arguments": { "content": "Sarah wants to start a consulting business" }
  }
}

What the MCP server sends back:
{
  "jsonrpc": "2.0",
  "result": {
    "content": [
      { "type": "text", "text": "{ \"success\": true, \"id\": \"uuid-here\" }" }
    ]
  }
}
```

The MCP server also handles a `tools/list` call that returns the full menu of available
tools with their names, descriptions, and input schemas. AI clients call this at the start
of a session to discover what is available.

### What MCP Replaces

Before MCP became a de facto standard in 2025, each AI platform had its own format:
- OpenAI had "function calling" — a JSON schema format in the API request
- Anthropic had "tool use" — a slightly different JSON schema format
- Google had "function declarations" — yet another format

If you wanted to give five AI clients the ability to search your notes, you maintained
five slightly different tool definitions for the same underlying logic.

MCP standardizes the tool definition format and the call/response protocol. Build one
MCP server and every MCP-compatible client can use it without any per-client integration
work.

**The catch:** if you only use one AI client, you don't get the "any client" benefit.
Standardization has value when there is diversity. For a single-client setup, MCP adds
a protocol layer (and its associated deployment complexity) in exchange for zero benefit.

### Three Alternatives to MCP

---

#### Alternative A: Plain Supabase REST API

Supabase auto-generates a REST API for every table in your database. No code required —
the endpoints appear as soon as your table exists. The `thoughts` table immediately has:

```
GET  https://YOUR_PROJECT_REF.supabase.co/rest/v1/thoughts
POST https://YOUR_PROJECT_REF.supabase.co/rest/v1/thoughts
PATCH https://YOUR_PROJECT_REF.supabase.co/rest/v1/thoughts?id=eq.UUID
```

You can configure your AI client to call these endpoints directly using its native tool
use or function calling mechanism.

```
+------------------+                         +------------------+
|                  |                         |                  |
|   Claude         |  native tool use        |   Supabase       |
|   (or ChatGPT)   | ----------------------> |   REST API       |
|                  |  POST /rest/v1/thoughts  |   (PostgREST)    |
|                  |  GET  /rest/v1/thoughts  |                  |
|                  | <----------------------  |                  |
|                  |  JSON row(s)             |                  |
+------------------+                         +--------+---------+
                                                      |
                                                      | SQL
                                                      v
                                             +------------------+
                                             |  PostgreSQL      |
                                             |  thoughts table  |
                                             +------------------+
```

No MCP server in the middle. Zero middleware.

The AI's system prompt would describe the available endpoints, and native tool use handles
the HTTP calls. Here is what that system prompt configuration looks like:

```
You have access to a personal knowledge database via REST API.
Base URL: https://YOUR_PROJECT_REF.supabase.co/rest/v1
API key header: apikey: YOUR_ANON_KEY
Content-Type: application/json

To save a thought:
  POST /thoughts
  Body: { "content": "your text here", "metadata": {} }

To retrieve recent thoughts:
  GET /thoughts?order=created_at.desc&limit=10

To search by metadata tag:
  GET /thoughts?metadata->>'type'=eq.task&order=created_at.desc
```

**What this gives you:** Direct database access with no deployment, no MCP server to
maintain, and no middleware.

**The critical limitation:** The Supabase REST API has no semantic search capability.
You can filter by exact metadata values (`type=task`) or sort by date, but you cannot
ask "find thoughts related to career planning." For that you need vector similarity search,
which requires embeddings, which requires the embedding pipeline — and that pipeline lives
in the Edge Function that you are trying to skip.

The REST-only approach works well for: capturing thoughts (just a POST), browsing recent
thoughts (GET with ordering), filtering by known tags. It does not work for the most
valuable feature of Open Brain, which is finding semantically related content across your
entire thought history.

**Verdict:** Appropriate if you do not need semantic search and want the absolute minimum
setup. If semantic search is why you're building this, you need the Edge Function, and
you're back to choosing between MCP and a plain REST wrapper around that function.

---

#### Alternative B: Plain HTTP API (Keep the Pipeline, Drop MCP)

This is the most practical alternative for single-client users. The same Edge Function
logic — embedding generation, metadata extraction, vector search — but exposed as plain
REST endpoints instead of through the MCP protocol.

You keep all the processing capability. You drop the JSON-RPC layer.

```
+------------------+          HTTPS POST           +------------------+
|                  |                               |                  |
|   Claude         |  POST /api/capture            |   Edge Function  |
|   (or ChatGPT)   |  { "content": "..." }         |   (Supabase)     |
|   native         | ----------------------------> |                  |
|   tool use       |                               |                  |
|                  |  POST /api/search             |    +----------+  |
|                  |  { "query": "..." }           |    | embed    |  |
|                  |                               |    | metadata |  |
|                  | <---------------------------- |    | insert / |  |
|                  |  JSON results                 |    | search   |  |
|                  |                               |    +----+-----+  |
+------------------+                               +------+---+-------+
                                                          |   |
                                             OpenRouter   |   | Supabase
                                             API calls    |   | DB calls
                                                          v   v
                                                   +-------+ +----------+
                                                   |OpenR  | |PostgreSQL|
                                                   +-------+ +----------+
```

The four endpoints would be:

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/capture` | POST | Store a thought (embed + extract metadata + insert) |
| `/api/search` | POST | Semantic search (embed query + vector similarity) |
| `/api/recent` | GET | List recent thoughts, ordered by date |
| `/api/stats` | GET | Aggregate counts and top topics |

You would then configure your AI client's native tool use to call these endpoints. Here
is how the function definitions would look for both Claude and ChatGPT:

**For Claude (Anthropic tool use format):**

```json
{
  "tools": [
    {
      "name": "capture_thought",
      "description": "Save a thought to your personal knowledge base. Use this when the user wants to remember something.",
      "input_schema": {
        "type": "object",
        "properties": {
          "content": {
            "type": "string",
            "description": "The thought to save"
          }
        },
        "required": ["content"]
      }
    },
    {
      "name": "search_thoughts",
      "description": "Search your personal knowledge base by meaning, not just keywords.",
      "input_schema": {
        "type": "object",
        "properties": {
          "query": {
            "type": "string",
            "description": "What to search for"
          },
          "threshold": {
            "type": "number",
            "description": "Similarity threshold 0.0-1.0, default 0.7"
          }
        },
        "required": ["query"]
      }
    }
  ]
}
```

**For ChatGPT (OpenAI function calling format):**

```json
{
  "functions": [
    {
      "name": "capture_thought",
      "description": "Save a thought to the personal knowledge base.",
      "parameters": {
        "type": "object",
        "properties": {
          "content": {
            "type": "string",
            "description": "The thought to save"
          }
        },
        "required": ["content"]
      }
    },
    {
      "name": "search_thoughts",
      "description": "Search the personal knowledge base semantically.",
      "parameters": {
        "type": "object",
        "properties": {
          "query": { "type": "string" },
          "threshold": { "type": "number" }
        },
        "required": ["query"]
      }
    }
  ]
}
```

**What you gain:** Same full processing pipeline. No MCP server deployment, no MCP SDK
dependencies, no JSON-RPC framing.

**What you lose:** Each AI client needs its own tool/function definition. If you use
three AI clients, you maintain three copies of the same definition in three different
formats. MCP solves this; the plain REST approach does not. This is the exact trade-off
MCP was designed to eliminate — it matters more as client count grows.

**Verdict:** The right choice for a single-client user who wants the full semantic search
capability. The development effort to build the REST wrapper is minimal — it is the
same function logic with `app.post("/capture", handler)` instead of
`mcpServer.tool("capture_thought", schema, handler)`. The decision tree below helps you
decide.

---

#### Alternative C: Direct Code Execution (Coding Tools Only)

Claude Code, Cursor, and similar coding-focused AI tools can run code directly in your
terminal or project environment. They do not need a server at all — the AI can just
execute database calls inline.

```
+------------------+
|                  |
|   Claude Code    |   (no network call to any server)
|   or Cursor      |
|                  |
|   AI runs:       |
|   const result   |
|   = await        |
|   supabase.rpc(  |
|    'match_       |
|     thoughts',   |
|     {...}        |
|   )              |
|                  |
+--------+---------+
         |
         | supabase-js (direct DB client)
         v
+------------------+
|  PostgreSQL      |
|  (Supabase)      |
+------------------+
```

The AI assistant writes and executes `supabase.rpc()` calls or raw SQL queries directly.
No protocol. No server. No deployment. The AI is both the client and the runtime.

**What you gain:** Absolute minimum complexity. Zero infrastructure beyond the database
itself.

**What you lose:** This only works in tools that execute code on your behalf. Claude
Desktop, ChatGPT, and conversational AI interfaces cannot execute arbitrary code — they
can only call tools that are pre-registered. This path is closed to them. It is also
entirely local: if you switch to a different machine or a different tool, you lose the
capability unless you reconfigure it there.

**Verdict:** Appropriate for developers using Claude Code or Cursor as their primary
interface, who do not need the system to work from Claude Desktop or ChatGPT. Eliminates
the most complex part of the setup (Edge Function deployment) entirely.

---

### Decision Matrix

Use this tree to pick the right architecture for your situation:

```
START HERE
    |
    v
Do you use more than one AI client?
(Claude Desktop AND ChatGPT, or Claude Desktop AND Cursor, etc.)
    |
    +--YES--> MCP is worth the setup cost.
    |         Build the Edge Function with MCP.
    |         Follow docs/01-getting-started.md as written.
    |
    +--NO---> Do you use only a coding tool (Claude Code, Cursor)?
                  |
                  +--YES--> Consider Alternative C: direct code execution.
                  |         No server needed at all.
                  |         Skip docs/01-getting-started.md Steps 5-7.
                  |
                  +--NO---> Do you need semantic search
                            (find related thoughts by meaning)?
                                |
                                +--YES--> Alternative B: plain HTTP API.
                                |         Build the Edge Function, expose REST
                                |         endpoints instead of MCP tools.
                                |         One-time tool definition per client.
                                |
                                +--NO---> Alternative A: Supabase REST API.
                                          No Edge Function needed at all.
                                          Configure native tool use with
                                          direct POST/GET to /rest/v1/thoughts.

ADDITIONAL QUESTIONS:

Are you the only user?
    |
    +--YES--> Skip RLS complexity.
    |         The single "service role full access" policy from setup is enough.
    |         Do NOT build the shared MCP server (primitives/shared-mcp).
    |
    +--NO---> Multiple users sharing the same database?
                  |
                  +--YES--> Implement RLS. See primitives/rls/README.md.
                            Consider the shared MCP server pattern.
                            See primitives/shared-mcp/README.md.
                  |
                  +--NO---> Each user has their own Supabase project.
                            No sharing infrastructure needed.
```

---

## 4. What's Genuinely Well-Designed

### pgvector in PostgreSQL

The decision to use pgvector inside PostgreSQL rather than a dedicated vector database
is the best architectural choice in the system. Dedicated vector databases (Pinecone,
Weaviate, Qdrant, Chroma) are separate services you run, pay for, and synchronize with
your relational data. They exist because, historically, relational databases had no vector
support. pgvector eliminates the need for them entirely.

The result is one data store, one connection string, one backup, one place to look when
something goes wrong. The HNSW index (Hierarchical Navigable Small World) that pgvector
uses is the same class of algorithm that dedicated vector databases use — the performance
is comparable for the scale Open Brain operates at.

### Parallel Embedding and Metadata Extraction

The `capture_thought` handler runs its two external API calls — embedding generation and
metadata extraction — in parallel using `Promise.all`:

```typescript
const [embedding, metadata] = await Promise.all([
  getEmbedding(content),
  extractMetadata(content),
]);
```

This is the correct implementation. The two calls are genuinely independent: neither needs
the other's result. Running them sequentially would double the latency for no reason.
For typical API response times of 100–400ms each, parallelization cuts capture latency
by 30–50%. This is a small detail that most developers would get wrong on the first pass.

### Edge Functions (Zero Infrastructure)

Deploying the MCP server as a Supabase Edge Function means there is no server to provision,
no VM to keep running, no auto-scaling to configure, and no monthly instance cost. The
function runs on demand, billed per invocation (effectively free at personal use scale),
and is automatically available globally.

This is the right call for a system at this scale. Running a persistent Node.js server
for a personal memory system would mean either accepting availability risk (the server is
down when your laptop is closed) or paying for a always-on cloud VM.

### The `thoughts` Table Schema

The schema is minimal and correct:

```sql
create table thoughts (
  id uuid default gen_random_uuid() primary key,
  content text not null,
  embedding vector(1536),
  metadata jsonb default '{}'::jsonb,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);
```

`content` is the raw text — the source of truth. `embedding` is derived from `content`
via the external API. `metadata` is a JSONB column that holds arbitrary structured tags
without requiring schema migrations as the metadata schema evolves. The three indexes
cover the three query patterns: vector similarity search (HNSW), metadata filtering
(GIN), and date ordering (BTREE).

Nothing in this schema is wrong. It is the right shape for what the system does.

### The Extension Pattern

Each extension (household knowledge, home maintenance, family calendar, meal planning,
professional CRM, job hunt) is an independent unit: its own tables, its own MCP server,
its own tools. Extensions do not depend on each other. Adding one does not affect the
others.

This is the right model for optional capabilities. Users can install exactly the
extensions they need and ignore the rest. The extension code is also simple —
extensions do no embedding and no LLM calls, they are pure CRUD over PostgreSQL via
the Supabase JS client. They are harder to get wrong than the core server.

---

## 5. What Could Be Improved

### Setup Requires Too Many Browser Tabs

Building the core system involves:
1. Creating a Supabase account and project (supabase.com)
2. Running three SQL commands in the Supabase SQL editor
3. Finding and copying the Project URL and Secret key from Supabase settings
4. Creating an OpenRouter account (openrouter.ai), adding credits, generating a key
5. Generating a random access key in the terminal
6. Installing the Supabase CLI
7. Running CLI commands to link, set secrets, create and deploy the function
8. Configuring the AI client (Claude Desktop settings, or ChatGPT settings)

That is five different surfaces (browser tabs / apps / terminal), seven credentials to
track, and roughly 30–45 minutes if everything works on the first try. None of these
individual steps is hard, but the coordination overhead is real, and any step that fails
requires understanding which of the five surfaces to debug.

The credential tracker section in `docs/01-getting-started.md` acknowledges this
honestly and tells you to copy it into a text editor immediately. That is good advice.
The underlying problem — too many services to coordinate — is not easily fixed without
changing the architecture.

An automated setup script (`setup.sh` or a web-based wizard) that handled the Supabase
CLI commands, secret generation, and function deployment in sequence would eliminate most
of this friction. The script could output the MCP Connection URL as its final step. This
does not exist yet.

### Metadata Extraction Is Presented as Essential

The setup guide and system architecture describe the two parallel API calls — embedding
and metadata extraction — as the capture pipeline, creating the impression that both are
required. They are not equally essential.

The embedding is what powers semantic search. Without it, you cannot find semantically
related content. The embedding call is genuinely critical.

The metadata extraction calls `gpt-4o-mini` to extract structured tags: `type`, `topics`,
`people`, `action_items`. These tags enable filtered queries (`find all tasks`, `find
mentions of Sarah`). That is useful — but semantic search already finds relevant content
without any filters. You can successfully use Open Brain for months without ever using
the metadata filter.

Skipping metadata extraction would:
- Reduce capture latency (one API call instead of two in parallel, which is actually
  still the same wall-clock time because they run in parallel — but the metadata call
  does still consume tokens and add cost)
- Eliminate the dependency on `gpt-4o-mini` entirely
- Leave the `metadata` column empty (which is fine — it defaults to `{}`)
- Cause no degradation in semantic search quality

The setup guide should make this trade-off explicit. Metadata extraction is a
nice-to-have, not a requirement.

### RLS and Shared Servers Add Complexity Most Users Don't Need

Row Level Security (RLS) is a PostgreSQL mechanism that controls which rows a given
database connection can see. It is valuable for multi-user systems where different users
share one database and must not see each other's data.

The default Open Brain setup — one user, one Supabase project, one service role key —
does not benefit from RLS. The service role key bypasses RLS entirely (this is by design;
the Supabase docs and the RLS primitive documentation both state this). Enabling RLS and
creating a "service role full access" policy in the setup guide is technically correct
but creates cognitive overhead: the user reads about a security feature and then
immediately learns that their key bypasses it.

The result is mixed messaging: the setup enables RLS, the MCP server uses the key that
bypasses it, and most users are single-user setups that would be fine with no RLS at all.

For single-user setups, the security model is simpler: the `MCP_ACCESS_KEY` gates
access to the Edge Function, and the Edge Function is the only thing that talks to the
database. No authentication layer inside the database is needed.

The shared MCP server pattern (`primitives/shared-mcp/README.md`) is well-documented,
but it requires building a separate Node.js server, compiling TypeScript, and deploying
it to another person's machine. The prerequisites section lists "Understanding of database
roles and permissions" — which puts it firmly in developer territory, not the target
audience of the main setup guide. This is fine as long as it is clearly positioned as an
advanced optional feature. In the current docs, it is a primitive alongside RLS, which
implies it is foundational. It is not foundational for most users.

### No Automated Setup Script

Every step in `docs/01-getting-started.md` is manual copy-paste. For a system that is
explicitly targeting people with "zero coding experience," this is a significant friction
point. The document is well-written and genuinely achieves its goal, but an idempotent
setup script that handles the CLI steps would be materially better for users who are
not comfortable with the terminal.

The Supabase CLI supports non-interactive commands. Most of the setup steps after
installing the CLI could be scripted:

```bash
supabase login
supabase link --project-ref $PROJECT_REF
supabase secrets set OPENROUTER_API_KEY=$OPENROUTER_KEY
supabase secrets set MCP_ACCESS_KEY=$(openssl rand -hex 32)
supabase functions deploy open-brain-mcp --no-verify-jwt
echo "Done. Connect to: https://$PROJECT_REF.supabase.co/functions/v1/open-brain-mcp?key=$MCP_ACCESS_KEY"
```

The SQL commands (creating the table, creating the function, enabling RLS) are harder
to script because they require the Supabase Dashboard or direct PostgreSQL access, but
the CLI does support running SQL migrations.

### Extensions Use the Service Role Key, Which Bypasses RLS

This is the sharpest inconsistency in the system. The setup guide creates RLS policies.
The extension servers connect to the database with `SUPABASE_SERVICE_ROLE_KEY`, which
bypasses all RLS policies. The RLS primitive documentation (`primitives/rls/README.md`)
explicitly states this is intentional: the service role has superuser privileges and
ignores policies.

In practice, for a single-user personal system, this is fine — the service role key is
the intended credential for server-to-database communication. The inconsistency is that
the system introduces RLS concepts and then immediately short-circuits them. A user who
reads the RLS documentation expecting it to protect their data will be confused to
discover that all current MCP server implementations bypass it.

The honest documentation would be: "RLS matters if you ever give another user direct
database access via the `anon` key. If all access goes through the MCP server with the
service role key, RLS does not currently do anything. We include it for completeness
and because future extensions may use authenticated user tokens."

---

## 6. Recommendations for Different Users

### "I just want a working memory system"

You do not need extensions, RLS complexity, shared servers, or the Slack/Discord
integrations. The core system — one table, one Edge Function, one AI client — is
complete and useful on its own.

**What to build:**
- `thoughts` table and `match_thoughts()` SQL function (Step 2 of `docs/01-getting-started.md`)
- OpenRouter API key for embeddings and metadata extraction (Step 4)
- Core MCP Edge Function deployed (Steps 5–6)
- Single AI client connected (Step 7)

**What to skip:**
- All six extensions
- RLS policies beyond the basic "service role full access" setup
- Shared MCP server (`primitives/shared-mcp/`)
- Slack/Discord capture integrations

**Time:** 30 minutes if you follow the guide and nothing goes wrong.

**Monthly cost:** $0 for infrastructure; roughly $0.10–$0.30 in API costs at 20
thoughts/day.

**The one thing to get right:** Semantic search is only as good as the questions you
ask. The embedding-based search works by meaning, not keywords. "What do I know about
Sarah's career situation?" will find a thought you captured as "Sarah mentioned she's
thinking about leaving her job" even though those strings share no words. Learn to phrase
queries as questions or descriptions of what you're trying to remember, not as keyword
searches.

---

### "I want the full system for my household"

If you have a partner or family members who would benefit from shared meal planning,
shopping lists, or family calendar access, the full system is worth building.

**What to build:**
- Core system as above
- Extensions 1–4 (household knowledge, home maintenance, family calendar, meal planning)
  — these cover the household use cases
- RLS policies if multiple people will share the same Supabase project
- Shared MCP server for household members who should have limited access

**What to skip:**
- Extension 5 (professional CRM) and Extension 6 (job hunt pipeline) — unless you are
  actively managing professional relationships and job searching
- The Slack integration — the MCP capture from AI clients is simpler unless you already
  live in Slack

**Time:** 2–3 hours for the full setup including extensions.

**Monthly cost:** $0 for infrastructure; slightly higher API costs as capture volume
increases across household members.

**The complexity checkpoint:** Setting up the shared MCP server requires Node.js installed
on the other person's machine, TypeScript compilation, and careful credential scoping.
Before committing to this, make sure the other person will actually use it. The complexity
is only justified by active usage.

---

### "I'm a developer who wants to understand and extend"

Build everything, then read the code. The architecture is small enough that you can hold
all of it in your head after a few hours of study.

**What to build:**
- The entire system, including all six extensions
- Read and understand the Edge Function source before deploying it
- Understand the `match_thoughts()` SQL function and the HNSW index
- Build a small extension of your own to internalize the pattern

**Where to study:**
- `docs/HOW_OPEN_BRAIN_WORKS.md` — the most detailed technical explanation of how each
  component works, written for developers coming from C/Java backgrounds
- `docs/ARCHITECTURE.md` — component diagrams and data flow
- `docs/mcp-servers/00-core-mcp-server.md` — deep dive into the MCP Edge Function
- `docs/API_REFERENCE.md` — full tool schemas and database schema
- `docs/STUDY_PLAN.md` — a 13-phase learning path from web fundamentals through
  capstone projects, if you want to build general web development fluency alongside
  this specific system

**Time:** Weeks to achieve full fluency; months to complete the full study plan.

**Monthly cost:** $0.

**The thing worth understanding deeply:** The HNSW index and how vector similarity search
works. This is what makes semantic search possible. Once you understand that a vector is
a position in 1536-dimensional space and that `<=>` is cosine distance between two
positions, the rest of the system's search behavior becomes predictable and debuggable.
The pgvector documentation and the `match_thoughts()` function in
`docs/01-getting-started.md:164-194` together give you everything you need.

---

*This document critiques the Open Brain architecture as of the date this file was
committed. The system is actively developed; some limitations described here may be
addressed in later versions.*
