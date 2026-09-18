# .NET SDK on GCP Cloud Run

<!-- Source: docs/develop/dotnet/workers/serverless-workers/cloud-run.mdx -->

Use this reference for .NET-specific Worker construction, versioning behavior, connection configuration, image packaging, and scale-in safety. For the shared Cloud Run deployment lifecycle, permissions, versioning model, observability, and diagnostics, see `setup.md`, `iam.md`, `versioning.md`, `observability.md`, and `diagnostics.md`.

**There is no Cloud Run Worker package.** This is an ordinary long-lived .NET Worker plus Worker Versioning, which Serverless Workers require.

## Inspect the versioning API before generating code

Worker Versioning is a Public Preview surface and the option names differ between SDKs. Read the installed version's API rather than writing from memory:

```bash
dotnet list package
unzip -p ~/.nuget/packages/temporalio/<version>/temporalio.<version>.nupkg \
  'lib/net*/Temporalio.xml' | grep -A3 'WorkerDeploymentOptions\|WorkerDeploymentVersion'
```

## Versioned Worker

Set `DeploymentOptions` on `TemporalWorkerOptions`. The Worker reads its connection settings and Task Queue from the environment so one image runs against any Namespace:

```csharp
using System.Runtime.InteropServices;
using Temporalio.Client;
using Temporalio.Common;
using Temporalio.Worker;

var client = await TemporalClient.ConnectAsync(
    new(Environment.GetEnvironmentVariable("TEMPORAL_ADDRESS")!)
    {
        Namespace = Environment.GetEnvironmentVariable("TEMPORAL_NAMESPACE")!,
        ApiKey = Environment.GetEnvironmentVariable("TEMPORAL_API_KEY"),
        Tls = new(),
    });

var options = new TemporalWorkerOptions(
    Environment.GetEnvironmentVariable("TEMPORAL_TASK_QUEUE")!)
{
    DeploymentOptions = new(new("my-app", "build-1"), useWorkerVersioning: true)
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
COPY *.csproj ./
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /out

FROM mcr.microsoft.com/dotnet/runtime:9.0

WORKDIR /app
COPY --from=build /out ./
CMD ["dotnet", "MyWorker.dll"]
```

The Debian-based runtime image above includes CA certificates. Verify that any alternative runtime image also provides a readable CA bundle. → `setup.md` Step 2.

## Graceful shutdown on scale-in

Cloud Run sends `SIGTERM`, which is distinct from the `SIGINT` raised by Ctrl+C. The example registers both paths and cancels the token passed to `ExecuteAsync`. That stops polling and starts the Temporal Worker's shutdown sequence.

`GracefulShutdownTimeout` defaults to zero. Keep a non-zero value below Cloud Run's ten-second termination window, leaving time for cancellation and final completions to propagate. Activities should observe `ActivityExecutionContext.Current.WorkerShutdownToken` or `CancellationToken` and record Heartbeats; Cloud Run can still send `SIGKILL` before a long Activity finishes.

## Keep Activities safe across scale-in

The WCI removes instances based on Task Queue activity, not on what an individual instance is doing, so **an instance running a long Activity can be stopped mid-execution.** Record the next unprocessed index only after processing succeeds, then read it from Heartbeat details so a retry resumes at that index:

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
| Worker starts but the intended Workflow does not progress | Check the deployment name, build ID, Task Queue, and that the version is current. |

## Observability

A Cloud Run Worker emits the same traces and metrics as a Worker anywhere else, with no Cloud Run-specific wiring. Use the SDK's normal metrics export and OpenTelemetry tracing interceptors. → `observability.md`, and `docs/develop/dotnet/platform/observability`.
