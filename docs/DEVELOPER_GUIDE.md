# Open Brain Developer Guide

A comprehensive reference for developers who want to understand, extend, or contribute to the Open Brain project. Written for people with C, C++, or Java experience who are new to web and full-stack development.

---

## Table of Contents

1. [Technology Prerequisites](#1-technology-prerequisites)
2. [Repository Structure Deep Dive](#2-repository-structure-deep-dive)
3. [Development Environment Setup](#3-development-environment-setup)
4. [Understanding the Core System](#4-understanding-the-core-system)
5. [How to Build a New Extension](#5-how-to-build-a-new-extension)
6. [How to Build a New Recipe](#6-how-to-build-a-new-recipe)
7. [How to Build a New Integration](#7-how-to-build-a-new-integration)
8. [Testing Your Code](#8-testing-your-code)
9. [The PR Review Process](#9-the-pr-review-process)
10. [Common Patterns and Best Practices](#10-common-patterns-and-best-practices)
11. [Debugging Guide](#11-debugging-guide)
12. [Glossary](#12-glossary)

---

## 1. Technology Prerequisites

This section explains every technology used in Open Brain. If you have experience with C/C++/Java, each comparison will help anchor the concept.

### 1.1 Node.js 18+

**What it is:**
Node.js is a runtime environment that lets you run JavaScript outside of a web browser. JavaScript was originally invented for browsers. Node.js took the JavaScript engine out of Chrome and let you run it anywhere — on your server, on your laptop, inside a script.

If you think in C terms: Node.js is the equivalent of a runtime like the JVM or glibc, but for JavaScript. Your `.js` or `.ts` files are the equivalent of `.c` or `.cpp` source files; Node.js compiles and runs them.

**Why Open Brain uses it:**
The MCP server that lets AI tools talk to Supabase is written in TypeScript and runs on Node.js. It is a small server process that stays running on your machine and handles requests from your AI client.

**How to install on Windows:**
1. Go to https://nodejs.org
2. Download the LTS (Long Term Support) version — as of 2026, that is Node.js 20.x
3. Run the installer `.msi` file, accept defaults
4. Open PowerShell or Command Prompt and verify:

```bash
node --version
# Should print: v20.x.x (or higher, minimum v18.x.x)
```

**How to install on macOS:**
```bash
# Option 1: Homebrew (recommended)
brew install node

# Option 2: Download the .pkg installer from nodejs.org
```

**How to install on Linux (Ubuntu/Debian):**
```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs
node --version
```

**How to verify:**
```bash
node --version   # Should show v18.x.x or higher
npm --version    # Should show 9.x.x or higher
```

---

### 1.2 npm (Node Package Manager)

**What it is:**
npm is the package manager that ships with Node.js. It downloads and manages libraries your project depends on. Think of it as:
- Like Maven for Java (`pom.xml` → `package.json`)
- Like pip for Python (`requirements.txt` → `package.json`)
- Like apt/yum for Linux system packages, but for JavaScript libraries

**Key concepts:**

`package.json` is the manifest file — it lists your project's name, version, and all its dependencies. Every extension in this repo has one. Compare it to `pom.xml` in Maven or `build.gradle` in Gradle.

`node_modules/` is the directory where npm downloads all dependencies. It is the equivalent of your Maven local repository cache. It can be large (hundreds of megabytes). It is always listed in `.gitignore` — you never commit it.

`npm install` reads `package.json` and downloads everything listed in it into `node_modules/`. This is equivalent to running `mvn install` or `pip install -r requirements.txt`.

`package-lock.json` locks exact dependency versions, like Maven's `pom.xml` dependency management but stricter. Always commit this file.

**Common commands:**
```bash
# Install all dependencies listed in package.json
npm install

# Install a new package and add it to package.json
npm install @supabase/supabase-js

# Install a package only for development (not production)
npm install --save-dev typescript

# Run a script defined in package.json
npm run build

# Install a package globally (available everywhere on your system)
npm install -g typescript
```

---

### 1.3 TypeScript

**What it is:**
TypeScript is a typed superset of JavaScript. It adds a static type system to JavaScript, similar to how Java's type system works but layered on top of a dynamically typed language.

If you know Java: TypeScript is to JavaScript what Java is to a scripting language. You write TypeScript (`.ts` files), run the TypeScript compiler (`tsc`), and it produces JavaScript (`.js` files) that Node.js actually executes. TypeScript does not run directly — it compiles to JavaScript first.

**Analogy:** TypeScript is like writing C with type annotations, which then gets compiled to assembly. The type annotations give you safety at compile time, but the output is still the lower-level thing.

**Why Open Brain uses it:**
TypeScript catches bugs at compile time instead of at runtime. When you call a Supabase function and it returns a result, TypeScript forces you to define what the result shape looks like. If you try to access a field that does not exist, the compiler tells you before you deploy.

**TypeScript vs JavaScript comparison:**

```typescript
// JavaScript (no types — discover problems at runtime)
function addItem(name, quantity) {
  return { name: name, qty: quantity };
}

// TypeScript (types checked at compile time)
interface Item {
  name: string;
  quantity: number;
}

function addItem(name: string, quantity: number): Item {
  return { name, quantity };
}

// This would be a compile error in TypeScript:
addItem(42, "hello");  // Error: Argument of type 'number' is not assignable to parameter of type 'string'
```

**The TypeScript compile process:**
```bash
# Compile TypeScript to JavaScript
npx tsc

# Or if typescript is installed globally
tsc

# Watch mode — recompile whenever you save a file
tsc --watch
```

**`tsconfig.json`** is the TypeScript configuration file. Every extension has one. Here is what the settings in our extensions mean:

```json
{
  "compilerOptions": {
    "target": "ES2022",          // Compile to modern JavaScript (like targeting x86-64)
    "module": "Node16",          // Use Node.js module system
    "moduleResolution": "Node16", // How to find imported modules
    "outDir": "./build",         // Put compiled .js files in build/
    "rootDir": "./",             // Source files are in the root
    "strict": true,              // Enable all strict type checks (recommended)
    "esModuleInterop": true,     // Compatibility for different module formats
    "skipLibCheck": true,        // Skip type checking of .d.ts declaration files
    "declaration": true,         // Generate .d.ts files (type declarations)
    "sourceMap": true            // Generate source maps for debugging
  }
}
```

---

### 1.4 PostgreSQL

**What it is:**
PostgreSQL (often called "Postgres") is a powerful, open-source relational database system. If you know MySQL or SQLite, PostgreSQL is similar but more feature-rich. If you know Oracle DB or SQL Server, it is the same category of tool.

**Comparison table:**

| Feature | PostgreSQL | MySQL | SQLite |
|---------|-----------|-------|--------|
| ACID transactions | Yes | Yes (InnoDB) | Yes |
| JSON support | JSONB (advanced) | JSON | Limited |
| Extensions | Many (pgvector, PostGIS) | Few | Very few |
| Full-text search | Built-in | Built-in | Limited |
| Vector search | pgvector extension | No | No |
| Hosted options | Supabase, RDS, Neon | RDS, PlanetScale | None |

**Why Open Brain uses it:**
PostgreSQL supports pgvector, the extension that allows storing and searching vector embeddings. No other major free database had this when Open Brain was designed.

**Key PostgreSQL concepts you will use:**

- **Tables:** Same as in any relational database. Rows and columns.
- **UUID:** Universally Unique Identifier — a 128-bit random ID. Postgres generates them with `gen_random_uuid()`. This is the standard primary key type used throughout the project.
- **JSONB:** Binary JSON column type. Stores arbitrary JSON data that you can query and index. Much more powerful than storing JSON as a string. Think of it like a `Map<String, Object>` that the database understands natively.
- **TIMESTAMPTZ:** Timestamp with time zone. Always use this instead of plain `TIMESTAMP`. It stores time in UTC and handles time zone conversions correctly.
- **TEXT[]:** An array of text strings. PostgreSQL supports array columns natively, unlike most databases.
- **Triggers:** Code that runs automatically when rows are inserted, updated, or deleted. Used in Open Brain to automatically update `updated_at` timestamps.
- **Functions:** Stored procedures written in PL/pgSQL (PostgreSQL's procedural language). The `match_thoughts()` function is a key example.
- **Extensions:** Plugins that add capabilities to PostgreSQL. The two critical ones here are `pgvector` (for vector search) and `uuid-ossp` (for UUID generation, though `gen_random_uuid()` is built-in in modern Postgres).

---

### 1.5 Supabase

**What it is:**
Supabase is a hosted PostgreSQL service that bundles several tools on top of the database:
- A managed PostgreSQL database (the actual data store)
- An auto-generated REST API (so you can query your database over HTTP without writing server code)
- Authentication (user accounts, JWT tokens, OAuth)
- Edge Functions (serverless functions that run close to your users)
- Storage (file uploads)
- A web dashboard (so you can browse your data visually)

**Analogy for Java/C developers:**
If you have used Firebase from Google, Supabase is the open-source equivalent but built on PostgreSQL instead of Firebase's proprietary database. The key difference: Supabase data is standard PostgreSQL — you can always export it, run it yourself, or connect to it with any PostgreSQL client.

Think of Supabase as: PostgreSQL + Spring Boot REST API generator + Spring Security + hosted infrastructure, all managed for you with a free tier.

**Why Open Brain uses it:**
- The free tier is enough for personal use
- pgvector support is built-in (just enable the extension)
- Edge Functions let you deploy server code without managing servers
- The auto-generated REST API means you can query your database from TypeScript without writing SQL manually (you use the Supabase client library)
- The `SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` are the two credentials that identify your database instance

**Key Supabase concepts:**

`SUPABASE_URL`: The HTTPS endpoint for your project. Format: `https://YOUR_PROJECT_REF.supabase.co`. Everything talks to this URL.

`SUPABASE_SERVICE_ROLE_KEY`: A JWT token that gives full admin access to your database, bypassing all Row Level Security. Used by server-side code (MCP servers, Edge Functions) that you control. Never expose this in client-side code.

`anon key` (also called `publishable key`): A limited key for client-side code. Respects Row Level Security policies. Not used much in Open Brain because everything runs server-side.

**The Supabase JavaScript client:**
```typescript
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY);

// Query a table — equivalent to: SELECT * FROM household_items WHERE user_id = ?
const { data, error } = await supabase
  .from("household_items")
  .select("*")
  .eq("user_id", userId);

// Insert a row — equivalent to: INSERT INTO household_items (name, ...) VALUES (?, ...)
const { data, error } = await supabase
  .from("household_items")
  .insert({ name: "Dishwasher", category: "appliance" })
  .select()
  .single();
```

The `.select().single()` at the end means "return the inserted row as a single object." Without `.select()`, the insert returns no data. Without `.single()`, it returns an array.

---

### 1.6 pgvector

**What it is:**
pgvector is a PostgreSQL extension that adds a new column type called `vector` and fast similarity search operations over vectors. A vector in this context is an array of floating-point numbers — for example, `vector(1536)` is an array of 1,536 floats.

**Why this matters for Open Brain:**
When you capture a thought, an AI model converts that text into a 1,536-dimensional vector (a "vector embedding"). Similar texts produce similar vectors. pgvector lets you query "find me all thoughts whose vector is close to this query's vector" — which is semantic search.

**The HNSW index:**
```sql
CREATE INDEX ON thoughts USING hnsw (embedding vector_cosine_ops);
```
This creates a Hierarchical Navigable Small World (HNSW) index — a graph-based index optimized for approximate nearest-neighbor search in high-dimensional space. For your purposes, just know it makes vector search fast. Without it, every search would do a full table scan comparing your query vector against every stored vector.

**The distance operator:**
```sql
-- Cosine distance (lower = more similar)
embedding <=> query_embedding

-- Cosine similarity (higher = more similar)
1 - (embedding <=> query_embedding)
```
The `<=>` operator is added by pgvector. It is the core of semantic search.

---

### 1.7 Deno

**What it is:**
Deno is an alternative JavaScript and TypeScript runtime — similar to Node.js but with a different design philosophy. It was created by the same person who created Node.js.

Key differences from Node.js:
- Deno can run TypeScript directly without a compilation step
- Deno uses a permission-based security model (explicit file, network, env access)
- Deno uses URLs to import dependencies instead of `npm install`
- Supabase Edge Functions run on Deno, not Node.js

**Why Open Brain uses it:**
Supabase Edge Functions run on Deno. You do not choose this — when you deploy a Supabase Edge Function, it runs on Deno. The important consequence is that the import syntax in Edge Functions looks different from the TypeScript in the extension MCP servers:

```typescript
// Node.js / npm import (used in extension MCP servers)
import { createClient } from "@supabase/supabase-js";

// Deno import (used in Edge Functions like the Slack capture integration)
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";
```

In Deno, you import from URLs. The `esm.sh` service converts npm packages into Deno-compatible ES modules on the fly.

**Environment variables in Deno:**
```typescript
// Node.js
process.env.MY_VARIABLE

// Deno
Deno.env.get("MY_VARIABLE")
```

You will see both patterns in this repo: `process.env` in extension `index.ts` files (Node.js), and `Deno.env.get()` in Edge Function code.

---

### 1.8 Git and GitHub

**What they are:**
Git is a distributed version control system. GitHub is a web platform that hosts Git repositories and adds collaboration features (pull requests, issues, actions).

If you are coming from Subversion (SVN): Git is distributed, meaning every developer has a full copy of history. There is no single "server" — GitHub is just a hosting service that also serves as the canonical upstream.

**Key concepts for contributing to Open Brain:**

```bash
# Clone the repository to your local machine
git clone https://github.com/NateBJones-Projects/OB1.git
cd OB1

# Create a branch for your contribution
# Branch naming convention: contrib/<your-github-username>/<short-description>
git checkout -b contrib/yourname/add-reading-list-schema

# Stage your changes
git add recipes/reading-list/README.md
git add recipes/reading-list/metadata.json
git add recipes/reading-list/schema.sql

# Commit your changes
git commit -m "[recipes] Add reading list schema with rating metadata"

# Push to GitHub
git push origin contrib/yourname/add-reading-list-schema
```

**Then create a Pull Request on GitHub** by going to the repository URL and clicking "Compare & pull request."

---

### 1.9 MCP (Model Context Protocol)

**What it is:**
MCP is an open protocol developed by Anthropic that defines how AI assistants (like Claude) communicate with external tools and services. It standardizes the interface between an AI model and the "tools" it can call.

**Analogy:**
Think of MCP as a standard interface definition — like a header file in C or an interface in Java — that any AI client and any tool server can implement. Just as a USB device works with any USB host, an MCP server works with any MCP client (Claude Desktop, Claude Code, ChatGPT with MCP support, etc.).

**How it works:**

```
User types: "What are my paint colors?"
     |
     v
Claude (AI client) -- decides to use a tool
     |
     v
MCP protocol: Sends a JSON message to the MCP server
  {
    "method": "tools/call",
    "params": {
      "name": "search_household_items",
      "arguments": { "user_id": "abc123", "category": "paint" }
    }
  }
     |
     v
MCP server (your extension's index.ts, running on Node.js)
     |
     v
Supabase (queries the database)
     |
     v
Returns results back through MCP to Claude
     |
     v
Claude formats the answer: "You have Living Room Paint (Sherwin Williams Sea Salt SW 6204)..."
```

**Two transport modes:**

1. **stdio transport** (used by extension MCP servers): The MCP server runs as a local process. Claude Desktop starts it as a subprocess and communicates via stdin/stdout. Think of it like pipe-based IPC (`|` in shell).

2. **HTTP transport** (used by the core Open Brain MCP server): The MCP server is a hosted URL (a Supabase Edge Function). The AI client sends HTTP requests to it. This is what the `?key=your-access-key` URL pattern is about.

**Key MCP concepts in the codebase:**

- **Tools:** Named functions the AI can call. Each tool has a name, description, and input schema (JSON Schema format).
- **`ListToolsRequestSchema`:** When Claude connects to an MCP server, it asks "what tools do you have?" This handler responds with the list.
- **`CallToolRequestSchema`:** When Claude wants to use a tool, it sends this request. Your handler routes it to the right function.
- **`inputSchema`:** Describes what parameters a tool accepts, in JSON Schema format. Claude uses this to know what arguments to pass.

---

## 2. Repository Structure Deep Dive

Every file and folder has a specific purpose. This section explains all of them.

### Top-Level Files

```
OB1/
├── README.md              # Project overview and getting-started guide
├── CONTRIBUTING.md        # Rules for contributors (READ THIS BEFORE CONTRIBUTING)
├── CONTRIBUTORS.md        # List of everyone who has contributed
├── CODE_OF_CONDUCT.md     # Community behavior expectations
├── SECURITY.md            # How to report security vulnerabilities
├── LICENSE.md             # FSL-1.1-MIT license terms
└── CLAUDE.md              # Instructions for AI coding tools (you are reading the guide it points to)
```

### `.github/` — GitHub Configuration

```
.github/
├── metadata.schema.json         # JSON Schema for validating metadata.json files
├── PULL_REQUEST_TEMPLATE.md     # Template shown when opening a PR
├── ob1-logo.png                 # Repository logo
├── ob1-logo-wide.png            # Wide version of logo
├── ISSUE_TEMPLATE/              # Templates for different issue types
│   ├── bug-report.yml
│   ├── feature-request.yml
│   ├── extension-submission.yml
│   ├── recipe-submission.yml
│   ├── primitive-submission.yml
│   ├── non-technical-contribution.yml
│   ├── blank.yml
│   └── config.yml
└── workflows/
    ├── ob1-review.yml           # THE most important file — 11 automated PR checks
    ├── markdown-lint.yml        # Checks Markdown formatting
    ├── auto-label.yml           # Automatically labels PRs by category
    └── welcome.yml              # Posts welcome message for first-time contributors
```

`ob1-review.yml` is the automated gatekeeper. Every PR is run through it. Understanding what it checks (see Section 9) tells you exactly what your contribution needs to pass.

`metadata.schema.json` defines the exact structure required for every `metadata.json` file. It is a JSON Schema document — a standard format for describing what valid JSON should look like.

### `extensions/` — The Curated Learning Path (Curated, Do Not Add Without Approval)

```
extensions/
├── README.md                    # Overview of all 6 extensions and the learning path
├── _template/                   # Template for new extensions (read before creating one)
│   ├── README.md
│   └── metadata.json
├── household-knowledge/         # Extension 1 (Beginner)
│   ├── index.ts                 # MCP server — the main server code
│   ├── schema.sql               # Database tables, indexes, RLS policies
│   ├── package.json             # npm manifest (dependencies, scripts)
│   ├── tsconfig.json            # TypeScript compiler config
│   ├── metadata.json            # Structured contribution metadata
│   └── README.md                # Setup instructions
├── home-maintenance/            # Extension 2 (Beginner)
├── family-calendar/             # Extension 3 (Intermediate)
├── meal-planning/               # Extension 4 (Intermediate)
│   └── shared-server.ts         # Extra file: shared access MCP server
├── professional-crm/            # Extension 5 (Intermediate)
└── job-hunt/                    # Extension 6 (Advanced)
```

Extensions form a learning path. Each one introduces new concepts:
- Extension 1 (`household-knowledge`): Basic MCP server pattern, JSONB, simple CRUD
- Extension 2 (`home-maintenance`): Scheduling, date queries
- Extension 3 (`family-calendar`): Multi-person data, arrays
- Extension 4 (`meal-planning`): Row Level Security, shared access, the `shared-server.ts` pattern
- Extension 5 (`professional-crm`): Cross-extension integration (reading from the core `thoughts` table), database triggers
- Extension 6 (`job-hunt`): Complex schema (5 tables), advanced queries

**Every extension has exactly these 6 files** (plus `shared-server.ts` in Extension 4):
- `index.ts` — The MCP server
- `schema.sql` — The database schema
- `package.json` — Dependencies and scripts
- `tsconfig.json` — TypeScript config
- `metadata.json` — Contribution metadata
- `README.md` — Setup instructions

### `primitives/` — Reusable Concept Guides (Curated)

```
primitives/
├── README.md
├── _template/
│   ├── README.md
│   └── metadata.json
├── rls/
│   ├── README.md                # Comprehensive guide to Row Level Security
│   └── metadata.json
└── shared-mcp/
    ├── README.md                # Guide to building shared-access MCP servers
    └── metadata.json
```

Primitives are concept guides, not code. They explain a pattern in depth so that extensions can link to them instead of re-explaining. The rule: a primitive must be referenced by at least 2 extensions.

`rls/` teaches Row Level Security — how to make each user see only their own data.
`shared-mcp/` teaches how to build a separate MCP server that gives another person scoped access to parts of your database.

### `recipes/` — Community Standalone Builds (Open for Contributions)

```
recipes/
├── README.md
├── _template/
│   ├── README.md
│   └── metadata.json
├── chatgpt-conversation-import/
│   ├── import-chatgpt.py        # Python script (recipes can use any language)
│   ├── requirements.txt         # Python dependencies
│   ├── .env.example             # Template for environment variables (never .env itself)
│   ├── .gitignore
│   ├── README.md
│   └── metadata.json
├── daily-digest/
│   ├── README.md
│   └── metadata.json
├── email-history-import/
│   ├── README.md
│   └── metadata.json
└── panning-for-gold/
    ├── panning-for-gold.skill.md
    ├── README.md
    └── metadata.json
```

Recipes are standalone capability adds. They do not have to be TypeScript — the ChatGPT conversation import uses Python. They do not have to have a running server — some are just SQL scripts or manual workflows.

**Minimum required for a recipe:** `README.md` + `metadata.json` + at least one code file OR three or more numbered steps in the README.

### `schemas/` — Database Table Extensions (Open for Contributions)

```
schemas/
├── README.md
└── _template/
    ├── README.md
    └── metadata.json
```

Currently only the template exists. Schemas are SQL files that add new tables to someone's Supabase database. They must not touch the core `thoughts` table in breaking ways.

**Minimum required for a schema:** `README.md` + `metadata.json` + at least one `.sql` file.

### `dashboards/` — Frontend Templates (Open for Contributions)

```
dashboards/
├── README.md
└── _template/
    ├── README.md
    └── metadata.json
```

Currently only the template exists. Dashboards are web frontends meant to be hosted on Vercel or Netlify, pointing at a user's Supabase backend.

**Minimum required for a dashboard:** `README.md` + `metadata.json` + frontend code (`.html`, `.jsx`, `.tsx`, `.vue`, or `.svelte`) or a `package.json`.

### `integrations/` — MCP Extensions and Webhooks (Open for Contributions)

```
integrations/
├── README.md
├── _template/
│   ├── README.md
│   └── metadata.json
├── slack-capture/
│   ├── README.md                # Includes the full Edge Function code inline
│   └── metadata.json
└── discord-capture/
    ├── README.md
    └── metadata.json
```

Integrations are capture sources and MCP extensions — ways to get data into Open Brain from external services, or ways to extend what the MCP server can do.

**Minimum required for an integration:** `README.md` + `metadata.json` + code files (`.ts`, `.js`, or `.py`).

Note: The Slack integration embeds its Edge Function code inline in the README rather than in a separate file. This is valid because the README has step-by-step instructions that include the code as copy-paste blocks.

### `docs/` — Setup Guides and Documentation

```
docs/
├── 01-getting-started.md       # Main setup guide (30-45 minutes, zero coding required)
├── 02-companion-prompts.md     # Prompts for migrating memories and building habits
├── 03-faq.md                   # Common questions and answers
└── 04-ai-assisted-setup.md     # Guide for using AI tools (Cursor, Claude Code) to build
```

These are end-user docs. The DEVELOPER_GUIDE.md you are reading is also in this folder.

### `resources/` — Companion Tools and Files

```
resources/
├── ob-cli/              # ob CLI tool — direct Open Brain access without MCP
│   ├── ob               # The bash script (curl + jq only)
│   └── README.md        # Installation and usage
└── open-brain-companion.skill  # Claude Skill for AI-assisted help
```

Resources are companion tools. The `ob` CLI provides a lightweight alternative to MCP for terminal-based AI tools — see [`docs/CLI_DIRECT_APPROACH.md`](CLI_DIRECT_APPROACH.md) for the full guide. The Claude Skill helps users when they get stuck.

### `.ignore/` — Files Hidden from Some Tools

```
.ignore/
├── prompt.txt    # System prompt used for the automated review AI agent (planned v2)
└── substack.txt  # Content reference for the Substack newsletter
```

These files are excluded from certain tooling contexts but are tracked in git.

### `.claude/` — Claude Code Configuration

```
.claude/
└── skills/
    └── review-pr.md    # Claude Skill definition for PR review assistance
```

---

## 3. Development Environment Setup

Follow these steps in order. Do not skip any.

### Step 1: Install Node.js 18+

Follow the installation instructions in Section 1.1 for your operating system. Verify with:

```bash
node --version   # Must be v18.x.x or higher
npm --version    # Should be 9.x.x or higher
```

### Step 2: Install TypeScript Globally

```bash
npm install -g typescript

# Verify
tsc --version
# Should print: Version 5.x.x
```

The `-g` flag installs it globally so the `tsc` command is available anywhere on your system. This is the TypeScript compiler.

### Step 3: Install the Supabase CLI

The Supabase CLI lets you deploy Edge Functions, manage secrets, and link your local project to your Supabase instance.

**Windows (recommended method using Scoop):**
```powershell
# First install Scoop if you do not have it (run in PowerShell)
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
Invoke-RestMethod -Uri https://get.scoop.sh | Invoke-Expression

# Then install Supabase
scoop bucket add supabase https://github.com/supabase/scoop-bucket.git
scoop install supabase
```

**macOS (using Homebrew):**
```bash
brew install supabase/tap/supabase
```

**Linux or macOS without Homebrew:**
```bash
npm install -g supabase
```

**Verify:**
```bash
supabase --version
# Should print: 1.x.x or higher
```

### Step 4: Clone the Repository

```bash
git clone https://github.com/NateBJones-Projects/OB1.git
cd OB1
```

### Step 5: Set Up a Working Open Brain Instance

You cannot develop extensions without a working Supabase database to connect to. Follow the full setup guide at `docs/01-getting-started.md` before continuing. You need:
- A Supabase project with the `thoughts` table and `match_thoughts()` function
- Your `SUPABASE_URL` (format: `https://YOUR_PROJECT_REF.supabase.co`)
- Your `SUPABASE_SERVICE_ROLE_KEY` (the secret key from Supabase Settings → API)

### Step 6: Run an Extension MCP Server Locally

This is how you test an extension on your own machine before contributing it.

**Step 6a: Navigate to the extension directory**
```bash
cd extensions/household-knowledge
```

**Step 6b: Install dependencies**
```bash
npm install
# This reads package.json and downloads @modelcontextprotocol/sdk and @supabase/supabase-js
# into node_modules/ (this directory is not committed to git)
```

**Step 6c: Build (compile TypeScript to JavaScript)**
```bash
npm run build
# This runs `tsc` per the package.json scripts section
# Output goes to build/ directory
```

**Step 6d: Set environment variables**

On macOS/Linux, you can set them inline:
```bash
export SUPABASE_URL="https://YOUR_PROJECT_REF.supabase.co"
export SUPABASE_SERVICE_ROLE_KEY="your-service-role-key-here"
```

On Windows PowerShell:
```powershell
$env:SUPABASE_URL = "https://YOUR_PROJECT_REF.supabase.co"
$env:SUPABASE_SERVICE_ROLE_KEY = "your-service-role-key-here"
```

**Step 6e: Run the server manually to verify it starts**
```bash
node build/index.js
# Should print: "Household Knowledge Base MCP Server running on stdio"
# Press Ctrl+C to stop
```

If the server exits immediately with an error, check your environment variables.

### Step 7: Connect Your Local MCP Server to Claude Desktop

Claude Desktop is configured via a JSON file that tells it which MCP servers to start.

**Find the config file:**
- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json` (usually `C:\Users\YourName\AppData\Roaming\Claude\claude_desktop_config.json`)

**Edit the config file** (create it if it does not exist):

```json
{
  "mcpServers": {
    "household-knowledge": {
      "command": "node",
      "args": [
        "/absolute/path/to/OB1/extensions/household-knowledge/build/index.js"
      ],
      "env": {
        "SUPABASE_URL": "https://YOUR_PROJECT_REF.supabase.co",
        "SUPABASE_SERVICE_ROLE_KEY": "your-service-role-key-here"
      }
    }
  }
}
```

**Important:** Use the absolute path to the compiled `build/index.js` file. On Windows, use forward slashes or escaped backslashes.

**Restart Claude Desktop** after editing the config. The server will start automatically when Claude Desktop opens. You can verify it is running by checking Settings → Developer (or similar) in Claude Desktop.

**Test it:** Open a conversation in Claude Desktop and ask: "Add a household item: Living Room Paint, category paint, Sherwin Williams Sea Salt SW 6204." The AI should call the `add_household_item` tool.

---

## 4. Understanding the Core System

Before building anything, you need to understand how the core Open Brain system works at a technical level.

### 4.1 The Thoughts Table

This is the core database table. Every piece of information stored in Open Brain lives here. It is defined in the getting-started guide:

```sql
CREATE TABLE thoughts (
  id          UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  content     TEXT NOT NULL,
  embedding   VECTOR(1536),
  metadata    JSONB DEFAULT '{}'::jsonb,
  created_at  TIMESTAMPTZ DEFAULT now(),
  updated_at  TIMESTAMPTZ DEFAULT now()
);
```

**Column breakdown:**

| Column | Type | Purpose |
|--------|------|---------|
| `id` | UUID | Unique identifier, auto-generated. Like a database GUID. Never set this manually. |
| `content` | TEXT | The actual text of the thought. No length limit (PostgreSQL TEXT is unlimited). |
| `embedding` | VECTOR(1536) | The 1,536-dimensional vector encoding the semantic meaning of `content`. This is what makes search work. |
| `metadata` | JSONB | Flexible key-value pairs: type, topics, people, action_items, source, etc. No fixed schema. |
| `created_at` | TIMESTAMPTZ | When the thought was created. Auto-set by database default. |
| `updated_at` | TIMESTAMPTZ | When the thought was last modified. Kept current by a trigger. |

**The guard rail:** You must never ALTER or DROP the columns of this table. You can add new columns to it if needed, but existing columns (`id`, `content`, `embedding`, `metadata`, `created_at`, `updated_at`) are sacred. The automated review will reject any SQL file that tries to modify them.

### 4.2 How Vector Embeddings Work

This is the intellectual core of the system. Understanding it will make everything else click.

**The concept:**
An embedding is a function that converts text into a point in a high-dimensional space. In Open Brain, the model is `text-embedding-3-small` from OpenAI (accessed via OpenRouter), which produces 1,536-dimensional vectors.

Imagine a simpler case: embedding words in 2D space. In a well-trained 2D embedding:
- "king" might be at (0.8, 0.7)
- "queen" might be at (0.75, 0.9) (close to "king")
- "car" might be at (-0.5, 0.1) (far from "king")
- "automobile" might be at (-0.48, 0.15) (close to "car")

The system works in 1,536 dimensions instead of 2, and the model has learned from billions of text examples what "similar meaning" looks like in that space.

**Practical consequence:**
When you search for "What did I note about career changes?" and you have a thought "Sarah mentioned she's thinking about leaving her job," these two phrases have zero keywords in common. But their 1,536-dimensional vectors are nearby in space, so the system finds the match.

**In C/Java terms:** Think of the 1,536 floats as a feature vector — like a hash, but instead of being designed to be unique, it is designed so that semantically similar inputs produce similar output vectors. The "distance" between two vectors measures semantic dissimilarity.

**How Open Brain generates embeddings:**

```typescript
// This is what the Edge Function does when you capture a thought
const response = await fetch("https://openrouter.ai/api/v1/embeddings", {
  method: "POST",
  headers: {
    "Authorization": `Bearer ${OPENROUTER_API_KEY}`,
    "Content-Type": "application/json"
  },
  body: JSON.stringify({
    model: "openai/text-embedding-3-small",
    input: "Sarah mentioned she's thinking about leaving her job"
  })
});

const result = await response.json();
const embedding = result.data[0].embedding;
// embedding is now a JavaScript array of 1536 floats
// Example: [0.024, -0.015, 0.082, ..., -0.007]  (1536 numbers)
```

This array is stored in the `embedding` column.

### 4.3 The match_thoughts() Function

This PostgreSQL function is the search engine. It takes a query embedding and returns the most semantically similar thoughts:

```sql
CREATE OR REPLACE FUNCTION match_thoughts(
  query_embedding VECTOR(1536),   -- The embedding of the search query
  match_threshold FLOAT DEFAULT 0.7, -- Only return results above this similarity
  match_count INT DEFAULT 10,     -- Maximum number of results
  filter JSONB DEFAULT '{}'::jsonb -- Optional: filter by metadata fields
)
RETURNS TABLE (
  id UUID,
  content TEXT,
  metadata JSONB,
  similarity FLOAT,
  created_at TIMESTAMPTZ
)
LANGUAGE plpgsql
AS $$
BEGIN
  RETURN QUERY
  SELECT
    t.id,
    t.content,
    t.metadata,
    1 - (t.embedding <=> query_embedding) AS similarity,  -- cosine similarity
    t.created_at
  FROM thoughts t
  WHERE 1 - (t.embedding <=> query_embedding) > match_threshold
    AND (filter = '{}'::jsonb OR t.metadata @> filter)
  ORDER BY t.embedding <=> query_embedding  -- order by cosine distance (ascending)
  LIMIT match_count;
END;
$$;
```

**What the operators mean:**
- `<=>` is the pgvector cosine distance operator. A value of 0 means identical vectors; a value of 2 means maximally dissimilar.
- `1 - (t.embedding <=> query_embedding)` converts distance to similarity. Similarity of 1.0 = identical; 0.0 = completely unrelated.
- `t.metadata @> filter` checks if the thought's metadata contains all the key-value pairs in `filter`. The `@>` operator means "contains."

**Example call from TypeScript:**
```typescript
const { data, error } = await supabase.rpc("match_thoughts", {
  query_embedding: queryVector,     // 1536-element float array
  match_threshold: 0.7,             // Only return results with >70% similarity
  match_count: 10,                  // Return at most 10 results
  filter: { source: "slack" }       // Only return thoughts captured from Slack
});
```

### 4.4 How the MCP Server Works

Looking at the extension MCP server (`extensions/household-knowledge/index.ts`) section by section:

**Section 1: Environment validation (startup)**
```typescript
const SUPABASE_URL = process.env.SUPABASE_URL;
const SUPABASE_SERVICE_ROLE_KEY = process.env.SUPABASE_SERVICE_ROLE_KEY;

if (!SUPABASE_URL || !SUPABASE_SERVICE_ROLE_KEY) {
  console.error("Error: SUPABASE_URL and SUPABASE_SERVICE_ROLE_KEY must be set");
  process.exit(1);
}
```
This runs when the server starts. If the required environment variables are missing, the server exits immediately with exit code 1. This is equivalent to checking `argc` and `argv` in a C `main()` function.

**Section 2: Supabase client initialization**
```typescript
const supabase: SupabaseClient = createClient(
  SUPABASE_URL,
  SUPABASE_SERVICE_ROLE_KEY,
  {
    auth: {
      autoRefreshToken: false,
      persistSession: false,
    },
  }
);
```
Creates one Supabase client instance at module level, shared by all requests. This is a singleton. The `autoRefreshToken: false` and `persistSession: false` options are appropriate for server-side code that does not need to maintain user sessions.

**Section 3: Type definitions**
```typescript
interface HouseholdItem {
  id: string;
  user_id: string;
  name: string;
  category: string | null;  // The | null means this field can be null
  // ...
}
```
TypeScript interfaces define the shape of objects. This is like a struct in C or a POJO in Java. The `| null` syntax means the field can be either a string or null (equivalent to a nullable type).

**Section 4: Tool definitions**
```typescript
const TOOLS: Tool[] = [
  {
    name: "add_household_item",
    description: "Add a new household item (paint color, appliance, measurement, document, etc.)",
    inputSchema: {
      type: "object",
      properties: {
        user_id: { type: "string", description: "User ID (UUID)" },
        name: { type: "string", description: "Name or description of the item" },
        // ...
      },
      required: ["user_id", "name"],
    },
  },
  // ...
];
```
The `TOOLS` array is a constant that declares what tools this server provides. Claude reads this list when it connects to the server and uses it to know what it can call. The `inputSchema` is a JSON Schema specification — it tells Claude the exact parameters each tool accepts.

**Section 5: Tool handlers**
```typescript
async function handleAddHouseholdItem(args: any): Promise<string> {
  const { user_id, name, category, location, details, notes } = args;

  const { data, error } = await supabase
    .from("household_items")
    .insert({
      user_id,
      name,
      category: category || null,
      // ...
    })
    .select()
    .single();

  if (error) {
    throw new Error(`Failed to add household item: ${error.message}`);
  }

  return JSON.stringify({ success: true, message: `Added household item: ${name}`, item: data }, null, 2);
}
```
Each handler is an `async` function (like Java's `CompletableFuture` pattern) that:
1. Destructures the input arguments (like unpacking a struct)
2. Calls Supabase to do the actual database work
3. Checks for errors
4. Returns a JSON string as the result

The `await` keyword waits for an asynchronous operation to complete. In Node.js, all I/O (network calls, file reads) is non-blocking and returns a Promise. `await` pauses the function until the Promise resolves. It is similar to calling `.get()` on a Java Future, but without blocking a thread.

**Section 6: Server setup**
```typescript
const server = new Server(
  { name: "household-knowledge", version: "1.0.0" },
  { capabilities: { tools: {} } }
);
```
Creates an MCP Server object. The `capabilities: { tools: {} }` tells MCP clients that this server provides tools.

**Section 7: Request handler registration**
```typescript
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: TOOLS,
}));

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  try {
    switch (name) {
      case "add_household_item":
        return { content: [{ type: "text", text: await handleAddHouseholdItem(args) }] };
      // ...
      default:
        throw new Error(`Unknown tool: ${name}`);
    }
  } catch (error) {
    const errorMessage = error instanceof Error ? error.message : String(error);
    return {
      content: [{ type: "text", text: JSON.stringify({ success: false, error: errorMessage }) }],
      isError: true,
    };
  }
});
```
This is the routing layer. `ListToolsRequestSchema` responses tell Claude what tools are available. `CallToolRequestSchema` responses handle actual tool calls. The `switch` statement routes by tool name. All errors are caught and returned as structured JSON with `isError: true`.

**Section 8: Start the server**
```typescript
async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error("Household Knowledge Base MCP Server running on stdio");
}

main().catch((error) => {
  console.error("Fatal error in main():", error);
  process.exit(1);
});
```
`StdioServerTransport` means the server communicates via stdin/stdout. When Claude Desktop starts this process, it sends MCP messages via stdin and reads responses from stdout. `console.error` (not `console.log`) is used deliberately — stdout is reserved for MCP protocol messages, so all server logging must go to stderr.

### 4.5 How Capture Works

When you tell Claude to "remember this: Sarah is thinking about leaving her job":

1. Claude calls the `capture_thought` tool on the core Open Brain MCP server
2. The Edge Function receives the tool call via HTTP
3. The function calls OpenRouter for two things **in parallel** (using `Promise.all`):
   - Generate a 1,536-dimensional embedding of the text
   - Ask an LLM to extract structured metadata (type, topics, people, action items)
4. Both results come back (in whichever order they finish)
5. The function does one `INSERT` into the `thoughts` table with both the embedding and metadata
6. Returns a confirmation to Claude

The parallel execution (step 3) is important — embedding and metadata extraction are independent operations, so running them simultaneously halves the latency compared to running them sequentially.

### 4.6 How Search Works

When you ask Claude "What did I capture about career changes?":

1. Claude calls the `search_thoughts` tool
2. The Edge Function receives the tool call
3. The function calls OpenRouter to generate an embedding of the query string "career changes"
4. The function calls `match_thoughts(query_embedding, 0.7, 10)` via `supabase.rpc()`
5. PostgreSQL uses the HNSW index to find the nearest vectors to the query vector
6. Results are returned sorted by similarity (highest similarity first)
7. Claude receives the results and formats them as a human-readable answer

The HNSW index (defined during table creation) is why step 5 is fast even with thousands of stored thoughts.

---

## 5. How to Build a New Extension

Extensions are the most complex contribution type. They require approval before you start building — open an issue at the repository to discuss your idea first. Once approved, follow these steps exactly.

### Step 1: Design Your Schema

Before writing any code, design your database tables on paper (or in a doc).

Ask yourself:
- What data does this extension need to store?
- What are the natural entities? (People, items, events, records?)
- What relationships exist between entities? (One-to-many, many-to-many?)
- What queries will be most common? (By user? By date? By category?)
- Which tables need Row Level Security?

**Schema design checklist:**
- Every table must have `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
- Every table that stores user data must have `user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL`
- Use `TIMESTAMPTZ` for all timestamp columns
- Use `JSONB` for flexible metadata (things that vary per row)
- Use `TEXT[]` for arrays of strings (tags, lists)
- Add indexes for every column combination you will filter/sort by
- Enable RLS on every table that stores user data

**Example: Designing a reading list schema**
```
Entities:
- books (title, author, isbn, cover_url)
- reading_sessions (book_id, started_at, finished_at, pages_read)
- book_notes (book_id, content, page_number)

Queries I need:
- All books by user (filter by user_id)
- Books by status (reading, finished, want-to-read)
- Notes for a specific book (filter by book_id)
- Recent reading sessions (order by started_at DESC)

Indexes needed:
- (user_id, status) on books
- (book_id, started_at) on reading_sessions
- (book_id) on book_notes
```

### Step 2: Create schema.sql

Create `extensions/your-extension-name/schema.sql`. Follow the exact pattern from existing extensions.

**Template:**
```sql
-- Extension N: Your Extension Name
-- Brief description of what these tables store

-- Table: your_main_table
CREATE TABLE IF NOT EXISTS your_main_table (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
    -- Your columns here
    name TEXT NOT NULL,
    category TEXT,
    details JSONB DEFAULT '{}',
    notes TEXT,
    created_at TIMESTAMPTZ DEFAULT now() NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT now() NOT NULL
);

-- Indexes for efficient querying
CREATE INDEX IF NOT EXISTS idx_your_main_table_user_category
    ON your_main_table(user_id, category);

-- Row Level Security
ALTER TABLE your_main_table ENABLE ROW LEVEL SECURITY;

CREATE POLICY your_main_table_user_policy ON your_main_table
    FOR ALL
    USING (auth.uid() = user_id)
    WITH CHECK (auth.uid() = user_id);

-- Function to automatically update updated_at timestamp
-- (Only needed once per database — reuse if already exists)
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Trigger for updated_at
DROP TRIGGER IF EXISTS update_your_main_table_updated_at ON your_main_table;
CREATE TRIGGER update_your_main_table_updated_at
    BEFORE UPDATE ON your_main_table
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

**Important rules for schema.sql:**
- Never use `DROP TABLE` — use `CREATE TABLE IF NOT EXISTS`
- Never use `TRUNCATE` — not allowed
- Never `ALTER TABLE thoughts DROP COLUMN` or `ALTER TABLE thoughts ALTER COLUMN` — adding columns is fine, modifying or dropping existing ones is not
- Never use unqualified `DELETE FROM table_name` without a `WHERE` clause
- Always use `CREATE INDEX IF NOT EXISTS` to avoid errors on re-run
- Always enable RLS with `ALTER TABLE ... ENABLE ROW LEVEL SECURITY`

### Step 3: Write the MCP Server (index.ts)

This is the main development task. Create `extensions/your-extension-name/index.ts`.

**Complete template with all sections:**

```typescript
#!/usr/bin/env node

/**
 * Extension N: Your Extension Name MCP Server
 *
 * Brief description of what tools this server provides.
 */

// ─── Section 1: Imports ──────────────────────────────────────────────────────

import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
  Tool,
} from "@modelcontextprotocol/sdk/types.js";
import { createClient, SupabaseClient } from "@supabase/supabase-js";

// ─── Section 2: Environment Validation ───────────────────────────────────────

const SUPABASE_URL = process.env.SUPABASE_URL;
const SUPABASE_SERVICE_ROLE_KEY = process.env.SUPABASE_SERVICE_ROLE_KEY;

if (!SUPABASE_URL || !SUPABASE_SERVICE_ROLE_KEY) {
  console.error("Error: SUPABASE_URL and SUPABASE_SERVICE_ROLE_KEY must be set");
  process.exit(1);
}

// ─── Section 3: Supabase Client Initialization ───────────────────────────────

const supabase: SupabaseClient = createClient(
  SUPABASE_URL,
  SUPABASE_SERVICE_ROLE_KEY,
  {
    auth: {
      autoRefreshToken: false,
      persistSession: false,
    },
  }
);

// ─── Section 4: Type Definitions ─────────────────────────────────────────────

interface YourMainEntity {
  id: string;
  user_id: string;
  name: string;
  category: string | null;
  details: Record<string, any>;
  notes: string | null;
  created_at: string;
  updated_at: string;
}

// ─── Section 5: Tool Definitions ─────────────────────────────────────────────

const TOOLS: Tool[] = [
  {
    name: "add_item",
    description: "One sentence description of what this tool does. Be specific about what gets stored.",
    inputSchema: {
      type: "object",
      properties: {
        user_id: {
          type: "string",
          description: "User ID (UUID) - typically auth.uid()",
        },
        name: {
          type: "string",
          description: "Name or description of the item",
        },
        category: {
          type: "string",
          description: "Optional category for filtering",
        },
        // Add all your parameters here
      },
      required: ["user_id", "name"],  // List required fields
    },
  },
  {
    name: "search_items",
    description: "Search items by query text, with optional filters.",
    inputSchema: {
      type: "object",
      properties: {
        user_id: {
          type: "string",
          description: "User ID (UUID)",
        },
        query: {
          type: "string",
          description: "Search term",
        },
      },
      required: ["user_id"],
    },
  },
];

// ─── Section 6: Tool Handlers ─────────────────────────────────────────────────

async function handleAddItem(args: any): Promise<string> {
  const { user_id, name, category, details, notes } = args;

  const { data, error } = await supabase
    .from("your_main_table")
    .insert({
      user_id,
      name,
      category: category || null,
      details: details || {},
      notes: notes || null,
    })
    .select()
    .single();

  if (error) {
    throw new Error(`Failed to add item: ${error.message}`);
  }

  return JSON.stringify({
    success: true,
    message: `Added item: ${name}`,
    item: data,
  }, null, 2);
}

async function handleSearchItems(args: any): Promise<string> {
  const { user_id, query, category } = args;

  let queryBuilder = supabase
    .from("your_main_table")
    .select("*")
    .eq("user_id", user_id);

  if (category) {
    queryBuilder = queryBuilder.ilike("category", `%${category}%`);
  }

  if (query) {
    queryBuilder = queryBuilder.or(
      `name.ilike.%${query}%,notes.ilike.%${query}%`
    );
  }

  const { data, error } = await queryBuilder.order("created_at", { ascending: false });

  if (error) {
    throw new Error(`Failed to search items: ${error.message}`);
  }

  return JSON.stringify({
    success: true,
    count: data.length,
    items: data,
  }, null, 2);
}

// ─── Section 7: Server Setup ──────────────────────────────────────────────────

const server = new Server(
  {
    name: "your-extension-name",
    version: "1.0.0",
  },
  {
    capabilities: {
      tools: {},
    },
  }
);

// ─── Section 8: Request Handler Registration ─────────────────────────────────

server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: TOOLS,
}));

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  try {
    switch (name) {
      case "add_item":
        return { content: [{ type: "text", text: await handleAddItem(args) }] };
      case "search_items":
        return { content: [{ type: "text", text: await handleSearchItems(args) }] };
      default:
        throw new Error(`Unknown tool: ${name}`);
    }
  } catch (error) {
    const errorMessage = error instanceof Error ? error.message : String(error);
    return {
      content: [{ type: "text", text: JSON.stringify({ success: false, error: errorMessage }) }],
      isError: true,
    };
  }
});

// ─── Section 9: Entry Point ───────────────────────────────────────────────────

async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error("Your Extension Name MCP Server running on stdio");
}

