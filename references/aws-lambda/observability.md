# Observability for Serverless Workers

<!-- Sources:
  docs/develop/go/workers/serverless-workers/aws-lambda.mdx
  docs/develop/python/workers/serverless-workers/aws-lambda.mdx
  docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx
-->

## Overview

Each SDK provides an OpenTelemetry integration package with defaults configured for the AWS Distro for OpenTelemetry (ADOT) Lambda layer. When enabled, the Worker emits SDK metrics and distributed traces for Workflow and Activity executions. The ADOT Lambda layer collects this telemetry and can forward traces to AWS X-Ray and metrics to Amazon CloudWatch. <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:129-131 -->

## Go SDK

### OTel package

Import: `otel "go.temporal.io/sdk/contrib/aws/lambdaworker/otel"` <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:145 -->

### OTel functions

- `otel.ApplyDefaults` — configures both metrics and tracing. <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:157,172 -->
- `otel.ApplyMetrics` — configures metrics only. <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:240 -->
- `otel.ApplyTracing` — configures tracing only. <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:240 -->

Usage in the configure callback: <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:154-159 -->

```go
if err := otel.ApplyDefaults(opts, &opts.ClientOptions, otel.Options{}); err != nil {
    return err
}
```

By default, telemetry is sent to `localhost:4317`, which is the ADOT Lambda layer's default collector endpoint. <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:173 -->

### ADOT layer setup (Go)

Attach the ADOT Collector layer to your Lambda function. <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:175 -->
Go does not need a language-specific ADOT layer because the OTel SDK is compiled into the binary. <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:177 -->

### Collector config env var (Go)

`OPENTELEMETRY_COLLECTOR_CONFIG_URI=/var/task/otel-collector-config.yaml` <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:226 -->

---

## Python SDK

### OTel package

Import: `from temporalio.contrib.aws.lambda_worker.otel import apply_defaults` <!-- docs/develop/python/workers/serverless-workers/aws-lambda.mdx:147 -->

To install with OTel support: `pip install temporalio[lambda-worker-otel]` <!-- docs/production-deployment/worker-deployments/serverless-workers/aws-lambda.mdx:214 -->

### OTel functions

- `apply_defaults` — configures both metrics and tracing. <!-- docs/develop/python/workers/serverless-workers/aws-lambda.mdx:155,165 -->
- `build_metrics_telemetry_config` — configures metrics only. <!-- docs/develop/python/workers/serverless-workers/aws-lambda.mdx:232 -->
- `apply_tracing` — configures tracing only. <!-- docs/develop/python/workers/serverless-workers/aws-lambda.mdx:232 -->

Usage in the configure callback: <!-- docs/develop/python/workers/serverless-workers/aws-lambda.mdx:151-155 -->

```python
def configure(config: LambdaWorkerConfig) -> None:
    config.worker_config["task_queue"] = TASK_QUEUE
    config.worker_config["workflows"] = [SampleWorkflow]
    config.worker_config["activities"] = [hello_activity]
    apply_defaults(config)
```

By default, telemetry is sent to `localhost:4317`, which is the ADOT Lambda layer's default collector endpoint. <!-- docs/develop/python/workers/serverless-workers/aws-lambda.mdx:166 -->

### ADOT layer setup (Python)

Attach the ADOT Python Lambda layer to your Lambda function. The layer includes both auto-instrumentation and an OpenTelemetry Collector that receives telemetry on `localhost:4317` and forwards traces to AWS X-Ray and metrics to Amazon CloudWatch. <!-- docs/develop/python/workers/serverless-workers/aws-lambda.mdx:168-169 -->

### Collector config env var (Python)

`OPENTELEMETRY_COLLECTOR_CONFIG_FILE=/var/task/otel-collector-config.yaml` <!-- docs/develop/python/workers/serverless-workers/aws-lambda.mdx:218 -->

Note: Python uses `_FILE` while Go and TypeScript use `_URI`.

---

## Java SDK

### OTel package

No extra dependency is required: `temporal-aws-lambda` already depends on `io.temporal:temporal-opentelemetry`, `io.opentelemetry:opentelemetry-api` (BOM 1.25.0), and `io.opentelemetry.contrib:opentelemetry-aws-xray`. The helper class ships inside the module. <!-- verified in io.temporal:temporal-aws-lambda:1.38.0 pom + jar -->

Import: `io.temporal.aws.lambda.OtelLambdaWorkerConfigurationHelper`

### OTel functions

<!-- verified with javap against io.temporal:temporal-aws-lambda:1.38.0 -->

- `configure(LambdaWorkerOptions.Builder)` — configures metrics and tracing with defaults.
- `configure(LambdaWorkerOptions.Builder, Consumer<Builder>)` — same, with customization.
- `configureMetrics(LambdaWorkerOptions.Builder, OpenTelemetry)` — metrics only (an overload also takes a service name and a `Duration` report interval).
- `configureTracing(LambdaWorkerOptions.Builder, OpenTelemetry)` — tracing only.
- `configureFlushHook(LambdaWorkerOptions.Builder, OpenTelemetry, Duration)` — registers a flush before the invocation ends.

