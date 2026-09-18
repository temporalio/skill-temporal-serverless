# GCP Cloud Run — Diagnostics & troubleshooting

## Scaling flow (when working correctly)

<!-- docs/troubleshooting/serverless-workers/cloud-run.mdx:29-40 -->

1. You deploy the Worker image to a Worker Pool at zero instances.
2. You create a Worker Deployment Version pointing at that pool. This starts a WCI Workflow.
3. An instance starts, the Worker polls, and the server **binds the Task Queue** to the version.
4. The WCI monitors that Task Queue; the Matching Service also signals it when a Task arrives with no free Worker.
5. As work arrives the WCI raises the instance count through the Cloud Run admin API; as it drains, lowers it, possibly to zero.

**Temporal does not invoke your Worker per Task** — it changes how many instances run. Diagnose accordingly: "was it invoked?" is the wrong first question here.

## Start here: read the pool's annotations

Start by describing the pool:

```bash
gcloud run worker-pools describe <POOL_NAME> \
  --region <REGION> --project <YOUR_GCP_PROJECT> --format=yaml
```

Three fields under `metadata.annotations`: <!-- docs/troubleshooting/serverless-workers/cloud-run.mdx:57-63 -->

| Field | What it tells you |
|---|---|
| `run.googleapis.com/manualInstanceCount` | The instance count currently requested. `0` means no Worker is running. |
| `run.googleapis.com/scalingMode` | Should be `manual` — the WCI scales by writing the manual instance count. |
| `serving.knative.dev/lastModifier` | Who most recently changed the pool. The invoker service account means Temporal made the latest change; a later manual update replaces it with the operator's identity. |

Treat `lastModifier` as a current clue, not historical proof:

- The invoker service account → **Temporal successfully updated the pool after its last manual change.** If instances are running, skip to "instances running but Tasks not completing."
- The account you deployed with → the latest change was manual. To determine whether Temporal has ever updated the pool, inspect the registration Activity in the WCI history and the Task Queue binding below before diagnosing permissions.

## The Worker Pool is not scaling up

### 1. Validate Connection — and know what it does *not* prove

Workers → Deployments → select deployment → Actions → **Validate Connection**. For Cloud Run this impersonates the invoker and reads the pool, confirming three things: the compute configuration names a pool that exists, Temporal can impersonate the invoker, and the invoker can read.

**It starts no instance and does not exercise `run.workerPools.update`.** Version registration is different: its Task Queue bootstrap does update the pool. An invoker with read but not update permission can therefore pass this manual validation, but version registration or a later resize fails. If validation succeeds but registration never changes `lastModifier`, **check the update permission**. <!-- docs/troubleshooting/serverless-workers/cloud-run.mdx:75-99 -->

On failure, check each part of the compute configuration against the pool:

- **Project, region, pool name.** Temporal addresses the pool as `projects/<PROJECT>/locations/<REGION>/workerPools/<POOL_NAME>`. **A wrong region reports the pool as not found, identical to a wrong name** — so a "not found" does not tell you which field is wrong.
- **Impersonation.** Temporal's identity needs `roles/iam.serviceAccountTokenCreator` on the invoker. The Terraform module grants this on Cloud; self-hosted grants it to the server's GCP identity.
- **Invoker permissions.** `run.workerPools.get` to read, `run.workerPools.update` to scale.

→ `iam.md`.

### 2. Did the registration bootstrap bind the Task Queue?

```bash
temporal worker deployment describe-version \
  --namespace <NS> --deployment-name <NAME> --build-id <BUILD_ID> --report-task-queue-stats
```

The server creates the binding when a Worker running that version connects and polls. **No Task Queues listed means no Worker has polled successfully under this version.**

Registration performs this bootstrap: the WCI reads the pool, updates its manual instance count to at least one, and Cloud Run starts an instance. An absent binding means that sequence failed or the instance started but did not connect under the expected deployment name and build ID. Inspect the WCI's `ValidateSpec`/registration Activity failure, the pool's `lastModifier` and instance count, and then the pool logs. Do not wait for a first Workflow to repair registration.

### 3. Is the version current?

The registration bootstrap does not make the version current. New traffic routes only after the version is current, and a CLI-created version is not current automatically. Verify with `temporal worker deployment describe`.

