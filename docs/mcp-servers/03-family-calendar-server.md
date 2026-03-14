# Extension 3: Family Calendar MCP Server — Deep Dive

This document explains every technical decision in the Family Calendar extension. It
assumes you have read the Open Brain architecture overview and are comfortable with the
basics of TypeScript, Supabase, and MCP after working through Extensions 1 and 2. Where
Extension 3 introduces new patterns, those patterns are explained from first principles.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Role in the Extension Architecture](#2-role-in-the-extension-architecture)
3. [Database Schema](#3-database-schema)
4. [The Recurring Event Model](#4-the-recurring-event-model)
5. [Server Initialization — Simpler Than You Expect](#5-server-initialization--simpler-than-you-expect)
6. [Tool Definition Style — Inline vs. Separate Array](#6-tool-definition-style--inline-vs-separate-array)
7. [The Six Tools](#7-the-six-tools)
8. [The Week Schedule Query — How Conflict Detection Works](#8-the-week-schedule-query--how-conflict-detection-works)
9. [Relational Selects](#9-relational-selects)
10. [No RLS — Why, and What Changes in Extension 4](#10-no-rls--why-and-what-changes-in-extension-4)
11. [Indexes and Why They Matter Here](#11-indexes-and-why-they-matter-here)
12. [How This Server Fits with Other Extensions](#12-how-this-server-fits-with-other-extensions)
13. [Source File Reference](#13-source-file-reference)

---

## 1. Overview

The Family Calendar MCP server is a multi-person scheduling system. It tracks three
categories of time-related data:

- **Who** is in the household (`family_members`)
- **What recurring or one-time events** each person has (`activities`)
- **Dates that must not be forgotten** — birthdays, anniversaries, deadlines
  (`important_dates`)

The server exposes six MCP tools that an AI agent uses to populate and query these tables.
The central capability — and the most technically interesting piece — is `get_week_schedule`,
which runs a single query that retrieves both one-time events and recurring weekly activities
simultaneously, without requiring duplicate rows or a separate recurrence expansion engine.

The server is 403 lines of TypeScript across a single file (`index.ts`). The schema is 51
lines of SQL (`schema.sql`). There are no external services beyond Supabase.

---

## 2. Role in the Extension Architecture

Extension 3 sits at the midpoint of the six-extension learning path and earns the label
"intermediate difficulty" for two reasons.

**First**, it introduces multi-entity data modeling. Extensions 1 and 2 work with a single
primary table per domain (items, tasks). Extension 3 introduces a parent-child relationship:
`family_members` is the parent, and `activities` references it. A single MCP query must
join across both tables to return useful results.

**Second**, it introduces time-aware querying. Date ranges, times of day, day-of-week
strings, and the difference between a one-time event and a recurring event all live in the
same table and must be handled by a single query. This is harder to get right than a simple
equality filter.

Extension 3 also provides the `family_members` table that Extension 4 (Meal Planning)
depends on. Meal planning needs to know who is home this week to calculate how many meals
to prepare. That cross-extension dependency is intentional: the calendar is the source of
truth for household occupancy.

The extension does not use Row Level Security. RLS is introduced in Extension 4, where a
shared household requires scoped access. Extensions 1 through 3 are single-user systems
where the `user_id` column provides logical isolation without a formal security policy.
This is covered in detail in [Section 10](#10-no-rls--why-and-what-changes-in-extension-4).

---

## 3. Database Schema

The full schema lives in `schema.sql`. All three tables are created with plain `CREATE
TABLE` statements — no stored procedures, no triggers, no RLS policies.

### 3.1 `family_members`

```sql
CREATE TABLE family_members (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
    name TEXT NOT NULL,
    relationship TEXT, -- e.g. 'self', 'spouse', 'child', 'parent'
    birth_date DATE,
    notes TEXT,
    created_at TIMESTAMPTZ DEFAULT now()
);
```

`family_members` is a roster. Every person in the household gets a row. The `relationship`
column is free-text — the schema does not enforce a controlled vocabulary. The
`birth_date` column is optional. If supplied, it can feed `important_dates` logic, but the
schema does not enforce that connection.

`user_id` references `auth.users(id)` with `ON DELETE CASCADE`. If the user's Supabase
account is deleted, all their household members are deleted too. This is the standard
cascade pattern used across all Open Brain extensions.

### 3.2 `activities`

```sql
CREATE TABLE activities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
    family_member_id UUID REFERENCES family_members, -- null means whole family
    title TEXT NOT NULL,
    activity_type TEXT, -- e.g. 'sports', 'medical', 'school', 'social'
    day_of_week TEXT, -- for recurring: 'monday', 'tuesday', etc. null for one-time
    start_time TIME,
    end_time TIME,
    start_date DATE,
    end_date DATE, -- null for one-time or ongoing recurring
    location TEXT,
    notes TEXT,
    created_at TIMESTAMPTZ DEFAULT now()
);
```

This is the most important table in the extension. Notice that `family_member_id`
references `family_members` without `ON DELETE CASCADE` and without `NOT NULL`. Both
omissions are deliberate:

- **Nullable**: an activity that belongs to the whole family (a vacation, a household
  dinner reservation) has no single family member to attach to. `NULL` here means
  "applies to everyone."
- **No cascade specified**: if a family member row is deleted, their activities are not
  automatically deleted. This is a tradeoff — the schema author chose to leave orphaned
  activity rows rather than silently delete history.

The column layout models two kinds of events with one table structure:

| Column | One-time event | Recurring event |
|---|---|---|
| `day_of_week` | NULL | `'monday'`, `'tuesday'`, etc. |
| `start_date` | The event date | When the recurrence begins |
| `end_date` | NULL (or same as start) | When the recurrence ends (NULL = ongoing) |
| `start_time` | Time of the event | Time of each weekly occurrence |
| `end_time` | End time of the event | End time of each weekly occurrence |

This design and how it is queried is explained fully in [Section 4](#4-the-recurring-event-model).

### 3.3 `important_dates`

```sql
CREATE TABLE important_dates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
    family_member_id UUID REFERENCES family_members, -- null for family-wide dates
    title TEXT NOT NULL,
    date_value DATE NOT NULL,
    recurring_yearly BOOLEAN DEFAULT false,
    reminder_days_before INTEGER DEFAULT 7,
    notes TEXT,
    created_at TIMESTAMPTZ DEFAULT now()
);
```

`important_dates` handles a different class of time data: not scheduled blocks of time
with a start and end, but specific calendar dates that carry significance. Birthdays,
anniversaries, tax deadlines, passport renewal dates.

Two columns deserve attention:

- `recurring_yearly BOOLEAN DEFAULT false` — when `true`, this date repeats every year on
  the same month and day. The schema stores this intent as metadata; the application layer
  (or the AI agent) is responsible for acting on it. The database does not automatically
  generate future-year occurrences.
- `reminder_days_before INTEGER DEFAULT 7` — how many days in advance to surface this
  date. The `get_upcoming_dates` tool uses a date-range query; the agent can interpret
  this field to decide whether to proactively mention an upcoming date.

---

## 4. The Recurring Event Model

Calendar systems face a fundamental design choice: how do you represent an event that
happens every Tuesday?

**Option A — Expand on write.** When you create "Soccer every Tuesday for 6 months," the
system generates one row per occurrence (24 rows). Queries are simple: just filter by
date. But storage grows, and changing the time means updating 24 rows.

**Option B — Single row with recurrence metadata.** One row stores the pattern
(`day_of_week = 'tuesday'`, `start_date`, `end_date`). Queries must expand the pattern
at read time. Storage is minimal. Changes are a single-row update. But the query logic
is more complex.

Extension 3 uses Option B. The `activities` table stores one row per logical event
pattern, not one row per occurrence. The `get_week_schedule` query then has to figure out
which patterns are active during a given week — which is the work done by the `.or()`
filter described in [Section 8](#8-the-week-schedule-query--how-conflict-detection-works).

This approach does not do full iCalendar-style recurrence expansion (no RRULE, no
exception dates). It handles the common household case: something that happens on a
specific weekday between a start date and an optional end date. That covers most real
scheduling needs — school pickups, sports practices, weekly lessons — without the
complexity of a full calendaring engine.

---

## 5. Server Initialization — Simpler Than You Expect

```typescript
// index.ts, lines 11-18
const SUPABASE_URL = process.env.SUPABASE_URL;
const SUPABASE_SERVICE_ROLE_KEY = process.env.SUPABASE_SERVICE_ROLE_KEY;

if (!SUPABASE_URL || !SUPABASE_SERVICE_ROLE_KEY) {
  throw new Error("Missing required environment variables");
}

const supabase = createClient(SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY);
```

Compare this to Extensions 1 and 2. Those extensions pass additional options to
`createClient` — typically auth configuration to work with Row Level Security. Extension 3
calls `createClient` with just the URL and key, nothing else. This is the minimal
initialization pattern.

The reason is the absence of RLS. When an extension uses RLS, the Supabase client must
operate in a way that attaches a user context to each request so the database's security
policies can evaluate it. Without RLS, the service role key provides full access to all
rows in all tables, and no additional auth configuration is needed.

The `if (!SUPABASE_URL || !SUPABASE_SERVICE_ROLE_KEY)` guard at startup is a hard fail.
The process throws before the MCP server is registered. This is intentional: a server that
starts but cannot connect to the database is more dangerous than one that refuses to start,
because it will silently return errors for every tool call and give the appearance of being
functional.

---

## 6. Tool Definition Style — Inline vs. Separate Array

Extensions vary in how they structure their tool definitions. Understanding the difference
helps you read unfamiliar extension code quickly.

**Separate TOOLS array (Extensions 1 and 2 style):**
```typescript
// Defined at module scope, outside any handler
const TOOLS = [
  { name: "tool_one", description: "...", inputSchema: { ... } },
  { name: "tool_two", description: "...", inputSchema: { ... } },
];

server.setRequestHandler(ListToolsRequestSchema, async () => {
  return { tools: TOOLS };
});
```

**Inline definition (Extension 3 style):**
```typescript
// Defined directly inside the handler, returned immediately
server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      { name: "add_family_member", description: "...", inputSchema: { ... } },
      { name: "add_activity", description: "...", inputSchema: { ... } },
      // ...
    ],
  };
});
```

Both are valid. The difference is organizational, not functional. The MCP protocol does not
care where the tool objects are constructed — it only cares that the `ListToolsRequestSchema`
handler returns a properly shaped response.

The inline style is slightly more compact for smaller extensions. The separate array style
scales better when tools are reused elsewhere (for example, in validation logic or
documentation generation). In Extension 3, the inline style is fine because the tool
definitions are only ever used in the `ListToolsRequestSchema` handler.

---

## 7. The Six Tools

### 7.1 `add_family_member`

**Purpose:** Add a person to the household roster.

**Required parameters:** `user_id`, `name`
**Optional parameters:** `relationship`, `birth_date`, `notes`

**Implementation (lines 188–211):**
```typescript
const { data, error } = await supabase
  .from("family_members")
  .insert({
    user_id: args.user_id,
    name: args.name,
    relationship: args.relationship,
    birth_date: args.birth_date,
    notes: args.notes,
  })
  .select()
  .single();
```

A straightforward insert. `.select().single()` returns the newly created row including its
generated UUID, which the calling agent needs to use when adding activities for that person.

The `relationship` field is free-text. The tool description suggests values like `'self'`,
`'spouse'`, `'child'`, `'parent'` but does not enforce them. This is a deliberate
tradeoff: strict enumerations require schema migrations to extend. Open-text fields let
the agent adapt to unusual household structures without code changes.

---

### 7.2 `add_activity`

**Purpose:** Schedule a one-time event or a recurring weekly activity.

**Required parameters:** `user_id`, `title`
**Optional parameters:** `family_member_id`, `activity_type`, `day_of_week`, `start_time`,
`end_time`, `start_date`, `end_date`, `location`, `notes`

**Implementation (lines 213–242):**
```typescript
const { data, error } = await supabase
  .from("activities")
  .insert({
    user_id: args.user_id,
    family_member_id: args.family_member_id || null,
    title: args.title,
    activity_type: args.activity_type,
    day_of_week: args.day_of_week,
    start_time: args.start_time,
    end_time: args.end_time,
    start_date: args.start_date,
    end_date: args.end_date,
    location: args.location,
    notes: args.notes,
  })
  .select()
  .single();
```

The key pattern is `args.family_member_id || null`. If the AI agent passes an empty string
or omits the parameter entirely, both cases resolve to `null`, which the database treats as
"whole family." Without this coercion, an empty string would be inserted as a string,
which would fail the foreign key check against `family_members.id`.

To schedule a recurring event, the agent sets `day_of_week` (e.g., `'tuesday'`) and
leaves `start_date` as the first occurrence date. To schedule a one-time event, the agent
sets `start_date` to the event date and leaves `day_of_week` null.

---

### 7.3 `get_week_schedule`

**Purpose:** Return all activities active during a specified week.

This is the most complex tool in the extension. It is covered in detail in
[Section 8](#8-the-week-schedule-query--how-conflict-detection-works).

---

### 7.4 `search_activities`

**Purpose:** Find activities by title, type, or family member.

**Required parameters:** `user_id`
**Optional parameters:** `query`, `activity_type`, `family_member_id`

**Implementation (lines 281–318):**
```typescript
let query = supabase
  .from("activities")
  .select(`
    *,
    family_members:family_member_id (name, relationship)
  `)
  .eq("user_id", args.user_id);

if (args.query) {
  query = query.ilike("title", `%${args.query}%`);
}

if (args.activity_type) {
  query = query.eq("activity_type", args.activity_type);
}

if (args.family_member_id) {
  query = query.eq("family_member_id", args.family_member_id);
}

const { data, error } = await query.order("start_date", { ascending: false });
```

This is a progressively-built query. The base query is created first, then filter clauses
are conditionally appended based on which optional parameters were provided. The Supabase
JavaScript client returns a query builder object from each chained call, so the partial
query can be stored in a variable and extended before it is executed.

`ilike` is case-insensitive pattern matching. The `%` wildcards on both sides of the
search term match the pattern anywhere in the title — equivalent to SQL `ILIKE '%soccer%'`.
Without the wildcards, the match would require an exact full-string match.

All three optional filters are independent and additive: if all three are provided, the
query requires all three conditions to be true simultaneously.

---

### 7.5 `add_important_date`

**Purpose:** Record a date worth remembering — birthday, anniversary, deadline.

**Required parameters:** `user_id`, `title`, `date_value`
**Optional parameters:** `family_member_id`, `recurring_yearly`, `reminder_days_before`,
`notes`

**Implementation (lines 320–345):**
```typescript
const { data, error } = await supabase
  .from("important_dates")
  .insert({
    user_id: args.user_id,
    family_member_id: args.family_member_id || null,
    title: args.title,
    date_value: args.date_value,
    recurring_yearly: args.recurring_yearly || false,
    reminder_days_before: args.reminder_days_before || 7,
    notes: args.notes,
  })
  .select()
  .single();
```

The defaults are applied in the application layer with `|| false` and `|| 7` rather than
relying solely on the database defaults. This is belt-and-suspenders: even if the database
default is somehow removed, the application will still supply a value. The database schema
also specifies `DEFAULT false` and `DEFAULT 7`, so either layer would be sufficient alone.

---

### 7.6 `get_upcoming_dates`

**Purpose:** Return important dates falling within the next N days.

**Required parameters:** `user_id`
**Optional parameters:** `days_ahead` (defaults to 30)

**Implementation (lines 347–376):**
```typescript
const daysAhead = args.days_ahead || 30;
const today = new Date();
const futureDate = new Date();
futureDate.setDate(today.getDate() + daysAhead);

const { data, error } = await supabase
  .from("important_dates")
  .select(`
    *,
    family_members:family_member_id (name, relationship)
  `)
  .eq("user_id", args.user_id)
  .gte("date_value", today.toISOString().split("T")[0])
  .lte("date_value", futureDate.toISOString().split("T")[0])
  .order("date_value");
```

`.toISOString().split("T")[0]` converts a JavaScript `Date` object to the `YYYY-MM-DD`
string format that PostgreSQL's `DATE` type expects. `toISOString()` produces a full
ISO 8601 timestamp like `2026-03-13T14:30:00.000Z`. Splitting on `T` and taking index `[0]`
discards the time portion.

Note that this query uses the raw `date_value` from the database for filtering. For dates
where `recurring_yearly` is `true`, the stored `date_value` is the original year of the
event (e.g., `1990-06-15` for a 1990 birthday). A query for "upcoming dates in the next 30
days" using the stored value will only match in the original year. Handling yearly
recurrence — adjusting the date to the current year before querying — is left to the agent
or a future enhancement. The schema stores the intent (`recurring_yearly = true`); the full
recurrence logic is not yet implemented in the query layer.

---

## 8. The Week Schedule Query — How Conflict Detection Works

`get_week_schedule` (lines 244–279) is the most complex piece of code in the extension.
Understanding it fully requires understanding how the Supabase PostgREST query builder
translates into SQL.

### 8.1 Setting Up the Date Window

```typescript
const weekStart = new Date(args.week_start);
const weekEnd = new Date(weekStart);
weekEnd.setDate(weekEnd.getDate() + 7);
```

`week_start` is expected to be a Monday in `YYYY-MM-DD` format. Seven days are added to
produce the exclusive upper bound. If `week_start` is `2026-03-16` (Monday), then
`weekEnd` is `2026-03-23` (the following Monday). The window covers exactly seven days.

### 8.2 The Core Query

```typescript
let query = supabase
  .from("activities")
  .select(`
    *,
    family_members:family_member_id (name, relationship)
  `)
  .eq("user_id", args.user_id)
  .or(
    `and(start_date.lte.${weekEnd.toISOString().split("T")[0]},or(end_date.gte.${args.week_start},end_date.is.null)),day_of_week.not.is.null`
  );
```

The `.or()` clause is the core of the logic. Let us decompose it.

The string passed to `.or()` is a PostgREST filter expression. PostgREST is the HTTP API
layer that Supabase puts in front of PostgreSQL. It has its own filter syntax. The
Supabase JavaScript client translates chained method calls (`.eq()`, `.gte()`, etc.) into
PostgREST query parameters, but when you use `.or()` with nested `and()` conditions, you
pass the expression as a raw string.

The filter string is:
```
and(start_date.lte.WEEK_END, or(end_date.gte.WEEK_START, end_date.is.null)),
day_of_week.not.is.null
```

Parsed as a logical expression:
```
(
  start_date <= weekEnd
  AND (end_date >= weekStart OR end_date IS NULL)
)
OR
(
  day_of_week IS NOT NULL
)
```

**Branch 1 — Date-range events:**
`start_date <= weekEnd AND (end_date >= weekStart OR end_date IS NULL)`

This is the standard interval overlap test. Two intervals `[A, B]` and `[C, D]` overlap
when `A <= D AND B >= C`. Here:
- The event's interval is `[start_date, end_date]`
- The week's interval is `[weekStart, weekEnd]`
- Overlap condition: `start_date <= weekEnd AND end_date >= weekStart`
- `end_date IS NULL` catches one-time events (no end date) and ongoing recurring events

**Branch 2 — Recurring events:**
`day_of_week IS NOT NULL`

Any row with a non-null `day_of_week` is a recurring weekly event. The query retrieves
all of them for the user, regardless of their date range. The agent then has to determine
which occurrences actually fall in the requested week by examining `day_of_week` and
comparing it against the week's days.

This is a deliberate simplification. Filtering recurring events purely on `day_of_week IS
NOT NULL` will return recurring events that have already ended (where `end_date` is in the
past) or have not yet started. The query returns more rows than strictly necessary and
relies on the agent to interpret the `start_date`, `end_date`, and `day_of_week` fields
together to determine actual relevance.

A stricter query would combine both conditions:
```
day_of_week IS NOT NULL
AND start_date <= weekEnd
AND (end_date >= weekStart OR end_date IS NULL)
```

The current design trades query precision for simplicity. For a household calendar with
a small number of activities, the extra rows cause no practical problems.

### 8.3 Optional Family Member Filter

```typescript
if (args.family_member_id) {
  query = query.eq("family_member_id", args.family_member_id);
}
```

Applied after the `.or()`, this narrows the results to a single family member. The AI
agent can call `get_week_schedule` with no `family_member_id` to get the full household
view, or pass a specific ID to get one person's week.

### 8.4 Result Ordering

```typescript
const { data, error } = await query.order("start_time");
```

Results are sorted by `start_time` (a `TIME` column). This groups early-morning activities
before afternoon ones, regardless of date. The agent receives a flat array sorted by time
of day; grouping by day of week (Monday through Sunday) is left to the agent's
presentation layer.

---

## 9. Relational Selects

Two tools use a relational select pattern that deserves explicit explanation:

```typescript
.select(`
  *,
  family_members:family_member_id (name, relationship)
`)
```

This is a PostgREST feature called an embedded resource select. The syntax breaks down as:

- `*` — select all columns from the `activities` table
- `family_members:family_member_id` — follow the foreign key on `family_member_id` to the
  `family_members` table. The part before the colon is an alias for the embedded object in
  the response; it does not have to match the table name, but using the table name is
  conventional.
- `(name, relationship)` — from the joined row in `family_members`, include only these
  two columns

The result for each activity row looks like:
```json
{
  "id": "...",
  "title": "Soccer practice",
  "day_of_week": "tuesday",
  "start_time": "17:00:00",
  "family_member_id": "...",
  "family_members": {
    "name": "Emma",
    "relationship": "child"
  }
}
```

Without the embedded select, the agent would receive only `family_member_id` (a UUID) and
would need to make a separate query to `family_members` to resolve it to a name. The
embedded select collapses what would be a two-query operation into one.

Supabase translates this into a SQL JOIN internally. From the application developer's
perspective, it reads like a nested object access; from the database's perspective, it is
a `LEFT JOIN family_members ON activities.family_member_id = family_members.id` with a
column projection.

If `family_member_id` is `NULL` (whole-family activity), the `family_members` embedded
object in the response is `null`. The agent must handle this case.

---

## 10. No RLS — Why, and What Changes in Extension 4

Row Level Security is a PostgreSQL feature that attaches access control policies directly
to tables. When RLS is enabled and policies are defined, the database itself enforces that
a user can only see their own rows — regardless of what query is sent.

Extension 3 omits RLS entirely. The `schema.sql` file contains no `ALTER TABLE ... ENABLE
ROW LEVEL SECURITY` statement and no `CREATE POLICY` statements.

Logical isolation is still present: every query includes `.eq("user_id", args.user_id)`.
If the agent supplies the correct `user_id`, it only retrieves that user's data. If a
malicious or buggy request omitted the `user_id` filter, it would retrieve everyone's
data. RLS prevents that failure mode at the database layer; the absence of RLS means the
application layer is solely responsible.

For a single-user system where the MCP server runs on the user's own machine with their
own service role key, this is an acceptable tradeoff. The threat model does not include
other users of the same Supabase project.

Extension 4 (Meal Planning) changes this. It introduces shared household access: a spouse
or partner needs to view and update meal plans without having access to all of Open Brain.
That requires a scoped MCP server with a separate API key and RLS policies that restrict
what that key can see. The schema pattern becomes meaningfully more complex. Extensions 1
through 3 deliberately avoid that complexity so it can be introduced once, with full
explanation, at the point where it becomes necessary.

---

## 11. Indexes and Why They Matter Here

The schema defines five indexes:

```sql
CREATE INDEX idx_activities_user_dow ON activities(user_id, day_of_week);
CREATE INDEX idx_activities_family_member ON activities(family_member_id);
CREATE INDEX idx_activities_user_dates ON activities(user_id, start_date, end_date);
CREATE INDEX idx_important_dates_user_date ON important_dates(user_id, date_value);
CREATE INDEX idx_family_members_user ON family_members(user_id);
```

If you come from systems programming, think of an index as a sorted auxiliary data
structure — a B-tree by default in PostgreSQL — that lets the database locate rows
matching a filter without scanning every row in the table. Without an index, a query
like `.eq("user_id", x).eq("day_of_week", "tuesday")` would read every row in `activities`
and discard rows that do not match. With `idx_activities_user_dow`, the database jumps
directly to the matching rows.

Each index maps to a query pattern:

| Index | Supports |
|---|---|
| `idx_activities_user_dow` | `get_week_schedule` recurring branch (filter by user and day) |
| `idx_activities_family_member` | All queries that filter by `family_member_id` |
| `idx_activities_user_dates` | `get_week_schedule` date-range branch (filter by user, start, end) |
| `idx_important_dates_user_date` | `get_upcoming_dates` (filter by user, date range) |
| `idx_family_members_user` | Any lookup of all family members for a user |

For a household calendar with dozens of activities, these indexes are not necessary for
performance — PostgreSQL would scan the table faster than any observable latency. They
become meaningful if the same Supabase project is used across many users or if activities
accumulate over years. They also serve as documentation: the indexes tell you which query
patterns the schema author anticipated as the hot paths.

---

## 12. How This Server Fits with Other Extensions

The Family Calendar server is the connective tissue between earlier and later extensions.

**Extension 1 (Household Knowledge Base)** stores factual information: who the pediatrician
is, what the WiFi password is, when the car was last serviced. Extension 3 stores scheduled
time. An agent can query both: "Who is Emma's dentist?" (Extension 1) → "Schedule Emma's
checkup for next Tuesday" (Extension 3). The two servers run independently; the agent
coordinates between them.

**Extension 2 (Home Maintenance Tracker)** tracks tasks and due dates for the house
itself. Extension 3 tracks activities for the people in the house. Both use date-based
queries but for different entities. They share no database tables.

**Extension 4 (Meal Planning)** explicitly depends on Extension 3. The `family_members`
table created here is referenced by the meal planning extension to know how many people
are eating and who is home on any given night. The cross-extension dependency is handled
by the agent, not by a database foreign key: the meal planning server queries the same
Supabase project and reads from `family_members` directly.

**Extension 5 (Professional CRM)** uses the same parent-child relational pattern introduced
here: contacts (analogous to family members) and interactions (analogous to activities).
Understanding how `family_member_id` creates a nullable foreign key relationship that
enables both individual and whole-family activities prepares you for the contacts →
interactions model in Extension 5.

---

## 13. Source File Reference

| File | Lines | Purpose |
|---|---|---|
| `extensions/family-calendar/index.ts` | 403 | MCP server — all tool handlers and server wiring |
| `extensions/family-calendar/schema.sql` | 51 | Three tables and five indexes |
| `extensions/family-calendar/package.json` | — | Node.js dependencies |
| `extensions/family-calendar/tsconfig.json` | — | TypeScript compiler configuration |
| `extensions/family-calendar/metadata.json` | — | Extension metadata for the repo registry |
| `extensions/family-calendar/README.md` | — | Setup guide and usage instructions |

### Key line references in `index.ts`

| Lines | What happens |
|---|---|
| 11–18 | Environment variable validation and Supabase client initialization |
| 20–30 | MCP Server object creation |
| 32–181 | `ListToolsRequestSchema` handler with all six tool definitions inline |
| 183–392 | `CallToolRequestSchema` handler — the `switch` statement dispatching to each tool |
| 188–211 | `add_family_member` implementation |
| 213–242 | `add_activity` implementation |
| 244–279 | `get_week_schedule` implementation — the complex `.or()` query |
| 281–318 | `search_activities` implementation — progressive query building |
| 320–345 | `add_important_date` implementation |
| 347–376 | `get_upcoming_dates` implementation |
| 394–402 | `main()` function and process error handler |