Its own `newBuilder()` exposes `setOpenTelemetry`, `setEndpoint`, `setServiceName`, `setMetricsReportInterval`, `setFlushTimeout`, and `setFlushHook`.

Usage in the cold-start configure callback:

```java
LambdaWorker.define(
    new WorkerDeploymentVersion("my-app", "build-1"),
    builder -> {
      builder.setTaskQueue("my-task-queue");
      builder.registerWorkflowImplementationTypes(MyWorkflowImpl.class);
      builder.registerActivitiesImplementations(new MyActivitiesImpl());
      OtelLambdaWorkerConfigurationHelper.configure(builder);
    });
```

Defaults come from the constants `DEFAULT_OTLP_ENDPOINT` and `DEFAULT_SERVICE_NAME`, and the helper reads `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_SERVICE_NAME`, and `AWS_LAMBDA_FUNCTION_NAME` from the environment. As with the other SDKs, the default endpoint is the ADOT layer's collector on `localhost:4317`.

**Flush before the deadline.** A Serverless Worker's invocation ends on a deadline rather than after a request, so telemetry buffered past that point is lost. Use `configureFlushHook` (or a report interval shorter than the invocation deadline) so metrics and spans are exported before shutdown. This matters more on Java's shorter recommended deadline (90s) than on the 600s used elsewhere.

### ADOT layer setup (Java)

Attach the ADOT Collector layer. Because the OpenTelemetry SDK arrives as an ordinary Maven dependency of `temporal-aws-lambda`, no language-specific auto-instrumentation layer is required for the Worker's own telemetry — the same situation as Go. <!-- inferred from the artifact's dependency graph (temporal-aws-lambda:1.38.0 pom), not stated in the docs; confirm against a deployed function before relying on it -->

### Collector config env var (Java)

`OPENTELEMETRY_COLLECTOR_CONFIG_URI=/var/task/otel-collector-config.yaml` — the `_URI` form, as with Go and TypeScript. The official Java sample packages `otel-collector-config.template.yaml` into the artifact root as `otel-collector-config.yaml` during `shadowJar`; with Maven, add it under `src/main/resources`.

---

## .NET SDK

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
    new WorkerDeploymentVersion(deploymentName, buildId),
    config =>
    {
        config.ApplyOpenTelemetryDefaults();
        config.WorkerOptions.TaskQueue = taskQueue;
        config.WorkerOptions.AddWorkflow<SampleWorkflow>().AddActivity(Activities.HelloActivity);
    });
```
<!-- verified against samples-dotnet@main src/LambdaWorker/Worker/Function.cs -->

`ApplyOpenTelemetryDefaults()` configures metrics and tracing against the ADOT layer's collector. As with the other SDKs, telemetry must be exported before the invocation ends — keep any metrics export interval shorter than the Lambda timeout.

### ADOT layer setup (.NET)

Attach an **ADOT Collector layer** for the target region and architecture. No language-specific auto-instrumentation layer is needed, because the OpenTelemetry SDK arrives as an ordinary package dependency — the same situation as Go and Java. The sample's prerequisites list the collector layer ARN as something you supply per region.

### Telemetry IAM permissions (.NET)

The sample ships an `enable-telemetry.sh` that adds an inline policy to the **execution** role and turns on active tracing — a concrete, copyable form of the permissions listed under "Required IAM permissions" below: <!-- verified against samples-dotnet@main src/LambdaWorker/Deploy/enable-telemetry.sh -->

- `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`, scoped to `arn:aws:logs:<region>:<account>:log-group:/aws/lambda/<function>:*`
- `xray:PutTraceSegments`, `xray:PutTelemetryRecords` on `*`
- `cloudwatch:PutMetricData` on `*`

It then runs `aws lambda update-function-configuration --tracing-config Mode=Active`, without which traces do not appear under the `AWS::Lambda::Function` filter in X-Ray.

### Collector config env var (.NET)

`OPENTELEMETRY_COLLECTOR_CONFIG_URI=/var/task/otel-collector-config.yaml`. The sample copies `otel-collector-config.yaml` into the publish directory before zipping so it lands in the task root. <!-- inferred from the sample's deploy script copying the file into the publish output; the variable name is not stated in the .NET docs page read -->

---

## TypeScript SDK

### OTel package

Import: `import { applyDefaults } from '@temporalio/lambda-worker/otel'` <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:138 -->

### OTel functions

- `applyDefaults` — registers Temporal SDK interceptors for tracing and configures the Core SDK to export metrics via OTLP. <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:148,153 -->
- `makeOtelPlugin` — returns a plugin for pre-bundling Workflow code that includes Workflow interceptor modules. <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:222,226 -->

Usage in the configure callback: <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:142-149 -->

```typescript
export const handler = runWorker({ deploymentName: 'sdk-demo', buildId: 'v1' }, (config) => {
  config.workerOptions.taskQueue = TASK_QUEUE;
  config.workerOptions.workflowBundle = {
    codePath: require.resolve('./workflow-bundle.js'),
  };
  config.workerOptions.activities = activities;
  applyDefaults(config);
});
```

By default, telemetry is sent to `localhost:4317`, which is the ADOT Lambda layer's default collector endpoint. <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:154 -->

### Pre-bundling with OTel

When pre-bundling Workflow code, pass the plugin from `makeOtelPlugin()` so that Workflow interceptor modules are included in the bundle: <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:222 -->

```typescript
import { bundleWorkflowCode } from '@temporalio/worker';
import { makeOtelPlugin } from '@temporalio/lambda-worker/otel';