**A `set-current-version` run without `--yes` may have done nothing** — it prompts, and non-interactively exits without applying the change, which reads as success.

### 4. Is the pool at its ceiling?

If instances are running but the count stops growing while backlog builds:

- **The maximum defaults to 30.** Raise it in the version's Scaling and Lifecycle settings, or update the existing version from the CLI:

  ```bash
  temporal worker deployment update-version-compute-config \
    --namespace <NS> \
    --deployment-name <NAME> \
    --build-id <BUILD_ID> \
    --gcp-cloud-run-min-instances 0 \
    --gcp-cloud-run-max-instances <NEW_MAX> \
    --gcp-cloud-run-initial-instances 0 \
    --gcp-cloud-run-utilization-target 0.8 \
    --gcp-cloud-run-scale-down-stabilization-duration 90s
  ```

  The five scaler flags must be supplied together, even when changing only the maximum. Choose values appropriate for the version rather than blindly copying this default-shaped example; the initial count must be between the minimum and maximum. The scale-down stabilization flag requires Temporal CLI v1.8.3 or later; with an older CLI, omit all five and accept the default scaling settings rather than supplying a partial group.
- If the count stalls *below the configured maximum*, check the project's [Cloud Run quotas](https://cloud.google.com/run/quotas) for that region. Cloud Run caps instances and CPU per region regardless of what the WCI requests.

## Instances are running but Tasks are not completing

### Read the pool logs

```bash
gcloud run worker-pools logs read <POOL_NAME> --region <REGION> --project <YOUR_GCP_PROJECT>
```

**A scaled-to-zero pool emits no new logs.** `logs read` still returns historical entries; use `logs tail` only while an instance is running. An empty result may simply mean the pool has never started.

Common errors: <!-- docs/troubleshooting/serverless-workers/cloud-run.mdx:156-166 -->

- **Connection failures** — check `TEMPORAL_ADDRESS` and `TEMPORAL_NAMESPACE` on the pool. Self-hosted: verify network reachability from Cloud Run to the frontend.
- **Missing secrets** — the instance cannot read the API key or TLS material. The **runner** service account needs `roles/secretmanager.secretAccessor` on the secret. That is the account in `spec.template.spec.serviceAccountName`, **not the invoker.**
- **Authentication errors** — key invalid, expired, or without access to the Namespace.
- **`TransportError: … NativeCertsNotFound`** — a Rust-core SDK in a minimal base image with no CA certificates. Install `ca-certificates` in the image. → `setup.md`.

### Deployment name and build ID

Instances start and poll but no Task is ever processed → the name or build ID in the code does not match the version. The Worker polls under a version the WCI does not manage, **so its polls never satisfy the Tasks the WCI is scaling for.**

This appears as a running pool with healthy-looking logs but no Workflow progress.

## Activities interrupted mid-execution

Activities failing partway and retrying from the beginning, correlated with the pool shrinking, means **scale-in is stopping instances that are still working.** The WCI does not track whether the instance Cloud Run stops is mid-Activity.

This is expected behavior, not a misconfiguration. Confirm the Worker handles `SIGTERM` and has a non-zero graceful-shutdown timeout below Cloud Run's ten-second termination window; this lets short work drain. Long-running work still needs **Activity Heartbeats** so a retry resumes from its last recorded progress. → `constraints.md`.

## Rule out a GCP-side cause

If every check passes, the cause may be in Cloud Run rather than your configuration: <!-- docs/troubleshooting/serverless-workers/cloud-run.mdx:191-203 -->

- [Cloud Run known issues](https://cloud.google.com/run/docs/known-issues) — includes issues affecting how long pool operations take.
- [Google Cloud Service Health](https://status.cloud.google.com/) — active incidents by product and region.

If a provider issue is confirmed, wait for recovery or move to another region. **Moving region means creating a new pool and updating the compute configuration**, since Temporal addresses a pool by project, region, and name.

## Never create or manage the WCI

Temporal creates one WCI per Worker Deployment Version with a compute provider, and a running WCI is not evidence that scaling works—it continues-as-new while its Activities fail. Read its history for Activity failures, and use the pool's `lastModifier` annotation to determine who made the latest update. Do not enumerate Cloud Run resources across regions to reverse-engineer state.
