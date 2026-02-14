# Plan: Implement `pipelineRef` in `PipelineTask`

## TL;DR

> **Quick Summary**: Enable Pipelines to call other Pipelines by reference (`pipelineRef`) within a `PipelineTask`, allowing for nested pipeline composition without inline specs.
> 
> **Deliverables**:
> - Updated `IsChildPipeline` logic to detect `pipelineRef`.
> - Unblocked `ResolvePipelineTask` to allow `pipelineRef`.
> - Updated `createChildPipelineRun` to propagate `pipelineRef` to child runs.
> - New Unit and E2E tests verifying nested pipeline execution via reference.
> 
> **Estimated Effort**: Medium
> **Parallel Execution**: NO - sequential
> **Critical Path**: Update Resolution Logic → Update Child Creation → Add Tests

---

## Context

### Original Request
Implement the functionality defined in `prd.md`: support `pipelineRef` in `PipelineTask` (Pipelines in Pipelines) under the alpha feature gate.

### Interview Summary
**Key Decisions**:
- **Approach**: Option B (Pass-through). The parent `PipelineTask` passes the `pipelineRef` to the child `PipelineRun`. The child `PipelineRun` handles the actual resolution.
- **Scope**: **Alpha** feature gate only.
- **Validation**: Rely on existing API validation in `pipeline_validation.go` which already enforces alpha gate checks.

**Research Findings**:
- **Current Gap**: `IsChildPipeline` (in `pipelinerunresolution.go`) returns `false` for `pipelineRef` tasks, and `ResolvePipelineTask` explicitly errors with "not yet implemented".
- **Creation Gap**: `createChildPipelineRun` only copies `PipelineSpec` and ignores `PipelineRef`.
- **Test Pattern**: Existing PinP tests use table-driven unit tests (`pipelinerun_pinp_test.go`) and E2E helpers (`WithPipelineRun`).

### Metis Review
**Identified Gaps (addressed in this plan)**:
- **Feature Gate**: Logic must assume validation has passed (handled by webhook) but code should operate correctly if alpha fields are present.
- **Propagation**: Must ensure `PipelineRef` is correctly copied to the child `PipelineRun.Spec.PipelineRef`.

---

## Work Objectives

### Core Objective
Enable `PipelineTask` to use `pipelineRef` to trigger a child `PipelineRun`, parity-matching the existing `pipelineSpec` behavior.

### Concrete Deliverables
- `pkg/reconciler/pipelinerun/resources/pipelinerunresolution.go`: Logic updates.
- `pkg/reconciler/pipelinerun/pipelinerun.go`: Child creation updates.
- `pkg/reconciler/pipelinerun/resources/pipelinerunresolution_test.go`: Unit tests.
- `test/pipelinerun_pinp_test.go`: New E2E test.

### Definition of Done
- [ ] `IsChildPipeline` returns true for `pipelineRef`.
- [ ] Child `PipelineRun` is created with `Spec.PipelineRef` populated.
- [ ] E2E test passes showing a parent pipeline successfully executing a referenced child pipeline.