main().catch((error) => {
  console.error("Fatal error in main():", error);
  process.exit(1);
});
```

### Step 4: Create package.json

Create `extensions/your-extension-name/package.json`:

```json
{
  "name": "your-extension-name-mcp",
  "version": "1.0.0",
  "description": "MCP server for [what it does]",
  "type": "module",
  "main": "build/index.js",
  "bin": {
    "your-extension-name-mcp": "build/index.js"
  },
  "scripts": {
    "build": "tsc",
    "watch": "tsc --watch",
    "prepare": "npm run build"
  },
  "keywords": ["mcp", "your-keyword"],
  "author": "Your Name",
  "license": "MIT",
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.0.4",
    "@supabase/supabase-js": "^2.48.1"
  },
  "devDependencies": {
    "@types/node": "^22.10.5",
    "typescript": "^5.7.3"
  },
  "engines": {
    "node": ">=18.0.0"
  }
}
```

The `"type": "module"` field is critical — it tells Node.js to treat `.js` files as ES modules (the modern JavaScript module system). Without it, the imports from the MCP SDK will not work.

### Step 5: Create tsconfig.json

Copy this exactly — it is standard for all extensions:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "outDir": "./build",
    "rootDir": "./",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["*.ts"],
  "exclude": ["node_modules", "build"]
}
```

