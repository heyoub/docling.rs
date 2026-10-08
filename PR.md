Title: fix(pdf): a change of font no longer ends a line cell
---
## What's wrong

When a phrase changes font in the middle of a line, docling.rs ends the line cell at each font change, and assembly then joins the cells with a space. Text appears with spaces the page does not have. `tests/data/pdf/sources/redp5110_sampled.pdf`, page 8, is an example. The content stream shows `Work Function Usage (` in Helvetica, then `WRKFCNUSG` in BookMasterGothic-Bold, then `)` in Helvetica, and the three runs touch.

| | page 8 bullet |
|---|---|
| PDF shows (pdftotext) | `Work Function Usage (WRKFCNUSG)` |
| docling.rs `master` (snapshot `redp5110_sampled.pdf.md:183`) | `- /SM590000 Work Function Usage ( WRKFCNUSG )` |
| docling `main` groundtruth (`redp5110_sampled.md:181`) | `- SM590000 Work Function Usage (WRKFCNUSG)` |
| this PR (line cell) | `Work Function Usage (WRKFCNUSG)` |

The same rule is why `multi_page.pdf` reads `Wang Laboratories :` here. docling `main` has `Wang Laboratories:`, and so does this PR.

## Why

`dp_lines.rs` ports docling-parse's pre-7.21 line contractor. In that contractor, `applicable_for_merge` refuses to merge cells in different fonts (`enforce_same_font`, `cells.h` at v7.20.0). docling-parse 7.21.0 (docling-parse#351, "bbox of brackets, summations, missing font-mappings etc.") replaced it. The PR description says `enforce_same_font` is "deprecated and ignored". Words now split only at explicit space glyphs, or at a cursor gap above min(0.35, …) neighbouring advances (`cells.h` at v7.22.1; docling-parse#361 only tuned that rule, and `cells.h` has not changed since). docling requires `docling-parse>=7.22.1` since 42de1e2, first released in v2.132.0, and its groundtruth moved with it.

`applicable` here still has the font gate for both contractions:

```rust
if a.font != 0 && b.font != 0 && a.font != b.font && !font_neutral {
    return false;
}
```

## The fix

Two commits.

1. **`dp_lines.rs`: the font gate applies to word cells only** (`block_spaces`). Line cells now merge across a font change under the existing v1 geometry. The 0.33 × average-width gap still inserts a space where the runs are apart, and real space glyphs are kept, so `VERIFY_GROUP_FOR_USER ( SESSION_USER , 'HR', 'EMP' )` does not change. Word cells keep the gate. TableFormer matches against them, and dropping the gate there too changes a TOC cell (`Access Control.` where docling writes `Access Control .`), because docling-parse#361's TOC-leader rule is not ported. The comment that cited `Laboratories :` as docling behaviour is corrected. The `font` field doc, `MIGRATION.md` and `PDF_CONFORMANCE.md` now say the gate is word-only.

2. **`textparse.rs`: a blank cell is never overpaint.** Commit 1 alone breaks the text-layer path (`convert_text_layer`, `--text-layer-only`, wasm). On `table_mislabeled_as_picture.pdf`, a checkbox glyph, a space glyph and the option text now make one line cell, while the space glyph's own blank cell stays inside that cell's box. `drop_overpainted_cells` then reads the pair as glyph artwork and drops the whole line. The text-layer Markdown lost half its text (3105 → 1528 non-space characters). A blank cell has no ink, so it is now skipped. On `master` this guard also recovers text the text-layer path already loses the same way: `amt_handbook_sample` gains 1166 non-space characters of body text, and `normal_4pages` gains a Korean heading (`이슈와 논점`). The ML pipeline does not call this function.

This is not a port of the docling-parse 7.21/7.22 contractor (cursor gaps, one normalized space per word boundary, the TOC-leader rule). It removes the one rule docling-parse deleted and keeps the v1 geometry.

## How I tested

Windows 11, rustc 1.99.0, branch on `master` 33d3d6f (v2.3.0), without the models (tests that need them skip).

