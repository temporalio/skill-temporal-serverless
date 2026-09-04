# GCP Cloud Run — execution-model constraints

<!-- Sources:
  docs/encyclopedia/workers/serverless-workers/cloud-run.mdx
  docs/production-deployment/worker-deployments/serverless-workers/cloud-run/index.mdx
-->

Consequences of Cloud Run's execution model: **Temporal resizes a pool of long-lived instances, scaling it to zero when there is no work.** Temporal does *not* invoke your Worker per Task.

For a deliberate cross-provider comparison, see `../aws-lambda/constraints.md`.

## Worker lifetime is an instance, not an invocation

Each pool instance runs **standard long-lived Worker code**: it connects, registers Workflows and Activities, and polls the Task Queue for its whole lifetime. There is no handler, no per-Task lifecycle, and **no serverless Worker package**. Use the Cloud Run SDK reference in this directory; some SDKs add optional conveniences, but none are required. <!-- docs/encyclopedia/workers/serverless-workers/cloud-run.mdx:27-34 -->

The WCI controls how many instances run; each instance manages its own polling and Task processing.

## Timing and Activity limits

- **No invocation deadline.** Nothing bounds how long a Worker lives except scale-in.
- **No shutdown deadline buffer.** Cloud Run terminates instances through its own scale-in lifecycle.
- **No timeout triple to tune.** Two of its three values do not exist here.
- **Activities are not bounded by an invocation limit.** Their practical interruption boundary is scale-in.
- **Eager Activities are not disabled by the platform.** A pool instance holds its connection for its lifetime; confirm support against the selected SDK. <!-- inferred: the per-invocation rationale does not apply; not stated for Cloud Run in the docs read -->

Cloud Run sends `SIGTERM` during scale-in and can send `SIGKILL` ten seconds later. Handle `SIGTERM` and configure the SDK's graceful-shutdown timeout below that window. → `sdk-<language>.md`.

## What bounds an Activity instead: scale-in

**The WCI decides when to remove an instance from Task Queue activity, not from what any individual instance is doing.** It does not track how long an instance has been running or whether it is mid-Activity, so the instance Cloud Run stops may be one that is still executing work. <!-- docs/encyclopedia/workers/serverless-workers/cloud-run.mdx:112-116 -->

Graceful shutdown lets short work drain but cannot guarantee an Activity will finish. **Use Activity Heartbeats** so interrupted work resumes from its last recorded progress instead of restarting.

*Symptom of ignoring this:* Activities failing partway and retrying from the beginning, correlated with the pool shrinking.

## Autoscaling behavior

<!-- docs/encyclopedia/workers/serverless-workers/cloud-run.mdx:36-69 -->

The WCI combines two mechanisms:

- **Immediate** — bring up instances when a Task arrives and no Worker is free to take it (a sync match failure). Absorbs bursts without waiting for the evaluation cycle.
- **Periodic** — rate-based re-sizing. The WCI measures how fast Tasks arrive and how fast one Worker processes them, computes the instance count needed, and applies it through the Cloud Run admin API.

It sizes to a **target utilization of 80% by default** rather than loading every Worker fully, so there is headroom to take new Tasks immediately, and adds instances on top when a backlog exists.

**Scale-in is deliberately more conservative than scale-out:** it holds capacity while sync match failures are still occurring and applies a cooldown before reducing the pool. With no work, it can scale to zero; the next sync match failure or backlog scales it back up.

The scaler defaults are **minimum `0`, maximum `30`, initial count `0`, and target utilization `0.8`**. Configure them in the version's Scaling and Lifecycle settings or with the Temporal CLI. The four CLI flags are coupled: omit all four to use the defaults, or provide `--gcp-cloud-run-min-instances`, `--gcp-cloud-run-max-instances`, `--gcp-cloud-run-initial-instances`, and `--gcp-cloud-run-utilization-target` together. A partial group is rejected. <!-- docs/troubleshooting/serverless-workers/cloud-run.mdx:130-138 -->

An initial count and minimum of zero do not suppress registration. The rate-based algorithm temporarily requests at least one instance when the version is registered so its Task Queues can bind, then normal scaling can return the pool to zero. A pool that later stops growing under backlog is either at its configured maximum or at a regional Cloud Run quota.

## One Worker Pool per Worker Deployment Version

**The compute configuration names a project, region, and pool — not a revision.** Temporal runs whichever revision the pool serves at the time, which ties a pool to a single build. A new build needs a **new pool**, and the Build ID belongs in the pool name to keep that mapping visible. <!-- docs/encyclopedia/workers/serverless-workers/cloud-run.mdx:82-89 -->

Keep an older version's pool in place while Pinned Workflows are still running on it. It can sit at zero instances; its WCI scales it back up when a Task arrives for that version.

**The mutable-build hazard:** deploying a new image into a pool that a live Worker Deployment Version points at creates a new revision, and Cloud Run promotes it to every instance by default. The version does not change, but the code behind it does — causing non-determinism errors for in-flight Workflows, **including Pinned ones**. → `versioning.md`.

## Do not share a Task Queue with long-lived Workers

**The pool scales up to cover the Task Queue's full workload even when independently managed Workers are already handling all of it**, so you run and pay for duplicate capacity. <!-- docs/encyclopedia/workers/serverless-workers/cloud-run.mdx:71-80 -->

The WCI sizes the pool from the rate of Tasks arriving on the version's Task Queues, and nothing in that measurement accounts for the long-lived Workers. Sync matching to a long-lived Worker suppresses the *immediate* scale-up, but the periodic re-sizing scales the pool up regardless. Fixing poller counts on the long-lived side does not help — use separate Task Queues.

## Pre-release status

Cloud Run support is **Pre-release, and its APIs may change in backwards-incompatible ways.** Access is not open: the user must create a support ticket or contact their account team. Confirm access exists before planning a deployment — this is a gate no amount of correct configuration gets past. <!-- docs/encyclopedia/workers/serverless-workers/cloud-run.mdx:19-23 -->

## Provider-side issue worth knowing

Google currently reports **high deployment latency creating or updating Cloud Run resources in some regions, including `us-central1`**, and recommends deploying elsewhere while it is open. If pool operations are slow, check [Cloud Run known issues](https://cloud.google.com/run/docs/known-issues) before assuming misconfiguration. <!-- docs/production-deployment/worker-deployments/serverless-workers/cloud-run/index.mdx:53-63 -->

## What does *not* follow from this model

- **Use ordinary Worker code.** Handler, configure-callback, and provider-package patterns do not apply.
- **Scale-to-zero is not per-Task.** An idle pool costs nothing, but a running instance is billed for its lifetime, not per unit of work.
- **A passing Validate Connection exercises only the read permission and starts no instance.** → `diagnostics.md`.
