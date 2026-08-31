# SDK Configuration for Serverless Workers

<!-- Sources:
  docs/develop/go/workers/serverless-workers/aws-lambda.mdx
  docs/develop/python/workers/serverless-workers/aws-lambda.mdx
  docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx
-->

## Go SDK

### Package

Import: `lambdaworker "go.temporal.io/sdk/contrib/aws/lambdaworker"` <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:50 -->

Install: `go get go.temporal.io/sdk/contrib/aws/lambdaworker` — **this is a separate Go module** from `go.temporal.io/sdk`, versioned independently (`v0.1.1` at the time of writing). Having the main SDK in `go.mod` does not make it importable; add it explicitly, then `go mod tidy`. Verify the installed surface with `go doc go.temporal.io/sdk/contrib/aws/lambdaworker` before generating code — the API is Public Preview and drifts.

### Entry point

`lambdaworker.RunWorker` — starts a Lambda-based Worker. Pass a `WorkerDeploymentVersion` and a callback that registers Workflows and Activities. <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:39-40 -->

### Configure callback

The `Options` callback gives access to the same registration methods as a traditional Worker: `RegisterWorkflow`, `RegisterWorkflowWithOptions`, `RegisterActivity`, `RegisterActivityWithOptions`, and `RegisterNexusService`. <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:81 -->

### Versioning behavior

Set per-Workflow at registration time with `workflow.VersioningBehaviorPinned` or `workflow.VersioningBehaviorAutoUpgrade`. <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:78 -->
Or set a Worker-level default with `DefaultVersioningBehavior` in `DeploymentOptions`. <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:79 -->

### Lambda-tuned defaults

<!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:103-115 -->

| Setting | Lambda default |
|---|---|
| `MaxConcurrentActivityExecutionSize` | 2 |
| `MaxConcurrentWorkflowTaskExecutionSize` | 10 |
| `MaxConcurrentLocalActivityExecutionSize` | 2 |
| `MaxConcurrentNexusTaskExecutionSize` | 5 |
| `MaxConcurrentActivityTaskPollers` | 1 |
| `MaxConcurrentWorkflowTaskPollers` | 2 |
| `MaxConcurrentNexusTaskPollers` | 1 |
| `WorkerStopTimeout` | 5 seconds |
| `DisableEagerActivities` | Always true |
| Sticky cache size | 100 |
| `ShutdownDeadlineBuffer` | 7 seconds |

These are the same `worker.Options` available to any Temporal Worker, just with lower values for Lambda's constrained environment. Except for `ShutdownDeadlineBuffer`, which is specific to the `lambdaworker` package. <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:101,120 -->

`DisableEagerActivities` is always true and cannot be overridden. Eager Activities require a persistent connection, which Lambda invocations don't maintain. <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:117-118 -->

`ShutdownDeadlineBuffer` controls how much time before the Lambda deadline the Worker begins its graceful shutdown. The default is `WorkerStopTimeout` + 2 seconds. <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:120-122 -->

If your Worker handles long-running Activities, increase `WorkerStopTimeout`, `ShutdownDeadlineBuffer`, and the Lambda invocation deadline (`--timeout`) together. <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:124-125 -->

### Connection configuration

The `lambdaworker` package automatically loads Temporal client configuration from a TOML config file and environment variables (see the Environment Configuration docs, `/develop/environment-configuration`). <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:85 -->

TOML config file resolution order: <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:87-91 -->

1. `TEMPORAL_CONFIG_FILE` environment variable, if set.
2. `temporal.toml` in `$LAMBDA_TASK_ROOT` (typically `/var/task`).
3. `temporal.toml` in the current working directory.

The file is optional. If absent, only environment variables are used. <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:93 -->

---

## Python SDK

### Package

Import: `from temporalio.contrib.aws.lambda_worker import LambdaWorkerConfig, run_worker` <!-- docs/develop/python/workers/serverless-workers/aws-lambda.mdx:47 -->

Install: `pip install temporalio` — the contrib module ships inside the main package here (unlike Go and TypeScript, which need a separate dependency). Use `temporalio[lambda-worker-otel]` for OpenTelemetry support.

### Entry point

`run_worker` — takes a `WorkerDeploymentVersion` and a configure callback, returns a Lambda handler. <!-- docs/develop/python/workers/serverless-workers/aws-lambda.mdx:39-40,66 -->

