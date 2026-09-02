# .NET SDK on AWS Lambda

<!-- Sources: Temporalio.Extensions.Aws.Lambda 1.18.0 XML docs, NuGet package metadata, and samples-dotnet@main -->

Use this reference for .NET SDK-specific package, entry-point, Worker configuration, tuned defaults, observability, and diagnostic details. For shared AWS Lambda deployment, observability infrastructure, and diagnostic flow, see `setup.md`, `observability.md`, and `diagnostics.md`.

## Package

Import: `using Temporalio.Extensions.Aws.Lambda;` plus `Temporalio.Common` (for `WorkerDeploymentVersion`) and `Amazon.Lambda.Core` (for `ILambdaContext`). <!-- verified against Temporalio.Extensions.Aws.Lambda 1.18.0 XML docs and samples-dotnet@main -->

Install: `dotnet add package Temporalio.Extensions.Aws.Lambda` — a **separate NuGet package** from `Temporalio`, published in **lockstep** with it (both 1.18.0), the same relationship Java has. Published versions: 1.17.0 and 1.18.0. The package targets `netstandard2.0` and declares `Temporalio` 1.18.0 and `Amazon.Lambda.Core` 3.1.0.

OpenTelemetry lives in a **second package**, `Temporalio.Extensions.Aws.Lambda.OpenTelemetry` (also 1.18.0) — unlike Python, where OTel is an extra on the same package. → Observability below.

