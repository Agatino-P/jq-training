# jq Training Plan

> Teacher-led course to learn the parts of jq that actually matter for daily use.
> Manual reference: https://jqlang.org/manual/ (we're on **jq 1.8.1**, matches the 1.8 manual)
> Rule of the course: learn the ~20% that covers ~80% of real usage *well*. Skip nitty-gritty; note it and move on.

---

## How we use this doc

- Each module has a **status**, a short **goal**, the **manual section** it maps to, and **exercises**.
- **Always pause for questions before handing over exercises.** Teach → ask "any questions?" → only then give exercises.
- **After any divergence (a question or side-thread), once it's sorted, re-present the whole thing in full** — the lesson recap, question prompt, or exercises — so nothing has to be scrolled back for. Never resume with just a pointer like "back to where we were."
- **Baby steps: whenever a module splits sensibly, split it.** Teach one small sub-topic at a time (question → mini-exercise → next), not the whole module in one dump. Smaller is always better.
- Status legend: `[ ]` not started · `[~]` in progress · `[x]` done · `[!]` needs review
- After each session I (Claude) update statuses and drop gotchas into **`cheatsheet.md`** ("Things to keep").
- When something is out of scope but worth remembering, it goes in **Parked (revisit later)** — we don't rabbit-hole.
- Exercises live in `exercises/` and sample data in `data/`. Scratch answers go in `scratch/`.
- **`lessons.md`** = condensed study notes, one section per module (distilled teaching, not the chat). Update it as each module finishes.
- **`cheatsheet.md`** = "Things to keep": gotchas, one-liners, aha moments. Update it as we hit them.

---

## The mental model (read this first, we'll keep coming back to it)

Three ideas explain almost all of jq:

1. **A jq program is a filter**: JSON in → zero or more JSON values out.
2. **Everything is a stream.** A filter doesn't return "a value" — it produces a *stream* of values (0, 1, or many). `.[]` and `,` create streams; most confusion comes from forgetting this.
3. **`|` pipes each value of the left stream into the right filter**, one at a time — exactly like a shell pipe.

Keep asking: *"how many values is this producing?"*

---

## Modules

### Module 0 — Setup & how to run jq  `[x]`
- **Goal:** run jq confidently from the terminal; know the 5 flags that matter.
- **Manual:** Invoking jq
- **Must-know flags:** `-r` (raw output), `-c` (compact), `-s` (slurp), `-n` (null input), `--arg`/`--argjson` (pass values in).
- **Exercise:** pipe a JSON file through `jq '.'`; try `-r` vs without on a string result.

### Module 1 — Core filters: the backbone  `[x]`  ⭐ core
- **Goal:** identity, indexing, pipe, comma, parentheses — the grammar of everything else.
- **Manual:** Basic filters
- **Cover:** `.`, `.foo`, `.foo.bar`, `.foo?`, `.["key"]`, `.[0]`, `.[2:4]` (slices), `.[]` (iterate), `,` (multiple outputs), `|` (pipe), `( )` grouping.
- **Exercise:** from a sample object, pull two fields at once with `,`; chain `.a.b.c`; slice an array.

### Module 2 — Iteration & the stream model  `[x]`  ⭐ core
- **Goal:** truly *get* `.[]` and how many outputs a filter makes. This is the make-or-break concept.
- **Manual:** Basic filters (Value Iterator) + Types
- **Cover:** `.[]` over arrays vs objects; how `.[] | ...` maps over a stream; `[ ... ]` to re-collect a stream into an array.
- **Exercise:** iterate an array of objects, extract one field from each; then wrap it back into an array.

### Module 3 — Building output  `[x]`  ⭐ core
- **Goal:** construct new JSON shapes, not just extract.
- **Manual:** Types and Values (Array/Object construction) + String interpolation
- **Cover:** `[ ... ]`, `{ ... }`, shorthand `{name, age}`, computed keys `{(.k): .v}`, `"\(.expr)"` interpolation.
- **Exercise:** reshape `{first,last,age}` into `{name:"first last", age}`.

### Module 4 — select / map / map_values  `[ ]`  ⭐ core
- **Goal:** filter and transform collections — the daily-driver trio.
- **Manual:** Builtin functions
- **Cover:** `select(cond)`, `map(f)`, `map_values(f)`, and why `map(f)` == `[.[] | f]`.
- **Exercise:** keep only objects where `.active == true`; double every number in an array.

### Module 5 — Essential builtins  `[ ]`  ⭐ core
- **Goal:** the ~15 builtins you'll actually reach for.
- **Manual:** Builtin functions
- **Cover:** `length`, `keys`, `values`, `has`, `in`, `to_entries`/`from_entries`/`with_entries`, `add`, `group_by`, `unique`/`unique_by`, `sort`/`sort_by`, `min_by`/`max_by`, `flatten`, `range`.
- **Exercise:** group an array of records by a field; sum a field with `map(.x) | add`.

