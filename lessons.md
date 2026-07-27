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

---

## Module 2 — Iteration & the stream model ⭐

One question runs through everything: *"how many values are flowing right now?"*

**2a — `.[]` explodes a collection into a stream.** Emits each element as a *separate output*, not an array.
- On arrays: each element. On objects: each *value*, keys dropped.
- `{"a":1,"b":2,"c":3} → .[]` → `1, 2, 3` (three outputs).

**2b — a stream is N separate outputs; `|` runs the next filter once per value.**
- `.[] | .first` → 4 outputs; the pipe is a silent "for each." Count doesn't change by piping: `.[] | .first` == `.[].first`.

**2c — `[ ... ]` re-collects a stream into one array** (inverse of `.[]`).
- `.[].age` = 4 outputs; `[ .[].age ]` = 1 output `[36,41,85,101]`. `.[]` collapses 1→N, `[ ]` collapses N→1.
- Everyday idiom **explode→transform→collect**: `[ .[] | .first | ascii_upcase ]`. This shape *is* `map(...)` (Module 4).
- Trap: `[ ]` collects *outputs*. Wrapping an already-single value double-wraps: `[ .[0:2] ]` → `[[ ... ]]`.

---

## Module 3 — Building output ⭐

From extracting to *constructing* new JSON.

**3a — object construction `{ }`.** Inside braces, `key: filter`; the filter runs against the input and its output becomes the value.
- `{name: .first, years: .age}` → `{"name":"Ada","years":36}`.
- Shorthand `{first, age}` == `{first: .first, age: .age}` — plucks a subset of fields.

**3b — string interpolation `"\(.expr)"`.** Inside a string literal, `\( ... )` evaluates a filter and splices its result into the text.
- `"\(.first) \(.last) is \(.age)"` → `Ada Lovelace is 36` (with `-r`).
- Non-strings auto-convert inside interpolation (no `tostring`). Text outside `\( )` is literal — you can nest inside literal parens: `"\(.last), \(.first) (\(.age))"`.

**3c — array construction `[ ]` + computed keys.**
- `[.first, .last, .age]` → `["Ada","Lovelace",36]` (same N→1 collection as 2c; commas make the stream).
- Computed key: wrap the key filter in `( )` → `{(.first): .age}` → `{"Ada":36}`. Without parens the key is the literal string. Key expression must produce a string.
