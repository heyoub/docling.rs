# How to post the upstream pull request (not executed)

Nothing below has been run. Every command is for the owner to run by hand, from the
directory that holds this file and `PR.md`. The clone `docling.rs/` in that directory
has the local branch `fix/pdf-font-change-line-cells` (one commit on upstream `master`
33d3d6f, v2.3.0) and the remote `fork` (the owner's fork).

## 0. Before posting

1. Decide the open points in the report (`upstream-pr-report.md`, section "Owner
   decisions"): sign-off identity, the AI trailers, and whether to refresh the PDF
   snapshots first.
2. Upstream moves several times a day. Bring the branch up to date with a merge (no
   rebase needed for an unpublished branch, but a merge keeps the reviewed commit as is):

   ```sh
   cd docling.rs
   git fetch origin
   git checkout fix/pdf-font-change-line-cells
   git merge --no-ff origin/master        # only if master moved; resolve, re-test
   ```

   If `crates/docling-pdf/src/dp_lines.rs` conflicts again, it is most likely another
   test appended at the end of `mod tests`; keep both tests.
3. Re-run the checks the PR description claims (PowerShell on Windows; set
   `CARGO_BUILD_JOBS` / `CARGO_TARGET_DIR` as usual):

   ```powershell
   cargo fmt --all --check
   cargo clippy -p docling-pdf --all-targets -- -D warnings
   cargo test --locked --lib --tests -p docling-core -p docling -p docling-asr -p docling-serve -p docling-pdf
   ```

4. Optional but expected upstream (CONTRIBUTING: "A change that moves conformance numbers
   must say so ... and update docs ... with the real measured numbers"): refresh the PDF
   snapshots on Linux (WSL works) with the models and the docling-parse render shim, as a
   second commit `test(pdf): refresh snapshots for the font-run line cells`:

   ```sh
   scripts/install/download_dependencies.sh --with-docling-parse
   git checkout --detach origin/master
   scripts/conformance/pdf_conformance.sh        # baseline: expect every fixture EXACT
   scripts/conformance/pdf_groundtruth.sh        # baseline groundtruth numbers
   git checkout fix/pdf-font-change-line-cells
   scripts/conformance/pdf_conformance.sh        # lists the snapshots this change moves
   # regenerate them with the same environment the script exports
   # (DOCLING_RS_SLOW_RESIZE=1, DOCLING_RS_RENDERER=docling-parse, the model paths):
   cargo run --release -q -p docling-pdf --example snapshot -- tests/data tests/snapshots
   git diff --stat tests/snapshots               # review every changed line
   scripts/conformance/pdf_groundtruth.sh        # after numbers
   ```

   Commit only the snapshots this change moves; if any other snapshot drifts on the
   baseline run too, it is not ours and stays out of the commit. Put both scripts'
   before/after numbers into `PR.md`, and the groundtruth numbers into
   `docs/PDF_CONFORMANCE.md` where it lists them.

   Then edit the "How I tested" section of `PR.md` to match.

## 1. Amend the commit if a decision changes it (local, unpublished, so amend is fine)

```sh
cd docling.rs
# a different sign-off identity, for example the GitHub noreply address:
git -c user.email=<address-you-choose> commit --amend --no-edit -s --reset-author
# drop the AI trailers, if you decide so: edit the message
git commit --amend
```

Check the result: `git log -1 --format='%an%n%B'`.

## 2. Push the branch to your fork

```sh
cd docling.rs
git push fork fix/pdf-font-change-line-cells
```

## 3. Open the pull request against upstream

`PR.md` holds the title on its first line (after `Title: `) and the body after the
first `---` line. Extract both and open the PR:

```sh
title=$(sed -n '1s/^Title: //p' PR.md)
sed '1,/^---$/d' PR.md > pr-body.md
gh pr create --repo docling-project/docling.rs \
  --base master \
  --head Heyoub:fix/pdf-font-change-line-cells \
  --title "$title" \
  --body-file pr-body.md
```

Add `--draft` to open it as a draft first.

## 4. After it is merged and released

okf-jawn can drop the `[patch.crates-io]` git dependency on the fork once a published
`docling-pdf` contains the change and okf-jawn's `docling` requirement reaches that
version. That is an integration-owner change to manifests and lockfiles, followed by
re-running `bun qualification/docling/run.mjs`.
