# arborium-cobol (patched snapshot)

**This is not an upstream repository. It is a temporary, patched snapshot of the
published [`arborium-cobol`](https://crates.io/crates/arborium-cobol) crate,
version 2.18.1, existing solely so `plot` can consume two COBOL scanner fixes
before they land upstream. Delete it once they have.**

Nothing here is published to crates.io. `plot` consumes it via
`[patch.crates-io]`, which replaces the crate *by name* — no registry name
ownership is involved.

## Change 1: parse hang at end of file

`grammar/scanner.c`, the `LINE_PREFIX_COMMENT` branch of the external scanner,
looped on the column alone:

```c
while(lexer->get_column(lexer) <= 5) {
    lexer->advance(lexer, true);
}
```

At end of file `advance` is a no-op, so the column freezes at ≤5, never reaches
6, and the scanner spins at 100% CPU forever. tree-sitter marks all external
tokens valid during error recovery, so this is triggered by **content, not file
size** — a one-byte file containing `!` is enough. The spin is inside
`external_scanner_scan`, so it cannot be interrupted from outside: a
`ParseOptions::progress_callback` budget never fires.

The fix stops the loop when input is exhausted and returns `false` rather than
emitting a zero-width token. The exact diff against the pristine published crate
is kept at `patches/0001-line-prefix-comment-eof-guard.patch`.

### Evidence

- Differential test over 157 real COBOL files (AWS CardDemo + CMS IPF Pricer),
  comparing full parse-tree fingerprints before and after: **116 byte-identical,
  41 rescued from hanging, 0 changed.** No successful parse is altered.
- All 41 files of the CMS IPF Pricer corpus hung before this patch; all 41 parse
  after it. `ipcal140.txt`: never returned (135s CPU, killed) → **15ms**.
- In the upstream grammar repo: `tree-sitter test` 81/81, and the NIST COBOL85
  suite unchanged at 382 tests (371 success, 0 fail, 11 skip).

## Change 2: single-quoted literals continued across lines

A fixed-format alphanumeric literal may continue onto the next line: the literal runs to column 72, and the next line carries `-` in column 7 followed by a reopening quote.
The external scanner already handled this for double-quoted literals (its `multiline_string` branch), but only for `"`.
A continued single-quoted literal, the common form, fell to the grammar's one-line pattern, failed, and became an ERROR node that swallowed the literal and much of the code after it.

The fix adds `scan_continued_literal` for `'` literals.
It claims the token only when the literal actually continues, and it treats a doubled quote as an escaped quote.
A literal that closes on its own line returns `false` and is left to the internal lexer, so single-line literals parse exactly as before.
The double-quote branch is untouched.
The diff is kept at `patches/0002-single-quote-continued-literal.patch`.

### Evidence

- Differential test over 113 real COBOL files (AWS CardDemo at `59cc6c2` plus the Open Mainframe Project COBOL course), comparing full parse-tree fingerprints before and after: **112 byte-identical, 1 changed, 0 regressed**.
  The changed file is CardDemo's `app/cbl/CBSTM03A.CBL`, whose error-byte coverage drops from 0.131 (one ERROR span, bytes 7313-11966) to 0.000.
- A continued literal now yields the same tree shape as the equivalent single-line literal, also with an escaped quote, a literal continued over three lines, sequence numbers in columns 1-6 and 73-80, and CRLF line endings.

## Upstream status

Change 1 is proposed against the grammar repository arborium vendors from:
<https://github.com/zharinov/tree-sitter-cobol/pull/1>

Provenance of the buggy file, outermost first:

| Layer | Repository | Role |
| --- | --- | --- |
| 1 | `CoreStory-Frontier/plot` | consumer |
| 2 | crates.io `arborium-cobol` | published crate (this snapshot) |
| 3 | `bearcove/arborium` | vendors the grammar into `langs/group-rowan/cobol/def/` |
| 4 | `zharinov/tree-sitter-cobol` | arborium's pinned commit — PR #1 filed here |
| 5 | `yutaro-sakamoto/tree-sitter-cobol` | canonical grammar; same bug, still unfixed |

Change 2 is not yet proposed upstream.
The canonical grammar still handles only double-quoted continuation (checked at `550020dd`, 2026-09-09).

## Retirement

When `bearcove/arborium` carries the fix and publishes a release, drop the
`[patch.crates-io]` block from plot's root `Cargo.toml`, bump the version
requirement, and delete this repository.

## License and attribution

MIT, unchanged. The crate is authored by Amos Wenger &lt;amos@bearcove.eu&gt; as part
of [arborium](https://github.com/bearcove/arborium); the COBOL grammar itself
originates from [`yutaro-sakamoto/tree-sitter-cobol`](https://github.com/yutaro-sakamoto/tree-sitter-cobol)
(also MIT). See `LICENSE-MIT`, and `README.upstream.md` for the crate's original
README.