### Configure callback

The `configure` callback receives a `LambdaWorkerConfig` dataclass with fields pre-populated with Lambda-appropriate defaults. Set the Task Queue, Workflows, and Activities through `worker_config`, which accepts the same keyword arguments as the `Worker` constructor. <!-- docs/develop/python/workers/serverless-workers/aws-lambda.mdx:71-72 -->

### Versioning behavior

Set per-Workflow in the `@workflow.defn` decorator: `VersioningBehavior.PINNED` or `VersioningBehavior.AUTO_UPGRADE`. <!-- docs/develop/python/workers/serverless-workers/aws-lambda.mdx:74-75 -->
Or set a Worker-level default with `default_versioning_behavior` in the worker config. <!-- docs/develop/python/workers/serverless-workers/aws-lambda.mdx:75 -->

### Lambda-tuned defaults

<!-- docs/develop/python/workers/serverless-workers/aws-lambda.mdx:108-120 -->

| Setting | Lambda default |
|---|---|
| `max_concurrent_activities` | 2 |
| `max_concurrent_workflow_tasks` | 10 |
| `max_concurrent_local_activities` | 2 |
| `max_concurrent_nexus_tasks` | 5 |
| `workflow_task_poller_behavior` | `SimpleMaximum(2)` |
| `activity_task_poller_behavior` | `SimpleMaximum(1)` |
| `nexus_task_poller_behavior` | `SimpleMaximum(1)` |
| `graceful_shutdown_timeout` | 5 seconds |
| `max_cached_workflows` | 30 |
| `disable_eager_activity_execution` | Always `True` |
| `shutdown_deadline_buffer` | 7 seconds |

`disable_eager_activity_execution` is always `True` and cannot be overridden. Eager Activities require a persistent connection, which Lambda invocations don't maintain. <!-- docs/develop/python/workers/serverless-workers/aws-lambda.mdx:122-123 -->

`shutdown_deadline_buffer` is specific to the `lambda_worker` package. It controls how much time before the Lambda deadline the Worker begins its graceful shutdown. The default is `graceful_shutdown_timeout` + 2 seconds. <!-- docs/develop/python/workers/serverless-workers/aws-lambda.mdx:125-127 -->

If your Worker handles long-running Activities, increase `graceful_shutdown_timeout`, `shutdown_deadline_buffer`, and the Lambda invocation deadline (`--timeout`) together. <!-- docs/develop/python/workers/serverless-workers/aws-lambda.mdx:129-130 -->

### Connection configuration

The `lambda_worker` package automatically loads Temporal client configuration from a TOML config file and environment variables (see the Environment Configuration docs, `/develop/environment-configuration`). <!-- docs/develop/python/workers/serverless-workers/aws-lambda.mdx:91 -->

TOML config file resolution order: <!-- docs/develop/python/workers/serverless-workers/aws-lambda.mdx:93-97 -->

1. `TEMPORAL_CONFIG_FILE` environment variable, if set.
2. `temporal.toml` in `$LAMBDA_TASK_ROOT` (typically `/var/task`).
3. `temporal.toml` in the current working directory.

The file is optional. If absent, only environment variables are used. <!-- docs/develop/python/workers/serverless-workers/aws-lambda.mdx:99 -->

---

## Java SDK

### Package

Import: `io.temporal.aws.lambda.LambdaWorker`, `io.temporal.aws.lambda.LambdaWorkerOptions`, `io.temporal.common.WorkerDeploymentVersion` <!-- verified against io.temporal:temporal-aws-lambda:1.38.0 sources jar -->

Install: `io.temporal:temporal-aws-lambda` — a **separate Maven artifact** from `io.temporal:temporal-sdk`, but published on the **same version line** (both 1.38.0). This is a third packaging pattern: unlike Go and TypeScript it is not independently versioned, and unlike Python it does not ship inside the main SDK. Use `io.temporal:temporal-bom` in `dependencyManagement` to keep them aligned.

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>io.temporal</groupId><artifactId>temporal-bom</artifactId>
      <version>1.38.0</version><type>pom</type><scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

`aws-lambda-java-core` (1.4.0) arrives transitively from `temporal-aws-lambda`; declare it explicitly if you compile against `RequestHandler`/`Context`.

### Entry point

