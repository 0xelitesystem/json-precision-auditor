# JSON Precision Auditor

Paste JSON and see which number literals change value when parsed into IEEE 754 doubles, including the ones that re-print byte-for-byte identical to what you pasted.

**Live demo:** https://0xelitesystem.github.io/json-precision-auditor/

## The finding this exists for

`801730537304183000` parses to `801730537304183040`.

The double is off by 40. It also re-prints as `801730537304183000`, so the obvious check for this,
`String(JSON.parse(x)) !== x`, returns false and reports nothing. Two more literals do the same
thing:

| Literal | Exact value of the double | Off by | Re-prints as | `String(JSON.parse(x)) !== x` |
| --- | --- | --- | --- | --- |
| `801730537304183000` | `801730537304183040` | +40 | identical | silent |
| `3981889031900142000` | `3981889031900142080` | +80 | identical | silent |
| `42420814076809180` | `42420814076809184` | +4 | identical | silent |

Those are 17 to 19 digit IDs ending in round digits, which is what you get when an upstream
JavaScript service already round-tripped a bigint through a double before handing it to you. The
text survives the round trip. The value does not. Compare that ID against a database bigint and it
will not match, and nothing in the JSON you received looks wrong.

The page computes every digit in that table in your browser, from the three literals, on load. It
is not stored as text in the file. Press **Load example** to see the same three inside a document,
with JSON paths.

## Use

1. Paste JSON into the box and press **Audit**, or press **Load example**.
2. Read the top row of counters: invisible severity 1 findings first, then how many value changes
   `String(JSON.parse(x)) !== x` misses, then how many it false-alarms on.
3. Work down the lanes. They are ordered by round-trip survival, not by exactness:

- **Severity 1, invisible.** The value changed and the re-print is byte-for-byte identical. Nothing
  in the text tells you. This is the lane the tool exists for.
- **Severity 1, visible.** The value changed and the re-print differs: integers past 2^53-1 such as
  Snowflake and Discord IDs, bigint primary keys and nanosecond timestamps, decimals carrying more
  precision than binary64 holds, exponents that overflow to IEEE 754 infinity, magnitudes that
  underflow to zero.
- **Severity 2.** Exact in binary64 today, but outside the interoperability range RFC 8259 names.
  The value is fine. The guarantee that every implementation agrees on it is gone.
- **Severity 3.** Ordinary inexact decimals, collapsed into one row with a count.
- **Formatting only.** The value survives and the spelling does not: `1.0`, `1e2`, `-0`, `1.500`.
  Every row here is a false alarm from the naive check.
- **RFC 8259 number grammar.** A separate lane, described below.

Both copy buttons export the full report, as text or as JSON. Nothing is truncated silently: if a
table is capped, the row above it prints the real total.

## Why this exists

**Ranking by round-trip survival is the point.** A tool that flags every inexact number is a
firehose. Most decimals are inexact in binary64 and almost none of them matter, so ordinary
inexact decimals collapse into a single counted row. In the shipped example, 8 of the 9 price
literals are inexact, and they produce one row rather than eight findings with deltas near 1e-15
that would bury the three that matter. A thousand-row price list produces the same one row.

**The naive heuristic is wrong in both directions.** `String(JSON.parse(x)) !== x` is silent on the
three literals above, and it false-alarms on `1.0`, `1e2`, `-0`, `1.500` and `1e22`, whose values
are exactly preserved and whose spelling is not. The page runs that heuristic beside the exact
engine on every literal and prints both answers, so you can see the gap on your own document.

**The comparison is exact, in BigInt.** The literal is converted to an exact decimal value. The
double is decomposed through a DataView into sign, an 11 bit exponent and a 52 bit significand, and
reconstructed as an exact decimal value. The two are compared by cross multiplication. No floating
point is used anywhere in the comparison, so the comparison cannot inherit the error it is looking
for. For each changed literal the page also verifies in BigInt that twice the absolute error is at
most one ulp, which is the check that the browser handed back the correctly rounded nearest double.

**The grammar lane is separate on purpose.** "Your producer emitted non-standard JSON that some
parsers accept" and "your value changed" are different problems with different owners. Mixing them
is what produces formatting false positives. `007`, `+1`, `.5`, `5.`, `0x1F`, `1_000_000`, `NaN`,
`Infinity` and `1e` are reported in their own lane with the reason and, where a tolerant parser
would still produce a value, what that value does. Press **Load non-standard example** for all
nine. RFC 8259 section 6 is quoted verbatim on the page, including the ABNF, because the grammar
itself imposes no magnitude or precision bound: the repetition operators are unbounded, so an
arbitrarily long integer part, fraction and exponent are all well-formed JSON. The interoperability
range of -(2**53)+1 through (2**53)-1 is a note in the same section, not a limit in the grammar.

**What it refuses to do.** It does not report string-versus-number type drift for a key, because
that falls out of a per-key type union during schema inference, which is
[json-to-json-schema](https://0xelitesystem.github.io/json-to-json-schema/)'s job, not this one. It
does not rewrite your JSON or recommend a fix, because the fix is a wire-format decision: quote the
ID as a string, use a bigint-aware parser, or accept the loss. It does not tell you whether your
production system is broken. Every verdict it prints is a property of the text in the box.

## Privacy

Everything runs in the browser. There are no network requests of any kind: no analytics, no fonts,
no CDN, no telemetry, no upload. The only thing written to storage is the theme choice in
localStorage, wrapped in try/catch so it degrades in private-mode contexts. Paste production JSON
into it offline if you like: pull the page down, disconnect, and it behaves the same.

## Run locally

Download `index.html` and open it in a browser. That is the whole procedure. It works over
`file://`; nothing on the page needs a server or an internet connection.

    git clone https://github.com/0xelitesystem/json-precision-auditor.git
    cd json-precision-auditor
    start index.html    # or: open index.html, or xdg-open index.html

## Build

There is no build step and no toolchain. `index.html` is the entire tool: one file, inline
`<style>` and inline `<script>`, no dependencies, no bundler, no package manifest, no lockfile.
Edit it and reload the page.

Limits are constants at the top of the script, and each one prints a message when it is hit rather
than failing quietly: `MAX_INPUT_CHARS` (input refused above 4,000,000 characters),
`MAX_DIGITS` (exact comparison skipped for a single literal past 20,000 digits), `MAX_ROWS`
(rendered rows per table, with the real total always printed), `MAX_EXPORT` (rows per exported
lane, with the real total printed).

## Related

- [json-to-json-schema](https://0xelitesystem.github.io/json-to-json-schema/) infers a schema from
  pasted JSON. Per-key type drift belongs there, which is why this tool does not report it.
- [json-formatter-and-validator](https://0xelitesystem.github.io/json-formatter-and-validator/)
  formats and validates through the browser's own JSON parser. That is the parser this tool audits:
  by the time a formatter has a value to print, the evidence is gone.
- [unix-timestamp-converter](https://0xelitesystem.github.io/unix-timestamp-converter/) converts
  timestamps. Nanosecond timestamps past 2^53 are one of the visible severity 1 classes here.
- [structured-output-and-json-mode-reference](https://github.com/0xelitesystem/structured-output-and-json-mode-reference)
  covers getting reliable JSON out of a language model. Reference repo, no live page.

## License

MIT. See [LICENSE](LICENSE).

## More

Part of a catalog of single-file browser tools and plain-language references,
all MIT licensed and dependency-free: [0xelitesystem.github.io](https://0xelitesystem.github.io/).
Built by [elitesystem.ai](https://elitesystem.ai).