### Step 6: Create metadata.json

Create `extensions/your-extension-name/metadata.json`:

```json
{
  "name": "Your Extension Name",
  "description": "One to two sentences describing what this extension adds to Open Brain.",
  "category": "extensions",
  "author": {
    "name": "Your Full Name",
    "github": "your-github-username"
  },
  "version": "1.0.0",
  "requires": {
    "open_brain": true,
    "services": [],
    "tools": ["Node.js 18+"]
  },
  "requires_primitives": [],
  "learning_order": 7,
  "tags": ["your-tag", "another-tag"],
  "difficulty": "beginner",
  "estimated_time": "45 minutes",
  "created": "2026-03-13",
  "updated": "2026-03-13"
}
```

**Required fields** (the automated review checks all of these):
- `name` — Human-readable name
- `description` — What it does
- `category` — Must be `"extensions"` for an extension
- `author.name` — Your name
- `version` — Semantic version (`major.minor.patch`)
- `requires.open_brain` — Must be `true`
- `tags` — At least one tag
- `difficulty` — Must be exactly `"beginner"`, `"intermediate"`, or `"advanced"`
- `estimated_time` — How long setup takes

If your extension uses RLS or shared MCP patterns, add them to `requires_primitives`:
```json
"requires_primitives": ["rls", "shared-mcp"]
```

