# jq Lessons — condensed study notes

> Distilled teaching, one section per module. Progress + exercises live in `jq-training-plan.md`.
> jq 1.8.1 · manual: https://jqlang.org/manual/
> Format: each concept = **syntax line** → **example line** → notes.

---

## The big picture (read this first — everything else fits into it)

```
                 ┌──────────── the real work ────────────┐
input values ──→ explode? ──→ per-element filters ──→ collect? ──→ output
                  (.[])        (.field, select, …)      ([ ])
```

> **Get values as a stream; work per element; drop to array-land only when an
> operation needs the whole collection; collect at the end only if the output
> should be one value.**

The stream of values is the center of jq. Arrays aren't a mandatory stage — they're a place you enter and leave:

- **Getting in:** one array → `.[]` explodes it. NDJSON → already a stream, nothing to do.
- **Working:** per-element filters (`.field`, `select`, `"\(...)"`) run once per value — no array exists here.
- **Array-land, only when needed:** ops that must see all elements at once — `sort_by`, `unique`, `group_by`, `add`, `length`, slices, top-N. That's why `-s` exists (stream → array) and why `map(f)` exists (stay in array-land when you're going right back).
- **Getting out:** don't collect → stream/NDJSON output (`-c`). Collect with `[ ]` / build `{ }` → one value.

Two contrasting shapes:

```
# never needs an array: stream in, per-element, stream out
echo '{"a":1} {"a":2} {"a":3}' | jq -c 'select(.a > 1)'       → {"a":2}  {"a":3}

# needs array-land: a whole-collection op (sort), then back out
echo '{"a":3} {"a":1} {"a":2}' | jq -s 'sort_by(.a) | .[0]'   → {"a":1}
```

---

## Module 0 — Running jq & the 5 flags

```
jq 'PROGRAM' file.json          # or:  cat file.json | jq 'PROGRAM'
```

The 5 flags that matter:

| Flag | Does | When |
|------|------|------|
| `-r` | raw output — strings without quotes | feeding a human or another CLI tool |
| `-c` | compact — one line per output | NDJSON output, logs |
| `-s` | slurp — whole input stream → one array | gather many values / NDJSON → array |
| `-n` | null input — start from `null` | generating data, `-n 'inputs'` |
| `--arg N V` / `--argjson N V` | inject `$N` as string / parsed JSON | passing shell values in |

**`.` — identity**
```
anything | .                    → same value, unchanged
{"a":1} | .                     → {"a":1}
```
It does not "make" an array; it just pretty-prints what was already there.

**`-s` — slurp**
```
jq -s '.'   (stream of N values) → one array of N
{"u":1}⏎{"u":2}  | jq -s '.'    → [{"u":1},{"u":2}]
```
Wraps all *top-level input values* in one array. A single array input → `[thatArray]` (double-wrapped!). It's about how many values the input stream has, not the contents.

**NDJSON** — one JSON value per line, no wrapping array. jq streams these for free (no flag); `-s` gathers them.
```
{"u":1}⏎{"u":2}⏎{"u":3}   = 3 values (a stream)   — NDJSON file
[{"u":1},{"u":2},{"u":3}] = 1 value (an array)    — regular JSON file
```

---

## Module 1 — Core filters: the grammar of everything

**`.foo` / `.foo.bar` — object indexing**
```
object | .key                   → value at key
{"a":{"b":2}} | .a.b            → 2
```
Chains left→right; each step operates on the previous step's output.

**`.["key"]` — bracket form**
```
object | .["any key"]           → value at key
{"odd-key":1} | .["odd-key"]    → 1
```
`.foo` is sugar for `.["foo"]`. Brackets are required when the key isn't a plain identifier (space, dash, leading digit).

**`.[0]` / `.[-1]` — array indexing**
```
array | .[i]                    → element i (negatives from the end)
["a","b","c"] | .[-1]           → "c"
```

**`.[2:4]` — slice**
```
array | .[from:to]              → new array, half-open (to excluded)
["a","b","c","d"] | .[1:3]      → ["b","c"]
```
1 output, still an array — contrast with `.[]`.

**`.[]` — value iterator ⭐**
```
collection | .[]                → stream of its values (N outputs)
["a","b"] | .[]                 → "a"  "b"
```
Explodes; does NOT return an array. `.[].first` → one output per element.

