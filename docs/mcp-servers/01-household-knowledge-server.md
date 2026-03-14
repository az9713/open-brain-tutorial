# Extension 1: Household Knowledge Base MCP Server

**Learning path position:** 1 of 6
**Difficulty:** Beginner
**Source:** `extensions/household-knowledge/`

---

## Overview

The Household Knowledge Base is the first extension in the Open Brain learning path. Its purpose is deliberately narrow: give Claude a place to store and retrieve household facts that would otherwise disappear at the end of every conversation.

Typical data stored here:

- Paint colors (brand, color name, hex code, rooms painted)
- Appliances (brand, model number, serial number, purchase date, warranty expiry)
- Measurements (window dimensions, ceiling height, room square footage)
- Documents (insurance policy numbers, HOA contact details, warranty references)
- Vendor contacts (plumber, electrician, landscaper, HVAC technician)

The server exposes **5 MCP tools** backed by **2 database tables**. It runs locally as a child process of Claude Desktop over stdio, and it connects outbound to Supabase over HTTPS.

This extension is intentionally standalone. It introduces the foundational patterns that every subsequent extension builds on: environment-validated startup, a Supabase client configured for server-side use, JSONB for flexible metadata, ILIKE text search, and user-scoped row-level security.

---

## Role in the Architecture

### Where it sits

```
+------------------+       stdio        +----------------------------+
|  Claude Desktop  | <----------------> |  household-knowledge       |
|  (MCP client)    |   (JSON-RPC 2.0)   |  MCP Server (Node.js)      |
+------------------+                   +----------------------------+
                                                    |
                                                    | HTTPS (REST)
                                                    |
                                        +-----------v----------------+
                                        |  Supabase                  |
                                        |  PostgreSQL + pgvector     |
                                        |                            |
                                        |  household_items           |
                                        |  household_vendors         |
                                        +-----------+----------------+
                                                    |
                                        +-----------v----------------+
                                        |  auth.users (built-in)     |
                                        |  RLS policies enforce      |
                                        |  user_id isolation         |
                                        +----------------------------+
```

### Key architectural facts

**stdio transport.** The server is launched by Claude Desktop as a child process. Communication is line-delimited JSON-RPC 2.0 over stdin/stdout. There is no network port, no HTTP server, and no authentication layer between Claude and the MCP server — the OS process boundary is the trust boundary.

**Service role key.** The Supabase client is initialized with `SUPABASE_SERVICE_ROLE_KEY`, which bypasses RLS. User isolation is instead enforced in application logic: every query includes an explicit `.eq("user_id", user_id)` filter, and the `user_id` is passed in as a tool argument. This is a deliberate teaching choice — it keeps the queries readable without adding Supabase Auth session management.

**Separate from the core MCP server.** Open Brain's primary `thoughts` table lives in a different server. This extension runs in its own process with its own Supabase client instance. The two servers share the same Supabase project but operate independently.

---

## Database Schema

Both tables are defined in `extensions/household-knowledge/schema.sql`.

### `household_items` (lines 6–16)

```sql
CREATE TABLE IF NOT EXISTS household_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
    name TEXT NOT NULL,
    category TEXT,
    location TEXT,
    details JSONB DEFAULT '{}',
    notes TEXT,
    created_at TIMESTAMPTZ DEFAULT now() NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT now() NOT NULL
);
```

Column breakdown:

| Column | Type | Constraints | Purpose |
|--------|------|-------------|---------|
| `id` | UUID | PK, `gen_random_uuid()` | Stable row identifier passed back to Claude for follow-up lookups |
| `user_id` | UUID | FK to `auth.users`, NOT NULL, CASCADE | Ties the row to a Supabase user; cascade delete removes all items if the user account is deleted |
| `name` | TEXT | NOT NULL | Human-readable label: "Living Room Paint", "Dishwasher", "Master Bedroom Window Width" |
| `category` | TEXT | nullable | Free-form grouping: `paint`, `appliance`, `measurement`, `document` |
| `location` | TEXT | nullable | Where in the home: "Kitchen", "Master Bedroom", "Garage" |
| `details` | JSONB | DEFAULT `'{}'` | Schema-less metadata bucket — see below |
| `notes` | TEXT | nullable | Freeform prose: purchase story, when last repainted, reminder to buy two cans |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT `now()` | Set on insert, never updated |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT `now()` | Maintained by trigger (see below) |

