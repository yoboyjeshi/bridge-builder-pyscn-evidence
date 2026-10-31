# Reproduction instructions

## Environment

Application builds/tests used Go 1.25.5 on Linux amd64.
A C compiler is required for tree-sitter.
Lint used golangci-lint v2.15.0; see the saved lint-version.txt.

Use fresh, separate pyscn checkouts at these revisions:
- Baseline: `0615b1a0df793ab5644a8cdc78a8d2a735e5cdaf`
- PR head: `ce64e8879dcf643bbd6f779f433261735c6e1b16`

Use ppizarror/pygame-menu at:
`5acb8ff1fbe02b4fc6be3e5ea293e59a00ce1761`

Paths below labelled PATCH or BINARY are placeholders to replace.

## Original local fix: red then green

In a fresh baseline checkout, check and apply only the test patch:

    git apply --check TEST_PATCH
    git apply TEST_PATCH
    go test -v -count=1 ./internal/analyzer -run '^TestCBOAnalyzer_QualifiedClassCallArguments$'

TEST_PATCH is patches/original-regression-tests.patch in this repository.
The saved red evidence reports three failing cases and four passing controls.

Then check and apply patches/original-production-fix.patch:

    git apply --check PRODUCTION_PATCH
    git apply PRODUCTION_PATCH

Rerun the focused test above. All seven cases should pass.

## Unmodified PR head

In a fresh checkout at the recorded PR revision:

    go mod download
    go mod verify
    go test -v -count=1 ./internal/analyzer -run '^TestCBOAnalyzer_DottedClassReferenceAsCallArgument$'
    go test -v -count=1 -timeout 15m ./...
    go vet ./...

Then apply only TEST_PATCH using git apply --check followed by git apply.
Do not apply the local production fix to the PR checkout.

Run combined coverage and lint:

    go test -v -count=1 ./internal/analyzer -run '^TestCBOAnalyzer_(QualifiedClassCallArguments|DottedClassReferenceAsCallArgument)$'
    golangci-lint run --timeout=10m

The keyword fixture is parser input, not executable Python.
The non-class control covers ordinary calls, not non-class arguments.

## Real-project comparison

Build each binary from its own baseline or PR source directory:

    go build -o BINARY ./cmd/pyscn

Use distinct absolute binary paths. In a fresh pinned pygame-menu
checkout, create .pyscn.toml with:

    [cbo]
    show_zeros = true

From that fixture directory, run each binary against the same absolute
fixture path and configuration:

    BINARY analyze --json --no-open --select cbo --output - FIXTURE_PATH > result.json 2> result.err

Save each exit code immediately with echo $? and use separate output
files for baseline and PR. Both recorded analyses exited zero.

Compare cbo.classes keyed by file_path, name and start_line.
Compare dependent_classes as sets; do not rely on list ordering.
Also compare coupling_count, category counts, configurations and diagnostics.

Expected observations: same 103 classes; Widget CBO 7 to 10;
11 classes gain dependencies, none lose any. Decorator retains CBO 4
but its attribute-access dependency count increases from 2 to 3.
Both recorded runs have zero errors, zero failures and 12 warnings.

See validation/pr-873/real-project/dependency-comparison.txt.
These are scoped observations, not proof that every possible case is correct.
