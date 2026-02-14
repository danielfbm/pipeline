## Learnings - PipelineRef Support in PipelineTask
- Updated `IsChildPipeline` to include `PipelineRef != nil` check.
- Updated `setChildPipelineRunsAndResolvedPipeline` to assign `rp.PipelineRef = pipelineTask.PipelineRef`.
- Updated unit tests to reflect that `PipelineRef` is now supported and no longer returns an error.
- Verified that `IsChildPipeline` is used correctly in other parts of the codebase (e.g., `pipelinerunstate.go`) to skip child pipelines when processing TaskRun results.
Updated createChildPipelineRun to support PipelineRef and PipelineSpec from rpt.PipelineTask. Used direct assignment in the struct literal which handles nil pointers correctly for 'one of' logic.

- Implemented `TestPipelineRef_ChildExecution` in `test/pipelinerun_pinp_test.go`.
- Used `parse.MustParseV1Pipeline` and `parse.MustParseV1PipelineRun` to define test resources, consistent with existing tests.
- Found that `assertPinP` helper assumes `PipelineSpec` behavior (where child pipeline name is derived from parent PR name for labels).
- For `PipelineRef`, the child PipelineRun correctly inherits the `tekton.dev/pipeline` label from the referenced pipeline name.
- Implemented custom assertions for `PipelineRef` label propagation in the test.
### Child PipelineRun Propagation
- Propagated Workspaces, Timeouts, and TaskRunTemplate to child PipelineRuns in .
- Used  to retrieve the appropriate template for the child PipelineRun.
- Mapped workspaces by finding the parent binding and joining SubPaths.
### Child PipelineRun Propagation
- Propagated Workspaces, Timeouts, and TaskRunTemplate to child PipelineRuns in createChildPipelineRun.
- Used pr.GetTaskRunSpec to retrieve the appropriate template for the child PipelineRun.
- Mapped workspaces by finding the parent binding and joining SubPaths.