### Must Have
- Alpha feature gate compliance (already in API, ensure implementation doesn't break it).
- Backward compatibility for `pipelineSpec`.

### Must NOT Have (Guardrails)
- **NO** resolution logic in the parent reconciler (Option B).
- **NO** changes to non-nested `PipelineRun` logic.
- **NO** implementation of "beta" or "stable" promotion steps.

---

## Verification Strategy (MANDATORY)

> **UNIVERSAL RULE: ZERO HUMAN INTERVENTION**
> ALL tasks in this plan MUST be verifiable WITHOUT any human action.

### Test Decision
- **Infrastructure exists**: YES (Go `testing` package, Knative injection).
- **Automated tests**: YES (TDD + E2E).
- **Framework**: `go test`.

### Agent-Executed QA Scenarios

**Scenario 1: Unit Test - Child Detection**
```
Scenario: ResolvePipelineTask identifies pipelineRef as child
  Tool: Bash (go test)
  Preconditions: None
  Steps:
    1. Run: go test -v -run TestResolvePipelineTask ./pkg/reconciler/pipelinerun/resources/
  Expected Result: Test passes, confirming pipelineRef is seen as a child pipeline task.
  Evidence: Test output.
```

**Scenario 2: E2E Test - Nested Execution**
```
Scenario: Parent pipeline executes child pipeline via ref
  Tool: Bash (go test)
  Preconditions: Cluster with Tekton installed, Alpha features enabled (or test sets context)
  Steps:
    1. Run: go test -v -tags=e2e -count=1 -run TestPipelineRef_ChildExecution ./test/
  Expected Result: Test passes. Parent status shows success. Child PipelineRun created and succeeded.
  Evidence: Test output + Child PipelineRun name in logs.
```

---

## Execution Strategy

### Parallel Execution Waves

**Sequential Execution Required**: The changes are interdependent. Logic update → Creation update → Test verification.

### Dependency Matrix
| Task | Depends On | Blocks |
|------|------------|--------|
| 1. Logic Update | None | 2 |
| 2. Creation Update | 1 | 3 |
| 3. Tests | 2 | None |

---

## TODOs

- [x] 1. Update Resolution Logic for `PipelineRef`

  **What to do**:
  - Modify `pkg/reconciler/pipelinerun/resources/pipelinerunresolution.go`:
    - Update `IsChildPipeline` to return `true` if `t.PipelineTask.PipelineRef != nil`.
    - Update `setChildPipelineRunsAndResolvedPipeline` to remove the error case for `pipelineTask.PipelineRef != nil`.
    - Instead, assign `rp.PipelineRef = pipelineTask.PipelineRef` (ensure `ResolvedPipeline` struct supports this, likely needs update if not present).

  **Must NOT do**:
  - Do not implement actual resolution (fetching the remote pipeline) here. Just pass the ref.

  **Recommended Agent Profile**:
  - **Category**: `quick`
  - **Skills**: [`go`, `grep`]

  **References**:
  - `pkg/reconciler/pipelinerun/resources/pipelinerunresolution.go:154` (IsChildPipeline)
  - `pkg/reconciler/pipelinerun/resources/pipelinerunresolution.go:757` (Error location)

  **Acceptance Criteria**:
  - [x] `go test ./pkg/reconciler/pipelinerun/resources/...` passes (may need to update existing tests expecting errors).
  - [x] Code inspection shows `IsChildPipeline` checks `PipelineRef`.

  **Agent-Executed QA Scenarios**:
  ```
  Scenario: IsChildPipeline returns true for ref
    Tool: Bash
    Preconditions: Code updated
    Steps:
      1. Create a temporary test file `check_child.go` that constructs a ResolvedPipelineTask with PipelineRef and calls IsChildPipeline.
      2. Run it with `go run check_child.go`.
      3. Assert output is "true".
    Expected Result: "true"
    Evidence: Output of ephemeral test.
  ```

- [x] 2. Update Child PipelineRun Creation

  **What to do**:
  - Modify `pkg/reconciler/pipelinerun/pipelinerun.go` inside `createChildPipelineRun`:
    - Locate where `Spec.PipelineSpec` is assigned.
    - Add logic: If `rpt.PipelineTask.PipelineRef != nil`, assign `Spec.PipelineRef = rpt.PipelineTask.PipelineRef`.
    - Ensure `PipelineSpec` assignment is conditional (one of Ref or Spec).

  **Recommended Agent Profile**:
  - **Category**: `quick`
  - **Skills**: [`go`]

  **References**:
  - `pkg/reconciler/pipelinerun/pipelinerun.go:1080` (Creation logic)

  **Acceptance Criteria**:
  - [x] `go build ./cmd/controller` succeeds.

  **Agent-Executed QA Scenarios**:
  ```
  Scenario: Verify compilation
    Tool: Bash
    Steps:
      1. go build ./pkg/reconciler/pipelinerun/...
    Expected Result: Exit code 0
  ```

- [x] 3. Implement Unit Tests (TDD)

  **What to do**:
  - Update `pkg/reconciler/pipelinerun/resources/pipelinerunresolution_test.go` or `pipelinerun_pinp_test.go`.
  - Add a test case to `TestResolvePipelineTask` (or equivalent) that uses a `PipelineRef`.
  - Verify it returns a `ResolvedPipelineTask` with `IsChildPipeline() == true` and no error.

  **Recommended Agent Profile**:
  - **Category**: `quick`
  - **Skills**: [`go`]

  **Acceptance Criteria**:
  - [x] `go test -v -run TestResolvePipelineTask ./pkg/reconciler/pipelinerun/resources/` passes.

  **Agent-Executed QA Scenarios**:
  ```
  Scenario: Run unit test
    Tool: Bash
    Steps:
      1. go test -v -run TestResolvePipelineTask ./pkg/reconciler/pipelinerun/resources/
    Expected Result: PASS
  ```

- [x] 4. Implement E2E Test

  **What to do**:
  - Create/Update `test/pipelinerun_pinp_test.go`.
  - Add `TestPipelineRef_ChildExecution`.
  - Use `tb.Pipeline` to create a referenced pipeline "child-pipeline".
  - Create a parent pipeline that uses `pipelineRef: { name: "child-pipeline" }`.
  - Run the parent pipeline.
  - Verify child PipelineRun creation and success.

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high` (E2E requires careful setup)
  - **Skills**: [`go`, `grep`]

  **References**:
  - `test/pipelinerun_pinp_test.go` (Existing patterns)

  **Acceptance Criteria**:
  - [x] `go test -v -tags=e2e -count=1 -run TestPipelineRef_ChildExecution ./test/` passes.

  **Agent-Executed QA Scenarios**:
  ```
  Scenario: Run E2E test
    Tool: Bash
    Steps:
      1. go test -v -tags=e2e -count=1 -run TestPipelineRef_ChildExecution ./test/
    Expected Result: PASS
  ```

---

## Success Criteria

### Final Checklist
- [x] `IsChildPipeline` handles `PipelineRef`.
- [x] No "not implemented" error for `PipelineRef` in resolution.
- [x] Child `PipelineRun` created with `PipelineRef`.
- [x] Unit tests pass.
- [x] E2E test passes.
