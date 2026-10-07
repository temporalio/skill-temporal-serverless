# AgentCore — Diagnostics and troubleshooting

<!-- Sources:
  docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx
  docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx
  docs/guides/durable-agent-on-agentcore.mdx
  docs/encyclopedia/workers/serverless-workers.mdx (via ../wci.md)
  aws: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/agent-runtime-versioning.html
  aws: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-lifecycle-settings.html
  samples-python@d67795f: bedrock_agentcore/strands_agent/
-->

Temporal has no AgentCore troubleshooting page. This guide uses only the gotchas documented in the sources above and the shared [Worker Controller Instance (WCI)](../wci.md) inspection flow. If a symptom is not covered here, gather the evidence below and report it to the user. Do not attach an undocumented explanation to it.

## Invocation flow (when working correctly)

<!-- docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx:34-37,135-136; docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:419-422; docs/guides/durable-agent-on-agentcore.mdx:350-352 -->

1. Creating the Worker Deployment Version starts its WCI, which invokes the named endpoint and waits for the Worker to register.
2. The Runtime's `/invocations` handler starts the Worker in the background and acknowledges.
3. The Worker polls the Task Queue as `<deployment-name>/<build-id>`, and the version binds the Task Queue.
4. When Tasks arrive and no Worker is polling, the WCI invokes the endpoint again.
5. The Worker drains when its idle policy fires, or stops when AgentCore ends the compute.

## Start here: did the expected Worker register?

```bash
temporal worker deployment describe-version \
  --namespace <NAMESPACE> --deployment-name <DEPLOYMENT_NAME> --build-id <BUILD_ID> \
  --report-task-queue-stats -o json
```

The decisive signal is that the expected Task Queue appears in `taskQueuesInfos` with the types the Worker polls. That only happens after a Worker that identifies as this deployment name and Build ID has polled. A successful Validate Connection or a Running WCI supports the diagnosis but does not replace this check. Wait for this state to appear rather than for a fixed time.

| Result | Next check |
|---|---|
| Task Queue bound | Registration succeeded. Confirm the version is current, then go to [Tasks not completing](#worker-runs-but-tasks-do-not-complete). |
| Not bound, and the WCI history shows invocation failures | [The Runtime is not invoked](#the-runtime-is-not-invoked) |
| Not bound, invocation succeeded | [Invoked, but no Worker registers](#invoked-but-no-worker-registers) |

## The Runtime is not invoked

1. **Validate Connection (Cloud).** In the version's **Actions → Validate Connection**, a failure means Temporal cannot assume the invocation role, get the named endpoint, or invoke the Runtime. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:401-403 --> Check the following:
   - `--aws-agentcore-endpoint-arn` is the **endpoint** ARN (`…/runtime-endpoint/<name>`), not the Runtime ARN. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:266-285 -->
   - The role's `AgentRuntimeARNs` is the Runtime ARN **with a trailing `*`**. Without the wildcard, `GetAgentRuntimeEndpoint` on the endpoint is not covered. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:305-306; assets/temporal-cloud-serverless-worker-agentcore-role.yaml:13-19 -->
   - The External ID is identical in the stack and in the version. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:300-301 -->
   - Self-hosted: the server's AWS identity is the stack's `TemporalIamRoleArn` and has `sts:AssumeRole` on the role. The server can also reach the AgentCore control-plane and data-plane APIs. → `self-hosted.md`.
2. **Stack never finished.** If the role stack failed, check the role-name length. On Cloud, `<ROLE_NAME>-<STACK_NAME>` must be 64 characters or fewer. → `iam.md`. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:308-315 -->
3. **Endpoint not ready.** Check the endpoint's state with `agentcore status --type runtime-endpoint --json`. `CREATE_FAILED` and `UPDATE_FAILED` mean a failure caused by permissions, the artifact, or another configuration problem. <!-- aws:agent-runtime-versioning.html#endpoint-lifecycle -->
4. **Version not current.** If registration succeeded but new Workflows never reach the Worker, set the version current (`setup.md` Step 6). <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:405-417 -->
5. **Read the WCI history** for the failing Activity result. Use the inspection commands and interpretation rules in [`../wci.md`](../wci.md#inspect-the-wci).

## Invoked, but no Worker registers

Read the Runtime logs first: <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:428; docs/guides/durable-agent-on-agentcore.mdx:384-388 -->

```bash
agentcore logs --runtime <RUNTIME_NAME> --since 1h
```

- **Name mismatch.** The Runtime's `TEMPORAL_DEPLOYMENT_NAME` / `TEMPORAL_BUILD_ID` differ from the version's. Temporal starts compute that registers as some other version. Fix the environment, which creates a new Runtime version. Then follow `versioning.md`. <!-- docs/guides/durable-agent-on-agentcore.mdx:211-214 -->
- **Missing versioning settings.** The sample Worker refuses to start without `TEMPORAL_DEPLOYMENT_NAME` and `TEMPORAL_BUILD_ID`. <!-- samples-python/bedrock_agentcore/strands_agent/README.md:92 -->
- **Cannot reach Temporal.** With a VPC network mode, the VPC needs outbound access to the Temporal Service. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:287-288 --> For self-hosted, see `self-hosted.md` step 1.
- **Endpoint runs other code.** The version points at `DEFAULT`, or at a named endpoint that was moved to another Runtime version. → `versioning.md`. <!-- docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx:118-124 -->
- **Deploy failed with `uv install failed ... with exit code null`.** `uv` is missing on the operator's machine. <!-- samples-python/bedrock_agentcore/strands_agent/bin/create-runtime.sh:16-18 -->

## Worker runs but Tasks do not complete

- **Non-determinism errors after a deploy.** Code changed underneath a live Worker Deployment Version, because the version uses `DEFAULT` or a repointed endpoint. Pinned Workflows are affected too. Restore the old endpoint-to-version mapping, as described in `versioning.md`. <!-- docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx:118-124 -->
- **Old code still running after a deploy.** Existing sessions keep the code that was deployed when their microVM was created, until they terminate. <!-- aws:runtime-lifecycle-settings.html#lifecycle-and-session-relationship -->
- **Activities interrupted.** AgentCore ended the compute (`maxLifetime`) before the Activity completed, or the idle policy drained too early. Configure Activity timeouts and Heartbeats so retries recover the work. → `constraints.md`. <!-- docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx:157-159 -->

## Sessions stay alive after work is done

In the sample, `/ping` reports `HealthyBusy` while an async task is registered, and AgentCore keeps the session alive while it does. If the handler never calls `complete_async_task` when the Worker exits, the session stays `HealthyBusy` until `maxLifetime`. <!-- samples-python/bedrock_agentcore/strands_agent/agentcore_worker.py:4-12,146-151 --> The idle session timeout does not help either, because it resets only on AgentCore Runtime invocations, not on Temporal polling. <!-- docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx:146-148 --> The fix is a Worker idle policy that drains the Worker and releases the async task in all cases, including errors. The language-specific code is in the selected `sdk-<language>.md`.

## Lifecycle configuration rejected

`ValidationException` on a Runtime create or update means one of these: a lifecycle value is below 60 seconds or above 28800 seconds (microVMs), or `idleRuntimeSessionTimeout` is greater than `maxLifetime`. <!-- aws:runtime-lifecycle-settings.html#validation-and-constraints -->

## Rules

- Inspect the specific Runtime, endpoint, stack and version named in the inventory. Do not enumerate AgentCore resources across Regions to reconstruct state.
- Do not change IAM, redeploy, or repoint an endpoint until the evidence above identifies the cause.
