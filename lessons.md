# jq Lessons — condensed study notes

> Distilled teaching, one section per module. Progress + exercises live in `jq-training-plan.md`.
> jq 1.8.1 · manual: https://jqlang.org/manual/
> Format: each concept = **syntax line** → **example line** → notes.

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
`unique_by(f)` dedupes by computed key, keeps one representative each.

**5c — `min_by(f)` / `max_by(f)`**
```
array | max_by(f)               → the ONE element with the largest key
[{"n":"a","age":36},{"n":"b","age":85}] | max_by(.age)   → {"n":"b","age":85}
```
Plain `min`/`max` for arrays of scalars.

Family pattern: `sort_by` / `unique_by` / `min_by` / `max_by` (and `group_by`) all apply `f` to each element and use the result as the comparison key. Learn one, learned all.
