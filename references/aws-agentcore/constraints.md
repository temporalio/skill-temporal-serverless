# AgentCore — Execution-model constraints

<!-- Sources:
  docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx
  docs/encyclopedia/workers/serverless-workers/index.mdx
  docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx
  docs/production-deployment/worker-deployments/serverless-workers/agentcore-self-hosted-setup.mdx
  docs/guides/durable-agent-on-agentcore.mdx
  aws: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-lifecycle-settings.html
  samples-python@d67795f: bedrock_agentcore/strands_agent/
-->

These are the consequences of AgentCore Runtime's execution model. **The WCI invokes a named AgentCore Runtime endpoint when it needs capacity. The Runtime starts a standard long-running Worker inside a Runtime session, and that Worker polls until it drains or AgentCore ends its compute.** <!-- docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx:34-37,135-136 -->

## Pre-release status

AgentCore Runtime support is in **Pre-release**, and its APIs may change in backwards-incompatible ways. A Temporal Cloud Namespace needs Pre-release access, which the user requests through a [support ticket](https://docs.temporal.io/evaluate/cloud/support#support-ticket) or their account team. A self-hosted Temporal Service v1.32.0 or later does not need an access request. See `self-hosted.md`. <!-- docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx:19-24; docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:41-43 -->

The Cloud Namespace must be **hosted on AWS**. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:41 -->

## Worker code is a standard Worker inside a Runtime handler

AgentCore Workers are standard long-lived Temporal Workers. There is no serverless Worker package. <!-- docs/encyclopedia/workers/serverless-workers/index.mdx:215 --> The Runtime entry point must implement the AgentCore Runtime HTTP contract by serving `/invocations` and `/ping`. For a Serverless Worker, `/invocations` starts the Worker as background work and acknowledges the request without waiting for the Worker to stop. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:61-65,157-164 --> The per-language entry point is in the selected `sdk-<language>.md` reference.

## Compute lifetime

- **Up to 8 hours per compute.** With the serverless microVM compute type, a Runtime session can run for up to 8 hours. <!-- docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx:73 --> The `maxLifetime` setting accepts 60–28800 seconds for microVM runtimes and defaults to 28800 seconds. Its timer starts when the microVM is created and cannot be reset. <!-- aws:runtime-lifecycle-settings.html#configuration-attributes, #lifecycle-and-session-relationship -->
- **No fixed invocation deadline.** An AgentCore Worker has no invocation deadline to monitor. It polls until it drains or AgentCore ends its compute. <!-- docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx:135-136 -->
- **The session is not durable storage.** AgentCore can resume a session on new compute after the previous compute ends, and a later Task can run on another Worker. Keep the state a Workflow needs in the Workflow or another durable store. <!-- docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx:131-133 -->
- **Process-local reuse is an optimization only.** The Worker can reuse initialization work, in-memory caches and temporary files while its Runtime compute remains available. Do not make Workflow progress depend on them. <!-- docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx:57,74 -->
- **Workflow duration is unbounded.** A Workflow continues across as many Worker processes as needed. Durable Timers and waits for approvals or external events do not keep compute running. <!-- docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx:64-67 -->

## Two independent stop controls

Two sets of controls determine when the Worker stops. Both must be configured deliberately. <!-- docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx:138-142 -->

| Control | Owner | What it measures |
|---|---|---|
| Worker idle and graceful-shutdown policy | The Runtime handler you write | When the Worker has been idle. It then stops polling and waits for in-flight Activities. |
| `idleRuntimeSessionTimeout` | AgentCore lifecycle settings | Time since the last **AgentCore Runtime invocation** to the session. Default 900 seconds. Polling Temporal does **not** reset it. |
| `maxLifetime` | AgentCore lifecycle settings | Time since the compute was created. Default and microVM maximum: 28800 seconds (8 hours). |

<!-- docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx:144-150; aws:runtime-lifecycle-settings.html#configuration-attributes -->

The idle session timeout is not a Worker idle timer, and it does not replace one. Implement a Worker shutdown policy in the handler: when the idle condition is met, stop polling and drain in-flight Activities before the handler releases the session. Choose the idle period and drain timeout for the workload, and keep both inside `maxLifetime`. <!-- docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx:152-155 -->

`idleRuntimeSessionTimeout` must be less than or equal to `maxLifetime`, and both must be at least 60 seconds. A request that breaks these rules fails with `ValidationException`. When AgentCore ends compute, termination can take up to 15 seconds. <!-- aws:runtime-lifecycle-settings.html#configuration-attributes, #validation-and-constraints -->

In the Python sample, the Worker drains after `AGENTCORE_DEBOUNCE_SECONDS` without an Activity starting or finishing. The default is `60`. The sample's drain timeout is 120 seconds, and its `agentcore.json` keeps the AgentCore defaults (`idleRuntimeSessionTimeout: 900`, `maxLifetime: 28800`). These are configured values, not platform limits. <!-- samples-python/bedrock_agentcore/strands_agent/agentcore_worker.py:54-56; samples-python/bedrock_agentcore/strands_agent/agentcore/agentcore.json:50-57; docs/guides/durable-agent-on-agentcore.mdx:373-375 -->

## What bounds an Activity: AgentCore ending the compute

AgentCore can end the compute before an Activity completes. Configure Activity timeouts. For long-running Activities, also configure [Activity Heartbeats](https://docs.temporal.io/encyclopedia/detecting-activity-failures#activity-heartbeat) so a retry can recover the work. <!-- docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx:157-159 --> A Worker crash follows standard retry semantics: the Activity Timeout fires and Temporal retries the Activity on another Worker. <!-- docs/encyclopedia/workers/serverless-workers/index.mdx:180-186 -->

## Autoscaling

Autoscaling is event-driven (`no-sync`). The WCI invokes individual Runtime sessions when it needs more capacity. It does not manage a target-sized pool. <!-- docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx:87-91; docs/production-deployment/worker-deployments/serverless-workers/agentcore-self-hosted-setup.mdx:70-72 --> For the shared WCI lifecycle and inspection commands, see [Worker Controller Instance (WCI)](../wci.md).

When the provider's capacity limit is reached, the WCI cannot add Workers. Tasks wait in the backlog without data loss. <!-- docs/encyclopedia/workers/serverless-workers/index.mdx:188-195 -->

## One named endpoint per Worker Deployment Version

The compute configuration names a Runtime **endpoint** ARN, not a Runtime ARN. Each Worker Deployment Version needs its own named endpoint pinned to one immutable Runtime version. Never configure the `DEFAULT` endpoint, because it moves to the latest Runtime version on every update. → `versioning.md`. <!-- docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx:95-124 -->

## Region and high availability

The AWS account must be in an [AgentCore-supported Region](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/agentcore-regions.html). <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:47 --> Compute provider configuration is scoped to one region. On failover of a replicated Namespace, the WCI keeps invoking Workers in the original region until the compute provider is repointed manually. <!-- docs/encyclopedia/workers/serverless-workers/index.mdx:217 -->

## AgentCore services are not configured by the compute provider

Configuring AgentCore Runtime as a compute provider does not set up AgentCore Identity, Gateway, Policy, Memory or Observability. Call those services from Activities, and keep credentials out of Workflow state. <!-- docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx:44-57 --> Writing the agent itself (Workflow and Activity boundaries, Strands, model calls) is out of scope for this skill. Link to [Build a durable agent on Amazon Bedrock AgentCore](https://docs.temporal.io/guides/durable-agent-on-agentcore) instead. <!-- docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx:29-32 -->
