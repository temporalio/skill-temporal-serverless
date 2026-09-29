# TypeScript SDK on GCP Cloud Run

<!-- Source: docs/develop/typescript/workers/serverless-workers/cloud-run.mdx -->

Use this reference for TypeScript-specific Worker construction, versioning behavior, connection configuration, image packaging, and scale-in safety. For shared Cloud Run execution constraints, deployment lifecycle, permissions, versioning, observability, and diagnostics, see `constraints.md`, `setup.md`, `iam.md`, `versioning.md`, `observability.md`, and `diagnostics.md`.

## Inspect the versioning API before generating code

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

Cloud Run needs no `workflowBundle` — `workflowsPath` is sufficient because the instance is long-lived and bundling only improves startup time.

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
- **Leave headroom for native memory.** The instance limit covers both V8 and Temporal's Rust core. Set `NODE_OPTIONS=--max-old-space-size=<MB>` only when measurements justify it. A pool defaults to 512 MiB per instance, so raise `--memory` if the Worker needs more.

→ `setup.md` Step 2, `diagnostics.md`.

## Graceful shutdown on scale-in

The versioned Worker example uses `await worker.run()`. The TypeScript SDK Runtime registers `SIGINT`, `SIGTERM`, `SIGQUIT`, and `SIGUSR2` as shutdown signals by default, so Cloud Run's `SIGTERM` starts the Worker's normal shutdown without an application-level signal handler. `shutdownGraceTime` gives received Activities eight seconds before cancellation; `shutdownForceTime` prevents a non-cooperative Activity from keeping the process alive until Cloud Run kills it. If the application installs a custom Runtime, preserve `SIGTERM` in its `shutdownSignals`.

## Keep Activities safe across scale-in

Apply the Heartbeat-resume invariant from `constraints.md` in TypeScript:

```ts
import { activityInfo, heartbeat } from '@temporalio/activity';

export async function myActivity(items: string[]): Promise<string> {
  const startIndex = (activityInfo().heartbeatDetails as number | undefined) ?? 0;

  for (let i = startIndex; i < items.length; i++) {
    // ... process items[i]
    heartbeat(i + 1);
  }
  return 'done';
}
```

→ `constraints.md` for what else follows from the pool model.

## Observability

For TypeScript SDK configuration, see `docs/develop/typescript/platform/observability`. For shared Cloud Run behavior and provider-specific scaling signals, see `observability.md`.
