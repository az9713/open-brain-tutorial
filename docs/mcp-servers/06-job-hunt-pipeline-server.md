# Extension 6: Job Hunt Pipeline MCP Server — Deep Dive

This document explains every technical decision in the Job Hunt Pipeline extension. It
assumes you have worked through Extensions 1 through 5 and are comfortable with
TypeScript, Supabase relational selects, Row Level Security, and the cross-extension
patterns introduced earlier in the learning path. Extension 6 is the most complex
extension in the set. Where it introduces new patterns — multi-table FK chains, 3-level
nested relational selects, cross-extension writes, and application-layer soft foreign keys
— those patterns are explained from first principles.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Role in the Extension Architecture](#2-role-in-the-extension-architecture)
3. [Database Schema](#3-database-schema)
4. [The Eight-State Application Pipeline](#4-the-eight-state-application-pipeline)
5. [The Eight Tools](#5-the-eight-tools)
6. [The 3-Level Nested Relational Select](#6-the-3-level-nested-relational-select)
7. [The Cross-Extension Write: `link_contact_to_professional_crm`](#7-the-cross-extension-write-link_contact_to_professional_crm)
8. [The Soft Foreign Key Pattern](#8-the-soft-foreign-key-pattern)
9. [Indexes — Including the Partial Index](#9-indexes--including-the-partial-index)
10. [Row Level Security and Triggers](#10-row-level-security-and-triggers)
11. [How This Server Fits with Other Extensions](#11-how-this-server-fits-with-other-extensions)
12. [Source File Reference](#12-source-file-reference)

---

## 1. Overview

The Job Hunt Pipeline MCP server manages the full lifecycle of a job search from initial
company research through offer acceptance. It is the largest and most interconnected
extension in the set: 697 lines of TypeScript, 175 lines of SQL, 5 database tables, and
8 tools.

The server covers five distinct domains:

- **Company research** — organizations being tracked, including size, location, remote
  policy, and Glassdoor rating
- **Job postings** — specific open roles at those companies, with salary ranges, sourcing
  information, and requirement lists
- **Applications** — the act of applying, with an 8-state status pipeline tracking the
  application from draft through final outcome
- **Interviews** — individual interview events with scheduling, preparation notes, and
  post-interview feedback
- **Job contacts** — people encountered during the search (recruiters, hiring managers,
  referrals, interviewers), with an optional bridge to Extension 5's Professional CRM

The extension introduces three patterns not seen in earlier extensions:

1. **Multi-table FK chains** — a 4-table chain (companies → job_postings → applications
   → interviews) where queries must traverse the entire chain to produce useful output
2. **Cross-extension writes** — `link_contact_to_professional_crm` does not just read from
   another extension's table; it inserts a new row into Extension 5's
   `professional_contacts` table and writes back the resulting ID
3. **Soft foreign keys** — `job_contacts.professional_crm_contact_id` holds a UUID
   reference to `professional_contacts.id` with no database-level `REFERENCES` constraint.
   The link is managed entirely in application code.

---

## 2. Role in the Extension Architecture

Extension 6 is the terminal extension in the learning path and the only one that writes to
another extension's tables at runtime.

```
┌─────────────────────────────────────────────────────────┐
│                   Extension 6: Job Hunt                  │
│                                                         │
│  companies ──► job_postings ──► applications ──► interviews
│                                                         │
│  job_contacts ──────────────────────────────────────────┤
│       │         soft FK (no REFERENCES constraint)      │
│       │                                                 │
│       ▼                                                 │
│  ┌─────────────────────────────────────────────────┐    │
│  │       Extension 5: Professional CRM              │    │
│  │       professional_contacts table                │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

Extension 5 (Professional CRM) is a hard dependency for the `link_contact_to_professional_crm`
tool. If Extension 5 is not set up — meaning the `professional_contacts` table does not
exist in the Supabase project — that tool will throw a Supabase error at runtime. The
other 7 tools are fully independent.

The dependency is intentional and architecturally meaningful: people you meet during a job
search are often long-term professional relationships. A recruiter who places you, a
hiring manager who interviews you, a peer who refers you — these contacts have value beyond
the job search itself. The bridge tool lets the AI agent promote a job-search contact into
the professional network with a single call.

---

## 3. Database Schema

The schema defines a strict 4-level FK hierarchy for the core pipeline tables, plus one
supporting table for contacts.

### 3.1 Entity Relationship Overview

```
auth.users
    │
    ├──► companies
    │         │
    │         ├──► job_postings
    │         │         │
    │         │         └──► applications
    │         │                   │
    │         │                   └──► interviews
    │         │
    │         └──► job_contacts (optional, ON DELETE SET NULL)
    │
    └──► job_contacts (direct user FK)
```

All five tables reference `auth.users(id) ON DELETE CASCADE`. The cross-table FK chain
uses `ON DELETE CASCADE` at every level: deleting a company cascades to its job postings,
which cascades to applications, which cascades to interviews.

The `job_contacts` table is the only table with a different delete behavior. Its
`company_id` column references `companies(id) ON DELETE SET NULL` (not CASCADE). If a
company is deleted, its associated contacts are not deleted — their `company_id` is set to
NULL. This preserves contact records even when the company relationship is dissolved.

### 3.2 `companies`

```sql
CREATE TABLE IF NOT EXISTS companies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
    name TEXT NOT NULL,
    industry TEXT,
    website TEXT,
    size TEXT CHECK (size IN ('startup', 'mid-market', 'enterprise') OR size IS NULL),
    location TEXT,
    remote_policy TEXT CHECK (remote_policy IN ('remote', 'hybrid', 'onsite') OR size IS NULL),
    notes TEXT,
    glassdoor_rating DECIMAL(2,1) CHECK (glassdoor_rating >= 1.0 AND glassdoor_rating <= 5.0 OR glassdoor_rating IS NULL),
    created_at TIMESTAMPTZ DEFAULT now() NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT now() NOT NULL
);
```

Three columns have constrained value sets enforced at the database level:

- `size` — `CHECK (size IN ('startup', 'mid-market', 'enterprise') OR size IS NULL)`. The
  `OR size IS NULL` clause is required in every enum CHECK here. Without it, a `NULL` value
  would fail the constraint, making every enum column implicitly `NOT NULL`.
- `remote_policy` — `CHECK (remote_policy IN ('remote', 'hybrid', 'onsite') OR ...)`. Same
  pattern. Three values cover the practical range without over-specifying.
- `glassdoor_rating` — `DECIMAL(2,1)` stores values like `4.2`, with a range check of 1.0
  to 5.0. `DECIMAL(2,1)` means at most 2 significant digits, 1 after the decimal point.
  This is the only numeric precision type in the Extension 6 schema; all other numeric
  columns use `INTEGER`.

`companies` is one of two tables in this extension with an `updated_at` column and a
corresponding trigger. The trigger fires automatically on every `UPDATE` statement.

### 3.3 `job_postings`

```sql
CREATE TABLE IF NOT EXISTS job_postings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id UUID REFERENCES companies(id) ON DELETE CASCADE NOT NULL,
    user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
    title TEXT NOT NULL,
    url TEXT,
    salary_min INTEGER,
    salary_max INTEGER,
    salary_currency TEXT DEFAULT 'USD',
    requirements TEXT[],
    nice_to_haves TEXT[],
    notes TEXT,
    source TEXT CHECK (source IN ('linkedin', 'company-site', 'referral', 'recruiter', 'other') OR source IS NULL),
    posted_date DATE,
    closing_date DATE,
    created_at TIMESTAMPTZ DEFAULT now() NOT NULL
);
```

Two columns use PostgreSQL arrays (`TEXT[]`):

- `requirements TEXT[]` — the list of required qualifications from the job posting
- `nice_to_haves TEXT[]` — the list of preferred but non-required skills

Array columns are stored as a PostgreSQL array type, not as JSON. In the TypeScript
handler, they default to empty arrays with `requirements: requirements || []`. This means
a job posting with no requirements specified will store `{}` in PostgreSQL (empty array)
rather than `NULL`. Both are valid; defaulting to empty array makes client-side iteration
safe without a null check.

`salary_min` and `salary_max` are `INTEGER` — whole numbers in whatever currency
`salary_currency` specifies. Fractional salary values are not modeled. The currency
defaults to `'USD'` in the schema, and the tool handler also applies `|| "USD"` as an
application-layer default.

`source` enforces a 5-value enum: `linkedin`, `company-site`, `referral`, `recruiter`,
`other`. This is more granular than "where did you hear about us?" fields typically seen
in HR systems — it distinguishes active sourcing (recruiter reached out) from passive
discovery (found it on LinkedIn) and social proof (referral).

Note that `job_postings` does not have an `updated_at` column or trigger. The schema
author treats job postings as append-only records — the posting itself does not change;
if information is wrong, a new posting row would be created. This is a deliberate design
choice, not an omission.

### 3.4 `applications`

```sql
CREATE TABLE IF NOT EXISTS applications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_posting_id UUID REFERENCES job_postings(id) ON DELETE CASCADE NOT NULL,
    user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
    status TEXT DEFAULT 'applied' CHECK (status IN ('draft', 'applied', 'screening',
        'interviewing', 'offer', 'accepted', 'rejected', 'withdrawn')),
    applied_date DATE,
    response_date DATE,
    resume_version TEXT,
    cover_letter_notes TEXT,
    referral_contact TEXT,
    notes TEXT,
    created_at TIMESTAMPTZ DEFAULT now() NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT now() NOT NULL
);
```

The `status` column implements an 8-state pipeline described fully in
[Section 4](#4-the-eight-state-application-pipeline). The default is `'applied'` because
`submit_application` is the tool for recording an application — pre-submission drafts are
supported but not the primary use case.

`referral_contact` is a free-text field, not a foreign key to `job_contacts`. This is
intentional: the referral might be someone not tracked as a contact, or the contact data
might not be available at application time. Free-text is more flexible here.

`applications` is the second table with `updated_at` and a corresponding trigger.
Status transitions — from `applied` to `screening` to `interviewing` — are updates to
this row, so tracking when the row last changed is meaningful.

### 3.5 `interviews`

```sql
CREATE TABLE IF NOT EXISTS interviews (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id UUID REFERENCES applications(id) ON DELETE CASCADE NOT NULL,
    user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
    interview_type TEXT CHECK (interview_type IN ('phone_screen', 'technical', 'behavioral',
        'system_design', 'hiring_manager', 'team', 'final')),
    scheduled_at TIMESTAMPTZ,
    duration_minutes INTEGER,
    interviewer_name TEXT,
    interviewer_title TEXT,
    status TEXT DEFAULT 'scheduled' CHECK (status IN ('scheduled', 'completed', 'cancelled', 'no_show')),
    notes TEXT,        -- pre-interview prep notes
    feedback TEXT,     -- post-interview reflection
    rating INTEGER CHECK (rating >= 1 AND rating <= 5 OR rating IS NULL),
    created_at TIMESTAMPTZ DEFAULT now() NOT NULL
);
```

`interviews` has two separate text fields for notes:

- `notes` — pre-interview prep notes, written before the interview
- `feedback` — post-interview reflection, written after

The `log_interview_notes` tool writes to `feedback` and `rating`, not `notes`. This
preserves the preparation context (what you planned to say) alongside the reflection (what
actually happened).

`rating` is an integer 1–5 representing the applicant's self-assessment of how the
interview went. It is distinct from any employer rating; this is a personal record.

`interview_type` enumerates 7 values covering a typical engineering hiring funnel:
`phone_screen` → `technical` → `behavioral` → `system_design` → `hiring_manager` →
`team` → `final`. Not every process uses every stage; the enum is a superset.

`scheduled_at` is nullable. An interview can be created without a confirmed time — for
example, when a company says "we'll do a phone screen next week" but hasn't provided a
calendar slot yet. The partial index in [Section 9](#9-indexes--including-the-partial-index)
covers this nullable column specifically.

`interviews` does not have an `updated_at` column or trigger. Interview rows are created
once (scheduled) and updated once (when notes are logged). The `created_at` timestamp and
the `status` field together provide sufficient audit trail.

### 3.6 `job_contacts`

```sql
CREATE TABLE IF NOT EXISTS job_contacts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
    company_id UUID REFERENCES companies(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    title TEXT,
    email TEXT,
    phone TEXT,
    linkedin_url TEXT,
    role_in_process TEXT CHECK (role_in_process IN ('recruiter', 'hiring_manager',
        'referral', 'interviewer', 'other') OR role_in_process IS NULL),
    professional_crm_contact_id UUID,  -- soft FK to Extension 5
    notes TEXT,
    last_contacted TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT now() NOT NULL
);
```

`job_contacts` is the integration point between this extension and Extension 5. The
`professional_crm_contact_id` column is the soft FK; its design is explained fully in
[Section 8](#8-the-soft-foreign-key-pattern).

`role_in_process` classifies the contact's function in the hiring process: `recruiter`,
`hiring_manager`, `referral`, `interviewer`, or `other`. This value is used by the
`link_contact_to_professional_crm` tool to populate the `how_we_met` field in
Extension 5 — it becomes `"Job search - recruiter"` or `"Job search - hiring_manager"`.

`company_id` uses `ON DELETE SET NULL` (not CASCADE). The intention is that deleting a
company from your tracker should not erase a contact record — the person still exists and
may be valuable to know even if you stop tracking that company.

---

## 4. The Eight-State Application Pipeline

The `applications.status` column is a finite state machine with 8 values:

```
draft ──► applied ──► screening ──► interviewing ──► offer ──► accepted
                                                         │
                                                         └──► rejected
           │                │                │
           └────────────────┴────────────────┴──► withdrawn
```

| State | Meaning |
|---|---|
| `draft` | Application being prepared; not yet submitted |
| `applied` | Submitted; awaiting response (default) |
| `screening` | Recruiter or phone screen stage |
| `interviewing` | Active interview loop in progress |
| `offer` | Offer received; evaluating |
| `accepted` | Offer accepted |
| `rejected` | Rejected at any stage |
| `withdrawn` | Candidate withdrew from process |

The schema does not enforce state machine transitions — any `UPDATE` to `status` is
accepted regardless of the current state. Moving from `accepted` back to `interviewing`
would not be blocked at the database level. Transition logic, if needed, would live in
the application layer or be enforced by the AI agent's instructions.

The `get_pipeline_overview` tool uses this status field to produce a dashboard. It fetches
all application rows with `.select("status")` and then uses JavaScript's `Array.reduce()`
to count occurrences of each status value:

```typescript
const statusCounts = applications.reduce((acc: any, app: any) => {
  acc[app.status] = (acc[app.status] || 0) + 1;
  return acc;
}, {});
```

This produces a shape like:

```json
{
  "applied": 12,
  "screening": 4,
  "interviewing": 2,
  "offer": 1,
  "rejected": 8,
  "withdrawn": 1
}
```

The reduce approach is done in JavaScript rather than SQL (`GROUP BY status`) because the
same query already fetches all rows for the user, and a JavaScript aggregation avoids a
second database round trip. At the scale of a personal job search (tens to low hundreds
of applications), the performance difference is negligible.

---

## 5. The Eight Tools

### 5.1 `add_company`

**Purpose:** Add a company to track.

**Required parameters:** `user_id`, `name`
**Optional parameters:** `industry`, `website`, `size`, `location`, `remote_policy`,
`notes`, `glassdoor_rating`

**Implementation (lines 303–331):**

A straightforward single-table insert into `companies`. All optional fields use `|| null`
coercion so that missing or empty-string values from the AI agent are stored as `NULL`
rather than empty strings. The handler returns the full inserted row via `.select().single()`
so the caller receives the generated UUID for subsequent `add_job_posting` calls.

The `glassdoor_rating` field accepts a number between 1.0 and 5.0. The tool schema
enforces this range at the MCP layer with `minimum: 1.0` and `maximum: 5.0`; the database
also enforces it with a CHECK constraint. Both layers validate independently.

---

### 5.2 `add_job_posting`

**Purpose:** Add a specific open role at a tracked company.

**Required parameters:** `user_id`, `company_id`, `title`
**Optional parameters:** `url`, `salary_min`, `salary_max`, `salary_currency`,
`requirements`, `nice_to_haves`, `notes`, `source`, `posted_date`, `closing_date`

**Implementation (lines 333–368):**

```typescript
const { data, error } = await supabase
  .from("job_postings")
  .insert({
    user_id,
    company_id,
    title,
    // ...
    requirements: requirements || [],
    nice_to_haves: nice_to_haves || [],
    salary_currency: salary_currency || "USD",
    // ...
  })
  .select()
  .single();
```

The array fields default to `[]` (empty array) rather than `null`. This makes client-side
code that iterates over requirements safe without a null check: `posting.requirements.map(...)`
works even if no requirements were specified.

`company_id` is required, which means the calling agent must use `add_company` before
`add_job_posting` and pass the company UUID forward. This is the first time in the
extension the agent must chain two tool calls and retain state (the company ID) between
them.

---

### 5.3 `submit_application`

**Purpose:** Record an application for a job posting.

**Required parameters:** `user_id`, `job_posting_id`
**Optional parameters:** `status`, `applied_date`, `resume_version`, `cover_letter_notes`,
`referral_contact`, `notes`

**Implementation (lines 370–400):**

The default status is `"applied"`, applied in code with `status: status || "applied"`. This
mirrors the database default but is explicit in the handler so the returned JSON always
shows the effective status, not `null`.

`job_posting_id` is required. The chain is now: company UUID → job_posting UUID →
application UUID. Each layer depends on the previous. An agent managing a full job search
session must maintain these IDs across tool calls — typically by storing them in the
conversation context.

---

### 5.4 `schedule_interview`

**Purpose:** Schedule an interview for an existing application.

**Required parameters:** `user_id`, `application_id`, `interview_type`
**Optional parameters:** `scheduled_at`, `duration_minutes`, `interviewer_name`,
`interviewer_title`, `notes`

**Implementation (lines 402–433):**

The insert hardcodes `status: "scheduled"`. A newly created interview row is always in
the `scheduled` state. The `log_interview_notes` tool transitions it to `"completed"`.
There is no tool to set `"cancelled"` or `"no_show"` — those statuses exist in the schema
for completeness, but the current tool set does not expose them. A future tool could
handle cancellations.

`scheduled_at` is optional (`|| null`). This enables recording that an interview is
coming without knowing the exact time, a common real-world situation where a company
confirms a stage but has not sent a calendar invite yet.

---

### 5.5 `log_interview_notes`

**Purpose:** Record post-interview reflection and mark the interview as completed.

**Required parameters:** `user_id`, `interview_id`
**Optional parameters:** `feedback`, `rating`

**Implementation (lines 435–459):**

```typescript
const { data, error } = await supabase
  .from("interviews")
  .update({
    feedback: feedback || null,
    rating: rating || null,
    status: "completed",
  })
  .eq("id", interview_id)
  .eq("user_id", user_id)
  .select()
  .single();
```

This is the only tool in the extension that uses `.update()` rather than `.insert()`. Three
things happen in a single database round trip:

1. `feedback` is written (post-interview notes)
2. `rating` is written (self-assessment score 1–5)
3. `status` is set to `"completed"`

The double `.eq()` — filtering by both `id` and `user_id` — is the RLS guard pattern used
throughout the extension. Even though RLS policies exist on this table, the application
layer also scopes every write to the authenticated user. This is belt-and-suspenders:
the application cannot accidentally update another user's interview even if RLS were
misconfigured.

---

### 5.6 `get_pipeline_overview`

**Purpose:** Return a dashboard: application counts by status, plus upcoming interviews.

**Required parameters:** `user_id`
**Optional parameters:** `days_ahead` (defaults to 7)

**Implementation (lines 461–513):**

This tool makes two separate database queries and combines their results:

**Query 1 — Status counts:**
```typescript
const { data: applications } = await supabase
  .from("applications")
  .select("status")
  .eq("user_id", user_id);

const statusCounts = applications.reduce((acc, app) => {
  acc[app.status] = (acc[app.status] || 0) + 1;
  return acc;
}, {});
```

Only the `status` column is fetched — not the full row — because only the status value is
needed for counting. This is a narrow projection that reduces data transfer.

**Query 2 — Upcoming interviews with full context:**

The nested relational select here is identical to the one used in `get_upcoming_interviews`.
It is documented fully in [Section 6](#6-the-3-level-nested-relational-select).

The final response shape:

```json
{
  "success": true,
  "total_applications": 28,
  "status_breakdown": { "applied": 12, "interviewing": 3, "rejected": 8, ... },
  "upcoming_interviews_count": 2,
  "upcoming_interviews": [ ... ]
}
```

---

### 5.7 `get_upcoming_interviews`

**Purpose:** List interviews in the next N days with full company and role context.

**Required parameters:** `user_id`
**Optional parameters:** `days_ahead` (defaults to 14)

**Implementation (lines 515–549):**

This tool is a focused version of part of `get_pipeline_overview`. It performs a single
nested relational select filtered to `status = "scheduled"` and `scheduled_at` within
the time window. The default lookahead is 14 days (versus 7 days for the pipeline
overview), reflecting the different use case: the overview is a daily dashboard while
this tool is for interview preparation.

The nested select structure is the most complex Supabase query in the codebase. It is
covered in [Section 6](#6-the-3-level-nested-relational-select).

---

### 5.8 `link_contact_to_professional_crm`

**Purpose:** Promote a job contact into Extension 5's Professional CRM.

**Required parameters:** `user_id`, `job_contact_id`

This is the most important and most complex tool in the extension. It is covered fully in
[Section 7](#7-the-cross-extension-write-link_contact_to_professional_crm).

---

## 6. The 3-Level Nested Relational Select

The most complex Supabase query in the entire codebase appears in both `get_pipeline_overview`
and `get_upcoming_interviews`:

```typescript
const { data, error } = await supabase
  .from("interviews")
  .select(`
    *,
    applications!inner(
      *,
      job_postings!inner(
        *,
        companies!inner(*)
      )
    )
  `)
  .eq("user_id", user_id)
  .eq("status", "scheduled")
  .gte("scheduled_at", new Date().toISOString())
  .lte("scheduled_at", futureDate.toISOString())
  .order("scheduled_at", { ascending: true });
```

### 6.1 How Supabase Embedded Resource Selects Work

The Supabase JavaScript client uses PostgREST's embedded resource select syntax. When you
write `applications!inner(*)` inside a `.select()` string, you are instructing PostgREST
to:

1. Identify the foreign key relationship between `interviews` and `applications`
   (`interviews.application_id → applications.id`)
2. Perform a JOIN on that relationship
3. Embed the joined row(s) as a nested object in the response

The `!inner` modifier changes the join type from `LEFT JOIN` to `INNER JOIN`. Without
`!inner`, an interview row with no matching application (which cannot happen given the
schema's NOT NULL FK constraint, but PostgREST does not know that) would still be returned
with a `null` applications object. With `!inner`, only interviews that have a matching
application row are returned. More importantly, `!inner` also propagates filtering: if you
filter on a column inside the nested selection, `!inner` ensures that interviews with no
matching filtered application are excluded from results.

### 6.2 The Three Levels

```
interviews
  └── applications!inner          (level 1: interviews → applications via application_id)
        └── job_postings!inner    (level 2: applications → job_postings via job_posting_id)
              └── companies!inner (level 3: job_postings → companies via company_id)
```

Each `!inner` follows one foreign key:

| Join | FK Column | Direction |
|---|---|---|
| Level 1 | `interviews.application_id` | → `applications.id` |
| Level 2 | `applications.job_posting_id` | → `job_postings.id` |
| Level 3 | `job_postings.company_id` | → `companies.id` |

### 6.3 Why This Query Is Necessary

An interview row on its own contains:
```json
{
  "id": "...",
  "application_id": "...",
  "interview_type": "technical",
  "scheduled_at": "2026-03-16T14:00:00Z",
  "interviewer_name": "Sarah Chen"
}
```

That is not useful to a human or an AI agent preparing for the interview. Without the
nested select, the agent would need to:

1. Fetch the interview row
2. Use `application_id` to fetch the application row
3. Use `application_id → job_posting_id` to fetch the job posting
4. Use `job_posting_id → company_id` to fetch the company

Four separate round trips. The nested relational select collapses all four into a single
query. The result includes the company name, the job title, the application status, and
all interview details in one response object.

### 6.4 The Result Shape

```json
{
  "id": "interview-uuid",
  "interview_type": "technical",
  "scheduled_at": "2026-03-16T14:00:00Z",
  "interviewer_name": "Sarah Chen",
  "status": "scheduled",
  "notes": "Review system design fundamentals",
  "applications": {
    "id": "application-uuid",
    "status": "interviewing",
    "applied_date": "2026-02-20",
    "job_postings": {
      "id": "posting-uuid",
      "title": "Senior Software Engineer",
      "salary_min": 180000,
      "salary_max": 220000,
      "companies": {
        "id": "company-uuid",
        "name": "Acme Corp",
        "industry": "Enterprise Software",
        "remote_policy": "hybrid"
      }
    }
  }
}
```

The `*` selector at each level includes all columns from that table. In production use,
you would narrow these projections to only the columns the agent actually needs, but the
current implementation favors completeness over efficiency.

---

## 7. The Cross-Extension Write: `link_contact_to_professional_crm`

This tool is unique in the extension set. Every other cross-extension interaction in
Extensions 1–5 is a read — one extension's server queries another extension's tables. This
tool writes to another extension's table.

**Implementation (lines 551–633):**

The tool performs four operations in sequence:

**Step 1 — Fetch the job contact:**
```typescript
const { data: jobContact, error: contactError } = await supabase
  .from("job_contacts")
  .select("*")
  .eq("id", job_contact_id)
  .eq("user_id", user_id)
  .single();
```

Standard read with user scoping. If the contact does not exist or belongs to another user,
the query returns an error and the tool throws.

**Step 2 — Check if already linked:**
```typescript
if (jobContact.professional_crm_contact_id) {
  return JSON.stringify({
    success: true,
    message: "Contact already linked to Professional CRM",
    job_contact: jobContact,
    already_linked: true,
  }, null, 2);
}
```

Idempotency guard. If the contact already has a `professional_crm_contact_id`, the tool
returns immediately with `already_linked: true` rather than creating a duplicate entry in
Extension 5. This is important because the tool might be called multiple times — by the
agent retrying after an apparent failure, or by a user who has forgotten the contact was
already linked.

**Step 3 — Optionally resolve the company name:**
```typescript
let companyName = null;
if (jobContact.company_id) {
  const { data: company } = await supabase
    .from("companies")
    .select("name")
    .eq("id", jobContact.company_id)
    .single();
  companyName = company?.name;
}
```

`job_contacts.company_id` is nullable. If the contact has a company association, this
step resolves the UUID to a name string. Extension 5's `professional_contacts.company`
column is a TEXT field (free-text company name), not a UUID FK. The bridge must translate
from the FK-based Extension 6 model to the text-based Extension 5 model.

**Step 4 — Write to Extension 5:**
```typescript
const { data: professionalContact, error: crmError } = await supabase
  .from("professional_contacts")
  .insert({
    user_id,
    name: jobContact.name,
    company: companyName,
    title: jobContact.title,
    email: jobContact.email,
    phone: jobContact.phone,
    linkedin_url: jobContact.linkedin_url,
    how_we_met: `Job search - ${jobContact.role_in_process || 'contact'}`,
    tags: ["job-hunt", jobContact.role_in_process || "contact"],
    notes: jobContact.notes,
    last_contacted: jobContact.last_contacted,
  })
  .select()
  .single();
```

This is an `INSERT` into `professional_contacts` — a table owned by Extension 5. The
`how_we_met` field is constructed from `role_in_process`, producing values like
`"Job search - recruiter"` or `"Job search - hiring_manager"`. The `tags` array receives
two values: the constant `"job-hunt"` (for filtering all job-search contacts in Extension
5) and the role value.

**Step 5 — Write back the link ID:**
```typescript
const { data: updatedJobContact } = await supabase
  .from("job_contacts")
  .update({ professional_crm_contact_id: professionalContact.id })
  .eq("id", job_contact_id)
  .eq("user_id", user_id)
  .select()
  .single();
```

After Extension 5 creates the new row and returns its UUID, that UUID is written back to
`job_contacts.professional_crm_contact_id`. This completes the bidirectional link: from
Extension 6 you can follow `professional_crm_contact_id` to find the contact in Extension
5; from Extension 5 you can search by the `"job-hunt"` tag to find all job-search-origin
contacts.

### 7.1 What Happens If Step 4 Succeeds But Step 5 Fails

This is the data integrity risk in the cross-extension write. If the `professional_contacts`
insert succeeds but the `job_contacts` update fails, the system is in an inconsistent
state: a professional contact exists in Extension 5 with no corresponding
`professional_crm_contact_id` in Extension 6. The next call to `link_contact_to_professional_crm`
for the same contact would pass the idempotency check (Step 2) because
`professional_crm_contact_id` is still null, and would create a second duplicate row in
Extension 5.

This is a known limitation. A full solution would use a database transaction wrapping
both writes. The current implementation does not use a transaction because the two writes
span logically separate tables (and conceptually separate extensions), and Supabase's
JavaScript client does not provide a simple cross-table transaction API without using an
RPC stored procedure. For a personal job search tool, the risk is low enough to accept.
If you need atomicity, the fix is a Supabase RPC function that wraps both writes in
`BEGIN`/`COMMIT`.

---

## 8. The Soft Foreign Key Pattern

`job_contacts.professional_crm_contact_id` is declared as `UUID` with no `REFERENCES`
constraint:

```sql
professional_crm_contact_id UUID,
-- FK to Extension 5's professional_contacts table (not enforced by DB, managed by application)
```

The comment in the schema file makes the intent explicit. This is a deliberate architectural
decision, not an oversight.

### 8.1 Why Not a Hard FK?

A hard FK — `professional_crm_contact_id UUID REFERENCES professional_contacts(id)` —
would require that `professional_contacts` exists in the same database at schema creation
time. In the Open Brain setup, both tables live in the same Supabase project, so this is
technically feasible. The reasons for omitting it are:

**Setup order independence.** Extension 6 can be set up before Extension 5, or without
Extension 5 at all. If `professional_contacts` does not exist, `schema.sql` still runs
cleanly. A hard FK would cause the `job_contacts` table creation to fail with a
`relation "professional_contacts" does not exist` error.

**Cascading deletes across extension boundaries.** If `professional_contacts` had a
cascade-on-delete relationship to `job_contacts`, deleting a professional contact would
nullify or delete the job contact record. Cross-extension cascade behavior is difficult
to reason about and can cause unexpected data loss. Managing the relationship at the
application layer keeps the side effects visible in code.

**Optional dependency.** The entire `link_contact_to_professional_crm` tool is optional
functionality. 7 of the 8 tools work with no dependency on Extension 5. Making the FK
hard would force Extension 5 to be present even for users who never intend to link contacts.

### 8.2 How the Soft FK Is Enforced

The application layer provides all referential integrity:

1. The `link_contact_to_professional_crm` tool only writes a valid UUID (the one returned
   by the `professional_contacts` insert) to `professional_crm_contact_id`.
2. The idempotency check in Step 2 prevents the column from being overwritten.
3. No other tool in Extension 6 modifies `professional_crm_contact_id`.

The tradeoff: if a professional contact is deleted from Extension 5, the stale UUID in
`job_contacts.professional_crm_contact_id` is not cleaned up. Any code that follows the
soft FK to Extension 5 must handle a not-found result gracefully.

---

## 9. Indexes — Including the Partial Index

The schema defines 7 indexes. One of them is a partial index — a pattern not used in any
earlier extension.

```sql
CREATE INDEX IF NOT EXISTS idx_companies_user_id
    ON companies(user_id);

CREATE INDEX IF NOT EXISTS idx_job_postings_company_id
    ON job_postings(company_id);

CREATE INDEX IF NOT EXISTS idx_applications_user_status
    ON applications(user_id, status);

CREATE INDEX IF NOT EXISTS idx_applications_job_posting
    ON applications(job_posting_id);

CREATE INDEX IF NOT EXISTS idx_interviews_application_scheduled
    ON interviews(application_id, scheduled_at);

CREATE INDEX IF NOT EXISTS idx_interviews_user_scheduled
    ON interviews(user_id, scheduled_at)
    WHERE scheduled_at IS NOT NULL;

CREATE INDEX IF NOT EXISTS idx_job_contacts_user_company
    ON job_contacts(user_id, company_id);
```

### 9.1 Index-to-Query Mapping

| Index | Supports |
|---|---|
| `idx_companies_user_id` | All company lookups by user |
| `idx_job_postings_company_id` | Finding all postings for a company |
| `idx_applications_user_status` | `get_pipeline_overview` status count; filter by user+status |
| `idx_applications_job_posting` | Finding all applications for a posting |
| `idx_interviews_application_scheduled` | Finding interviews for a specific application, ordered by time |
| `idx_interviews_user_scheduled` | `get_upcoming_interviews` and `get_pipeline_overview` interview queries |
| `idx_job_contacts_user_company` | Finding contacts by user and company |

### 9.2 The Partial Index

`idx_interviews_user_scheduled` is a partial index:

```sql
CREATE INDEX IF NOT EXISTS idx_interviews_user_scheduled
    ON interviews(user_id, scheduled_at)
    WHERE scheduled_at IS NOT NULL;
```

A partial index only indexes rows that satisfy the `WHERE` condition. Rows where
`scheduled_at IS NULL` are excluded from the index entirely.

`scheduled_at` is nullable in the `interviews` table — interviews can be created without
a confirmed time. The upcoming-interview queries filter on `scheduled_at`:

```typescript
.gte("scheduled_at", new Date().toISOString())
.lte("scheduled_at", futureDate.toISOString())
```

A query filtering on `scheduled_at` with a range condition will never match rows where
`scheduled_at IS NULL` — PostgreSQL does not compare NULL values with `>=` or `<=`.
Including NULL rows in the index would waste index space and maintenance overhead for rows
that can never be matched by this query pattern.

The partial index is smaller, faster to maintain on write, and more cache-efficient than
a full index on the same columns. This is the canonical use case for partial indexes:
when a significant fraction of rows have a NULL (or otherwise excluded) value for an
indexed column and the query patterns never need to match those rows.

---

## 10. Row Level Security and Triggers

### 10.1 RLS Policies

All five tables have RLS enabled with symmetric user-scoped policies:

```sql
ALTER TABLE companies ENABLE ROW LEVEL SECURITY;
-- (repeated for all 5 tables)

CREATE POLICY companies_user_policy ON companies
    FOR ALL
    USING (auth.uid() = user_id)
    WITH CHECK (auth.uid() = user_id);
```

`FOR ALL` covers SELECT, INSERT, UPDATE, and DELETE in a single policy. `USING` applies
to reads; `WITH CHECK` applies to writes. Setting both to the same condition ensures a
user can only read their own rows and can only write rows with their own `user_id`.

This is the same RLS pattern used in earlier extensions. The addition of 5 tables with
RLS makes Extension 6 the schema with the most RLS surface area in the set.

The `job_contacts` table has the same policy as the others. Note that `company_id` in
`job_contacts` is not scoped by RLS — if a contact has a `company_id`, that reference
is only valid if the company row also belongs to the same user. Cross-user company
references are not possible given how companies are inserted, but the RLS policies do
not explicitly prevent them.

### 10.2 Triggers

Two tables have `updated_at` triggers: `companies` and `applications`.

```sql
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

DROP TRIGGER IF EXISTS update_companies_updated_at ON companies;
CREATE TRIGGER update_companies_updated_at
    BEFORE UPDATE ON companies
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

DROP TRIGGER IF EXISTS update_applications_updated_at ON applications;
CREATE TRIGGER update_applications_updated_at
    BEFORE UPDATE ON applications
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

The `DROP TRIGGER IF EXISTS` before each `CREATE TRIGGER` makes the schema idempotent —
running `schema.sql` a second time will not fail with "trigger already exists."

`job_postings` and `interviews` do not have `updated_at` triggers. This is a consistent
design choice: job postings are treated as immutable records, and interviews only have
one meaningful update (logging notes after completion), where `created_at` plus the
`status` field suffice for audit purposes.

---

## 11. How This Server Fits with Other Extensions

Extension 6 sits at the end of the learning path and draws on patterns from all five
preceding extensions.

**Extension 1 (Household Knowledge Base)** establishes the single-table insert-and-query
pattern. Every `add_*` tool in Extension 6 follows the same structure: validate inputs,
insert with `|| null` coercion, return the created row.

**Extension 3 (Family Calendar)** introduces date-range querying with time windows —
the same pattern used by `get_upcoming_interviews`. The `futureDate.setDate(futureDate.getDate() + daysToCheck)` window calculation is identical to Extension 3's week-schedule logic.

**Extension 4 (Meal Planning)** introduces multi-table schemas with parent-child FK
relationships. Extension 6's 4-table chain (companies → postings → applications →
interviews) is the deepest such chain in the set.

**Extension 5 (Professional CRM)** is both a predecessor and a dependency. The CRM
introduces the contacts-and-interactions model. The `link_contact_to_professional_crm`
tool makes Extension 6 the first extension to write back into another extension's data
at runtime.

**The Job Hunt extension is the only extension in the set that writes to another
extension's tables.** All other cross-extension interactions are reads. Understanding
the implications of that write — the soft FK, the idempotency guard, the partial atomicity
risk — is the capstone concept of the Extension 6 learning path.

---

## 12. Source File Reference

| File | Lines | Purpose |
|---|---|---|
| `extensions/job-hunt/index.ts` | 697 | MCP server — all tool handlers and server wiring |
| `extensions/job-hunt/schema.sql` | 175 | Five tables, seven indexes, RLS policies, triggers |
| `extensions/job-hunt/package.json` | — | Node.js dependencies |
| `extensions/job-hunt/tsconfig.json` | — | TypeScript compiler configuration |
| `extensions/job-hunt/metadata.json` | — | Extension metadata for the repo registry |
| `extensions/job-hunt/README.md` | — | Setup guide and usage instructions |

### Key line references in `index.ts`

| Lines | What happens |
|---|---|
| 1–43 | Environment validation and Supabase client initialization |
| 46–124 | TypeScript interface definitions for all 5 table row types |
| 127–300 | `TOOLS` array — all 8 tool definitions with input schemas |
| 303–331 | `handleAddCompany` |
| 333–368 | `handleAddJobPosting` |
| 370–400 | `handleSubmitApplication` |
| 402–433 | `handleScheduleInterview` |
| 435–459 | `handleLogInterviewNotes` — single UPDATE setting feedback, rating, and status |
| 461–513 | `handleGetPipelineOverview` — reduce() aggregation + 3-level nested select |
| 515–549 | `handleGetUpcomingInterviews` — 3-level nested select |
| 551–633 | `handleLinkContactToProfessionalCRM` — cross-extension write with idempotency guard |
| 635–684 | Server registration, `ListToolsRequestSchema` handler, `CallToolRequestSchema` dispatch |
| 686–696 | `main()` function and process error handler |

### Key line references in `schema.sql`

| Lines | What happens |
|---|---|
| 6–19 | `companies` table with size and remote_policy enums, DECIMAL glassdoor_rating |
| 23–39 | `job_postings` table with TEXT[] arrays and source enum |
| 43–56 | `applications` table with 8-state status pipeline |
| 60–74 | `interviews` table with 7 interview types and 1–5 rating |
| 78–92 | `job_contacts` table with soft FK on line 88 |
| 95–115 | Seven indexes including the partial index at line 111 |
| 118–148 | RLS enables and policies for all 5 tables |
| 151–170 | `update_updated_at_column` function and triggers for companies and applications |
