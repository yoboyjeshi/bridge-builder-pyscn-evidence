# Bug Specification (`BUG_SPEC.md`)

**Issue Reference:** https://github.com/ludo-technologies/pyscn/issues/871
**Investigator:** OpenCode agent — bridge-builder workshop
**Date:** 2026-10-31

---

## 1. Current Behavior

- **Observed Outcome:** A module-qualified class reference that is passed directly as a positional argument to a call is omitted from the CBO dependency set of the referencing class, so the class is reported with no dependent classes and a coupling count of zero. The very same reference wrapped in a tuple is counted, producing an asymmetric result.
- **Failure Symptoms:** CBO under-reports class coupling whenever a class reference reaches the analyzer through a direct positional call argument (the shape produced by `isinstance` checks and by callbacks that forward class objects). The coupling score, risk level, and dependent-class lists are affected.
- **Observed Logs / Output:** Reproduction results observed during live diagnosis (see section 4 for the full fixture and command).

### Observed reproduction results (before any fix)

| Case | Class | CBO | Dependent classes | Expected |
|---|---|---|---|---|
| Direct positional class reference | `Direct` | 0 | `[]` | 1 / class ref |
| Tuple-wrapped class reference | `Wrapped` | 1 | `['pkg.models.Frame']` | 1 / class ref |
| Class passed as argument to ordinary function | `Passed` | 0 | `[]` | 1 / class ref |

Only the tuple-wrapped case produced a dependency; the direct positional case and the function-argument case did not.

## 2. Expected Behavior

- Direct positional, tuple-wrapped, and keyword references to the same eligible class must produce the same dependency identity and count.
- Repeated references to the same class must not increase the count (the dependency set is a set; one edge per class).
- Existing callee handling and non-class exclusions must be preserved.

## 3. Target Area

- **Target File(s):** `internal/analyzer/cbo.go`, `internal/parser/ast.go`, `internal/parser/ast_builder.go`
- **Target Function / Method:** `CBOAnalyzer.shouldSkipAttributeAccess`; `CBOAnalyzer.analyzeInstantiationAndAccess`; `CBOAnalyzer.callDependencyName`; `CBOAnalyzer.extractClassNameFromCallNode`; `parser.Node.RefreshParentLinks`; `parser.Node.OrderedChildren`; `ASTBuilder.buildCall`; `ASTBuilder.buildTuple`; `ASTBuilder.buildCallArguments`
- **Suspect Line Range:** `internal/analyzer/cbo.go` around lines 934-945 for `shouldSkipAttributeAccess`, with call sites around lines 413-476 in `analyzeInstantiationAndAccess`.
- **Root Cause Analysis:** The CBO dependency walk treats an attribute-shaped class reference as a potential callee of a call and skips it based on the type of its parent node. The skip predicate conflates "the attribute is the callee being invoked" with "the attribute appears somewhere inside a call". The parser gives both genuine callees and positional arguments a parent of type call, so the broader skip removes legitimate argument references while still allowing tuple-wrapped references (whose parent is a tuple node) through. The intent of the skip — to avoid re-handling callee attributes in the attribute-access pass because the call pass already owns callee extraction — was not expressed in terms of the callee node itself. The broken assumption was that a parent of type call is sufficient to identify a callee; it is also the parent of legitimate positional arguments.

## 4. Reproduction Case

- **Entry Point / Trigger:** `pyscn analyze` with CBO selected, via the Go CLI (`go run ./cmd/pyscn analyze ...`).
- **Input Data / Fixture:** A scratch Python package with a module holding two classes and a consumer module holding three classes:
  - Module with `class Frame` and `class Other`.
  - Consumer module with `Direct` doing `isinstance(x, pkg.models.Frame)`, `Wrapped` doing `isinstance(x, (pkg.models.Frame,))`, and `Passed` doing `f(pkg.models.Other)`.
- **Exact Reproduction Fixture:**
  - File: `pkg/models.py` — full contents:

    ```python
    class Frame:
        pass


    class Other:
        pass
    ```

  - File: `pkg/user.py` — full contents:

    ```python
    import pkg.models


    class Direct:
        def check(self, x):
            return isinstance(x, pkg.models.Frame)


    class Wrapped:
        def check(self, x):
            return isinstance(x, (pkg.models.Frame,))


    class Passed:
        def check(self, f):
            return f(pkg.models.Other)
    ```

  - Configuration file: `.pyscn.toml` — content:

    ```toml
    [cbo]
    show_zeros = true
    ```

  - Runnable command: `go run ./cmd/pyscn analyze --json --no-open --select cbo /tmp/bridge-builder/cbo-repro`

- **Reproduction Steps:**
  1. Parse and analyze the consumer module with CBO enabled and zero-count classes visible.
  2. Inspect per-class CBO results for `Direct`, `Wrapped`, and `Passed`.
  3. Compare with the tuple-wrapped variant of the same reference.
- **Deterministic Repro Command:**
  ```bash
  go run ./cmd/pyscn analyze --json --no-open --select cbo <fixture-path>
  ```
- **Configuration:** A `[cbo]` section with `show_zeros = true` so that classes reporting zero coupling are still emitted.
- **Actual vs Expected:** Actual (observed): `Direct` → 0 with no dependents; `Wrapped` → 1 with `pkg.models.Frame`; `Passed` → 0 with no dependents. Expected: each of `Direct`, `Wrapped`, and `Passed` to report coupling count 1 on the referenced class.
- **Cases not yet tested:** Keyword-argument references, callee-only qualified references, repeated references, and non-class argument exclusions were identified during reconnaissance but not yet exercised against the live tool during this session. The observed `Passed` case already covers a qualified class passed to an ordinary function, so ordinary-function argument forwarding is covered by observation rather than listed as untested.

## 5. Acceptance Tests

Regression coverage required in `internal/analyzer/cbo_test.go`:

- A qualified class passed directly to `isinstance` must be counted as a dependency.
- A qualified class passed directly to an ordinary function must be counted as a dependency.
- The same class wrapped in a tuple must still be counted, with no duplicate count for the same class.
- A keyword argument using an ordinary function, `f(cls=...)`, must be handled consistently.
- A qualified class used as the callee must be counted exactly once.
- Repeated references to the same class must be counted once.
- Existing exclusions for non-class references must be preserved (function calls, module attributes, builtins).

Note: keyword arguments must not be used with `isinstance`, because Python's `isinstance` does not accept keyword arguments.

## 6. Scope and Non-goals

- Production changes limited to `shouldSkipAttributeAccess` unless tests establish a necessary reason to revisit scope.
- Tests belong in `internal/analyzer/cbo_test.go`.
- No new dependencies, API changes, parser refactors, or unrelated cleanup.
- Bare-name positional references such as `f(MyClass)` are outside this fix; record them separately rather than expanding scope.
- Files inspected for diagnosis only (read-only context; not edit targets): `internal/parser/ast.go`, `internal/parser/ast_builder.go`. The parser files are diagnostic context for this fix and are not permitted to change under this specification.
- Files permitted to change: `internal/analyzer/cbo.go` (production fix target) and `internal/analyzer/cbo_test.go` (tests).

## 7. Verification Plan

- Preserve the completed baseline log (`go test -v -timeout 15m ./...` completed with exit code 0; log at `../baseline-tests.log`).
- Demonstrate a regression test failing before the production fix, then passing after it.
- Run the full test suite and repository lint checks.
- Verify before/after output against `ppizarror/pygame-menu` at the revision identified in the issue. Treat its reported numbers as hypotheses until independently measured.
