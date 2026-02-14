# Agent Guide for Tekton Pipelines

This document provides critical information for AI agents working in this repository.

## Core Commands

### Build & Install
- **Build all binaries**: `make all`
- **Build specific package**: `go build ./cmd/controller`
- **Deploy to local cluster**: `ko apply -R -f config/` (Requires `KO_DOCKER_REPO` to be set)

### Linting & Formatting
The project uses a strict `golangci-lint` configuration.
- **Run all linters**: `make golangci-lint`
- **Format code**: `make fmt`
- **Organize imports**: `make goimports`
- **Run go vet**: `make vet`

### Testing
- **Unit Tests**: `make test-unit` or `go test ./...`
- **E2E Tests**: `make test-e2e` or `go test -v -count=1 -tags=e2e ./test`
- **Single Test**: `go test -v -run TestName ./path/to/package`
- **E2E Parallel/Serial**: 
  - Parallel: `go test -tags=e2e -category=parallel ./test`
  - Serial: `go test -tags=e2e -category=serial ./test`
  - *Note*: Tests modifying shared state (ConfigMaps in `tekton-pipelines`) MUST be marked serial with `// @test:execution=serial`.

### Code Generation
Tekton relies heavily on code generation for deepcopy, clients, and OpenAPI.
- **Run all codegen**: `make generated` or `./hack/update-codegen.sh`
- **MANDATORY**: Run codegen after any change to `pkg/apis/pipeline/...` structs.

---

## Project Structure

- `pkg/apis/pipeline/v1/`: Core API definitions (structs, validation, defaulting).
- `pkg/reconciler/`: Controller logic for Pipelines, Tasks, and Runs.
- `pkg/client/`: Generated Kubernetes clients and informers.
- `cmd/`: Entry points for binaries (controller, webhook).
- `test/`: E2E and integration tests.
- `hack/`: Scripts for development, codegen, and CI.

---

## Code Style Guidelines

### Go Conventions
- **Go Version**: 1.25.0
- **Imports**: Ordered in 3 blocks: Standard library, Third-party (k8s, knative), and Internal (`github.com/tektoncd/pipeline`). Use `goimports`.
- **Naming**:
  - Structs & Methods: `CamelCase`.
  - Receiver names: Short abbreviations (e.g., `pr` for `PipelineRun`, `tr` for `TaskRun`).
  - API fields: `CamelCase` in Go, `camelCase` in JSON tags.
- **Error Handling**: 
  - Wrap errors with context: `fmt.Errorf("doing X: %w", err)`.
  - API validation errors MUST use `knative.dev/pkg/apis.FieldError`.
- **API Implementation**:
  - Resources must implement `apis.Defaultable` (`SetDefaults` in `_defaults.go`) and `apis.Validatable` (`Validate` in `_validation.go`).

### Forbidden Patterns
- **Do NOT** use `io/ioutil`. Use `io` or `os` instead.
- **Do NOT** use `github.com/ghodss/yaml`. Use `sigs.k8s.io/yaml` instead.
- **Avoid** `as any` or bypassing type safety.

---

## Testing Patterns

### Unit Testing Controllers
- Use fake clients: `github.com/tektoncd/pipeline/pkg/client/clientset/versioned/fake`.
- Use `knative.dev/pkg/reconciler/testing` for table-driven reconciler tests.

### E2E Testing
- Tests are run in a random namespace prefixed with `arendelle-`.
- Use `WaitFor*` methods from `test/` package to poll for resource state.
- Use `SimpleNameGenerator` for unique resource names.