**The `details` JSONB field** is the most important design choice in this table. Rather than adding a column for every possible attribute an item might have, all structured metadata is packed into a single JSONB object. This lets the schema stay stable across wildly different item types:

```json
// Paint
{
  "brand": "Sherwin Williams",
  "color": "Sea Salt",
  "code": "SW 6204"
}

// Appliance
{
  "brand": "Bosch",
  "model": "SHPM65Z55N",
  "serial": "FD12345678",
  "purchase_date": "2024-06-15"
}

// Measurement
{
  "width_inches": 36,
  "height_inches": 48,
  "type": "double-hung"
}
```

The tradeoff is that JSONB keys are not indexed by default and cannot be enforced by `NOT NULL` constraints. For this use case that is acceptable — the data is queried by `name`, `category`, and `location`, not by fields inside `details`.

### `household_vendors` (lines 20–32)

```sql
CREATE TABLE IF NOT EXISTS household_vendors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
    name TEXT NOT NULL,
    service_type TEXT,
    phone TEXT,
    email TEXT,
    website TEXT,
    notes TEXT,
    rating INTEGER CHECK (rating >= 1 AND rating <= 5),
    last_used DATE,
    created_at TIMESTAMPTZ DEFAULT now() NOT NULL
);
```

Column breakdown:

| Column | Type | Constraints | Purpose |
|--------|------|-------------|---------|
| `id` | UUID | PK | Row identifier |
| `user_id` | UUID | FK, NOT NULL | User isolation |
| `name` | TEXT | NOT NULL | Vendor business name |
| `service_type` | TEXT | nullable | Trade: `plumber`, `electrician`, `landscaper`, `HVAC` |
| `phone` | TEXT | nullable | Stored as TEXT — no normalization, handles international formats |
| `email` | TEXT | nullable | Contact email |
| `website` | TEXT | nullable | URL |
| `notes` | TEXT | nullable | Any context: "only does emergency calls on weekends", "gave 10% discount" |
| `rating` | INTEGER | CHECK 1–5 | Database-enforced 5-star rating. Rejects 0 and 6+ at the constraint level |
| `last_used` | DATE | nullable | Calendar date, not timestamp — tracks recency without false precision |
| `created_at` | TIMESTAMPTZ | NOT NULL | Insert timestamp |

Note that `household_vendors` does not have an `updated_at` column. This is intentional — the trigger and function defined later in the schema only target `household_items`. If you need to track vendor edits, add the column and extend the trigger.

### Indexes (lines 35–39)

```sql
CREATE INDEX IF NOT EXISTS idx_household_items_user_category
    ON household_items(user_id, category);

CREATE INDEX IF NOT EXISTS idx_household_vendors_user_service
    ON household_vendors(user_id, service_type);
```

Both indexes are composite, leading with `user_id`. This matches the query pattern in every handler: all queries start with `.eq("user_id", user_id)`, so the planner uses the index to narrow to the user's rows before applying any secondary filter. Without the `user_id` prefix, a `category` index would scan all users' rows before filtering — inefficient in a multi-tenant database.

### Row-Level Security (lines 42–56)

```sql
ALTER TABLE household_items ENABLE ROW LEVEL SECURITY;
ALTER TABLE household_vendors ENABLE ROW LEVEL SECURITY;

CREATE POLICY household_items_user_policy ON household_items
    FOR ALL
    USING (auth.uid() = user_id)
    WITH CHECK (auth.uid() = user_id);

CREATE POLICY household_vendors_user_policy ON household_vendors
    FOR ALL
    USING (auth.uid() = user_id)
    WITH CHECK (auth.uid() = user_id);
```