### Step 7: Write the README.md

The README is what users read to set up your extension. It must contain these sections to pass the automated review:

**Required sections (automated check looks for these):**
- A section with the word "Prerequisites" (exact word, case-insensitive)
- Numbered steps (`1.`, `2.`, `3.`, etc.) for step-by-step instructions
- A section mentioning "Expected outcome" or "Expected" or "Result"

**Extensions additionally require** (human review checks these):
- "Why This Matters" section — the human problem this solves
- "Learning Path" table — where this sits in the 6-extension sequence
- "What You'll Learn" section — new concepts introduced
- "Cross-Extension Integration" section — how this connects to other extensions
- "Next Steps" section — link to the next extension

**README template for extensions:**

```markdown
# Extension N: Your Extension Name

## Why This Matters

[2-3 sentences about the human problem. Why does someone need this?
What are they doing today without it? What friction does this remove?]

## Learning Path

| Extension | What You Build | Status |
|-----------|----------------|--------|
| 1 | Household Knowledge Base | Complete first |
| 2 | Home Maintenance Tracker | Complete first |
| ... | ... | ... |
| N | This Extension | YOU ARE HERE |

## What You'll Learn

- [New concept 1 introduced in this extension]
- [New concept 2]
- [New concept 3]

## Prerequisites

- Working Open Brain setup (follow docs/01-getting-started.md)
- Node.js 18+ installed
- [Any other specific prerequisites]

## What It Does

[1-2 sentences on what this extension adds to Open Brain.]

## Step-by-Step Setup

### Step 1: Run the Database Schema

1. Open your Supabase project dashboard
2. Go to SQL Editor → New query
3. Paste the contents of `schema.sql` and click Run
4. Verify the tables appear in Table Editor

### Step 2: Install and Build the MCP Server

1. Navigate to the extension directory:
   ```bash
   cd extensions/your-extension-name
   ```
2. Install dependencies:
   ```bash
   npm install
   ```
3. Build:
   ```bash
   npm run build
   ```

### Step 3: Connect to Claude Desktop

1. Open your Claude Desktop config file:
   - macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
   - Windows: `%APPDATA%\Claude\claude_desktop_config.json`
2. Add this to the `mcpServers` section:
   ```json
   {
     "mcpServers": {
       "your-extension-name": {
         "command": "node",
         "args": ["/absolute/path/to/extensions/your-extension-name/build/index.js"],
         "env": {
           "SUPABASE_URL": "your-supabase-url",
           "SUPABASE_SERVICE_ROLE_KEY": "your-service-role-key"
         }
       }
     }
   }
   ```
3. Restart Claude Desktop

## Expected Outcome

[Specific description of what should happen when setup is complete.
Include example prompts and what the AI response should look like.]

## Cross-Extension Integration

[Describe how this extension connects to other extensions, if applicable.
For example: "The CRM extension can link contacts to thoughts stored
in the core Open Brain thoughts table."]

## Troubleshooting

### Issue: [Common problem]
**Cause:** [Why it happens]
**Fix:** [How to resolve it]

### Issue: [Another common problem]
**Cause:** [Why it happens]
**Fix:** [How to resolve it]

## Next Steps

[Link to the next extension in the learning path, or describe what to build next.]
```

