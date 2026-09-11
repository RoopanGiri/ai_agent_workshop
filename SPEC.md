# SPEC.md — mytools

Every `_______` is a decision you own. Leave a one-line rationale next to each — future-you and your agent both need it.

Filled-in entries say where the decision came from: **(Qn)** is question *n* of the spec
interview; **(CLAUDE.md)** is the project conventions file.

---

## 1. Scope

Subcommands in v1: `sort`, `merge`, `intersect`, `subtract`, `closest`
— the subset named in CLAUDE.md; chosen over all 44 bedtools subcommands once their cost
(BAM/FASTA/BEDPE formats, RNG, statistics) was clear (Q1).

Explicitly NOT in v1 (write these down — scope creep is the main failure mode here):
every other bedtools subcommand — nothing that makes `mytools` more than this subset (Q2).

## 2. Input formats

- Formats accepted: same as bedtools for each subcommand — BED (BED3–BED12), GFF/GTF and
  VCF for all five; BAM as well for `intersect`, as `-a` or `-b` (Q3).
- Read from file argument, stdin, or both? Both (CLAUDE.md).
- How is `-` interpreted? As stdin (CLAUDE.md).
- Compressed input (`.gz`)? Yes — gzip and other general-purpose compression (bzip2, xz,
  zstd), recognised by file content, not extension (Q3: "if it contains appropriate
  data"). bedtools reads only gzip, so this goes beyond it (§8).
- `track` / `browser` / `#` comment lines: skip, pass through, or error? Skip, as
  bedtools does; printed only with `-header` (§5) (Q4–Q6). A line is a header if it:
  - starts with `#` — any such line, as in bedtools (Q5);
  - starts with `track` or `browser` as a whole word, case-insensitive: followed by a
    space, a tab or end of line. Deliberately differs from bedtools' case-sensitive
    prefix check, which errors on `Track name=x` and drops data lines like
    `track1  5  10` (Q4, §8).

## 3. Interval semantics

- Coordinate system: **0-based half-open** (this is not a decision; BED says so).
- Do bookended intervals (`a.end == b.start`) overlap? No. They do merge under
  `merge -d 0` (CLAUDE.md; confirmed in bedtools' docs — see `-d` in §4).
- Are zero-length intervals (`start == end`) legal? What do they overlap? Legal; they
  overlap whatever bedtools says they do — match the oracle, don't "fix" them (CLAUDE.md).
- Minimum overlap to count as an overlap: same as bedtools — 1 bp by default, adjusted by
  `-f`, `-F`, `-r`, `-e` (follows from byte-identical output (Q9) and the full flag set (Q7)).

## 4. Flags per subcommand

For each subcommand, the flag subset you will support. Match bedtools' names and
meanings exactly — you are being graded against it.

Every flag bedtools v2.31.1 has for these five subcommands (Q7).

| Subcommand  | Flags in v1 | Notes |
|-------------|-------------|-------|
| `sort`      | `-sizeA -sizeD -chrThenSizeA -chrThenSizeD -chrThenScoreA -chrThenScoreD -g -faidx -header` | 9 flags |
| `merge`     | `-d -s -S -c -o -delim -prec -bed -header -iobuf -nobuf` | 11 flags; `-d` defined below |
| `intersect` | `-wa -wb -wo -wao -loj -u -v -c -C -f -F -r -e -s -S -split -sorted -g -names -filenames -sortout -nonamecheck -bed -ubam -header -iobuf -nobuf` | 27 flags |
| `subtract`  | `-A -N -f -F -r -e -s -S -split -sorted -g -wb -wo -nonamecheck -bed -header -iobuf -nobuf` | 18 flags |
| `closest`   | `-d -D -io -iu -id -fu -fd -t -k -mdb -N -f -F -r -e -s -S -split -g -names -filenames -sortout -nonamecheck -bed -header -iobuf -nobuf` | 27 flags |

- Strand-aware flags (`-s`, `-S`)? Yes — part of the full flag set (Q7).
- Does `merge` need pre-sorted input, or does it sort for you? Needs sorted input and
  rejects unsorted input, as bedtools does (Q8).
- Does `closest` require sorted input? Yes, for both `-a` and `-b`, which must also use
  the same chromosome order — as bedtools (Q8).
  - "Sorted", as bedtools checks it: each chromosome's records form one contiguous block
    and starts never decrease within it. Chromosome order and end order are not checked.

**`merge -d`** — from the bedtools docs and `-h`, behaviour checked on bedtools v2.31.1:

- Definition: "Maximum distance between features allowed for features to be merged."
  Default `0`: overlapping and bookended features are merged.
- An interval joins the current merged block when `next.start − end ≤ d`. At `-d 0`,
  `200–300` and `301–400` (a 1 bp gap) stay separate; at `-d 1` they merge. Whether
  `end` is the merged block's end or the previous record's end is settled by golden
  case B (§8).
- This is **not** the overlap predicate in §3 — that one keeps bookended intervals apart.
- **Negative values are valid.** At `-d -1`, bookended features stay separate; whether
  overlapping features still merge is settled by golden case A (§8). The argument parser
  must accept a value beginning with `-`: `-d -1` is a distance, not an unknown option.

## 5. Output

- Output format per subcommand: same as bedtools, byte-identical (Q9).
- Field separator, trailing newline, how empty results are printed: same as bedtools (Q9).
- `-header` handling: same as bedtools (Q6) — header lines from the top of the main input
  (`-i` or `-a`), verbatim, before the results, even when there are no results; never
  from `-b`; header-like lines further down the file are skipped and never printed.

## 6. Memory model

- Streaming, fully in-memory, or per-chromosome? Same as bedtools (Q10): `sort` holds the
  whole input in memory; `merge` streams; `-sorted` modes and `closest` stream both
  inputs; `intersect`/`subtract` without `-sorted` hold `-b` in memory.
- Largest input you promise to handle: whatever bedtools handles on the same machine, at
  the speed you'd expect of bedtools (Q10 — replaces the original 5 GB target).
- Which subcommands can stream and which fundamentally cannot? `merge`, and
  `intersect`/`subtract`/`closest` on sorted input, stream; `sort` cannot (Q10, as bedtools).

## 7. Errors and exit codes

Exit codes follow CLAUDE.md, not bedtools: `0` success, `1` bad input data, `2` usage
error. Errors go to stderr (Q11, CLAUDE.md).

| Situation             | stderr message | exit code |
|-----------------------|----------------|-----------|
| Success               | —              | `0`       |
| Malformed BED line    | `_______`      | `1`       |
| `start > end`         | `_______`      | `1`       |
| Unknown flag          | `_______`      | `2`       |
| Missing input file    | `_______`      | `_______` |
| Unsorted input to `closest` | `_______` | `1`      |

A non-integer start or end is bad input data: exit `1`, where bedtools crashes with exit
134 (Q11, §8).

## 8. Correctness

- Oracle: real `bedtools` on the files in `data/`. Non-negotiable.
- Which subcommand/flag combinations get a golden test in v1? Every subcommand and every
  flag in §4 (CLAUDE.md: new subcommand or flag ⇒ golden case in the same commit), plus:
  - Both cases below run on `data/a.bed` sorted by bedtools first, because `merge`
    requires sorted input: `bedtools sort -i data/a.bed > "$tmp/a.sorted.bed"`.
  - **Case A — `merge -d -1`.** `a.bed` has a bookended pair (`a01` 0–100, `a02`
    100–200) and an overlapping pair (`a02` 100–200, `a03` 150–250). Shows whether
    `-d -1` keeps the first apart while still merging the second.
  - **Case B — `merge -d 100`.** `a.bed` has `a06` (320–350) nested inside `a05`
    (300–400), followed by `a07` (500–500). Measured from the merged block's end (400)
    the gap is 100 and `a07` merges; measured from the previous record's end (350) it is
    150 and it doesn't. The output differs depending on which rule is right.
- Known deviations from bedtools you are accepting, and why:
  - bzip2 / xz / zstd input: read, where bedtools errors — user requirement (Q3). Golden
    tests compare against bedtools on the decompressed file.
  - `Track name=x`, `BROWSER …`: skipped as headers, where bedtools errors (Q4).
  - Data lines whose chromosome starts with `track`/`browser` (`track1`): kept as data,
    where bedtools silently drops them (Q4).
  - Exit codes: CLAUDE.md's scheme (§7), where bedtools uses 0 for no-argument help and 1
    for usage errors (Q11).
  - Non-integer start/end: exit 1, where bedtools crashes with exit 134 (Q11).

## 9. Language and layout

- Implementation language: R, base R only — no third-party packages (CLAUDE.md).
- Entry point / how it is invoked: `./mytools` in the repo root, an executable `Rscript`
  script (issue #1). It dispatches to one R file per subcommand under `R/`, plus shared
  modules (I/O, formats, intervals, BAM) — layout delegated to Claude, chosen so parallel
  agents don't edit the same file and unit tests can load a single module (Q12).
- Where tests live: `tests/` — `tests/run_golden.sh` against bedtools; unit tests in base
  R that run without bedtools (CLAUDE.md).
