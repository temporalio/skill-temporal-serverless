# Serverless Workers — Concepts

<!-- Sources: docs/encyclopedia/workers/serverless-workers.mdx, docs/evaluate/development-production-features/serverless-workers/index.mdx -->

## Release status

**AWS Lambda — Public Preview since July 30, 2026.** Open to all Temporal Cloud customers. There is no access request, no support ticket, and no manual toggle to enable: a customer selects "AWS Lambda (Public Preview)" as the compute provider in the UI and sets up their Worker Deployment directly. Never route a user to support to "get access" for Lambda.

**GCP Cloud Run — Pre-release.** Its APIs may change in backwards-incompatible ways, and **access is gated**: the customer creates a support ticket or contacts their account team. Unlike Lambda, routing a user to support *is* correct here.

Those are the two supported providers. Do not adapt either one's material to a third.

**Do not carry facts between them.** Anything about Worker lifetime, Activity duration bounds, timeouts, packaging, or tuning is provider-specific — see `<provider>/constraints.md`. This page names the provider whenever the action differs: Lambda is invoked; Cloud Run is resized.

Public Preview is not General Availability. APIs are still evolving and may be subject to backwards-incompatible changes between versions — pin SDK and CLI versions for anything long-lived, and read the installed package's real API surface rather than writing from memory.

## What is a Serverless Worker?

A Serverless Worker is a Temporal Worker whose compute lifecycle is controlled by Temporal instead of by an independently operated Worker fleet. <!-- docs/encyclopedia/workers/serverless-workers.mdx:43 -->
There is no always-on compute capacity to maintain: Temporal starts capacity when needed and can return it to zero when idle. The provider resource itself—a Lambda function or Cloud Run Worker Pool—continues to exist. <!-- docs/encyclopedia/workers/serverless-workers.mdx:43-45 -->

**"Starts" means different things per provider.** On AWS Lambda, Temporal invokes a function per unit of work and the Worker exits when that invocation ends. On GCP Cloud Run, Temporal resizes a pool of long-lived instances, each running an ordinary Worker that polls for its whole lifetime. Both scale to zero when idle; almost nothing else about their lifecycles is the same.

A Serverless Worker uses the same Temporal SDKs as a traditional long-lived Worker. It registers Workflows and Activities the same way. <!-- docs/encyclopedia/workers/serverless-workers.mdx:47-49 -->

What changes is the lifecycle, and only on Lambda does it change much: instead of polling continuously, the Worker is invoked on demand, starts, processes available Tasks, and shuts down — which is why Lambda needs a dedicated serverless Worker package (`aws-lambda/sdk-<language>.md`). **On Cloud Run the Worker code is unchanged from a long-lived Worker**; the only addition is Worker Versioning, and there is no Cloud Run Worker package at all.

Serverless Workers require Worker Versioning. Each Serverless Worker must be associated with a Worker Deployment Version that has a compute provider configured. <!-- docs/encyclopedia/workers/serverless-workers.mdx:51-52 -->

Each Workflow must have an `AutoUpgrade` or `Pinned` versioning behavior, set per-Workflow or as a Worker-level default. <!-- docs/encyclopedia/workers/serverless-workers.mdx:245 -->

## How Temporal controls Serverless Workers

With long-lived Workers, the Worker process starts, connects to Temporal, and polls a Task Queue for work. Temporal does not need to know anything about the Worker's infrastructure. <!-- docs/encyclopedia/workers/serverless-workers.mdx:59-60 -->

With Serverless Workers, Temporal starts the Worker. <!-- docs/encyclopedia/workers/serverless-workers.mdx:62 -->

### Worker Controller Instance (WCI)

The Worker Controller Instance (WCI) is a system Workflow that scales Serverless Workers based on Task Queue conditions. <!-- docs/encyclopedia/workers/serverless-workers.mdx:66 -->
One WCI Workflow runs per Worker Deployment Version that has a compute provider configured. The WCI runs in the same Namespace as your Worker Deployment. <!-- docs/encyclopedia/workers/serverless-workers.mdx:67-68 -->

The WCI responds to sync match failures and periodically reads Task Queue metrics. It turns those inputs into an action compatible with the provider: invoke a Lambda function, or update a Cloud Run Worker Pool's manual instance count. <!-- docs/encyclopedia/workers/serverless-workers.mdx:70-72 -->

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