---

## 6. How to Build a New Recipe

Recipes are simpler than extensions. They are standalone capability adds — no ordering, no prerequisites beyond a working Open Brain.

### What a Recipe Needs

**Required files:**
- `README.md` — with Prerequisites section, numbered steps, and Expected Outcome section
- `metadata.json` — with all required fields
- Code file(s) — at least one `.sql`, `.ts`, `.js`, or `.py` file, OR three or more numbered steps in the README

**Folder structure:**
```
recipes/your-recipe-name/
├── README.md
├── metadata.json
├── your-main-script.py    # or .ts, .js, .sql — whatever makes sense
└── .env.example           # If your recipe needs environment variables
```

### Creating metadata.json for a Recipe

```json
{
  "name": "Your Recipe Name",
  "description": "What capability this adds to Open Brain in one sentence.",
  "category": "recipes",
  "author": {
    "name": "Your Name",
    "github": "your-github-username"
  },
  "version": "1.0.0",
  "requires": {
    "open_brain": true,
    "services": ["Gmail API"],
    "tools": ["Python 3.10+"]
  },
  "tags": ["email", "import"],
  "difficulty": "intermediate",
  "estimated_time": "30 minutes",
  "created": "2026-03-13",
  "updated": "2026-03-13"
}
```

### Example: A Python Recipe

