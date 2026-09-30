# .NET SDK on GCP Cloud Run

<!-- Source: docs/develop/dotnet/workers/serverless-workers/cloud-run.mdx -->

Use this reference for .NET-specific Worker construction, versioning behavior, connection configuration, image packaging, and scale-in safety. For shared Cloud Run execution constraints, deployment lifecycle, permissions, versioning, observability, and diagnostics, see `constraints.md`, `setup.md`, `iam.md`, `versioning.md`, `observability.md`, and `diagnostics.md`.

## Install and scaffold

Create the project with the same target framework used by the runtime image, then pin the packages validated for this guide:

```bash
dotnet new console --framework net9.0 --name MyWorker
dotnet add MyWorker/MyWorker.csproj package Temporalio --version 1.20.0
dotnet add MyWorker/MyWorker.csproj package Microsoft.Extensions.Logging.Console --version 9.0.0
```

Do not rely on the locally installed SDK's default framework. For example, a .NET 10 SDK creates `net10.0` unless `--framework net9.0` is explicit, and that output cannot run in the 9.0 runtime image below.

## Inspect the versioning API before generating code

```bash
dotnet list package
unzip -p ~/.nuget/packages/temporalio/<version>/temporalio.<version>.nupkg \
  'lib/net*/Temporalio.xml' \
  | rg -n -A12 'T:Temporalio\.(Worker\.WorkerDeploymentOptions|Common\.WorkerDeploymentVersion)'
```

Run `dotnet restore` first so the `.nupkg` and XML documentation exist locally. If inspection runs in a temporary SDK container, mount the NuGet cache and install `unzip`; `NUGET_XMLDOC_MODE=skip` removes the XML file this command needs.

## Versioned Worker

Set `DeploymentOptions` on `TemporalWorkerOptions`. The Worker reads its connection settings and Task Queue from the environment so one image runs against any Namespace:

```csharp
using System.Runtime.InteropServices;
using Temporalio.Client;
using Temporalio.Common;
using Temporalio.Worker;

var deploymentName = Environment.GetEnvironmentVariable("TEMPORAL_DEPLOYMENT_NAME")
    ?? throw new InvalidOperationException("TEMPORAL_DEPLOYMENT_NAME must be set");
var buildId = Environment.GetEnvironmentVariable("TEMPORAL_BUILD_ID")
    ?? throw new InvalidOperationException("TEMPORAL_BUILD_ID must be set");
var taskQueue = Environment.GetEnvironmentVariable("TEMPORAL_TASK_QUEUE")
    ?? throw new InvalidOperationException("TEMPORAL_TASK_QUEUE must be set");

var client = await TemporalClient.ConnectAsync(
    new(Environment.GetEnvironmentVariable("TEMPORAL_ADDRESS")!)
    {
        Namespace = Environment.GetEnvironmentVariable("TEMPORAL_NAMESPACE")!,
        ApiKey = Environment.GetEnvironmentVariable("TEMPORAL_API_KEY"),
        Tls = new(),
    });

var options = new TemporalWorkerOptions(
    taskQueue)
{
    DeploymentOptions = new(new(deploymentName, buildId), useWorkerVersioning: true)
    {
        DefaultVersioningBehavior = VersioningBehavior.Pinned,
    },
    GracefulShutdownTimeout = TimeSpan.FromSeconds(8),
};
options.AddWorkflow<GreetingWorkflow>();
options.AddAllActivities(typeof(GreetingActivities), null);

using var shutdown = new CancellationTokenSource();
using var sigterm = PosixSignalRegistration.Create(
    PosixSignal.SIGTERM,
    context =>
    {
        context.Cancel = true;
        shutdown.Cancel();
    });
Console.CancelKeyPress += (_, eventArgs) =>
{
    eventArgs.Cancel = true;
    shutdown.Cancel();
};

using var worker = new TemporalWorker(client, options);
Console.WriteLine(
    $"Worker started deployment={deploymentName} build={buildId} taskQueue={taskQueue}");
try
{
    await worker.ExecuteAsync(shutdown.Token);
}
catch (OperationCanceledException) when (shutdown.IsCancellationRequested)
{
    // Expected during Cloud Run scale-in or an interactive stop.
}
```

`WorkerDeploymentVersion`'s two arguments are the deployment name and the build ID, and both must match the version created with `temporal worker deployment create-version` exactly. → `setup.md` Step 6.

Use the ordinary publish for the image's platform; it includes the native Rust bridge. Cloud Run needs no provider-specific handler or artifact format.

## Versioning behavior

Every Workflow needs `VersioningBehavior.Pinned` or `AutoUpgrade`. `DefaultVersioningBehavior` covers every Workflow; to set it per Workflow, set it on the `[Workflow]` attribute. No package supplies a Worker-level default, so one of the two must be set explicitly.

```csharp
using Temporalio.Common;
using Temporalio.Workflows;

[Workflow(VersioningBehavior = VersioningBehavior.Pinned)]
public class GreetingWorkflow
{
    [WorkflowRun]
    public async Task<string> RunAsync(string name) => // ...
}
```

**A Version set with no behavior fails at runtime**, not at build time.

## Connection configuration

Read `TEMPORAL_ADDRESS`, `TEMPORAL_NAMESPACE`, `TEMPORAL_API_KEY`, and `TEMPORAL_TASK_QUEUE` from the environment set on the pool, and mount the key from Secret Manager rather than passing it in plaintext. → `setup.md` Step 4.

To load them through the shared config format instead, use `ClientEnvConfig.LoadClientConnectOptions()` from `Temporalio.Common.EnvConfig`.

**No `SSL_CERT_FILE` override is needed.** The Debian-based `mcr.microsoft.com/dotnet/runtime` images ship a certificate store the Rust core can read.