**`LambdaWorker.define(version, configure)`** — returns a `RequestHandler<Object, Void>` that your handler class delegates to. There are four public overloads: `define` (2- and 3-arg) and `newHandler` (2- and 3-arg, taking a pre-built `LambdaWorkerOptions`).

Note that Java's entry point is not "run"-shaped like the other SDKs' (`RunWorker`, `run_worker`, `runWorker`) — confirm the method name against the version you install. <!-- verified against io.temporal:temporal-aws-lambda:1.37.0 and 1.38.0, and samples-java@main -->

### Configure callback — two phases, unlike the other SDKs

Java splits configuration in a way no other SDK does, and the distinction matters:

- The `Consumer<LambdaWorkerOptions.Builder>` passed to `define` is *"invoked once while the Lambda handler is constructed"* — i.e. at **cold start**, once per container, **not** per invocation.
- Per-invocation configuration is a **separate** callback, `LambdaWorker.InvocationConfigurator`, taking `(LambdaWorkerOptions.Builder, com.amazonaws.services.lambda.runtime.Context)`, *"invoked for each Lambda invocation before Temporal service stubs, client, and worker are created."* Use the 3-arg `define` overload for it.

Contrast Python, whose `configure` runs **once per invocation**. Do not describe them as equivalent, and do not carry per-invocation logic into Java's cold-start callback.

Registration methods on `LambdaWorkerOptions.Builder`: `setTaskQueue`, `registerWorkflowImplementationTypes`, `registerDynamicWorkflowImplementationType`, `registerWorkflowImplementationFactory` (3 overloads), `registerActivitiesImplementations`, `registerDynamicActivityImplementation`, `registerNexusServiceImplementation`, `addShutdownHook`, plus `getWorkerOptionsBuilder()` / `getWorkflowClientOptionsBuilder()` / `getWorkflowServiceStubsOptionsBuilder()` for lower-level tuning.

### Versioning behavior

Per-Workflow via the **annotation** `io.temporal.workflow.WorkflowVersioningBehavior` on the workflow method, taking `io.temporal.common.VersioningBehavior.PINNED` or `.AUTO_UPGRADE`:

```java
public class GreetingWorkflowImpl implements GreetingWorkflow {
  @Override
  @WorkflowVersioningBehavior(VersioningBehavior.PINNED)
  public String getGreeting(String name) { ... }
}
```

Or a Worker-level default with `DefaultVersioningBehavior` in `DeploymentOptions`.

### Lambda-tuned defaults

<!-- verified in io.temporal:temporal-aws-lambda:1.38.0, LambdaWorkerOptions.java:40-53 -->

| Setting | Lambda default |
|---|---|
| `MaxConcurrentActivityExecutionSize` | 2 |
| `MaxConcurrentWorkflowTaskExecutionSize` | 10 |
| `MaxConcurrentLocalActivityExecutionSize` | 2 |
| `MaxConcurrentNexusExecutionSize` | 5 |
| `MaxConcurrentWorkflowTaskPollers` | 2 |
| `MaxConcurrentActivityTaskPollers` | 1 |
| `MaxConcurrentNexusTaskPollers` | 1 |
| `WorkflowCacheSize` | 30 |
| `MaxWorkflowThreadCount` | 30 |
| `GracefulShutdownTimeout` | 5 seconds |
| `ShutdownDeadlineBuffer` | 7 seconds |

`MaxWorkflowThreadCount` has no counterpart in the other SDKs — Java runs Workflow code on real threads.

Eager Activities are disabled: `builder.setDisableEagerExecution(true)` (`LambdaWorkerOptions.java:258`). `ShutdownDeadlineBuffer` defaults to `GracefulShutdownTimeout` + 2s, the same relationship as the other SDKs.

### Logging — the binding must be SLF4J 1.7.x

The Java SDK compiles against `org.slf4j:slf4j-api:1.7.36`. A 2.x provider (`slf4j-simple:2.x`, Logback 1.3+) **will not bind to a 1.7 API**, and the Worker runs with no logs at all — the same silent outcome as Python's `logging.basicConfig()` no-op, by a different mechanism. Use a 1.7.x provider:

```xml
<dependency>
  <groupId>org.slf4j</groupId><artifactId>slf4j-simple</artifactId><version>1.7.36</version>
</dependency>
```

With a correct binding the module logs its own lifecycle unprompted, which is more than the other SDKs give you by default:

