# PRD: Support `pipelineRef` in `PipelineTask` (Pipelines in Pipelines)

- Status: Draft
- Date: 2026-02-12
- Author: Draft prepared in-repo from current controller/API behavior
- Target: Tekton Pipelines `tekton.dev/v1` controller behavior

## 1. Summary
Enable `PipelineTask.pipelineRef` so a Pipeline can execute another Pipeline by reference (local name or resolver-based reference), not only by inline `pipelineSpec`.

This PRD proposes an alpha-scoped implementation that preserves existing API contracts and unlocks nested pipeline composition using referenced Pipelines.

## 2. Background and Current State

### 2.1 What exists today
- `PipelineTask` already defines both `pipelineRef` and `pipelineSpec` fields in API types (`pkg/apis/pipeline/v1/pipeline_types.go:245`, `pkg/apis/pipeline/v1/pipeline_types.go:258`).
- Validation already accepts `pipelineRef`/`pipelineSpec` under alpha API fields and enforces one-of semantics with task fields (`pkg/apis/pipeline/v1/pipeline_validation.go:302`).
- Pipelines-in-Pipelines reconciliation is implemented for child PipelineRuns created from `pipelineSpec` (see child PipelineRun tests in `pkg/reconciler/pipelinerun/pipelinerun_pinp_test.go:23`).

### 2.2 Current gaps
- Child-pipeline detection only checks `pipelineSpec` (`pkg/reconciler/pipelinerun/resources/pipelinerunresolution.go:154`).
- `PipelineTask.pipelineRef` is explicitly rejected in child pipeline resolution (`pkg/reconciler/pipelinerun/resources/pipelinerunresolution.go:757`).
- Non-child resolution path surfaces an explicit unsupported error for `pipelineRef` tasks (`pkg/reconciler/pipelinerun/resources/pipelinerunresolution.go:843`).
- Child PipelineRun creation currently writes only `spec.pipelineSpec` (`pkg/reconciler/pipelinerun/pipelinerun.go:1080`).
- Unit test coverage confirms current failure for `PipelineTask.pipelineRef` (`pkg/reconciler/pipelinerun/resources/pipelinerunresolution_test.go:2731`).

### 2.3 Documentation mismatch
- Docs currently state that both `pipelineRef` and `pipelineSpec` in `PipelineTask` are not yet supported (`docs/pipelines-in-pipelines.md:20`), while controller tests show working `pipelineSpec` behavior.

## 3. Problem Statement
Pipeline authors can compose nested Pipelines only with inline `pipelineSpec`, which forces duplication and prevents reuse/versioning via named or remotely resolved Pipeline references.

Result: teams cannot use a referenced Pipeline as a reusable orchestration unit inside another Pipeline, despite API-level affordances.

## 4. Goals
1. Support `pipelineRef` in `PipelineTask` and `finally` tasks under alpha API fields.
2. Create child `PipelineRun` resources for `pipelineRef` tasks with behavior parity to existing child-run orchestration.
3. Support both local named references and resolver-based references through existing `PipelineRun` reference machinery.
4. Preserve backward compatibility for existing `taskRef`, `taskSpec`, and `pipelineSpec` behavior.
5. Remove unsupported/internal errors currently returned for `PipelineTask.pipelineRef`.

## 5. Non-Goals
- Promoting Pipelines-in-Pipelines to beta/stable in this work.
- Redesigning PipelineTask schema or adding new CRD fields.
- Broad refactor of nested workspace semantics beyond current behavior.
- Changing top-level `PipelineRun.spec.pipelineRef` behavior.

## 6. Users and Use Cases
- Platform teams maintaining shared Pipeline libraries (local named Pipelines).
- Multi-repo organizations using resolver-based Pipelines (e.g., git resolver).
- Pipeline authors who need nested composition without embedding large inline specs.

## 7. Product Requirements

### PR-1: PipelineTask reference acceptance
When `enable-api-fields=alpha`, a `PipelineTask` with `pipelineRef` must be treated as a child pipeline task (same orchestration class as current child `pipelineSpec` tasks).

### PR-2: Child PipelineRun creation from `pipelineRef`
For each runnable `PipelineTask` using `pipelineRef`, the controller must create a child `PipelineRun` with:
- ownership, labels, annotations, and child-reference tracking consistent with existing child pipeline runs.
- `spec.pipelineRef` populated from the parent `PipelineTask.pipelineRef`.

### PR-3: Resolver compatibility
`pipelineRef` in child tasks must support:
- local named pipeline references.
- resolver references (`resolver` + `params`) using existing `PipelineRun` resolution flow in the child run.

### PR-4: Runtime behavior and status
Parent `PipelineRun` status must continue to:
- track child references for nested pipelines.
- gate downstream task execution based on child run completion/failure.
- surface child failures through existing parent-state evaluation.