- `cargo test --locked --no-fail-fast --lib --tests -p docling-core -p docling -p docling-asr -p docling-serve -p docling-pdf`: SUITE_RESULTS
- `cargo fmt --all --check`: FMT_RESULT
- `cargo clippy -p docling-pdf --all-targets -- -D warnings` (1.99, not CI's 1.96): CLIPPY_RESULT
- New unit tests, each failing without its change:
  - `dp_lines::tests::font_change_does_not_split_a_line_but_splits_words`. With the gate line reverted it fails (GATE_REVERTED).
  - `textparse::overpainted::a_blank_cell_inside_a_line_is_not_paint`. Without the guard both cells are dropped.
- Not run: `pdf_conformance.sh` and `pdf_groundtruth.sh` (they need the models and the docling-parse shim), the wasm, MSRV and GPU legs. SNAPSHOT_NOTE

**Line cells against docling-parse.** For the 28 PDFs in `tests/data/pdf/sources` and `tests/data/odf/sources`, I dumped `textparse::pdf_textlines` on `master` and on this branch. I compared them per page with `textline_cells` from docling-parse 7.20.0 and 7.22.1. Line cells change in 17 PDFs. A token that is on the branch but not on `master` is a new join:

| | tokens |
|---|---:|
| newly joined by this change | 263 (in 9 PDFs) |
| also in docling-parse 7.22.1 | **251** |
| not in 7.22.1 | 12 |

The 12 that differ:
- 6 differ only in characters: five ORCID ids in 2305.03393v1 (7.22.1 joins them too, but writes ASCII `-`), and `~10%` in 2206 (7.22.1: `tildelow10%`).
- 1 is `-2-5` in right_to_left_03, the same as 7.20.0.
- 1 is a TOC leader, `Control.` (redp5110 p. 2): docling-parse#361's rule, not ported.
- 4 are in display formulas in 2203.01017v2, pp. 6-7, for example `TEDS(Ta,Tb)` where 7.22.1 has `TEDS (Ta, Tb)`. Its cursor-gap rule finds gaps that the v1 ink-gap rule misses once fonts no longer separate the cells.

No prose word is joined to a neighbouring prose word. The joins that agree are a bold or italic label and its colon (`Laboratories:`), an italic word and its comma, parentheses around a run in another font (`(WRKFCNUSG)`), URLs split across runs (`ibm.com/redbooks`), and markup tokens.

**Text-layer Markdown** (`convert_text_layer`, no models), `master` vs this branch: 14 of the 28 files change (358 lines). The non-space characters are identical in every file, apart from the recovered text above and wrap hyphens that land differently (2206: 5 fewer, 2203: 3 more, 2305.03393v1: 1 more). Paragraphs that `master` breaks apart at each font change are now whole, for example redp5110 `... the use of resource security. If implemented properly, ...`.

**Full pipeline**, measured earlier on published 1.93.6 with commit 1 only. `dp_lines.rs` was the same as on `master` apart from #609's ligature check. The redp5110_sampled Markdown changed 9 lines: `(WRKFCNUSG)`, `(CHGFCNUSG)`, `(DSPFCNUSG)`, `security policy.`, `resource security.`, `Tables,` / `Definition.`, an author's e-mail address and its full stop, `ibm.com/redbooks`, and the footnote URL `http://www.ponemon.org/`. 8 of these are now equal to docling `main`'s groundtruth, and the 9th differs only in whitespace. 0 of 196 table cells and no heading changed. I have not re-run the full pipeline on `master`.

## Risk

- **Where joining across fonts could be wrong:** a bold label and its value set less than one average character width apart, for example in adjacent table columns, would now share a line cell. I found none in the corpus. The v1 adjacency gate (1.0 × average character width) still separates anything wider. Word cells, and so TableFormer cell matching, do not change.
- **Display math** (the 4 tokens above): docling-parse 7.22.1 spaces these; here they join.
- **Whitespace:** when two cells in different fonts merge across a real space glyph, v1 can emit two spaces (`CREATE PERMISSION  <permission name>`, and TOC lines such as `Notices  . . .` on the text-layer path). docling-parse 7.21+ emits one. This is cosmetic in rendered Markdown and in tokenized search.
- **Conformance numbers:** the committed `tests/data/pdf/groundtruth` copies are from docling v2.130 (docling-parse 7.20) and still print `( WRKFCNUSG )` and `Laboratories :`. `pdf_groundtruth.sh` will count the changed lines as new differences until those files are re-mirrored from docling ≥ v2.132.0. Python docling `main` already has the new text.
- **Snapshots likely to move** (line cells change): 2203.01017v2, 2206.01062, 2305.03393v1, 2305.03393v1-pg9, amt_handbook_sample, code_and_formula, multi_page, normal_4pages, redp5110_sampled, right_to_left_02, right_to_left_03, skipped_2pages, table_mislabeled_as_picture, odf_presentation_01.odp, text_document_01/02/03.odt.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
