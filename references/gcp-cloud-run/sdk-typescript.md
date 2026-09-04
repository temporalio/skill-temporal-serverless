# TypeScript SDK on GCP Cloud Run

<!-- Source: docs/develop/typescript/workers/serverless-workers/cloud-run.mdx -->

Use this reference for TypeScript-specific Worker construction, versioning behavior, connection configuration, image packaging, and scale-in safety. For the shared Cloud Run deployment lifecycle, permissions, versioning model, observability, and diagnostics, see `setup.md`, `iam.md`, `versioning.md`, `observability.md`, and `diagnostics.md`.

**There is no Cloud Run Worker package.** This is an ordinary long-lived TypeScript Worker plus Worker Versioning, which Serverless Workers require. Nothing in `../aws-lambda/sdk-typescript.md` applies.

## Inspect the versioning API before generating code

Worker Versioning is a Public Preview surface and the option names differ between SDKs. Read the installed version's API rather than writing from memory:

```bash
npm ls @temporalio/worker
grep -rn "workerDeploymentOptions\|WorkerDeploymentOptions" node_modules/@temporalio/worker/lib/*.d.ts
```

## Versioned Worker

Pass `workerDeploymentOptions` to `Worker.create()`. The Worker reads its connection settings and Task Queue from the environment so one image runs against any Namespace:

```ts
import { NativeConnection, Worker } from '@temporalio/worker';

async function main(): Promise<void> {
  const connection = await NativeConnection.connect({
    address: process.env.TEMPORAL_ADDRESS,
    apiKey: process.env.TEMPORAL_API_KEY,
    tls: true,
  });

  const worker = await Worker.create({
    connection,
    namespace: process.env.TEMPORAL_NAMESPACE!,
    taskQueue: process.env.TEMPORAL_TASK_QUEUE!,
    workflowsPath: require.resolve('./workflows'),
    workerDeploymentOptions: {
      version: { deploymentName: 'my-app', buildId: 'build-1' },
      useWorkerVersioning: true,
      defaultVersioningBehavior: 'PINNED',
    },
    shutdownGraceTime: '8s',
    shutdownForceTime: '9s',
  });

  await worker.run();
}

main().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

`deploymentName` and `buildId` must match the version created with `temporal worker deployment create-version` exactly, or the Worker polls under a version the WCI does not manage. → `setup.md` Step 6.

Unlike Lambda, Cloud Run needs no `workflowBundle` — `workflowsPath` is fine, because the instance is long-lived and bundling only buys startup time.

## Versioning behavior

Every Workflow needs `'PINNED'` or `'AUTO_UPGRADE'`. `defaultVersioningBehavior` covers every Workflow; to set it per Workflow, use `setWorkflowOptions()` from `@temporalio/workflow`.

```ts
import { setWorkflowOptions } from '@temporalio/workflow';

setWorkflowOptions({ versioningBehavior: 'PINNED' }, myWorkflow);
export async function myWorkflow(): Promise<string> {
  // ...
}
```

**A Version set with no behavior fails at runtime**, not at build time.

## Connection configuration

Read the Namespace, address, and Task Queue from environment variables set on the pool, and mount the API key or TLS material from Secret Manager rather than passing it in plaintext. → `setup.md` Step 4.

To load them through the shared config format and profiles instead, use `loadClientConnectConfig()` from `@temporalio/envconfig` and pass its `connectionOptions` and `namespace` to `NativeConnection.connect()` and `Worker.create()`.

## Image packaging

Three things, all of which fail at startup rather than at build time:

- **Install `ca-certificates` in the runtime stage.** The Rust core reads TLS roots from the OS store and the slim Node images ship without one; on `node:22-slim` a TLS connection fails with `TransportError: tonic::transport::Error(Transport, NativeCertsNotFound)`.
- **Use a glibc image such as `node:22-slim`, not Alpine.** Alpine's musl is unsupported by the Rust core.
- **Set `NODE_OPTIONS=--max-old-space-size=<MB>`** to about 80% of the instance memory limit. Node sizes its heap from the host, not the container. A pool defaults to 512 MiB per instance, so raise `--memory` if the Worker needs more.

→ `setup.md` Step 2, `diagnostics.md`.

## Graceful shutdown on scale-in

The versioned Worker example uses `await worker.run()`. The TypeScript SDK Runtime registers `SIGINT`, `SIGTERM`, `SIGQUIT`, and `SIGUSR2` as shutdown signals by default, so Cloud Run's `SIGTERM` starts the Worker's normal shutdown without an application-level signal handler. `shutdownGraceTime` gives received Activities eight seconds before cancellation; `shutdownForceTime` prevents a non-cooperative Activity from keeping the process alive until Cloud Run kills it. If the application installs a custom Runtime, preserve `SIGTERM` in its `shutdownSignals`.

Cloud Run can send `SIGKILL` ten seconds later. Shutdown therefore improves draining but cannot guarantee that a long or non-cooperative Activity finishes; Activities must react to cancellation and record Heartbeats as shown below.

## Keep Activities safe across scale-in

The WCI removes instances based on Task Queue activity, not on what an individual instance is doing, so **an instance running a long Activity can be stopped mid-execution.** Record Heartbeats so a retry resumes from the last recorded progress:

```ts
import { heartbeat } from '@temporalio/activity';

export async function myActivity(items: string[]): Promise<string> {
  for (let i = 0; i < items.length; i++) {
    heartbeat(i);
    // ... process items[i]
  }
  return 'done';
}
```

→ `constraints.md` for what else follows from the pool model.

## Observability

A Cloud Run Worker emits the same traces and metrics as a Worker anywhere else — no Cloud Run-specific wiring, and none of Lambda's ADOT layer or collector configuration. Use the SDK's normal metrics export and OpenTelemetry tracing interceptors. → `observability.md`, and `docs/develop/typescript/platform/observability`.
