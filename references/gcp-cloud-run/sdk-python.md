# Python SDK on GCP Cloud Run

<!-- Source: docs/develop/python/workers/serverless-workers/cloud-run.mdx -->

Use this reference for Python-specific Worker construction, versioning behavior, connection configuration, image packaging, and scale-in safety. For the shared Cloud Run deployment lifecycle, permissions, versioning model, observability, and diagnostics, see `setup.md`, `iam.md`, `versioning.md`, `observability.md`, and `diagnostics.md`.

**There is no Cloud Run Worker package.** This is an ordinary long-lived Python Worker plus Worker Versioning, which Serverless Workers require. Nothing in `../aws-lambda/sdk-python.md` applies.

## Inspect the versioning API before generating code

Worker Versioning is a Public Preview surface and the option names differ between SDKs. Read the installed version's API rather than writing from memory:

```bash
python -c "import temporalio.worker as w; print([n for n in dir(w) if 'Deployment' in n])"
python -c "from temporalio.worker import WorkerDeploymentConfig; help(WorkerDeploymentConfig)"
python -c "from temporalio.common import VersioningBehavior; print(list(VersioningBehavior))"
```

## Versioned Worker

Pass `deployment_config` to `Worker()`. The Worker reads its connection settings and Task Queue from the environment so one image runs against any Namespace:

```python
import asyncio
import os
import signal
from datetime import timedelta

from temporalio.client import Client
from temporalio.common import VersioningBehavior, WorkerDeploymentVersion
from temporalio.envconfig import ClientConfig
from temporalio.worker import Worker, WorkerDeploymentConfig

from my_activities import my_activity
from my_workflows import MyWorkflow


async def main() -> None:
    client = await Client.connect(**ClientConfig.load_client_connect_config())

    worker = Worker(
        client,
        task_queue=os.environ["TEMPORAL_TASK_QUEUE"],
        workflows=[MyWorkflow],
        activities=[my_activity],
        deployment_config=WorkerDeploymentConfig(
            version=WorkerDeploymentVersion(
                deployment_name="my-app",
                build_id="build-1",
            ),
            use_worker_versioning=True,
            default_versioning_behavior=VersioningBehavior.PINNED,
        ),
        graceful_shutdown_timeout=timedelta(seconds=8),
    )
    stop = asyncio.Event()
    loop = asyncio.get_running_loop()
    for sig in (signal.SIGTERM, signal.SIGINT):
        loop.add_signal_handler(sig, stop.set)

    async with worker:
        await stop.wait()


if __name__ == "__main__":
    asyncio.run(main())
```

`deployment_name` and `build_id` must match the version created with `temporal worker deployment create-version` exactly, or the Worker polls under a version the WCI does not manage. → `setup.md` Step 6.

## Versioning behavior

Every Workflow needs `VersioningBehavior.PINNED` or `AUTO_UPGRADE`. `default_versioning_behavior` on `WorkerDeploymentConfig` covers every Workflow; to set it per Workflow, pass `versioning_behavior` to the decorator.

```python
@workflow.defn(versioning_behavior=VersioningBehavior.PINNED)
class MyWorkflow:
    @workflow.run
    async def run(self, name: str) -> str:
        ...
```

**A Version set with no behavior fails at runtime**, not at build time.

## Connection configuration

`temporalio.envconfig` loads client configuration from environment variables and an optional TOML file. Set non-secret values with `--set-env-vars` on the pool and mount the API key or TLS material from Secret Manager with `--set-secrets`. → `setup.md` Step 4.

`ClientConfig.load_client_connect_config()` returns keyword arguments for `Client.connect`, which is why it is unpacked with `**`. To inspect or override values first, load the profile instead:

```python
from temporalio.envconfig import ClientConfigProfile

profile = ClientConfigProfile.load()
connect_config = profile.to_client_connect_config()
client = await Client.connect(**connect_config)
```

## Image packaging

`pip install "temporalio>=1.30.0,<2"` and run the Worker module as the entrypoint. Python shares the Rust core, so a minimal base image needs `ca-certificates` present. → `setup.md` Step 2.

## Graceful shutdown on scale-in

Cloud Run sends `SIGTERM` before stopping an instance. The example converts it into an `asyncio.Event`; leaving the `async with worker` block calls `worker.shutdown()` and waits for the SDK's shutdown sequence. `graceful_shutdown_timeout` gives received Activities up to eight seconds before cancellation. Prefer this explicit path over cancelling `worker.run()`, because cancellation can also cancel the shutdown operation.

Cloud Run can send `SIGKILL` ten seconds later. Shutdown therefore improves draining but cannot guarantee that a long Activity finishes; Activities must cooperate with cancellation and record Heartbeats as shown below.

## Keep Activities safe across scale-in

The WCI removes instances based on Task Queue activity, not on what an individual instance is doing, so **an instance running a long Activity can be stopped mid-execution.** Record Heartbeats so a retry resumes from the last recorded progress:

```python
@activity.defn
async def my_activity(items: list[str]) -> str:
    for i, item in enumerate(items):
        activity.heartbeat(i)
        # ... process item
    return "done"
```

→ `constraints.md` for what else follows from the pool model.

## Logging and diagnostic signatures

Configure application logging before constructing the client or Worker. Cloud Run captures stdout and stderr automatically:

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s %(levelname)s %(name)s %(message)s",
)
logging.getLogger("temporalio").setLevel(logging.INFO)
```

Do not enable DEBUG logging globally in production without first verifying that dependency logs cannot contain credentials or payloads.

| Log signature | Meaning / action |
|---|---|
| `NativeCertsNotFound` | The runtime image lacks CA certificates. Install `ca-certificates` and rebuild. |
| `TransportError` during startup | Check the mounted API key, address including port, TLS, and Namespace. |
| Worker starts but the intended Workflow does not progress | Check the deployment name, build ID, Task Queue, and that the version is current. |
| No application `INFO` records | Configure the root logger before Worker construction; do not rely on an implicit handler. |

## Observability

A Cloud Run Worker emits the same traces and metrics as a Worker anywhere else — no Cloud Run-specific wiring, and none of Lambda's ADOT layer or collector configuration. Use the SDK's normal metrics export and OpenTelemetry tracing interceptors. → `observability.md`, and `docs/develop/python/platform/observability`.
