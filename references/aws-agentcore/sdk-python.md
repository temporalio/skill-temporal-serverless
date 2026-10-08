# AgentCore — Python SDK

<!-- Sources:
  docs/develop/python/workers/serverless-workers/agentcore.mdx
  docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx
  aws: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-http-protocol-contract.html
  aws: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-code-deploy-supported-runtimes.html
  samples-python@d67795f: bedrock_agentcore/strands_agent/
-->

Use this reference for the Python entry point of an AgentCore Worker: dependencies, project layout, the versioned Worker, the Runtime handler, the idle and drain policy, and connection settings. For the shared deploy flow, permissions, lifecycle limits, versioning, observability and diagnostics, see `setup.md`, `iam.md`, `constraints.md`, `versioning.md`, `observability.md` and `diagnostics.md`. This file supplies the code for `setup.md` Step 2 and the `runtimeVersion` for Step 1; every other step is the same for every SDK.

The Worker is a standard long-lived Python Worker. The Runtime handler uses the AWS `bedrock-agentcore` package to serve AgentCore Runtime invocations. <!-- docs/develop/python/workers/serverless-workers/agentcore.mdx:24-29 -->

**Fastest path:** start from the [Python AgentCore sample](https://github.com/temporalio/samples-python/tree/main/bedrock_agentcore/strands_agent). It has a working Runtime handler, idle policy, `agentcore/agentcore.json` and deploy scripts. Replace `StrandsPlugin`, `StrandsAgentWorkflow` and `execute_code` with the application's plugins, Workflows and Activities. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:241-242 --> To design the agent itself (Strands, Bedrock model calls, Workflow and Activity boundaries), link the user to [Build a durable agent on Amazon Bedrock AgentCore](https://docs.temporal.io/guides/durable-agent-on-agentcore); this skill does not cover it.

## Install and scaffold

Declare `temporalio`, `bedrock-agentcore` and the application's dependencies in `pyproject.toml`. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:157; docs/develop/python/workers/serverless-workers/agentcore.mdx:36-42 --> The sample's floors are: <!-- samples-python/bedrock_agentcore/strands_agent/pyproject.toml:9-14 -->

```toml
[project]
requires-python = ">=3.10"
dependencies = [
    "temporalio>=1.32.0,<2",
    "bedrock-agentcore>=1.9.1",
]
```

For a CodeZip Runtime, AgentCore packages the `codeLocation` directory, and `entrypoint` is relative to it. That directory must contain the entry point, every local module it imports, and `pyproject.toml`. The sample sets `codeLocation` to `.` and uses this layout: <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:135-151 -->

```text
project-root/
├── agentcore/
│   ├── agentcore.json
│   ├── aws-targets.json
│   └── cdk/
├── agentcore_worker.py   # entrypoint: "agentcore_worker.py"
├── workflows.py
├── activities.py
└── pyproject.toml
```

If the AgentCore project keeps code in a subdirectory such as `app/MyAgent`, set `codeLocation` to that directory and put the entry point, its modules and `pyproject.toml` there. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:153-155 -->

Set `runtimeVersion` in `agentcore.json` to a supported Python identifier. The sample uses `PYTHON_3_12`; AgentCore also lists `PYTHON_3_10` through `PYTHON_3_14`. Check the AWS table for deprecation dates before choosing one. <!-- samples-python/bedrock_agentcore/strands_agent/agentcore/agentcore.json:13-16; aws:runtime-code-deploy-supported-runtimes.html#concept-supported-runtimes-python -->

The AgentCore CLI's CDK build installs the dependencies for the Runtime's platform with `uv`, so `uv` must be installed on the operator's machine. → `setup.md` Prerequisites.

## Versioned Worker

Serverless Workers require Worker Versioning. Pass `deployment_config` to `Worker()` with the deployment name and Build ID from the Runtime environment: <!-- docs/develop/python/workers/serverless-workers/agentcore.mdx:44-61 -->

```python
worker = Worker(
    client,
    task_queue=TASK_QUEUE,
    workflows=[MyWorkflow],
    activities=[my_activity],
    interceptors=[tracker],
    deployment_config=WorkerDeploymentConfig(
        version=WorkerDeploymentVersion(
            deployment_name=DEPLOYMENT_NAME,
            build_id=BUILD_ID,
        ),
        use_worker_versioning=True,
        default_versioning_behavior=VersioningBehavior.PINNED,
    ),
    graceful_shutdown_timeout=DRAIN,
)
```

`DEPLOYMENT_NAME` and `BUILD_ID` come from `TEMPORAL_DEPLOYMENT_NAME` and `TEMPORAL_BUILD_ID` in the Runtime's `envVars`. They must match the Worker Deployment Version created in `setup.md` Step 5 exactly. <!-- docs/develop/python/workers/serverless-workers/agentcore.mdx:63-66; samples-python/bedrock_agentcore/strands_agent/agentcore_worker.py:48-50 --> A new Build ID needs a new Runtime version and a new named endpoint. → `versioning.md`.

## Versioning behavior

Every Workflow needs a versioning behavior, `PINNED` or `AUTO_UPGRADE`. `default_versioning_behavior` applies one behavior to every Workflow on the Worker. To set it per Workflow, pass `versioning_behavior` to `@workflow.defn`. <!-- docs/develop/python/workers/serverless-workers/agentcore.mdx:68-70 -->

```python
@workflow.defn(versioning_behavior=VersioningBehavior.PINNED)
class MyWorkflow:
    @workflow.run
    async def run(self, prompt: str) -> str:
        ...
```

## Runtime handler

`BedrockAgentCoreApp` from `bedrock_agentcore.runtime` serves the AgentCore Runtime HTTP contract: `POST /invocations` and `GET /ping` on port 8080. <!-- samples-python/bedrock_agentcore/strands_agent/agentcore_worker.py:4-6,25,41; aws:runtime-http-protocol-contract.html#container-requirements-http, #path-requirements-http --> Do not write these routes by hand.

The `/invocations` payload is not Workflow input. The Worker Controller Instance invokes the endpoint only to add Worker capacity; applications start Workflows through a Temporal Client as usual. <!-- docs/develop/python/workers/serverless-workers/agentcore.mdx:148-149 --> The handler must therefore:

1. Register an AgentCore async task with `app.add_async_task(...)`. While a task is registered, `/ping` reports `HealthyBusy` and AgentCore keeps the session alive. <!-- docs/develop/python/workers/serverless-workers/agentcore.mdx:149-151; aws:runtime-http-protocol-contract.html#ping-endpoint -->
2. Start the Worker as a background `asyncio` task and return an acknowledgment immediately. Do not hold the response open while the Worker polls. <!-- samples-python/bedrock_agentcore/strands_agent/agentcore_worker.py:8-12 -->
3. Keep a module-level reference to that task. It stops garbage collection of the task and prevents a second invocation in the same session from starting a duplicate Worker. <!-- docs/develop/python/workers/serverless-workers/agentcore.mdx:74-77; samples-python/bedrock_agentcore/strands_agent/agentcore_worker.py:44-46 -->
4. Call `app.complete_async_task(task_id)` in a `finally` block, whether the Worker drained or failed. Without it the session stays `HealthyBusy` until `maxLifetime`. → `diagnostics.md`.

The sample's handler, with the application-specific names replaced: <!-- docs/develop/python/workers/serverless-workers/agentcore.mdx:79-146; samples-python/bedrock_agentcore/strands_agent/agentcore_worker.py:20-50,142-172 -->

```python
import asyncio
import os

import boto3
from bedrock_agentcore.runtime import BedrockAgentCoreApp
from temporalio.client import Client
from temporalio.common import VersioningBehavior
from temporalio.worker import Worker, WorkerDeploymentConfig, WorkerDeploymentVersion

from activities import my_activity
from workflows import MyWorkflow

app = BedrockAgentCoreApp()
log = app.logger

_worker: asyncio.Task[None] | None = None

DEPLOYMENT_NAME = os.environ["TEMPORAL_DEPLOYMENT_NAME"]
BUILD_ID = os.environ["TEMPORAL_BUILD_ID"]
TASK_QUEUE = os.environ["TEMPORAL_TASK_QUEUE"]

# ActivityTracker, DEBOUNCE and DRAIN: see "Stop and drain the Worker".


def load_api_key() -> str | None:
    """Read the Temporal API key from Secrets Manager; never log it."""
    secret_arn = os.environ.get("TEMPORAL_API_KEY_SECRET_ARN")
    if not secret_arn:
        return None
    secrets = boto3.client("secretsmanager")
    return secrets.get_secret_value(SecretId=secret_arn)["SecretString"]


async def run_worker() -> None:
    """Poll until idle, then drain."""
    api_key = await asyncio.to_thread(load_api_key)
    client = await Client.connect(
        os.environ["TEMPORAL_ADDRESS"],
        namespace=os.environ["TEMPORAL_NAMESPACE"],
        api_key=api_key,
        tls=bool(api_key),
    )
    tracker = ActivityTracker()
    log.info("polling %s as %s/%s", TASK_QUEUE, DEPLOYMENT_NAME, BUILD_ID)
    worker = Worker(...)  # the versioned Worker above
    async with worker:
        await tracker.wait_until_idle(DEBOUNCE)
    log.info("worker idle for %ss; drained", DEBOUNCE)


async def _run_until_idle(task_id: int) -> None:
    try:
        await run_worker()
    except Exception:
        # Nothing awaits this task, so an error would otherwise be swallowed.
        log.exception("worker failed in async task")
    finally:
        # Without this the session stays HealthyBusy until maxLifetime.
        app.complete_async_task(task_id)


@app.entrypoint
async def invoke(payload: dict) -> dict:
    """Start the Worker and acknowledge. The payload is unused."""
    global _worker
    if _worker is not None and not _worker.done():
        log.info("worker already polling %s", TASK_QUEUE)
        return {"message": "worker already polling", "task_queue": TASK_QUEUE}
    task_id = app.add_async_task("temporal-worker")
    _worker = asyncio.create_task(_run_until_idle(task_id))
    return {"message": "worker starting", "task_queue": TASK_QUEUE}


if __name__ == "__main__":
    app.run()
```

The sample falls back to defaults when these variables are unset; this version reads `os.environ[...]` so a missing setting fails at startup instead of polling the wrong Namespace or Task Queue. If an Activity is a synchronous function, pass an `activity_executor` such as a `ThreadPoolExecutor` to `Worker()`, as the sample does for `execute_code`. <!-- samples-python/bedrock_agentcore/strands_agent/agentcore_worker.py:48-50; docs/develop/python/workers/serverless-workers/agentcore.mdx:95-102 -->

## Stop and drain the Worker

AgentCore cannot tell whether a polling Worker has work. Because the async task keeps the Runtime busy, the Worker would otherwise stay up until the 8-hour maximum lifetime. The handler must decide when the Worker is idle, then leave the `async with worker` block: the Worker stops polling and waits up to `graceful_shutdown_timeout` for in-flight Activities. <!-- docs/develop/python/workers/serverless-workers/agentcore.mdx:163-170 -->

The sample's `ActivityTracker` is an Activity inbound Interceptor that counts running Activities and returns once no Activity has started or finished for `DEBOUNCE` seconds and none is running: <!-- docs/develop/python/workers/serverless-workers/agentcore.mdx:172-230 -->

```python
from datetime import timedelta

from temporalio.worker import (
    ActivityInboundInterceptor,
    ExecuteActivityInput,
    Interceptor,
)

# How long the Worker keeps polling after it goes idle.
DEBOUNCE = float(os.environ.get("AGENTCORE_DEBOUNCE_SECONDS", "60"))
# How long the drain waits for in-flight Activities.
DRAIN = timedelta(seconds=120)


class ActivityTracker(Interceptor):
    def __init__(self) -> None:
        self.inflight = 0
        self.changed = asyncio.Event()

    def intercept_activity(
        self, next: ActivityInboundInterceptor
    ) -> ActivityInboundInterceptor:
        return _TrackedActivity(next, self)

    async def wait_until_idle(self, debounce: float) -> None:
        while True:
            self.changed.clear()
            try:
                await asyncio.wait_for(self.changed.wait(), timeout=debounce)
            except asyncio.TimeoutError:
                if self.inflight == 0:
                    return


class _TrackedActivity(ActivityInboundInterceptor):
    def __init__(
        self, next: ActivityInboundInterceptor, tracker: ActivityTracker
    ) -> None:
        super().__init__(next)
        self._tracker = tracker

    async def execute_activity(self, input: ExecuteActivityInput):
        self._tracker.inflight += 1
        self._tracker.changed.set()
        try:
            return await self.next.execute_activity(input)
        finally:
            self._tracker.inflight -= 1
            self._tracker.changed.set()
```

A long-running Activity keeps the count above zero, so the idle policy does not interrupt it. `graceful_shutdown_timeout` is a safety limit for Activities still running when shutdown starts. <!-- docs/develop/python/workers/serverless-workers/agentcore.mdx:247-249 -->

`AGENTCORE_DEBOUNCE_SECONDS` (set in the Runtime's `envVars`) controls the idle period and `DRAIN` controls the drain. The sample's 60 and 120 seconds are configured values, not requirements. Choose both for the workload and keep them inside `maxLifetime`. → `constraints.md`. <!-- docs/develop/python/workers/serverless-workers/agentcore.mdx:256-259 -->

Memory pressure can be an additional retirement condition: the handler can monitor process memory and start the same graceful shutdown above a threshold. It decides when to recycle the Worker, not whether it is idle. Test such a policy against the Runtime's memory limit and the Activities' retry behavior. <!-- docs/develop/python/workers/serverless-workers/agentcore.mdx:251-254 -->

## Connection configuration

Set `TEMPORAL_ADDRESS`, `TEMPORAL_NAMESPACE`, `TEMPORAL_TASK_QUEUE`, `TEMPORAL_DEPLOYMENT_NAME` and `TEMPORAL_BUILD_ID` in the Runtime's `envVars` (`setup.md` Step 1). For Temporal Cloud, also set `TEMPORAL_API_KEY_SECRET_ARN`. The handler above reads the key from that secret each time it starts a Worker, passes it to `Client.connect`, and turns TLS on only when a key is set. That covers Temporal Cloud with an API key and a self-hosted Service without TLS. <!-- docs/develop/python/workers/serverless-workers/agentcore.mdx:82-91 --> `boto3` is already installed as a dependency of `bedrock-agentcore`, and it authenticates as the Runtime execution role. <!-- samples-python/bedrock_agentcore/strands_agent/uv.lock:244-245 -->

To use the shared environment-configuration format and profiles instead (including mTLS settings), load the connection with `temporalio.envconfig`. <!-- docs/develop/python/workers/serverless-workers/agentcore.mdx:153-161 -->

```python
from temporalio.envconfig import ClientConfig

client = await Client.connect(**ClientConfig.load_client_connect_config())
```

The API key and any TLS material live in AWS Secrets Manager, never in `agentcore.json` or a Runtime env var. The user stores the key from their own terminal, and the execution role gets `secretsmanager:GetSecretValue` on that one secret ARN. → `setup.md`, [Store the Temporal API key](setup.md#store-the-temporal-api-key). Rotation needs no redeploy, because each new Worker reads the current version. → `iam.md`, [Temporal API key lifecycle](iam.md#temporal-api-key-lifecycle). Never log the value. <!-- docs/develop/python/workers/serverless-workers/agentcore.mdx:156-158; docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:128-131 -->

## Package and deploy

There is no separate Python build step. `agentcore validate` and `agentcore deploy --target <TARGET> -y` package the `codeLocation` directory as a CodeZip archive, install the dependencies from `pyproject.toml` for the Runtime's ARM64 platform, and deploy it. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:244-254; samples-python/bedrock_agentcore/strands_agent/README.md:40-42 --> → `setup.md` Step 3. Do not build a container image.

Before deploying, check that:

- `entrypoint` names the file that calls `app.run()`, relative to `codeLocation`;
- every module the entry point imports is inside `codeLocation`; and
- `pyproject.toml` in `codeLocation` lists `temporalio` and `bedrock-agentcore`.

## Keep Activities safe across Worker termination

AgentCore can end the compute that runs a Worker, and an Activity running at that moment is retried. Set Activity timeouts, and Heartbeat long-running Activities so a retry resumes from its last recorded progress: <!-- docs/develop/python/workers/serverless-workers/agentcore.mdx:261-277 -->

```python
from temporalio import activity


@activity.defn
async def my_activity(items: list[str]) -> str:
    start = activity.info().heartbeat_details[0] if activity.info().heartbeat_details else 0
    for i in range(start, len(items)):
        # ... process items[i]; each step must be safe to repeat
        activity.heartbeat(i + 1)
    return "done"
```

→ `constraints.md`.

## Logging and diagnostic signatures

Write Worker lifecycle logs with `app.logger`, as the handler above does, and read them with `agentcore logs --runtime <RUNTIME_NAME>`. The handler logs the four points `observability.md` requires, with the same messages as the sample. <!-- samples-python/bedrock_agentcore/strands_agent/agentcore_worker.py:118,139,148,160 --> Never log the API key or the secret value.

| Log signature | Meaning / action |
|---|---|
| `polling <TASK_QUEUE> as <DEPLOYMENT_NAME>/<BUILD_ID>` | A Worker started in this session. If the deployment name or Build ID differs from the Worker Deployment Version, the Runtime's env vars are wrong. → `diagnostics.md`, [Invoked, but no Worker registers](diagnostics.md#invoked-but-no-worker-registers). |
| `worker already polling <TASK_QUEUE>` | Another invocation reached a session whose Worker is still running. No second Worker started. Expected. |
| `worker idle for <N>s; drained` | The idle policy fired and the Worker drained. The session emits no further Worker logs; this is not a failure. → `observability.md`. |
| `worker failed in async task`, followed by a traceback | `run_worker` raised, and the async task was released. Diagnose from the exception in the traceback. |
| `KeyError` naming a `TEMPORAL_*` variable at startup | A required variable is missing from the Runtime's `envVars`. Set it (`setup.md` Step 1) and redeploy. |
| `uv install failed ... with exit code null` from `agentcore deploy` | `uv` is missing on the operator's machine. → `setup.md` Prerequisites. <!-- samples-python/bedrock_agentcore/strands_agent/bin/create-runtime.sh:16-18 --> |

Temporal documents no other AgentCore-specific Python log signatures. For anything else, follow `diagnostics.md`.

## Observability

An AgentCore Worker emits the same traces and metrics as a Worker on other compute. Configure them with the Python SDK's standard metrics and OpenTelemetry tracing interceptors; there is no AgentCore-specific helper. <!-- docs/develop/python/workers/serverless-workers/agentcore.mdx:279-283 --> `BedrockAgentCoreApp` exposes `app.logger`, which the sample uses for Worker lifecycle logs; read them with `agentcore logs --runtime <RUNTIME_NAME>`. <!-- samples-python/bedrock_agentcore/strands_agent/agentcore_worker.py:41-42 --> → `observability.md`.
