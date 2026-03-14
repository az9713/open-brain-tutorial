# Extension 4: Meal Planning MCP Server

This extension introduces something none of the previous three do: **two servers that share the same database**. One server is yours — full read/write access to build recipe collections, schedule weekly meals, and generate shopping lists. The second server is scoped to household members — read-only access to recipes and plans, plus the narrow ability to tick items off the shopping list. Both talk to the same Supabase tables. Neither server knows the other exists.

This is also the first extension in the learning path that puts the two database primitives to work together: the `rls` primitive (row-level security) and the `shared-mcp` primitive (scoped server pattern). Understanding how they combine here makes both primitives click.

---

## Table of Contents

1. [Role in Architecture](#1-role-in-architecture)
2. [Database Schema](#2-database-schema)
3. [Main Server — 6 Tools](#3-main-server--6-tools)
4. [Shared Server — 4 Tools](#4-shared-server--4-tools)
5. [Key Patterns](#5-key-patterns)
6. [How It Fits (Primitives and Extensions)](#6-how-it-fits-primitives-and-extensions)
7. [Source Files](#7-source-files)

---

## 1. Role in Architecture

```
                        YOUR MACHINE
┌───────────────────────────────────────────────────────────┐
│                                                           │
│   Claude Desktop                                          │
│        │                                                  │
│        ▼  (stdio)                                         │
│   meal-planning server  ──────────────────────┐           │
│   (index.ts)                                  │           │
│   SERVICE_ROLE_KEY                            │           │
│                                               │           │
└───────────────────────────────────────────────┼───────────┘
                                                │
                                         Supabase DB
                                         (3 tables)
                                                │
┌───────────────────────────────────────────────┼───────────┐
│                                               │           │
│   Claude Desktop  (household member)          │           │
│        │                                      │           │
│        ▼  (stdio)                             │           │
│   meal-planning-shared server ────────────────┘           │
│   (shared-server.ts)                                      │
│   HOUSEHOLD_KEY                                           │
│                                                           │
│        HOUSEHOLD MEMBER'S MACHINE                         │
└───────────────────────────────────────────────────────────┘
```

The household member's machine runs a completely separate Node process. It connects to the same Supabase project using a different credential — `SUPABASE_HOUSEHOLD_KEY` — whose JWT carries `role: household_member`. The RLS policies on the database tables read that JWT claim and allow access accordingly. Neither machine can see the other's processes or credentials.

---

## 2. Database Schema

Three tables, all in `schema.sql`.

### `recipes` (lines 5–20)

```sql
CREATE TABLE recipes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES auth.users NOT NULL,
    name TEXT NOT NULL,
    cuisine TEXT,
    prep_time_minutes INTEGER,
    cook_time_minutes INTEGER,
    servings INTEGER,
    ingredients JSONB NOT NULL DEFAULT '[]',   -- [{name, quantity, unit}, ...]
    instructions JSONB NOT NULL DEFAULT '[]',  -- ["Step 1...", "Step 2...", ...]
    tags TEXT[] DEFAULT '{}',
    rating INTEGER CHECK (rating >= 1 AND rating <= 5),
    notes TEXT,
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now()
);
```

Two things to notice. First, `ingredients` and `instructions` are both `JSONB` columns, not normalized tables. Each recipe is self-contained. Second, `tags` is a native `TEXT[]` PostgreSQL array — not JSONB. This matters for indexing: tags gets a GIN index (`idx_recipes_user_tags`) which makes array containment queries fast.

### `meal_plans` (lines 23–34)

```sql
CREATE TABLE meal_plans (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES auth.users NOT NULL,
    week_start DATE NOT NULL,       -- should be a Monday
    day_of_week TEXT NOT NULL,      -- 'monday', 'tuesday', etc.
    meal_type TEXT NOT NULL CHECK (meal_type IN ('breakfast', 'lunch', 'dinner', 'snack')),
    recipe_id UUID REFERENCES recipes,
    custom_meal TEXT,               -- for meals without a recipe
    servings INTEGER,
    notes TEXT,
    created_at TIMESTAMPTZ DEFAULT now()
);
```

Each row is one slot: a single meal on a single day in a single week. Creating a full week's plan means inserting multiple rows at once — the `create_meal_plan` tool does this with a batch insert. The `meal_type` column has a `CHECK` constraint that rejects anything outside the four allowed values at the database level.

`recipe_id` is nullable. A meal can reference a recipe (joining back to get ingredients and instructions) or it can just be free-text in `custom_meal` — useful for nights when you're ordering out.

### `shopping_lists` (lines 37–45)

```sql
CREATE TABLE shopping_lists (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES auth.users NOT NULL,
    week_start DATE NOT NULL,
    items JSONB NOT NULL DEFAULT '[]',  -- [{name, quantity, unit, purchased: bool, recipe_id}]
    notes TEXT,
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now()
);
```

The entire shopping list for a week lives in one JSONB array column. Each element is an object with `name`, `quantity`, `unit`, `purchased` (boolean), and optionally `recipe_id`. To mark an item purchased, you fetch the array, map over it to toggle the matching element, and write the whole array back. This is the JSONB array mutation pattern shown in `mark_item_purchased`.

### Indexes

```sql
CREATE INDEX idx_recipes_user_cuisine ON recipes(user_id, cuisine);
CREATE INDEX idx_recipes_user_tags    ON recipes USING GIN (tags);
CREATE INDEX idx_meal_plans_user_week ON meal_plans(user_id, week_start);
CREATE INDEX idx_shopping_lists_user_week ON shopping_lists(user_id, week_start);
```

The GIN index on `tags` supports the `.contains("tags", [args.tag])` query efficiently. B-tree indexes cover the common week-scoped lookups.

### Row Level Security

RLS is enabled on all three tables (lines 53–113). The policies follow a consistent pattern: users own their own rows via `auth.uid() = user_id`, and household members get read access via a JWT claim check.

```sql
-- Household member access for recipes (SELECT only)
CREATE POLICY "Household members can view recipes"
    ON recipes FOR SELECT
    USING (
        auth.jwt() ->> 'role' = 'household_member'
        OR auth.uid() = user_id
    );
```

The same pattern applies to `meal_plans`. For `shopping_lists`, household members get both SELECT and UPDATE — they need to be able to tick items off, but cannot create or delete lists.

```sql
-- Household members can update shopping lists (to check off items)
CREATE POLICY "Household members can update shopping lists"
    ON shopping_lists FOR UPDATE
    USING (
        auth.jwt() ->> 'role' = 'household_member'
        OR auth.uid() = user_id
    )
    WITH CHECK (
        auth.jwt() ->> 'role' = 'household_member'
        OR auth.uid() = user_id
    );
```

The `USING` clause controls which rows can be targeted; the `WITH CHECK` clause validates the new values being written. Both are set identically here — any household member can target and update any shopping list, as long as they're a household member.

Note that the main server uses `SERVICE_ROLE_KEY`, which bypasses RLS entirely. RLS enforcement is relevant only when the household key (with `role: household_member` in its JWT) is used.

---

## 3. Main Server — 6 Tools

**File**: `extensions/meal-planning/index.ts`
**Credential**: `SUPABASE_SERVICE_ROLE_KEY`

### `add_recipe`

Inserts a row into `recipes`. The `ingredients` array (each element: `{name, quantity, unit}`) and `instructions` array (plain strings) are passed directly as JSONB. Tags default to an empty array if not provided. Required fields: `user_id`, `name`, `ingredients`, `instructions`.

### `search_recipes`

Builds a query dynamically, adding filters only for the parameters that are present:

```typescript
// name — case-insensitive substring match
query = query.ilike("name", `%${args.query}%`);

// cuisine — exact match
query = query.eq("cuisine", args.cuisine);

// tag — array containment on the TEXT[] column
query = query.contains("tags", [args.tag]);

// ingredient — JSONB containment: find arrays that contain an object with this name
query = query.or(
  `ingredients.cs.${JSON.stringify([{ name: args.ingredient }])}`
);
```

The ingredient search uses Supabase's `.or()` with the `cs` (contains) operator on a JSONB column. The value being searched for is `[{ name: "chicken" }]` — a partial object. PostgreSQL's `@>` operator for JSONB will match any array element that contains at least the specified keys and values.

### `update_recipe`

Builds an `updates` object dynamically — only fields explicitly present in the arguments are included. This prevents accidentally nullifying columns that weren't meant to change. Identified by `recipe_id`.

### `create_meal_plan`

Batch inserts the entire week's plan in one operation. The `meals` input array is transformed with `.map()` into database rows, each stamped with `user_id` and `week_start`:

```typescript
const mealEntries = args.meals.map((meal: any) => ({
  user_id: args.user_id,
  week_start: args.week_start,
  day_of_week: meal.day_of_week,
  meal_type: meal.meal_type,
  recipe_id: meal.recipe_id || null,
  custom_meal: meal.custom_meal || null,
  servings: meal.servings || null,
  notes: meal.notes || null,
}));

await supabase.from("meal_plans").insert(mealEntries).select();
```

Supabase accepts the array directly. One round trip creates all rows.

### `get_meal_plan`

Fetches the week's meal plan with a relational select that joins the linked recipe:

```typescript
await supabase
  .from("meal_plans")
  .select(`
    *,
    recipes:recipe_id (name, cuisine, prep_time_minutes, cook_time_minutes)
  `)
  .eq("user_id", args.user_id)
  .eq("week_start", args.week_start)
  .order("day_of_week")
  .order("meal_type");
```

The `recipes:recipe_id (...)` syntax tells Supabase to follow the foreign key from `meal_plans.recipe_id` to `recipes.id` and embed the selected columns inline. Rows with no `recipe_id` get `null` for the embedded object.

### `generate_shopping_list`

This is the most complex handler. It does five things in sequence:

**Step 1** — Fetch the week's meal plan, joining recipe ingredients:

```typescript
await supabase
  .from("meal_plans")
  .select(`*, recipes:recipe_id (id, ingredients, name)`)
  .eq("user_id", args.user_id)
  .eq("week_start", args.week_start);
```

**Step 2** — Aggregate ingredients across all meals using a `Map` keyed by `name-unit`:

```typescript
const itemsMap = new Map();

mealPlan?.forEach((meal: any) => {
  if (meal.recipes && meal.recipes.ingredients) {
    meal.recipes.ingredients.forEach((ingredient) => {
      const key = `${ingredient.name}-${ingredient.unit}`;
      if (itemsMap.has(key)) {
        const existing = itemsMap.get(key);
        existing.quantity = `${existing.quantity} + ${ingredient.quantity}`;
      } else {
        itemsMap.set(key, {
          name: ingredient.name,
          quantity: ingredient.quantity,
          unit: ingredient.unit,
          purchased: false,
          recipe_id: meal.recipes.id,
        });
      }
    });
  }
});
```

When the same ingredient (same name and unit) appears in multiple recipes, quantities are concatenated as strings (e.g. `"2 + 1"`). The comment in the code acknowledges this is intentionally simple — real quantity arithmetic would need unit parsing. Every item starts with `purchased: false`.

**Step 3** — Convert the Map to an array: `Array.from(itemsMap.values())`.

**Step 4** — Check whether a shopping list already exists for this week:

```typescript
const { data: existing } = await supabase
  .from("shopping_lists")
  .select("id")
  .eq("user_id", args.user_id)
  .eq("week_start", args.week_start)
  .single();
```

**Step 5** — Upsert: update the existing list if found, insert a new one if not. This makes the tool idempotent — running it twice after adding a new meal regenerates the list cleanly.

---

## 4. Shared Server — 4 Tools

**File**: `extensions/meal-planning/shared-server.ts`
**Credential**: `SUPABASE_HOUSEHOLD_KEY`

The server is registered as `"meal-planning-shared"` and runs as a completely separate process. It uses `SUPABASE_HOUSEHOLD_KEY` — a credential whose JWT carries `role: household_member`. All four tools operate within the bounds that the RLS policies allow for that role.

### `view_meal_plan`

Identical join query to the main server's `get_meal_plan`, but includes `servings` in the recipe projection. Read-only.

### `view_recipes`

Explicitly selects a limited column set — no `ingredients`, no `instructions`, no `notes`:

```typescript
.select("id, name, cuisine, prep_time_minutes, cook_time_minutes, servings, tags, rating")
```

A household member can browse the recipe collection and filter by name, cuisine, or tag. They cannot see full ingredient lists or step-by-step instructions from this tool. Supports the same `ilike`, `eq`, and `contains` filters as the main server.

### `view_shopping_list`

Fetches the full shopping list row for a week, including the `items` JSONB array with all purchased flags. Read-only.

### `mark_item_purchased`

The only write operation in the shared server. It implements the JSONB array mutation pattern:

```typescript
// Step 1: Fetch the current items array
const { data: list } = await supabase
  .from("shopping_lists")
  .select("items")
  .eq("id", args.shopping_list_id)
  .single();

// Step 2: Map over the array, toggling the matching item
const updatedItems = list.items.map((item) => {
  if (item.name === args.item_name) {
    return { ...item, purchased: args.purchased };
  }
  return item;
});

// Step 3: Write the whole array back
await supabase
  .from("shopping_lists")
  .update({ items: updatedItems, updated_at: new Date().toISOString() })
  .eq("id", args.shopping_list_id)
  .select()
  .single();
```

The `purchased` parameter is explicit — the caller sets the target state rather than toggling blindly. This is safer for concurrent use: if two household members are both shopping, setting `purchased: true` twice produces the correct result, whereas a toggle would flip back.

### Security boundary summary

| Operation | Main server | Shared server |
|---|---|---|
| Create recipe | Yes | No |
| Update recipe | Yes | No |
| Search recipes (full) | Yes | No |
| Browse recipes (limited columns) | Yes | Yes |
| Create meal plan | Yes | No |
| View meal plan | Yes | Yes |
| Generate shopping list | Yes | No |
| View shopping list | Yes | Yes |
| Mark item purchased | Yes | Yes |
| Delete anything | No | No |

---

## 5. Key Patterns

### JSONB arrays for structured data within a row

Both `ingredients` (in `recipes`) and `items` (in `shopping_lists`) are stored as JSONB arrays rather than normalized into child tables. This keeps the data model simple and each row self-contained, at the cost of making cross-ingredient queries harder. For a personal meal planning system that rarely needs "find all recipes using this ingredient across all users," the tradeoff is reasonable.

The `instructions` column in `recipes` follows the same pattern — an ordered array of strings stored as JSONB.

### TEXT[] vs JSONB for tags

Tags are a `TEXT[]` column, not JSONB. This is intentional. PostgreSQL has native operators for array membership (`@>`, `= ANY()`) and supports GIN indexes on `TEXT[]` columns. When the values are a flat list of strings with no structure, `TEXT[]` is more appropriate and slightly more efficient than JSONB.

### Ingredient aggregation with a Map

The `generate_shopping_list` handler uses a JavaScript `Map` to deduplicate and merge ingredients across recipes. The key `${name}-${unit}` ensures that "2 cups flour" and "1 cup flour" from different recipes combine into one entry (with concatenated quantities) rather than appearing twice. Items across different units (e.g., "1 cup milk" and "200ml milk") are kept separate because the unit is part of the key.

### Batch insert with `.map()`

`create_meal_plan` collects the whole week's meals in a single `.insert()` call by transforming the input array with `.map()`. This is more efficient than looping and inserting one row at a time, and it either succeeds fully or fails fully — there's no partial state where Monday was inserted but Friday wasn't (assuming the database transaction rolls back on error).

### Upsert by check-then-branch

`generate_shopping_list` checks for an existing list before deciding to insert or update. This is a manual upsert pattern. Supabase's `.upsert()` method could also work here if the table had a unique constraint on `(user_id, week_start)` — but since the schema doesn't define one, checking first and branching is the correct approach.

### Household RLS via JWT claims

Rather than building a `household_members` junction table (as the `rls` primitive illustrates as an alternative), this extension uses a simpler model: a JWT claim. If `auth.jwt() ->> 'role' = 'household_member'`, the policy allows access. This works when the "household" is effectively one shared credential rather than individually-authenticated members. It is straightforward to set up but less granular — you cannot, for example, distinguish between different household members or revoke one person's access without changing the shared key.

---

## 6. How It Fits (Primitives and Extensions)

This is the first extension in the path that references two primitives explicitly.

**`primitives/rls`** — The household-scoped RLS policies here are a direct application of Pattern 2 from the RLS primitive, adapted to use JWT claims instead of a `household_members` join. The RLS primitive explains the theory (additive policies, `auth.uid()` vs `auth.jwt()`, service role bypass); this extension shows the practice.

**`primitives/shared-mcp`** — The shared-server pattern described in the primitive is implemented verbatim here. `shared-server.ts` is the concrete artifact the primitive describes: a separate server, separate credential, limited column selection in `view_recipes`, write permission scoped to a single field mutation. If you read the primitive first and then read `shared-server.ts`, every design decision in the code should be recognizable.

**Extension 3 (home maintenance) → Extension 4 (meal planning)** — Home maintenance introduced JSONB for storing structured data (issue details, contractor notes) without a rigid schema. Meal planning uses the same technique more aggressively: `ingredients`, `instructions`, and the entire `shopping_list.items` are all JSONB arrays. The pattern is the same; the complexity increases.

**Future integration with Extension 2 (household knowledge)** — The shared server could be extended to call the household knowledge base when deciding what to cook. A more capable version of `generate_shopping_list` could check family calendar events (who's home for dinner, are guests expected) and adjust the week's plan accordingly. That integration point is outside this extension's scope but follows naturally from having both servers share a Supabase project.

---

## 7. Source Files

| File | Lines | Purpose |
|---|---|---|
| `extensions/meal-planning/index.ts` | 502 | Main MCP server — 6 tools, full read/write |
| `extensions/meal-planning/shared-server.ts` | 279 | Shared MCP server — 4 tools, household-scoped |
| `extensions/meal-planning/schema.sql` | 113 | 3 tables, indexes, RLS policies |

Related primitives:

- `primitives/rls/README.md` — Row Level Security patterns, including JWT-claim-based household access
- `primitives/shared-mcp/README.md` — Scoped server pattern, credential isolation, deployment to another machine