Looking at `recipes/chatgpt-conversation-import/` as a model:
- `import-chatgpt.py` — the main Python script
- `requirements.txt` — Python dependencies (like `npm install` but for Python)
- `.env.example` — shows what environment variables are needed, with placeholder values (never real values)
- `.gitignore` — lists `.env` so the real credentials are never committed

**The `.env.example` pattern is important:**
```bash
# .env.example — commit this file
SUPABASE_URL=your-supabase-url-here
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key-here
OPENROUTER_API_KEY=your-openrouter-key-here

# .env — add this to .gitignore, never commit it
SUPABASE_URL=https://abcdefghijklmnop.supabase.co
SUPABASE_SERVICE_ROLE_KEY=eyJhbGciOiJIUzI1NiI...
```

### README for a Recipe

```markdown
# Your Recipe Name

## What It Does

[1-2 sentences. What capability does this add?]

## Prerequisites

- Working Open Brain setup
- [Other requirements: API keys, software, accounts]

## Step-by-Step Instructions

1. [First step]
2. [Second step]
3. [Third step — be specific, assume the reader has never done this before]

## Expected Outcome

[What should happen when this is working? Be specific.]

## Troubleshooting

### [Common issue]
[How to fix it]
```

---

## 7. How to Build a New Integration

Integrations are event-driven: something happens in an external system (a Slack message, a Discord post, a new email), and your code captures it into Open Brain.

### Understanding Edge Functions

Supabase Edge Functions are serverless functions — code that runs in response to an HTTP request without you managing a server. The important characteristics:

- They run on Deno (not Node.js) — use Deno import syntax
- They are deployed to Supabase's infrastructure — no server to manage
- They are invoked by HTTP POST requests
- They have access to your Supabase database automatically
- They can access secrets you set via `supabase secrets set`
- Cold start latency (first request after idle) can be 2-5 seconds
- Response time limit: 150 seconds (more than enough for embedding + metadata extraction)

**A minimal Edge Function skeleton:**

```typescript
// supabase/functions/your-function-name/index.ts
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";

// Environment variables (set via supabase secrets set)
const SUPABASE_URL = Deno.env.get("SUPABASE_URL")!;
const SUPABASE_SERVICE_ROLE_KEY = Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!;
const YOUR_API_KEY = Deno.env.get("YOUR_API_KEY")!;

const supabase = createClient(SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY);

// Deno.serve is the Deno equivalent of an Express.js server
Deno.serve(async (req: Request): Promise<Response> => {
  try {
    // Parse the incoming request
    const body = await req.json();

    // Handle webhook verification challenges (common in Slack, Discord, etc.)
    if (body.type === "url_verification") {
      return new Response(JSON.stringify({ challenge: body.challenge }), {
        headers: { "Content-Type": "application/json" },
      });
    }

    // Extract the message text from the webhook payload
    const messageText = body.event?.text;
    if (!messageText) {
      return new Response("ok", { status: 200 });
    }

    // Store in Supabase
    const { error } = await supabase.from("thoughts").insert({
      content: messageText,
      metadata: { source: "your-integration" },
    });

    if (error) {
      console.error("Insert error:", error);
      return new Response("error", { status: 500 });
    }

    return new Response("ok", { status: 200 });
  } catch (err) {
    console.error("Function error:", err);
    return new Response("error", { status: 500 });
  }
});
```

### Webhook Pattern

Most integrations use the webhook pattern:
1. External service (Slack, Discord, email) sends an HTTP POST to your Edge Function URL when something happens
2. Your Edge Function processes the payload
3. Your Edge Function stores the data in Supabase
4. Your Edge Function returns a 200 OK response

**The verification challenge pattern:**
Many webhook providers (Slack, Discord) send a verification request when you first set up the webhook. They send `{ "type": "url_verification", "challenge": "some-random-string" }` and expect you to respond with `{ "challenge": "some-random-string" }`. The code above handles this.

### Deploying an Edge Function

```bash
# One-time setup: log in and link your project
supabase login
supabase link --project-ref YOUR_PROJECT_REF

# Create a new function
supabase functions new your-function-name

# Set secrets
supabase secrets set YOUR_API_KEY=the-actual-key-value

# Deploy
supabase functions deploy your-function-name --no-verify-jwt

# View logs (for debugging)
supabase functions logs your-function-name --follow
```

The `--no-verify-jwt` flag means the Edge Function accepts requests without a JWT auth token. For webhook receivers, this is necessary because the webhook sender (Slack, Discord) does not have your JWT. You protect the endpoint with the service's own verification mechanism (signing secrets, challenge handshakes, etc.) instead.

### Integration Folder Structure

```
integrations/your-integration-name/
├── README.md       # Setup instructions (webhook configuration, secrets, testing)
├── metadata.json   # Contribution metadata
└── index.ts        # The Edge Function code (or embedded in README as copy-paste)
```

If the code is short enough to copy-paste from the README (like the Slack integration), you do not need a separate `.ts` file. If it is complex, include the file.

---

## 8. Testing Your Code

### Testing SQL in the Supabase SQL Editor

Before running your `schema.sql` against your database, test each statement individually:

1. Open your Supabase project dashboard
2. Go to SQL Editor → New query
3. Paste the `CREATE TABLE` statement and click Run
4. Look for errors in the output panel
5. Go to Table Editor and verify the table exists with the right columns
6. Test your indexes by running `EXPLAIN ANALYZE SELECT * FROM your_table WHERE user_id = 'test'` — look for "Index Scan" in the output (not "Seq Scan")
7. Test your RLS policies by running queries with different auth contexts

**Useful SQL verification queries:**
```sql
-- Check that your table exists and has RLS enabled
SELECT tablename, rowsecurity
FROM pg_tables
WHERE schemaname = 'public' AND tablename = 'your_table';

-- List all policies on your table
SELECT policyname, cmd, qual
FROM pg_policies
WHERE tablename = 'your_table';

-- List all indexes on your table
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'your_table';

-- Test that auth.uid() returns something (run while authenticated)
SELECT auth.uid();
```

### Testing MCP Tools via Claude Desktop

1. Build and start your MCP server
2. Add it to Claude Desktop's config and restart Claude Desktop
3. In a Claude Desktop conversation, verify the tools appear (check Settings → Developer or ask Claude "what tools do you have?")
4. Test each tool with a natural language prompt:
   - "Add a new [entity] called Test Item"
   - "Search for [entity]"
   - "Get details of [entity id]"
5. Check the Supabase Table Editor to verify the data was actually stored
6. Test error cases: call a tool with missing required fields and verify the error message is helpful

### Testing via Claude Code

If you prefer the command line:

```bash
# Start the MCP server in the background
node build/index.js &

# Use Claude Code with MCP to test
claude mcp add --transport stdio test-server node build/index.js
```

Or add the server to your Claude Code MCP config and run:
```
claude "Use the add_item tool to add a test item"
```

### Checking Edge Function Logs

For integrations that use Edge Functions:

```bash
# Stream logs in real time
supabase functions logs your-function-name --follow

# Or view in the dashboard:
# Supabase dashboard → Edge Functions → your-function-name → Logs
```

Logs show:
- `console.log()` / `console.error()` output from your function
- Request metadata (timestamp, status code, duration)
- Any uncaught errors

### Verifying RLS Policies

RLS bugs are subtle. The most common mistake is testing with the service role key (which bypasses RLS) and thinking RLS works, when in reality you have not tested with a real user token.

**Test with actual user authentication:**
```typescript
// Create a test script that uses a user JWT instead of service role
import { createClient } from "@supabase/supabase-js";

// Use the anon key (not service role) + a real user's JWT
const supabaseUser = createClient(SUPABASE_URL, SUPABASE_ANON_KEY, {
  auth: { persistSession: false }
});

// Sign in as a test user
const { data: { session } } = await supabaseUser.auth.signInWithPassword({
  email: "test@example.com",
  password: "test-password"
});

// Query using the user's session (RLS is now enforced)
const { data, error } = await supabaseUser
  .from("your_table")
  .select("*");

// This should only return rows where user_id matches the test user's UUID
```

Alternatively, use the Supabase SQL Editor's built-in user impersonation feature (Table Editor → select a table → preview with different user contexts).

---

## 9. The PR Review Process

### Branch Naming

Every contribution must be on its own branch:

```bash
# Format: contrib/<your-github-username>/<short-description>
git checkout -b contrib/yourname/add-reading-list-schema
git checkout -b contrib/yourname/fix-household-knowledge-readme
git checkout -b contrib/matthallett/email-import-recipe
```

Use lowercase with hyphens. Keep the description short (3-6 words).

### PR Title Format

PR titles must start with the category in brackets:

```
[recipes] Email history import via Gmail API
[schemas] CRM contacts table with interaction tracking
[extensions] Household Knowledge Base MCP server
[primitives] Row Level Security guide
[integrations] Discord message capture bot
[dashboards] Weekly review frontend template
[docs] Developer guide for contributors
```

The automated review will fail if the title does not match this pattern exactly.

### The 11 Automated Checks

When you open a PR, GitHub Actions runs `ob1-review.yml`. Here is what each check verifies and how to pass it:

**Check 1: Folder structure**
All changed files must be under `recipes/`, `schemas/`, `dashboards/`, `integrations/`, `primitives/`, `extensions/`, `docs/`, or `resources/`. You cannot touch root-level files like `README.md` or `CONTRIBUTING.md` in a contribution PR.

**Check 2: Required files**
Every contribution folder must contain both `README.md` and `metadata.json`. If either is missing, this check fails.

