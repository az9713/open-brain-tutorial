# Open Brain: Architecture Reference

This document describes the complete technical architecture of the Open Brain system. It is written for developers who may be familiar with C, C++, or Java but are new to web development, cloud services, and the AI tooling ecosystem. Every concept is explained from first principles.

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Detailed Component Map](#2-detailed-component-map)
3. [The MCP Protocol](#3-the-mcp-protocol)
4. [Data Flow: Capturing a Thought](#4-data-flow-capturing-a-thought)
5. [Data Flow: Semantic Search](#5-data-flow-semantic-search)
6. [Extension Architecture](#6-extension-architecture)
7. [Database Schema](#7-database-schema)
8. [Security Boundaries](#8-security-boundaries)
9. [CI/CD Pipeline](#9-cicd-pipeline)
10. [Technology Reference](#10-technology-reference)

---

## 1. System Overview

Open Brain is a three-tier system. Think of it like a classic client-server application, except the "client" is an AI assistant and the "server" speaks a specialized protocol called MCP.

> **Alternative path:** Terminal-based AI tools (Claude Code, Codex, Gemini CLI) can bypass Tier 2 entirely using the [`ob` CLI tool](../resources/ob-cli/), which calls the Supabase REST API and OpenRouter directly. See [CLI-Direct Approach](CLI_DIRECT_APPROACH.md) for that architecture. Both paths use the same Tier 3 infrastructure and the same `thoughts` table.

```
┌─────────────────────────────────────────────────────────────────────┐
│                         TIER 1: AI CLIENTS                          │
│                                                                     │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────┐  ┌──────────┐  │
│   │ Claude       │  │   ChatGPT    │  │  Cursor  │  │Windsurf  │  │
│   │ Desktop      │  │  (Plus/Pro)  │  │          │  │          │  │
│   └──────┬───────┘  └──────┬───────┘  └────┬─────┘  └────┬─────┘  │
│          │                 │               │              │         │
│          └─────────────────┴───────────────┴──────────────┘         │
│                                    │                                 │
│                              MCP Protocol                            │
│                         (HTTP or stdio transport)                    │
└────────────────────────────────────┬────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      TIER 2: MCP SERVERS                            │
│                                                                     │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │              open-brain-mcp (Edge Function)                 │   │
│   │          Deployed on Supabase · Deno runtime                │   │
│   │                                                             │   │
│   │   Tools: capture_thought · search_thoughts ·                │   │
│   │          list_thoughts · stats                              │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│   ┌─────────────────┐  ┌──────────────────┐  ┌─────────────────┐   │
│   │  household-     │  │  home-           │  │  meal-planning  │   │
│   │  knowledge      │  │  maintenance     │  │  (+ shared      │   │
│   │  MCP server     │  │  MCP server      │  │   server)       │   │
│   │  (TypeScript)   │  │  (TypeScript)    │  │  (TypeScript)   │   │
│   └─────────────────┘  └──────────────────┘  └─────────────────┘   │
│                                                                     │
│   ┌─────────────────┐  ┌──────────────────┐  ┌─────────────────┐   │
│   │  family-        │  │  professional-   │  │  job-hunt       │   │
│   │  calendar       │  │  crm             │  │  MCP server     │   │
│   │  MCP server     │  │  MCP server      │  │  (TypeScript)   │   │
│   │  (TypeScript)   │  │  (TypeScript)    │  │                 │   │
│   └─────────────────┘  └──────────────────┘  └─────────────────┘   │
└──────────────────────────────────┬──────────────────────────────────┘
                                   │
                                   │ Supabase JS client
                                   │ (HTTPS + service role key)
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   TIER 3: SUPABASE INFRASTRUCTURE                   │
│                                                                     │
│   ┌──────────────────────────────────────────────────────────────┐  │
│   │                    PostgreSQL 14+                            │  │
│   │                                                              │  │
│   │   thoughts table     Extension tables     auth.users         │  │
│   │   (pgvector)         (per extension)      (Supabase Auth)    │  │
│   │                                                              │  │
│   │   HNSW index         GIN index            RLS policies       │  │
│   │   (vector search)    (JSONB metadata)     (per table)        │  │
│   └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│   ┌────────────────────────┐   ┌────────────────────────────────┐   │
│   │   Edge Functions       │   │        Supabase Secrets        │   │
│   │   (Deno runtime)       │   │                                │   │
│   │                        │   │   OPENROUTER_API_KEY           │   │
│   │   open-brain-mcp       │   │   MCP_ACCESS_KEY               │   │
│   │   ingest-thought       │   │   SLACK_BOT_TOKEN              │   │
│   │   (slack webhook)      │   │   SUPABASE_SERVICE_ROLE_KEY    │   │
│   └────────────────────────┘   └────────────────────────────────┘   │
│                                                                     │
│   ┌──────────────────────────────────────────────────────────────┐  │
│   │               OpenRouter AI Gateway                          │  │
│   │                                                              │  │
│   │   text-embedding-3-small  →  1536-dim vectors               │  │
│   │   gpt-4o-mini             →  metadata extraction            │  │
│   └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### Background: What Each Tier Does

**Tier 1 (AI Clients)** are tools like Claude Desktop or ChatGPT running on your computer or in your browser. They are "smart" in the sense that they can understand natural language, but they have no persistent memory between sessions. Open Brain gives them that memory.

**Tier 2 (MCP Servers)** implement the Model Context Protocol — a standard interface that lets any compatible AI client call functions on external systems. Think of MCP tools like RPC (Remote Procedure Calls): the AI calls a named function with typed arguments and gets a typed result back. The MCP servers here are either Edge Functions (serverless functions running on Supabase's infrastructure) or TypeScript processes running locally on your machine.

**Tier 3 (Supabase)** is a managed cloud platform built on top of open-source PostgreSQL. It provides the actual database, serverless compute (Edge Functions), authentication, and secret management. The `pgvector` extension adds support for storing and querying high-dimensional vectors — which is what makes semantic search possible.

---

## 2. Detailed Component Map

This diagram shows how every component in the repository relates to one another, including the external services.

```
 EXTERNAL SERVICES
 ─────────────────────────────────────────────────────────────────────
 OpenRouter (openrouter.ai)
 │
 ├── /embeddings endpoint
 │     Model: openai/text-embedding-3-small
 │     Input: plain text string
 │     Output: float[1536]  ◄── a 1536-element array of floating-point
 │                               numbers that encodes meaning
 │
 └── /chat/completions endpoint
       Model: openai/gpt-4o-mini
       Input: system prompt + user text
       Output: JSON  {people, action_items, dates_mentioned, topics, type}


 YOUR SUPABASE PROJECT (https://[ref].supabase.co)
 ─────────────────────────────────────────────────────────────────────
 │
 ├── PostgreSQL database
 │   │
 │   ├── public.thoughts          ← core table, never modified by extensions
 │   │     id            UUID
 │   │     content       TEXT
 │   │     embedding     VECTOR(1536)
 │   │     metadata      JSONB
 │   │     created_at    TIMESTAMPTZ
 │   │     updated_at    TIMESTAMPTZ
 │   │
 │   ├── Indexes on thoughts
 │   │     HNSW on embedding (vector_cosine_ops)  ← fast approximate nearest neighbor
 │   │     GIN  on metadata                       ← fast JSONB containment queries
 │   │     BTREE on created_at DESC               ← fast date-range scans
 │   │
 │   ├── public.match_thoughts()  ← PostgreSQL function for cosine similarity search
 │   │
 │   ├── auth.users               ← Supabase built-in authentication table
 │   │
 │   └── Extension tables (added by opt-in, see Section 6)
 │         household_items, household_vendors
 │         maintenance_tasks, maintenance_logs
 │         family_members, activities, important_dates
 │         recipes, meal_plans, shopping_lists
 │         professional_contacts, contact_interactions, opportunities
 │         companies, job_postings, applications, interviews, job_contacts
 │
 ├── Edge Functions  (Deno runtime, serverless)
 │   │
 │   ├── open-brain-mcp           ← primary MCP server (HTTP endpoint)
 │   │     URL: /functions/v1/open-brain-mcp
 │   │     Auth: ?key= query param  OR  x-brain-key header
 │   │     Tools exposed:
 │   │       capture_thought(content)
 │   │       search_thoughts(query, threshold?, filter?)
 │   │       list_thoughts(limit?, offset?, filter?)
 │   │       stats()
 │   │
 │   └── ingest-thought           ← Slack webhook receiver
 │         URL: /functions/v1/ingest-thought
 │         Receives Slack events → embeds → stores in thoughts
 │
 └── Supabase Secrets (environment variables for Edge Functions)
       OPENROUTER_API_KEY
       MCP_ACCESS_KEY
       SLACK_BOT_TOKEN  (optional)
       SLACK_CAPTURE_CHANNEL  (optional)


 LOCAL MCP SERVERS  (TypeScript, run on your machine via Node.js)
 ─────────────────────────────────────────────────────────────────────
 Configured in AI client config (e.g. claude_desktop_config.json)
 Communicate via stdio (standard input/output), not HTTP

 extensions/household-knowledge/index.ts
   Server name:  household-knowledge
   Transport:    StdioServerTransport
   Tools:        add_household_item, search_household_items,
                 get_item_details, add_vendor, list_vendors
   Env required: SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY

 extensions/home-maintenance/index.ts
   Server name:  home-maintenance
   Transport:    StdioServerTransport
   Tools:        add_task, complete_task, list_due_tasks,
                 get_maintenance_history, ...

 extensions/family-calendar/index.ts
   Server name:  family-calendar
   Transport:    StdioServerTransport
   Tools:        add_member, add_activity, list_activities,
                 add_important_date, ...

 extensions/meal-planning/index.ts          (owner server)
 extensions/meal-planning/shared-server.ts  (read-limited household server)
   Server names: meal-planning / meal-planning-shared
   The shared server uses SUPABASE_HOUSEHOLD_KEY (scoped credentials)

 extensions/professional-crm/index.ts
   Server name:  professional-crm
   Tools:        add_contact, log_interaction, add_opportunity,
                 search_contacts, get_contact_details, ...

 extensions/job-hunt/index.ts
   Server name:  job-hunt
   Tools:        add_company, add_job_posting, create_application,
                 schedule_interview, get_pipeline_summary, ...


 CAPTURE INTEGRATIONS
 ─────────────────────────────────────────────────────────────────────
 integrations/slack-capture/
   Pattern: Slack message → Slack Events API → Edge Function webhook
            → OpenRouter (embed + extract) → thoughts table

 integrations/discord-capture/
   Pattern: Discord message → Discord bot → similar pipeline
```

---

## 3. The MCP Protocol

If you have experience with RPC frameworks (gRPC, Thrift, COM), MCP will feel familiar. If not, here is the essential model.

### What MCP Is

MCP (Model Context Protocol) is an open standard that defines how AI applications communicate with external tools. It separates concerns cleanly:

- The **AI client** (Claude, ChatGPT) handles natural language understanding and decides when to call a tool
- The **MCP server** handles domain logic and data access
- The **protocol** standardizes the interface between them

### Communication Flow

```
AI CLIENT                                    MCP SERVER
──────────                                   ──────────

  1. User says: "Remember that the meeting
     moved to Thursday"

  2. AI decides this is a capture action

  3. AI sends ListTools request:
  ┌─────────────────────────────┐
  │  {                          │
  │    "jsonrpc": "2.0",        │ ────────────────────────►
  │    "method": "tools/list",  │
  │    "id": 1                  │
  │  }                          │
  └─────────────────────────────┘
                                              4. Server responds:
                                 ◄──────────  [{name: "capture_thought",
                                               description: "...",
                                               inputSchema: {...}}]

  5. AI sends CallTool request:
  ┌──────────────────────────────────────┐
  │  {                                   │
  │    "jsonrpc": "2.0",                 │ ────────────────►
  │    "method": "tools/call",           │
  │    "params": {                       │
  │      "name": "capture_thought",      │
  │      "arguments": {                  │
  │        "content": "Meeting moved     │
  │                    to Thursday"      │
  │      }                               │
  │    },                                │
  │    "id": 2                           │
  │  }                                   │
  └──────────────────────────────────────┘
                                              6. Server stores thought,
                                                 returns confirmation:
                                 ◄──────────  {content: [{type: "text",
                                               text: "Captured as task..."}]}

  7. AI presents result to user:
     "Got it — saved as a task.
      Topics: scheduling. Action item:
      attend Thursday meeting."
```

### Two Transport Modes

Open Brain MCP servers use two different transport mechanisms:

```
TRANSPORT 1: HTTP (Streamable HTTP)
Used by: open-brain-mcp Edge Function
Suitable for: remote deployment, multiple clients, no local Node.js needed

  AI CLIENT                    INTERNET                  SUPABASE
  ─────────                    ────────                  ───────
  Claude Desktop ──HTTPS──► [edge: supabase.co] ──► open-brain-mcp
                              ?key=abc123...


TRANSPORT 2: stdio (Standard Input/Output)
Used by: all extension MCP servers (TypeScript)
Suitable for: local development, single-user scenarios

  AI CLIENT              LOCAL MACHINE
  ─────────              ─────────────
  Claude Desktop
       │
       │  spawns process
       ▼
  node dist/index.js     ← MCP server running as a child process
       │     ▲
       │     │
    stdout  stdin        ← JSON-RPC messages exchanged over pipes
       │     │
       ▼     │
  (JSON-RPC messages)
```

The stdio model works exactly like piping data between programs in a Unix shell. The AI client spawns the MCP server as a child process and communicates with it by writing JSON to stdin and reading JSON from stdout. No network sockets are involved.

### How Extension Servers Register Tools

Every extension MCP server follows the same registration pattern. From `extensions/household-knowledge/index.ts`:

```typescript
// Create the server instance
const server = new Server(
  { name: "household-knowledge", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

// Handle "what tools do you have?" requests
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: TOOLS,   // array of tool definitions with names and JSON schemas
}));

// Handle "call this tool" requests
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;
  switch (name) {
    case "add_household_item":
      return { content: [{ type: "text", text: await handleAddHouseholdItem(args) }] };
    // ... other tools
  }
});

// Connect to stdio transport and start listening
const transport = new StdioServerTransport();
await server.connect(transport);
```

---

## 4. Data Flow: Capturing a Thought

This is the most important write path in the system. Whether you use Claude Desktop, ChatGPT, or Slack, capturing a thought always ends in the same database row.

### Via MCP (AI Clients)

```
                                    ┌─────────────────────────────────────────┐
                                    │         open-brain-mcp                  │
                                    │         Edge Function (Deno)            │
 User: "Remember: Sarah is          │                                         │
  leaving her job"                  │  1. Receive MCP tool call               │
       │                            │     capture_thought("Sarah is           │
       ▼                            │      leaving her job")                  │
 AI Client calls                    │                                         │
 capture_thought ──────────────────►│  2. Validate access key                 │
                  HTTPS POST        │     from ?key= param or x-brain-key     │
                  /functions/v1/    │     header                              │
                  open-brain-mcp    │                                         │
                                    │  3. Spawn two async operations          │
                                    │     IN PARALLEL:                        │
                                    │                                         │
                                    │  ┌─────────────────┐ ┌───────────────┐ │
                                    │  │  Generate        │ │  Extract      │ │
                                    │  │  Embedding       │ │  Metadata     │ │
                                    │  │                  │ │               │ │
                                    │  │  POST            │ │  POST         │ │
                                    │  │  openrouter.ai   │ │  openrouter.ai│ │
                                    │  │  /embeddings     │ │  /chat/...    │ │
                                    │  │                  │ │               │ │
                                    │  │  model:          │ │  model:       │ │
                                    │  │  text-embedding- │ │  gpt-4o-mini  │ │
                                    │  │  3-small         │ │               │ │
                                    │  │                  │ │  Prompt asks  │ │
                                    │  │  Returns:        │ │  for JSON:    │ │
                                    │  │  float[1536]     │ │  {people,     │ │
                                    │  │                  │ │   topics,     │ │
                                    │  │                  │ │   action_items│ │
                                    │  │                  │ │   type}       │ │
                                    │  └────────┬─────────┘ └──────┬────────┘ │
                                    │           │                   │          │
                                    │           └─────────┬─────────┘          │
                                    │                     │ await Promise.all  │
                                    │                     ▼                    │
                                    │  4. Single INSERT into thoughts:         │
                                    │     {                                    │
                                    │       content: "Sarah is leaving...",    │
                                    │       embedding: [0.021, -0.003, ...],   │  1536 values
                                    │       metadata: {                        │
                                    │         people: ["Sarah"],               │
                                    │         topics: ["career", "work"],      │
                                    │         type: "person_note",             │
                                    │         action_items: ["check in"],      │
                                    │         source: "claude-desktop"         │
                                    │       }                                  │
                                    │     }                                    │
                                    │                                          │
                                    │  5. Return confirmation to AI client     │
                                    └──────────────────────────────────────────┘
```

### Via Slack Integration

```
 Slack Channel                Edge Function               OpenRouter        Supabase
 ─────────────                ─────────────               ──────────        ───────
 User types message
       │
       │ Slack Events API
       │ POST /functions/v1/ingest-thought
       ▼
 ingest-thought ──────────────────────────────────────────────────────►
                                                                        │
                                    Validate channel ID               │
                                    (reject other channels)           │
                                            │                          │
                                    Promise.all([                     │
                                      getEmbedding(text),            ├──► /embeddings
                                      extractMetadata(text)          ├──► /chat/completions
                                    ])                                │
                                            │                          │
                                    supabase.from("thoughts")         │
                                      .insert({...})  ──────────────────────────────────►
                                                                                  INSERT
                                            │
                                    replyInSlack(channel, ts,
                                      "Captured as person_note...")
                                            │
                                    ◄───────
 Threaded reply appears in Slack
```

Both paths produce identical rows in the `thoughts` table. The only difference is the `metadata.source` field (`"claude-desktop"` vs `"slack"`).

---

## 5. Data Flow: Semantic Search

Semantic search is fundamentally different from keyword search. In keyword search, you look for exact matches. In semantic search, you look for meaning matches. The query "career changes" finds the thought "Sarah is leaving her job" even though those strings share zero words.

### How Semantic Search Works (Conceptual)

Every piece of text can be converted into a point in a very high-dimensional space (1536 dimensions in this case). Texts with similar meanings map to nearby points. The search function finds the stored thoughts whose points are closest to the query's point.

This distance is measured using **cosine similarity**: two vectors are similar if they point in roughly the same direction, regardless of their magnitude. The formula used is `1 - (embedding <=> query_embedding)` where `<=>` is the pgvector cosine distance operator.

### The Search Data Flow

```
 User asks: "What did I note about career changes?"
       │
       ▼
 AI Client calls search_thoughts(query="career changes")
       │
       │ HTTPS POST /functions/v1/open-brain-mcp
       ▼
 ┌──────────────────────────────────────────────────────────────────┐
 │                   open-brain-mcp Edge Function                   │
 │                                                                  │
 │  1. Validate access key                                          │
 │                                                                  │
 │  2. Generate query embedding:                                    │
 │     POST openrouter.ai/embeddings                                │
 │     { model: "text-embedding-3-small",                          │
 │       input: "career changes" }                                  │
 │     → returns float[1536]  (the query vector)                   │
 │                                                                  │
 │  3. Execute match_thoughts() in PostgreSQL:                      │
 │                                                                  │
 │  ┌──────────────────────────────────────────────────────────┐   │
 │  │  SELECT                                                   │   │
 │  │    id, content, metadata, created_at,                     │   │
 │  │    1 - (embedding <=> $query_vector) AS similarity        │   │
 │  │  FROM thoughts                                            │   │
 │  │  WHERE                                                    │   │
 │  │    1 - (embedding <=> $query_vector) > 0.7  ← threshold  │   │
 │  │    AND (filter = '{}' OR metadata @> filter)              │   │
 │  │  ORDER BY                                                 │   │
 │  │    embedding <=> $query_vector  ← ascending distance      │   │
 │  │  LIMIT 10;                                                │   │
 │  └──────────────────────────────────────────────────────────┘   │
 │                                                                  │
 │     PostgreSQL uses the HNSW index to find the k nearest        │
 │     neighbors without scanning every row. This is why           │
 │     search stays fast even with thousands of thoughts.          │
 │                                                                  │
 │  4. Return ranked results to AI client:                          │
 │     [                                                            │
 │       { content: "Sarah is leaving her job...",                  │
 │         similarity: 0.87,                                        │
 │         metadata: { people: ["Sarah"], type: "person_note" } }, │
 │       { content: "Thinking about switching to consulting...",    │
 │         similarity: 0.79, ... },                                 │
 │       ...                                                        │
 │     ]                                                            │
 │                                                                  │
 └──────────────────────────────────────────────────────────────────┘
       │
       ▼
 AI synthesizes the results and responds to the user:
 "You captured a note about Sarah considering leaving her job
  to start a consulting business..."
```

### The HNSW Index

The `HNSW` (Hierarchical Navigable Small World) index on the `embedding` column is critical for performance. Without it, every search would require computing the distance between the query vector and every row in the table — an O(n) scan. With it, PostgreSQL can find the nearest neighbors in O(log n) time using a graph-based data structure.

The index is created in `docs/01-getting-started.md` with:

```sql
CREATE INDEX ON thoughts USING hnsw (embedding vector_cosine_ops);
```

The `vector_cosine_ops` operator class tells PostgreSQL which distance metric the index is optimized for. This must match the distance operator used in queries (`<=>`).

---

## 6. Extension Architecture

Extensions are optional add-ons that add specialized structured data storage and MCP tools on top of the core `thoughts` system. They form a progressive learning path — each one builds on skills introduced by the previous ones.

### Extension Learning Path

```
 ┌─────────────────────────────────────────────────────────────────────────┐
 │                         EXTENSION LEARNING PATH                         │
 │                                                                         │
 │  Ext 1           Ext 2           Ext 3           Ext 4                  │
 │  Household       Home            Family          Meal Planning           │
 │  Knowledge       Maintenance     Calendar                               │
 │  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────────────────┐ │
 │  │ Basic    │   │ Triggers │   │ Multi-   │   │ RLS + Shared MCP     │ │
 │  │ CRUD     │   │ (auto    │   │ entity   │   │ Server pattern       │ │
 │  │ 2 tables │   │  update) │   │ linking  │   │ 3 tables             │ │
 │  │ indexes  │   │ 2 tables │   │ 3 tables │   │ household roles      │ │
 │  │ RLS      │   │ RLS      │   │          │   │                      │ │
 │  └────┬─────┘   └────┬─────┘   └────┬─────┘   └──────────┬───────────┘ │
 │       │              │              │                     │             │
 │       ▼              ▼              ▼                     ▼             │
 │  Concepts:      Concepts:      Concepts:            Concepts:           │
 │  - JSONB        - DB triggers  - FK relationships   - RLS policies      │
 │  - text search  - auto-        - NULL FK (optional  - scoped API keys   │
 │  - user_id RLS    scheduling     relationship)      - shared-server.ts  │
 │                                                     - household JWT     │
 │                                                                         │
 │  Ext 5                          Ext 6                                   │
 │  Professional CRM               Job Hunt Pipeline                       │
 │  ┌─────────────────────────┐   ┌─────────────────────────────────────┐  │
 │  │ 3 tables                │   │ 5 tables                            │  │
 │  │ professional_contacts   │   │ companies                           │  │
 │  │ contact_interactions    │   │ job_postings                        │  │
 │  │ opportunities           │   │ applications                        │  │
 │  │                         │   │ interviews                          │  │
 │  │ Concepts:               │   │ job_contacts                        │  │
 │  │ - cascade triggers      │   │                                     │  │
 │  │ - auto last_contacted   │   │ Concepts:                           │  │
 │  │ - stage tracking        │   │ - cross-extension references        │  │
 │  │ - RLS                   │   │ - soft FK (app-managed)             │  │
 │  └─────────────────────────┘   │ - pipeline state machines           │  │
 │                                │ - RLS                               │  │
 │                                └─────────────────────────────────────┘  │
 └─────────────────────────────────────────────────────────────────────────┘
```

### How an Extension Plugs In

Every extension is self-contained in its own directory and follows this structure:

```
extensions/household-knowledge/
  ├── schema.sql     PostgreSQL DDL: CREATE TABLE, CREATE INDEX,
  │                  ALTER TABLE ... ENABLE ROW LEVEL SECURITY,
  │                  CREATE POLICY, CREATE TRIGGER
  │
  ├── index.ts       TypeScript MCP server:
  │                  - imports @modelcontextprotocol/sdk
  │                  - imports @supabase/supabase-js
  │                  - defines TOOLS array (tool names + JSON schemas)
  │                  - registers ListToolsRequestSchema handler
  │                  - registers CallToolRequestSchema handler (switch/case)
  │                  - connects StdioServerTransport
  │
  ├── package.json   Node.js manifest:
  │                  dependencies: @modelcontextprotocol/sdk, @supabase/supabase-js
  │                  scripts: { build: "tsc", start: "node dist/index.js" }
  │
  ├── tsconfig.json  TypeScript compiler configuration
  ├── metadata.json  Structured metadata for the CI/CD review system
  └── README.md      User-facing setup guide with prerequisites + steps
```

### Relationship to the Core System

Extensions do NOT depend on the `thoughts` table. They create their own tables with their own schemas. The connection is conceptual, not technical: extensions store structured data that the `thoughts` table stores as free-form text with vector embeddings.

```
 thoughts table                  Extension tables
 (semantic, unstructured)        (relational, structured)
 ──────────────────────          ───────────────────────────
 "Sarah is considering           professional_contacts:
  leaving her job to              { name: "Sarah",
  start a consulting               company: "Acme",
  business"                        follow_up_date: "2026-04-01" }
       │                                    │
       │  Both record                       │
       │  information about Sarah,          │
       │  but in different ways             │
       │                                    │
       │  You can use BOTH simultaneously  │
       └────────────────────────────────────┘
       The AI can search thoughts for
       context AND query CRM contacts
       for structured data
```

---

## 7. Database Schema

This section shows the complete schema relationships using an ERD (Entity-Relationship Diagram) style. Each box represents a table; arrows represent foreign key relationships; the crow's-foot notation indicates cardinality.

### Core Schema

```
 ┌────────────────────────────────────────────────────────────────────┐
 │  auth.users  (Supabase managed — do not modify)                    │
 │  ─────────────────────────────────────────────                     │
 │  id           UUID  PRIMARY KEY                                    │
 │  email        TEXT                                                  │
 │  ...          (other Supabase auth fields)                         │
 └──────────────────────────────┬─────────────────────────────────────┘
                                │  1
                                │  Referenced by ALL tables below
                                │  via user_id UUID REFERENCES auth.users(id)
                                │  ON DELETE CASCADE
                                │
                                ▼  many

 ┌────────────────────────────────────────────────────────────────────┐
 │  public.thoughts  (core — immutable structure)                     │
 │  ─────────────────────────────────────────────                     │
 │  id           UUID         PK  DEFAULT gen_random_uuid()           │
 │  content      TEXT         NOT NULL                                 │
 │  embedding    VECTOR(1536) ← 1536-float array from OpenAI          │
 │  metadata     JSONB        DEFAULT '{}'                            │
 │  created_at   TIMESTAMPTZ  DEFAULT now()                           │
 │  updated_at   TIMESTAMPTZ  DEFAULT now()  (auto-updated by trigger)│
 │                                                                    │
 │  Indexes:                                                           │
 │    HNSW  on embedding (vector_cosine_ops)                          │
 │    GIN   on metadata                                               │
 │    BTREE on created_at DESC                                        │
 │                                                                    │
 │  RLS: service_role only (no user-scoped access)                    │
 └────────────────────────────────────────────────────────────────────┘

 Functions:
 ┌────────────────────────────────────────────────────────────────────┐
 │  match_thoughts(query_embedding, match_threshold, match_count,     │
 │                 filter)                                            │
 │  Returns: TABLE(id, content, metadata, similarity, created_at)    │
 │  Logic:   cosine similarity scan with HNSW index + optional JSONB │
 │           metadata filter                                          │
 └────────────────────────────────────────────────────────────────────┘
```

### Extension 1: Household Knowledge

```
 household_items                      household_vendors
 ───────────────────────────────      ────────────────────────────────
 id          UUID  PK                 id           UUID  PK
 user_id     UUID  FK→auth.users      user_id      UUID  FK→auth.users
 name        TEXT  NOT NULL           name         TEXT  NOT NULL
 category    TEXT                     service_type TEXT
 location    TEXT                     phone        TEXT
 details     JSONB  DEFAULT '{}'      email        TEXT
 notes       TEXT                     website      TEXT
 created_at  TIMESTAMPTZ              notes        TEXT
 updated_at  TIMESTAMPTZ              rating       INT  CHECK(1..5)
                                      last_used    DATE
  Indexes:                            created_at   TIMESTAMPTZ
    (user_id, category)
                                       Indexes:
  RLS: user_id = auth.uid()             (user_id, service_type)

                                       RLS: user_id = auth.uid()
```

### Extension 2: Home Maintenance

```
 maintenance_tasks                    maintenance_logs
 ───────────────────────────────      ─────────────────────────────────
 id             UUID  PK              id            UUID  PK
 user_id        UUID  FK→auth.users   task_id       UUID  FK→tasks.id
 name           TEXT  NOT NULL        user_id       UUID  FK→auth.users
 category       TEXT                  completed_at  TIMESTAMPTZ
 frequency_days INT   (null=one-time) performed_by  TEXT
 last_completed TIMESTAMPTZ           cost          DECIMAL(10,2)
 next_due       TIMESTAMPTZ           notes         TEXT
 priority       TEXT  CHECK(...)      next_action   TEXT
 notes          TEXT
 created_at     TIMESTAMPTZ                          1
 updated_at     TIMESTAMPTZ          ◄───────────── many

  Trigger: update_task_after_log
  When a maintenance_log row is inserted, the trigger automatically
  updates maintenance_tasks.last_completed and recalculates next_due
  as: completed_at + frequency_days DAYS
```

### Extension 3: Family Calendar

```
 family_members ───────────────────────────────────────────────────────
 id                UUID  PK
 user_id           UUID  FK→auth.users
 name              TEXT  NOT NULL
 relationship      TEXT  (self, spouse, child, parent)
 birth_date        DATE
 notes             TEXT
 created_at        TIMESTAMPTZ
       │
       │  1
       │         0..1 (nullable FK — null means whole family)
       ▼  many
 activities                           important_dates
 ─────────────────────────────────    ────────────────────────────────
 id                UUID  PK           id               UUID  PK
 user_id           UUID  FK           user_id          UUID  FK
 family_member_id  UUID  FK (null ok) family_member_id UUID  FK (null ok)
 title             TEXT  NOT NULL     title            TEXT  NOT NULL
 activity_type     TEXT               date_value       DATE  NOT NULL
 day_of_week       TEXT               recurring_yearly BOOL
 start_time        TIME               reminder_days_before INT
 end_time          TIME               notes            TEXT
 start_date        DATE               created_at       TIMESTAMPTZ
 end_date          DATE
 location          TEXT
 notes             TEXT
 created_at        TIMESTAMPTZ
```

### Extension 4: Meal Planning

```
 recipes ──────────────────────────────────────────────────────────────
 id                UUID  PK
 user_id           UUID  FK→auth.users
 name              TEXT  NOT NULL
 cuisine           TEXT
 prep_time_minutes INT
 cook_time_minutes INT
 servings          INT
 ingredients       JSONB  (array of {name, quantity, unit})
 instructions      JSONB  (array of step strings)
 tags              TEXT[]
 rating            INT  CHECK(1..5)
 notes             TEXT
 created_at / updated_at  TIMESTAMPTZ
       │
       │ 1                    ┌─── RLS: user-scoped + household_member role
       │                      │    (dual policy for sharing)
       ▼ many                 │
 meal_plans                   │    shopping_lists
 ─────────────────────────────┘    ────────────────────────────────────
 id                UUID  PK         id          UUID  PK
 user_id           UUID  FK         user_id     UUID  FK→auth.users
 week_start        DATE  NOT NULL   week_start  DATE  NOT NULL
 day_of_week       TEXT  NOT NULL   items       JSONB  (array of
 meal_type         TEXT  NOT NULL                {name, qty, unit,
 recipe_id         UUID  FK→recipes               purchased, recipe_id})
 custom_meal       TEXT             notes       TEXT
 servings          INT              created_at / updated_at
 notes             TEXT
 created_at        TIMESTAMPTZ

 Note: meal_plans and shopping_lists both use dual RLS policies:
   Policy 1: user_id = auth.uid()  (owner access)
   Policy 2: auth.jwt()->>'role' = 'household_member'  (shared access)
```

### Extension 5: Professional CRM

```
 professional_contacts ────────────────────────────────────────────────
 id              UUID  PK
 user_id         UUID  FK→auth.users
 name            TEXT  NOT NULL
 company         TEXT
 title           TEXT
 email / phone / linkedin_url  TEXT
 how_we_met      TEXT
 tags            TEXT[]
 notes           TEXT
 last_contacted  TIMESTAMPTZ  (auto-updated by trigger on interaction insert)
 follow_up_date  DATE
 created_at / updated_at  TIMESTAMPTZ
       │
       │ 1
       │           Trigger: update_last_contacted
       │           AFTER INSERT on contact_interactions,
       │           sets professional_contacts.last_contacted = NEW.occurred_at
       ▼ many
 contact_interactions                 opportunities
 ──────────────────────────────────   ─────────────────────────────────────
 id               UUID  PK            id                UUID  PK
 contact_id       UUID  FK→contacts   user_id           UUID  FK→auth.users
 user_id          UUID  FK            contact_id        UUID  FK→contacts
 interaction_type TEXT  NOT NULL      title             TEXT  NOT NULL
   CHECK(meeting|email|call|...)      description       TEXT
 occurred_at      TIMESTAMPTZ         stage             TEXT
 summary          TEXT  NOT NULL        CHECK(identified|in_conversation|
 follow_up_needed BOOL                       proposal|negotiation|won|lost)
 follow_up_notes  TEXT                value             DECIMAL(12,2)
 created_at       TIMESTAMPTZ         expected_close_date DATE
                                      notes             TEXT
                                      created_at / updated_at
```

### Extension 6: Job Hunt Pipeline

```
 companies ────────────────────────────────────────────────────────────
 id               UUID  PK
 user_id          UUID  FK→auth.users
 name             TEXT  NOT NULL
 industry         TEXT
 website          TEXT
 size             TEXT  CHECK(startup|mid-market|enterprise)
 location         TEXT
 remote_policy    TEXT  CHECK(remote|hybrid|onsite)
 notes            TEXT
 glassdoor_rating DECIMAL(2,1)
 created_at / updated_at  TIMESTAMPTZ
       │
       │ 1 (company has many job postings)
       ▼ many
 job_postings ─────────────────────────────────────────────────────────
 id               UUID  PK
 company_id       UUID  FK→companies.id  ON DELETE CASCADE
 user_id          UUID  FK→auth.users
 title            TEXT  NOT NULL
 url              TEXT
 salary_min/max   INT
 salary_currency  TEXT
 requirements     TEXT[]
 nice_to_haves    TEXT[]
 notes            TEXT
 source           TEXT  CHECK(linkedin|company-site|referral|...)
 posted_date / closing_date  DATE
 created_at       TIMESTAMPTZ
       │
       │ 1 (posting has many applications)
       ▼ many
 applications ─────────────────────────────────────────────────────────
 id                UUID  PK
 job_posting_id    UUID  FK→job_postings.id  ON DELETE CASCADE
 user_id           UUID  FK→auth.users
 status            TEXT  DEFAULT 'applied'
   CHECK(draft|applied|screening|interviewing|offer|accepted|rejected|withdrawn)
 applied_date / response_date  DATE
 resume_version    TEXT
 cover_letter_notes TEXT
 referral_contact  TEXT
 notes             TEXT
 created_at / updated_at  TIMESTAMPTZ
       │
       │ 1 (application has many interviews)
       ▼ many
 interviews ───────────────────────────────────────────────────────────
 id                UUID  PK
 application_id    UUID  FK→applications.id  ON DELETE CASCADE
 user_id           UUID  FK→auth.users
 interview_type    TEXT
   CHECK(phone_screen|technical|behavioral|system_design|hiring_manager|team|final)
 scheduled_at      TIMESTAMPTZ
 duration_minutes  INT
 interviewer_name / title  TEXT
 status            TEXT  CHECK(scheduled|completed|cancelled|no_show)
 notes / feedback  TEXT
 rating            INT  CHECK(1..5)
 created_at        TIMESTAMPTZ

 job_contacts ─────────────────────────────────────────────────────────
 id                          UUID  PK
 user_id                     UUID  FK→auth.users
 company_id                  UUID  FK→companies.id  ON DELETE SET NULL
 name / title / email / phone / linkedin_url  TEXT
 role_in_process             TEXT
   CHECK(recruiter|hiring_manager|referral|interviewer|other)
 professional_crm_contact_id UUID  ← soft FK to Extension 5
                                     (no DB constraint — application-managed)
 notes                       TEXT
 last_contacted              TIMESTAMPTZ
 created_at                  TIMESTAMPTZ
```

The `professional_crm_contact_id` field in `job_contacts` is a notable architectural decision. It references the `professional_contacts` table from Extension 5, but there is no database-level foreign key constraint. This is called a **soft foreign key** or **application-managed reference**: the application code is responsible for maintaining referential integrity. The comment in `extensions/job-hunt/schema.sql` explains this explicitly:

```sql
professional_crm_contact_id UUID,
-- FK to Extension 5's professional_contacts table
-- (not enforced by DB, managed by application)
```

This design avoids making Extension 6 dependent on Extension 5 at the database level, so users can install Job Hunt without Professional CRM.

---

## 8. Security Boundaries

Security in Open Brain is enforced at multiple layers. A failure at any outer layer still cannot bypass the inner layers.

```
 ┌─────────────────────────────────────────────────────────────────────┐
 │  LAYER 1: NETWORK                                                   │
 │                                                                     │
 │  All traffic is HTTPS. No unencrypted connections to Supabase.     │
 │  The Supabase project URL is semi-public but Supabase requires     │
 │  authenticated connections for all operations.                      │
 │                                                                     │
 │  ┌─────────────────────────────────────────────────────────────┐   │
 │  │  LAYER 2: MCP ACCESS KEY                                    │   │
 │  │                                                             │   │
 │  │  The open-brain-mcp Edge Function requires a valid key      │   │
 │  │  on every request. Without this key, the function returns   │   │
 │  │  401 Unauthorized before doing any database work.           │   │
 │  │                                                             │   │
 │  │  Key can be provided two ways:                              │   │
 │  │    ?key=abc123   (URL query parameter)                      │   │
 │  │    x-brain-key: abc123   (HTTP header)                      │   │
 │  │                                                             │   │
 │  │  The key is stored in Supabase Secrets (MCP_ACCESS_KEY)     │   │
 │  │  and never appears in source code.                          │   │
 │  │                                                             │   │
 │  │  ┌─────────────────────────────────────────────────────┐   │   │
 │  │  │  LAYER 3: SERVICE ROLE KEY                          │   │   │
 │  │  │                                                     │   │   │
 │  │  │  MCP servers (both Edge Functions and local         │   │   │
 │  │  │  TypeScript servers) connect to Supabase using      │   │   │
 │  │  │  the service_role key. This key:                    │   │   │
 │  │  │    - Is never committed to source code (CI checks)  │   │   │
 │  │  │    - Is stored in environment variables / Secrets   │   │   │
 │  │  │    - Bypasses Row Level Security (by design)        │   │   │
 │  │  │    - Grants full database access                    │   │   │
 │  │  │                                                     │   │   │
 │  │  │  IMPORTANT: MCP servers use service_role because    │   │   │
 │  │  │  the MCP layer is the access control boundary.      │   │   │
 │  │  │  The tool definitions define what operations are    │   │   │
 │  │  │  permitted — not the database credentials.          │   │   │
 │  │  │                                                     │   │   │
 │  │  │  ┌─────────────────────────────────────────────┐   │   │   │
 │  │  │  │  LAYER 4: ROW LEVEL SECURITY (RLS)          │   │   │   │
 │  │  │  │                                             │   │   │   │
 │  │  │  │  All extension tables have RLS enabled.     │   │   │   │
 │  │  │  │  When non-service-role connections are      │   │   │   │
 │  │  │  │  used (e.g. direct Supabase API calls),     │   │   │   │
 │  │  │  │  RLS policies enforce:                      │   │   │   │
 │  │  │  │                                             │   │   │   │
 │  │  │  │  Pattern 1: User-scoped                     │   │   │   │
 │  │  │  │    USING (auth.uid() = user_id)             │   │   │   │
 │  │  │  │    Each user sees only their own rows.      │   │   │   │
 │  │  │  │                                             │   │   │   │
 │  │  │  │  Pattern 2: Household-scoped (meal-planning)│   │   │   │
 │  │  │  │    USING (auth.uid() = user_id             │   │   │   │
 │  │  │  │           OR jwt->>'role' = 'household_    │   │   │   │
 │  │  │  │                            member')        │   │   │   │
 │  │  │  │    Household members can read shared data. │   │   │   │
 │  │  │  │                                             │   │   │   │
 │  │  │  │  Pattern 3: Public + private                │   │   │   │
 │  │  │  │    USING (visibility = 'public'             │   │   │   │
 │  │  │  │           OR auth.uid() = user_id)          │   │   │   │
 │  │  │  │                                             │   │   │   │
 │  │  │  │  See: primitives/rls/README.md              │   │   │   │
 │  │  │  └─────────────────────────────────────────────┘   │   │   │
 │  │  └─────────────────────────────────────────────────────┘   │   │
 │  └─────────────────────────────────────────────────────────────┘   │
 └─────────────────────────────────────────────────────────────────────┘


 SHARED MCP SERVER: 3-LAYER ISOLATION
 ─────────────────────────────────────────────────────────────────────
 When using the shared-mcp pattern (primitives/shared-mcp/README.md),
 a third-party user (e.g. a spouse) gets a separate MCP server with
 its own credentials and limited tool set:

 ┌─────────────────────────────────────────────────────────────────────┐
 │                                                                     │
 │  spouse's Claude Desktop                                            │
 │       │                                                             │
 │       │ stdio                                                       │
 │       ▼                                                             │
 │  meal-planning-shared-server  (extensions/meal-planning/            │
 │                                 shared-server.ts)                  │
 │       │                                                             │
 │       │ Uses SUPABASE_HOUSEHOLD_KEY (not the main service_role key) │
 │       │                                                             │
 │       ├── Layer A: Scoped credentials                               │
 │       │     A separate API key with limited table grants            │
 │       │     Revoking this key doesn't affect the owner's access    │
 │       │                                                             │
 │       ├── Layer B: Limited tools exposed                            │
 │       │     view_meal_plan, view_recipes, update_shopping_list      │
 │       │     NOT EXPOSED: add_recipe, delete_recipe, any thoughts    │
 │       │                  access, any CRM access                     │
 │       │                                                             │
 │       └── Layer C: Read/write permissions per operation             │
 │             meal_plans:     SELECT only                             │
 │             recipes:        SELECT only                             │
 │             shopping_lists: SELECT + UPDATE (no DELETE/INSERT)      │
 │                                                                     │
 └─────────────────────────────────────────────────────────────────────┘


 CI/CD SECURITY ENFORCEMENT
 ─────────────────────────────────────────────────────────────────────
 The automated review system (.github/workflows/ob1-review.yml)
 enforces security at contribution time:

  Rule 4 (No credentials):
    Scans all changed files for patterns:
      sk-[a-zA-Z0-9]{20,}      (OpenAI key pattern)
      AKIA[0-9A-Z]{16}          (AWS key pattern)
      ghp_[a-zA-Z0-9]{36}      (GitHub PAT pattern)
      xoxb-[0-9]                (Slack bot token pattern)
      SUPABASE_SERVICE_ROLE_KEY=ey... (Supabase key with actual value)
    Fails the PR if any match is found.

  Rule 5 (SQL safety):
    Scans .sql files for:
      DROP TABLE / DROP DATABASE / TRUNCATE  → blocked
      DELETE FROM without WHERE clause       → blocked
      ALTER TABLE thoughts DROP/ALTER COLUMN → blocked
                                               (only ADD COLUMN allowed)
```

---

## 9. CI/CD Pipeline

Every pull request to the `main` branch runs through an automated review system before a human maintainer can approve it. This is implemented as a GitHub Actions workflow in `.github/workflows/ob1-review.yml`.

```
 CONTRIBUTOR                  GITHUB                    ob1-review.yml
 ───────────                  ──────                    ──────────────
 git push branch
       │
       │ opens PR
       ▼
 PR: [recipes] Email import ──────────────────────────►
                                  on: pull_request
                                  types: [opened,
                                          synchronize,
                                          reopened]
                                  branches: [main]
                                          │
                                          ▼
                                  Step 1: Checkout PR code
                                  (fetch-depth: 0 for full history)
                                          │
                                          ▼
                                  Step 2: Get changed files
                                  git diff --name-only origin/main...HEAD
                                  → identify contribution directories
                                    (e.g. "recipes/email-import")
                                          │
                                          ▼
                                  Step 3: Run 13 checks in bash:

 ┌─────────────────────────────────────────────────────────────────────┐
 │  Check  1: Folder structure                                         │
 │    All changed files must be under:                                 │
 │    recipes|schemas|dashboards|integrations|primitives|extensions|   │
 │    docs|resources|.github                                           │
 │                                                                     │
 │  Check  2: Required files                                           │
 │    Every contribution folder must have README.md + metadata.json   │
 │                                                                     │
 │  Check  3: Metadata validity                                        │
 │    metadata.json must be valid JSON with required fields:           │
 │    name, description, category, version, estimated_time,           │
 │    author.name, requires.open_brain=true, tags[1+],                │
 │    difficulty (beginner|intermediate|advanced)                      │
 │                                                                     │
 │  Check  4: No credentials                                           │
 │    Scan for API key patterns in non-markdown, non-json files       │
 │    Fail if .env file contains non-placeholder values               │
 │                                                                     │
 │  Check  5: SQL safety                                               │
 │    No DROP TABLE / DROP DATABASE / TRUNCATE                        │
 │    No DELETE FROM without WHERE                                     │
 │    No ALTER TABLE thoughts DROP/ALTER COLUMN                       │
 │                                                                     │
 │  Check  6: Category artifacts                                       │
 │    recipes:      need .sql/.ts/.js/.py OR 3+ step-by-step items   │
 │    schemas:      need at least one .sql file                        │
 │    dashboards:   need .html/.jsx/.tsx/.vue/.svelte or package.json │
 │    integrations: need .ts/.js/.py code files                        │
 │    primitives:   README must be 200+ words                          │
 │    extensions:   need both .sql AND .ts/.js/.py files              │
 │                                                                     │
 │  Check  7: PR format                                                │
 │    Title must match: [category] Description                         │
 │    e.g. "[recipes] Gmail conversation import"                       │
 │                                                                     │
 │  Check  8: No binary blobs                                          │
 │    No file over 1MB                                                  │
 │    No .exe/.dmg/.zip/.tar.gz/.rar/.7z/.msi/.pkg/.deb/.rpm         │
 │                                                                     │
 │  Check  9: README completeness                                      │
 │    README must mention "prerequisite"                               │
 │    README must have numbered steps (^\s*[0-9]+\.)                  │
 │    README must mention expected/outcome/result                     │
 │                                                                     │
 │  Check 10: Primitive dependencies                                   │
 │    If metadata.json lists requires_primitives: [rls, shared-mcp]  │
 │    → the named primitive directory must exist in primitives/       │
 │    → the README must link to primitives/[name]/                    │
 │                                                                     │
 │  Check 11: LLM clarity review                                       │
 │    PLANNED for v2. Auto-passes in current version.                 │
 │                                                                     │
 │  Check 12: Scope check                                              │
 │    Contribution PRs must only touch files in their own folder      │
 │    e.g. recipes/email-import/* only                                │
 │                                                                     │
 │  Check 13: Internal link validation                                 │
 │    Relative links in README.md must resolve to existing files      │
 └─────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
                                  Step 4: Post result as PR comment
                                  (updates existing bot comment if any)

                                  ✅ N/N checks passed!
                                     Ready for human review.
                                  ── or ──
                                  ❌ M checks failed.
                                     Fix and push again.
                                          │
                                          ├──► If any failed: exit 1
                                          │    (GitHub marks check "failed")
                                          │    Branch protection blocks merge
                                          │
                                          └──► If all passed: exit 0
                                               Maintainer can now approve + merge

 POST-MERGE REMINDERS (non-blocking, informational):
 ─────────────────────────────────────────────────────────────────────
 After a clean run, the bot comment also includes a checklist for
 maintainers to complete after merging:
   - Add contribution to category README.md index
   - Add contribution to root README.md community section
   - Add contributor to CONTRIBUTORS.md (if not already listed)
   - Post in #show-and-tell on Discord
```

### Why GitHub Actions

If you have used build systems like Make, CMake, or Gradle: GitHub Actions is the equivalent for cloud-based workflows. It is event-driven (triggers on PR open/push) and runs shell scripts on a managed Ubuntu virtual machine. The entire review logic in `ob1-review.yml` is standard bash — no special frameworks or DSLs.

The key design decision is that the 13 checks run as a single bash script rather than separate jobs. This keeps the workflow fast (no inter-job setup overhead) and produces a single, unified comment on the PR.

---

## 10. Technology Reference

This section explains each technology in the stack for developers who may not have web development backgrounds.

### PostgreSQL and Supabase

PostgreSQL is an open-source relational database. If you know MySQL or SQL Server, you know PostgreSQL. Supabase is a managed cloud service that runs PostgreSQL for you and adds several things on top:

- **REST API**: Supabase auto-generates a REST API from your schema. Every table becomes an endpoint. The `@supabase/supabase-js` client wraps these endpoints in a TypeScript-friendly interface.
- **Auth**: Built-in user authentication with JWTs (JSON Web Tokens). The `auth.uid()` function in SQL policies returns the UUID of the currently authenticated user from the JWT.
- **Edge Functions**: Serverless functions that run on Deno (a modern JavaScript/TypeScript runtime) close to users. They are deployed with the Supabase CLI and are analogous to AWS Lambda or Google Cloud Functions.
- **Secrets**: Encrypted key-value store for environment variables, analogous to AWS Secrets Manager.

### pgvector

pgvector is a PostgreSQL extension that adds a `VECTOR(n)` data type and vector distance operators. This is what makes semantic search possible.

```
Without pgvector:
  You can only compare text character by character.
  "Sarah left her job" ≠ "career changes"

With pgvector:
  You convert both texts to float[1536] using an embedding model.
  The vectors are close in 1536-dimensional space.
  Distance query finds them as similar.
```

The `<=>` operator is the cosine distance (lower = more similar). `1 - (<=>)` gives cosine similarity (higher = more similar).

The HNSW index (`Hierarchical Navigable Small World`) makes vector search fast. It is an approximate nearest neighbor algorithm — it does not guarantee finding the absolute closest vector, but it finds a very good answer in O(log n) time instead of O(n).

### TypeScript and Node.js

TypeScript is a typed superset of JavaScript. It compiles to JavaScript (`tsc` command) and then runs on Node.js. If you know Java or C#, TypeScript will feel familiar: static types, interfaces, generics.

The extension MCP servers are TypeScript programs that compile to `dist/index.js` and run as standard Node.js processes. The AI client spawns these processes and communicates with them over stdio pipes.

```
Java analogy:
  interface MCPTool { ... }
  List<MCPTool> tools = new ArrayList<>();
  server.registerHandler(ListToolsRequest.class, req -> tools);

TypeScript equivalent:
  const TOOLS: Tool[] = [...];
  server.setRequestHandler(ListToolsRequestSchema, async () => ({ tools: TOOLS }));
```

### Deno

Deno is a JavaScript/TypeScript runtime (like Node.js but newer) built on V8 (the Chrome engine). Supabase Edge Functions run on Deno. Key differences from Node.js:

- No `package.json` / `node_modules`. Dependencies are imported by URL.
- TypeScript works natively without compilation.
- Built-in security model (permissions required for file system, network access).

In the Slack capture integration (`integrations/slack-capture/README.md`), you can see Deno-style imports:

```typescript
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";
// Deno fetches the module from a CDN at first run, then caches it.
// Equivalent to: import { createClient } from "@supabase/supabase-js";
```

### OpenRouter

OpenRouter is an AI API gateway — a proxy that lets you use many different AI models through a single API endpoint and billing account. Open Brain uses it to access:

- `openai/text-embedding-3-small`: An embedding model from OpenAI. Takes text, returns float[1536].
- `openai/gpt-4o-mini`: A lightweight chat model. Used for JSON metadata extraction.

The OpenRouter API is compatible with the OpenAI API format, so libraries and code written for OpenAI work with OpenRouter by just changing the base URL.

### The @modelcontextprotocol/sdk Package

This is the official TypeScript SDK for implementing MCP servers. It handles:

- The JSON-RPC message framing
- Schema validation of incoming requests
- Transport abstraction (stdio vs HTTP)
- Error handling and response formatting

From the developer perspective, you just define your tools and handler functions. The SDK takes care of the protocol mechanics. See `extensions/household-knowledge/index.ts` for a complete working example.

### GitHub Actions

GitHub Actions is a CI/CD platform built into GitHub. Workflows are defined in YAML files in `.github/workflows/`. They trigger on repository events (push, PR, release) and run jobs on managed virtual machines.

The review workflow at `.github/workflows/ob1-review.yml` runs a single job with bash steps. The bash script builds up a results string and then uses the GitHub Script action to post it as a PR comment using the GitHub REST API.

```yaml
on:
  pull_request:
    types: [opened, synchronize, reopened]
    branches: [main]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run review checks
        run: |
          # 300 lines of bash
```

---

## File Reference

Key files by architectural role:

| File | Role |
|------|------|
| `docs/01-getting-started.md` | Step-by-step setup guide; contains the `thoughts` table DDL and `match_thoughts()` function |
| `integrations/slack-capture/README.md` | Contains the complete `ingest-thought` Edge Function source code |
| `extensions/household-knowledge/index.ts` | Canonical example of an extension MCP server; cleanest implementation to read first |
| `extensions/household-knowledge/schema.sql` | Canonical example of extension schema with RLS and triggers |
| `extensions/meal-planning/schema.sql` | Example of household-scoped (dual) RLS policies |
| `extensions/meal-planning/shared-server.ts` | Concrete implementation of the shared MCP server pattern |
| `extensions/professional-crm/schema.sql` | Example of cascade triggers (`update_last_contacted`) |
| `extensions/job-hunt/schema.sql` | Example of soft FK (application-managed cross-extension reference) |
| `primitives/rls/README.md` | Detailed guide to all three RLS patterns with working SQL |
| `primitives/shared-mcp/README.md` | Detailed guide to building scoped shared MCP servers |
| `.github/workflows/ob1-review.yml` | Complete automated PR review system (13 checks) |
| `.github/metadata.schema.json` | JSON Schema definition for metadata.json validation |
| `CONTRIBUTING.md` | Contribution rules, metadata format, and review process |

---

*This document reflects the architecture as of March 2026. If the system has changed and this document has not been updated, the source of truth is always the code in `extensions/`, `integrations/`, `primitives/`, and `docs/`.*
