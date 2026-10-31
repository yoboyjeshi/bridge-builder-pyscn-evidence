# Bridge Builder: pyscn bug investigation

This repository documents my independent local fix for
[pyscn issue #871](https://github.com/ludo-technologies/pyscn/issues/871)
and subsequent validation of another contributor's
[PR #873](https://github.com/ludo-technologies/pyscn/pull/873).

**I did not author PR #873.** After discovering the overlapping PR,
I validated it rather than submit a duplicate fix.
Course acceptance of this alternative submission is not yet confirmed.

## The bug

CBO (Coupling Between Objects) analysis skipped qualified class references
passed directly as call arguments, while tuple-wrapped references counted.

## My work

- Diagnosed the bug and wrote [BUG_SPEC.md](BUG_SPEC.md).
- Developed a local fix and seven regression cases, saved in [patches](patches/).
- Collected failing-before/passing-after and real-project evidence.
- Validated the overlapping PR using the additional tests.

## PR validation results

- PR regression test and all seven added cases passed.
- Full test suite and go vet passed on the unmodified PR head.
- golangci-lint v2.15.0 reported zero issues with the added tests.
- Both real-project analyses exited 0 and produced valid JSON.
- Both analysed the same 103 classes with identical configuration.
- Eleven classes gained dependencies; none lost dependencies.

Widget CBO increased from **7 to 10**, adding:
- pygame_menu.BaseImage
- pygame_menu.widgets.DropSelect
- pygame_menu.widgets.Frame

WidgetManager increased from 26 to 29; Frame increased from 6 to 7.
Decorator retained CBO 4 and its dependency set, but its attribute-access
dependency count increased from 2 to 3. Thus 12 classes had metric changes.
Both analyses reported zero errors, zero failures and 12 warnings.

## Exact revisions

- Baseline: `0615b1a0df793ab5644a8cdc78a8d2a735e5cdaf`
- PR head: `ce64e8879dcf643bbd6f779f433261735c6e1b16`
- pygame-menu fixture: `5acb8ff1fbe02b4fc6be3e5ea293e59a00ce1761`

Results apply to these revisions, not later updates.

## Evidence

- [Original local-fix evidence](validation/local-fix/)
- [PR tests and static checks](validation/pr-873/checks/)
- [Real-project comparison](validation/pr-873/real-project/dependency-comparison.txt)

Local-fix evidence and PR-validation evidence are separate.
Machine paths were sanitised in this copy; originals were retained.
Upstream MIT licence text is preserved in attribution/upstream/LICENSE.

AI assistance supported investigation, code/test guidance, validation and
documentation. Commands were run in my Codio environment and outputs reviewed.
This repository does not represent an upstream PR authored by me.