**Check 3: Metadata valid**
Your `metadata.json` must:
- Be valid JSON (no syntax errors — test with `jq . metadata.json`)
- Contain all required fields: `name`, `description`, `category`, `version`, `estimated_time`, `author.name`
- Have `requires.open_brain` set to `true`
- Have at least one tag in `tags`
- Have `difficulty` set to exactly `beginner`, `intermediate`, or `advanced`

**Check 4: No credentials**
The automated scan looks for patterns like:
- `sk-` followed by 20+ alphanumeric chars (OpenAI API key pattern)
- `AKIA` followed by 16 uppercase chars (AWS access key pattern)
- `ghp_` followed by 36 chars (GitHub personal access token)
- `xoxb-` or `xoxp-` (Slack bot/user tokens)
- `SUPABASE_SERVICE_ROLE_KEY` assigned to a value starting with `ey` (JWT)
- `.env` files with real values (not placeholders)

If any of these appear in any committed file, the check fails. Always use `.env.example` with placeholder values.

**Check 5: SQL safety**
SQL files cannot contain:
- `DROP TABLE` — use `CREATE TABLE IF NOT EXISTS` instead
- `DROP DATABASE` — never allowed
- `TRUNCATE` — never allowed
- `DELETE FROM your_table` without a `WHERE` clause
- `ALTER TABLE thoughts DROP COLUMN` or `ALTER TABLE thoughts ALTER COLUMN` — you can only add columns to the thoughts table, never modify or drop existing ones

**Check 6: Category-specific artifacts**
- `recipes/` — need code files (`.sql`, `.ts`, `.js`, `.py`) or 3+ numbered steps in README
- `schemas/` — need at least one `.sql` file
- `dashboards/` — need frontend code (`.html`, `.jsx`, `.tsx`, `.vue`, `.svelte`) or `package.json`
- `integrations/` — need code files (`.ts`, `.js`, `.py`)
- `primitives/` — README must be 200+ words
- `extensions/` — need both a `.sql` file and code files (`.ts`, `.js`, `.py`)

**Check 7: PR format**
Title must start with `[category]` followed by a space and description. The category must be one of: `recipes`, `schemas`, `dashboards`, `integrations`, `primitives`, `extensions`, `docs`.

**Check 8: No binary blobs**
- No files over 1MB
- No files with these extensions: `.exe`, `.dmg`, `.zip`, `.tar.gz`, `.tar.bz2`, `.rar`, `.7z`, `.msi`, `.pkg`, `.deb`, `.rpm`

If you need to reference a large file, link to it externally.

**Check 9: README completeness**
Your contribution's README must contain:
- The word "prerequisites" (case-insensitive) somewhere
- At least one numbered list item (`1.`, `2.`, etc.) — your step-by-step instructions
- The word "expected", "outcome", or "result" somewhere — your expected outcome section

**Check 10: Primitive dependency validation**
If your `metadata.json` declares `"requires_primitives": ["rls"]`:
- The directory `primitives/rls/` must exist in the repo
- Your README must contain a link to `primitives/rls` somewhere

If you declare a dependency on a primitive that does not exist, this check fails.

**Check 11: LLM clarity review**
Currently auto-passes (planned for v2). When implemented, it will use an LLM to check whether your README instructions are clear and complete.

**Additional automated checks (also run, not in the original 11):**

**Scope check:** Your PR should only change files inside your own contribution folder. If your recipe is in `recipes/my-recipe/`, you should not be modifying files in `recipes/some-other-recipe/`.

**Internal link validation:** If your README links to other files using relative paths (like `../../../docs/01-getting-started.md`), those files must actually exist.

### What to Expect After Opening a PR

1. Within a few minutes, the automated review runs and posts a comment on your PR
2. If any checks fail, fix the issues, push to your branch, and the checks run again automatically
3. If all 11 checks pass, a human admin reviews for quality, clarity, and safety
4. Human review typically takes 2-5 business days
5. The admin may request changes or ask questions via PR comments
6. Once approved, the admin merges the PR

### What Gets Rejected (Human Review)

Even if all automated checks pass, human review may reject for:
- **Contains credentials** that the automated scan missed
- **Requires paid services** with no free-tier alternative
- **Poorly documented** — instructions that skip steps or assume knowledge
- **Duplicates an existing contribution** without meaningful improvement
- **Modifies core infrastructure** — the `thoughts` table, the core MCP server
- **Quality issues** — code that does not work, patterns that are not idiomatic

---

## 10. Common Patterns and Best Practices

### The Standard MCP Server Boilerplate

Every extension `index.ts` follows this exact pattern. Do not deviate from it without a good reason:

```typescript
#!/usr/bin/env node

// 1. Imports — external packages first
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { CallToolRequestSchema, ListToolsRequestSchema, Tool } from "@modelcontextprotocol/sdk/types.js";
import { createClient, SupabaseClient } from "@supabase/supabase-js";

// 2. Environment validation — fail fast if config is wrong
const SUPABASE_URL = process.env.SUPABASE_URL;
const SUPABASE_SERVICE_ROLE_KEY = process.env.SUPABASE_SERVICE_ROLE_KEY;
if (!SUPABASE_URL || !SUPABASE_SERVICE_ROLE_KEY) {
  console.error("Error: SUPABASE_URL and SUPABASE_SERVICE_ROLE_KEY must be set");
  process.exit(1);
}

// 3. Single Supabase client instance (module-level singleton)
const supabase: SupabaseClient = createClient(SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, {
  auth: { autoRefreshToken: false, persistSession: false },
});

// 4. TypeScript interfaces for your database rows
interface YourEntity { ... }

// 5. TOOLS constant — the tool registry
const TOOLS: Tool[] = [ ... ];

// 6. Handler functions — one async function per tool
async function handleToolName(args: any): Promise<string> { ... }

// 7. Server instantiation
const server = new Server({ name: "your-server", version: "1.0.0" }, { capabilities: { tools: {} } });

// 8. Handler registration — list tools and route calls
server.setRequestHandler(ListToolsRequestSchema, async () => ({ tools: TOOLS }));
server.setRequestHandler(CallToolRequestSchema, async (request) => { ... });

// 9. Start function
async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error("Your Server running on stdio");
}
main().catch((error) => { console.error("Fatal error:", error); process.exit(1); });
```

### Supabase Client Initialization

Always initialize with `autoRefreshToken: false` and `persistSession: false` in server-side code. These options are for browser clients that need to maintain long-lived sessions. Server-side code makes short-lived requests and does not need them.

```typescript
// Correct for server-side (MCP servers, Edge Functions)
const supabase = createClient(SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, {
  auth: {
    autoRefreshToken: false,
    persistSession: false,
  },
});
```

### Error Handling in Tool Handlers

Always follow this pattern:
1. Throw errors from handler functions
2. Catch them in the request handler
3. Return structured JSON with `isError: true`

```typescript
// Handler: throw errors
async function handleMyTool(args: any): Promise<string> {
  const { data, error } = await supabase.from("table").select("*");

  if (error) {
    throw new Error(`Database query failed: ${error.message}`);
  }

  if (!data || data.length === 0) {
    throw new Error("No data found");
  }

  return JSON.stringify({ success: true, data });
}

// Request handler: catch and return as structured error
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  try {
    // ... tool routing ...
  } catch (error) {
    const errorMessage = error instanceof Error ? error.message : String(error);
    return {
      content: [{ type: "text", text: JSON.stringify({ success: false, error: errorMessage }) }],
      isError: true,
    };
  }
});
```

Never let errors bubble up uncaught from a request handler. Always return a structured response, even on failure. The AI client needs to receive a valid MCP response either way.

### JSONB Usage for Flexible Metadata

Use `JSONB` columns when the data has variable structure:

```sql
-- Good use of JSONB: each appliance has different specs
details JSONB DEFAULT '{}'

-- Example values:
-- Dishwasher: {"brand": "Bosch", "model": "SHPM65Z55N", "serial": "FD123"}
-- Paint: {"brand": "Sherwin Williams", "color": "Sea Salt", "code": "SW 6204"}
```

In TypeScript, treat JSONB columns as `Record<string, any>`:
```typescript
interface HouseholdItem {
  details: Record<string, any>;  // Flexible — different items have different keys
}
```

Do not use JSONB for data you will filter on frequently. Use proper typed columns for anything you search/sort by, and JSONB only for the "everything else" bag.

### When to Use RLS vs Service Role Key

**Service role key (bypasses RLS):** Use when:
- Your code runs on a server you control (MCP servers, Edge Functions)
- You handle access control in your code (filtering by `user_id` parameter)
- You need to access data for multiple users in one query

**RLS + user JWT (enforces RLS):** Use when:
- You want the database to enforce access rules automatically
- You are building a frontend that queries Supabase directly
- You have a multi-user system where each user should see only their own data

In Open Brain extensions, the MCP server uses the service role key and manually filters by `user_id` in queries. RLS is enabled as a defense-in-depth measure, but the primary enforcement is in the code.

```typescript
// Primary enforcement: explicit user_id filter in code
const { data, error } = await supabase
  .from("household_items")
  .select("*")
  .eq("user_id", user_id);  // This ensures users only get their own data

// Secondary enforcement: RLS policy also checks auth.uid() = user_id
// (This catches edge cases if code filter is accidentally omitted)
```

### Index Design for Common Query Patterns

**Composite indexes (most important pattern):**
```sql
-- You almost always filter by user first, then by another column
-- Create a composite index: (user_id, other_column)
CREATE INDEX IF NOT EXISTS idx_household_items_user_category
    ON household_items(user_id, category);

-- This allows efficient: WHERE user_id = ? AND category = ?
-- And also: WHERE user_id = ?  (leading column is still used)
-- But NOT: WHERE category = ?  (non-leading column alone is not efficient)
```

**GIN indexes for arrays and JSONB:**
```sql
-- For TEXT[] columns that you query with array operators
CREATE INDEX IF NOT EXISTS idx_recipes_user_tags
    ON recipes USING GIN (tags);

-- For JSONB columns that you query with containment operators (@>)
CREATE INDEX ON thoughts USING GIN (metadata);
```

**Partial indexes for sparse data:**
```sql
-- Only index rows where follow_up_date is set (a small fraction of contacts)
CREATE INDEX IF NOT EXISTS idx_professional_contacts_follow_up
    ON professional_contacts(user_id, follow_up_date)
    WHERE follow_up_date IS NOT NULL;
```

**Always add an index for:**
- `user_id` alone (single-column, if you ever do `WHERE user_id = ?`)
- `(user_id, field)` for any field you filter by after user
- Foreign keys (`contact_id`, `recipe_id`, etc.) — used in JOINs and cascades
- `created_at DESC` if you ORDER BY recency

---

## 11. Debugging Guide

### How to Read Edge Function Logs

Logs from Edge Functions appear in two places:

**1. Supabase Dashboard:**
- Go to your Supabase project
- Click "Edge Functions" in the left sidebar
- Click your function name
- Click the "Logs" tab
- Logs show: timestamp, log level, message

