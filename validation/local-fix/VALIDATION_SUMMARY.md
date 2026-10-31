# Validation Summary — Issue #871 (CBO qualified class call arguments)

- pyscn baseline commit: `0615b1a0df793ab5644a8cdc78a8d2a735e5cdaf` (HEAD; fix is uncommitted working-tree change)
- pygame-menu revision: `5acb8ff1fbe02b4fc6be3e5ea293e59a00ce1761` (short: `5acb8ff1`)
- Go version: go1.25.5 linux/amd64

## Lint

- Tool: golangci-lint **v2.15.0** (linux-amd64; built go1.27.0 from 25c5369a), checksum-verified against official v2.15.0 checksums (sha256 `9b7fe6a6bf6334b98decf8d86b11415ea7889afdabdd1959fdce23af2a9ebcea`)
- Command: `golangci-lint run --timeout=10m`
- Exit code: **0** — findings: **0 issues**

## Verification evidence

- RED focused: `go test -v -count=1 ./internal/analyzer -run '^TestCBOAnalyzer_QualifiedClassCallArguments'` → exit **1** (log: red-tests.log); after justified callee-control fix → red-tests-reviewed.log → still exit **1** (three bug failures + four passing controls)
- GREEN focused: same focused command → exit **0**, all **7 subtests PASS** (green-focused.log)
- GREEN full suite: `go test -v -count=1 -timeout 15m ./...` → exit **0** (green-full.log)
- GREEN vet: `go vet ./...` → exit **0** (green-vet.log)
- GREEN lint: `golangci-lint run --timeout=10m` → exit **0** (green-lint.log)
- GREEN whitespace: `git diff --check` → exit **0**

## Real-project before/after (pygame-menu @ 5acb8ff1…)

- Baseline build: `git archive HEAD | tar -x -C /tmp/bridge-builder/pyscn-baseline-src && cd /tmp/bridge-builder/pyscn-baseline-src && go build -o /tmp/bridge-builder/pyscn-baseline ./cmd/pyscn` → **0**
- Patched build: `go build -o /tmp/bridge-builder/pyscn-patched ./cmd/pyscn` → **0**
- Baseline analyze: `/tmp/bridge-builder/pyscn-baseline analyze --json --no-open --select cbo --output - /tmp/bridge-builder/pygame-menu > /tmp/bridge-builder/pygame-baseline.json 2> /tmp/bridge-builder/pygame-baseline.err` → **0**
- Patched analyze: `/tmp/bridge-builder/pyscn-patched analyze --json --no-open --select cbo --output - /tmp/bridge-builder/pygame-menu > /tmp/bridge-builder/pygame-patched.json 2> /tmp/bridge-builder/pygame-patched.err` → **0**
- Measured before/after: **Widget CBO 7 → 10** (added `pygame_menu.BaseImage`, `pygame_menu.widgets.DropSelect`, `pygame_menu.widgets.Frame`); Frame 6 → 7; WidgetManager 26 → 29; plus 8 other classes with +1 dependency additions (no removals, no class-set changes).

## Evidence directory contents

- `pygame-baseline.json`, `pygame-patched.json`, `pygame-baseline.err`, `pygame-patched.err`
- `baseline-tests.log`, `red-tests.log`, `red-tests-reviewed.log`, `green-focused.log`, `green-full.log`, `green-vet.log`, `green-lint.log`
- `BUG_SPEC.md` (copied workshop artifact)