const { plugin } = makeOtelPlugin();
const { code } = await bundleWorkflowCode({
  workflowsPath: require.resolve('./workflows'),
  plugins: [plugin],
});
```
<!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:224-233 -->

### ADOT layer setup (TypeScript)

Attach two ADOT Lambda layers: <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:156 -->

1. The ADOT JavaScript layer for Node.js-side auto-instrumentation and trace export. <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:158 -->
2. The ADOT Collector layer (`aws-otel-collector-amd64`) to run the OTel Collector as a Lambda extension, receiving telemetry via OTLP on `localhost:4317` and forwarding traces to X-Ray and metrics to CloudWatch. <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:159 -->

### Collector config env var (TypeScript)

`OPENTELEMETRY_COLLECTOR_CONFIG_URI=/var/task/otel-collector-config.yaml` <!-- docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:208 -->

---

## Common across all SDKs

### Custom Collector configuration required

The default ADOT Collector configuration does not route OpenTelemetry Protocol (OTLP) data to the traces pipeline. You must provide a custom Collector configuration that wires the OTLP receiver to both the traces and metrics pipelines. <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:179-180 -->

Example `otel-collector-config.yaml` (bundle in your Lambda deployment package): <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:181 -->

```yaml
receivers:
    otlp:
        protocols:
            grpc:
                endpoint: "localhost:4317"
            http:
                endpoint: "localhost:4318"

exporters:
    debug:
    awsxray:
        region: us-west-2
    awsemf:
        namespace: TemporalWorkerMetrics
        log_group_name: /aws/lambda/<your-function-name>
        region: us-west-2
        dimension_rollup_option: NoDimensionRollup
        resource_to_telemetry_conversion:
            enabled: true

service:
    pipelines:
        traces:
            receivers: [otlp]
            exporters: [awsxray, debug]
        metrics:
            receivers: [otlp]
            exporters: [awsemf]
    telemetry:
        logs:
            level: debug
        metrics:
            address: localhost:8888
```
<!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:185-221 -->

### Enable X-Ray active tracing

```bash
aws lambda update-function-configuration \
  --function-name <your-function-name> \
  --tracing-config Mode=Active
```
<!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:230-234 -->

### Required IAM permissions

The Lambda execution role must have permissions to write to X-Ray and CloudWatch: <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:236 -->

- `xray:PutTraceSegments` <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:237 -->
- `xray:PutTelemetryRecords` <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:237 -->
- `cloudwatch:PutMetricData` <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:237 -->

Without these permissions, the Collector fails silently and no telemetry appears. <!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:238 -->

For Python, the `AWSXRayDaemonWriteAccess` managed policy can be attached instead. <!-- docs/develop/python/workers/serverless-workers/aws-lambda.mdx:229 -->

### Collector config env var summary

<!-- docs/develop/go/workers/serverless-workers/aws-lambda.mdx:226, docs/develop/python/workers/serverless-workers/aws-lambda.mdx:218, docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:208 -->

| SDK | Environment variable |
|---|---|
| Go | `OPENTELEMETRY_COLLECTOR_CONFIG_URI` |
| Python | `OPENTELEMETRY_COLLECTOR_CONFIG_FILE` |
| TypeScript | `OPENTELEMETRY_COLLECTOR_CONFIG_URI` |
| Java | `OPENTELEMETRY_COLLECTOR_CONFIG_URI` |
| .NET | `OPENTELEMETRY_COLLECTOR_CONFIG_URI` (config file copied into the task root by the sample's deploy script) |

### ADOT layer summary

| SDK | Layers needed |
|---|---|
| Go | ADOT Collector layer only (no language-specific layer; OTel SDK is compiled into the binary) |
| Python | ADOT Python Lambda layer (includes collector and auto-instrumentation) |
| TypeScript | ADOT JavaScript layer + ADOT Collector layer (`aws-otel-collector-amd64`) |
| Java | ADOT Collector layer only (no language-specific layer; the OTel SDK is a Maven dependency of `temporal-aws-lambda`) |
| .NET | ADOT Collector layer only (no language-specific layer; OTel arrives via `Temporalio.Extensions.Aws.Lambda.OpenTelemetry`) |

<!-- Go: docs/develop/go/workers/serverless-workers/aws-lambda.mdx:175-177 -->
<!-- Python: docs/develop/python/workers/serverless-workers/aws-lambda.mdx:168-169 -->
<!-- TypeScript: docs/develop/typescript/workers/serverless-workers/aws-lambda.mdx:156-159 -->