RLS is enabled but the server uses the service role key, which bypasses these policies entirely. The policies exist as a safety net for direct database access via the Supabase dashboard or any anon-key client that might be added later. They also document intent: this data is strictly per-user.

`FOR ALL` covers SELECT, INSERT, UPDATE, and DELETE. The `USING` clause governs read visibility; `WITH CHECK` governs write permission. Both enforce the same condition: `auth.uid()` (the authenticated session's user) must equal the row's `user_id`.

### The `update_updated_at` trigger (lines 59–72)

```sql
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

DROP TRIGGER IF EXISTS update_household_items_updated_at ON household_items;
CREATE TRIGGER update_household_items_updated_at
    BEFORE UPDATE ON household_items
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

A `BEFORE UPDATE` trigger fires before the row is written to disk, so setting `NEW.updated_at` affects the row that actually gets stored. `RETURN NEW` is required — without it the update is cancelled. The `DROP TRIGGER IF EXISTS` guard makes the schema idempotent: re-running the SQL file (e.g., during a redeploy) does not error.

---

## The Five Tools

### Tool 1: `add_household_item` (handler: lines 217–242)

**Purpose:** Insert a single row into `household_items`.

**Required inputs:** `user_id`, `name`
**Optional inputs:** `category`, `location`, `details`, `notes`

```typescript
async function handleAddHouseholdItem(args: any): Promise<string> {
  const { user_id, name, category, location, details, notes } = args;

  const { data, error } = await supabase
    .from("household_items")
    .insert({
      user_id,
      name,
      category: category || null,
      location: location || null,
      details: details || {},
      notes: notes || null,
    })
    .select()
    .single();

  if (error) {
    throw new Error(`Failed to add household item: ${error.message}`);
  }

  return JSON.stringify({
    success: true,
    message: `Added household item: ${name}`,
    item: data,
  }, null, 2);
}
```

**Pattern notes:**

`.select().single()` after `.insert()` is the Supabase pattern for "insert and return the created row". Without `.select()`, the insert returns no data. Without `.single()`, it returns an array. The combination returns a single typed object that includes the server-generated `id`, `created_at`, and `updated_at` values — Claude receives these back in the response and can use the `id` in a follow-up `get_item_details` call.

Optional fields are normalized to `null` explicitly (`category || null`) rather than letting `undefined` pass through. This is defensive — the Supabase JS client's behavior with `undefined` values in insert payloads has changed across versions, and explicit `null` is unambiguous.

`details` defaults to `{}` rather than `null`. The schema column has `DEFAULT '{}'`, but explicitly providing `{}` in the insert makes the intent clear to any reader.

---

### Tool 2: `search_household_items` (handler: lines 244–277)

**Purpose:** Query `household_items` with up to three simultaneous filters.

**Required inputs:** `user_id`
**Optional inputs:** `query` (text search), `category` (exact-ish filter), `location` (exact-ish filter)

```typescript
async function handleSearchHouseholdItems(args: any): Promise<string> {
  const { user_id, query, category, location } = args;

  let queryBuilder = supabase
    .from("household_items")
    .select("*")
    .eq("user_id", user_id);

  if (category) {
    queryBuilder = queryBuilder.ilike("category", `%${category}%`);
  }

  if (location) {
    queryBuilder = queryBuilder.ilike("location", `%${location}%`);
  }

  if (query) {
    queryBuilder = queryBuilder.or(
      `name.ilike.%${query}%,category.ilike.%${query}%,location.ilike.%${query}%,notes.ilike.%${query}%`
    );
  }

  const { data, error } = await queryBuilder.order("created_at", { ascending: false });
  // ...
}
```

**Pattern notes — dynamic query building:**

The `queryBuilder` variable is reassigned each time a filter is conditionally appended. This is the standard Supabase JS pattern for optional filters. Each method call on the builder returns a new builder instance (fluent interface), so the chain is safe to extend conditionally.

The filter application order matters:

1. `.eq("user_id", user_id)` — always applied first. Every subsequent filter operates on the already-scoped result set.
2. `.ilike("category", ...)` — applied if `category` was provided. ILIKE is case-insensitive LIKE; the `%` wildcards allow substring matching, so "paint" matches "wall paint" and "Paint (exterior)".
3. `.ilike("location", ...)` — applied if `location` was provided, same pattern.
4. `.or(...)` — applied if a freetext `query` was provided.

**The `.or()` call** is the most nuanced part. The argument is a PostgREST filter string, not a JavaScript expression:

```
name.ilike.%${query}%,category.ilike.%${query}%,location.ilike.%${query}%,notes.ilike.%${query}%
```

This translates to:

```sql
WHERE (
  name ILIKE '%query%'
  OR category ILIKE '%query%'
  OR location ILIKE '%query%'
  OR notes ILIKE '%query%'
)
```

The outer `.eq("user_id")` and any `.ilike()` filters are ANDed with this OR block. So if a caller provides both `category: "paint"` and `query: "sherwin"`, the effective query is:

```sql
WHERE user_id = $1
  AND category ILIKE '%paint%'
  AND (
    name ILIKE '%sherwin%'
    OR category ILIKE '%sherwin%'
    OR location ILIKE '%sherwin%'
    OR notes ILIKE '%sherwin%'
  )
```

Results are returned newest-first via `.order("created_at", { ascending: false })`.

---

### Tool 3: `get_item_details` (handler: lines 279–301)

**Purpose:** Fetch a single `household_items` row by primary key.

**Required inputs:** `item_id`, `user_id`

```typescript
async function handleGetItemDetails(args: any): Promise<string> {
  const { item_id, user_id } = args;

  const { data, error } = await supabase
    .from("household_items")
    .select("*")
    .eq("id", item_id)
    .eq("user_id", user_id)
    .single();

  if (error) {
    throw new Error(`Failed to get item details: ${error.message}`);
  }

  if (!data) {
    throw new Error("Item not found or access denied");
  }
  // ...
}
```

**Pattern notes — dual `.eq()` for authorization:**

The query filters on both `id` and `user_id`. Filtering on `id` alone would be sufficient to find the row — UUIDs are globally unique. The second `.eq("user_id", user_id)` is an authorization check: if `item_id` belongs to a different user, `.single()` returns no rows and the explicit `if (!data)` guard throws "Item not found or access denied". This message is deliberately ambiguous — it does not tell the caller whether the item exists but belongs to someone else, or simply does not exist at all.

This dual-filter pattern is used instead of relying solely on RLS because the server uses the service role key (RLS bypass). The application layer replicates what RLS would do in a user-session context.

---

### Tool 4: `add_vendor` (handler: lines 303–331)

**Purpose:** Insert a row into `household_vendors`.

**Required inputs:** `user_id`, `name`
**Optional inputs:** `service_type`, `phone`, `email`, `website`, `notes`, `rating`, `last_used`

```typescript
async function handleAddVendor(args: any): Promise<string> {
  const { user_id, name, service_type, phone, email, website, notes, rating, last_used } = args;

  const { data, error } = await supabase
    .from("household_vendors")
    .insert({
      user_id,
      name,
      service_type: service_type || null,
      phone: phone || null,
      email: email || null,
      website: website || null,
      notes: notes || null,
      rating: rating || null,
      last_used: last_used || null,
    })
    .select()
    .single();

  if (error) {
    throw new Error(`Failed to add vendor: ${error.message}`);
  }
  // ...
}
```

**Pattern notes:**

`add_vendor` has 8 optional fields compared to `add_household_item`'s 4. The handler follows the same `field || null` normalization pattern for all of them. The `rating` field warrants attention: the schema enforces `CHECK (rating >= 1 AND rating <= 5)` at the database level, so if a caller passes `rating: 0` or `rating: 6`, the insert will fail with a constraint violation. The error propagates through the `throw new Error(...)` in the error check and is returned to Claude as `{ success: false, error: "..." }`.

`last_used` is a `DATE` column in Postgres, so the string passed by the caller must be parseable as a date. The tool description specifies `YYYY-MM-DD` format. Postgres will reject malformed dates with a type error.

---

### Tool 5: `list_vendors` (handler: lines 333–356)

**Purpose:** List all vendors for a user, optionally filtered by service type.

**Required inputs:** `user_id`
**Optional inputs:** `service_type`

```typescript
async function handleListVendors(args: any): Promise<string> {
  const { user_id, service_type } = args;

  let queryBuilder = supabase
    .from("household_vendors")
    .select("*")
    .eq("user_id", user_id);

  if (service_type) {
    queryBuilder = queryBuilder.ilike("service_type", `%${service_type}%`);
  }

  const { data, error } = await queryBuilder.order("name", { ascending: true });
  // ...
}
```

**Pattern notes:**

`list_vendors` orders by `name` ascending — alphabetical order is the natural way to scan a contact list. This contrasts with `search_household_items` which orders by `created_at` descending (newest first is more useful for a knowledge base that grows over time).

The `service_type` filter uses ILIKE, not exact match. A call with `service_type: "plumb"` returns both "plumber" and "master plumbing services". This is intentional — Claude often generates reasonable-but-not-exact category labels, and fuzzy matching makes the tool more forgiving of that.

---

## Server Setup and Startup

### Supabase client initialization (lines 30–39)

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

`autoRefreshToken: false` and `persistSession: false` are set because this is a server-side, non-interactive process. There is no user session to refresh or persist. These options prevent the Supabase client from attempting background token refresh operations that would fail in a stdio environment.

### Environment validation (lines 21–27)

```typescript
const SUPABASE_URL = process.env.SUPABASE_URL;
const SUPABASE_SERVICE_ROLE_KEY = process.env.SUPABASE_SERVICE_ROLE_KEY;

if (!SUPABASE_URL || !SUPABASE_SERVICE_ROLE_KEY) {
  console.error("Error: SUPABASE_URL and SUPABASE_SERVICE_ROLE_KEY must be set");
  process.exit(1);
}
```

Validation runs at module load time, before the MCP server is instantiated. This means a misconfigured process exits immediately with a clear error message rather than starting successfully and then failing on the first tool call. `console.error` writes to stderr, which does not interfere with the JSON-RPC stdio channel on stdout.

### MCP server registration (lines 358–401)

```typescript
const server = new Server(
  {
    name: "household-knowledge",
    version: "1.0.0",
  },
  {
    capabilities: {
      tools: {},
    },
  }
);

server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: TOOLS,
}));

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  try {
    switch (name) {
      case "add_household_item":
        return { content: [{ type: "text", text: await handleAddHouseholdItem(args) }] };
      // ... other cases
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

Two request handlers are registered:

- `ListToolsRequestSchema` — returns the static `TOOLS` array when the MCP client asks what tools are available (happens at startup).
- `CallToolRequestSchema` — dispatches incoming tool calls to handler functions via a switch statement.

All errors thrown by handlers are caught in the `CallToolRequestSchema` handler and returned as MCP error responses with `isError: true`. This prevents unhandled rejections from crashing the process and surfaces the error message to Claude in a readable form.

### Entry point (lines 404–413)

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

`StdioServerTransport` wires the MCP SDK to stdin/stdout. The startup confirmation message goes to stderr (not stdout) to keep the JSON-RPC channel clean. The top-level `.catch()` ensures the process exits with a non-zero code if `main()` rejects — important for Claude Desktop to detect a failed server launch.

---

## Key Patterns Introduced

This extension introduces four patterns that recur throughout the learning path.

### 1. JSONB for schema-less metadata

The `details` column on `household_items` accepts arbitrary key-value pairs without requiring schema changes. Claude can store `{"brand": "Sherwin Williams", "code": "SW 6204"}` for paint and `{"serial": "FD12345678", "warranty_expiry": "2026-06"}` for appliances in the same column of the same table. When reading the data back, Claude reconstructs the context from whatever keys are present.

The pattern is appropriate when: the set of attributes varies significantly across rows, the exact keys are not known at schema design time, and you do not need to filter or index on specific JSONB keys.

### 2. ILIKE for case-insensitive substring search

All text searches in this server use `ILIKE '%value%'` rather than `= 'value'`. This makes searches tolerant of capitalization differences and partial matches. The cost is that ILIKE cannot use a standard B-tree index — it requires a `pg_trgm` trigram index or a full-text search index for good performance at scale. For a single-user household knowledge base, table size will never be large enough for this to matter.

### 3. `.or()` for multi-column freetext search

The `search_household_items` tool demonstrates how to search across multiple columns in a single query using PostgREST's filter string syntax inside `.or()`. The pattern scales to any number of columns and can be extended to include JSONB text operators if needed.

### 4. `user_id` scoping in every query

Every query in this server includes `.eq("user_id", user_id)` as the first filter. The `user_id` is passed in as a tool argument by Claude, sourced from the user's Supabase auth context. This application-level scoping works in combination with the composite indexes (which lead with `user_id`) to keep queries efficient in a multi-user deployment.

---

## How It Fits in the Extension Learning Path

Extension 1 is fully standalone. It has no foreign keys to other extensions and no code dependencies outside the `@modelcontextprotocol/sdk` and `@supabase/supabase-js` packages.

**Relationship to Extension 2 (Maintenance Tracker):** The `household_vendors` table in this extension and whatever vendor storage Extension 2 uses are structurally related — both track service providers. However, there is no formal foreign key between them. If you want to cross-reference, you would join on `user_id` and `name` (or add a `vendor_id` column to Extension 2's maintenance records referencing `household_vendors.id`). That is not done in the current schema.

**What later extensions build on:** The patterns introduced here — JSONB metadata, ILIKE search, user-scoped queries, `.select().single()` after insert — appear in every subsequent extension. Reading this extension first makes the others easier to understand.

---

## Source Files

| File | Lines | Contents |
|------|-------|----------|
| `extensions/household-knowledge/index.ts` | 414 | Full server implementation |
| `extensions/household-knowledge/index.ts:11–18` | 8 | Imports: MCP SDK, Supabase client |
| `extensions/household-knowledge/index.ts:21–27` | 7 | Environment variable validation |
| `extensions/household-knowledge/index.ts:30–39` | 10 | Supabase client initialization |
| `extensions/household-knowledge/index.ts:42–66` | 25 | TypeScript interfaces: `HouseholdItem`, `HouseholdVendor` |
| `extensions/household-knowledge/index.ts:69–214` | 146 | Tool definitions array (5 tools with JSON Schema input schemas) |
| `extensions/household-knowledge/index.ts:217–242` | 26 | `handleAddHouseholdItem` |
| `extensions/household-knowledge/index.ts:244–277` | 34 | `handleSearchHouseholdItems` |
| `extensions/household-knowledge/index.ts:279–301` | 23 | `handleGetItemDetails` |
| `extensions/household-knowledge/index.ts:303–331` | 29 | `handleAddVendor` |
| `extensions/household-knowledge/index.ts:333–356` | 24 | `handleListVendors` |
| `extensions/household-knowledge/index.ts:358–401` | 44 | Server instantiation and request handler registration |
| `extensions/household-knowledge/index.ts:404–413` | 10 | `main()` entry point and top-level error handler |
| `extensions/household-knowledge/schema.sql` | 78 | Full schema |
| `extensions/household-knowledge/schema.sql:6–16` | 11 | `household_items` table definition |
| `extensions/household-knowledge/schema.sql:20–32` | 13 | `household_vendors` table definition |
| `extensions/household-knowledge/schema.sql:35–39` | 5 | Composite indexes |
| `extensions/household-knowledge/schema.sql:42–56` | 15 | RLS enable + policies |
| `extensions/household-knowledge/schema.sql:59–72` | 14 | `update_updated_at_column` function and trigger |