### Module 6 — Conditionals, comparisons & error handling  `[ ]`  ⭐ core
- **Goal:** branching and the operators that make jq resilient to messy data.
- **Manual:** Conditionals and Comparisons
- **Cover:** `==` `!=` `<` `>`, `and`/`or`/`not`, `if a then b else c end` (and `elif`), the **alternative operator `//`** (default values), `try ... catch`, trailing `?`.
- **Exercise:** default a missing field to `"unknown"` with `//`; guard a risky index with `?`.

### Module 7 — reduce & foreach  `[ ]`
- **Goal:** fold a stream into a single accumulated value (sums, counts, custom aggregations).
- **Manual:** Advanced features
- **Cover:** `reduce STREAM as $x (init; update)`; a taste of `foreach` for running/intermediate state.
- **Exercise:** count occurrences of each value into an object with `reduce`.

### Module 8 — Assignment & update-in-place  `[ ]`
- **Goal:** modify JSON at a path — jq's "editing" superpower.
- **Manual:** Assignment
- **Cover:** `.a = v`, `.a |= f` (update with function), `+=` etc., path expressions, `del(.a)`, `.a.b[].c |= ...`.
- **Exercise:** uppercase every `.name` in an array; delete a field; increment a counter.

### Module 9 — Strings & regex (practical subset)  `[ ]`
- **Goal:** the string ops you actually need; a working slice of regex.
- **Manual:** String functions + Regular expressions
- **Cover:** `split`, `join`, `ltrimstr`/`rtrimstr`, `ascii_downcase`/`upcase`, `test`, `match`, `capture` (named groups), `sub`/`gsub`.
- **Exercise:** split a CSV-ish line; extract a date with a named-capture regex.

### Module 10 — Variables, functions & recursion  `[ ]`
- **Goal:** bind values, define reusable filters, walk nested structures.
- **Manual:** Advanced features + Recursive descent
- **Cover:** `EXPR as $x | ...`, `{$x}` binding shorthand, `def f: ...;` and `def f(a): ...;`, `..` (recursive descent), `recurse`, `getpath`/`paths`, `walk`.
- **Exercise:** define a `def average: add/length;`; find all values of a key at any depth with `..`.

### Module 11 — I/O & the CLI flags that matter  `[ ]`  ⭐ core
- **Goal:** wire jq into real pipelines: multiple inputs, slurping, passing shell vars, NDJSON.
- **Manual:** I/O + Invoking jq
- **Cover:** `-s` slurp, `-n` + `inputs`, `--arg`/`--argjson`, `$ENV`/`env`, `-r`/`-j`, handling NDJSON (stream of JSON lines).
- **Exercise:** merge an NDJSON stream into one array; inject a shell variable with `--arg`.

### Module 12 — Real-world recipes & capstone  `[ ]`
- **Goal:** stitch it all together on realistic data (API responses, logs, config).
- **Cover:** filter→reshape→aggregate→sort→top-N; group-and-count; flatten nested API payloads.
- **Exercise (capstone):** given a sample API dump, produce a leaderboard-style summary in one pipeline.

---

## Parked (revisit later — deliberately skipping for now)

- Streaming parser (`--stream`, `tostream`/`fromstream`) — only for giant inputs.
- Modules / `import` / `include` — for reusable libraries.
- SQL-style operators (`INDEX`, `GROUP_BY`, `IN`) — sugar over things we'll learn directly.
- `$__loc__`, `input_line_number`, `@base64d`/other `@` formats beyond `@csv`/`@tsv`/`@json`.
- Colors / `--tab` / output cosmetics.
- Math builtins beyond the obvious (full list of `sqrt`, `pow`, etc.).
- SQL date/time deep dive (`strftime`, `mktime`, timezones) — learn on demand.

---

## Progress log

| Date | Module(s) | What we did | Next up |
|------|-----------|-------------|---------|
| 2026-07-17 | setup | Created plan, folder, verified jq 1.8.1 | Module 0/1 |
| 2026-07-27 | 0 | Ran jq, 5 flags, `-r`/`-c`/`-s`/`-n`; nailed identity vs slurp & NDJSON-as-stream | Module 1 |
| 2026-07-27 | 1 | Core filters: `.`, indexing, brackets, slices, `.[]`, `,`, `|`, `()`, `?`; got `.[]` (N outputs) vs slice (1 array) | Module 2 |
| 2026-07-27 | 2 | Stream model (2a explode `.[]`, 2b pipe = per-value, 2c `[ ]` collect N→1); explode→transform→collect idiom | Module 3 |
| 2026-07-27 | 3 | Building output (3a `{}` + shorthand, 3b `"\(.expr)"` interp, 3c `[]` + computed keys `{(.k):.v}`) | Module 4 |
