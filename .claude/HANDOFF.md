# Session handoff — jq training

**Role:** You are the user's jq teacher. Course lives in [`../jq-training-plan.md`](../jq-training-plan.md) — source of truth for curriculum + progress + **course rules** (read "How we use this doc" first and follow every rule there).

## Where things are
- `jq-training-plan.md` — 13 modules (0–12), statuses, course rules, progress log.
- `lessons.md` — condensed study notes, one section per module; starts with "The big picture" pipeline diagram. Update as each module finishes.
- `cheatsheet.md` — "Things to keep": gotchas, one-liners.
- `data/people.json`, `data/events.ndjson` — sample data. `exercises/` unused so far; `scratch/` gitignored.

## Environment
- jq **1.8.1** (matches jq 1.8 manual: https://jqlang.org/manual/).
- Repo: https://github.com/Agatino-P/jq-training (`origin`, branch `main`). Push works via `gh` as `AP-Datamars`; just `git push`.

## How the course runs (rules live in the plan — key ones)
- Baby steps: split modules into sub-topics (5a, 5b, …). Teach one bite → **pause for questions** → mini-exercise → next bite.
- After any divergence, re-present the current bite/exercise **in full**.
- Lessons/cheatsheet format: syntax line → concrete self-contained example line (inline input → output; never reference `data/` files) → prose. Never use untaught builtins without a gloss + forward pointer.
- Update lessons.md + plan statuses + progress log per module; commit + push at module end.

## Status
- Modules **0–5 done** (setup, core filters, stream model, building output, select/map/map_values, essential builtins).
- **Next up: Module 6 — conditionals, comparisons & error handling** (`if/elif/else`, `and/or/not`, `//` defaults, `try/catch`, `?`). Already previewed: `if`-without-`else` = identity (cheatsheet), `?` suffix (Module 1).
- Agreed: when reaching Modules 7 and 10, compress them to their useful 20% (user signed off).
- User is sharp — caught a real error (`if C then T end` defaults to `else .`, NOT empty). Both note files carry the correction.