## Image packaging

Publish and run on a .NET runtime image:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build

WORKDIR /src
COPY MyWorker.csproj ./
RUN dotnet restore
COPY . .
RUN dotnet publish MyWorker.csproj -c Release -o /out \
    --runtime linux-x64 --self-contained false

FROM mcr.microsoft.com/dotnet/runtime:9.0

WORKDIR /app
COPY --from=build /out ./
CMD ["dotnet", "MyWorker.dll"]
```

`MyWorker.dll` must match the project's `<AssemblyName>` (or the project filename when `AssemblyName` is unset), and `<TargetFramework>` must remain `net9.0` for this image. Cloud Run Worker Pools run it as `linux/amd64`.

Save the following alongside the Dockerfile as `.gcloudignore`. The Dockerfile uses `COPY . .`, so excluding local `bin/` and `obj/` prevents stale restore or build output from entering the Cloud Build context:

```gitignore
.git
.gitignore
.idea
bin
obj
TestResults
```

The Debian-based runtime image above includes CA certificates. Verify that any alternative runtime image also provides a readable CA bundle. → `setup.md` Step 2.

## Graceful shutdown on scale-in

Cloud Run sends `SIGTERM`, which is distinct from the `SIGINT` raised by Ctrl+C. The example registers both paths and cancels the token passed to `ExecuteAsync`. That stops polling and starts the Temporal Worker's shutdown sequence.

`GracefulShutdownTimeout` defaults to zero. Keep a non-zero value below the termination window described in `constraints.md`, leaving time for cancellation and final completions to propagate. Activities should observe `ActivityExecutionContext.Current.WorkerShutdownToken` or `CancellationToken` and record Heartbeats.

## Keep Activities safe across scale-in

Apply the Heartbeat-resume invariant from `constraints.md` in .NET:

```csharp
using Temporalio.Common;
using Temporalio.Workflows;

var result = await Workflow.ExecuteActivityAsync(
    () => GreetingActivities.ProcessAsync(items),
    new ActivityOptions
    {
        StartToCloseTimeout = TimeSpan.FromMinutes(10),
        HeartbeatTimeout = TimeSpan.FromSeconds(10),
        RetryPolicy = new RetryPolicy
        {
            InitialInterval = TimeSpan.FromSeconds(1),
            MaximumAttempts = 5,
        },
    });
```

The Activity records the next item to process:

```csharp
[Activity]
public static async Task<string> ProcessAsync(IReadOnlyList<string> items)
{
    var context = ActivityExecutionContext.Current;
    var info = context.Info;
    var startIndex = info.HeartbeatDetails.Count > 0
        ? await info.HeartbeatDetailAtAsync<int>(0)
        : 0;

    for (var i = startIndex; i < items.Count; i++)
    {
        // ... process items[i]
        context.Heartbeat(i + 1);
    }
    return "done";
}
```

→ `constraints.md` for what else follows from the pool model.

## Logging and diagnostic signatures

The .NET SDK defaults to `NullLoggerFactory`, so configure a console provider explicitly. Add `Microsoft.Extensions.Logging.Console`, create the factory before connecting, and assign it to the client options:

```csharp
using Microsoft.Extensions.Logging;

using var loggerFactory = LoggerFactory.Create(builder =>
{
    builder.AddSimpleConsole(options => options.SingleLine = true);
    builder.SetMinimumLevel(LogLevel.Information);
    builder.AddFilter("Grpc", LogLevel.Warning);
});

var client = await TemporalClient.ConnectAsync(
    new(Environment.GetEnvironmentVariable("TEMPORAL_ADDRESS")!)
    {
        Namespace = Environment.GetEnvironmentVariable("TEMPORAL_NAMESPACE")!,
        ApiKey = Environment.GetEnvironmentVariable("TEMPORAL_API_KEY"),
        Tls = new(),
        LoggerFactory = loggerFactory,
    });
```

Do not enable DEBUG logging globally in production without first verifying that dependency logs cannot contain credentials or payloads.

| Log signature | Meaning / action |
|---|---|
| No SDK logs | The default null logger is still in use; pass an `ILoggerFactory` to the client. |
| `NativeCertsNotFound` | The runtime image lacks a readable CA store. Use the Debian runtime image or install CA certificates. |
| `OperationCanceledException` immediately after SIGTERM | Expected when it is caught by the shutdown path shown above. |

## Observability

For the optional Cloud Run OpenTelemetry path, add the released extension matching the SDK version:

```bash
dotnet add MyWorker/MyWorker.csproj package Temporalio.Extensions.Gcp.CloudRun.OpenTelemetry --version 1.20.0
```

Apply its defaults to the same connect options used by the Worker, retain the returned handle, and flush after the Worker stops:

```csharp
using Temporalio.Extensions.Gcp.CloudRun.OpenTelemetry;

var connectOptions = new TemporalClientConnectOptions(
    Environment.GetEnvironmentVariable("TEMPORAL_ADDRESS")!)
{
    Namespace = Environment.GetEnvironmentVariable("TEMPORAL_NAMESPACE")!,
    ApiKey = Environment.GetEnvironmentVariable("TEMPORAL_API_KEY"),
    Tls = new(),
};
using var telemetry = connectOptions.ApplyGoogleCloudRunOpenTelemetryDefaults();
var client = await TemporalClient.ConnectAsync(connectOptions);

// After worker.ExecuteAsync returns:
await telemetry.FlushAsync(TimeSpan.FromSeconds(2));
```

The helper defaults to the local OTLP/gRPC Collector endpoint. Use the multi-container topology, IAM, and shutdown order in `observability.md`; do not add the extension unless that Collector path is enabled. The complete maintained example is in [samples-dotnet PR #236](https://github.com/temporalio/samples-dotnet/pull/236).
