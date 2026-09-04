# Go SDK on GCP Cloud Run

<!-- Source: docs/develop/go/workers/serverless-workers/cloud-run.mdx -->

Use this reference for Go-specific Worker construction, versioning behavior, connection configuration, image packaging, and scale-in safety. For the shared Cloud Run deployment lifecycle, permissions, versioning model, observability, and diagnostics, see `setup.md`, `iam.md`, `versioning.md`, `observability.md`, and `diagnostics.md`.

**There is no Cloud Run Worker package.** This is an ordinary long-lived Go Worker plus Worker Versioning, which Serverless Workers require. Nothing in `../aws-lambda/sdk-go.md` applies.

## Inspect the versioning API before generating code

Worker Versioning is a Public Preview surface and the option names differ between SDKs. Read the installed version's API rather than writing from memory:

```bash
go doc go.temporal.io/sdk/worker.DeploymentOptions
go doc go.temporal.io/sdk/worker.WorkerDeploymentVersion
go doc go.temporal.io/sdk/workflow.RegisterOptions
```

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
	c, err := client.Dial(envconfig.MustLoadDefaultClientOptions())
	if err != nil {
		log.Fatalln("Unable to create client", err)
	}
	defer c.Close()

	w := worker.New(c, os.Getenv("TEMPORAL_TASK_QUEUE"), worker.Options{
		WorkerStopTimeout: 8 * time.Second,
		DeploymentOptions: worker.DeploymentOptions{
			UseVersioning: true,
			Version: worker.WorkerDeploymentVersion{
				DeploymentName: "my-app",
				BuildID:        "build-1",
			},
		},
	})

	w.RegisterWorkflowWithOptions(myapp.MyWorkflow, workflow.RegisterOptions{
		VersioningBehavior: workflow.VersioningBehaviorPinned,
	})
	w.RegisterActivity(myapp.MyActivity)

	if err := w.Run(worker.InterruptCh()); err != nil {
		log.Fatalln("Unable to start worker", err)
	}
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

Use `CGO_ENABLED=0` with a `distroless/static` base. Go still reads system CA roots; that image includes them. → `setup.md` Step 2.

## Graceful shutdown on scale-in

The versioned Worker example uses `w.Run(worker.InterruptCh())` and gives received Tasks up to eight seconds through `WorkerStopTimeout`. `InterruptCh` receives both `SIGINT` and `SIGTERM`, so Cloud Run's `SIGTERM` makes the Worker stop polling and begin its normal shutdown. Do not replace it with an unhandled blocking channel, and do not leave `WorkerStopTimeout` at its zero default when draining is required.

Cloud Run can send `SIGKILL` ten seconds later. Shutdown therefore improves draining but cannot guarantee that a long Activity finishes; Activities must cooperate with cancellation and record Heartbeats as shown below.

## Keep Activities safe across scale-in

The WCI removes instances based on Task Queue activity, not on what an individual instance is doing, so **an instance running a long Activity can be stopped mid-execution.** Record Heartbeats so a retry resumes from the last recorded progress:

```go
func MyActivity(ctx context.Context, input MyInput) (string, error) {
	for i := range input.Items {
		activity.RecordHeartbeat(ctx, i)
		// ... process input.Items[i]
	}
	return "done", nil
}
```

→ `constraints.md` for what else follows from the pool model.

## Observability

A Cloud Run Worker emits the same traces and metrics as a Worker anywhere else — no Cloud Run-specific wiring, and none of Lambda's ADOT layer or collector configuration. Use the SDK's normal metrics export and OpenTelemetry tracing interceptors. → `observability.md`, and `docs/develop/go/platform/observability`.
