# Go SDK on GCP Cloud Run

<!-- Source: docs/develop/go/workers/serverless-workers/cloud-run.mdx -->

Use this reference for Go-specific Worker construction, versioning behavior, connection configuration, image packaging, and scale-in safety. For shared Cloud Run execution constraints, deployment lifecycle, permissions, versioning, observability, and diagnostics, see `constraints.md`, `setup.md`, `iam.md`, `versioning.md`, `observability.md`, and `diagnostics.md`.

## Install and scaffold

From the application directory, initialize the module and pin the versions validated for this guide. `contrib/envconfig` is a separate Go module:

```bash
go mod init example.com/myapp
go get go.temporal.io/sdk@v1.49.0 go.temporal.io/sdk/contrib/envconfig@v1.0.2
```

## Inspect the versioning API before generating code

```bash
SDK_DIR=$(go list -m -f '{{.Dir}}' go.temporal.io/sdk)
rg -n -A40 --glob '*.go' 'WorkerDeploymentOptions struct' "$SDK_DIR/internal"
rg -n -A12 --glob '*.go' 'WorkerDeploymentVersion struct' "$SDK_DIR/internal"
go doc go.temporal.io/sdk/workflow.RegisterOptions
```

The exported Worker types are aliases, so ordinary `go doc` may print only `type X = internal.Y`; inspect the aliased definitions in the installed module as above.

## Versioned Worker

Set `DeploymentOptions` in `worker.Options`. The Worker reads its connection settings and Task Queue from the environment so one image runs against any Namespace:

```go
package main

import (
	"log"
	"os"
	"time"

	"go.temporal.io/sdk/client"
	"go.temporal.io/sdk/contrib/envconfig"
	"go.temporal.io/sdk/worker"
	"go.temporal.io/sdk/workflow"

	"example.com/myapp"
)

func main() {
	deploymentName := mustEnv("TEMPORAL_DEPLOYMENT_NAME")
	buildID := mustEnv("TEMPORAL_BUILD_ID")
	taskQueue := mustEnv("TEMPORAL_TASK_QUEUE")

	c, err := client.Dial(envconfig.MustLoadDefaultClientOptions())
	if err != nil {
		log.Fatalln("Unable to create client", err)
	}
	defer c.Close()

	w := worker.New(c, taskQueue, worker.Options{
		WorkerStopTimeout: 8 * time.Second,
		DeploymentOptions: worker.DeploymentOptions{
			UseVersioning: true,
			Version: worker.WorkerDeploymentVersion{
				DeploymentName: deploymentName,
				BuildID:        buildID,
			},
		},
	})

	w.RegisterWorkflowWithOptions(myapp.MyWorkflow, workflow.RegisterOptions{
		VersioningBehavior: workflow.VersioningBehaviorPinned,
	})
	w.RegisterActivity(myapp.MyActivity)
	log.Printf("Worker started deployment=%s build=%s taskQueue=%s", deploymentName, buildID, taskQueue)

	if err := w.Run(worker.InterruptCh()); err != nil {
		log.Fatalln("Unable to start worker", err)
	}
}

func mustEnv(name string) string {
	value := os.Getenv(name)
	if value == "" {
		log.Fatalf("%s must be set", name)
	}
	return value
}
```

`DeploymentName` and `BuildID` must match the version created with `temporal worker deployment create-version` exactly, or the Worker polls under a version the WCI does not manage. → `setup.md` Step 6.

## Versioning behavior

Every Workflow needs `workflow.VersioningBehaviorPinned` or `VersioningBehaviorAutoUpgrade`. Set it per Workflow at registration as above, or set `DefaultVersioningBehavior` in `DeploymentOptions` to cover every Workflow. Registration **panics** with `workflow type does not have a versioning behavior` if a Version is set and neither is given.

```go
w.RegisterWorkflowWithOptions(myapp.MyWorkflow, workflow.RegisterOptions{
	VersioningBehavior: workflow.VersioningBehaviorPinned,
})
```

**A Version set with no behavior fails at runtime**, not at build time.

## Connection configuration

`go.temporal.io/sdk/contrib/envconfig` loads client configuration from environment variables and an optional TOML file, so the Worker carries no Namespace or credentials. Set non-secret values with `--set-env-vars` on the pool and mount the API key or TLS material from Secret Manager with `--set-secrets`. → `setup.md` Step 4.