```
[main] INFO io.temporal.aws.lambda.LambdaWorker - Temporal Lambda worker started
  awsRequestId=<id> invokedFunctionArn=<arn> taskQueue=<tq> identity=<id>@<arn>
```

### Connection configuration

Loaded automatically from environment variables and an optional TOML config file. Public constants on `LambdaWorkerOptions`: `TEMPORAL_TASK_QUEUE`, `TEMPORAL_CONFIG_FILE`, `LAMBDA_TASK_ROOT`.

Resolution order (`LambdaWorkerOptions.resolveConfigFilePath`):

1. `TEMPORAL_CONFIG_FILE` environment variable, if set.
2. `temporal.toml` in `$LAMBDA_TASK_ROOT` (typically `/var/task`).
3. `temporal.toml` in the current working directory.

**`HOME=/tmp` is not required for Java** — unlike the Go and TypeScript examples. The module never reads `HOME`; when no file is found it passes a null path to `ClientConfig.load`, which falls back to `<user.home>/.config/temporalio/temporal.toml` and treats both a missing home directory and a `FileNotFoundException` as "empty config, no error". It is a read, not a write, so Lambda's read-only filesystem is not involved. Note also that Java reads the `user.home` **system property**, not the `HOME` environment variable, so setting `HOME` is not even the right lever — use `-Duser.home` via `JAVA_TOOL_OPTIONS` if you ever need to steer it.

---

## .NET SDK

### Package

Import: `using Temporalio.Extensions.Aws.Lambda;` plus `Temporalio.Common` (for `WorkerDeploymentVersion`) and `Amazon.Lambda.Core` (for `ILambdaContext`). <!-- verified against Temporalio.Extensions.Aws.Lambda 1.18.0 XML docs and samples-dotnet@main -->

Install: `dotnet add package Temporalio.Extensions.Aws.Lambda` — a **separate NuGet package** from `Temporalio`, published in **lockstep** with it (both 1.18.0), the same relationship Java has. Published versions: 1.17.0 and 1.18.0. The package targets `netstandard2.0` and declares `Temporalio` 1.18.0 and `Amazon.Lambda.Core` 3.1.0.

OpenTelemetry lives in a **second package**, `Temporalio.Extensions.Aws.Lambda.OpenTelemetry` (also 1.18.0) — unlike Python, where OTel is an extra on the same package. → `<provider>/observability.md`.

### Entry point

**`TemporalLambdaWorker.CreateHandler(version, configure)`** — returns a `Func<object?, ILambdaContext, Task>` that your handler method delegates to. Overloads take either a synchronous `Action<TemporalLambdaWorkerOptions>` or an asynchronous `Func<TemporalLambdaWorkerOptions, Task>` for setup that must await. A further overload takes `TemporalLambdaWorkerHandlerOptions`, which the XML docs describe as "internal test seams" — not for production use.

```csharp
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
                    .AddActivity(Activities.MyActivity);
            });

    public Task HandlerAsync(Stream input, ILambdaContext context) =>
        WorkerHandler(input, context);
}
```

`TemporalLambdaWorker.LoadClientConnectOptions(...)` is also public, for loading connection options with Lambda-aware config resolution outside the handler.

### Configure callback

Receives a `TemporalLambdaWorkerOptions` with `ClientOptions`, `WorkerOptions`, `ShutdownDeadlineBuffer`, `ShutdownHooks`, and `AddShutdownHook(Func<CancellationToken, Task>)`. The Task Queue and registrations go through `WorkerOptions` — an ordinary `TemporalWorkerOptions`, so `TaskQueue`, `AddWorkflow<T>()` and `AddActivity(...)` behave exactly as they do for a long-lived Worker. The callback runs **per invocation** (Java is the outlier that runs its at cold start).

### Versioning behavior

Per-Workflow via the `[Workflow]` attribute:

```csharp
[Workflow(VersioningBehavior = VersioningBehavior.Pinned)]
public class MyWorkflow { ... }
```

Or a Worker-level default through `DefaultVersioningBehavior` in `DeploymentOptions`.

**The .NET Worker-level default is `AutoUpgrade`**, whereas TypeScript's is `PINNED`. Defaults are not uniform across SDKs — never state one globally, and prefer setting the behavior explicitly per Workflow.

### Lambda-tuned defaults

