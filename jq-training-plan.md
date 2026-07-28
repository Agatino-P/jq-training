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
- **Note format (lessons + cheatsheet): every concept starts with a syntax line, then an example line**, then prose. E.g. `collection | .[] → stream` / `["a","b"] | .[] → "a" "b"`.
- **Examples in lessons/cheatsheet must be self-contained**: concrete inline input → output. Never reference `data/` files or placeholders like `person |` — the reader may not have the repo's data.
- **Never use an untaught builtin silently in an example.** Either use only already-covered material, or gloss it inline with a forward pointer, e.g. `(ascii_upcase = uppercase; Module 9)`.
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

The master diagram (full version at the top of `lessons.md`):

```
input values ──→ explode? (.[]) ──→ per-element filters ──→ collect? ([ ]) ──→ output
```

> Get values as a stream; work per element; drop to array-land only when an operation
> needs the whole collection; collect at the end only if the output should be one value.

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

### Module 4 — select / map / map_values  `[x]`  ⭐ core
- **Goal:** filter and transform collections — the daily-driver trio.
- **Manual:** Builtin functions
- **Cover:** `select(cond)`, `map(f)`, `map_values(f)`, and why `map(f)` == `[.[] | f]`.
- **Exercise:** keep only objects where `.active == true`; double every number in an array.

### Module 5 — Essential builtins  `[x]`  ⭐ core
- **Goal:** the ~15 builtins you'll actually reach for.
- **Manual:** Builtin functions
- **Cover:** `length`, `keys`, `values`, `has`, `in`, `to_entries`/`from_entries`/`with_entries`, `add`, `group_by`, `unique`/`unique_by`, `sort`/`sort_by`, `min_by`/`max_by`, `flatten`, `range`.
- **Exercise:** group an array of records by a field; sum a field with `map(.x) | add`.

### Module 6 — Conditionals, comparisons & error handling  `[x]`  ⭐ core
- **Goal:** branching and the operators that make jq resilient to messy data.
- **Manual:** Conditionals and Comparisons
- **Cover:** `==` `!=` `<` `>`, `and`/`or`/`not`, `if a then b else c end` (and `elif`), the **alternative operator `//`** (default values), `try ... catch`, trailing `?`.
- **Exercise:** default a missing field to `"unknown"` with `//`; guard a risky index with `?`.

### Module 7 — reduce & foreach  `[parked]`
- Cut 2026-07-28: user's real use (filter/reshape/grep kubectl output) is covered by `group_by`/`add`/`length`. Revisit only if a custom aggregation ever comes up. Syntax reminder: `reduce .[] as $x (init; update)`.

### Module 8 — Assignment & update-in-place  `[x]` (8a only; 8b parked)
- **Done:** `.a = v`, `.a |= f`, `+=`/`//=` sugar, `=` vs `==`, `=` (RHS sees whole doc) vs `|=` (RHS sees old value).
- **Parked** (deep-path editing — not needed for read-only kubectl use): `.a.b[].c |= f`, `select`-in-path conditional updates, `del(.a)`.

### Module 9 — Strings & regex, kubectl-lite  `[x]`
- **Goal:** the "grep verb" for JSON: regex filtering + the string ops that show up in pipelines.
- **Manual:** String functions + Regular expressions
- **Cover (one bite):** `test("re")` in `select`, `split`/`join`, `startswith`/`endswith`; mention `capture` exists.
- **Cut:** `match` internals, `sub`/`gsub`, trimming functions — learn on demand.

### Module 10 — Variables, functions & recursion  `[parked]`
- Cut 2026-07-28: not needed for read-only querying. `..` (recursive descent) gets a cheatsheet entry; `as $x` was previewed in Module 7's syntax. `def` et al: revisit if reusable filters ever matter.

### Module 11 — I/O & the CLI flags that matter  `[ ]`  ⭐ core
- **Goal:** wire jq into real pipelines: multiple inputs, slurping, passing shell vars, NDJSON.
- **Manual:** I/O + Invoking jq
- **Cover:** `-s` slurp, `-n` + `inputs`, `--arg`/`--argjson`, `$ENV`/`env`, `-r`/`-j`, handling NDJSON (stream of JSON lines).
- **Exercise:** merge an NDJSON stream into one array; inject a shell variable with `--arg`.

### Module 12 — kubectl capstone  `[ ]`
- **Goal:** stitch it together on the user's real terrain.
- **Exercise (capstone):** given a `kubectl get pods -o json`-style dump: filter by status/label with `select`+`test`, extract fields, group-and-count, top-N — one pipeline.

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
| 2026-07-27 | 4 | select/map/map_values (4a gate+stream-filter, 4b `map`==`[.[]|f]`, 4c values-in-place); N-vs-1 choice | Module 5 |
| 2026-07-28 | 5 | Builtins: 5a length/keys/has, 5b add/flatten/range, 5c sort_by family, 5d group_by+count recipe, 5e entries family; big-picture pipeline diagram added | Module 6 |
| 2026-07-28 | 6 | 6a comparisons/truthiness, 6b and/or/not, 6c if/elif, 6d `//` (+boolean trap), 6e `?`/try-catch. All core modules done | Module 7 (compressed) |
| 2026-07-28 | 8a + trim | Assignment basics (`=`, `|=`, `+=`). Then trimmed course to real use (kubectl querying): parked 7, 8b, 10; 9 → one regex/strings bite; 12 → kubectl capstone | Module 9-lite |
| 2026-07-28 | 9-lite | test-in-select grep move, split/join, startswith/endswith (preferred over regex for prefixes); capture parked | Module 11 |
