# GCP Cloud Run — observability

## Use standard Worker telemetry

**A Cloud Run Serverless Worker emits the same traces and metrics as a Worker anywhere else.** It is an ordinary long-lived Worker, so the SDK's normal metrics and OpenTelemetry tracing setup applies unchanged, and each SDK's general observability guide is the right reference. <!-- docs/develop/go/workers/serverless-workers/cloud-run.mdx:115-118; docs/develop/python/workers/serverless-workers/cloud-run.mdx:132-135; docs/develop/typescript/workers/serverless-workers/cloud-run.mdx:139-142; docs/develop/java/workers/serverless-workers/cloud-run.mdx:152-155; docs/develop/dotnet/workers/serverless-workers/cloud-run.mdx:138-141 -->

Do not add provider-specific helper layers, collector environment variables, or invocation-deadline flush logic. Export telemetry as you would from any long-lived container.

Some SDKs add optional Cloud Run conveniences, such as OpenTelemetry helpers. **They are optional**, and where they exist they are documented in that SDK's Cloud Run guide. <!-- docs/encyclopedia/workers/serverless-workers/cloud-run.mdx:31-34 -->

## Logs

Cloud Run collects `stdout` and `stderr` into [Cloud Logging](https://cloud.google.com/run/docs/logging) through its own infrastructure. **The runner service account needs no grant for this** — `roles/logging.logWriter` is required only if the Worker writes through the Cloud Logging API instead. → `iam.md`.

Read a pool's logs:

```bash
gcloud run worker-pools logs read <POOL_NAME> --region <REGION> --project <YOUR_GCP_PROJECT>
```

**A scaled-to-zero pool emits no new logs.** `logs read` returns historical entries; `logs tail` shows new output only while an instance is running.

## What to watch that is specific to this provider

Standard Worker metrics tell you about the Worker. Two Cloud Run-specific signals tell you about the *scaling*, and neither comes from the SDK:

- **`run.googleapis.com/manualInstanceCount`** on the pool — what the WCI has asked for. A value that remains `0` while a backlog exists indicates a scaling failure.
- **`serving.knative.dev/lastModifier`** on the pool — who most recently updated it. The invoker service account means the WCI made the latest update; another identity does not prove that the WCI has never updated the pool.

Both are read with `gcloud run worker-pools describe … --format=yaml`. → `diagnostics.md`.
