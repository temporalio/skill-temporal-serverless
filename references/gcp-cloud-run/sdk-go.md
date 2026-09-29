# Go SDK on GCP Cloud Run

<!-- Source: docs/develop/go/workers/serverless-workers/cloud-run.mdx -->

Use this reference for Go-specific Worker construction, versioning behavior, connection configuration, image packaging, and scale-in safety. For shared Cloud Run execution constraints, deployment lifecycle, permissions, versioning, observability, and diagnostics, see `constraints.md`, `setup.md`, `iam.md`, `versioning.md`, `observability.md`, and `diagnostics.md`.

## Inspect the versioning API before generating code

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

## Keep Activities safe across scale-in

Apply the Heartbeat-resume invariant from `constraints.md` in Go:

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

For Go SDK configuration, see `docs/develop/go/platform/observability`. For shared Cloud Run behavior and provider-specific scaling signals, see `observability.md`.
