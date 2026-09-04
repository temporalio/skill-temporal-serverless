# GCP Cloud Run — observability

<!-- Sources:
  docs/develop/<sdk>/workers/serverless-workers/cloud-run.mdx
  docs/production-deployment/worker-deployments/serverless-workers/cloud-run/index.mdx
-->

## There is nothing serverless-specific to configure

**A Cloud Run Serverless Worker emits the same traces and metrics as a Worker anywhere else.** It is an ordinary long-lived Worker, so the SDK's normal metrics and OpenTelemetry tracing setup applies unchanged, and each SDK's general observability guide is the right reference. <!-- docs/develop/<sdk>/workers/serverless-workers/cloud-run.mdx, "Add observability" -->

This is a genuine simplification over AWS Lambda, and the contrast is worth keeping in mind:

| | AWS Lambda | GCP Cloud Run |
|---|---|---|
| OTel wiring | Per-SDK helper in the serverless package (`ApplyDefaults`, `apply_defaults`, `OtelLambdaWorkerConfigurationHelper`, a separate `…Aws.Lambda.OpenTelemetry` package for .NET) | **None — use the SDK's standard setup** |
| Collector | ADOT Lambda layer, plus a custom `otel-collector-config.yaml` because the default does not route OTLP to the traces pipeline | No layer; export as you would from any container |
| Env var | `OPENTELEMETRY_COLLECTOR_CONFIG_URI` / `_FILE`, differing by SDK | n/a |
| Flush timing | Must flush before the invocation deadline, or telemetry is lost | No deadline to beat |
| IAM | Execution role needs X-Ray and CloudWatch permissions | Runner needs nothing for stdout/stderr logging |

Some SDKs add optional Cloud Run conveniences, such as OpenTelemetry helpers. **They are optional**, and where they exist they are documented in that SDK's Cloud Run guide. <!-- docs/encyclopedia/workers/serverless-workers/cloud-run.mdx:31-34 -->

## Logs

Cloud Run collects `stdout` and `stderr` into [Cloud Logging](https://cloud.google.com/run/docs/logging) through its own infrastructure. **The runner service account needs no grant for this** — `roles/logging.logWriter` is required only if the Worker writes through the Cloud Logging API instead. → `iam.md`.

Read a pool's logs:

```bash
gcloud run worker-pools logs read <POOL_NAME> --region <REGION> --project <YOUR_GCP_PROJECT>
```

**The pool produces no logs while scaled to zero.** Read them while an instance is up; an empty log is not evidence of a problem. This is the most common source of confusion when checking a Cloud Run Worker's health, and it has no Lambda equivalent — a Lambda log group retains history after the invocation ends.

## Memory: give the runtime the instance, not the host

A Worker Pool instance defaults to **512 MiB**; raise `--memory` when creating the pool if the Worker needs more. Two runtimes need to be told about the container limit explicitly, or they size themselves to a fraction of it:

| SDK | Setting | Why |
|---|---|---|
| Java | `-XX:MaxRAMPercentage=75` | The JVM reads the container limit but defaults max heap to 25% of it, leaving most of a small instance unused. |
| TypeScript | `NODE_OPTIONS=--max-old-space-size=<MB>`, ~80% of the instance limit | Node's default heap is unrelated to the container limit. |

Both are container-sizing concerns rather than Temporal ones, but they surface as Worker instability under load and are easy to miss. → `setup.md`.

## What to watch that is specific to this provider

Standard Worker metrics tell you about the Worker. Two Cloud Run-specific signals tell you about the *scaling*, and neither comes from the SDK:

- **`run.googleapis.com/manualInstanceCount`** on the pool — what the WCI has asked for. Persistently `0` while a backlog exists is the headline symptom of a scaling failure.
- **`serving.knative.dev/lastModifier`** on the pool — whether Temporal has ever successfully written to it. The cheapest proof that impersonation and the update permission both work.

Both are read with `gcloud run worker-pools describe … --format=yaml`. → `diagnostics.md`.
