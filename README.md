# Sanitized Design Documentation — Backlink Tracking & On-Call Roster Platform

Architecture, data-model and algorithm documentation for a production internal platform, rewritten so it can be
shared outside the company. All diagrams are [Mermaid](https://mermaid.js.org/) and render natively on GitHub,
GitLab, VS Code (Markdown Preview Mermaid Support) and most Markdown viewers.

## What the system is

One TypeScript monorepo, one Next.js app, one Supabase (Postgres) project, two business modules:

| Module | Purpose | Data direction |
|---|---|---|
| **Backlink Tracking** | Reads hand-maintained Google Sheets as-is on a daily schedule, validates them, computes dashboard metrics and stores them as precomputed snapshots. | Sheets → DB → read-only UI |
| **On-Call Roster ("duty")** | Real-time 24/7 shift roster for a three-region operations team: shift swaps, live checklist checks, incident log, audit log. | Users write through guarded DB functions, with realtime refresh |

## Documents

| # | Document | Covers |
|---|---|---|
| 1 | [System Architecture](01-system-architecture.md) | 4-package monorepo, runtime topology, trust boundaries, write paths, key design decisions and rejected alternatives |
| 2 | [ERD — Data Model](02-erd.md) | All 23 Postgres tables as three ER diagrams, table dictionary, schema-level invariants |
| 3 | [Sequence Diagrams](03-sequence-diagrams.md) | Hardest flow (shift swap: idempotency + in-transaction server clock + row locks), scheduled sync (pg_cron dual-redundancy), realtime refresh, fail-closed log archiving |
| 4 | [Code Snippets & Pseudocode](04-code-snippets.md) | Header-anchor spreadsheet parser, default-deny RLS, idempotent RPC contract, lock-window decision, server clock estimator, rotation algorithm, sync mutex, fail-closed archive |

## Verified facts (counted from the source tree)

| Item | Value |
|---|---|
| Workspace packages | 4 (`core`, `sync`, `duty`, `web`) |
| Postgres tables | 23 (12 backlink + 1 shared access table + 10 on-call) |
| SQL migrations | 29 |
| Test files | 282 (`*.test.ts` / `*.test.tsx`) |
| Scheduled pipelines | 2 GitHub Actions workflows (sync, maintenance) + 1 `pg_cron` job |

## Sanitization policy

What was removed or generalized, so a reader can judge what is real and what is a placeholder:

| Original kind | Replaced by |
|---|---|
| Company, team and product names, repository URL | "the company", "the operations team", generic module names |
| Real site/domain names, competitor domains | `site-a.example`, `site-b.example`, `competitor.example` |
| Employee names, e-mail addresses, account IDs | `staff_1 … staff_5`, `user@company.example` |
| Google Sheet / file IDs, Supabase project ref, tokens, keys | `<SHEET_ID>`, `<PROJECT_REF>`, `<GITHUB_TOKEN>` (never real values) |
| Business vertical and region names | "live-broadcast operations", regions `R1 / R2 / R3` |
| Real business thresholds (daily targets, calendar dates) | Illustrative values, marked as examples |
| Non-English identifiers, messages and comments | Translated to English |

Not changed: table/column names that are generic (`site`, `output`, `duty_shift`, …), the algorithms, the
security model, and the structure of the schema. The lock window (`15:30–19:00`) and the UTC+7 timezone appear as
**example configuration values** — in the real system they live in one config row, never in code.

No document contains source files verbatim: snippets are rewritten as standalone illustrations of the technique.
