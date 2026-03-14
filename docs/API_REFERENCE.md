# Open Brain — API Reference

Complete reference for the Open Brain MCP Tools API, extension MCP servers, database schema, and authentication model. This document is the authoritative source for developers building on top of Open Brain.

---

## Table of Contents

1. [Core MCP Tools API](#1-core-mcp-tools-api)
2. [Extension MCP Tools](#2-extension-mcp-tools)
3. [Database Schema Reference](#3-database-schema-reference)
4. [Authentication and Access](#4-authentication-and-access)
5. [Environment Variables and Secrets](#5-environment-variables-and-secrets)
6. [Metadata JSONB Structure](#6-metadata-jsonb-structure)
7. [Cross-Extension Integration](#7-cross-extension-integration)
8. [Error Handling](#8-error-handling)

---

## 1. Core MCP Tools API

The core MCP server is deployed as a Supabase Edge Function. It provides four tools that give any MCP-connected AI client the ability to read and write to the `thoughts` table.

> **CLI alternative:** The [`ob` CLI tool](../resources/ob-cli/) provides the same four capabilities (capture, search, recent, stats) as direct shell commands using `curl` and `jq`. It talks to the same Supabase REST API and `match_thoughts()` RPC function documented below — no MCP server required. See [CLI-Direct Approach](CLI_DIRECT_APPROACH.md) for the full architecture.

**Deployment path:** `supabase/functions/open-brain-mcp/index.ts`

**Live URL pattern:**

```
https://YOUR_PROJECT_REF.supabase.co/functions/v1/open-brain-mcp
```

**Dependencies** (declared in `supabase/functions/open-brain-mcp/deno.json`):

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

---

### capture_thought

Stores a new thought with automatic embedding generation and metadata extraction.

**Purpose:** The primary write path. Takes a string, generates a 1536-dimensional vector embedding, extracts structured metadata via LLM, and stores a single row in the `thoughts` table.

**Input schema:**

| Parameter | Type   | Required | Description                     |
| --------- | ------ | -------- | ------------------------------- |
| `content` | string | Yes      | The text to capture and embed   |

**Processing pipeline:**

1. Receives `content` string from the AI client
2. Calls `text-embedding-3-small` via OpenRouter to generate a `vector(1536)` embedding
3. Calls `gpt-4o-mini` via OpenRouter to extract structured metadata:
   - `type` — classification of the thought
   - `topics` — array of topic strings
   - `people` — array of names mentioned
   - `action_items` — array of follow-up tasks
4. Inserts a single row into the `thoughts` table with `content`, `embedding`, and `metadata`
5. Returns a confirmation message with the extracted metadata

**Output format:**

```json
{
  "success": true,
  "id": "uuid",
  "content": "Sarah mentioned she wants to start consulting",
  "metadata": {
    "type": "person_note",
    "topics": ["consulting", "career"],
    "people": ["Sarah"],
    "action_items": []
  }
}
```

**Example prompt:**

```
Remember this: Sarah mentioned she wants to start consulting
```

**Notes:**

- Embedding and metadata extraction happen in parallel for performance
- Metadata extraction is best-effort — the LLM makes its best guess from the content alone
- The embedding is what powers semantic search; metadata is an additional filter layer
- `SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` are auto-available inside Edge Functions; `OPENROUTER_API_KEY` must be set via `supabase secrets set`

---

### search_thoughts

Performs semantic vector search across all stored thoughts.

**Purpose:** The primary read path. Converts the query into an embedding and calls the `match_thoughts()` PostgreSQL function to find semantically similar thoughts.

**Input schema:**

| Parameter   | Type   | Required | Default | Description                                              |
| ----------- | ------ | -------- | ------- | -------------------------------------------------------- |
| `query`     | string | Yes      | —       | The search query (converted to embedding for comparison) |
| `threshold` | float  | No       | `0.7`   | Minimum similarity score (0.0–1.0)                       |
| `count`     | int    | No       | `10`    | Maximum number of results to return                      |
| `filter`    | JSONB  | No       | `{}`    | Metadata filter applied before similarity ranking        |

**Processing pipeline:**

1. Generates an embedding of the `query` string via `text-embedding-3-small`
2. Calls `match_thoughts(query_embedding, match_threshold, match_count, filter)` PostgreSQL function
3. Returns results ranked by cosine similarity in descending order

**Output format:**

```json
{
  "success": true,
  "count": 2,
  "results": [
    {
      "id": "uuid",
      "content": "Sarah wants to start consulting",
      "metadata": { "type": "person_note", "people": ["Sarah"], "topics": ["consulting"] },
      "similarity": 0.89,
      "created_at": "2026-03-10T14:22:00Z"
    }
  ]
}
```

**Filter examples:**

```json
// Only return thoughts of type "task"
{ "type": "task" }

// Only return thoughts mentioning a specific topic
{ "topics": ["consulting"] }
```

**Notes:**

- Similarity is cosine similarity calculated as `1 - (embedding <=> query_embedding)`
- Lower threshold (e.g., `0.3`) returns more results with looser semantic matching
- The `filter` parameter uses PostgreSQL's `@>` JSONB containment operator
- "career changes" will match "Sarah wants to start consulting" even with zero keyword overlap

---

### list_thoughts

Browses recent thoughts in reverse chronological order.

**Also referred to as:** `browse_recent`

**Purpose:** Paginated listing of stored thoughts without semantic search. Useful for reviewing recent captures.

**Input schema:**

| Parameter | Type | Required | Default | Description                          |
| --------- | ---- | -------- | ------- | ------------------------------------ |
| `count`   | int  | No       | `10`    | Number of thoughts to return         |
| `offset`  | int  | No       | `0`     | Number of thoughts to skip (for pagination) |

**Output format:**

```json
{
  "success": true,
  "count": 10,
  "thoughts": [
    {
      "id": "uuid",
      "content": "Decided to move launch to March 15 because of QA blockers",
      "metadata": { "type": "observation", "topics": ["launch", "QA"] },
      "created_at": "2026-03-13T09:00:00Z"
    }
  ]
}
```

**Notes:**

- Results are ordered by `created_at DESC`
- No embedding is generated for this tool (pure SQL query)

---

### stats

Returns an overview of the thoughts database.

**Purpose:** Provides a summary of what is stored — total count, date range, and topic distribution. Useful for understanding the state of the brain without reading individual records.

**Input schema:** None

**Output format:**

```json
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

---

## 2. Extension MCP Tools

Each extension is a standalone Node.js MCP server using `@modelcontextprotocol/sdk`. All extensions share the same pattern: they connect to Supabase using `SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY`, and expose tools via stdio transport.

**Common server startup pattern** (from `extensions/*/index.ts`):

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!,
  { auth: { autoRefreshToken: false, persistSession: false } }
);
```

**All tools return one of two shapes:**

Success:
```json
{ "content": [{ "type": "text", "text": "{ \"success\": true, ... }" }] }
```

Error:
```json
{ "content": [{ "type": "text", "text": "{ \"success\": false, \"error\": \"...\" }" }], "isError": true }
```

---

### Extension 1: Household Knowledge Base

**Source:** `extensions/household-knowledge/index.ts`
**Schema:** `extensions/household-knowledge/schema.sql`
**Tables:** `household_items`, `household_vendors`

All tools require `user_id` (UUID matching `auth.users.id`).

---

#### add_household_item

Adds a new household item record.

| Parameter  | Type   | Required | Description                                                       |
| ---------- | ------ | -------- | ----------------------------------------------------------------- |
| `user_id`  | UUID   | Yes      | User ID                                                           |
| `name`     | string | Yes      | Name or description of the item                                   |
| `category` | string | No       | Category: `"paint"`, `"appliance"`, `"measurement"`, `"document"` |
| `location` | string | No       | Location in the home: `"Living Room"`, `"Kitchen"`, etc.          |
| `details`  | object | No       | Flexible JSONB metadata (brand, model, color, serial number, etc.)|
| `notes`    | string | No       | Additional notes or context                                       |

**Success response:**

```json
{
  "success": true,
  "message": "Added household item: Living Room Paint",
  "item": {
    "id": "uuid",
    "user_id": "uuid",
    "name": "Living Room Paint",
    "category": "paint",
    "location": "Living Room",
    "details": { "brand": "Sherwin Williams", "color": "Sea Salt", "code": "SW 6204" },
    "notes": "Purchased 2 gallons in March 2025",
    "created_at": "2026-03-13T09:00:00Z",
    "updated_at": "2026-03-13T09:00:00Z"
  }
}
```

---

#### search_household_items

Searches items by name, category, or location using ILIKE pattern matching.

| Parameter  | Type   | Required | Description                                      |
| ---------- | ------ | -------- | ------------------------------------------------ |
| `user_id`  | UUID   | Yes      | User ID                                          |
| `query`    | string | No       | Searches `name`, `category`, `location`, `notes` |
| `category` | string | No       | Exact category filter (ILIKE, partial match)     |
| `location` | string | No       | Exact location filter (ILIKE, partial match)     |

**Notes:** When `query` is provided, it applies an OR across `name`, `category`, `location`, and `notes` using ILIKE. Results are ordered by `created_at DESC`.

---

#### get_item_details

Retrieves the full record for a specific item.

| Parameter | Type | Required | Description                          |
| --------- | ---- | -------- | ------------------------------------ |
| `item_id` | UUID | Yes      | Item ID                              |
| `user_id` | UUID | Yes      | User ID (enforces ownership check)   |

**Notes:** Returns a 404-style error if the item does not exist or if `user_id` does not match the stored record.

---

#### add_vendor

Adds a service provider record.

| Parameter      | Type   | Required | Description                                              |
| -------------- | ------ | -------- | -------------------------------------------------------- |
| `user_id`      | UUID   | Yes      | User ID                                                  |
| `name`         | string | Yes      | Vendor name                                              |
| `service_type` | string | No       | Type of service: `"plumber"`, `"electrician"`, etc.      |
| `phone`        | string | No       | Phone number                                             |
| `email`        | string | No       | Email address                                            |
| `website`      | string | No       | Website URL                                              |
| `notes`        | string | No       | Additional notes                                         |
| `rating`       | number | No       | Rating 1–5 (enforced by DB CHECK constraint)             |
| `last_used`    | string | No       | Date last used, `YYYY-MM-DD` format                      |

---

#### list_vendors

Lists all vendors for a user, optionally filtered by service type.

| Parameter      | Type   | Required | Description                              |
| -------------- | ------ | -------- | ---------------------------------------- |
| `user_id`      | UUID   | Yes      | User ID                                  |
| `service_type` | string | No       | Filter by service type (ILIKE, partial)  |

**Notes:** Results are ordered by `name ASC`.

---

### Extension 2: Home Maintenance Tracker

**Source:** `extensions/home-maintenance/index.ts`
**Schema:** `extensions/home-maintenance/schema.sql`
**Tables:** `maintenance_tasks`, `maintenance_logs`

**Key behavior:** When a log entry is inserted via `log_maintenance`, a database trigger (`update_task_after_log`) automatically updates the parent task's `last_completed` and recalculates `next_due` based on `frequency_days`.

---

#### add_maintenance_task

Creates a recurring or one-time maintenance task.

| Parameter        | Type   | Required | Description                                               |
| ---------------- | ------ | -------- | --------------------------------------------------------- |
| `user_id`        | UUID   | Yes      | User ID                                                   |
| `name`           | string | Yes      | Task name                                                 |
| `category`       | string | No       | Category: `"hvac"`, `"plumbing"`, `"exterior"`, `"appliance"`, `"landscaping"` |
| `frequency_days` | number | No       | Recurrence interval in days (null = one-time task). E.g., `90` for quarterly, `365` for annual |
| `next_due`       | string | No       | ISO 8601 date for next occurrence: `"2026-04-15"`         |
| `priority`       | string | No       | `"low"`, `"medium"` (default), `"high"`, `"urgent"`       |
| `notes`          | string | No       | Additional notes                                          |

---

#### log_maintenance

Records a completed maintenance event. Automatically triggers `next_due` recalculation on the parent task.

| Parameter      | Type   | Required | Description                                                   |
| -------------- | ------ | -------- | ------------------------------------------------------------- |
| `task_id`      | UUID   | Yes      | ID of the maintenance task                                    |
| `user_id`      | UUID   | Yes      | User ID                                                       |
| `completed_at` | string | No       | ISO 8601 timestamp; defaults to `now()` if omitted            |
| `performed_by` | string | No       | Who performed the work: `"self"`, vendor name, etc.           |
| `cost`         | number | No       | Cost in your currency (stored as `DECIMAL(10,2)`)             |
| `notes`        | string | No       | Notes about the work performed                                |
| `next_action`  | string | No       | Recommendations from the technician for next time             |

**Database trigger behavior** (from `extensions/home-maintenance/schema.sql`):

```sql
-- After INSERT on maintenance_logs, calculates new next_due:
next_due = completed_at + (frequency_days || ' days')::INTERVAL
```

**Success response includes both the log record and the updated task:**

```json
{
  "success": true,
  "message": "Maintenance logged successfully",
  "log": { "id": "uuid", "task_id": "uuid", "completed_at": "2026-03-13T..." },
  "updated_task": { "id": "uuid", "next_due": "2026-06-11T...", "last_completed": "2026-03-13T..." }
}
```

---

#### get_upcoming_maintenance

Lists tasks due within the next N days, ordered by `next_due ASC`.

| Parameter    | Type   | Required | Default | Description                       |
| ------------ | ------ | -------- | ------- | --------------------------------- |
| `user_id`    | UUID   | Yes      | —       | User ID                           |
| `days_ahead` | number | No       | `30`    | Number of days to look ahead      |

**Notes:** Only returns tasks with a non-null `next_due` value on or before the cutoff date.

---

#### search_maintenance_history

Searches maintenance logs with optional filters. Supports cross-table filtering by joining task name/category with log date ranges.

| Parameter   | Type   | Required | Description                                               |
| ----------- | ------ | -------- | --------------------------------------------------------- |
| `user_id`   | UUID   | Yes      | User ID                                                   |
| `task_name` | string | No       | Filter by task name (ILIKE partial match on `maintenance_tasks.name`) |
| `category`  | string | No       | Filter by category (ILIKE partial match)                  |
| `date_from` | string | No       | ISO 8601 date — only logs on or after this date           |
| `date_to`   | string | No       | ISO 8601 date — only logs on or before this date          |

**Response includes joined task information:**

```json
{
  "success": true,
  "count": 3,
  "logs": [
    {
      "id": "uuid",
      "completed_at": "2026-03-01T...",
      "cost": 120.00,
      "maintenance_tasks": { "id": "uuid", "name": "HVAC Filter", "category": "hvac" }
    }
  ]
}
```

---

### Extension 3: Family Calendar

**Source:** `extensions/family-calendar/index.ts`
**Schema:** `extensions/family-calendar/schema.sql`
**Tables:** `family_members`, `activities`, `important_dates`

**Note:** This extension does not enable RLS by default. See `extensions/family-calendar/schema.sql` — no `ENABLE ROW LEVEL SECURITY` statements are present. Data isolation relies entirely on filtering by `user_id` in queries.

---

#### add_family_member

Adds a person to the household roster.

| Parameter      | Type   | Required | Description                                              |
| -------------- | ------ | -------- | -------------------------------------------------------- |
| `user_id`      | UUID   | Yes      | User ID                                                  |
| `name`         | string | Yes      | Person's name                                            |
| `relationship` | string | No       | `"self"`, `"spouse"`, `"child"`, `"parent"`, etc.        |
| `birth_date`   | string | No       | `YYYY-MM-DD` format                                      |
| `notes`        | string | No       | Additional notes                                         |

---

#### add_activity

Schedules a one-time or recurring activity.

| Parameter          | Type   | Required | Description                                                        |
| ------------------ | ------ | -------- | ------------------------------------------------------------------ |
| `user_id`          | UUID   | Yes      | User ID                                                            |
| `title`            | string | Yes      | Activity title                                                     |
| `family_member_id` | UUID   | No       | Links to a `family_members` record; null means whole-family event  |
| `activity_type`    | string | No       | `"sports"`, `"medical"`, `"school"`, `"social"`, etc.              |
| `day_of_week`      | string | No       | For recurring events: `"monday"`, `"tuesday"`, etc. Null = one-time|
| `start_time`       | string | No       | `HH:MM` format                                                     |
| `end_time`         | string | No       | `HH:MM` format                                                     |
| `start_date`       | string | No       | `YYYY-MM-DD`                                                       |
| `end_date`         | string | No       | `YYYY-MM-DD` for recurring end; null = ongoing                     |
| `location`         | string | No       | Location description                                               |
| `notes`            | string | No       | Additional notes                                                   |

---

#### get_week_schedule

Returns all activities for a given week, joining family member names.

| Parameter          | Type   | Required | Description                                          |
| ------------------ | ------ | -------- | ---------------------------------------------------- |
| `user_id`          | UUID   | Yes      | User ID                                              |
| `week_start`       | string | Yes      | Monday of the week in `YYYY-MM-DD` format            |
| `family_member_id` | UUID   | No       | Filter to a specific family member                   |

**Notes:** Includes both one-time events that fall within the week and recurring events with a matching `day_of_week`. Results include joined `family_members.name` and `family_members.relationship`.

---

#### search_activities

Searches activities by title, type, or family member.

| Parameter          | Type   | Required | Description                                  |
| ------------------ | ------ | -------- | -------------------------------------------- |
| `user_id`          | UUID   | Yes      | User ID                                      |
| `query`            | string | No       | Searches `title` (ILIKE partial match)       |
| `activity_type`    | string | No       | Exact match on `activity_type`               |
| `family_member_id` | UUID   | No       | Filter to a specific family member           |

---

#### add_important_date

Stores a date to remember — birthday, anniversary, deadline, etc.

| Parameter               | Type    | Required | Description                                      |
| ----------------------- | ------- | -------- | ------------------------------------------------ |
| `user_id`               | UUID    | Yes      | User ID                                          |
| `title`                 | string  | Yes      | Event title                                      |
| `date_value`            | string  | Yes      | `YYYY-MM-DD` format                              |
| `family_member_id`      | UUID    | No       | Links to a family member; null = family-wide     |
| `recurring_yearly`      | boolean | No       | Whether this repeats every year (default: false) |
| `reminder_days_before`  | number  | No       | Days before the date to remind (default: 7)      |
| `notes`                 | string  | No       | Additional notes                                 |

---

#### get_upcoming_dates

Returns important dates within the next N days, ordered by `date_value ASC`.

| Parameter    | Type   | Required | Default | Description                  |
| ------------ | ------ | -------- | ------- | ---------------------------- |
| `user_id`    | UUID   | Yes      | —       | User ID                      |
| `days_ahead` | number | No       | `30`    | How many days to look ahead  |

---

### Extension 4: Meal Planning

**Source:** `extensions/meal-planning/index.ts`, `extensions/meal-planning/shared-server.ts`
**Schema:** `extensions/meal-planning/schema.sql`
**Tables:** `recipes`, `meal_plans`, `shopping_lists`

**RLS model:** Full RLS is enabled on all three tables. Standard users access their own data via `auth.uid() = user_id`. Household members with `role = 'household_member'` in their JWT can SELECT from all three tables and UPDATE `shopping_lists`. See `extensions/meal-planning/schema.sql` for the full policy definitions.

**Shared access:** A second server (`shared-server.ts`) exposes read-only tools using `SUPABASE_HOUSEHOLD_KEY` — a scoped credential that grants limited access. See [Extension 4: Shared Server Tools](#extension-4-shared-server-tools) below.

---

#### add_recipe

Adds a recipe with structured ingredients and instructions.

| Parameter             | Type     | Required | Description                                                           |
| --------------------- | -------- | -------- | --------------------------------------------------------------------- |
| `user_id`             | UUID     | Yes      | User ID                                                               |
| `name`                | string   | Yes      | Recipe name                                                           |
| `ingredients`         | array    | Yes      | Array of `{ name: string, quantity: string, unit: string }` objects   |
| `instructions`        | array    | Yes      | Array of step strings                                                 |
| `cuisine`             | string   | No       | Cuisine type                                                          |
| `prep_time_minutes`   | number   | No       | Preparation time                                                      |
| `cook_time_minutes`   | number   | No       | Cooking time                                                          |
| `servings`            | number   | No       | Serving count                                                         |
| `tags`                | string[] | No       | Tags for categorization (stored as PostgreSQL `TEXT[]`)               |
| `rating`              | number   | No       | Rating 1–5 (enforced by DB CHECK)                                     |
| `notes`               | string   | No       | Additional notes                                                      |

---

#### search_recipes

Searches recipes by name, cuisine, tags, or ingredient.

| Parameter    | Type   | Required | Description                                                          |
| ------------ | ------ | -------- | -------------------------------------------------------------------- |
| `user_id`    | UUID   | Yes      | User ID                                                              |
| `query`      | string | No       | Searches `name` (ILIKE)                                              |
| `cuisine`    | string | No       | Exact match on `cuisine`                                             |
| `tag`        | string | No       | Searches `tags` array using `.contains()`                            |
| `ingredient` | string | No       | Searches `ingredients` JSONB array for `name` field using `cs` filter|

---

#### update_recipe

Updates fields on an existing recipe. Only provided fields are changed.

| Parameter             | Type     | Required | Description                                  |
| --------------------- | -------- | -------- | -------------------------------------------- |
| `recipe_id`           | UUID     | Yes      | Recipe ID                                    |
| `name`                | string   | No       | New name                                     |
| `cuisine`             | string   | No       | New cuisine                                  |
| `prep_time_minutes`   | number   | No       | New prep time                                |
| `cook_time_minutes`   | number   | No       | New cook time                                |
| `servings`            | number   | No       | New serving count                            |
| `ingredients`         | array    | No       | Full replacement of ingredients array        |
| `instructions`        | array    | No       | Full replacement of instructions array       |
| `tags`                | string[] | No       | Full replacement of tags array               |
| `rating`              | number   | No       | New rating                                   |
| `notes`               | string   | No       | New notes                                    |

---

#### create_meal_plan

Plans meals for an entire week. Each entry in `meals` is a slot (day + meal type).

| Parameter    | Type   | Required | Description                                                        |
| ------------ | ------ | -------- | ------------------------------------------------------------------ |
| `user_id`    | UUID   | Yes      | User ID                                                            |
| `week_start` | string | Yes      | Monday of the week in `YYYY-MM-DD` format                          |
| `meals`      | array  | Yes      | Array of meal objects (see below)                                  |

**Meal object schema:**

| Field          | Type   | Required | Description                                               |
| -------------- | ------ | -------- | --------------------------------------------------------- |
| `day_of_week`  | string | Yes      | `"monday"`, `"tuesday"`, `"wednesday"`, etc.              |
| `meal_type`    | string | Yes      | `"breakfast"`, `"lunch"`, `"dinner"`, `"snack"`           |
| `recipe_id`    | UUID   | No       | Link to a `recipes` record                                |
| `custom_meal`  | string | No       | Free-text meal name for meals without a recipe            |
| `servings`     | number | No       | Serving count override                                    |
| `notes`        | string | No       | Notes for this meal slot                                  |

---

#### get_meal_plan

Retrieves the full meal plan for a week, joining recipe details.

| Parameter    | Type   | Required | Description                               |
| ------------ | ------ | -------- | ----------------------------------------- |
| `user_id`    | UUID   | Yes      | User ID                                   |
| `week_start` | string | Yes      | Monday of the week in `YYYY-MM-DD` format |

---

#### generate_shopping_list

Auto-generates a shopping list by aggregating ingredients from all recipes in a meal plan. Creates a new `shopping_lists` record or updates an existing one for the same week.

| Parameter    | Type   | Required | Description                               |
| ------------ | ------ | -------- | ----------------------------------------- |
| `user_id`    | UUID   | Yes      | User ID                                   |
| `week_start` | string | Yes      | Monday of the week in `YYYY-MM-DD` format |

**Shopping list item schema** (stored in `shopping_lists.items` JSONB array):

```json
{
  "name": "chicken breast",
  "quantity": "2",
  "unit": "lbs",
  "purchased": false,
  "recipe_id": "uuid"
}
```

**Notes:** When the same ingredient appears in multiple recipes, quantities are concatenated as strings (e.g., `"2 + 1"`). More sophisticated aggregation is noted in the source as a production improvement opportunity.

---

#### Extension 4: Shared Server Tools

**Source:** `extensions/meal-planning/shared-server.ts`
**Required environment variable:** `SUPABASE_HOUSEHOLD_KEY` (scoped credential)

These tools are designed for household members who should have read access to meal data but not access to the owner's full Open Brain system.

| Tool                 | Description                                              |
| -------------------- | -------------------------------------------------------- |
| `view_meal_plan`     | Read-only view of meal plan for a given week             |
| `view_recipes`       | Browse or search recipes (read-only)                     |
| `view_shopping_list` | View the shopping list for a given week                  |
| `mark_item_purchased`| Toggle an item's `purchased` field in a shopping list    |

**`mark_item_purchased` parameters:**

| Parameter          | Type    | Required | Description                         |
| ------------------ | ------- | -------- | ----------------------------------- |
| `shopping_list_id` | UUID    | Yes      | Shopping list ID                    |
| `item_name`        | string  | Yes      | Name of the item to update          |
| `purchased`        | boolean | Yes      | New purchased status                |

---

### Extension 5: Professional CRM

**Source:** `extensions/professional-crm/index.ts`
**Schema:** `extensions/professional-crm/schema.sql`
**Tables:** `professional_contacts`, `contact_interactions`, `opportunities`

**Key behavior:** Inserting a `contact_interactions` record triggers `update_last_contacted()`, which automatically sets `professional_contacts.last_contacted` to the interaction's `occurred_at` timestamp.

**Cross-extension:** Provides `link_thought_to_contact` to pull thoughts from the core `thoughts` table and append them to a contact's notes.

---

#### add_professional_contact

Adds a contact to the professional network.

| Parameter       | Type     | Required | Description                                        |
| --------------- | -------- | -------- | -------------------------------------------------- |
| `user_id`       | UUID     | Yes      | User ID                                            |
| `name`          | string   | Yes      | Contact's full name                                |
| `company`       | string   | No       | Company name                                       |
| `title`         | string   | No       | Job title                                          |
| `email`         | string   | No       | Email address                                      |
| `phone`         | string   | No       | Phone number                                       |
| `linkedin_url`  | string   | No       | LinkedIn profile URL                               |
| `how_we_met`    | string   | No       | How you met this person                            |
| `tags`          | string[] | No       | Tags for categorization (stored as `TEXT[]`)       |
| `notes`         | string   | No       | Additional notes                                   |

---

#### search_contacts

Searches contacts by name, company, title, notes, or tags.

| Parameter | Type     | Required | Description                                                  |
| --------- | -------- | -------- | ------------------------------------------------------------ |
| `user_id` | UUID     | Yes      | User ID                                                      |
| `query`   | string   | No       | ILIKE search across `name`, `company`, `title`, `notes`      |
| `tags`    | string[] | No       | Filter by tags using `.contains()` (all tags must be present)|

---

#### log_interaction

Logs a touchpoint with a contact. Automatically updates `professional_contacts.last_contacted` via database trigger.

| Parameter           | Type    | Required | Description                                                           |
| ------------------- | ------- | -------- | --------------------------------------------------------------------- |
| `user_id`           | UUID    | Yes      | User ID                                                               |
| `contact_id`        | UUID    | Yes      | Contact ID                                                            |
| `interaction_type`  | string  | Yes      | `"meeting"`, `"email"`, `"call"`, `"coffee"`, `"event"`, `"linkedin"`, `"other"` |
| `summary`           | string  | Yes      | Summary of the interaction                                            |
| `occurred_at`       | string  | No       | ISO 8601 timestamp; defaults to `now()`                               |
| `follow_up_needed`  | boolean | No       | Whether a follow-up is required (default: false)                      |
| `follow_up_notes`   | string  | No       | Notes about the required follow-up                                    |

---

#### get_contact_history

Returns a contact's full profile plus all interactions and related opportunities.

| Parameter    | Type | Required | Description |
| ------------ | ---- | -------- | ----------- |
| `user_id`    | UUID | Yes      | User ID     |
| `contact_id` | UUID | Yes      | Contact ID  |

**Response shape:**

```json
{
  "success": true,
  "contact": { ... },
  "interactions": [ ... ],
  "opportunities": [ ... ],
  "interaction_count": 7
}
```

---

#### create_opportunity

Creates a deal, project, or potential collaboration record.

| Parameter              | Type   | Required | Description                                                                     |
| ---------------------- | ------ | -------- | ------------------------------------------------------------------------------- |
| `user_id`              | UUID   | Yes      | User ID                                                                         |
| `title`                | string | Yes      | Opportunity title                                                               |
| `contact_id`           | UUID   | No       | Link to a `professional_contacts` record                                        |
| `description`          | string | No       | Detailed description                                                            |
| `stage`                | string | No       | `"identified"` (default), `"in_conversation"`, `"proposal"`, `"negotiation"`, `"won"`, `"lost"` |
| `value`                | number | No       | Estimated value (stored as `DECIMAL(12,2)`)                                     |
| `expected_close_date`  | string | No       | `YYYY-MM-DD`                                                                    |
| `notes`                | string | No       | Additional notes                                                                |

---

#### get_follow_ups_due

Lists contacts with a `follow_up_date` on or before the cutoff date, split into overdue and upcoming.

| Parameter    | Type   | Required | Default | Description                       |
| ------------ | ------ | -------- | ------- | --------------------------------- |
| `user_id`    | UUID   | Yes      | —       | User ID                           |
| `days_ahead` | number | No       | `7`     | Number of days to look ahead      |

**Response shape:**

```json
{
  "success": true,
  "overdue_count": 2,
  "upcoming_count": 3,
  "overdue": [ ... ],
  "upcoming": [ ... ]
}
```

---

#### link_thought_to_contact

Cross-extension tool. Retrieves a thought from the core `thoughts` table and appends its content to a contact's `notes` field with a dated annotation.

| Parameter    | Type | Required | Description                                           |
| ------------ | ---- | -------- | ----------------------------------------------------- |
| `user_id`    | UUID | Yes      | User ID                                               |
| `thought_id` | UUID | Yes      | Thought ID from the core `thoughts` table             |
| `contact_id` | UUID | Yes      | Contact ID from `professional_contacts`               |

**Appended note format:**

```
[Linked Thought 2026-03-13]: Sarah mentioned she's thinking about consulting
```

---

### Extension 6: Job Hunt Pipeline

**Source:** `extensions/job-hunt/index.ts`
**Schema:** `extensions/job-hunt/schema.sql`
**Tables:** `companies`, `job_postings`, `applications`, `interviews`, `job_contacts`

**Data model hierarchy:**

```
companies
  └── job_postings
        └── applications
              └── interviews

job_contacts (linked to companies; optionally linked to professional_contacts via Extension 5)
```

**Cross-extension:** Provides `link_contact_to_professional_crm` to promote a `job_contacts` record into Extension 5's `professional_contacts` table and back-populate the `professional_crm_contact_id` foreign key.

---

#### add_company

Tracks a target company in the job search.

| Parameter          | Type   | Required | Description                                             |
| ------------------ | ------ | -------- | ------------------------------------------------------- |
| `user_id`          | UUID   | Yes      | User ID                                                 |
| `name`             | string | Yes      | Company name                                            |
| `industry`         | string | No       | Industry                                                |
| `website`          | string | No       | Company website URL                                     |
| `size`             | string | No       | `"startup"`, `"mid-market"`, `"enterprise"`             |
| `location`         | string | No       | Location description                                    |
| `remote_policy`    | string | No       | `"remote"`, `"hybrid"`, `"onsite"`                      |
| `notes`            | string | No       | Additional notes                                        |
| `glassdoor_rating` | number | No       | Rating 1.0–5.0 (enforced by DB CHECK as `DECIMAL(2,1)`)|

---

#### add_job_posting

Adds a specific role at a company.

| Parameter          | Type     | Required | Description                                                      |
| ------------------ | -------- | -------- | ---------------------------------------------------------------- |
| `user_id`          | UUID     | Yes      | User ID                                                          |
| `company_id`       | UUID     | Yes      | Company ID (must exist in `companies`)                           |
| `title`            | string   | Yes      | Job title                                                        |
| `url`              | string   | No       | Job posting URL                                                  |
| `salary_min`       | number   | No       | Minimum salary (stored as `INTEGER`)                             |
| `salary_max`       | number   | No       | Maximum salary (stored as `INTEGER`)                             |
| `salary_currency`  | string   | No       | Currency code (default: `"USD"`)                                 |
| `requirements`     | string[] | No       | Required qualifications (stored as `TEXT[]`)                     |
| `nice_to_haves`    | string[] | No       | Nice-to-have qualifications (stored as `TEXT[]`)                 |
| `notes`            | string   | No       | Notes about the role                                             |
| `source`           | string   | No       | `"linkedin"`, `"company-site"`, `"referral"`, `"recruiter"`, `"other"` |
| `posted_date`      | string   | No       | `YYYY-MM-DD`                                                     |
| `closing_date`     | string   | No       | Application deadline `YYYY-MM-DD`                                |

---

#### submit_application

Records an application to a job posting.

| Parameter              | Type   | Required | Description                                                          |
| ---------------------- | ------ | -------- | -------------------------------------------------------------------- |
| `user_id`              | UUID   | Yes      | User ID                                                              |
| `job_posting_id`       | UUID   | Yes      | Job posting ID (must exist in `job_postings`)                        |
| `status`               | string | No       | `"applied"` (default), `"draft"`, `"screening"`, `"interviewing"`, `"offer"`, `"accepted"`, `"rejected"`, `"withdrawn"` |
| `applied_date`         | string | No       | `YYYY-MM-DD`                                                         |
| `resume_version`       | string | No       | Resume version identifier                                            |
| `cover_letter_notes`   | string | No       | Notes about the cover letter                                         |
| `referral_contact`     | string | No       | Name of referral contact                                             |
| `notes`                | string | No       | Additional notes                                                     |

---

#### schedule_interview

Schedules an interview linked to an application.

| Parameter            | Type   | Required | Description                                                          |
| -------------------- | ------ | -------- | -------------------------------------------------------------------- |
| `user_id`            | UUID   | Yes      | User ID                                                              |
| `application_id`     | UUID   | Yes      | Application ID (must exist in `applications`)                        |
| `interview_type`     | string | Yes      | `"phone_screen"`, `"technical"`, `"behavioral"`, `"system_design"`, `"hiring_manager"`, `"team"`, `"final"` |
| `scheduled_at`       | string | No       | ISO 8601 datetime                                                    |
| `duration_minutes`   | number | No       | Expected duration                                                    |
| `interviewer_name`   | string | No       | Interviewer's name                                                   |
| `interviewer_title`  | string | No       | Interviewer's job title                                              |
| `notes`              | string | No       | Pre-interview prep notes                                             |

**Notes:** Status is automatically set to `"scheduled"` on insert.

---

#### log_interview_notes

Records post-interview feedback and marks the interview as completed.

| Parameter      | Type   | Required | Description                                    |
| -------------- | ------ | -------- | ---------------------------------------------- |
| `user_id`      | UUID   | Yes      | User ID (enforces ownership)                   |
| `interview_id` | UUID   | Yes      | Interview ID                                   |
| `feedback`     | string | No       | Post-interview reflection                      |
| `rating`       | number | No       | Your assessment of how it went, 1–5            |

**Notes:** Updates `status` to `"completed"` automatically.

---

#### get_pipeline_overview

Returns an aggregate dashboard: application counts by status, upcoming interviews with full company/role context.

| Parameter    | Type   | Required | Default | Description                                        |
| ------------ | ------ | -------- | ------- | -------------------------------------------------- |
| `user_id`    | UUID   | Yes      | —       | User ID                                            |
| `days_ahead` | number | No       | `7`     | Lookahead window for upcoming interviews           |

**Response shape:**

```json
{
  "success": true,
  "total_applications": 12,
  "status_breakdown": {
    "applied": 5,
    "interviewing": 3,
    "offer": 1,
    "rejected": 3
  },
  "upcoming_interviews_count": 2,
  "upcoming_interviews": [ ... ]
}
```

---

#### get_upcoming_interviews

Lists scheduled interviews within N days, with full company and job posting context via joins.

| Parameter    | Type   | Required | Default | Description                         |
| ------------ | ------ | -------- | ------- | ----------------------------------- |
| `user_id`    | UUID   | Yes      | —       | User ID                             |
| `days_ahead` | number | No       | `14`    | Number of days to look ahead        |

**Join chain:** `interviews → applications → job_postings → companies`

---

#### link_contact_to_professional_crm

Cross-extension tool. Creates a `professional_contacts` record in Extension 5 from a `job_contacts` record, then writes the new contact's UUID back to `job_contacts.professional_crm_contact_id`.

| Parameter         | Type | Required | Description                                              |
| ----------------- | ---- | -------- | -------------------------------------------------------- |
| `user_id`         | UUID | Yes      | User ID                                                  |
| `job_contact_id`  | UUID | Yes      | Job contact ID from `job_contacts`                       |

**Notes:** If the contact is already linked (`professional_crm_contact_id` is not null), returns the existing record without creating a duplicate. The `professional_crm_contact_id` column is managed at the application layer — the database does not enforce a foreign key constraint to `professional_contacts`.

---

## 3. Database Schema Reference

### Core Table: thoughts

The immovable foundation of Open Brain. Do not alter or drop existing columns.

```sql
CREATE TABLE thoughts (
  id         UUID        DEFAULT gen_random_uuid() PRIMARY KEY,
  content    TEXT        NOT NULL,
  embedding  VECTOR(1536),
  metadata   JSONB       DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- Vector similarity search index (HNSW cosine distance)
CREATE INDEX ON thoughts USING hnsw (embedding vector_cosine_ops);

-- Metadata containment filter index
CREATE INDEX ON thoughts USING gin (metadata);

-- Date range query index
CREATE INDEX ON thoughts (created_at DESC);
```

**Column reference:**

| Column       | Type           | Description                                          |
| ------------ | -------------- | ---------------------------------------------------- |
| `id`         | UUID           | Primary key, auto-generated                          |
| `content`    | TEXT           | The raw thought text                                 |
| `embedding`  | VECTOR(1536)   | Generated by `text-embedding-3-small` via OpenRouter |
| `metadata`   | JSONB          | LLM-extracted metadata (see Section 6)               |
| `created_at` | TIMESTAMPTZ    | Auto-set on insert                                   |
| `updated_at` | TIMESTAMPTZ    | Auto-updated via trigger on each UPDATE              |

**RLS policy:**

```sql
ALTER TABLE thoughts ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Service role full access"
  ON thoughts
  FOR ALL
  USING (auth.role() = 'service_role');
```

---

### Core Function: match_thoughts

The PostgreSQL function that powers semantic search.

```sql
CREATE OR REPLACE FUNCTION match_thoughts(
  query_embedding vector(1536),
  match_threshold float  DEFAULT 0.7,
  match_count     int    DEFAULT 10,
  filter          jsonb  DEFAULT '{}'::jsonb
)
RETURNS TABLE (
  id          uuid,
  content     text,
  metadata    jsonb,
  similarity  float,
  created_at  timestamptz
)
LANGUAGE plpgsql
AS $$
BEGIN
  RETURN QUERY
  SELECT
    t.id,
    t.content,
    t.metadata,
    1 - (t.embedding <=> query_embedding) AS similarity,
    t.created_at
  FROM thoughts t
  WHERE 1 - (t.embedding <=> query_embedding) > match_threshold
    AND (filter = '{}'::jsonb OR t.metadata @> filter)
  ORDER BY t.embedding <=> query_embedding
  LIMIT match_count;
END;
$$;
```

**Notes:**
- Similarity is cosine similarity: `1 - cosine_distance`
- Cosine distance is computed with the `<=>` pgvector operator
- The `filter` JSONB uses `@>` (containment): all key-value pairs in `filter` must be present in `t.metadata`
- Results are ordered by ascending distance (most similar first)

---

### Extension 1: Household Knowledge Base

**Source:** `extensions/household-knowledge/schema.sql`

#### household_items

| Column       | Type        | Constraints                | Description                              |
| ------------ | ----------- | -------------------------- | ---------------------------------------- |
| `id`         | UUID        | PK, default gen_random_uuid| Primary key                              |
| `user_id`    | UUID        | NOT NULL, FK auth.users    | Owner                                    |
| `name`       | TEXT        | NOT NULL                   | Item name or description                 |
| `category`   | TEXT        |                            | `"paint"`, `"appliance"`, etc.           |
| `location`   | TEXT        |                            | Room or area in the home                 |
| `details`    | JSONB       | DEFAULT `{}`               | Flexible metadata                        |
| `notes`      | TEXT        |                            | Additional notes                         |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now()    | Created timestamp                        |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now()    | Updated via trigger                      |

**Indexes:**
- `idx_household_items_user_category` on `(user_id, category)`

#### household_vendors

| Column         | Type        | Constraints                            | Description                |
| -------------- | ----------- | -------------------------------------- | -------------------------- |
| `id`           | UUID        | PK                                     | Primary key                |
| `user_id`      | UUID        | NOT NULL, FK auth.users                | Owner                      |
| `name`         | TEXT        | NOT NULL                               | Vendor name                |
| `service_type` | TEXT        |                                        | Service category           |
| `phone`        | TEXT        |                                        | Phone number               |
| `email`        | TEXT        |                                        | Email address              |
| `website`      | TEXT        |                                        | Website URL                |
| `notes`        | TEXT        |                                        | Additional notes           |
| `rating`       | INTEGER     | CHECK (rating >= 1 AND rating <= 5)    | Rating 1–5                 |
| `last_used`    | DATE        |                                        | Date last used             |
| `created_at`   | TIMESTAMPTZ | NOT NULL, DEFAULT now()                | Created timestamp          |

**Indexes:**
- `idx_household_vendors_user_service` on `(user_id, service_type)`

**RLS:** Both tables have `ENABLE ROW LEVEL SECURITY` with policies restricting all operations to `auth.uid() = user_id`.

---

### Extension 2: Home Maintenance Tracker

**Source:** `extensions/home-maintenance/schema.sql`

#### maintenance_tasks

| Column           | Type        | Constraints                                               | Description                           |
| ---------------- | ----------- | --------------------------------------------------------- | ------------------------------------- |
| `id`             | UUID        | PK                                                        | Primary key                           |
| `user_id`        | UUID        | NOT NULL, FK auth.users                                   | Owner                                 |
| `name`           | TEXT        | NOT NULL                                                  | Task name                             |
| `category`       | TEXT        |                                                           | `"hvac"`, `"plumbing"`, etc.          |
| `frequency_days` | INTEGER     |                                                           | Null = one-time                       |
| `last_completed` | TIMESTAMPTZ |                                                           | Auto-updated by trigger               |
| `next_due`       | TIMESTAMPTZ |                                                           | Auto-calculated by trigger            |
| `priority`       | TEXT        | CHECK (IN ('low','medium','high','urgent')), DEFAULT 'medium'| Priority level                     |
| `notes`          | TEXT        |                                                           | Additional notes                      |
| `created_at`     | TIMESTAMPTZ | NOT NULL, DEFAULT now()                                   | Created timestamp                     |
| `updated_at`     | TIMESTAMPTZ | NOT NULL, DEFAULT now()                                   | Updated via trigger                   |

#### maintenance_logs

| Column         | Type           | Constraints                 | Description                               |
| -------------- | -------------- | --------------------------- | ----------------------------------------- |
| `id`           | UUID           | PK                          | Primary key                               |
| `task_id`      | UUID           | NOT NULL, FK maintenance_tasks ON DELETE CASCADE | Parent task       |
| `user_id`      | UUID           | NOT NULL, FK auth.users     | Owner                                     |
| `completed_at` | TIMESTAMPTZ    | NOT NULL, DEFAULT now()     | When work was done                        |
| `performed_by` | TEXT           |                             | `"self"` or vendor name                   |
| `cost`         | DECIMAL(10,2)  |                             | Cost of the work                          |
| `notes`        | TEXT           |                             | Notes about the work                      |
| `next_action`  | TEXT           |                             | Technician recommendations                |

**Indexes:**
- `idx_maintenance_tasks_user_next_due` on `(user_id, next_due)`
- `idx_maintenance_logs_task_completed` on `(task_id, completed_at DESC)`
- `idx_maintenance_logs_user_completed` on `(user_id, completed_at DESC)`

**Database triggers:**
- `update_maintenance_tasks_updated_at` — BEFORE UPDATE, sets `updated_at = now()`
- `update_task_after_log` — AFTER INSERT on `maintenance_logs`, updates parent task's `last_completed` and recalculates `next_due`

---

### Extension 3: Family Calendar

**Source:** `extensions/family-calendar/schema.sql`

#### family_members

| Column         | Type        | Constraints             | Description                        |
| -------------- | ----------- | ----------------------- | ---------------------------------- |
| `id`           | UUID        | PK                      | Primary key                        |
| `user_id`      | UUID        | NOT NULL, FK auth.users | Owner                              |
| `name`         | TEXT        | NOT NULL                | Person's name                      |
| `relationship` | TEXT        |                         | `"self"`, `"spouse"`, `"child"`, etc.|
| `birth_date`   | DATE        |                         | Birth date                         |
| `notes`        | TEXT        |                         | Additional notes                   |
| `created_at`   | TIMESTAMPTZ | DEFAULT now()           | Created timestamp                  |

#### activities

| Column             | Type        | Constraints             | Description                                |
| ------------------ | ----------- | ----------------------- | ------------------------------------------ |
| `id`               | UUID        | PK                      | Primary key                                |
| `user_id`          | UUID        | NOT NULL, FK auth.users | Owner                                      |
| `family_member_id` | UUID        | FK family_members       | Null = whole-family event                  |
| `title`            | TEXT        | NOT NULL                | Activity title                             |
| `activity_type`    | TEXT        |                         | `"sports"`, `"medical"`, `"school"`, etc.  |
| `day_of_week`      | TEXT        |                         | `"monday"` etc. for recurring; null = one-time|
| `start_time`       | TIME        |                         | Start time                                 |
| `end_time`         | TIME        |                         | End time                                   |
| `start_date`       | DATE        |                         | Start date                                 |
| `end_date`         | DATE        |                         | End date for recurring; null = ongoing     |
| `location`         | TEXT        |                         | Location description                       |
| `notes`            | TEXT        |                         | Additional notes                           |
| `created_at`       | TIMESTAMPTZ | DEFAULT now()           | Created timestamp                          |

#### important_dates

| Column                  | Type        | Constraints              | Description                              |
| ----------------------- | ----------- | ------------------------ | ---------------------------------------- |
| `id`                    | UUID        | PK                       | Primary key                              |
| `user_id`               | UUID        | NOT NULL, FK auth.users  | Owner                                    |
| `family_member_id`      | UUID        | FK family_members        | Null = family-wide date                  |
| `title`                 | TEXT        | NOT NULL                 | Event title                              |
| `date_value`            | DATE        | NOT NULL                 | The date                                 |
| `recurring_yearly`      | BOOLEAN     | DEFAULT false            | Whether it repeats each year             |
| `reminder_days_before`  | INTEGER     | DEFAULT 7                | Days in advance to remind                |
| `notes`                 | TEXT        |                          | Additional notes                         |
| `created_at`            | TIMESTAMPTZ | DEFAULT now()            | Created timestamp                        |

**Indexes:**
- `idx_activities_user_dow` on `(user_id, day_of_week)`
- `idx_activities_family_member` on `(family_member_id)`
- `idx_activities_user_dates` on `(user_id, start_date, end_date)`
- `idx_important_dates_user_date` on `(user_id, date_value)`
- `idx_family_members_user` on `(user_id)`

**RLS:** Not enabled in this extension. Data isolation via `user_id` in application queries.

---

### Extension 4: Meal Planning

**Source:** `extensions/meal-planning/schema.sql`

#### recipes

| Column               | Type        | Constraints                         | Description                                    |
| -------------------- | ----------- | ----------------------------------- | ---------------------------------------------- |
| `id`                 | UUID        | PK                                  | Primary key                                    |
| `user_id`            | UUID        | NOT NULL, FK auth.users             | Owner                                          |
| `name`               | TEXT        | NOT NULL                            | Recipe name                                    |
| `cuisine`            | TEXT        |                                     | Cuisine type                                   |
| `prep_time_minutes`  | INTEGER     |                                     | Preparation time                               |
| `cook_time_minutes`  | INTEGER     |                                     | Cooking time                                   |
| `servings`           | INTEGER     |                                     | Number of servings                             |
| `ingredients`        | JSONB       | NOT NULL, DEFAULT `[]`              | Array of `{ name, quantity, unit }` objects    |
| `instructions`       | JSONB       | NOT NULL, DEFAULT `[]`              | Array of instruction strings                   |
| `tags`               | TEXT[]      | DEFAULT `{}`                        | Tag array                                      |
| `rating`             | INTEGER     | CHECK (rating >= 1 AND rating <= 5) | Rating 1–5                                     |
| `notes`              | TEXT        |                                     | Additional notes                               |
| `created_at`         | TIMESTAMPTZ | DEFAULT now()                       | Created timestamp                              |
| `updated_at`         | TIMESTAMPTZ | DEFAULT now()                       | Updated timestamp                              |

#### meal_plans

| Column         | Type        | Constraints                                                   | Description                     |
| -------------- | ----------- | ------------------------------------------------------------- | ------------------------------- |
| `id`           | UUID        | PK                                                            | Primary key                     |
| `user_id`      | UUID        | NOT NULL, FK auth.users                                       | Owner                           |
| `week_start`   | DATE        | NOT NULL                                                      | Monday of the week              |
| `day_of_week`  | TEXT        | NOT NULL                                                      | `"monday"`, `"tuesday"`, etc.   |
| `meal_type`    | TEXT        | NOT NULL, CHECK (IN ('breakfast','lunch','dinner','snack'))    | Meal slot                       |
| `recipe_id`    | UUID        | FK recipes                                                    | Optional linked recipe          |
| `custom_meal`  | TEXT        |                                                               | Free-text for meals without a recipe |
| `servings`     | INTEGER     |                                                               | Serving count override          |
| `notes`        | TEXT        |                                                               | Notes for this slot             |
| `created_at`   | TIMESTAMPTZ | DEFAULT now()                                                 | Created timestamp               |

#### shopping_lists

| Column       | Type        | Constraints             | Description                                             |
| ------------ | ----------- | ----------------------- | ------------------------------------------------------- |
| `id`         | UUID        | PK                      | Primary key                                             |
| `user_id`    | UUID        | NOT NULL, FK auth.users | Owner                                                   |
| `week_start` | DATE        | NOT NULL                | Monday of the week                                      |
| `items`      | JSONB       | NOT NULL, DEFAULT `[]`  | Array of `{ name, quantity, unit, purchased, recipe_id }`|
| `notes`      | TEXT        |                         | Additional notes                                        |
| `created_at` | TIMESTAMPTZ | DEFAULT now()           | Created timestamp                                       |
| `updated_at` | TIMESTAMPTZ | DEFAULT now()           | Updated timestamp                                       |

**Indexes:**
- `idx_recipes_user_cuisine` on `(user_id, cuisine)`
- `idx_recipes_user_tags` GIN on `tags`
- `idx_meal_plans_user_week` on `(user_id, week_start)`
- `idx_shopping_lists_user_week` on `(user_id, week_start)`

**RLS:** Full RLS enabled on all three tables. Household members with `auth.jwt() ->> 'role' = 'household_member'` can SELECT from all tables and UPDATE `shopping_lists`.

---

### Extension 5: Professional CRM

**Source:** `extensions/professional-crm/schema.sql`

#### professional_contacts

| Column            | Type        | Constraints                | Description                        |
| ----------------- | ----------- | -------------------------- | ---------------------------------- |
| `id`              | UUID        | PK                         | Primary key                        |
| `user_id`         | UUID        | NOT NULL, FK auth.users    | Owner                              |
| `name`            | TEXT        | NOT NULL                   | Full name                          |
| `company`         | TEXT        |                            | Company name                       |
| `title`           | TEXT        |                            | Job title                          |
| `email`           | TEXT        |                            | Email address                      |
| `phone`           | TEXT        |                            | Phone number                       |
| `linkedin_url`    | TEXT        |                            | LinkedIn URL                       |
| `how_we_met`      | TEXT        |                            | How you met                        |
| `tags`            | TEXT[]      | DEFAULT `{}`               | Tags                               |
| `notes`           | TEXT        |                            | Notes (may include linked thoughts)|
| `last_contacted`  | TIMESTAMPTZ |                            | Auto-updated by trigger            |
| `follow_up_date`  | DATE        |                            | Reminder date                      |
| `created_at`      | TIMESTAMPTZ | NOT NULL, DEFAULT now()    | Created timestamp                  |
| `updated_at`      | TIMESTAMPTZ | NOT NULL, DEFAULT now()    | Updated via trigger                |

#### contact_interactions

| Column              | Type        | Constraints                                                    | Description                |
| ------------------- | ----------- | -------------------------------------------------------------- | -------------------------- |
| `id`                | UUID        | PK                                                             | Primary key                |
| `contact_id`        | UUID        | NOT NULL, FK professional_contacts ON DELETE CASCADE           | Parent contact             |
| `user_id`           | UUID        | NOT NULL, FK auth.users                                        | Owner                      |
| `interaction_type`  | TEXT        | NOT NULL, CHECK (IN ('meeting','email','call','coffee','event','linkedin','other')) | Type |
| `occurred_at`       | TIMESTAMPTZ | NOT NULL, DEFAULT now()                                        | When it happened           |
| `summary`           | TEXT        | NOT NULL                                                       | Interaction summary        |
| `follow_up_needed`  | BOOLEAN     | DEFAULT false                                                  | Follow-up flag             |
| `follow_up_notes`   | TEXT        |                                                                | Follow-up details          |
| `created_at`        | TIMESTAMPTZ | NOT NULL, DEFAULT now()                                        | Created timestamp          |

#### opportunities

| Column                | Type           | Constraints                                                        | Description           |
| --------------------- | -------------- | ------------------------------------------------------------------ | --------------------- |
| `id`                  | UUID           | PK                                                                 | Primary key           |
| `user_id`             | UUID           | NOT NULL, FK auth.users                                            | Owner                 |
| `contact_id`          | UUID           | FK professional_contacts ON DELETE SET NULL                        | Associated contact    |
| `title`               | TEXT           | NOT NULL                                                           | Opportunity title     |
| `description`         | TEXT           |                                                                    | Details               |
| `stage`               | TEXT           | DEFAULT 'identified', CHECK (IN ('identified','in_conversation','proposal','negotiation','won','lost')) | Pipeline stage |
| `value`               | DECIMAL(12,2)  |                                                                    | Estimated value       |
| `expected_close_date` | DATE           |                                                                    | Expected close date   |
| `notes`               | TEXT           |                                                                    | Additional notes      |
| `created_at`          | TIMESTAMPTZ    | NOT NULL, DEFAULT now()                                            | Created timestamp     |
| `updated_at`          | TIMESTAMPTZ    | NOT NULL, DEFAULT now()                                            | Updated via trigger   |

**Indexes:**
- `idx_professional_contacts_user_last_contacted` on `(user_id, last_contacted)`
- `idx_professional_contacts_follow_up` partial index on `(user_id, follow_up_date) WHERE follow_up_date IS NOT NULL`
- `idx_contact_interactions_contact_occurred` on `(contact_id, occurred_at DESC)`
- `idx_opportunities_user_stage` on `(user_id, stage)`

**Database triggers:**
- `update_professional_contacts_updated_at` — BEFORE UPDATE, sets `updated_at = now()`
- `update_opportunities_updated_at` — BEFORE UPDATE, sets `updated_at = now()`
- `update_contact_last_contacted` — AFTER INSERT on `contact_interactions`, sets `professional_contacts.last_contacted = NEW.occurred_at`

---

### Extension 6: Job Hunt Pipeline

**Source:** `extensions/job-hunt/schema.sql`

#### companies

| Column             | Type           | Constraints                                                      | Description          |
| ------------------ | -------------- | ---------------------------------------------------------------- | -------------------- |
| `id`               | UUID           | PK                                                               | Primary key          |
| `user_id`          | UUID           | NOT NULL, FK auth.users                                          | Owner                |
| `name`             | TEXT           | NOT NULL                                                         | Company name         |
| `industry`         | TEXT           |                                                                  | Industry             |
| `website`          | TEXT           |                                                                  | Website URL          |
| `size`             | TEXT           | CHECK (IN ('startup','mid-market','enterprise') OR NULL)         | Company size         |
| `location`         | TEXT           |                                                                  | Location             |
| `remote_policy`    | TEXT           | CHECK (IN ('remote','hybrid','onsite') OR NULL)                  | Remote policy        |
| `notes`            | TEXT           |                                                                  | Notes                |
| `glassdoor_rating` | DECIMAL(2,1)   | CHECK (1.0–5.0 OR NULL)                                          | Glassdoor rating     |
| `created_at`       | TIMESTAMPTZ    | NOT NULL, DEFAULT now()                                          | Created timestamp    |
| `updated_at`       | TIMESTAMPTZ    | NOT NULL, DEFAULT now()                                          | Updated via trigger  |

#### job_postings

| Column             | Type        | Constraints                                                       | Description                |
| ------------------ | ----------- | ----------------------------------------------------------------- | -------------------------- |
| `id`               | UUID        | PK                                                                | Primary key                |
| `company_id`       | UUID        | NOT NULL, FK companies ON DELETE CASCADE                          | Parent company             |
| `user_id`          | UUID        | NOT NULL, FK auth.users                                           | Owner                      |
| `title`            | TEXT        | NOT NULL                                                          | Job title                  |
| `url`              | TEXT        |                                                                   | Posting URL                |
| `salary_min`       | INTEGER     |                                                                   | Minimum salary             |
| `salary_max`       | INTEGER     |                                                                   | Maximum salary             |
| `salary_currency`  | TEXT        | DEFAULT 'USD'                                                     | Currency                   |
| `requirements`     | TEXT[]      |                                                                   | Required qualifications    |
| `nice_to_haves`    | TEXT[]      |                                                                   | Nice-to-have qualifications|
| `notes`            | TEXT        |                                                                   | Role notes                 |
| `source`           | TEXT        | CHECK (IN ('linkedin','company-site','referral','recruiter','other') OR NULL) | Discovery source |
| `posted_date`      | DATE        |                                                                   | Date posted                |
| `closing_date`     | DATE        |                                                                   | Application deadline       |
| `created_at`       | TIMESTAMPTZ | NOT NULL, DEFAULT now()                                           | Created timestamp          |

#### applications

| Column               | Type        | Constraints                                                                    | Description              |
| -------------------- | ----------- | ------------------------------------------------------------------------------ | ------------------------ |
| `id`                 | UUID        | PK                                                                             | Primary key              |
| `job_posting_id`     | UUID        | NOT NULL, FK job_postings ON DELETE CASCADE                                    | Parent posting           |
| `user_id`            | UUID        | NOT NULL, FK auth.users                                                        | Owner                    |
| `status`             | TEXT        | DEFAULT 'applied', CHECK (IN ('draft','applied','screening','interviewing','offer','accepted','rejected','withdrawn')) | Status |
| `applied_date`       | DATE        |                                                                                | Date applied             |
| `response_date`      | DATE        |                                                                                | Date of response         |
| `resume_version`     | TEXT        |                                                                                | Resume version used      |
| `cover_letter_notes` | TEXT        |                                                                                | Cover letter notes       |
| `referral_contact`   | TEXT        |                                                                                | Referral name            |
| `notes`              | TEXT        |                                                                                | Additional notes         |
| `created_at`         | TIMESTAMPTZ | NOT NULL, DEFAULT now()                                                        | Created timestamp        |
| `updated_at`         | TIMESTAMPTZ | NOT NULL, DEFAULT now()                                                        | Updated via trigger      |

#### interviews

| Column               | Type        | Constraints                                                                  | Description                  |
| -------------------- | ----------- | ---------------------------------------------------------------------------- | ---------------------------- |
| `id`                 | UUID        | PK                                                                           | Primary key                  |
| `application_id`     | UUID        | NOT NULL, FK applications ON DELETE CASCADE                                  | Parent application           |
| `user_id`            | UUID        | NOT NULL, FK auth.users                                                      | Owner                        |
| `interview_type`     | TEXT        | CHECK (IN ('phone_screen','technical','behavioral','system_design','hiring_manager','team','final')) | Type |
| `scheduled_at`       | TIMESTAMPTZ |                                                                              | Scheduled datetime           |
| `duration_minutes`   | INTEGER     |                                                                              | Expected duration            |
| `interviewer_name`   | TEXT        |                                                                              | Interviewer name             |
| `interviewer_title`  | TEXT        |                                                                              | Interviewer title            |
| `status`             | TEXT        | DEFAULT 'scheduled', CHECK (IN ('scheduled','completed','cancelled','no_show'))| Status                     |
| `notes`              | TEXT        |                                                                              | Pre-interview prep notes     |
| `feedback`           | TEXT        |                                                                              | Post-interview reflection    |
| `rating`             | INTEGER     | CHECK (1–5 OR NULL)                                                          | Self-assessed rating         |
| `created_at`         | TIMESTAMPTZ | NOT NULL, DEFAULT now()                                                      | Created timestamp            |

#### job_contacts

| Column                        | Type        | Constraints                                                     | Description                           |
| ----------------------------- | ----------- | --------------------------------------------------------------- | ------------------------------------- |
| `id`                          | UUID        | PK                                                              | Primary key                           |
| `user_id`                     | UUID        | NOT NULL, FK auth.users                                         | Owner                                 |
| `company_id`                  | UUID        | FK companies ON DELETE SET NULL                                 | Associated company                    |
| `name`                        | TEXT        | NOT NULL                                                        | Contact name                          |
| `title`                       | TEXT        |                                                                 | Job title                             |
| `email`                       | TEXT        |                                                                 | Email                                 |
| `phone`                       | TEXT        |                                                                 | Phone                                 |
| `linkedin_url`                | TEXT        |                                                                 | LinkedIn URL                          |
| `role_in_process`             | TEXT        | CHECK (IN ('recruiter','hiring_manager','referral','interviewer','other') OR NULL) | Role in hiring     |
| `professional_crm_contact_id` | UUID        | No FK enforced                                                  | Soft link to Extension 5 contact      |
| `notes`                       | TEXT        |                                                                 | Notes                                 |
| `last_contacted`              | TIMESTAMPTZ |                                                                 | Last contact timestamp                |
| `created_at`                  | TIMESTAMPTZ | NOT NULL, DEFAULT now()                                         | Created timestamp                     |

**Indexes:**
- `idx_companies_user_id` on `(user_id)`
- `idx_job_postings_company_id` on `(company_id)`
- `idx_applications_user_status` on `(user_id, status)`
- `idx_applications_job_posting` on `(job_posting_id)`
- `idx_interviews_application_scheduled` on `(application_id, scheduled_at)`
- `idx_interviews_user_scheduled` partial on `(user_id, scheduled_at) WHERE scheduled_at IS NOT NULL`
- `idx_job_contacts_user_company` on `(user_id, company_id)`

**Database triggers:**
- `update_companies_updated_at` — BEFORE UPDATE on `companies`
- `update_applications_updated_at` — BEFORE UPDATE on `applications`

---

## 4. Authentication and Access

### MCP Server Authentication

The core MCP server (`open-brain-mcp` Edge Function) checks an access key on every request. The key must match the value stored in the `MCP_ACCESS_KEY` Supabase secret.

Two delivery methods are accepted:

**Query parameter** — for Claude Desktop, ChatGPT, and any client that supports a remote URL with a query string:

```
https://YOUR_PROJECT_REF.supabase.co/functions/v1/open-brain-mcp?key=your-access-key
```

**Request header** — for Claude Code and `mcp-remote`:

```
x-brain-key: your-access-key
```

Generate the access key:

```bash
# macOS / Linux
openssl rand -hex 32

# Windows PowerShell
-join ((1..32) | ForEach-Object { '{0:x2}' -f (Get-Random -Maximum 256) })
```

Store it in Supabase:

```bash
supabase secrets set MCP_ACCESS_KEY=your-generated-key-here
```

**Troubleshooting 401 errors:** The `?key=` value must exactly match the stored secret. For the header approach, the header name must be `x-brain-key` (lowercase, with the dash).

---

### Claude Desktop

1. Settings → Connectors → Add custom connector
2. Name: `Open Brain`
3. Remote MCP server URL: `https://YOUR_PROJECT_REF.supabase.co/functions/v1/open-brain-mcp?key=your-access-key`

---

### Claude Code

```bash
claude mcp add --transport http open-brain \
  https://YOUR_PROJECT_REF.supabase.co/functions/v1/open-brain-mcp \
  --header "x-brain-key: your-access-key"
```

---

### ChatGPT

Requires a paid plan and Developer Mode enabled (Settings → Apps & Connectors → Advanced settings → Developer mode ON).

Add connector:
- MCP endpoint URL: `https://YOUR_PROJECT_REF.supabase.co/functions/v1/open-brain-mcp?key=your-access-key`
- Authentication: No Authentication (key is embedded in the URL)

---

### Other Clients via mcp-remote

For clients that only support stdio-based MCP servers (configured via JSON):

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

**Note:** No space after the colon in `x-brain-key:${BRAIN_KEY}`. Some clients mangle spaces inside args.

---

### Supabase REST API (Direct Access)

For direct database access outside of MCP:

```
Base URL: https://YOUR_PROJECT_REF.supabase.co/rest/v1/
```

Required headers:

| Header          | Value                                  |
| --------------- | -------------------------------------- |
| `apikey`        | Your anon key or service role key      |
| `Authorization` | `Bearer <anon-key-or-service-role-key>`|

**Example — query recent thoughts:**

```bash
curl "https://YOUR_PROJECT_REF.supabase.co/rest/v1/thoughts?order=created_at.desc&limit=10" \
  -H "apikey: YOUR_SERVICE_ROLE_KEY" \
  -H "Authorization: Bearer YOUR_SERVICE_ROLE_KEY"
```

**Key distinction:**
- The **anon key** (now called "Publishable key" in the Supabase UI) respects RLS policies
- The **service role key** (now called "Secret key") bypasses RLS and has full access — treat it as a password

---

### Extension Server Authentication

Extension MCP servers (Extensions 1–6) run as local Node.js processes and authenticate to Supabase using `SUPABASE_SERVICE_ROLE_KEY`. They do not expose their own authentication layer — the host machine's environment is the security boundary.

```bash
# Start an extension server with credentials
SUPABASE_URL=https://your-project.supabase.co \
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key \
node extensions/household-knowledge/index.ts
```

---

## 5. Environment Variables and Secrets

### Core MCP Server (Edge Function)

| Variable                    | Where Set                         | Auto-Available | Description                                   |
| --------------------------- | --------------------------------- | -------------- | --------------------------------------------- |
| `SUPABASE_URL`              | Auto-injected by Supabase runtime | Yes            | Your project URL                              |
| `SUPABASE_SERVICE_ROLE_KEY` | Auto-injected by Supabase runtime | Yes            | Full database access key                      |
| `OPENROUTER_API_KEY`        | `supabase secrets set`            | No             | For embeddings and metadata extraction        |
| `MCP_ACCESS_KEY`            | `supabase secrets set`            | No             | Incoming request authentication               |

Set secrets via CLI:

```bash
supabase secrets set OPENROUTER_API_KEY=sk-or-...
supabase secrets set MCP_ACCESS_KEY=your-generated-key
```

Verify secrets are set:

```bash
supabase secrets list
```

---

### Extension Servers (Local Node.js)

| Variable                    | Required   | Description                                          |
| --------------------------- | ---------- | ---------------------------------------------------- |
| `SUPABASE_URL`              | Yes        | Your Supabase project URL                            |
| `SUPABASE_SERVICE_ROLE_KEY` | Yes        | Service role key for database access                 |

For Extension 4 shared server only:

| Variable                   | Required | Description                                           |
| -------------------------- | -------- | ----------------------------------------------------- |
| `SUPABASE_HOUSEHOLD_KEY`   | Yes      | Scoped credential with limited table access           |

---

### Credential Tracker Template

Copy this into a text file and fill in as you set up:

```
OPEN BRAIN -- CREDENTIAL TRACKER
--------------------------------------

SUPABASE
  Project URL:        ____________  (Settings → API)
  Project ref:        ____________  (from dashboard URL)
  Secret key:         ____________  (Settings → API → Secret key)

OPENROUTER
  API key:            ____________  (openrouter.ai/keys)

GENERATED DURING SETUP
  MCP Access Key:     ____________  (generated via openssl or PowerShell)
  MCP Server URL:     ____________  (https://YOUR_PROJECT_REF.supabase.co/functions/v1/open-brain-mcp)
  MCP Connection URL: ____________  (server URL + ?key=your-access-key)

--------------------------------------
```

---

## 6. Metadata JSONB Structure

The `metadata` column in the `thoughts` table contains a JSON object extracted by `gpt-4o-mini` at capture time. The extraction is best-effort — the model makes its best classification given the content.

**Full schema:**

```typescript
interface ThoughtMetadata {
  type:         "idea" | "task" | "person_note" | "reference" | "observation";
  topics:       string[];   // extracted topic strings
  people:       string[];   // names mentioned in the thought
  action_items: string[];   // tasks or follow-ups extracted from content
  source:       string;     // where the thought came from
}
```

**Field reference:**

| Field          | Type     | Description                                                                  |
| -------------- | -------- | ---------------------------------------------------------------------------- |
| `type`         | string   | Classification of the thought                                                |
| `topics`       | string[] | Topics the LLM identifies as central to the thought                         |
| `people`       | string[] | Names of people mentioned by name in the thought                             |
| `action_items` | string[] | Tasks, next steps, or follow-ups the LLM identifies in the content          |
| `source`       | string   | Capture source (e.g., `"claude-desktop"`, `"chatgpt"`, `"slack"`)           |

**Type values:**

| Value          | When used                                                            |
| -------------- | -------------------------------------------------------------------- |
| `idea`         | A new concept, insight, or creative thought                          |
| `task`         | Something that needs to be done                                      |
| `person_note`  | Information about a specific person                                  |
| `reference`    | A fact, link, or piece of information to remember                   |
| `observation`  | A note about something observed or experienced                       |

**Example metadata objects:**

```json
{
  "type": "person_note",
  "topics": ["career", "consulting"],
  "people": ["Sarah"],
  "action_items": ["Follow up with Sarah next week"],
  "source": "claude-desktop"
}
```

```json
{
  "type": "task",
  "topics": ["product", "launch"],
  "people": [],
  "action_items": ["Move launch to March 15", "Notify QA team"],
  "source": "chatgpt"
}
```

**Using metadata in filters:**

```json
// search_thoughts filter: only tasks
{ "type": "task" }

// Only thoughts mentioning a topic
// Note: topics is an array, so use array containment
{ "topics": ["consulting"] }
```

**Supabase REST filter syntax:**

```
GET /rest/v1/thoughts?metadata=cs.{"type":"task"}
```

---

## 7. Cross-Extension Integration

Two cross-extension tools bridge data between extensions and the core `thoughts` table.

### Professional CRM ↔ Core Thoughts

**Tool:** `link_thought_to_contact` in Extension 5

Retrieves a `thoughts` record by `id` and appends its `content` to `professional_contacts.notes` with a dated prefix. Both records must belong to the same `user_id`.

```
[Linked Thought 2026-03-13]: Sarah mentioned she wants to start consulting
```

**Use case:** You captured something about a contact in the core brain before you had the CRM set up. This tool pulls it over without duplication.

---

### Job Hunt ↔ Professional CRM

**Tool:** `link_contact_to_professional_crm` in Extension 6

Copies a `job_contacts` record into `professional_contacts` (Extension 5), filling in company name from the `companies` table, tagging the new record with `["job-hunt", role_in_process]`, and writing the new contact's UUID back to `job_contacts.professional_crm_contact_id`.

The `professional_crm_contact_id` column is intentionally not enforced by a database foreign key — the link is managed at the application layer to avoid cross-extension hard coupling.

**Use case:** You met a recruiter during your job search and want them in your long-term professional network after the search is over.

---

### Data Flow Diagram

```
thoughts (core)
    │
    │  link_thought_to_contact
    ▼
professional_contacts (Ext 5)
    │
    │  link_contact_to_professional_crm
    ▲
job_contacts (Ext 6)
    │
    └── company_id ──► companies (Ext 6)
                           ▲
                      job_postings
                           ▲
                       applications
                           ▲
                        interviews
```

---

## 8. Error Handling

### MCP Tool Error Format

All extension servers return errors in a consistent shape:

```json
{
  "success": false,
  "error": "Failed to add household item: duplicate key value violates unique constraint"
}
```

The MCP response wrapper sets `isError: true`:

```json
{
  "content": [{ "type": "text", "text": "{ \"success\": false, \"error\": \"...\" }" }],
  "isError": true
}
```

### Common Error Conditions

| Error                                    | Cause                                                         | Resolution                                           |
| ---------------------------------------- | ------------------------------------------------------------- | ---------------------------------------------------- |
| `401 Unauthorized`                       | Access key mismatch on core MCP server                        | Verify `?key=` or `x-brain-key` header matches secret|
| `Item not found or access denied`        | `user_id` does not match the stored record                    | Confirm the correct `user_id` is being passed        |
| `Missing required environment variables` | Extension server started without `SUPABASE_URL` or key set    | Set environment variables before starting server     |
| `Failed to add X: <Supabase error>`      | Database constraint violation or invalid data type            | Check column constraints in the schema reference     |
| Search returns no results                | Threshold too high or no matching thoughts captured yet       | Lower threshold (e.g., `0.3`) or capture test data   |
| Slow first response from core MCP        | Edge Function cold start                                      | Expected behavior; subsequent calls are faster       |

### Supabase Edge Function Logs

To diagnose Edge Function errors:

1. Supabase Dashboard → Edge Functions → `open-brain-mcp` → Logs
2. Paste the error message into the Supabase AI assistant (chat icon, bottom-right of dashboard) for diagnosis

---

*This reference covers Open Brain as of March 2026. For setup instructions, see `docs/01-getting-started.md`. For contribution guidelines, see `CONTRIBUTING.md`.*