### PR-5: Validation and compatibility
- Keep one-of validation semantics (`taskRef|taskSpec|pipelineRef|pipelineSpec`) unchanged.
- Keep alpha gate enforcement unchanged.
- Existing `pipelineSpec` nested behavior must remain intact.

### PR-6: Error semantics
- Remove current unsupported-path errors for `PipelineTask.pipelineRef`.
- For reference resolution failures, rely on child `PipelineRun` resolution lifecycle and reasons (rather than parent internal errors).

## 8. Approach Options

### Option A: Resolve `pipelineRef` in parent, materialize `pipelineSpec` in child
Pros:
- Parent has early visibility into resolved spec.
- Child runs remain spec-materialized.

Cons:
- Larger parent-controller changes.
- Duplicates existing child `PipelineRun` reference-resolution capability.
- More coupling to resolver/trusted-resource paths.

### Option B (Recommended): Pass through `pipelineRef` to child `PipelineRun`
Pros:
- Reuses existing `PipelineRun` pipelineRef resolution path.
- Smaller, lower-risk changes in parent reconciliation.
- Aligns with current top-level reference semantics.

Cons:
- Parent does not resolve child pipeline pre-creation.
- Failure visibility occurs via child run status progression.

### Option C: Hybrid (resolve in parent for local refs, pass through resolver refs)
Pros:
- Potentially faster local failures.

Cons:
- Split behavior model.
- Higher maintenance complexity and test burden.

Recommendation: **Option B** for MVP.

## 9. Detailed Design Requirements (MVP)

### 9.1 Controller state classification
- Update child-pipeline classification to include `PipelineTask.pipelineRef` in addition to `pipelineSpec`.
- Ensure resolution path does not route `pipelineRef` tasks through Task resolution.

### 9.2 Child run construction
- When parent task uses `pipelineSpec`: keep current behavior.
- When parent task uses `pipelineRef`: create child `PipelineRun` using `spec.pipelineRef`.

### 9.3 Execution semantics
- Child run naming and childReferences must continue to use existing deterministic naming and tracking.
- Existing DAG dependency behavior and `when`/`onError` handling remain unchanged.

### 9.4 Failure and retry behavior
- If child Pipeline reference cannot be resolved, failure reason should be expressed by child run resolution conditions and propagated via parent child-state evaluation.
- No panic or generic internal-error path for supported `pipelineRef` task type.

## 10. Rollout Plan

### Phase 1: Alpha implementation
- Enable `PipelineTask.pipelineRef` orchestration in controller.
- Add/adjust unit tests for resolution, state classification, and child run creation.
- Add integration/e2e coverage for local and resolver-based child references.

### Phase 2: Documentation alignment
- Update `docs/pipelines-in-pipelines.md` and `docs/pipelines.md` to reflect actual support matrix.
- Add examples for nested `pipelineRef` usage.

## 11. Acceptance Criteria
1. A Pipeline with a `PipelineTask.pipelineRef.name` (alpha enabled) creates and tracks a child `PipelineRun` instead of failing with unsupported errors.
2. A Pipeline with `PipelineTask.pipelineRef.resolver` creates a child `PipelineRun` that enters existing reference-resolution lifecycle.
3. Existing nested `pipelineSpec` scenarios continue to pass unchanged.
4. Existing one-of validation errors remain unchanged for invalid combinations.
5. Tests no longer expect unsupported `PipelineTask.pipelineRef` errors.
6. Documentation no longer states that nested `pipelineSpec` is unsupported, and clearly documents nested `pipelineRef` support level.

## 12. Test Strategy
- Unit tests:
  - child pipeline detection for `pipelineRef` and `pipelineSpec`.
  - `ResolvePipelineTask` behavior for `pipelineRef` tasks.
  - child `PipelineRun` spec fields (`pipelineRef` vs `pipelineSpec`) based on task type.
- Reconciler tests:
  - parent status transitions with child runs from `pipelineRef`.
  - failure propagation from unresolved/nonexistent child pipeline refs.
- Integration/e2e:
  - local named nested pipeline reference.
  - resolver-based nested pipeline reference.

## 13. Risks and Mitigations
- Risk: behavior drift between nested `pipelineSpec` and nested `pipelineRef` paths.
  - Mitigation: explicit parity tests for child-run ownership, labels, status, and dependency handling.
- Risk: documentation confusion due to stale feature statements.
  - Mitigation: docs update in same delivery window.
- Risk: resolver/verification expectations in parent vs child context.
  - Mitigation: keep resolution in child run (recommended option), reuse existing lifecycle.

## 14. Open Questions
1. Should parent-level events include a dedicated message when creating a child run from `pipelineRef` (for observability)?
2. Do we want explicit conformance tests for nested resolver timeouts/retries in this milestone, or as follow-up?
3. Should this feature continue as alpha-only until nested workspace/parameter parity is fully characterized?

## 15. Out of Scope Follow-ups
- Workspace forwarding enhancements for nested pipelines.
- Broader Pipelines-in-Pipelines UX polish and promotion beyond alpha.