<!-- docs/develop/dotnet/workers/serverless-workers/aws-lambda.mdx (line unverified); values are internal to the assembly and not artifact-verifiable -->

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

### Native dependency — publish must be RID-specific

The .NET SDK wraps a **native Rust core** (`libtemporalio_sdk_core_c_bridge.so`). A portable publish does not include the Linux build of it, and the failure appears only at first invocation. Always publish for an explicit runtime identifier matching the function's architecture:

| `--runtime` | `--architectures` |
|---|---|
| `linux-x64` | `x86_64` |
| `linux-arm64` | `arm64` |

This is .NET's equivalent of Python's `manylinux` wheels and Go's `GOARCH`. Temporal's own deploy script asserts the file is present before zipping, which is worth copying:

```bash
[[ -f "$PUBLISH_DIR/libtemporalio_sdk_core_c_bridge.so" ]] || {
  echo "Publish output is missing the $TARGET_RUNTIME Temporal native bridge." >&2; exit 1; }
```
<!-- verified against samples-dotnet@main src/LambdaWorker/Deploy/deploy-lambda.sh -->

### Connection configuration

Loaded automatically from environment variables and an optional TOML config file, with the same resolution order as the other SDKs:

1. `TEMPORAL_CONFIG_FILE` environment variable, if set.
2. `temporal.toml` in the Lambda task root (typically `/var/task`).
3. `temporal.toml` in the current working directory.

The sample copies a `temporal.toml` into the publish directory before zipping, so it lands in the task root, and keeps the API key in `TEMPORAL_API_KEY` rather than in the file. Supplying an API key enables TLS automatically.

**TLS caveat specific to .NET:** some AWS Lambda .NET images override `SSL_CERT_FILE` in a way that prevents the SDK's Rust-based runtime from loading system root CAs. It surfaces as a TLS failure at first invocation. → `<provider>/diagnostics.md`.

---

## TypeScript SDK

### Package

Import: `import { runWorker } from '@temporalio/lambda-worker'` <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:45 -->

Install: `npm install @temporalio/lambda-worker` — a separate npm package from `@temporalio/worker`, versioned independently.

### Entry point

`runWorker` — creates a Lambda handler that runs a Temporal Worker. Pass a deployment version and a configure callback. <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:39-40 -->

### Configure callback

Set Worker options via `config.workerOptions`. For Workflow code, use `workflowBundle` with pre-bundled code instead of `workflowsPath` to avoid webpack bundling overhead on Lambda cold starts. <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:68-69 -->

### Pre-bundling Workflow code

Build the bundle as a separate build step: <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:71 -->

```typescript
import { bundleWorkflowCode } from '@temporalio/worker';
import { writeFile } from 'fs/promises';

const { code } = await bundleWorkflowCode({
  workflowsPath: require.resolve('./workflows'),
});
await writeFile('./workflow-bundle.js', code);
```
<!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:73-81 -->

Then reference the bundle in your handler with `workflowBundle: { codePath: require.resolve('./workflow-bundle.js') }`. <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:83 -->

### Versioning behavior

Set per-Workflow with `setWorkflowOptions` in the Workflow file, or set a default for all Workflows with `defaultVersioningBehavior` in the configure callback. <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:172-174 -->
Values are `'PINNED'` or `'AUTO_UPGRADE'`. The default versioning behavior is `PINNED`. <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:63-64 -->

Access via: `config.workerOptions.workerDeploymentOptions!.defaultVersioningBehavior = 'PINNED'` <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:64 (full expression at docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:165) -->

### Lambda-tuned defaults

<!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:104-115 -->

| Setting | Lambda default |
|---|---|
| `maxConcurrentActivityTaskExecutions` | 2 |
| `maxConcurrentWorkflowTaskExecutions` | 10 |
| `maxConcurrentLocalActivityExecutions` | 2 |
| `maxConcurrentNexusTaskExecutions` | 5 |
| `workflowTaskPollerBehavior` | `SimpleMaximum(2)` |
| `activityTaskPollerBehavior` | `SimpleMaximum(1)` |
| `nexusTaskPollerBehavior` | `SimpleMaximum(1)` |
| `shutdownGraceTime` | 5 seconds |
| `maxCachedWorkflows` | 30 |
| `shutdownDeadlineBufferMs` | 7000 |