### Shared Task routing

1. A Task is submitted, for example by `StartWorkflow` or `ScheduleActivity`. <!-- docs/encyclopedia/workers/serverless-workers.mdx:103 -->
2. Matching attempts to route it directly to an available Worker in a sync match. <!-- docs/encyclopedia/workers/serverless-workers.mdx:104-105 -->
3. If no Worker is available, Matching adds the Task to the backlog and signals the WCI for that Worker Deployment Version. <!-- docs/encyclopedia/workers/serverless-workers.mdx:107-108 -->
4. The WCI asks the version's scaling algorithm for an action and applies it through the configured provider.

After that point the providers diverge:

- **AWS Lambda:** the WCI invokes the function. Each invocation creates a Worker and client connection, processes available Tasks, and shuts down. Invocations do not share a connection or in-memory state.
- **GCP Cloud Run:** the WCI increases the Worker Pool's manual instance count. Cloud Run starts an instance whose ordinary Worker connects once and polls for the instance's lifetime. The WCI later lowers the count as demand drains.

## Autoscaling

The WCI automatically scales Serverless Workers from Task Queue signals and metrics. The resulting action depends on the provider. <!-- docs/encyclopedia/workers/serverless-workers.mdx:117-120 -->

### Sync match failure

When a Task is submitted, Matching attempts to route it directly to an available Worker. If no Worker is available, the sync match fails and Matching signals the WCI. Lambda's no-sync algorithm can invoke another function; Cloud Run's rate-based algorithm can immediately increase the planned pool size, subject to its cooldown and maximum. <!-- docs/encyclopedia/workers/serverless-workers.mdx:124-126 -->

Because the Matching Service pushes match failures to the WCI as they happen rather than the WCI polling on a timer, latency stays low and scaling is responsive. <!-- docs/encyclopedia/workers/serverless-workers.mdx:126-128 -->

### Task Queue backlog

The WCI periodically reads version-level Task Queue arrival rate, dispatch rate, and backlog. Cloud Run's rate-based algorithm uses these metrics to calculate a desired instance count and explicitly resizes the pool; the periodic path also scales the pool down, including to zero. Lambda Workers instead end with their invocations and do not use this worker-set sizing model. <!-- docs/encyclopedia/workers/serverless-workers.mdx:132-133 -->

## Scaling with long-lived Workers

- **AWS Lambda:** a Lambda Worker can share a Task Queue with a fixed long-lived fleet and act as spillover when sync matching finds no available poller. Do not dynamically scale the long-lived fleet as well; the two scaling systems cannot coordinate. <!-- docs/encyclopedia/workers/serverless-workers.mdx:137-146 -->
- **GCP Cloud Run:** use a separate Task Queue from any independently managed long-lived fleet. The rate-based WCI scaler reads the full version-level arrival, dispatch, and backlog metrics and cannot subtract work handled by the other fleet, so sharing provisions duplicate capacity. → `gcp-cloud-run/constraints.md`.

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

### Worker crash or instance termination

If a Worker crashes or its compute is terminated, the in-flight Task is not acknowledged. Temporal applies the configured timeout and retry policy, and another Worker can receive the retry. On Lambda that means another invocation; on Cloud Run it means another running or replacement pool instance. Activity Heartbeats preserve progress for long-running work that can be interrupted. <!-- docs/encyclopedia/workers/serverless-workers.mdx:210-215 -->

### Provider capacity limit

If the provider cannot add capacity—Lambda account concurrency, Cloud Run regional quotas, or the configured Cloud Run `max_count`—Tasks remain in the Task Queue backlog without data loss and processing slows until capacity becomes available. The provider-side symptom differs: Lambda invocations are throttled or rejected, while a Cloud Run pool stops growing or its update fails. <!-- docs/encyclopedia/workers/serverless-workers.mdx:219-223 -->

### Resource exhaustion across Activity slots

A Worker process may run multiple Activity slots, so a crash or resource exhaustion in one Activity can affect other Activities in that same process. On Lambda, split Workflow and Activity Workers into separate functions or use one Activity slot per invocation for execution-environment isolation. On Cloud Run, size instance resources and Worker concurrency together; one slot limits concurrency within an instance but does not turn the long-lived instance into a per-Activity environment. → `<provider>/constraints.md`.

