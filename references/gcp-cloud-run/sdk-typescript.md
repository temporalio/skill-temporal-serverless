# TypeScript SDK on GCP Cloud Run

<!-- Source: docs/develop/typescript/workers/serverless-workers/cloud-run.mdx -->

Use this reference for TypeScript-specific Worker construction, versioning behavior, connection configuration, image packaging, and scale-in safety. For shared Cloud Run execution constraints, deployment lifecycle, permissions, versioning, observability, and diagnostics, see `constraints.md`, `setup.md`, `iam.md`, `versioning.md`, `observability.md`, and `diagnostics.md`.

## Install and scaffold

Initialize the project and pin the SDK and compiler versions validated for this guide:

```bash
npm init -y
npm install @temporalio/activity@1.24.0 @temporalio/client@1.24.0 @temporalio/worker@1.24.0 @temporalio/workflow@1.24.0
npm install --save-dev typescript@7.0.2 @types/node@22.20.4
npx tsc --init --rootDir src --outDir dist --module commonjs --target es2022 --esModuleInterop
npm pkg set scripts.build='tsc' scripts.start='node dist/worker.js'
```

## Inspect the versioning API before generating code

```bash
npm ls @temporalio/worker
grep -rn "workerDeploymentOptions\|WorkerDeploymentOptions" node_modules/@temporalio/worker/lib/*.d.ts
```

## Versioned Worker

Pass `workerDeploymentOptions` to `Worker.create()`. The Worker reads its connection settings and Task Queue from the environment so one image runs against any Namespace:

```ts
import { NativeConnection, Worker } from '@temporalio/worker';

function requiredEnv(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`${name} must be set`);
  return value;
}

async function main(): Promise<void> {
  const deploymentName = requiredEnv('TEMPORAL_DEPLOYMENT_NAME');
  const buildId = requiredEnv('TEMPORAL_BUILD_ID');
  const taskQueue = requiredEnv('TEMPORAL_TASK_QUEUE');
  const connection = await NativeConnection.connect({
    address: requiredEnv('TEMPORAL_ADDRESS'),
    apiKey: requiredEnv('TEMPORAL_API_KEY'),
    tls: true,
  });

  const worker = await Worker.create({
    connection,
    namespace: requiredEnv('TEMPORAL_NAMESPACE'),
    taskQueue,
    workflowsPath: require.resolve('./workflows'),
    workerDeploymentOptions: {
      version: { deploymentName, buildId },
      useWorkerVersioning: true,
      defaultVersioningBehavior: 'PINNED',
    },
    shutdownGraceTime: '8s',
    shutdownForceTime: '9s',
  });

  console.info(`Worker started deployment=${deploymentName} build=${buildId} taskQueue=${taskQueue}`);
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

Use this multi-stage image and `.gcloudignore`:

```dockerfile
FROM node:22-slim AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY tsconfig.json ./
COPY src ./src
RUN npm run build

FROM node:22-slim
RUN apt-get update \
    && apt-get install -y --no-install-recommends ca-certificates \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
COPY --from=build /app/dist ./dist
CMD ["node", "dist/worker.js"]
```

```gitignore
.git
.gitignore
.idea
node_modules
dist
npm-debug.log
```

Cloud Run Worker Pools run this image as `linux/amd64`.

## Graceful shutdown on scale-in

The versioned Worker example uses `await worker.run()`. The TypeScript SDK Runtime registers `SIGINT`, `SIGTERM`, `SIGQUIT`, and `SIGUSR2` as shutdown signals by default, so Cloud Run's `SIGTERM` starts the Worker's normal shutdown without an application-level signal handler. `shutdownGraceTime` gives received Activities eight seconds before cancellation; `shutdownForceTime` prevents a non-cooperative Activity from keeping the process alive until Cloud Run kills it. If the application installs a custom Runtime, preserve `SIGTERM` in its `shutdownSignals`.

## Keep Activities safe across scale-in

Apply the Heartbeat-resume invariant from `constraints.md` in TypeScript:

```ts
import { proxyActivities } from '@temporalio/workflow';
import type * as activities from './activities';

const { myActivity } = proxyActivities<typeof activities>({
  startToCloseTimeout: '10m',
  heartbeatTimeout: '10s',
  retry: {
    initialInterval: '1s',
    maximumAttempts: 5,
  },
});

export async function myWorkflow(items: string[]): Promise<string> {
  return myActivity(items);
}
```

The Activity records the next item to process:

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
