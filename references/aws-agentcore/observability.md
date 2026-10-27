# AgentCore — Observability

## Two complementary views

| Question | Where to look |
|---|---|
| What did the Workflow or agent do, and which Activities ran, failed or retried? | Temporal Event History (`temporal workflow show --workflow-id <ID>`) and the Temporal UI |
| Which Worker process did the work, when did it start, and when did it drain? | AgentCore Runtime logs (`aws logs tail /aws/bedrock-agentcore/runtimes/<RUNTIME_ID>-<ENDPOINT_NAME> --since <WINDOW>`) |
| Runtime, model and tool behavior | [AgentCore Observability](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/observability-configure.html) |

Use AgentCore telemetry for the Runtime, model and tool behavior. Use Event History for durable execution progress. Neither replaces the other. AgentCore Observability is a separate AgentCore service. Configuring AgentCore as a compute provider does not set it up. Set it up only when the user asks for Runtime telemetry export.

## Logs

The Runtime logs show the Worker starting and processing Tasks. AgentCore writes each endpoint's logs to its own log group, `/aws/bedrock-agentcore/runtimes/<RUNTIME_ID>-<ENDPOINT_NAME>`. Temporal invokes the named endpoint, so read that group:

```bash
aws logs tail /aws/bedrock-agentcore/runtimes/<RUNTIME_ID>-<ENDPOINT_NAME> --since 1h --region <AWS_REGION>
```

`agentcore logs --runtime <RUNTIME_NAME>` (CLI 0.28.1) returned no Worker logs for a Runtime that Temporal invoked through a named endpoint, so use the log group directly. Log at least these points in the entry point, because they are what `diagnostics.md` correlates: Worker start (with deployment name, Build ID and Task Queue), duplicate-invocation skips, idle drain, and worker failure. The Python sample logs each of these, and the handlers in both SDK guides log them with the same messages; the exact lines are in the selected `sdk-<language>.md`, under "Logging and diagnostic signatures".

When the Worker has drained, a session that is no longer running emits no new Worker logs. An empty recent window after a drain is expected. It does not mean the deployment failed.

## Provider-specific signals

Standard Temporal SDK metrics describe the Worker. The signals below describe the AgentCore side.

- **Version registration:** `taskQueuesInfos` from `temporal worker deployment describe-version` (see `diagnostics.md`).
- **Connection:** Validate Connection on the version (Cloud only).
- **Endpoint state and version:** `agentcore status --type runtime-endpoint --json`.
- **WCI activity:** WCI history, as described in [`../wci.md`](../wci.md#inspect-the-wci).

Temporal SDK metrics and tracing are configured as for any long-running Worker. The language-specific setup, if any, is in the selected `sdk-<language>.md`.
