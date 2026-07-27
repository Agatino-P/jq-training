# Session handoff — jq training

**Role:** You are the user's jq teacher. Course lives in [`../jq-training-plan.md`](../jq-training-plan.md) — that file is the source of truth for curriculum + progress. Read it first.

## Where things are
- `jq-training-plan.md` — 13 modules (0–12), statuses, "Things to keep" cheat sheet, progress log.
- `data/people.json` — array of records for exercises.
- `data/events.ndjson` — JSON-lines stream (for slurp / `-n`+`inputs` practice).
- `exercises/` — exercise files. `scratch/` — user's working files (gitignored).

## Environment
- jq **1.8.1** installed (matches the jq 1.8 manual: https://jqlang.org/manual/).
- Repo: https://github.com/Agatino-P/jq-training (remote `origin`, branch `main`).
- Push works via `gh` authed as `AP-Datamars` (has push access to the `Agatino-P` repo). Just `git push`.

## How the course runs
- Teach a module in chat → user does exercises against sample data → update statuses in the plan → drop gotchas into "Things to keep" → commit + push.

## Status
- Setup done (plan, data, folders, repo, first push). No module taught yet.
- **Next up:** Module 1 — core filters (`.`, indexing, `|`, `,`, `()`), then the stream mental model in Module 2.
