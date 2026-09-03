# Serverless Workers — Concepts

<!-- Sources: docs/encyclopedia/workers/serverless-workers.mdx, docs/evaluate/development-production-features/serverless-workers/index.mdx -->

## Release status

**AWS Lambda — Public Preview since July 30, 2026.** Open to all Temporal Cloud customers. There is no access request, no support ticket, and no manual toggle to enable: a customer selects "AWS Lambda (Public Preview)" as the compute provider in the UI and sets up their Worker Deployment directly. Never route a user to support to "get access" for Lambda.

**GCP Cloud Run — Pre-release.** Its APIs may change in backwards-incompatible ways, and **access is gated**: the customer creates a support ticket or contacts their account team. Unlike Lambda, routing a user to support *is* correct here.

Those are the two supported providers. Do not adapt either one's material to a third.

**Do not carry facts between them.** Anything about Worker lifetime, Activity duration bounds, timeouts, packaging, or tuning is provider-specific — see `<provider>/constraints.md`. Where this file says "invoke" or "invocation" below, read it as Lambda's model; the Cloud Run equivalent is the WCI raising a pool's instance count.

Public Preview is not General Availability. APIs are still evolving and may be subject to backwards-incompatible changes between versions — pin SDK and CLI versions for anything long-lived, and read the installed package's real API surface rather than writing from memory.

## What is a Serverless Worker?

A Serverless Worker is a Temporal Worker that runs on serverless compute instead of a long-lived process. <!-- docs/encyclopedia/workers/serverless-workers.mdx:43 -->
There is no always-on infrastructure to provision or scale. Temporal starts the Worker when Tasks arrive on a Task Queue, and the compute scales back to zero when the work is done. <!-- docs/encyclopedia/workers/serverless-workers.mdx:43-45 -->

**"Starts" means different things per provider.** On AWS Lambda, Temporal invokes a function per unit of work and the Worker exits when that invocation ends. On GCP Cloud Run, Temporal resizes a pool of long-lived instances, each running an ordinary Worker that polls for its whole lifetime. Both scale to zero when idle; almost nothing else about their lifecycles is the same.

A Serverless Worker uses the same Temporal SDKs as a traditional long-lived Worker. It registers Workflows and Activities the same way. <!-- docs/encyclopedia/workers/serverless-workers.mdx:47-49 -->

What changes is the lifecycle, and only on Lambda does it change much: instead of polling continuously, the Worker is invoked on demand, starts, processes available Tasks, and shuts down — which is why Lambda needs a dedicated serverless Worker package (`aws-lambda/sdk-<language>.md`). **On Cloud Run the Worker code is unchanged from a long-lived Worker**; the only addition is Worker Versioning, and there is no Cloud Run Worker package at all.

Serverless Workers require Worker Versioning. Each Serverless Worker must be associated with a Worker Deployment Version that has a compute provider configured. <!-- docs/encyclopedia/workers/serverless-workers.mdx:51-52 -->

Each Workflow must have an `AutoUpgrade` or `Pinned` versioning behavior, set per-Workflow or as a Worker-level default. <!-- docs/encyclopedia/workers/serverless-workers.mdx:245 -->

## How Serverless invocation works

With long-lived Workers, the Worker process starts, connects to Temporal, and polls a Task Queue for work. Temporal does not need to know anything about the Worker's infrastructure. <!-- docs/encyclopedia/workers/serverless-workers.mdx:59-60 -->

With Serverless Workers, Temporal starts the Worker. <!-- docs/encyclopedia/workers/serverless-workers.mdx:62 -->

### Worker Controller Instance (WCI)

The Worker Controller Instance (WCI) is a system Workflow that scales Serverless Workers based on Task Queue conditions. <!-- docs/encyclopedia/workers/serverless-workers.mdx:66 -->
One WCI Workflow runs per Worker Deployment Version that has a compute provider configured. The WCI runs in the same Namespace as your Worker Deployment. <!-- docs/encyclopedia/workers/serverless-workers.mdx:67-68 -->

