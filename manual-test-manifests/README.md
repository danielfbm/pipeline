# Manual Testing Guide for PipelineRef in PipelineTask

This directory contains manifests to manually verify the `pipelineRef` support in `PipelineTask`.

## Prerequisites
- A Kubernetes cluster with Tekton Pipelines installed (with your changes applied).
- `kubectl` configured.

## Setup

1. **Configure Feature Flags**:
   Apply the feature flags to enable "alpha" fields (required for `pipelineRef`) and "restricted" security context support (required for restricted clusters).
   ```bash
   kubectl apply -f 00-configure-feature-flags.yaml
   # Restart the controller to pick up changes immediately (optional but recommended)
   kubectl rollout restart deployment/tekton-pipelines-controller -n tekton-pipelines
   ```

## Test Cases

### 1. Basic Functionality
**Goal**: Verify a parent pipeline can trigger a child pipeline via `pipelineRef`.

```bash
# Apply the child pipeline and task
kubectl apply -f 01-basic-child.yaml

# Apply the parent pipeline and run it
kubectl create -f 02-basic-parent.yaml
```

**Verification**:
- Check the PipelineRun status: `kubectl get pr`
- You should see a child PipelineRun created (name starts with `run-basic-parent-...-call-child`).
- Logs should show "I am the child pipeline!".

### 2. Complex Case (Params & Workspaces)
**Goal**: Verify parameters and workspaces are correctly propagated to the child pipeline.

```bash
# Apply the complex child pipeline and task
kubectl apply -f 03-complex-child.yaml

# Apply the parent pipeline and run it
kubectl create -f 04-complex-parent.yaml
```

**Verification**:
- Check the PipelineRun status.
- The child PipelineRun should succeed.
- Logs should show "Message passed from Parent to Child!".

### 3. Failure Case (Missing Reference)
**Goal**: Verify that referencing a non-existent pipeline fails gracefully (or as expected).

```bash
# Apply the failure parent pipeline and run it
kubectl create -f 05-failure-case.yaml
```

**Verification**:
- The parent PipelineRun should fail.
- The status should indicate that `this-pipeline-does-not-exist` could not be resolved.