**`,` — comma**
```
input | f, g                    → outputs of f, then outputs of g (same input)
{"a":1,"b":2} | .a, .b          → 1  2
```

**`|` — pipe**
```
stream | f                      → f runs once per value ("for each")
[{"x":1},{"x":2}] | .[] | .x    → 1  2
```
`.[] | .x` == `.[].x`. Piping never changes the count by itself.

**`( )` — grouping**
```
(expr) op ...                   → force precedence, like math
{"age":36} | (.age) + 1         → 37
```

**`?` — safety valve**
```
.foo?                           → value, or *nothing* (no error) if path doesn't apply
"str" | .foo?                   → (empty — no output, no error)
```

Through-line: `.[]` and `,` **create** streams; `|` **maps** over a stream. Always ask: *"how many values is this producing?"*

---

## Module 2 — Iteration & the stream model ⭐

**2a — `.[]` explodes a collection**
```
array | .[]                     → each element   (N outputs)
object | .[]                    → each VALUE, keys dropped
{"a":1,"b":2,"c":3} | .[]       → 1  2  3
```

**2b — a stream is N separate outputs; `|` runs per value**
```
.[] | f                         → f applied to each element   (N in, N out)
[{"n":"Ada"},{"n":"Alan"}] | .[] | .n   → "Ada"  "Alan"
```
The pipe is a silent "for each" — count in == count out (for navigation filters).

**2c — `[ ... ]` re-collects a stream**
```
[ stream ]                      → ONE array of all outputs   (N→1)
[{"age":36},{"age":41}] | [ .[].age ]   → [36,41]
```
Inverse of `.[]`. Everyday idiom **explode→transform→collect**:
```
[ .[] | f ]                     → transform each element, back to array
["ada","alan"] | [ .[] | ascii_upcase ]   → ["ADA","ALAN"]     (ascii_upcase = uppercase; Module 9)
```
This shape *is* `map(f)` (Module 4).
Trap: `[ ]` collects *outputs* — wrapping an already-single value double-wraps: `[ .[0:2] ]` → `[[...]]`.

---

## Module 3 — Building output ⭐

**3a — object construction `{ }`**
```
input | {key: filter, ...}      → new object
{"first":"Ada","age":36} | {name: .first, years: .age}   → {"name":"Ada","years":36}
```
Shorthand when key == field name:
```
{first, age}                    == {first: .first, age: .age}
{"first":"Ada","age":36,"x":9} | {first, age}   → {"first":"Ada","age":36}
```

**3b — string interpolation `"\(...)"`**
```
"text \(filter) text"           → string with filter result spliced in
{"first":"Ada","age":36} | "\(.first) is \(.age)"   → "Ada is 36"
```
Non-strings auto-convert (numbers just work). Text outside `\( )` is literal — literal parens nest fine:
```
{"last":"Hopper","age":85} | "\(.last) (\(.age))"   → "Hopper (85)"
```

**3c — array construction + computed keys**
```
input | [f, g, ...]             → array from the stream inside
{"first":"Alan","age":41} | [.first, .age]   → ["Alan",41]
```
```
input | {(keyFilter): valueFilter}   → key computed from data
{"last":"Turing","active":true} | {(.last): .active}   → {"Turing":true}
```
Without parens the key is the literal string. Key expression must produce a string.

---

## Module 4 — select / map / map_values ⭐

**4a — `select(cond)`: keep or drop**
```
value | select(cond)            → value if cond true, ELSE NOTHING (empty)
41 | select(. > 40)             → 41
26 | select(. > 40)             → (no output)
```
Over a stream it filters:
```
.[] | select(cond)              → stream of matching elements
[{"n":"a","ok":true},{"n":"b","ok":false}] | .[] | select(.ok)   → {"n":"a","ok":true}
```

**4b — `map(f)`: transform an array**
```
array | map(f)                  → array, f applied to each element
[1,2,3] | map(. + 10)           → [11,12,13]
```
`map(f)` **==** `[ .[] | f ]` — the named 2c idiom.
Array-filter idiom:
```
array | map(select(cond))       → array of matching elements
[1,2,3,4] | map(select(. % 2 == 0))  → [2,4]
```
N-vs-1 choice: `.[] | select` = stream; `map(select(...))` = one array.

