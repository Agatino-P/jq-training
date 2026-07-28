# Session handoff — jq training

**This file is self-sufficient — everything needed to continue is in this repo.** On a new PC: clone https://github.com/Agatino-P/jq-training, ensure jq ≥ 1.8 is installed (`jq --version`), read this file, go.

**Role:** You are the user's jq teacher. Course lives in [`../jq-training-plan.md`](../jq-training-plan.md) — source of truth for curriculum + progress + **course rules** (read "How we use this doc" first and follow every rule there).

## Where things are
- `jq-training-plan.md` — 13 modules (0–12), statuses, course rules, progress log.
- `lessons.md` — condensed study notes, one section per module; starts with "The big picture" pipeline diagram. Update as each module finishes.
- `cheatsheet.md` — "Things to keep": gotchas, one-liners.
- `data/people.json`, `data/events.ndjson` — sample data. `exercises/` unused so far; `scratch/` gitignored.

## Environment
- jq **1.8.1** (matches jq 1.8 manual: https://jqlang.org/manual/). Any 1.8+ works.
- Repo: https://github.com/Agatino-P/jq-training (`origin`, branch `main`). On the original Mac, push works via `gh` authed as `AP-Datamars`; on another PC verify push access before promising commits.

## How the course runs (rules live in the plan — key ones)
- Baby steps: split modules into sub-topics (5a, 5b, …). Teach one bite → **pause for questions** → mini-exercise → next bite.
- After any divergence, re-present the current bite/exercise **in full**.
- Lessons/cheatsheet format: syntax line → concrete self-contained example line (inline input → output; never reference `data/` files) → prose. Never use untaught builtins without a gloss + forward pointer.
- Update lessons.md + plan statuses + progress log per module; commit + push at module end.

## Status
- Modules **0–6 done**, plus **8a** (assignment basics).
- **Course trimmed 2026-07-28 to the user's real use** — filter/reshape/grep kubectl JSON output, read-only. Parked: 7 (reduce), 8b (deep-path editing), 10 (functions/recursion; `..` is on the cheatsheet). See plan for details.
- **9-lite done.** Remaining (one short session): **Module 11** (I/O & CLI flags ⭐: `--arg`, `-n inputs`, NDJSON, `-r`) → **Module 12** (kubectl-style capstone).
- User is sharp — caught a real error (`if C then T end` defaults to `else .`, NOT empty). Both note files carry the correction.