**2. Supabase CLI:**
```bash
# Follow logs in real time (like tail -f)
supabase functions logs your-function-name --follow

# Show last 100 lines
supabase functions logs your-function-name --limit 100
```

**What you will see:**
```
2026-03-13T14:23:01.123Z  INFO  Function started
2026-03-13T14:23:01.456Z  ERROR  Supabase insert error: duplicate key value
2026-03-13T14:23:01.460Z  INFO  Function completed in 337ms
```

`console.log()` appears as INFO, `console.error()` as ERROR.

**Important:** Edge Function stdout is reserved for HTTP responses. Use `console.error()` for debug logging in Edge Functions (same reason you use `console.error()` in MCP server main functions — stdout has a specific purpose).

### Common Errors and Their Fixes

**"SUPABASE_URL and SUPABASE_SERVICE_ROLE_KEY must be set"**
The MCP server exits immediately on startup because environment variables are missing.
Fix: Add the environment variables to your Claude Desktop config's `env` section, or set them in your terminal before running the server manually.

**"relation 'your_table' does not exist"**
The table was not created. You may have run the schema in the wrong Supabase project.
Fix: Open Supabase Dashboard → SQL Editor → run your schema.sql. Verify in Table Editor that the table now exists.

**"permission denied for table your_table"**
RLS is blocking access, or the service role key is not being used correctly.
Fix: Verify your `SUPABASE_SERVICE_ROLE_KEY` is the secret key, not the anon key. Check that the Supabase client is initialized with the service role key. Run `SELECT auth.uid();` in Supabase SQL Editor to see what role the query runs as.

**"Unknown tool: tool_name"**
Claude tried to call a tool that does not exist in your `TOOLS` array, or you misspelled the tool name in the switch statement.
Fix: Check that the `name` in the tool definition in `TOOLS` exactly matches the `case` string in the switch statement. Check that Claude knows the correct tool name (restart Claude Desktop to refresh the tool list).

**"Failed to parse JSON" or "Unexpected token"**
Your tool handler returned malformed JSON, or the metadata.json file has a syntax error.
Fix: Validate JSON with `jq . metadata.json` (command line) or an online JSON validator. For tool handler output, make sure `JSON.stringify()` is called correctly.

**TypeScript compilation errors:**

```
error TS2339: Property 'xyz' does not exist on type 'YourInterface'
```
You tried to access a property that is not defined in your TypeScript interface. Add it to the interface, or change how you access it.

```
error TS2345: Argument of type 'string | null' is not assignable to parameter of type 'string'
```
A function expects a non-null string but you are passing something that might be null. Fix with a null check: `if (value !== null) { ... }` or use a default: `value ?? ""`.

```
error TS2307: Cannot find module '@supabase/supabase-js'
```
The package is not installed. Run `npm install` in the extension directory.

**"MCP server failed to start" in Claude Desktop**

This error appears in Claude Desktop when the MCP server process crashes on startup.

1. Test the server manually:
   ```bash
   SUPABASE_URL="your-url" SUPABASE_SERVICE_ROLE_KEY="your-key" node build/index.js
   ```
2. If it exits immediately, read the error output — it will tell you what is wrong
3. Check Claude Desktop logs:
   - macOS: `~/Library/Logs/Claude/mcp*.log`
   - Windows: `%APPDATA%\Claude\logs\`
4. Verify the path in your config file points to the compiled `build/index.js`, not `index.ts`
5. Verify Node.js is accessible at the path used in the config (`"command": "node"`)

### How to Debug MCP Connection Issues

**Step 1:** Verify the server starts manually (see above).

**Step 2:** Check Claude Desktop config syntax:
```bash
# On macOS
cat ~/Library/Application\ Support/Claude/claude_desktop_config.json | jq .
# If jq prints an error, your JSON is malformed
```

**Step 3:** Verify paths are absolute. Relative paths do not work in Claude Desktop config.

**Step 4:** After changing the config, fully quit and restart Claude Desktop. On macOS, right-click the Dock icon → Quit. On Windows, right-click the system tray icon → Quit.

**Step 5:** Check that tools are available. In a new Claude Desktop conversation, ask: "What MCP tools do you have available?" or "List your tools." If the server is connected but the tools do not appear, the server's `ListToolsRequestSchema` handler may not be returning the expected format.

**Step 6:** For the core Open Brain MCP server (HTTP-based), verify the URL is correct:
```bash
curl "https://YOUR_PROJECT_REF.supabase.co/functions/v1/open-brain-mcp?key=your-access-key" \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","params":{},"id":1}'
```
A valid response will be a JSON object. A 401 means the access key is wrong. A 404 means the function is not deployed.

---

## 12. Glossary

**anon key (Supabase):** A public API key that respects Row Level Security policies. Safe to use in client-side (browser) code. Not to be confused with the service role key, which bypasses RLS.

**async/await:** JavaScript/TypeScript syntax for handling asynchronous operations. `async` before a function makes it return a Promise. `await` inside an async function pauses execution until the awaited Promise resolves. Conceptually similar to Java's `Future.get()` but without blocking a thread.

**CONTRIBUTING.md:** The source-of-truth document for how to contribute to Open Brain. Read it before submitting anything.

**cosine similarity:** A measure of similarity between two vectors, based on the angle between them. Values range from -1 (opposite directions) to 1 (same direction). For embedding vectors, values near 1 mean similar meaning; values near 0 mean unrelated. Used by `match_thoughts()`.

**`<=>` operator:** The pgvector cosine distance operator. Lower values mean more similar. `1 - (a <=> b)` gives cosine similarity.

**Deno:** An alternative JavaScript/TypeScript runtime. Used by Supabase Edge Functions. Key difference from Node.js: imports use URLs instead of package names, and `Deno.env.get()` instead of `process.env`.

**Edge Function:** A serverless function deployed to Supabase's infrastructure. Runs on Deno. Invoked by HTTP requests. Scales to zero (costs nothing when idle). Used for the core Open Brain MCP server, the Slack capture integration, and other event-driven integrations.

**embedding:** A high-dimensional floating-point vector that represents the semantic meaning of a piece of text. Produced by a machine learning model (in Open Brain: `text-embedding-3-small` via OpenRouter). Similar texts produce similar vectors. Stored in the `embedding` column of the `thoughts` table.

**ESM (ES Modules):** The modern JavaScript module system. Uses `import`/`export` syntax instead of the older `require()`/`module.exports`. All Open Brain TypeScript code uses ESM. The `"type": "module"` in `package.json` enables it.

**Extension:** One of the 6 curated, ordered contributions in `extensions/`. Each teaches new concepts through something practical. Build them in order. They include `schema.sql`, `index.ts`, and supporting files.

**HNSW (Hierarchical Navigable Small World):** The index type used for pgvector. A graph-based index that enables fast approximate nearest-neighbor search. Created by `USING hnsw (embedding vector_cosine_ops)` in the `CREATE INDEX` statement.

**Integration:** A contribution in `integrations/` that connects an external service (Slack, Discord, email, etc.) to Open Brain. Typically an Edge Function webhook receiver.

**JSONB:** PostgreSQL's binary JSON column type. Stores arbitrary JSON, supports indexing and query operators. Used for flexible metadata in the `thoughts` table and in extension tables.

**JWT (JSON Web Token):** A signed, encoded token that proves identity. Supabase uses JWTs for authentication. The service role key is a JWT. The anon key is a JWT. User session tokens are JWTs.

**MCP (Model Context Protocol):** An open protocol from Anthropic that defines how AI assistants call external tools. An MCP server exposes named tools with typed inputs; an MCP client (Claude Desktop, Claude Code, etc.) calls them.

**metadata.json:** The structured metadata file required for every contribution. Validated by the automated review against `.github/metadata.schema.json`.

**`node_modules/`:** The directory where npm downloads packages. Never commit this. Always add to `.gitignore`.

**npm:** Node Package Manager. Ships with Node.js. Manages JavaScript dependencies. Reads `package.json`. Equivalent to Maven (Java) or pip (Python).

**`package.json`:** The npm manifest file. Lists the project name, version, scripts, and dependencies. Every extension has one.

**pgvector:** A PostgreSQL extension that adds a `vector` data type and similarity search operations (`<=>` distance operator). Enables semantic search over embeddings.

**Primitive:** A reusable concept guide in `primitives/` that explains a pattern used by multiple extensions. Currently: `rls/` (Row Level Security) and `shared-mcp/` (shared access MCP servers).

**Promise:** A JavaScript object representing the eventual completion (or failure) of an asynchronous operation. Equivalent to Java's `CompletableFuture` or C#'s `Task`. Use `await` to wait for a Promise to resolve.

**Recipe:** A standalone contribution in `recipes/` that adds a specific capability to Open Brain. Not ordered, no learning path. Open for community contributions.

**RLS (Row Level Security):** A PostgreSQL feature that enforces access control at the row level based on the current database user. Enabled with `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` and configured with `CREATE POLICY` statements. Used to ensure users can only see their own data.

**Service role key:** A Supabase credential (also called the "secret key") that gives full admin access to your database, bypassing all RLS policies. Keep this private. Use only in server-side code (MCP servers, Edge Functions). Never put it in client-side JavaScript or commit it to git.

**`stdio` transport:** The MCP transport mode used by extension servers. Claude Desktop starts the MCP server as a subprocess and communicates with it via stdin/stdout pipe. Contrast with HTTP transport (used by the core Open Brain Edge Function).

**Supabase:** A hosted PostgreSQL platform that bundles database, auto-generated REST API, authentication, Edge Functions, and a web dashboard. The infrastructure layer that Open Brain runs on.

**thoughts table:** The core database table in Open Brain. Stores all captured knowledge as text (`content`), a semantic embedding (`embedding`), and flexible metadata (`metadata`). Never alter or drop its existing columns.

**`tsconfig.json`:** The TypeScript compiler configuration file. Defines how TypeScript source files are compiled to JavaScript output.

**TypeScript:** A typed superset of JavaScript. Compiled to JavaScript before execution. Adds interfaces, type annotations, and compile-time type checking. Used for all MCP server code in this project.

**UUID (Universally Unique Identifier):** A 128-bit random identifier. Format: `550e8400-e29b-41d4-a716-446655440000`. Generated by PostgreSQL's `gen_random_uuid()`. Used as primary keys for all tables. Never set manually.

**vector:** An ordered array of floating-point numbers. In Open Brain, vectors have 1,536 dimensions (one for each output of the `text-embedding-3-small` model). Stored in `vector(1536)` columns. Used for semantic search.

**`vector(1536)`:** A pgvector column type storing a 1,536-element float array. The dimension must match the embedding model. If you change models, the dimensions must match.

**webhook:** An HTTP callback pattern where an external service (Slack, Discord, GitHub) sends a POST request to your server URL when an event occurs. Your Edge Function receives these requests, processes them, and stores the result in Supabase.

---

*This guide was written for Open Brain contributors as of March 2026. As the project evolves, some details may become outdated. If you find an error or gap, submit a `[docs]` PR to fix it.*
