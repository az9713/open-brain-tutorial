# Extension 2: Home Maintenance Tracker MCP Server

Deep-dive reference for `extensions/home-maintenance/`. Covers every table, column,
index, RLS policy, trigger, and tool handler in full technical detail. Assumes you
are comfortable with SQL, TypeScript, and the MCP protocol basics.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Role in Architecture](#2-role-in-architecture)
3. [Database Schema](#3-database-schema)
   - [maintenance_tasks](#31-maintenance_tasks)
   - [maintenance_logs](#32-maintenance_logs)
   - [Indexes](#33-indexes)
   - [Row Level Security](#34-row-level-security)
   - [The Time-Bridging Trigger](#35-the-time-bridging-trigger)
4. [The Four Tools](#4-the-four-tools)
   - [add_maintenance_task](#41-add_maintenance_task)
   - [log_maintenance](#42-log_maintenance)
   - [get_upcoming_maintenance](#43-get_upcoming_maintenance)
   - [search_maintenance_history](#44-search_maintenance_history)
5. [Key Patterns](#5-key-patterns)
   - [Database Triggers vs. Application Logic](#51-database-triggers-vs-application-logic)
   - [One-to-Many Parent/Child Relationship](#52-one-to-many-parentchild-relationship)
   - [Date Arithmetic in PostgreSQL and JavaScript](#53-date-arithmetic-in-postgresql-and-javascript)
   - [Two-Step Relational Query](#54-two-step-relational-query)
6. [How It Fits in the Extension Path](#6-how-it-fits-in-the-extension-path)
7. [Source Files](#7-source-files)

---

## 1. Overview

The Home Maintenance Tracker gives an AI assistant the ability to manage recurring
and one-time home maintenance tasks, log work as it is completed, query what is
coming due, and search through historical records.

The central design decision is that the database — not the application code — owns
the responsibility of keeping a task's schedule current. When a maintenance log is
inserted, a PostgreSQL trigger fires, reads the task's recurrence interval, and
writes the new `last_completed` and `next_due` values back onto the parent task.
The MCP server never performs that calculation itself.

This pattern has a name used throughout this document: **time-bridging**. The
trigger bridges the gap between the moment work was done (recorded in a log row)
and the moment the next scheduled work is due (stored on the task row), so that
both are always consistent without requiring coordinated updates from the client.

---

## 2. Role in Architecture

```
+--------------------------------------------------------------+
|                      AI CLIENT (Claude, etc.)                |
+-----------------------------+--------------------------------+
                              | MCP stdio transport
+-----------------------------v--------------------------------+
|               home-maintenance MCP Server (Node.js)         |
|                                                              |
|   add_maintenance_task   log_maintenance                     |
|   get_upcoming_maintenance   search_maintenance_history      |
+----+-------------------------+-------------------------------+
     |  Supabase JS client     |
     |  (service-role key)     |
+----v-------------------------v-------------------------------+
|                         Supabase                             |
|                                                              |
|   maintenance_tasks  <----+  maintenance_logs               |
|   (parent / schedule)     |  (child / history)              |
|                           |                                  |
|   DB Trigger: update_task_after_maintenance_log              |
|   fires AFTER INSERT on maintenance_logs,                    |
|   writes last_completed + next_due back to task              |
+--------------------------------------------------------------+
```

The server has no dependencies on Extension 1 (Open Brain core) or any other
extension. It connects directly to Supabase using a service-role key and two
tables it owns entirely. A vendor or professional services firm could optionally
cross-reference the `name` or `category` columns against Extension 1's
`thoughts` table to surface relevant memory when a task comes due, but that
wiring is not built into this extension.

---

## 3. Database Schema

Source: `extensions/home-maintenance/schema.sql`

### 3.1 maintenance_tasks

```sql
CREATE TABLE IF NOT EXISTS maintenance_tasks (
    id               UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id          UUID        REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
    name             TEXT        NOT NULL,
    category         TEXT,
    frequency_days   INTEGER,
    last_completed   TIMESTAMPTZ,
    next_due         TIMESTAMPTZ,
    priority         TEXT        CHECK (priority IN ('low', 'medium', 'high', 'urgent'))
                                 DEFAULT 'medium',
    notes            TEXT,
    created_at       TIMESTAMPTZ DEFAULT now() NOT NULL,
    updated_at       TIMESTAMPTZ DEFAULT now() NOT NULL
);
```

Column-by-column:

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | UUID | No | Auto-generated primary key |
| `user_id` | UUID | No | FK to `auth.users`; cascades on user deletion |
| `name` | TEXT | No | Human-readable task name, e.g. "HVAC Filter Replacement" |
| `category` | TEXT | Yes | Free-text category: `hvac`, `plumbing`, `exterior`, `appliance`, `landscaping` |
| `frequency_days` | INTEGER | Yes | Recurrence interval in days. `NULL` means one-time. `90` = quarterly. `365` = annual |
| `last_completed` | TIMESTAMPTZ | Yes | Timestamp of last completion. Written by the trigger, not by the application |
| `next_due` | TIMESTAMPTZ | Yes | Next scheduled date. Written by the trigger for recurring tasks; `NULL` after a one-time task completes |
| `priority` | TEXT (enum) | No | `low` / `medium` (default) / `high` / `urgent` |
| `notes` | TEXT | Yes | Persistent notes about the task (e.g. "Use 16x25x1 pleated filters") |
| `created_at` | TIMESTAMPTZ | No | Set once on insert |
| `updated_at` | TIMESTAMPTZ | No | Updated automatically by the `update_updated_at_column` trigger on every modification |

The `frequency_days` column encodes the entire recurrence model. There is no
cron expression, no calendar rule, no separate schedule table. The interval is
a plain integer that the trigger adds to the completion timestamp. This keeps
the schema simple at the cost of not supporting irregular schedules (e.g.
"first Monday of the month").

### 3.2 maintenance_logs

```sql
CREATE TABLE IF NOT EXISTS maintenance_logs (
    id            UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id       UUID         REFERENCES maintenance_tasks(id) ON DELETE CASCADE NOT NULL,
    user_id       UUID         REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
    completed_at  TIMESTAMPTZ  DEFAULT now() NOT NULL,
    performed_by  TEXT,
    cost          DECIMAL(10,2),
    notes         TEXT,
    next_action   TEXT
);
```

Column-by-column:

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | UUID | No | Auto-generated primary key |
| `task_id` | UUID | No | FK to `maintenance_tasks(id)`; cascades on task deletion |
| `user_id` | UUID | No | FK to `auth.users`; redundant with the parent task but allows direct user-scoped queries on logs without a join |
| `completed_at` | TIMESTAMPTZ | No | When the work was performed. Defaults to `now()` but can be set to a past date to backfill history |
| `performed_by` | TEXT | Yes | Who did the work: `"self"`, a vendor name, a contractor company |
| `cost` | DECIMAL(10,2) | Yes | Monetary cost with two decimal places |
| `notes` | TEXT | Yes | Free-form notes about the specific work session |
| `next_action` | TEXT | Yes | Recommendations from the technician or contractor for future work |

Logs are append-only by convention. There is no update path provided by any
tool, which preserves an accurate audit trail. Deleting a task cascades to
delete all its logs.

### 3.3 Indexes

```sql
-- Used by get_upcoming_maintenance: filters by user, orders by next_due
CREATE INDEX IF NOT EXISTS idx_maintenance_tasks_user_next_due
    ON maintenance_tasks(user_id, next_due);

-- Used by search_maintenance_history when filtering by task: orders logs for a task newest-first
CREATE INDEX IF NOT EXISTS idx_maintenance_logs_task_completed
    ON maintenance_logs(task_id, completed_at DESC);

-- Used by search_maintenance_history when no task filter is applied: user's logs newest-first
CREATE INDEX IF NOT EXISTS idx_maintenance_logs_user_completed
    ON maintenance_logs(user_id, completed_at DESC);
```

All three indexes match the exact column ordering used in the queries. The
composite `(user_id, next_due)` index on tasks means that the upcoming query
is a single index range scan with no heap fetches for the filter, only for
the selected columns.

### 3.4 Row Level Security

```sql
ALTER TABLE maintenance_tasks ENABLE ROW LEVEL SECURITY;
ALTER TABLE maintenance_logs  ENABLE ROW LEVEL SECURITY;

CREATE POLICY maintenance_tasks_user_policy ON maintenance_tasks
    FOR ALL
    USING (auth.uid() = user_id)
    WITH CHECK (auth.uid() = user_id);

CREATE POLICY maintenance_logs_user_policy ON maintenance_logs
    FOR ALL
    USING (auth.uid() = user_id)
    WITH CHECK (auth.uid() = user_id);
```

Both tables are locked to the authenticated user's own rows. The MCP server
uses a service-role key which bypasses RLS, so user isolation is enforced by
passing `user_id` explicitly in every query rather than relying on
`auth.uid()`. This is the same pattern used across all Open Brain extensions
when operating from a server-side context.

### 3.5 The Time-Bridging Trigger

This is the most important mechanism in the extension. Read it carefully.

```sql
-- schema.sql lines 77–107

CREATE OR REPLACE FUNCTION update_task_after_maintenance_log()
RETURNS TRIGGER AS $$
DECLARE
    task_frequency INTEGER;
BEGIN
    -- Step 1: Read the recurrence interval from the parent task
    SELECT frequency_days INTO task_frequency
    FROM maintenance_tasks
    WHERE id = NEW.task_id;

    -- Step 2: Write last_completed and recalculate next_due on the parent task
    UPDATE maintenance_tasks
    SET
        last_completed = NEW.completed_at,
        next_due = CASE
            WHEN task_frequency IS NOT NULL
                THEN NEW.completed_at + (task_frequency || ' days')::INTERVAL
            ELSE NULL
        END,
        updated_at = now()
    WHERE id = NEW.task_id;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

DROP TRIGGER IF EXISTS update_task_after_log ON maintenance_logs;
CREATE TRIGGER update_task_after_log
    AFTER INSERT ON maintenance_logs
    FOR EACH ROW
    EXECUTE FUNCTION update_task_after_maintenance_log();
```

What happens when `log_maintenance` inserts a row:

```
INSERT into maintenance_logs
         |
         v
  PostgreSQL fires update_task_after_log (AFTER INSERT, per row)
         |
         v
  Function reads frequency_days from maintenance_tasks WHERE id = NEW.task_id
         |
         +-- frequency_days IS NULL (one-time task)
         |         next_due = NULL          (task is retired from the schedule)
         |
         +-- frequency_days = 90 (quarterly)
                   next_due = completed_at + INTERVAL '90 days'
                   (e.g. completed 2026-03-13 -> next_due 2026-06-11)
         |
         v
  UPDATE maintenance_tasks SET last_completed, next_due, updated_at
         |
         v
  Control returns to application
```

The expression `(task_frequency || ' days')::INTERVAL` deserves explanation.
PostgreSQL cannot directly cast an integer to an interval, but it can cast a
string like `'90 days'` to one. Concatenating the integer with the literal
string `' days'` and then casting the result is a standard PostgreSQL idiom
for building variable-length intervals from integer columns.

Because this runs inside the same database transaction as the `INSERT`, there
is no window in which the log exists but the parent task still shows stale
schedule data. The parent is always consistent with its most recent log entry.

There is also a second, simpler trigger keeping `updated_at` current:

```sql
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_maintenance_tasks_updated_at
    BEFORE UPDATE ON maintenance_tasks
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

This fires `BEFORE UPDATE` (rather than after insert like the time-bridging
trigger) and simply stamps the current time. It fires automatically when the
time-bridging trigger executes its `UPDATE maintenance_tasks`, so `updated_at`
is always current without any explicit assignment in application code.

---

## 4. The Four Tools

Source: `extensions/home-maintenance/index.ts`

### 4.1 add_maintenance_task

**Purpose:** Create a new task record, either recurring or one-time.

**Required parameters:** `user_id`, `name`

**Optional parameters:** `category`, `frequency_days`, `next_due`, `priority`,
`notes`

**Handler (index.ts lines 199–225):**

```typescript
async function handleAddMaintenanceTask(args: any): Promise<string> {
  const { user_id, name, category, frequency_days, next_due, priority, notes } = args;

  const { data, error } = await supabase
    .from("maintenance_tasks")
    .insert({
      user_id,
      name,
      category: category || null,
      frequency_days: frequency_days || null,
      next_due: next_due || null,
      priority: priority || "medium",
      notes: notes || null,
    })
    .select()
    .single();

  if (error) {
    throw new Error(`Failed to add maintenance task: ${error.message}`);
  }

  return JSON.stringify({
    success: true,
    message: `Added maintenance task: ${name}`,
    task: data,
  }, null, 2);
}
```

The handler does a single `INSERT ... RETURNING *` (`.select().single()`
achieves this in the Supabase client). The `|| null` fallback pattern ensures
that omitted optional fields are stored as SQL `NULL` rather than the
JavaScript `undefined` or an empty string.

`next_due` is set by the caller at creation time. For a new recurring task
that has never been completed, `last_completed` starts as `NULL` and the
trigger has not yet fired, so the initial `next_due` must come from the
application. Subsequent `next_due` values are owned by the trigger.

**Example call:**

```json
{
  "name": "add_maintenance_task",
  "arguments": {
    "user_id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "HVAC Filter Replacement",
    "category": "hvac",
    "frequency_days": 90,
    "next_due": "2026-04-15",
    "priority": "medium",
    "notes": "Use 16x25x1 pleated filters"
  }
}
```

**Example response:**

```json
{
  "success": true,
  "message": "Added maintenance task: HVAC Filter Replacement",
  "task": {
    "id": "a1b2c3d4-...",
    "user_id": "550e8400-...",
    "name": "HVAC Filter Replacement",
    "category": "hvac",
    "frequency_days": 90,
    "last_completed": null,
    "next_due": "2026-04-15T00:00:00+00:00",
    "priority": "medium",
    "notes": "Use 16x25x1 pleated filters",
    "created_at": "2026-03-13T10:00:00+00:00",
    "updated_at": "2026-03-13T10:00:00+00:00"
  }
}
```

---

### 4.2 log_maintenance

**Purpose:** Record that a task was completed. Relies on the database trigger
to update the parent task's schedule.

**Required parameters:** `task_id`, `user_id`

**Optional parameters:** `completed_at`, `performed_by`, `cost`, `notes`,
`next_action`

**Handler (index.ts lines 227–267):**

```typescript
async function handleLogMaintenance(args: any): Promise<string> {
  const { task_id, user_id, completed_at, performed_by, cost, notes, next_action } = args;

  // Step 1: Insert the log.
  // The database trigger fires here, updating the parent task automatically.
  const { data, error } = await supabase
    .from("maintenance_logs")
    .insert({
      task_id,
      user_id,
      completed_at: completed_at || new Date().toISOString(),
      performed_by: performed_by || null,
      cost: cost || null,
      notes: notes || null,
      next_action: next_action || null,
    })
    .select()
    .single();

  if (error) {
    throw new Error(`Failed to log maintenance: ${error.message}`);
  }

  // Step 2: Fetch the updated task to show the new next_due to the user.
  // This is a read-after-write: the trigger has already run, so the task
  // reflects the new schedule.
  const { data: task, error: taskError } = await supabase
    .from("maintenance_tasks")
    .select("*")
    .eq("id", task_id)
    .single();

  if (taskError) {
    console.error("Warning: Could not fetch updated task:", taskError.message);
  }

  return JSON.stringify({
    success: true,
    message: "Maintenance logged successfully",
    log: data,
    updated_task: task,
  }, null, 2);
}
```

The sequence of events in a single `log_maintenance` call:

```
Client calls log_maintenance
       |
       v
Handler inserts into maintenance_logs (Step 1)
       |
       v  [within the same DB transaction]
Trigger update_task_after_log fires
       |
       +-- reads frequency_days from maintenance_tasks
       +-- writes last_completed, next_due, updated_at to maintenance_tasks
       |
       v
Insert commit completes; supabase client returns the new log row
       |
       v
Handler fetches maintenance_tasks WHERE id = task_id (Step 2)
       |
       v
Returns { log: <new log row>, updated_task: <task with updated schedule> }
```

The `taskError` check uses `console.error` rather than `throw` because the
log was already committed. Failing to read the updated task back is a minor
UX degradation (the response omits `updated_task`), not a data integrity
problem. The warning is written to stderr so it appears in the server's
process log without surfacing as a tool error to the AI client.

**Example call:**

```json
{
  "name": "log_maintenance",
  "arguments": {
    "task_id": "a1b2c3d4-...",
    "user_id": "550e8400-...",
    "completed_at": "2026-03-13T09:30:00Z",
    "performed_by": "self",
    "cost": 12.50,
    "notes": "Replaced with Filtrete 1500 16x25x1",
    "next_action": "Check for mold around intake next time"
  }
}
```

**Example response (quarterly task, `frequency_days = 90`):**

```json
{
  "success": true,
  "message": "Maintenance logged successfully",
  "log": {
    "id": "f9e8d7c6-...",
    "task_id": "a1b2c3d4-...",
    "user_id": "550e8400-...",
    "completed_at": "2026-03-13T09:30:00+00:00",
    "performed_by": "self",
    "cost": "12.50",
    "notes": "Replaced with Filtrete 1500 16x25x1",
    "next_action": "Check for mold around intake next time"
  },
  "updated_task": {
    "id": "a1b2c3d4-...",
    "last_completed": "2026-03-13T09:30:00+00:00",
    "next_due": "2026-06-11T09:30:00+00:00",
    ...
  }
}
```

---

### 4.3 get_upcoming_maintenance

**Purpose:** List all tasks due within the next N days, ordered earliest first.

**Required parameters:** `user_id`

**Optional parameters:** `days_ahead` (default `30`)

**Handler (index.ts lines 269–293):**

```typescript
async function handleGetUpcomingMaintenance(args: any): Promise<string> {
  const { user_id, days_ahead = 30 } = args;

  const cutoffDate = new Date();
  cutoffDate.setDate(cutoffDate.getDate() + days_ahead);

  const { data, error } = await supabase
    .from("maintenance_tasks")
    .select("*")
    .eq("user_id", user_id)
    .not("next_due", "is", null)
    .lte("next_due", cutoffDate.toISOString())
    .order("next_due", { ascending: true });

  if (error) {
    throw new Error(`Failed to get upcoming maintenance: ${error.message}`);
  }

  return JSON.stringify({
    success: true,
    days_ahead,
    count: data.length,
    tasks: data,
  }, null, 2);
}
```

The cutoff calculation uses the JavaScript `Date` object:

```
today = new Date()              // e.g. 2026-03-13T10:00:00Z
cutoffDate = today + 30 days    // 2026-04-12T10:00:00Z
```

The query translates to:

```sql
SELECT *
FROM maintenance_tasks
WHERE user_id = $1
  AND next_due IS NOT NULL
  AND next_due <= $2          -- $2 = cutoffDate.toISOString()
ORDER BY next_due ASC;
```

The `.not("next_due", "is", null)` clause is important: one-time tasks that
have been completed have `next_due = NULL` (set by the trigger). Without this
filter they would appear in every upcoming query because `NULL <= any_date`
evaluates to `NULL` (unknown) in SQL, but the Supabase client translates the
`.not("next_due", "is", null)` clause to `next_due IS NOT NULL`, which
correctly excludes them.

The `(user_id, next_due)` composite index directly serves this query.

**Note on overdue tasks:** The query uses `lte` (less than or equal to), not
`gte`. Tasks whose `next_due` is in the past (already overdue) are included in
the results because `next_due <= cutoffDate` is true for any date earlier than
the cutoff. A task due last month will appear every time you call
`get_upcoming_maintenance` until it is logged as completed.

---

### 4.4 search_maintenance_history

**Purpose:** Search through completed maintenance logs, optionally filtered by
task name, category, and/or date range.

**Required parameters:** `user_id`

**Optional parameters:** `task_name`, `category`, `date_from`, `date_to`

**Handler (index.ts lines 295–369):**

```typescript
async function handleSearchMaintenanceHistory(args: any): Promise<string> {
  const { user_id, task_name, category, date_from, date_to } = args;

  // Phase 1: If filtering by task name or category, find the relevant task IDs first.
  // maintenance_logs does not store name or category directly, so we must
  // resolve them via the parent table before querying logs.
  let taskIds: string[] | null = null;

  if (task_name || category) {
    let taskQuery = supabase
      .from("maintenance_tasks")
      .select("id")
      .eq("user_id", user_id);

    if (task_name) {
      taskQuery = taskQuery.ilike("name", `%${task_name}%`);
    }

    if (category) {
      taskQuery = taskQuery.ilike("category", `%${category}%`);
    }

    const { data: tasks, error: taskError } = await taskQuery;

    if (taskError) {
      throw new Error(`Failed to search tasks: ${taskError.message}`);
    }

    taskIds = tasks.map(t => t.id);

    if (taskIds.length === 0) {
      // Early exit: no tasks match the name/category filter,
      // so there can be no matching logs.
      return JSON.stringify({ success: true, count: 0, logs: [] }, null, 2);
    }
  }

  // Phase 2: Query logs, optionally scoped to the task IDs resolved above.
  // The relational select syntax embeds the parent task's name and category
  // directly in each log row.
  let logQuery = supabase
    .from("maintenance_logs")
    .select(`
      *,
      maintenance_tasks (
        id,
        name,
        category
      )
    `)
    .eq("user_id", user_id);

  if (taskIds) {
    logQuery = logQuery.in("task_id", taskIds);
  }

  if (date_from) {
    logQuery = logQuery.gte("completed_at", date_from);
  }

  if (date_to) {
    logQuery = logQuery.lte("completed_at", date_to);
  }

  const { data, error } = await logQuery.order("completed_at", { ascending: false });

  if (error) {
    throw new Error(`Failed to search maintenance history: ${error.message}`);
  }

  return JSON.stringify({
    success: true,
    count: data.length,
    logs: data,
  }, null, 2);
}
```

The two-phase approach exists because the text filters (`task_name`,
`category`) live on `maintenance_tasks` but the date filters (`date_from`,
`date_to`) live on `maintenance_logs`. There is no single table that holds all
four filter columns. Rather than a SQL `JOIN`, the handler uses Supabase's
PostgREST-style relational `select` to embed the parent task inline:

```
maintenance_logs row
  + maintenance_tasks { id, name, category }   <-- nested object in the response
```

The `.ilike()` calls generate `ILIKE '%value%'` predicates, giving
case-insensitive substring matching on both `name` and `category`.

**Example call (search by category and date range):**

```json
{
  "name": "search_maintenance_history",
  "arguments": {
    "user_id": "550e8400-...",
    "category": "hvac",
    "date_from": "2025-01-01",
    "date_to": "2026-03-13"
  }
}
```

**Example response:**

```json
{
  "success": true,
  "count": 2,
  "logs": [
    {
      "id": "f9e8d7c6-...",
      "task_id": "a1b2c3d4-...",
      "user_id": "550e8400-...",
      "completed_at": "2026-03-13T09:30:00+00:00",
      "performed_by": "self",
      "cost": "12.50",
      "notes": "Replaced with Filtrete 1500 16x25x1",
      "next_action": "Check for mold around intake next time",
      "maintenance_tasks": {
        "id": "a1b2c3d4-...",
        "name": "HVAC Filter Replacement",
        "category": "hvac"
      }
    },
    ...
  ]
}
```

---

## 5. Key Patterns

### 5.1 Database Triggers vs. Application Logic

The time-bridging trigger moves business logic from the application layer into
the database. The trade-off is:

| Aspect | Application logic | Database trigger |
|---|---|---|
| Visibility | Easy to read in source code | Requires knowing to look in schema.sql |
| Consistency | Requires explicit coordination (e.g. two UPDATE calls) | Atomic with the INSERT |
| Reuse | Only fires when called through the MCP server | Fires for any client that inserts into maintenance_logs |
| Testability | Straightforward unit tests | Requires a running Postgres instance |
| Latency | Extra round-trip to recalculate and update | Single round-trip; update happens inside the same transaction |

For a schedule calculation that must remain consistent with the log record
that triggered it, the database trigger is the better fit. If the trigger
were removed and the application handled the update, a crash between the
`INSERT` and the subsequent `UPDATE` would leave `last_completed` and
`next_due` permanently stale.

### 5.2 One-to-Many Parent/Child Relationship

`maintenance_tasks` is the parent. `maintenance_logs` is the child. Each task
can have zero or more logs. Each log belongs to exactly one task.

The cascade behavior on the foreign keys means:
- Deleting a user cascades to delete all their tasks, which cascades to delete
  all their logs.
- Deleting a task cascades to delete all its logs.
- Deleting a log does not affect the parent task (and does not reverse the
  trigger's update to `last_completed` and `next_due`).

The `user_id` column on `maintenance_logs` is denormalized — it could be
derived by joining through `task_id -> maintenance_tasks.user_id`. It is
stored directly to support queries that filter logs by user without requiring
a join to the tasks table, as the user-scoped log query and the RLS policy
both do.

### 5.3 Date Arithmetic in PostgreSQL and JavaScript

Two different environments perform date arithmetic in this extension:

**PostgreSQL (trigger, `schema.sql` lines 91–93):**

```sql
next_due = NEW.completed_at + (task_frequency || ' days')::INTERVAL
```

This is timezone-aware: `completed_at` is `TIMESTAMPTZ`, and adding an
interval to a `TIMESTAMPTZ` value preserves the timezone offset. The result
is also a `TIMESTAMPTZ`. Adding 90 days to
`2026-03-13T09:30:00+00:00` yields `2026-06-11T09:30:00+00:00`, exactly 90
24-hour periods later regardless of DST transitions in the UTC-normalized
representation.

**JavaScript (handler, `index.ts` lines 272–273):**

```typescript
const cutoffDate = new Date();
cutoffDate.setDate(cutoffDate.getDate() + days_ahead);
```

`setDate` mutates the `Date` object by adding calendar days to the current
local date. This is the standard idiom but has a subtle implication: if the
server's local timezone observes DST, a `days_ahead = 30` calculation that
spans a DST boundary will result in a cutoff that is 30 calendar days later
but either 29 or 31 days away in UTC hours. For practical purposes — deciding
whether to show a maintenance task in a "next 30 days" list — this is
acceptable.

### 5.4 Two-Step Relational Query

`search_maintenance_history` deliberately avoids a SQL `JOIN` and uses two
sequential queries instead. This is idiomatic for the Supabase JavaScript
client, which builds PostgREST queries rather than raw SQL. The two-step
pattern is:

1. Query the "dimension" table (`maintenance_tasks`) to get a set of IDs
   that match the filter criteria.
2. Query the "fact" table (`maintenance_logs`) with `.in("task_id", taskIds)`
   to fetch rows whose foreign key falls in that set.

The relational `.select()` syntax in Phase 2:

```typescript
.select(`
  *,
  maintenance_tasks (
    id,
    name,
    category
  )
`)
```

generates a PostgREST request that asks Supabase to embed the joined parent
columns as a nested object. Under the hood Supabase issues a single query
with a subselect, not two queries. The result is that each log row in the
response carries its parent task's `id`, `name`, and `category` inline,
removing the need for the caller to perform a second lookup.

---

## 6. How It Fits in the Extension Path

Extension 2 introduces two concepts that are prerequisites for later builds:

**Parent/child tables with cascading deletes.** The `maintenance_tasks` /
`maintenance_logs` relationship is the first one-to-many structure in the
extension path. Later extensions that track items with associated events
(e.g. vehicles and service records, recipes and cook logs) use the same
pattern.

**Database triggers for derived state.** The time-bridging trigger is the
first example of moving computed state into the database layer. Later
extensions use triggers for similar purposes: auto-populating denormalized
columns, maintaining running totals, and synchronizing related records
without requiring coordinated application updates.

**Cross-reference opportunity with Extension 1.** This extension has no
runtime dependency on Extension 1. A vendor or power user could optionally
add a step to `log_maintenance` that writes a corresponding entry into
Extension 1's `thoughts` table (e.g. "Replaced HVAC filter on 2026-03-13,
cost $12.50") so the AI has a searchable memory of completed home
maintenance. That wiring is not in scope for this extension but requires
only adding a second Supabase insert call in `handleLogMaintenance` after
the existing one.

---

## 7. Source Files

| File | Lines | Description |
|---|---|---|
| `extensions/home-maintenance/index.ts` | 425 | MCP server: environment setup, type definitions, tool schemas, four handlers, server registration and startup |
| `extensions/home-maintenance/schema.sql` | 114 | Two tables, three indexes, two RLS policies, two trigger functions, two trigger declarations, commented sample data |

**Key line references in `index.ts`:**

| Lines | Content |
|---|---|
| 1–41 | Environment validation and Supabase client initialization |
| 44–68 | TypeScript interfaces: `MaintenanceTask`, `MaintenanceLog` |
| 70–196 | `TOOLS` array: four tool definitions with full JSON Schema input schemas |
| 199–225 | `handleAddMaintenanceTask` |
| 227–267 | `handleLogMaintenance` (trigger interaction described in comment at line 231) |
| 269–293 | `handleGetUpcomingMaintenance` |
| 295–369 | `handleSearchMaintenanceHistory` |
| 372–412 | Server setup, request handler registration, dispatch switch |
| 415–424 | `main()` entry point and error handler |

**Key line references in `schema.sql`:**

| Lines | Content |
|---|---|
| 6–18 | `maintenance_tasks` table definition |
| 22–31 | `maintenance_logs` table definition |
| 34–41 | Three indexes |
| 44–57 | RLS enable and policies |
| 60–73 | `update_updated_at_column` function and trigger |
| 77–107 | `update_task_after_maintenance_log` function and `update_task_after_log` trigger (the time-bridging mechanism) |
| 109–113 | Commented sample INSERT statements |