**4c — `map_values(f)`: transform values in place**
```
object | map_values(f)          → same keys, each value through f
{"a":1,"b":2} | map_values(.+10)  → {"a":11,"b":12}
```
Contrast `map(f)` on an object → array, keys lost.
Gotchas:
- If `f` outputs *empty* for a key, the key is **deleted** → `map_values(select(cond))` drops entries: `{"ada":90,"alan":40} | map_values(select(. >= 50))` → `{"ada":90}`.
- **`if C then T end` (no else) defaults to `else .` (identity), NOT empty** — it keeps the value on false. To delete via `if`: `else empty` explicitly.
- Two clean delete-by-condition idioms: `map_values(select(cond))` or `map_values(if cond then . else empty end)`.

More `select` examples:
- `[{"age":36},{"age":85}] | map(select(.age > 50))` → `[{"age":85}]` — filter, keep as one array.
- `.[] | select(has("tags"))` — keep records having a key (`has` → taught in 5a).

---

## Module 5 — Essential builtins ⭐

**5a — `length`**
```
anything | length               → "how big" (polymorphic)
[1,2,3] | length                → 3     ({"a":1}→1 keys, "hello"→5 chars, null→0)
```

**5a — `keys`**
```
object | keys                   → SORTED array of key names
{"b":2,"a":1} | keys            → ["a","b"]     (keys_unsorted: file order; arrays → [0,1,2])
```

**5a — `has("k")`**
```
object | has("key")             → true/false: does the key EXIST?
{"a":null} | has("a")           → true    (but .a → null, same as missing — has tells apart)
```
Compose: `select(has("email"))`, `.tags | length`. Plain `length` suffices — `. | length` is redundant.

**5b — `add`**
```
array | add                     → elements combined with "+" (null for [])
[1,2,3] | add                   → 6    (strings concat, arrays concat, objects merge)
```
The forever-combo — sum a field:
```
array-of-objects | map(.f) | add     → total of that field
[{"age":36},{"age":41}] | map(.age) | add   → 77
```

**5b — `flatten`**
```
array | flatten                 → nesting removed (flatten(1): one level)
[[1,2],[3,[4]]] | flatten       → [1,2,3,4]     (flatten(1) → [1,2,3,[4]])
```

**5b — `range`**
```
range(n) / range(from;to;step)  → STREAM of numbers (to exclusive — wrap in [ ] for array)
jq -n '[range(0;10;2)]'         → [0,2,4,6,8]
```

**5c — `sort` / `sort_by(f)`**
```
array | sort                    → ascending by natural order
[3,1,2] | sort                  → [1,2,3]
```
```
array | sort_by(f)              → sorted by computed key
[{"n":"b","age":41},{"n":"a","age":36}] | sort_by(.age)   → [{"n":"a",...},{"n":"b",...}]
```
`sort_by(-.age)` for descending.

**5c — `unique` / `unique_by(f)`**
```
array | unique                  → sorted + deduped
[3,1,2,3,1] | unique            → [1,2,3]
```
```
array | unique_by(f)            → sorted by f, ONE element per distinct f-value
[{"n":"kat","t":"b"},{"n":"ada","t":"a"},{"n":"alan","t":"a"}] | unique_by(.t)
                                → [{"n":"ada","t":"a"},{"n":"kat","t":"b"}]
```
Keeps the *first* (input order) of each key — silent data loss; want the groups? use `group_by`.

**5c — `min_by(f)` / `max_by(f)`**
```
array | max_by(f)               → the ONE element with the largest key
[{"n":"a","age":36},{"n":"b","age":85}] | max_by(.age)   → {"n":"b","age":85}
```
Plain `min`/`max` for arrays of scalars.

Family pattern: `sort_by` / `unique_by` / `min_by` / `max_by` / `group_by` all apply `f` to each element and use the result as the comparison key. Learn one, learned all. They *select and arrange* original elements, never transform them — transforming is `map`'s job. Prefer `max_by(.s).n` over sort-descending-take-first; the sort form is for **top N**: `sort_by(-.s) | .[0:3]`.

