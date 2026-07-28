# Things to keep — jq cheat sheet

> Gotchas, one-liners, and "aha" moments. Filled in as we go.
> Lessons: `lessons.md` · Curriculum + progress: `jq-training-plan.md`
> Format: syntax line → example line → the gotcha.

**`.` is identity**
```
anything | .                    → same value, unchanged (just pretty-printed)
```
It does NOT "make an array"; the file was already whatever it was.

**`-s` (slurp) wraps the input STREAM, not the contents**
```
jq -s '.'  (N top-level values) → one array of N
stream of 4                     → [a,b,c,d]     single array input → [thatArray] !!
```
Slurp counts input values; a lone array gets double-wrapped.

**NDJSON = one JSON value per line**
```
jq '.field' file.ndjson         → streams for free, one output per line (no flag)
jq -s '.'   file.ndjson         → gathered into one array
```

**`-r` = raw strings**
```
jq -r '.name'                   → Ada        (no quotes — for humans & CLI tools)
```

**`.[]` explodes (N outputs) vs slice stays one array**
```
["a","b","c"] | .[]             → "a" "b" "c"      (3 outputs)
["a","b","c"] | .[0:3]          → ["a","b","c"]    (1 output)
```

**The standard pipeline: stream at the center, arrays only when needed**
```
input values → explode? (.[]) → per-element filters → collect? ([ ]) → output
{"a":1}⏎{"a":2} | select(.a>1) → {"a":2}      (NDJSON: already a stream — no -s)
```
Per-element work (`.field`, `select`, `"\(...)"`) needs no array. Enter array-land only for whole-collection ops (`sort_by`, `unique`, `group_by`, `add`, `length`, slices, top-N) — that's what `-s` (NDJSON→array) and `map` (stay in array-land) are for. Collect at the end only if the output should be one value.

**Only `false` and `null` are falsy — `0`, `""`, `[]`, `{}` are truthy**
```
0 | if . then "yes" else "no" end    → "yes"     (JS/Python instinct says "no"!)
select(.count)                       → does NOT skip zeros; say select(.count != 0)
```

**`not` is a filter, not an operator — pipe into it**
```
.vip | not                      → negated       (`not .vip` = syntax error)
```

**`//` swallows `false` — never default booleans with it**
```
{"muted":false} | .muted // true     → true !!   (real false "corrected" away)
{"muted":false} | if has("muted") then .muted else true end   → false (right)
```

**ISO dates compare/sort as plain strings**
```
"2026-07-28" > "2026-01-01"     → true      (YYYY-MM-DD: alphabetical == chronological)
```
Works in `select`, `sort_by(.ts)`, `min/max_by` — no date parsing needed. Non-ISO formats sort garbage.

**`? // ` — the resilient-access combo**
```
{} | (.a[]? // "none")          → "none"    (? : error→empty · // : empty→default)
```

**`if C then T end` (no else) defaults to `else .` — NOT `else empty`**
```
40 | if . >= 50 then . end      → 40   (value passes through on false!)
40 | if . >= 50 then . else empty end   → (nothing)
```
This is why `map_values(if C then . end)` keeps everything, but `map_values(select(C))` deletes non-matches.
