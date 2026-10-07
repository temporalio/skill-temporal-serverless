# AgentCore — Observability

<!-- Sources:
  docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx
  docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx
  docs/guides/durable-agent-on-agentcore.mdx
-->

## Two complementary views

| Question | Where to look |
|---|---|
| What did the Workflow or agent do, and which Activities ran, failed or retried? | Temporal Event History (`temporal workflow show --workflow-id <ID>`) and the Temporal UI |
| Which Worker process did the work, when did it start, and when did it drain? | AgentCore Runtime logs (`agentcore logs --runtime <RUNTIME_NAME> --since <WINDOW>`) |
| Runtime, model and tool behavior | [AgentCore Observability](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/observability-configure.html) |

<!-- docs/guides/durable-agent-on-agentcore.mdx:377-391; docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx:56 -->

Use AgentCore telemetry for the Runtime, model and tool behavior. Use Event History for durable execution progress. Neither replaces the other. <!-- docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx:56 --> AgentCore Observability is a separate AgentCore service. Configuring AgentCore as a compute provider does not set it up. <!-- docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx:44-46 --> Set it up only when the user asks for Runtime telemetry export.

## Logs

`agentcore logs --runtime <RUNTIME_NAME>` shows the Worker starting and processing Tasks. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:428 --> Log at least these points in the entry point, because they are what `diagnostics.md` correlates: Worker start (with deployment name, Build ID and Task Queue), duplicate-invocation skips, idle drain, and worker failure. The Python sample logs each of these. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:172-238 -->

When the Worker has drained, a session that is no longer running emits no new Worker logs. An empty recent window after a drain is expected. It does not mean the deployment failed.

## Provider-specific signals

Standard Temporal SDK metrics describe the Worker. The signals below describe the AgentCore side.

- **Version registration:** `taskQueuesInfos` from `temporal worker deployment describe-version` (see `diagnostics.md`).
- **Connection:** Validate Connection on the version (Cloud only). <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:401-403 -->
- **Endpoint state and version:** `agentcore status --type runtime-endpoint --json`. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:259-262 -->
- **WCI activity:** WCI history, as described in [`../wci.md`](../wci.md#inspect-the-wci).

Temporal SDK metrics and tracing are configured as for any long-running Worker. The language-specific setup, if any, is in the selected `sdk-<language>.md`.
