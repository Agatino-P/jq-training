# jq Lessons — condensed study notes

> Distilled teaching, one section per module. Progress + exercises live in `jq-training-plan.md`.
> jq 1.8.1 · manual: https://jqlang.org/manual/

---

## Module 0 — Running jq & the 5 flags

Shape: `jq 'PROGRAM' file.json` or `cat file.json | jq 'PROGRAM'`. The program is the *filter*.

The 5 flags that matter:

| Flag | Does | When |
|------|------|------|
| `-r` | raw output — strings without quotes | feeding a human or another CLI tool |
| `-c` | compact — one line per output, no pretty-print | NDJSON output, logs |
| `-s` | slurp — read whole input stream into one array | gather many values / NDJSON → array |
| `-n` | null input — start from `null`, don't read stdin | generating data, `-n 'inputs'` |
| `--arg N V` / `--argjson N V` | inject `$N` as string / parsed JSON | passing shell values in |

Key distinctions:
- **`.` is identity** — returns input unchanged (just pretty-prints). It does not "make" an array.
- **`-s` (slurp)** = take the whole input *stream* and wrap all top-level values in one array.
  Stream of 4 → `[a,b,c,d]`. A single array input → `[thatArray]` (double-wrapped!).
  It's about how many values the *input stream* has, not about the contents.
- **NDJSON** = one JSON value per line, no wrapping array. jq streams these for free (no flag).
  `people.json` = 1 value (an array). `events.ndjson` = 4 values (a stream).
- **`-r`** strips the quotes off string output.

---

## Module 1 — Core filters: the grammar of everything

Six-ish pieces; everything later is combinations of these.

- **`.` identity** — input out unchanged; the starting point for paths.
- **`.foo` / `.foo.bar` object indexing** — reach into an object by key; chains left→right, each step operates on the previous step's output.
- **`.["key"]` bracket form** — `.foo` is sugar for `.["foo"]`. Required when the key isn't a plain identifier (space, dash, leading digit): `.["odd-key"]`.
- **`.[0]` / `.[-1]` / `.[2:4]` array indexing & slices** — `.[-1]` = last. Slices are half-open (like Python) and return a *new array*.
- **`.[]` value iterator** ⭐ — does NOT return an array; it **explodes** a collection into a *stream* of its values. `.[].first` → one output per element. Contrast `.[0].first` = 1 output.
- **`,` comma** — run several filters on the *same* input, outputs side by side: `.first, .last` → two outputs.
- **`|` pipe** — feed each value of the left stream into the right filter, one at a time (like a shell pipe). `.[] | .first` == `.[].first`.
- **`( )` grouping** — force precedence, like math: `(.[0].age) + 1`.
- **`?` safety valve** — `.foo?` suppresses the error if the path doesn't apply, producing *nothing* instead of failing.

Through-line: `.[]` and `,` **create** streams (many outputs); `|` **maps** over a stream; everything else is single-step navigation. Always ask: *"how many values is this producing?"*
