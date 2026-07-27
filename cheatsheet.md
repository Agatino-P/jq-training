# Things to keep — jq cheat sheet

> Gotchas, one-liners, and "aha" moments. Filled in as we go.
> Lessons: `lessons.md` · Curriculum + progress: `jq-training-plan.md`

- **`.` is identity** — returns input unchanged (just pretty-prints). It does NOT "make an array"; the file was already whatever it was.
- **`-s` (slurp) = "take the whole input *stream* and wrap all top-level values in one array."** Stream of 4 → `[a,b,c,d]`. A single array input → `[thatArray]` (double-wrapped!). Slurp is about how many values the *input stream* has, not about the contents.
- **NDJSON** = one JSON value per line, no wrapping array. jq streams these for free (no flag). Use `-s` to gather into an array. `people.json` = 1 value (array); `events.ndjson` = 4 values (stream).
- **`-r`** = raw output: strings without quotes. Use when feeding a human or another CLI tool.
- **`if C then T end` (no else) defaults to `else .`, not `else empty`.** The value passes through unchanged on false. For empty/deletion write `else empty` explicitly. (This is why `map_values(if C then . end)` keeps everything, but `map_values(select(C))` deletes non-matches.)