## Constraints

<!-- docs/encyclopedia/workers/serverless-workers.mdx:240-245 -->

| Constraint | Detail |
|---|---|
| Activity duration | On a provider that invokes per unit of work, must complete within its invocation limit minus the shutdown deadline buffer — Lambda's ceiling is 15 minutes. A pool-based provider such as Cloud Run imposes no per-invocation ceiling. → `<provider>/constraints.md`. |
| Workflow duration | No compute-provider limit. Workflow state is durable in Temporal and does not depend on one Lambda invocation or Cloud Run instance remaining alive. |
| Worker code | Same Temporal SDK Worker code. On Lambda it runs through that SDK's serverless Worker package; on Cloud Run it is an ordinary long-lived Worker with no extra package. |
| Versioning | Worker Versioning is required. Each Workflow must have an `AutoUpgrade` or `Pinned` behavior, set per-Workflow or as a Worker-level default. |

## Worker Versioning with Serverless Workers

Serverless Workers require Worker Versioning, and the compute provider must target a **stable, immutable build** for each Worker Deployment Version. That means aligning two versioning systems: <!-- docs/encyclopedia/workers/serverless-workers.mdx:249-250 -->

- **Temporal Worker Deployment Versions** — identified by deployment name and Build ID. Each Workflow runs against a specific Worker Deployment Version (Pinned) or moves between them on routing changes (Auto-Upgrade). <!-- docs/encyclopedia/workers/serverless-workers.mdx:252-253 -->
- **The provider's own unit of immutability** — a published Lambda function version, pinned by a qualified ARN; or on Cloud Run a dedicated Worker Pool per Build ID, because the compute configuration names a pool and not a revision. Keep a one-to-one mapping between it and the Build ID.

**Pointing a Worker Deployment Version at a mutable target causes non-determinism errors for in-flight Workflows, including Pinned ones.** Pinned routes Workflows to a version; it cannot pin code that changed underneath that version. The failure is the same on both providers but is reached differently — on Lambda you have to choose it by registering an unqualified ARN, while on Cloud Run a plain redeploy into a live pool does it — so read the provider file for which action is the dangerous one. → `aws-lambda/versioning.md`, `gcp-cloud-run/versioning.md`.

Pinned or Auto-Upgrade controls how Workflows move between Worker Deployment Versions in Temporal. It does not change how a Worker Deployment Version targets the provider; both behaviors expect one immutable build per version. <!-- docs/encyclopedia/workers/serverless-workers.mdx:294-296 -->

## Compute providers

A compute provider is the configuration that tells Temporal how to control Serverless Worker capacity. It is set on a Worker Deployment Version and specifies the provider type, compute target, and credentials Temporal needs to invoke a function or resize a worker set. <!-- docs/encyclopedia/workers/serverless-workers.mdx:310-312 -->

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

- **Reduce operational overhead.** Temporal drives capacity from Task Queue demand: invoking Lambda or setting the Cloud Run Worker Pool size.
- **Avoid managing a continuously provisioned fleet.** Deploy a function or a Worker Pool without building a separate autoscaling control plane.
- **Scale automatically.** Capacity grows with demand and can return to zero when idle.
- **Pay only while compute runs.** Lambda bills invocations; Cloud Run bills running pool instances. Both can reduce idle cost for low or intermittent workloads.

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

| | Independently managed Worker | AWS Lambda Serverless Worker | GCP Cloud Run Serverless Worker |
|---|---|---|---|
| **Lifecycle** | Long-lived process; you decide when it runs. | Short-lived Worker created for each function invocation. | Ordinary long-lived Worker inside each pool instance; WCI controls the instance count. |
| **Scaling** | You manage replicas or instances. | WCI invokes functions after Task Queue signals. | WCI explicitly resizes the Worker Pool from signals and periodic queue metrics. |
| **Connection** | Persistent for the process lifetime. | Fresh connection per invocation. | Persistent for the pool instance lifetime. |
| **Scale to zero** | Only if you build and operate it. | Invocations end when work drains. | WCI lowers the pool's manual instance count to zero. |