The WCI responds to two triggers: sync match failures and Task Queue backlog. When either trigger fires, the WCI produces a scaling action, such as invoking the configured compute provider (for example, calling AWS Lambda's `InvokeFunction` API) to start new Workers. <!-- docs/encyclopedia/workers/serverless-workers.mdx:70-72 -->

You can list WCI Workflows in your Namespace: <!-- docs/encyclopedia/workers/serverless-workers.mdx:75 -->

```bash
temporal workflow list \
  --namespace <NAMESPACE> \
  --query 'TemporalNamespaceDivision = "TemporalWorkerControllerInstance"'
```
<!-- docs/encyclopedia/workers/serverless-workers.mdx:77-81 -->

WCI Workflow IDs follow the pattern `temporal-sys-worker-controller-instance:<deployment-name>:<build-id>`. <!-- docs/encyclopedia/workers/serverless-workers.mdx:83 -->

You can inspect a WCI Workflow's history to see its recent Activity results: <!-- docs/encyclopedia/workers/serverless-workers.mdx:83-84 -->

```bash
temporal workflow show \
  --namespace <NAMESPACE> \
  --workflow-id 'temporal-sys-worker-controller-instance:<DEPLOYMENT_NAME>:<BUILD_ID>'
```
<!-- docs/encyclopedia/workers/serverless-workers.mdx:86-90 -->

### Invocation flow

The invocation flow works as follows: <!-- docs/encyclopedia/workers/serverless-workers.mdx:101 -->

1. A Task is submitted (for example, `StartWorkflow` or `ScheduleActivity`). <!-- docs/encyclopedia/workers/serverless-workers.mdx:103 -->
2. The Matching Service attempts to route the Task directly to an available Worker (a sync match). <!-- docs/encyclopedia/workers/serverless-workers.mdx:104-105 -->
3. If a Worker is available, the Task is routed to that Worker. <!-- docs/encyclopedia/workers/serverless-workers.mdx:106 -->
4. If no Worker is available (sync match fails), the Matching Service pushes a signal to the WCI, and the WCI invokes the configured compute provider. <!-- docs/encyclopedia/workers/serverless-workers.mdx:107-108 -->
5. The Serverless Worker starts, creates a Temporal Client, and begins polling the Task Queue. <!-- docs/encyclopedia/workers/serverless-workers.mdx:109 -->
6. The Worker processes available Tasks until it exits (see Worker lifecycle). <!-- docs/encyclopedia/workers/serverless-workers.mdx:110 -->

Each invocation is independent. The Worker creates a fresh client connection on every invocation. There is no connection reuse or shared state across invocations. <!-- docs/encyclopedia/workers/serverless-workers.mdx:112-113 -->

## Autoscaling

The WCI automatically scales Serverless Workers based on Task Queue signals. When Tasks arrive and no Worker is available, the WCI invokes new Workers. When the Tasks are done, Workers exit and scale to zero. <!-- docs/encyclopedia/workers/serverless-workers.mdx:117-118 -->

The WCI uses two signals to decide when to invoke new Workers: <!-- docs/encyclopedia/workers/serverless-workers.mdx:120 -->

### Sync match failure

When a Task is submitted, the Matching Service attempts to route it directly to an available Worker. If no Worker is available, the sync match fails, and the Matching Service pushes a signal to the WCI. The WCI then invokes a new Worker. This is the primary scaling path. <!-- docs/encyclopedia/workers/serverless-workers.mdx:124-126 -->

Because the Matching Service pushes match failures to the WCI as they happen rather than the WCI polling on a timer, latency stays low and scaling is responsive. <!-- docs/encyclopedia/workers/serverless-workers.mdx:126-128 -->

### Task Queue backlog

The WCI monitors Task Queue metadata to determine whether pending Tasks exist without enough Workers to process them. If there are Tasks on the queue and not enough Workers, the WCI invokes additional Workers. <!-- docs/encyclopedia/workers/serverless-workers.mdx:132-133 -->

## Scaling with long-lived Workers

Serverless Workers can share a Task Queue with long-lived Workers. Because Serverless Workers are only invoked on sync match failure, Serverless Workers only pick up Tasks that no long-lived Worker was available to handle. In practice, the Serverless Workers act as spillover capacity for the long-lived fleet. <!-- docs/encyclopedia/workers/serverless-workers.mdx:137-139 -->

**Warning:** If you configure Serverless and long-lived Workers on the same Task Queue, do not enable dynamic scaling on the long-lived Workers. The two groups cannot coordinate their scaling behavior. If both scale dynamically, the long-lived Workers may scale up to handle the same Tasks that Temporal is simultaneously invoking Serverless Workers for, leading to unnecessary invocations and unpredictable scaling. <!-- docs/encyclopedia/workers/serverless-workers.mdx:143-146 -->

## Worker lifecycle

**This section describes providers that invoke per unit of work, such as AWS Lambda.** On a provider that scales a pool of long-lived instances, such as GCP Cloud Run, an instance connects once and polls for its whole lifetime: there are no per-invocation phases, and none of the tuning below applies. → `<provider>/constraints.md`.

A single Serverless Worker invocation has three phases: init, work, and shutdown. <!-- docs/encyclopedia/workers/serverless-workers.mdx:152 -->

### Init phase

The Worker initializes and establishes a client connection to Temporal. <!-- docs/encyclopedia/workers/serverless-workers.mdx:161 -->

### Work phase

The Worker polls the Task Queue and processes Tasks. <!-- docs/encyclopedia/workers/serverless-workers.mdx:163 -->

### Shutdown phase

The Worker stops polling, waits for in-flight Tasks to finish, and runs any shutdown hooks (for example, OpenTelemetry telemetry flushes). Shutdown begins before the invocation deadline so the Worker can exit cleanly before the compute provider forcibly terminates the execution environment. <!-- docs/encyclopedia/workers/serverless-workers.mdx:165-167 -->

### Tuning for long-running Activities

Three values must be tuned together — worker stop timeout, shutdown deadline buffer, and invocation deadline — and raising one alone does not help. The exact relationships, a worked example, the failure symptom, and the Activity Heartbeat threshold are provider-specific. → `<provider>/constraints.md`.

## Failure handling

Serverless Workers rely on Temporal's standard retry and timeout semantics to recover from failures. <!-- docs/encyclopedia/workers/serverless-workers.mdx:205-206 -->

### Worker crash

If a Worker invocation crashes (out of memory, unhandled exception, etc.): <!-- docs/encyclopedia/workers/serverless-workers.mdx:210-211 -->

- The Activity Timeout fires after the configured duration. <!-- docs/encyclopedia/workers/serverless-workers.mdx:213 -->
- Temporal retries the Activity on a different Worker invocation. <!-- docs/encyclopedia/workers/serverless-workers.mdx:214 -->
- No manual intervention is required. <!-- docs/encyclopedia/workers/serverless-workers.mdx:215 -->

### Provider concurrency limit

If the compute provider's concurrency limit is reached (for example, AWS Lambda account concurrency): <!-- docs/encyclopedia/workers/serverless-workers.mdx:219 -->

- Further invocations from the WCI fail. <!-- docs/encyclopedia/workers/serverless-workers.mdx:221 -->
- Tasks remain in the Task Queue backlog. No data loss occurs. <!-- docs/encyclopedia/workers/serverless-workers.mdx:222 -->
- Processing slows until concurrency frees up. <!-- docs/encyclopedia/workers/serverless-workers.mdx:223 -->

### Resource exhaustion across Activity slots

By default, a single Worker invocation may run multiple Activity slots. A crash or resource exhaustion in one Activity can affect other Activities running in the same invocation. <!-- docs/encyclopedia/workers/serverless-workers.mdx:227-229 -->

To isolate Activities from each other: <!-- docs/encyclopedia/workers/serverless-workers.mdx:231 -->

- Split Workflow and Activity Workers into separate compute functions. <!-- docs/encyclopedia/workers/serverless-workers.mdx:233 -->
- Set Activity slots to 1 per invocation. <!-- docs/encyclopedia/workers/serverless-workers.mdx:234 -->

With single-slot configuration, each Activity gets a dedicated execution environment. <!-- docs/encyclopedia/workers/serverless-workers.mdx:236 -->

## Constraints

<!-- docs/encyclopedia/workers/serverless-workers.mdx:240-245 -->

| Constraint | Detail |
|---|---|
| Activity duration | On a provider that invokes per unit of work, must complete within its invocation limit minus the shutdown deadline buffer — Lambda's ceiling is 15 minutes. A pool-based provider such as Cloud Run imposes no per-invocation ceiling. → `<provider>/constraints.md`. |
| Workflow duration | No limit. Workflows of any duration work, regardless of the invocation timeout. A Workflow runs across as many invocations as needed. |
| Worker code | Same Temporal SDK Worker code. On Lambda it runs through that SDK's serverless Worker package; on Cloud Run it is an ordinary long-lived Worker with no extra package. |
| Versioning | Worker Versioning is required. Each Workflow must have an `AutoUpgrade` or `Pinned` behavior, set per-Workflow or as a Worker-level default. |

## Worker Versioning with Serverless Workers

Serverless Workers require Worker Versioning, and the compute provider must invoke a **stable, immutable build** for each Worker Deployment Version. That means aligning two versioning systems: <!-- docs/encyclopedia/workers/serverless-workers.mdx:249-250 -->

- **Temporal Worker Deployment Versions** — identified by deployment name and Build ID. Each Workflow runs against a specific Worker Deployment Version (Pinned) or moves between them on routing changes (Auto-Upgrade). <!-- docs/encyclopedia/workers/serverless-workers.mdx:252-253 -->
- **The provider's own unit of immutability** — a published Lambda function version, pinned by a qualified ARN; or on Cloud Run a dedicated Worker Pool per Build ID, because the compute configuration names a pool and not a revision. Keep a one-to-one mapping between it and the Build ID.

**Pointing a Worker Deployment Version at a mutable target causes non-determinism errors for in-flight Workflows, including Pinned ones.** Pinned routes Workflows to a version; it cannot pin code that changed underneath that version. The failure is the same on both providers but is reached differently — on Lambda you have to choose it by registering an unqualified ARN, while on Cloud Run a plain redeploy into a live pool does it — so read the provider file for which action is the dangerous one. → `aws-lambda/versioning.md`, `gcp-cloud-run/versioning.md`.

Pinned or Auto-Upgrade controls how Workflows move between Worker Deployment Versions in Temporal. It does not change how a Worker Deployment Version targets the provider; both behaviors expect one immutable build per version. <!-- docs/encyclopedia/workers/serverless-workers.mdx:294-296 -->

## Compute providers

A compute provider is the configuration that tells Temporal how to invoke a Serverless Worker. The compute provider is set on a Worker Deployment Version and specifies the provider type, the invocation target, and the credentials Temporal needs to trigger the invocation. <!-- docs/encyclopedia/workers/serverless-workers.mdx:310-312 -->

For example, an AWS Lambda compute provider includes the Lambda function ARN and the IAM role that Temporal assumes to invoke the function; a Cloud Run compute provider names the project, region, and Worker Pool, plus the service account Temporal impersonates to scale it. <!-- docs/encyclopedia/workers/serverless-workers.mdx:314-315 -->

Compute providers are only needed for Serverless Workers. Traditional long-lived Workers do not require a compute provider because the Worker process lifecycle is not managed by the Temporal server. <!-- docs/encyclopedia/workers/serverless-workers.mdx:317-318 -->

### Supported providers

<!-- docs/encyclopedia/workers/serverless-workers.mdx:322-324 -->

| Provider | Description |
|---|---|
| AWS Lambda | Temporal assumes an IAM role in your AWS account to invoke a Lambda function. |
| GCP Cloud Run | Temporal impersonates a service account in your Google Cloud project to scale a Worker Pool. |

## Why use Serverless Workers?

<!-- docs/evaluate/development-production-features/serverless-workers/index.mdx:34-71 -->

- **Reduce operational overhead.** No always-on infrastructure to manage and no autoscaling policies to tune. Temporal and the compute provider handle invocation and scaling.
- **Get started faster.** Deploying a Worker is as simple as deploying a function. No Kubernetes, container orchestration, or scaling strategy required.
- **Scale automatically.** The compute provider handles scaling natively. When traffic drops, instances scale down. When there is no work, there is no compute running.
- **Pay only for what you use.** Workers run only when Tasks are available. For low or intermittent volume workloads, this pay-per-invocation model can significantly reduce compute costs.

## When to use Serverless Workers

<!-- docs/evaluate/development-production-features/serverless-workers/index.mdx:75-86 -->

Good fit when:

- Workloads are bursty or event-driven (order processing, notifications, webhook handlers).
- Traffic is low or intermittent.
- You want a simpler getting-started path.
- Your organization has standardized on serverless.
- You serve multiple tenants with infrequent workloads.

May not be ideal when:

<!-- docs/evaluate/development-production-features/serverless-workers/index.mdx:88-97 -->

- Activities are long-running and cannot be interrupted, on a provider with a per-invocation ceiling — Lambda's is 15 minutes. Activities that run longer and cannot be broken into smaller steps need a different hosting strategy, or a provider without that ceiling.
- Workloads require sustained high throughput. Long-lived Workers on dedicated compute may be more cost-effective and performant.
- You need persistent connections and the provider invokes per unit of work. Some features require a persistent connection between the Worker and Temporal, which per-invocation Workers do not maintain; a pool-based provider holds one for the instance's lifetime.

## How Serverless Workers compare to long-lived Workers

<!-- docs/evaluate/development-production-features/serverless-workers/index.mdx:99-105 -->

|                | Long-lived Worker | Serverless Worker |
|---|---|---|
| **Lifecycle** | Long-lived process that runs continuously. | Invoked on demand. Starts and stops per invocation. |
| **Scaling** | You manage scaling (Kubernetes HPA, instance count, etc.). | Temporal invokes additional instances as needed, within the compute provider's concurrency limits. |
| **Connection** | Persistent connection to Temporal. | Fresh connection on each invocation. |