**5d — `group_by(f)`**
```
array | group_by(f)             → array of ARRAYS, one per distinct f-value (sorted by f)
[{"t":"a","n":1},{"t":"b","n":2},{"t":"a","n":3}] | group_by(.t)
                                → [ [{"t":"a","n":1},{"t":"a","n":3}], [{"t":"b","n":2}] ]
```
Inner arrays hold the **original elements, whole and unchanged** — only the nesting changed. No key label is added: recover the key from any member (`.[0].t`).
The group-and-count recipe (used weekly):
```
group_by(f) | map({key: .[0].f, aggregate})
[{"u":"ada"},{"u":"kat"},{"u":"ada"}] | group_by(.u) | map({user: .[0].u, count: length})
                                → [{"user":"ada","count":2},{"user":"kat","count":1}]
```
Inside the `map`, `.` is one group (a plain array of originals): `.[0].u` = the key, `length` = size, `map(.x)|add` = sum.

**5e — `to_entries` / `from_entries` / `with_entries`**

`.[]` on an object drops keys; the entries family keeps them as data.
```
object | to_entries             → array of {key, value} pairs
{"a":1,"b":2} | to_entries      → [{"key":"a","value":1},{"key":"b","value":2}]
```
```
pairs | from_entries            → object (inverse; also accepts name/k/v)
[{"key":"a","value":1}] | from_entries   → {"a":1}
```
```
object | with_entries(f)        == to_entries | map(f) | from_entries
{"a":1,"b":2} | with_entries(.value += 10)   → {"a":11,"b":12}
```
Inside `f`, `.` is one `{key, value}` pair — so keys are editable too (which `map_values` can't):
```
{"a":1,"b":2} | with_entries(.key |= ascii_upcase)   → {"A":1,"B":2}
```
(`|=` = update-in-place, Module 8; `ascii_upcase` = uppercase, Module 9.)
`select` inside drops the pair (same empty-deletes rule as 4c):
```
{"a":1,"b":25} | with_entries(select(.value > 10))   → {"b":25}
```
Choosing: values only → `map_values(f)` (`.` = the value). Keys involved / pairs-as-data → entries family (`.` = the pair). Full `to_entries|map|from_entries` when you need array-land ops in the middle (sort pairs, dedupe…).

---

## Module 6 — Conditionals, comparisons & error handling ⭐

**6a — comparisons & truthiness**
```
a == b   a != b   a < b   a <= b   a > b   a >= b      → true/false
{"age":41} | .age > 40          → true
```
Work on any JSON type — no coercion ever (`1 == "1"` → false). Cross-type order = sort order: `null < false < true < numbers < strings < arrays < objects`. Strings compare alphabetically:
```
"2026-07-28" > "2026-01-01"     → true    (ISO dates: alphabetical == chronological)
```
Truthiness: **only `false` and `null` are falsy** — `0`, `""`, `[]`, `{}` are all TRUE:
```
[0,"",null,false,[],"x"] | [ .[] | select(.) ]   → [0,"",[],"x"]
```
So `select(.count)` does NOT skip zeros — say `select(.count != 0)`.

**6b — `and` / `or` / `not`**
```
cond1 and cond2                 → true/false (truthiness rule above)
{"age":41,"vip":true} | .age > 40 and .vip   → true
```
`and`/`or` return only true/false — they never hand back an operand (defaults are `//`'s job).
```
cond | not                      → negated — not is a FILTER, you pipe into it
{"vip":false} | .vip | not      → true       (`not .vip` is a syntax error)
```
Comparisons bind tighter than `and`/`or`; parenthesize when in doubt.

**6c — `if / elif / else`**
```
if C then A elif C2 then B else D end        → a VALUE (usable mid-pipeline)
15 | if . < 10 then "small" elif . < 100 then "medium" else "big" end   → "medium"
```
`end` is mandatory. Omitted `else` = `else .` (identity, NOT empty) — to drop, write `else empty`.
Inside object construction, parenthesize:
```
{"n":"kat","s":30} | {n, grade: (if .s >= 60 then "pass" else "fail" end)}   → {"n":"kat","grade":"fail"}
```
Merge variant for wide objects: `. + {grade: (if ... end)}`. In practice `select` covers filtering and `//` covers defaults — `if` earns its keep for labeling/bucketing.

**6d — `//` alternative (default) operator**
```
a // b                          → a, unless a yields no values / only null/false → b
{} | .name // "unknown"         → "unknown"    (missing key → null → default)
```
Chain fallbacks:
```
[{"nick":"kat"},{"name":"ada"},{}] | map(.nick // .name // "anon")   → ["kat","ada","anon"]
```
Gotchas:
- `false` triggers the fallback → NEVER default booleans with `//`: `{"muted":false} | .muted // true` → `true` (real false "corrected" away!). Use `if has("muted") then .muted else true end`.
- `//` also fills in for *empty*, pairing perfectly with `?`:
```
{} | (.a[]? // "none")          → "none"   (? turns the error into empty, // fills the gap)
```

**6e — errors: `?` and `try/catch`**

Some ops ERROR (crash) rather than return null: iterating null, indexing a string/number, bad arithmetic. Field access `.name` is only defined on objects (and null → null); on `42` it's a type error.
```
EXPR?                           → EXPR, or empty if it errors ("skip the failures")
[{"name":"ada"},42] | .[].name?      → "ada"
```
```
try EXPR catch HANDLER          → EXPR, or HANDLER(error-message-string)
[{"name":"ada"},42] | .[] | try .name catch "bad-record"   → "ada"  "bad-record"
```
`try EXPR` alone ≡ `EXPR?`. Inside `catch`, `.` = the error message: `catch "oops: \(.)"`.
Choosing on a dirty stream: `?` = drop bad records silently · `try/catch` = replace them (keep row count / log why) · naked = fail loudly (right when an error means your assumption broke).
Scope: `?` guards the whole postfix chain it ends (`.a.b[]?`); `try` guards exactly what you give it.

---

## Module 8 — Assignment (8a only; deep-path editing parked as not needed)

jq never mutates — assignment returns a **whole new value with one path changed**.

**`=` — set**
```
.path = VALUE                   → whole input, with that path set (missing paths created)
{"a":1,"b":2} | .a = 99         → {"a":99,"b":2}
```
Output is the entire object, not `99`. RHS is evaluated against the **whole input**: `{"n":"ada","up":"XL"} | .n = .up` → `{"n":"XL","up":"XL"}`.

**`|=` — update using the old value**
```
.path |= FILTER                 → whole input, path replaced by (old value | FILTER)
{"n":5} | .n |= . * 2           → {"n":10}
```
Inside the RHS, `.` = current value at that path — so a bare filter works: `.n |= ascii_upcase`.

**Sugar:** `.a += x` ≡ `.a |= . + x` (also `-=` `*=` `/=` `//=`):
```
{"hits":7} | .hits += 1         → {"hits":8}
{"name":null} | .name //= "anon"     → {"name":"anon"}   (fill only if null/false)
```

Gotchas: `=` sets, `==` compares — a lone `=` in a `select` rewrites instead of testing. Pipe vs assignment: `.user.name | f` outputs just the field; `.user.name |= f` outputs the whole doc with the field edited.

---

## Module 9-lite — the "grep verb": test, split/join, startswith/endswith

**`test("regex")` — regex match, boolean**
```
string | test("re")             → true/false     (flags: test("re";"i") = case-insensitive)
"nginx-7d4b" | test("^nginx-")  → true
```
The grep move — pipe the FIELD into test inside select:
```
.[] | select(.field | test("re"))
[{"name":"nginx-a"},{"name":"redis-x"}] | .[] | select(.name | test("^nginx"))   → {"name":"nginx-a"}
```
Regex is PCRE-ish: `^ $ . * + [abc] |` as in grep.

**`split` / `join` — string ↔ array**
```
string | split(SEP)             → array of pieces (SEP = plain string, not regex)
"a,b,c" | split(",")            → ["a","b","c"]
```
```
array | join(SEP)               → one string
["a","b","c"] | join("-")       → "a-b-c"
```
Piece of a structured name: `"nginx-7d4b-x2v" | split("-").[0]` → `"nginx"`. Clean shell output: `[.[].name] | join(",")`.

**`startswith` / `endswith` — no-regex fast path**
```
string | startswith("s")        → true/false
"kube-system" | startswith("kube-")   → true
```
Prefer over `test` for plain prefix/suffix — nothing to escape, no dot surprises.

**Parked:** `capture("(?<n>re)")` pulls named groups into an object — `"v1.29" | capture("v(?<maj>\\d+)")` → `{"maj":"29"}`. Learn on demand; also `sub`/`gsub` (replace), `match` internals, trim fns.
