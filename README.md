[![site-generator-speed-01-run-20261122](https://make.fun/badge/site-generator-speed-01-run-20261122.svg)](https://make.fun/bounty/site-generator-speed-01-run-20261122)
[![rank](https://make.fun/badge/site-generator-speed-01-run-20261122/Kinanty22/site-generator-speed-01/rank.svg)](https://make.fun/bounty/site-generator-speed-01-run-20261122)
[![reward](https://make.fun/badge/site-generator-speed-01-run-20261122/Kinanty22/site-generator-speed-01/reward.svg)](https://make.fun/bounty/site-generator-speed-01-run-20261122)

# site-generator-speed-01

A faster drop-in for make.fun's own makefun-site 1.0.0: it builds a static
site from `{ site, templates, files }` and returns exactly makefun-site's
answer — the output tree as a JSON object of every path it writes and that
file's text, byte for byte — for every input, in much less CPU time.

This entry is for the make.fun competition
[site-generator-speed-01-run-20261122](https://make.fun/bounty/site-generator-speed-01-run-20261122)
("Build a static site faster, 22 November 2026"). Places go by CPU time,
least first, among entries whose answers match makefun-site 1.0.0 on every
case, and a reward needs beating the house baseline at 95% confidence.

## How it works

makefun-site 1.0.0 is a pipeline of small plugins over token arrays: it
lexes every Markdown body into block tokens, renders inline Markdown one
character at a time (escaping each character through a regex), slugifies
with `normalize` + regex on every call, counts words with a `match` array,
and walks a scope chain with `hasOwn` on every template variable. This
entry keeps every rule and changes only the shape:

- **Markdown, no tokens.** Blocks render straight into one output array
  while scanning lines by `indexOf('\n')`; inline rendering works on
  ranges of the source string instead of slicing inner text for every
  strong/em/link, and plain runs are escaped in one chunk (a chunk with
  nothing to escape is returned untouched, no copy).
- **Slugify, cached per run.** A per-`run()` cache (nothing is carried
  from one case to the next) collapses the repeated slug calls — tags are
  slugified several times per post in the reference, headings repeat —
  and ASCII text takes a fast path that never calls `normalize`.
- **Words counted by scanning**, not by allocating a `match` array; the
  ASCII whitespace check is inlined in the loop.
- **Front matter parsed without splitting the body**: only the small
  front-matter region is split; the body is one slice.
- **Templates compiled once per case.** Each template is compiled to a
  plain JS function (`new Function`, no dependencies): includes are
  inlined up to the reference's depth of 8, loop variables become real
  lexical variables, and paths resolve to direct property access with the
  reference's own-property semantics (`getProp`). If compilation ever
  fails or a template's code grows too large, an interpreted renderer
  with the reference's exact scope-chain semantics takes over.
- **One view object per source** serves as both its card and its page
  (the templates only read those fields, and the values are identical),
  instead of rebuilding a card for every listing a post appears in.

Pure JavaScript, no dependencies, no WebAssembly, no native code.

On the kit practice set (3 cases, 1,070 sources, ~2.9 MB of input),
`run()` takes about 180 ms where makefun-site 1.0.0 takes about 520 ms
(fresh-process medians on this machine, interleaved runs; the machine was
shared with other builds, so treat the absolute numbers as approximate —
the gap never closed across any paired round, roughly 2.8–3×). The whole
program end to end is about 0.5–0.6 s against the baseline's ~1.8 s on a
quiet run. Correctness was checked beyond the practice set: 124 hand-made
differential edge cases (template parse errors, include cycles and chains
past depth 8, loop shadowing, unicode slugs, every front-matter refusal,
Markdown edge cases) and 27 fresh cases drawn from the kit's own
generator with new seeds — all byte-identical to makefun-site 1.0.0.

## Run

Node 22 or later. The harness runs `node main.ts`; it reads every case from
`/input/cases.json` and writes `/output/answers.json`. Outside a container,
`MAKEFUN_INPUT_DIR` and `MAKEFUN_OUTPUT_DIR` stand in for `/input` and
`/output`:

```sh
MAKEFUN_INPUT_DIR=./input MAKEFUN_OUTPUT_DIR=./output node main.ts
```

`main.ts` is the driver; `solve.js` holds the builder, exporting `run(input)`.

## License

MIT — see [LICENSE](LICENSE).

Referral code: `mjp730py`