`MustLoadDefaultClientOptions` **panics** on invalid configuration. Use `envconfig.LoadDefaultClientOptions` and check the error to fail with a readable message instead.

## Image packaging

Cloud Run Worker Pools run this image as `linux/amd64`. Use `CGO_ENABLED=0` with a distroless static runtime:

```dockerfile
FROM golang:1.26-bookworm AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -trimpath -o /out/worker ./cmd/worker

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/worker /worker
ENTRYPOINT ["/worker"]
```

Place this `.gcloudignore` beside the Dockerfile so build context does not include local binaries or repository metadata:

```gitignore
.git
.gitignore
.idea
bin
dist
tmp
```

The distroless image includes system CA roots. Adjust `./cmd/worker` only if the actual main package lives elsewhere. → `setup.md` Step 2.

## Graceful shutdown on scale-in

The versioned Worker example uses `w.Run(worker.InterruptCh())` and gives received Tasks up to eight seconds through `WorkerStopTimeout`. `InterruptCh` receives both `SIGINT` and `SIGTERM`, so Cloud Run's `SIGTERM` makes the Worker stop polling and begin its normal shutdown. Do not replace it with an unhandled blocking channel, and do not leave `WorkerStopTimeout` at its zero default when draining is required.

## Keep Activities safe across scale-in

Apply the Heartbeat-resume invariant from `constraints.md` in Go:

```go
import (
	"time"

	"go.temporal.io/sdk/temporal"
	"go.temporal.io/sdk/workflow"
)

func MyWorkflow(ctx workflow.Context, input MyInput) (string, error) {
	ctx = workflow.WithActivityOptions(ctx, workflow.ActivityOptions{
		StartToCloseTimeout: 10 * time.Minute,
		HeartbeatTimeout:    10 * time.Second,
		RetryPolicy: &temporal.RetryPolicy{
			InitialInterval: time.Second,
			MaximumAttempts: 5,
		},
	})
	var result string
	err := workflow.ExecuteActivity(ctx, MyActivity, input).Get(ctx, &result)
	return result, err
}
```

The Activity records the next item to process:

```go
func MyActivity(ctx context.Context, input MyInput) (string, error) {
	startIndex := 0
	if activity.HasHeartbeatDetails(ctx) {
		if err := activity.GetHeartbeatDetails(ctx, &startIndex); err != nil {
			return "", err
		}
	}

	for i := startIndex; i < len(input.Items); i++ {
		// ... process input.Items[i]
		activity.RecordHeartbeat(ctx, i+1)
	}
	return "done", nil
}
```

→ `constraints.md` for what else follows from the pool model.

## Observability

For the optional Cloud Run OpenTelemetry path, add the released contrib module:

```bash
go get go.temporal.io/sdk/contrib/gcp/cloudrun/otel@v0.1.0
```

Create the plugin before dialing and append it to the client options. When this optional path is enabled, remove the main example's `defer c.Close()` and make shutdown order explicit: `w.Run` stops the Worker before it returns, then close the client, then flush telemetry. The one-second flush keeps the eight-second drain, client close, and flush inside Cloud Run's roughly ten-second termination window together; see `observability.md`.

```go
ctx := context.Background()
otelPlugin, err := otel.NewPlugin(ctx, otel.PluginOptions{})
if err != nil {
	log.Fatalln("Unable to create OpenTelemetry plugin", err)
}
clientOptions := envconfig.MustLoadDefaultClientOptions()
clientOptions.Plugins = append(clientOptions.Plugins, otelPlugin)
c, err := client.Dial(clientOptions)
if err != nil {
	log.Fatalln("Unable to create client", err)
}

// After constructing and registering w as in the main example:
runErr := w.Run(worker.InterruptCh())
c.Close()

flushCtx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
flushErr := otelPlugin.Shutdown(flushCtx)
cancel()
if flushErr != nil {
	log.Println("Failed to flush OpenTelemetry", flushErr)
}
if runErr != nil {
	log.Fatalln("Unable to run worker", runErr)
}
```

Add `context` and import `go.temporal.io/sdk/contrib/gcp/cloudrun/otel`. The helper defaults to the local OTLP/gRPC Collector endpoint. Use the multi-container topology, IAM, and shutdown order in `observability.md`; do not add the plugin unless that Collector path is enabled. The complete example under review is in [samples-go PR #554](https://github.com/temporalio/samples-go/pull/554).
