# CLAUDE.md — Agent Instructions for Open Brain

This file helps AI coding tools (Claude Code, Codex, Cursor, etc.) work effectively in this repo.

## What This Repo Is

Open Brain is a persistent AI memory system — one database (Supabase + pgvector), one MCP protocol, any AI client. This repo contains the extensions, recipes, schemas, dashboards, and integrations that the community builds on top of the core Open Brain setup.

**License:** FSL-1.1-MIT. No commercial derivative works. Keep this in mind when generating code or suggesting dependencies.

## Repo Structure

```
extensions/     — Curated, ordered learning path (6 builds). Do NOT add without maintainer approval.
primitives/     — Reusable concept guides (must be referenced by 2+ extensions). Curated.
recipes/        — Standalone capability builds. Open for community contributions.
schemas/        — Database table extensions. Open.
dashboards/     — Frontend templates (Vercel/Netlify). Open.
integrations/   — MCP extensions, webhooks, capture sources. Open.
docs/           — Setup guides, FAQ, companion prompts.
resources/      — Claude Skill, companion files, ob CLI tool.
```

Every contribution lives in its own subfolder under the right category and must include `README.md` + `metadata.json`.

## Guard Rails

- **Never modify the core `thoughts` table structure.** Adding columns is fine; altering or dropping existing ones is not.
- **No credentials, API keys, or secrets in any file.** Use environment variables.
- **No binary blobs** over 1MB. No `.exe`, `.dmg`, `.zip`, `.tar.gz`.
- **No `DROP TABLE`, `DROP DATABASE`, `TRUNCATE`, or unqualified `DELETE FROM`** in SQL files.

## PR Standards

- **Title format:** `[category] Short description` (e.g., `[recipes] Email history import via Gmail API`)
- **Branch convention:** `contrib/<github-username>/<short-description>`
- **Commit prefixes:** `[category]` matching the contribution type
- Every PR must pass the 11 automated review checks in `.github/workflows/ob1-review.yml` before human review
- See `CONTRIBUTING.md` for the full review process, metadata.json template, and README requirements

## Key Files

- `CONTRIBUTING.md` — Source of truth for contribution rules, metadata format, and the review process
- `.github/workflows/ob1-review.yml` — Automated PR review (11 checks)
- `.github/metadata.schema.json` — JSON schema for metadata.json validation
- `.github/PULL_REQUEST_TEMPLATE.md` — PR description template
- `LICENSE.md` — FSL-1.1-MIT terms

## Documentation

- `docs/QUICKSTART.md` — 10-step guided tutorial for new users (first captures, searches, quick wins)
- `docs/USER_GUIDE.md` — Complete user guide with 10 use cases, extension overviews, and troubleshooting
- `docs/ARCHITECTURE.md` — System architecture with ASCII diagrams at multiple abstraction levels
- `docs/DEVELOPER_GUIDE.md` — Comprehensive developer guide (builds extensions, recipes, integrations; written for C/Java developers new to web dev)
- `docs/API_REFERENCE.md` — All 35+ MCP tools, database schemas, authentication, and environment variables
- `docs/STUDY_PLAN.md` — Zero-to-hero learning plan in 13 phases covering web fundamentals through capstone projects
- `docs/01-getting-started.md` — Full system setup guide (45 minutes, no coding experience needed)
- `docs/02-companion-prompts.md` — Five prompts for memory migration, use case discovery, and weekly review
- `docs/03-faq.md` — Common issues, architecture questions, and community tips
- `docs/04-ai-assisted-setup.md` — Build with Cursor, Claude Code, or other AI coding tools
- `docs/CLI_DIRECT_APPROACH.md` — CLI-Direct approach: use Open Brain from terminal AI tools without MCP