- .NET: [.NET Lambda Worker sample](https://github.com/temporalio/samples-dotnet/tree/main/src/LambdaWorker) — `Worker/`, `Starter/`, and `Deploy/` (deploy, IAM-role, execution-role, and telemetry scripts plus a CloudFormation template), with a test project under `tests/LambdaWorker`. <!-- verified against samples-dotnet@main -->

List the real public API of the resolved package before generating code — the `.nupkg` is a zip and ships full XML documentation:

```bash
curl -sO https://api.nuget.org/v3-flatcontainer/temporalio.extensions.aws.lambda/<ver>/temporalio.extensions.aws.lambda.<ver>.nupkg
unzip -p temporalio.extensions.aws.lambda.<ver>.nupkg \
  lib/netstandard2.0/Temporalio.Extensions.Aws.Lambda.xml
# the .nuspec lists the exact dependency versions:
unzip -p ...nupkg Temporalio.Extensions.Aws.Lambda.nuspec | grep dependency
```

**Ordering when sources disagree:** the installed artifact first, the SDK's maintained samples second (they are built in CI, so they cannot reference a method that does not exist), the prose docs last. Entry-point names are not consistent across SDKs, so check rather than pattern-match from another language.

## Entry point

**`TemporalLambdaWorker.CreateHandler(version, configure)`** — returns a `Func<object?, ILambdaContext, Task>` that your handler method delegates to. Overloads take either a synchronous `Action<TemporalLambdaWorkerOptions>` or an asynchronous `Func<TemporalLambdaWorkerOptions, Task>` for setup that must await. A further overload takes `TemporalLambdaWorkerHandlerOptions`, which the XML docs describe as "internal test seams" — not for production use.

`TemporalLambdaWorker.LoadClientConnectOptions(...)` is also public, for loading connection options with Lambda-aware config resolution outside the handler.

## Configure callback

Receives a `TemporalLambdaWorkerOptions` with `ClientOptions`, `WorkerOptions`, `ShutdownDeadlineBuffer`, `ShutdownHooks`, and `AddShutdownHook(Func<CancellationToken, Task>)`. The Task Queue and registrations go through `WorkerOptions` — an ordinary `TemporalWorkerOptions`, so `TaskQueue`, `AddWorkflow<T>()` and `AddActivity(...)` behave exactly as they do for a long-lived Worker. The callback runs **per invocation** (Java is the outlier that runs its at cold start).

## Versioning behavior

Per-Workflow via the `[Workflow]` attribute:

```csharp
[Workflow(VersioningBehavior = VersioningBehavior.Pinned)]
public class MyWorkflow
{
    [WorkflowRun]
    public async Task<string> RunAsync(string name) => /* ... */;
}
```

Or a Worker-level default through `DefaultVersioningBehavior` in `DeploymentOptions`.

**The .NET Worker-level default is `AutoUpgrade`**, whereas TypeScript's is `PINNED`. Defaults are not uniform across SDKs — never state one globally, and prefer setting the behavior explicitly per Workflow.

## Handler example

A plain class exposes an async method that delegates to the handler returned by `TemporalLambdaWorker.CreateHandler`. <!-- verified against Temporalio.Extensions.Aws.Lambda 1.18.0 and samples-dotnet@main src/LambdaWorker/Worker/Function.cs -->

```csharp
namespace MyCompany.Temporal.Worker;

using Amazon.Lambda.Core;
using Temporalio.Common;
using Temporalio.Extensions.Aws.Lambda;

public class LambdaFunction
{
    private static readonly Func<object?, ILambdaContext, Task> WorkerHandler =
        TemporalLambdaWorker.CreateHandler(
            new WorkerDeploymentVersion("my-app", "build-1"),
            config =>
            {
                config.WorkerOptions.TaskQueue = "my-task-queue";
                config.WorkerOptions
                    .AddWorkflow<MyWorkflow>()
                    .AddActivity(MyActivities.MyActivity);
            });

    public Task HandlerAsync(Stream input, ILambdaContext context) =>
        WorkerHandler(input, context);
}
```

Registrations go through `config.WorkerOptions`, an ordinary `TemporalWorkerOptions` — the same API a long-lived Worker uses. Use the `Func<TemporalLambdaWorkerOptions, Task>` overload when setup must await.

## Lambda-tuned defaults

<!-- docs/develop/dotnet/workers/serverless-workers/aws-lambda.mdx:125-137 -->

| Setting | Lambda default |
|---|---|
| `MaxConcurrentActivities` | 2 |
| `MaxConcurrentWorkflowTasks` | 10 |
| `MaxConcurrentLocalActivities` | 2 |
| `MaxConcurrentNexusTasks` | 5 |
| `MaxConcurrentWorkflowTaskPolls` | 2 |
| `MaxConcurrentActivityTaskPolls` | 1 |
| `MaxConcurrentNexusTaskPolls` | 1 |
| `MaxCachedWorkflows` | 30 |
| `GracefulShutdownTimeout` | 5 seconds |
| `ShutdownDeadlineBuffer` | 7 seconds |
| `DisableEagerActivityExecution` | Always `true`, cannot be overridden |

## Logging — set a LoggerFactory or Workflow logs vanish

`TemporalWorkerOptions.LoggerFactory` is unset by default and "defaults to the client logger factory", which is also unset — so `Workflow.Logger` output is discarded. Activity `Console.WriteLine` still reaches CloudWatch, which makes the gap look selective rather than total.

```csharp
config.WorkerOptions.LoggerFactory =
    LoggerFactory.Create(b => b.AddSimpleConsole().SetMinimumLevel(LogLevel.Information));
```

## Connection configuration

Loaded automatically from environment variables and an optional TOML config file, with the same resolution order as the other SDKs:

1. `TEMPORAL_CONFIG_FILE` environment variable, if set.
2. `temporal.toml` in the Lambda task root (typically `/var/task`).
3. `temporal.toml` in the current working directory.

The sample copies a `temporal.toml` into the publish directory before zipping, so it lands in the task root, and keeps the API key in `TEMPORAL_API_KEY` rather than in the file. Supplying an API key enables TLS automatically.

**TLS caveat specific to .NET — set `SSL_CERT_FILE` or the first invocation fails.** AWS's .NET 8 Lambda images force-override `SSL_CERT_FILE`, which prevents the SDK's Rust core from loading system root CAs. Set it explicitly on the function:

```
SSL_CERT_FILE=/etc/pki/tls/certs/ca-bundle.crt     # or /etc/ssl/certs/ca-certificates.crt
```

**This is server-certificate verification, not client credentials.** The API key is unaffected and is not the problem — an API key auto-enables TLS, and TLS requires verifying Temporal Cloud's certificate chain against root CAs. The connection fails before authentication is ever attempted.

Python shares the same Rust core (`temporalio/bridge/temporal_sdk_bridge.abi3.so`) but its Lambda image does not override the variable; Java uses gRPC/Netty and the JVM truststore, so neither is affected. → `diagnostics.md`. <!-- verified: reproduced and fixed on a real deployment -->

## Build and package

### Native dependency — publish must be RID-specific

The .NET SDK wraps a **native Rust core** (`libtemporalio_sdk_core_c_bridge.so`). A portable publish does not include the Linux build of it, and the failure appears only at first invocation. Always publish for an explicit runtime identifier matching the function's architecture:

| `--runtime` | `--architectures` |
|---|---|
| `linux-x64` | `x86_64` |
| `linux-arm64` | `arm64` |

This is .NET's equivalent of Python's `manylinux` wheels and Go's `GOARCH`.

Publish for an explicit Linux runtime identifier, then zip the publish output. <!-- verified against samples-dotnet@main src/LambdaWorker/Deploy/deploy-lambda.sh -->

```bash
dotnet publish path/to/Worker.csproj \
  --configuration Release \
  --runtime linux-x64 \
  --self-contained false \
  --output ./publish

# Guard: the SDK's native Rust bridge must be in the output, or the function
# fails at FIRST INVOCATION, not at build time.
[[ -f ./publish/libtemporalio_sdk_core_c_bridge.so ]] || {
  echo "Publish output is missing the linux-x64 Temporal native bridge." >&2; exit 1; }

# If you use a temporal.toml / otel-collector-config.yaml, copy them in so they
# land in the Lambda task root:
cp temporal.toml otel-collector-config.yaml ./publish/

cd ./publish && zip -r ../function.zip . && cd ..
```

Temporal's own deploy script asserts the file is present before zipping, which is worth copying. `--self-contained false` is correct: the `dotnet8` managed runtime supplies the framework.

## Deploy the Lambda function

<!-- verified against samples-dotnet@main src/LambdaWorker/Deploy/deploy-lambda.sh -->

```bash
aws lambda create-function \
  --function-name my-temporal-worker \
  --runtime dotnet8 \
  --architectures x86_64 \
  --handler 'MyAssembly::MyCompany.Temporal.Worker.LambdaFunction::HandlerAsync' \
  --role <EXECUTION_ROLE_ARN> \
  --zip-file fileb://function.zip \
  --timeout 600 \
  --memory-size 256 \
  --environment file:///tmp/lambda-env.json
```

**The environment block for .NET must include `SSL_CERT_FILE`**, in addition to the usual `TEMPORAL_*` variables:

```json
{"Variables":{
  "TEMPORAL_ADDRESS":"...", "TEMPORAL_NAMESPACE":"...", "TEMPORAL_API_KEY":"...",
  "SSL_CERT_FILE":"/etc/pki/tls/certs/ca-bundle.crt"}}
```

Without it the **first invocation fails**, the Task Queue is never bound, and the Worker is never invoked again — an otherwise-correct deployment that simply does not work. AWS's .NET 8 Lambda images force-override `SSL_CERT_FILE`, which stops the SDK's Rust core from loading system root CAs. `/etc/ssl/certs/ca-certificates.crt` also works; try the other if one fails. This is server-certificate verification, unrelated to your API key. → `diagnostics.md`. <!-- verified: reproduced and fixed on a real deployment; cause per temporalio/sdk-dotnet README, "AWS Lambda .NET 8 CA Loading Issues" -->

- `--runtime`: `dotnet8` (the sample targets `net8.0`). <!-- verified against samples-dotnet@main src/LambdaWorker/Deploy/deploy-lambda.sh and Directory.Build.props -->
- `--handler`: **`ASSEMBLY::NAMESPACE.TYPE::METHOD` — three colon-separated parts**, and the only SDK with that shape. Java uses two (`Class::method`); Go, Python and TypeScript use `module.function`-style. Getting this wrong presents as a handler-not-found error at first invocation.
- `--timeout 600` / `--memory-size 256`: **the same values as Go, Python and TypeScript.** Only Java's example differs (90/1024), which supports reading that as a Java-specific choice rather than a documentation inconsistency.
- `--architectures` must match the publish RID (`linux-x64` → `x86_64`, `linux-arm64` → `arm64`).
- Temporal's deploy script retries `create-function` up to 12 times to absorb IAM propagation delay on a freshly created execution role — the same behavior described under "A freshly created execution role may not be assumable immediately" in `setup.md`.

## Observability

### OTel package

A **second NuGet package**, separate from the Lambda extension itself: <!-- verified against NuGet Temporalio.Extensions.Aws.Lambda.OpenTelemetry 1.18.0 and samples-dotnet@main -->

```bash
dotnet add package Temporalio.Extensions.Aws.Lambda.OpenTelemetry
```

Published in lockstep with `Temporalio` and `Temporalio.Extensions.Aws.Lambda` (all 1.18.0). Unlike Python, where OTel is an extra on the existing package (`temporalio[lambda-worker-otel]`), .NET requires the extra reference.

### OTel functions

The package contributes an extension method on the options object, applied inside the configure callback:

```csharp
TemporalLambdaWorker.CreateHandler(
    new WorkerDeploymentVersion("my-app", "build-1"),
    config =>
    {
        config.ApplyOpenTelemetryDefaults();
        config.WorkerOptions.TaskQueue = "my-task-queue";
        config.WorkerOptions.AddWorkflow<MyWorkflow>().AddActivity(MyActivities.MyActivity);
    });
```
<!-- verified against samples-dotnet@main src/LambdaWorker/Worker/Function.cs -->

`ApplyOpenTelemetryDefaults()` configures metrics and tracing against the ADOT layer's collector. As with the other SDKs, telemetry must be exported before the invocation ends — keep any metrics export interval shorter than the Lambda timeout.

### ADOT layer setup

Attach an **ADOT Collector layer** for the target region and architecture. No language-specific auto-instrumentation layer is needed, because the OpenTelemetry SDK arrives as an ordinary package dependency — the same situation as Go and Java. The sample's prerequisites list the collector layer ARN as something you supply per region.

`OPENTELEMETRY_COLLECTOR_CONFIG_URI=/var/task/otel-collector-config.yaml`. The sample copies `otel-collector-config.yaml` into the publish directory before zipping so it lands in the task root. <!-- inferred from the sample's deploy script copying the file into the publish output; the variable name is not stated in the .NET docs page read -->

### Telemetry IAM permissions

The sample ships an `enable-telemetry.sh` that adds an inline policy to the **execution** role and turns on active tracing — a concrete, copyable form of the permissions listed under "Required IAM permissions" in `observability.md`: <!-- verified against samples-dotnet@main src/LambdaWorker/Deploy/enable-telemetry.sh -->

- `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`, scoped to `arn:aws:logs:<region>:<account>:log-group:/aws/lambda/<function>:*`
- `xray:PutTraceSegments`, `xray:PutTelemetryRecords` on `*`
- `cloudwatch:PutMetricData` on `*`

It then runs `aws lambda update-function-configuration --tracing-config Mode=Active`, without which traces do not appear under the `AWS::Lambda::Function` filter in X-Ray.

For the shared Collector configuration, X-Ray enablement, and execution-role permissions, see `observability.md`.

## Diagnostic signatures

| SDK | Cause | Fix |
|---|---|---|
| .NET | `TemporalWorkerOptions.LoggerFactory` is unset and "defaults to the client logger factory", which is also unset — so `Workflow.Logger` output is discarded. Activity `Console.WriteLine` still reaches CloudWatch, which makes the gap look selective rather than total | set `config.WorkerOptions.LoggerFactory` (e.g. `LoggerFactory.Create(b => b.AddSimpleConsole().SetMinimumLevel(LogLevel.Information))`) |

**.NET — `DllNotFoundException` / missing `libtemporalio_sdk_core_c_bridge.so` at first invocation.** The .NET SDK wraps a native Rust core, and a portable (non-RID) publish omits its Linux build. Republish with an explicit runtime identifier matching the function's architecture (`--runtime linux-x64` for `x86_64`, `linux-arm64` for `arm64`) and check the file is in the publish output before zipping. → Build and package above.

**.NET — `NativeCertsNotFound` at first invocation, despite a correct address, Namespace, and API key.**

```
System.InvalidOperationException: Connection failed: Server connection error:
  tonic::transport::Error(Transport, NativeCertsNotFound)
   at Temporalio.Bridge.Client.ConnectAsync(...)
   at Temporalio.Client.TemporalConnection.ConnectAsync(...)
```

*Cause:* AWS's .NET 8 Lambda images force-override `SSL_CERT_FILE`, so the SDK's Rust core cannot load system root CAs. *Fix:* set `SSL_CERT_FILE=/etc/pki/tls/certs/ca-bundle.crt` (or `/etc/ssl/certs/ca-certificates.crt`) on the function, then recover the binding as described under "Failed first invocation" in `diagnostics.md` — the failed validation invocation means no Task Queue was bound and Temporal will not retry on its own.

"Certs" here means the operating system's root CA store, not any credential of yours: an API key auto-enables TLS, and TLS requires verifying the *server's* certificate chain. The connection fails before authentication is attempted, so the API key, Namespace, invocation role, and External ID are all irrelevant. Two discriminators: the same credentials work from a local Worker against the same Namespace, and the stack trace ends in `ConnectAsync` rather than a Temporal API call.

Only .NET is affected. Python uses the same Rust core (`temporalio/bridge/temporal_sdk_bridge.abi3.so`) but its runtime image does not override the variable; Java uses gRPC/Netty with the JVM truststore. <!-- verified: reproduced and fixed on a real deployment; cause per temporalio/sdk-dotnet README, "AWS Lambda .NET 8 CA Loading Issues", referencing aws/aws-lambda-dotnet#1661 -->

**.NET — handler not found at first invocation.** The .NET handler string has **three** colon-separated parts, `ASSEMBLY::NAMESPACE.TYPE::METHOD`, and is the only SDK with that shape — Java uses two, the rest use `module.function`. Compare against the assembly name (not the project name, if they differ) and the fully-qualified type.