Eager Activities are not supported. Lambda invocations don't maintain persistent connections. <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:117 -->

`shutdownDeadlineBufferMs` is specific to the `@temporalio/lambda-worker` package. It controls how much time before the Lambda deadline the Worker begins its graceful shutdown. The default is `shutdownGraceTime` (5s) + 2s. <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:119-121 -->

If your Worker handles long-running Activities, increase `shutdownGraceTime`, `shutdownDeadlineBufferMs`, and the Lambda invocation deadline (`--timeout`) together. <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:123-124 -->

### Connection configuration

The `@temporalio/lambda-worker` package automatically loads Temporal client configuration from a TOML config file and environment variables (see the Environment Configuration docs, `/develop/environment-configuration`). <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:87 -->

TOML config file resolution order: <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:89-93 -->

1. `TEMPORAL_CONFIG_FILE` environment variable, if set.
2. `temporal.toml` in `$LAMBDA_TASK_ROOT` (typically `/var/task`).
3. `temporal.toml` in the current working directory.

The file is optional. If absent, only environment variables are used. <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:95 -->

---

## Cross-SDK comparison: Lambda-tuned defaults

| Concept | Go | Python | TypeScript | Java | .NET |
|---|---|---|---|---|---|
| Max concurrent activities | `MaxConcurrentActivityExecutionSize` = 2 | `max_concurrent_activities` = 2 | `maxConcurrentActivityTaskExecutions` = 2 | `MaxConcurrentActivityExecutionSize` = 2 | `MaxConcurrentActivities` = 2 |
| Max concurrent workflow tasks | `MaxConcurrentWorkflowTaskExecutionSize` = 10 | `max_concurrent_workflow_tasks` = 10 | `maxConcurrentWorkflowTaskExecutions` = 10 | `MaxConcurrentWorkflowTaskExecutionSize` = 10 | `MaxConcurrentWorkflowTasks` = 10 |
| Sticky cache size | 100 | `max_cached_workflows` = 30 | `maxCachedWorkflows` = 30 | `WorkflowCacheSize` = 30 | `MaxCachedWorkflows` = 30 |
| Worker stop timeout | `WorkerStopTimeout` = 5s | `graceful_shutdown_timeout` = 5s | `shutdownGraceTime` = 5s | `GracefulShutdownTimeout` = 5s | `GracefulShutdownTimeout` = 5s |
| Shutdown deadline buffer | `ShutdownDeadlineBuffer` = 7s | `shutdown_deadline_buffer` = 7s | `shutdownDeadlineBufferMs` = 7000 | `ShutdownDeadlineBuffer` = 7s | `ShutdownDeadlineBuffer` = 7s |
| Eager activities | `DisableEagerActivities` always true | `disable_eager_activity_execution` always `True` | Not supported | `setDisableEagerExecution(true)` | `DisableEagerActivityExecution` always `true` |
| Entry point | `RunWorker` | `run_worker` | `runWorker` | `LambdaWorker.define` | `TemporalLambdaWorker.CreateHandler` |
| Configure callback runs | per invocation | per invocation | per invocation | **at cold start**; separate `InvocationConfigurator` for per-invocation | per invocation |
| Package relationship to SDK | separate module, own version line | inside main SDK | separate npm package, own version line | separate artifact, **same version line** | separate package, **same version line** |
| Default versioning behavior | set explicitly | set explicitly | `PINNED` | set explicitly | **`AutoUpgrade`** |
| Platform-specific build artifact | `GOOS`/`GOARCH` static binary | `manylinux` wheels | none (JS) | none (bytecode; match `release` to runtime) | **native `libtemporalio_sdk_core_c_bridge.so`** — RID-specific publish |

Java-only: `MaxWorkflowThreadCount` = 30 (no counterpart elsewhere — Java runs Workflow code on real threads).

**Defaults are not uniform.** TypeScript defaults to `PINNED`, .NET to `AutoUpgrade`. Set the behavior explicitly per Workflow rather than relying on any of them.

<!-- Go: docs/develop/go/workers/serverless-workers/aws-lambda.mdx:103-115 -->
<!-- Python: docs/develop/python/workers/serverless-workers/aws-lambda.mdx:108-120 -->
<!-- TypeScript: docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:104-115 -->

Note: Go sticky cache size is 100, while Python and TypeScript are 30. These values come from each SDK's own docs and are not interchangeable.
