# AgentCore — Versioning, updates and rollback

<!-- Sources:
  docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx
  docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx
  aws: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/agent-runtime-versioning.html
  aws: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-lifecycle-settings.html
  samples-python@d67795f: bedrock_agentcore/strands_agent/
-->

## How the two version models map

Serverless Workers require Worker Versioning. **Map each Worker Deployment Version to a named AgentCore Runtime endpoint that points to exactly one Runtime version.** <!-- docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx:95-96 -->

| AgentCore primitive | Behavior | Role in Temporal |
|---|---|---|
| Runtime version | AgentCore creates version 1 when you create a Runtime. Every update to the Runtime creates a new immutable version, including code, protocol and network changes. | The immutable build behind one Worker Deployment Version. |
| Named endpoint | Has a stable ARN and points to a chosen version. It moves only when you update it explicitly. | The `--aws-agentcore-endpoint-arn` of one Worker Deployment Version. |
| `DEFAULT` endpoint | Moves to the latest version automatically on every Runtime update. | **Never** use it for a live Worker Deployment Version. |

<!-- aws:agent-runtime-versioning.html (intro, #endpoint-versioning, #versioning-scenarios); docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx:98-100,118-124 -->

**The hazard:** if a Worker Deployment Version points to an endpoint whose Runtime version changes, the code behind that Temporal version changes. Replay-unsafe changes can then cause non-determinism errors for in-flight Workflows, including Pinned ones. This happens with `DEFAULT` on every update, and with a named endpoint if you repoint it. <!-- docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx:118-124 -->

The Build ID in the Runtime environment (`TEMPORAL_BUILD_ID`) and the Worker Deployment Version's Build ID must match. The same applies to the deployment name. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:352-355 --> Changing an environment variable is a Runtime update, so it also creates a new Runtime version. <!-- aws:agent-runtime-versioning.html (intro: "Each update ... creates a new version") -->

## Rolling out a new build

1. Choose a new Build ID and set `TEMPORAL_BUILD_ID` to it in the Runtime's `envVars` in `agentcore/agentcore.json`.
2. Add a **new** named endpoint for the new Runtime version under the Runtime's `endpoints`, and keep the old endpoint unchanged. In the sample, `"version": 1` selects the first Runtime version. Point the new endpoint at the version that the deploy creates. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:67-89 -->
3. Run `agentcore validate` and `agentcore deploy --target <TARGET> -y`. Then read the Runtime version and endpoint ARNs from `agentcore status --type runtime-endpoint --json`. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:246-264 --> Before continuing, confirm that the new endpoint is `READY` and points to the intended version. Endpoint states are `CREATING`, `CREATE_FAILED`, `READY`, `UPDATING` and `UPDATE_FAILED`. <!-- aws:agent-runtime-versioning.html#endpoint-lifecycle -->
4. Create a new Worker Deployment Version with the new Build ID and the **new endpoint ARN**. Reuse the invocation role and External ID. A role created with the Runtime ARN plus a trailing wildcard already covers every endpoint of that Runtime. → `iam.md`. <!-- docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx:114-116; docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:305-306 -->
5. Set the new version current, or ramp to it. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:405-417 -->
6. **Keep the old endpoint** while Pinned Workflows can still need the older Worker code. <!-- docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx:114-116 -->

If the user's AgentCore CLI version or project layout does not let them declare the version an endpoint points to, ask them how they want to manage endpoints. Do not fall back to `DEFAULT` or repoint the live endpoint.

## Sessions already running keep their code

A microVM session uses the code assets that were deployed when the microVM was created. After a Runtime update, existing sessions keep the previous code until they terminate. <!-- aws:runtime-lifecycle-settings.html#lifecycle-and-session-relationship --> A Worker that is already polling therefore keeps running the old build until its idle policy drains it or `maxLifetime` ends its compute. This is another reason to give every build its own Worker Deployment Version and endpoint.

## Lifecycle settings are part of the Runtime configuration

`idleRuntimeSessionTimeout` and `maxLifetime` are set on the Runtime (`lifecycleConfiguration` in `agentcore.json`, or `LifecycleConfiguration` on `CreateAgentRuntime` and `UpdateAgentRuntime`). <!-- aws:runtime-lifecycle-settings.html (intro); samples-python/bedrock_agentcore/strands_agent/agentcore/agentcore.json:54-57 --> Changing them is a Runtime update, so it creates a new Runtime version. <!-- aws:agent-runtime-versioning.html (intro) --> Roll the change out like a new build: new endpoint and new Worker Deployment Version. For the meaning of each setting, see `constraints.md`.

## Rollback

Set the previous Worker Deployment Version current again. Its endpoint still points to the old immutable Runtime version, so the next invocation starts the old code without a redeploy. This works only if the old endpoint was kept. If it was deleted or repointed, recreate an endpoint for the known-good Runtime version and register it on a new Worker Deployment Version. <!-- docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx:111-116; aws:agent-runtime-versioning.html#endpoint-versioning -->
