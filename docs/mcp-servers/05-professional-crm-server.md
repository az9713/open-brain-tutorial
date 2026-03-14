# Extension 5: Professional CRM MCP Server — Deep Dive

This document explains every technical decision in the Professional CRM extension. It
assumes you have worked through Extensions 1 through 4 and are comfortable with
TypeScript, Supabase, MCP, and Row Level Security. Where Extension 5 introduces
patterns that have not appeared before — cross-extension integration, auto-updating
triggers, partial indexes, and the `ON DELETE SET NULL` foreign key behavior — those
patterns are explained from first principles.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Role in the Extension Architecture](#2-role-in-the-extension-architecture)
3. [Database Schema](#3-database-schema)
4. [The Auto-Update Trigger](#4-the-auto-update-trigger)
5. [The Seven Tools](#5-the-seven-tools)
6. [Key Patterns](#6-key-patterns)
7. [How This Server Fits with Other Extensions](#7-how-this-server-fits-with-other-extensions)
8. [Source File Reference](#8-source-file-reference)

---

## 1. Overview

The Professional CRM MCP server is a contact and relationship management system. It
tracks three categories of professional data:

- **Who** is in your network (`professional_contacts`)
- **Every touchpoint** with each contact (`contact_interactions`)
- **Deals and collaborations** you are pursuing (`opportunities`)

The server exposes seven MCP tools. Six operate entirely within the CRM's own tables.
The seventh — `link_thought_to_contact` — reaches across extension boundaries and reads
from the core Open Brain `thoughts` table. This is the first cross-extension integration
in the learning path: a single tool that joins data from two separate domains.

The server is 626 lines of TypeScript across a single file (`index.ts`). The schema is
132 lines of SQL (`schema.sql`). All three tables use Row Level Security.

---

## 2. Role in the Extension Architecture

Extension 5 is the second-to-last step in the six-extension learning path. It earns
the label "advanced" for two reasons.

**First**, it introduces three-table relational depth. Extensions 1 and 2 use a single
table. Extension 3 introduces a parent-child join (`family_members` → `activities`).
Extension 5 goes one level further: `professional_contacts` is the root, `contact_interactions`
is a child of contacts, and `opportunities` is a sibling of interactions that can also
exist without a contact (`contact_id` is nullable). The `get_contact_history` tool
must query all three tables in sequence to build a composite profile.

**Second**, it introduces the cross-extension bridge. Every extension before this one
operates on its own tables in isolation. Extension 5 is the first to deliberately query
a table from a different domain — the core `thoughts` table — inside a tool handler.
This is the architectural moment where the extensions begin to compound.

The position in the learning path:

```
Extension 1  Household Knowledge Base   single table, no RLS
Extension 2  Home Maintenance Tracker   single table, no RLS
Extension 3  Family Calendar            multi-table, no RLS
Extension 4  Meal Planning & Recipes    multi-table, RLS introduced
Extension 5  Professional CRM           multi-table, RLS, cross-extension integration
Extension 6  Job Hunt Pipeline          multi-table, RLS, cross-extension at scale
```

The data flow between Extension 5 and the rest of the system:

```
  Core Open Brain
  ┌─────────────────────┐
  │   thoughts table    │◄──────────────────────────────┐
  │   (pgvector)        │                               │
  └─────────────────────┘                               │
                                                        │ link_thought_to_contact
  Extension 5: Professional CRM                         │ reads thought.content
  ┌──────────────────────────────────────────────────┐  │
  │  professional_contacts                           │  │
  │    └── contact_interactions  (trigger updates    │──┘
  │    └── opportunities         last_contacted)     │
  └──────────────────────────────────────────────────┘
                    │
                    │ soft FK (contact UUID stored in
                    │ job_hunt.contacts.crm_contact_id)
                    ▼
  Extension 6: Job Hunt Pipeline
  ┌──────────────────────────────┐
  │  companies                   │
  │  job_postings                │
  │  applications                │
  │  interviews                  │
  │  contacts                    │
  └──────────────────────────────┘
```

---

## 3. Database Schema

The full schema lives in `schema.sql`. Three tables, four indexes (one partial), three
RLS policies, and two triggers.

### 3.1 `professional_contacts`

```sql
CREATE TABLE IF NOT EXISTS professional_contacts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
    name TEXT NOT NULL,
    company TEXT,
    title TEXT,
    email TEXT,
    phone TEXT,
    linkedin_url TEXT,
    how_we_met TEXT,
    tags TEXT[] DEFAULT '{}',
    notes TEXT,
    last_contacted TIMESTAMPTZ,
    follow_up_date DATE,
    created_at TIMESTAMPTZ DEFAULT now() NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT now() NOT NULL
);
```

Two columns here are not straightforward string fields.

`tags TEXT[]` is a PostgreSQL array of text values. It stores multiple categories in
a single column without a separate join table. The default `'{}'` is an empty array,
not null. Array columns support containment operators that flat text columns do not —
the `search_contacts` tool uses `.contains("tags", tags)` to find contacts that have
all of the requested tags, not just any of them. This is covered in
[Section 6.2](#62-array-containment-search).

`last_contacted TIMESTAMPTZ` starts null and is never updated by the application code.
The database updates it automatically via a trigger whenever a row is inserted into
`contact_interactions`. This design and its implications are covered in
[Section 4](#4-the-auto-update-trigger).

`follow_up_date DATE` stores the next scheduled follow-up as a plain date (no time
component). The `get_follow_ups_due` tool queries against this column and separates
results into overdue and upcoming buckets.

### 3.2 `contact_interactions`

```sql
CREATE TABLE IF NOT EXISTS contact_interactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contact_id UUID REFERENCES professional_contacts(id) ON DELETE CASCADE NOT NULL,
    user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
    interaction_type TEXT NOT NULL CHECK (interaction_type IN (
        'meeting', 'email', 'call', 'coffee', 'event', 'linkedin', 'other'
    )),
    occurred_at TIMESTAMPTZ DEFAULT now() NOT NULL,
    summary TEXT NOT NULL,
    follow_up_needed BOOLEAN DEFAULT false,
    follow_up_notes TEXT,
    created_at TIMESTAMPTZ DEFAULT now() NOT NULL
);
```

The `CHECK` constraint on `interaction_type` enforces a controlled vocabulary at the
database layer. The seven permitted values reflect the ways professionals actually
stay in touch. `other` exists as an escape hatch for situations the schema author did
not anticipate.

`contact_id` references `professional_contacts(id)` with `ON DELETE CASCADE`. If a
contact is deleted, all their interaction history is deleted too. This is the expected
behavior — interaction records without a parent contact are meaningless orphans.

Note that `contact_interactions` has its own `user_id` column in addition to the
`contact_id` foreign key. This is redundant from a relational standpoint — you could
derive `user_id` by joining to `professional_contacts`. The duplication exists for
RLS: the policy on `contact_interactions` uses `auth.uid() = user_id` directly, which
is simpler and faster than evaluating a join condition in a policy expression.

### 3.3 `opportunities`

```sql
CREATE TABLE IF NOT EXISTS opportunities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
    contact_id UUID REFERENCES professional_contacts(id) ON DELETE SET NULL,
    title TEXT NOT NULL,
    description TEXT,
    stage TEXT DEFAULT 'identified' CHECK (stage IN (
        'identified', 'in_conversation', 'proposal', 'negotiation', 'won', 'lost'
    )),
    value DECIMAL(12,2),
    expected_close_date DATE,
    notes TEXT,
    created_at TIMESTAMPTZ DEFAULT now() NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT now() NOT NULL
);
```

Two decisions here distinguish `opportunities` from the other tables.

**`contact_id` uses `ON DELETE SET NULL`, not `ON DELETE CASCADE`.**

Compare the three foreign key relationships in this schema:

| Table | References | On Delete |
|---|---|---|
| `professional_contacts` | `auth.users` | CASCADE |
| `contact_interactions` | `professional_contacts` | CASCADE |
| `opportunities` | `professional_contacts` | SET NULL |

Interactions are meaningless without their parent contact — if you delete a contact,
you delete their history. But an opportunity can outlive a contact. A consulting deal
is still in progress even if the contact who introduced it has been removed from your
CRM. `ON DELETE SET NULL` preserves the opportunity row and sets `contact_id` to null
when the referenced contact is deleted.

**`value DECIMAL(12,2)` is nullable.**

Not every opportunity has a known monetary value. A speaking invitation, a potential
co-author relationship, or an advisory role may have value that is not yet quantified.
The column is optional and the type is precise: `DECIMAL(12,2)` stores up to 10 digits
before the decimal point and exactly 2 after, which is correct for currency values.

**The six-stage pipeline:**

```
identified → in_conversation → proposal → negotiation → won
                                                       → lost
```

`identified` is the default stage. The pipeline is a linear progression except for the
two terminal states `won` and `lost`, which can be reached from any earlier stage. The
`CHECK` constraint ensures that an invalid stage value is rejected at the database
level; the tool's `inputSchema` enum communicates the valid values to the AI agent.

### 3.4 Indexes

```sql
-- Composite index for follow-up queries
CREATE INDEX IF NOT EXISTS idx_professional_contacts_user_last_contacted
    ON professional_contacts(user_id, last_contacted);

-- Partial index: only indexes rows where follow_up_date is set
CREATE INDEX IF NOT EXISTS idx_professional_contacts_follow_up
    ON professional_contacts(user_id, follow_up_date)
    WHERE follow_up_date IS NOT NULL;

-- Composite index for interaction history retrieval
CREATE INDEX IF NOT EXISTS idx_contact_interactions_contact_occurred
    ON contact_interactions(contact_id, occurred_at DESC);

-- Composite index for pipeline queries
CREATE INDEX IF NOT EXISTS idx_opportunities_user_stage
    ON opportunities(user_id, stage);
```

The second index is a **partial index** — note the `WHERE follow_up_date IS NOT NULL`
clause. A partial index only indexes rows that satisfy the condition, producing a smaller
and faster index than a full table index would. The `get_follow_ups_due` tool queries
only contacts with a non-null `follow_up_date`, so the index only needs to cover those
rows. Contacts without a follow-up date are never queried by that tool and do not need
to appear in the index at all.

### 3.5 Row Level Security

All three tables have RLS enabled with a single policy each:

```sql
ALTER TABLE professional_contacts ENABLE ROW LEVEL SECURITY;
ALTER TABLE contact_interactions ENABLE ROW LEVEL SECURITY;
ALTER TABLE opportunities ENABLE ROW LEVEL SECURITY;

CREATE POLICY professional_contacts_user_policy ON professional_contacts
    FOR ALL
    USING (auth.uid() = user_id)
    WITH CHECK (auth.uid() = user_id);
```

The same `FOR ALL` policy pattern applies to all three tables. The service role key
used by the MCP server bypasses RLS entirely, so these policies protect the tables from
any client that does not use the service role — for example, a frontend dashboard that
uses an anon key, or a user who queries the database directly via the Supabase client
libraries with their own session token.

---

## 4. The Auto-Update Trigger

The most important behavioral contract in this schema is the relationship between
`contact_interactions` and `professional_contacts.last_contacted`. The application code
does not update `last_contacted`. The database does it automatically.

### 4.1 The Function

```sql
-- schema.sql, lines 112-120
CREATE OR REPLACE FUNCTION update_last_contacted()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE professional_contacts
    SET last_contacted = NEW.occurred_at
    WHERE id = NEW.contact_id;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

`RETURNS TRIGGER` is the PostgreSQL return type for a trigger function. `NEW` is a
special variable available inside trigger functions that refers to the new row being
inserted or updated. Here, `NEW.occurred_at` is the timestamp of the interaction being
logged, and `NEW.contact_id` is the foreign key pointing to the parent contact.

The function executes an `UPDATE` on `professional_contacts`, setting `last_contacted`
to the interaction's `occurred_at` timestamp for the matching contact row.

### 4.2 The Trigger

```sql
-- schema.sql, lines 123-127
CREATE TRIGGER update_contact_last_contacted
    AFTER INSERT ON contact_interactions
    FOR EACH ROW
    EXECUTE FUNCTION update_last_contacted();
```

`AFTER INSERT` means the trigger fires after the interaction row has been successfully
written. `FOR EACH ROW` means the function runs once per inserted row (as opposed to
once per statement, which would only run once even if a bulk insert affected many rows).

### 4.3 What This Means for the Application Layer

The `log_interaction` handler (index.ts, line 370) inserts the interaction row and
returns immediately. It does not make a second Supabase call to update the contact.
The comment in the code is explicit about this:

```typescript
// Note: last_contacted is automatically updated by database trigger
```

This is an intentional design choice. Centralizing the update logic in the database
means it is impossible to log an interaction without updating `last_contacted`. If the
update were in the application layer, a bug or a direct database insert that bypassed
the MCP server would leave `last_contacted` stale. The trigger makes the invariant
enforced at the storage layer.

The schema also defines a second trigger function (`update_updated_at_column`) that
keeps the `updated_at` column current on `professional_contacts` and `opportunities`
whenever those rows are updated. This is a common pattern across all extensions.

---

## 5. The Seven Tools

### 5.1 `add_professional_contact`

**Purpose:** Add a person to your professional network.

**Required parameters:** `user_id`, `name`
**Optional parameters:** `company`, `title`, `email`, `phone`, `linkedin_url`,
`how_we_met`, `tags`, `notes`

**Implementation (index.ts, lines 308-337):**

```typescript
const { data, error } = await supabase
  .from("professional_contacts")
  .insert({
    user_id,
    name,
    company: company || null,
    title: title || null,
    email: email || null,
    phone: phone || null,
    linkedin_url: linkedin_url || null,
    how_we_met: how_we_met || null,
    tags: tags || [],
    notes: notes || null,
  })
  .select()
  .single();
```

Every optional field uses `|| null` coercion except `tags`, which uses `|| []`. The
difference is intentional: the column type is `TEXT[]`, and the database default is
an empty array. If the agent omits tags, we want an empty array, not a null value.
Using `|| null` on a `TEXT[]` column would store null in a column designed to hold
an array, which would break the `.contains("tags", tags)` filter in `search_contacts`.

`.select().single()` returns the newly created row including its generated UUID, which
the agent needs to reference when logging interactions or creating opportunities for
this contact.

---

### 5.2 `search_contacts`

**Purpose:** Find contacts by name, company, title, notes content, or tags.

**Required parameters:** `user_id`
**Optional parameters:** `query` (text search), `tags` (array filter)

**Implementation (index.ts, lines 339-368):**

```typescript
let queryBuilder = supabase
  .from("professional_contacts")
  .select("*")
  .eq("user_id", user_id);

if (query) {
  queryBuilder = queryBuilder.or(
    `name.ilike.%${query}%,company.ilike.%${query}%,title.ilike.%${query}%,notes.ilike.%${query}%`
  );
}

if (tags && tags.length > 0) {
  queryBuilder = queryBuilder.contains("tags", tags);
}

const { data, error } = await queryBuilder.order("name", { ascending: true });
```

Two filter types can be applied independently or together.

The text search uses a PostgREST `.or()` expression with `ilike` operators across four
columns. This is a multi-column ILIKE — if the query term appears anywhere in the
name, company, title, or notes field, the contact is returned.

The tag filter uses `.contains("tags", tags)`, which maps to the PostgreSQL `@>`
(array containment) operator. This is covered in detail in
[Section 6.2](#62-array-containment-search).

Both filters are additive: if both `query` and `tags` are provided, the agent gets
contacts that match the text AND have all the specified tags.

---

### 5.3 `log_interaction`

**Purpose:** Record a touchpoint with a contact. Automatically updates the contact's
`last_contacted` timestamp via the database trigger.

**Required parameters:** `user_id`, `contact_id`, `interaction_type`, `summary`
**Optional parameters:** `occurred_at`, `follow_up_needed`, `follow_up_notes`

**Implementation (index.ts, lines 370-398):**

```typescript
const { data, error } = await supabase
  .from("contact_interactions")
  .insert({
    user_id,
    contact_id,
    interaction_type,
    occurred_at: occurred_at || new Date().toISOString(),
    summary,
    follow_up_needed: follow_up_needed || false,
    follow_up_notes: follow_up_notes || null,
  })
  .select()
  .single();
```

`occurred_at` defaults to the current timestamp when not provided. This covers the
common case of logging an interaction that just happened. To backfill a past interaction
(a meeting from last week), the agent can supply an explicit ISO 8601 timestamp.

The interaction types accepted by the tool's `inputSchema` enum exactly match the
`CHECK` constraint values in the schema: `meeting`, `email`, `call`, `coffee`, `event`,
`linkedin`, `other`. This alignment means any value the agent is permitted to send
will be accepted by the database.

After this insert, the database trigger fires and updates `professional_contacts.last_contacted`
to the value of `occurred_at`. The application code does not participate in that update.

---

### 5.4 `get_contact_history`

**Purpose:** Return a contact's full profile, complete interaction timeline, and all
related opportunities in a single call.

**Required parameters:** `user_id`, `contact_id`

**Implementation (index.ts, lines 400-446):** Three sequential queries.

```typescript
// Query 1: contact details
const { data: contact, error: contactError } = await supabase
  .from("professional_contacts")
  .select("*")
  .eq("id", contact_id)
  .eq("user_id", user_id)
  .single();

// Query 2: all interactions, newest first
const { data: interactions, error: interactionsError } = await supabase
  .from("contact_interactions")
  .select("*")
  .eq("contact_id", contact_id)
  .eq("user_id", user_id)
  .order("occurred_at", { ascending: false });

// Query 3: related opportunities, newest first
const { data: opportunities, error: opportunitiesError } = await supabase
  .from("opportunities")
  .select("*")
  .eq("contact_id", contact_id)
  .eq("user_id", user_id)
  .order("created_at", { ascending: false });
```

Each query is independent and runs in sequence. The results are assembled into a single
composite response object:

```typescript
return JSON.stringify({
  success: true,
  contact,
  interactions,
  opportunities,
  interaction_count: interactions.length,
}, null, 2);
```

This is three round-trips to Supabase, not one. A single SQL query with two JOINs
could retrieve all this data simultaneously. The three-query approach is a readability
tradeoff: each query is straightforward to read and debug, the error handling is
granular (you know which of the three queries failed), and the result shapes are clean
arrays rather than a denormalized join result that would need to be restructured.

For a personal CRM with hundreds of contacts, three queries is negligible. The pattern
is worth understanding because it is the simplest approach to multi-table retrieval in
the Supabase JavaScript client, and it is the pattern you will see repeated in
Extension 6 when the job hunt pipeline needs to assemble similarly composite profiles.

---

### 5.5 `create_opportunity`

**Purpose:** Create a new deal, project, or potential collaboration, optionally linked
to a contact.

**Required parameters:** `user_id`, `title`
**Optional parameters:** `contact_id`, `description`, `stage`, `value`,
`expected_close_date`, `notes`

**Implementation (index.ts, lines 448-475):**

```typescript
const { data, error } = await supabase
  .from("opportunities")
  .insert({
    user_id,
    contact_id: contact_id || null,
    title,
    description: description || null,
    stage: stage || "identified",
    value: value || null,
    expected_close_date: expected_close_date || null,
    notes: notes || null,
  })
  .select()
  .single();
```

`contact_id` is explicitly nullable. An opportunity can exist without a linked contact
— for example, a tender you found posted publicly before identifying a person to engage
with. The `|| null` coercion ensures an empty string from the agent does not trigger a
foreign key violation.

`stage` defaults to `"identified"` in the application layer. The database schema also
specifies `DEFAULT 'identified'`, so either layer would be sufficient; the double
default is belt-and-suspenders.

The six pipeline stages represent a complete lifecycle:

| Stage | Meaning |
|---|---|
| `identified` | You know this opportunity exists |
| `in_conversation` | Active dialogue has started |
| `proposal` | A formal proposal has been made |
| `negotiation` | Terms are being negotiated |
| `won` | The deal closed successfully |
| `lost` | The opportunity did not materialize |

`won` and `lost` are terminal states. The schema does not enforce this — a `won`
opportunity can be moved back to `negotiation` via a direct database update. The
pipeline is a convention, not a state machine.

---

### 5.6 `get_follow_ups_due`

**Purpose:** Return contacts with a `follow_up_date` in the past or within the next
N days, separated into overdue and upcoming buckets.

**Required parameters:** `user_id`
**Optional parameters:** `days_ahead` (defaults to 7)

**Implementation (index.ts, lines 477-509):**

```typescript
const daysToCheck = days_ahead || 7;
const today = new Date().toISOString().split('T')[0];
const futureDate = new Date();
futureDate.setDate(futureDate.getDate() + daysToCheck);
const futureDateStr = futureDate.toISOString().split('T')[0];

const { data, error } = await supabase
  .from("professional_contacts")
  .select("*")
  .eq("user_id", user_id)
  .not("follow_up_date", "is", null)
  .lte("follow_up_date", futureDateStr)
  .order("follow_up_date", { ascending: true });

// In-application split
const overdue = data.filter(c => c.follow_up_date! < today);
const upcoming = data.filter(c => c.follow_up_date! >= today);
```

The query uses the partial index `idx_professional_contacts_follow_up` because it
filters on `follow_up_date` and the index covers only non-null rows. The `.not("follow_up_date", "is", null)` filter in the query aligns exactly with the index's
`WHERE follow_up_date IS NOT NULL` condition.

No lower bound is applied to the query — contacts with `follow_up_date` arbitrarily
far in the past are included. The upper bound is `today + days_ahead`. The intent is
that anything overdue should always appear, regardless of how overdue it is.

The split into `overdue` and `upcoming` arrays happens in JavaScript after the query
returns, not in SQL. This keeps the query simple and lets the tool return structured
output that the agent can present without further processing.

---

### 5.7 `link_thought_to_contact`

**Purpose:** Retrieve a thought from the core Open Brain `thoughts` table and append
its content to a professional contact's notes field.

**Required parameters:** `user_id`, `thought_id`, `contact_id`

This is the cross-extension tool. It is the first tool in the learning path that
queries a table outside its own extension's schema.

**Implementation (index.ts, lines 511-564):** Three sequential operations.

```typescript
// Step 1: Retrieve the thought from the core Open Brain
const { data: thought, error: thoughtError } = await supabase
  .from("thoughts")
  .select("*")
  .eq("id", thought_id)
  .eq("user_id", user_id)
  .single();

if (!thought) {
  throw new Error("Thought not found or access denied");
}

// Step 2: Retrieve the contact
const { data: contact, error: contactError } = await supabase
  .from("professional_contacts")
  .select("*")
  .eq("id", contact_id)
  .eq("user_id", user_id)
  .single();

// Step 3: Append the thought content to the contact's notes
const linkNote = `\n\n[Linked Thought ${new Date().toISOString().split('T')[0]}]: ${thought.content}`;
const updatedNotes = (contact.notes || "") + linkNote;

const { data: updatedContact, error: updateError } = await supabase
  .from("professional_contacts")
  .update({ notes: updatedNotes })
  .eq("id", contact_id)
  .eq("user_id", user_id)
  .select()
  .single();
```

Several design decisions are worth examining.

**The integration is read-only toward `thoughts`.** The tool reads from `thoughts` but
never modifies it. The core Open Brain table is treated as an authoritative source of
record that extension tools consume but do not alter.

**The link is stored as formatted text, not a foreign key.** There is no `thought_id`
column on `professional_contacts` and no junction table connecting the two. The
thought's content is appended to the `notes` text field with a datestamp prefix:

```
[Linked Thought 2026-03-13]: Ran into Sarah Chen at the AI meetup — she's
looking for someone to help with their RAG pipeline.
```

This approach has tradeoffs. In favor: no schema migration is required, the link is
human-readable in the notes field, and the contact's profile remains a coherent text
narrative. Against: if the thought is later edited in the core Open Brain, the contact's
notes will not reflect the update. The link is a snapshot, not a live reference.

**User scope is enforced on both tables.** The thought query includes `.eq("user_id", user_id)`. If the `thought_id` belongs to a different user, the query returns nothing and
the tool throws `"Thought not found or access denied"`. This prevents a user from linking
another user's thoughts to their contacts, even though the service role key technically
has access to all rows in the `thoughts` table.

---

## 6. Key Patterns

### 6.1 Cross-Extension Queries

`link_thought_to_contact` demonstrates the pattern for cross-extension integration in
the Open Brain framework. The key properties of this pattern are:

- **Single Supabase project.** All extensions share one database. Any table is
  accessible to any MCP server that has the service role key. There is no API boundary
  between extensions — they are all tables in the same PostgreSQL instance.
- **User scope is the access control boundary.** Because the service role bypasses RLS,
  the application code is responsible for scoping queries with `.eq("user_id", user_id)`.
  Cross-extension tools must apply this filter on every table they touch.
- **Cross-extension reads are safe; writes require care.** Reading from `thoughts`
  carries no risk of corrupting core data. Writing to `thoughts` from an extension tool
  would need careful consideration of how the core Open Brain server expects that table
  to behave (embeddings, metadata, etc.).

### 6.2 Array Containment Search

The `tags TEXT[]` column and the `.contains("tags", tags)` query in `search_contacts`
use PostgreSQL's array containment operator `@>`.

```sql
-- What Supabase generates from .contains("tags", ["ai", "consulting"])
SELECT * FROM professional_contacts
WHERE tags @> ARRAY['ai', 'consulting']
AND user_id = $1;
```

`@>` means "contains all of". A contact with tags `['ai', 'consulting', 'conference']`
matches the query for `['ai', 'consulting']`. A contact with only `['ai']` does not.

Compare this to a text column approach, where you would need to store tags as a
comma-separated string (`'ai,consulting,conference'`) and use `ILIKE '%ai%'`. That
approach cannot distinguish between `'ai'` and `'ai-tools'` and cannot enforce
"contains all of" semantics cleanly. A native array column with the `@>` operator is
the right tool for multi-valued categorical data.

### 6.3 Auto-Update Triggers vs. Application-Layer Updates

The `last_contacted` field is updated by the `update_last_contacted` trigger rather
than by the application. This architectural choice appears three times in this schema:
`last_contacted` (behavioral update), `updated_at` on `professional_contacts`
(housekeeping update), and `updated_at` on `opportunities` (housekeeping update).

The general principle: use triggers when an update must be guaranteed to happen as a
consequence of another write, regardless of which code path causes that write. Use
application-layer updates when the update logic needs context that the database does
not have (e.g., calling an external API, computing a value from application state).

`last_contacted` is a pure database concern: it is derived entirely from data already
in the database (the `occurred_at` of the new interaction). It belongs in the trigger.

### 6.4 Progressive Query Building

`search_contacts` builds its query in stages, adding filter clauses conditionally:

```typescript
let queryBuilder = supabase
  .from("professional_contacts")
  .select("*")
  .eq("user_id", user_id);

if (query) {
  queryBuilder = queryBuilder.or(`name.ilike.%${query}%,...`);
}

if (tags && tags.length > 0) {
  queryBuilder = queryBuilder.contains("tags", tags);
}

const { data, error } = await queryBuilder.order("name", { ascending: true });
```

The Supabase JavaScript client returns a query builder object from each chained call.
That object is lazy — it does not execute the query until `.then()` is called implicitly
by `await`. This means a partial query can be stored in a variable, extended with
additional filters, and then executed once at the end. The query is only sent to the
database once, regardless of how many filter clauses are attached.

This pattern avoids the alternative of constructing raw SQL strings with manual string
concatenation, which is fragile and vulnerable to injection. The query builder handles
parameter escaping internally.

---

## 7. How This Server Fits with Other Extensions

**Extension 3 (Family Calendar)** introduced the parent-child relational pattern
(`family_members` → `activities`). Extension 5 uses the same pattern at greater depth:
`professional_contacts` → `contact_interactions`, with `opportunities` as a parallel
child that uses a different deletion behavior. If the join and foreign key patterns in
this extension are confusing, re-reading Section 3 of the Family Calendar deep-dive
will help.

**Extension 4 (Meal Planning)** introduced Row Level Security. Extension 5 applies the
same `FOR ALL` RLS pattern to all three of its tables. There are no new RLS concepts
here; the policy structure is identical to what Extension 4 introduced.

**Extension 6 (Job Hunt Pipeline)** references Extension 5 via a soft foreign key. The
job hunt extension stores professional contacts it creates in the `professional_contacts`
table (or stores a reference UUID) so that recruiter and hiring manager relationships
persist beyond the job search. This is the pattern described in the Extension 5 README:

> If you build Extension 6, the recruiters and hiring managers from your job search
> automatically become professional contacts. Those relationships persist in your CRM,
> ready for the long-term connection.

The integration is "soft" because there is no database-level foreign key between the
job hunt tables and `professional_contacts`. The job hunt extension stores a UUID that
it treats as a reference. If the contact is deleted from the CRM, the job hunt table
retains a stale UUID. Extension 6 handles this gracefully by treating the reference
as optional.

---

## 8. Source File Reference

| File | Lines | Purpose |
|---|---|---|
| `extensions/professional-crm/index.ts` | 626 | MCP server — all tool handlers and server wiring |
| `extensions/professional-crm/schema.sql` | 132 | Three tables, four indexes, RLS policies, two triggers |
| `extensions/professional-crm/package.json` | — | Node.js dependencies |
| `extensions/professional-crm/tsconfig.json` | — | TypeScript compiler configuration |
| `extensions/professional-crm/metadata.json` | — | Extension metadata for the repo registry |
| `extensions/professional-crm/README.md` | — | Setup guide and usage instructions |

### Key line references in `index.ts`

| Lines | What happens |
|---|---|
| 24–42 | Environment variable validation and Supabase client initialization |
| 44–87 | TypeScript interface definitions: `ProfessionalContact`, `ContactInteraction`, `Opportunity` |
| 90–305 | `TOOLS` array — all seven tool definitions with input schemas |
| 308–337 | `handleAddProfessionalContact` — insert with array default handling |
| 339–368 | `handleSearchContacts` — progressive query building, array containment |
| 370–398 | `handleLogInteraction` — insert that triggers `last_contacted` update |
| 400–446 | `handleGetContactHistory` — three sequential queries, composite response |
| 448–475 | `handleCreateOpportunity` — nullable contact FK, stage defaulting |
| 477–509 | `handleGetFollowUpsDue` — partial index query, in-app overdue/upcoming split |
| 511–564 | `handleLinkThoughtToContact` — cross-extension read from `thoughts`, note append |
| 566–613 | Server setup, `ListToolsRequestSchema` and `CallToolRequestSchema` handlers |
| 615–625 | `main()` function and process error handler |

### Key line references in `schema.sql`

| Lines | What happens |
|---|---|
| 6–22 | `professional_contacts` table definition |
| 26–36 | `contact_interactions` table with `CHECK` constraint on `interaction_type` |
| 40–52 | `opportunities` table with `ON DELETE SET NULL` and `DECIMAL(12,2)` value |
| 55–66 | Four indexes including the partial index on `follow_up_date` |
| 69–87 | RLS enabled and policies created on all three tables |
| 90–109 | `update_updated_at_column` function and triggers for `professional_contacts` and `opportunities` |
| 112–127 | `update_last_contacted` function and `update_contact_last_contacted` trigger |
