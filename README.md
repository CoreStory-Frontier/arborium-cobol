# arborium-cobol (patched snapshot)

**This is not an upstream repository. It is a temporary, patched snapshot of the
published [`arborium-cobol`](https://crates.io/crates/arborium-cobol) crate,
version 2.18.1, existing solely so `plot` can consume the COBOL parse-hang fix
before it lands upstream. Delete it once it has.**

Nothing here is published to crates.io. `plot` consumes it via
`[patch.crates-io]`, which replaces the crate *by name* — no registry name
ownership is involved.

## The one change

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
is kept at `patches/0001-line-prefix-comment-eof-guard.patch`. Nothing else in
the crate is modified.

## Evidence

- Differential test over 157 real COBOL files (AWS CardDemo + CMS IPF Pricer),
  comparing full parse-tree fingerprints before and after: **116 byte-identical,
  41 rescued from hanging, 0 changed.** No successful parse is altered.
- All 41 files of the CMS IPF Pricer corpus hung before this patch; all 41 parse
  after it. `ipcal140.txt`: never returned (135s CPU, killed) → **15ms**.
- In the upstream grammar repo: `tree-sitter test` 81/81, and the NIST COBOL85
  suite unchanged at 382 tests (371 success, 0 fail, 11 skip).

## Upstream status

The fix is proposed against the grammar repository arborium vendors from:
<https://github.com/zharinov/tree-sitter-cobol/pull/1>

Provenance of the buggy file, outermost first:

| Layer | Repository | Role |
| --- | --- | --- |
| 1 | `CoreStory-Frontier/plot` | consumer |
| 2 | crates.io `arborium-cobol` | published crate (this snapshot) |
| 3 | `bearcove/arborium` | vendors the grammar into `langs/group-rowan/cobol/def/` |
| 4 | `zharinov/tree-sitter-cobol` | arborium's pinned commit — PR #1 filed here |
| 5 | `yutaro-sakamoto/tree-sitter-cobol` | canonical grammar; same bug, still unfixed |

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
