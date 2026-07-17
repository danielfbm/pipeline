<!--
---
linkTitle: "Pipelines in Pipelines"
weight: 406
---
-->

# Pipelines in Pipelines

- [Overview](#overview)
- [Specifying `pipelineRef` in `Tasks`](#specifying-pipelineref-in-pipelinetasks)
- [Specifying `pipelineSpec` in `Tasks`](#specifying-pipelinespec-in-pipelinetasks)
- [Specifying `Parameters`](#specifying-parameters)
- [Specifying `Workspaces`](#specifying-workspaces)
- [Consuming `Results` from a child Pipeline](#consuming-results-from-a-child-pipeline)
- [Known Limitations](#known-limitations)

## Overview

A mechanism to define and execute Pipelines in Pipelines, alongside Tasks and Custom Tasks, for a more in-depth background and inspiration, refer to the proposal [TEP-0056](https://github.com/tektoncd/community/blob/main/teps/0056-pipelines-in-pipelines.md "Proposal").

> :seedling: **Pipelines in Pipelines is an [alpha](additional-configs.md#alpha-features) feature.**
> The `enable-api-fields` feature flag must be set to `"alpha"` to specify `pipelineRef` or `pipelineSpec` in a `pipelineTask`.
> When enabled, a `PipelineTask` that references a child `Pipeline` will produce a child `PipelineRun` owned by the parent `PipelineRun`.
> Recursive `pipelineRef` references are detected by walking the parent `PipelineRun` owner chain and fail fast with a clear error.
> See [Known Limitations](#known-limitations) for the current caveats.

## Specifying `pipelineRef` in `pipelineTasks`

Defining Pipelines in Pipelines at authoring time, by either specifying `PipelineRef` or `PipelineSpec` fields to a `PipelineTask` alongside `TaskRef` and `TaskSpec`.

For example, a Pipeline named security-scans which is run within a Pipeline named clone-scan-notify where the PipelineRef is used:

```
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: security-scans
spec:
  tasks:
    - name: scorecards
      taskRef:
        name: scorecards
    - name: codeql
      taskRef:
        name: codeql
---
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: clone-scan-notify
spec:
  tasks:
    - name: git-clone
      taskRef:
        name: git-clone
    - name: security-scans
      pipelineRef:
        name: security-scans
    - name: notification
      taskRef:
        name: notification
```

## Specifying `pipelineSpec` in `pipelineTasks`

The `pipelineRef` [example](#specifying-pipelineref-in-pipelinetasks) can be modified to use PipelineSpec instead of PipelineRef to instead embed the Pipeline specification:
```
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: clone-scan-notify
spec:
  tasks:
    - name: git-clone
      taskRef:
        name: git-clone
    - name: security-scans
      pipelineSpec:
        tasks:
          - name: scorecards
            taskRef:
              name: scorecards
          - name: codeql
            taskRef:
              name: codeql
    - name: notification
      taskRef:
        name: notification
```

## Specifying `Parameters`

Pipelines in Pipelines consume Parameters in the same way as Tasks in Pipelines
```
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: clone-scan-notify
spec:
  params:
    - name: repo
      value: $(params.repo)
  tasks:
    - name: git-clone
      params:
        - name: repo
          value: $(params.repo)      
      taskRef:
        name: git-clone
    - name: security-scans
      params:
        - name: repo
          value: $(params.repo)
      pipelineRef:
        name: security-scans
    - name: notification
      taskRef:
        name: notification
```

## Specifying `Workspaces`

A `PipelineTask` that references a child `Pipeline` can map parent workspaces to the child's declared workspaces the same way it does for `Tasks`. The bindings declared on the `PipelineTask` are propagated to the child `PipelineRun`, which then forwards them to the child `Pipeline`'s workspaces.

```yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: security-scans
spec:
  workspaces:
    - name: source
  tasks:
    - name: scorecards
      workspaces:
        - name: source
      taskRef:
        name: scorecards
---
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: clone-scan-notify
spec:
  workspaces:
    - name: shared-ws
  tasks:
    - name: git-clone
      workspaces:
        - name: output
          workspace: shared-ws
      taskRef:
        name: git-clone
    - name: security-scans
      runAfter:
        - git-clone
      workspaces:
        - name: source
          workspace: shared-ws
      pipelineRef:
        name: security-scans
```

## Consuming `Results` from a child Pipeline

`Results` declared by a child `Pipeline` (in its `spec.results`) propagate to the parent
(see the [Results section of TEP-0056](https://github.com/tektoncd/community/blob/main/teps/0056-pipelines-in-pipelines.md#results)),
so they can be consumed like the results of a regular `PipelineTask`:

- Other `PipelineTasks` (including `finally` tasks) can consume them in `params` and `when`
  expressions via `$(tasks.<pipelineTaskName>.results.<resultName>)`, including array
  indexing (`...results.<resultName>[i]`) and object keys (`...results.<resultName>.<key>`).
- The parent `Pipeline`'s `results` can reference them, aggregating them into the parent
  `PipelineRun`'s `status.results`.

For example, the parent `Pipeline` below deploys with the image digest produced by the child
`Pipeline`, and surfaces it as one of its own `results`:

```yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: build
spec:
  results:
    - name: digest
      value: $(tasks.build-image.results.digest)
  tasks:
    - name: build-image
      taskRef:
        name: build-image
---
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: build-and-deploy
spec:
  results:
    - name: digest
      value: $(tasks.build.results.digest)
  tasks:
    - name: build
      pipelineRef:
        name: build
    - name: deploy
      params:
        - name: digest
          value: $(tasks.build.results.digest)
      taskRef:
        name: deploy
```

As with `Task` results, referencing a result that the child `Pipeline` declares but fails to
produce causes the consuming `PipelineTask` to fail result validation at runtime, and a parent
`Pipeline` result referencing it is omitted from `status.results`.

## Known Limitations

The initial alpha implementation has the following limitations. These are expected to be addressed in follow-up work.

### Other current limitations

- Per-`PipelineTask` `timeout` and `retries` are not applied to the child `PipelineRun`. The child runs with its own default timeouts and is created at most once per parent reconcile.
- The parent `PipelineRun.Spec.Timeouts` is not propagated to the child `PipelineRun`; the child uses its own defaults.
- `pipelineRef` cycle detection is best-effort and runs at child reconcile time: it walks the `ownerReferences` chain and matches the `tekton.dev/pipeline` label, so cycles are caught when the offending child is reconciled rather than at parent submission.
- Validation of non-optional child `Workspaces` happens at the child `PipelineRun`, not at the parent. If the parent omits a binding for a non-optional child workspace, the child fails rather than the parent (tracked in [#9924](https://github.com/tektoncd/pipeline/issues/9924)).
- [Execution `Status`](https://github.com/tektoncd/community/blob/main/teps/0056-pipelines-in-pipelines.md#execution-status) of a `PipelineTask` that uses `pipelineRef` or `pipelineSpec` is not surfaced, so a `finally` task cannot branch on whether a child `Pipeline` succeeded, failed, or was skipped via `$(tasks.<pipelineTaskName>.status)`.
- [`Matrix`](https://github.com/tektoncd/community/blob/main/teps/0056-pipelines-in-pipelines.md#matrix) fan-out of a `PipelineTask` that uses `pipelineRef` or `pipelineSpec` is not supported: such a task cannot be combined with a `matrix` to produce multiple child `PipelineRuns`. Consequently, results are always read from the single child `PipelineRun`.
- Results of a child `Pipeline` are not substituted into the parent `PipelineRun`'s `workspaces` bindings (`status.results`, `params` and `when` expressions are supported, see above).
- Results of a child `Pipeline` are not substituted into [propagated (undeclared) result references](pipelineruns.md#propagated-results) inside embedded `taskSpec`s — consume them through declared `params` instead. The same restriction already applies to Custom Task results.
