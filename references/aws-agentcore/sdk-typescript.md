# AgentCore — TypeScript SDK

<!-- Sources:
  docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx
  docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx
  docs/develop/typescript/workers/serverless-workers/cloud-run.mdx
  docs/develop/typescript/workers/interceptors.mdx
  aws: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-get-started-cli-typescript.html
  aws: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-http-protocol-contract.html
  aws: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-code-deploy-supported-runtimes.html
  npm: bedrock-agentcore@0.4.5 (package.json, dist/src/runtime/app.d.ts, dist/src/runtime/types.d.ts)
-->

Use this reference for the TypeScript entry point of an AgentCore Worker: dependencies, project layout, the versioned Worker, the Runtime handler, the idle and drain policy, and connection settings. For the shared deploy flow, permissions, lifecycle limits, versioning, observability and diagnostics, see `setup.md`, `iam.md`, `constraints.md`, `versioning.md`, `observability.md` and `diagnostics.md`. This file supplies the code for `setup.md` Step 2 and the `runtimeVersion` for Step 1; every other step is the same for every SDK.

**Coverage.** Temporal publishes AgentCore Worker code only for Python, and there is no TypeScript sample. The Temporal deploy guide defines what any entry point must do, language-neutrally. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:157-164 --> This guide implements that contract with the AWS TypeScript `bedrock-agentcore` package and the standard Temporal TypeScript SDK, following the structure of `sdk-python.md`. Statements marked **Not covered by the docs** have no published source: tell the user they are unverified, and confirm them in `agentcore dev` or the first deploy rather than presenting them as fact. To design the agent itself (Strands, Bedrock model calls, Workflow and Activity boundaries), link the user to [Build a durable agent on Amazon Bedrock AgentCore](https://docs.temporal.io/guides/durable-agent-on-agentcore); this skill does not cover it.

## Install and scaffold

Scaffold the TypeScript app with the AgentCore CLI rather than writing `agentcore.json` entries by hand. The CLI generates a CodeZip TypeScript agent under `app/<AGENT_NAME>/` with `main.ts`, `package.json` and `tsconfig.json`, and adds its Runtime to `agentcore/agentcore.json`: <!-- aws:runtime-get-started-cli-typescript.html#ts-create-agent -->

```bash
agentcore add agent --name <AGENT_NAME> --type create --build CodeZip \
  --language TypeScript --framework Strands --model-provider Bedrock --memory none
```

The generated starter is a Strands agent. Replace `main.ts` with the Worker entry point below, and remove starter files (`model/`, `mcp_client/`) that the application does not import. Then add the Temporal and AgentCore packages in `app/<AGENT_NAME>/`:

```bash
npm install bedrock-agentcore @temporalio/worker @temporalio/workflow @temporalio/activity \
  @aws-sdk/client-secrets-manager
```

- `bedrock-agentcore` requires Node.js 20 or later and is ESM-only (`"type": "module"`, `import` exports only). Keep `"type": "module"` in the app's `package.json`; AWS lists its absence as a TypeScript build failure cause. <!-- npm:bedrock-agentcore@0.4.5 package.json; aws:runtime-get-started-cli-typescript.html#ts-common-issues -->
- Set `runtimeVersion` to `NODE_22`, the only Node.js identifier AgentCore lists, and use Node.js 22 locally. <!-- aws:runtime-code-deploy-supported-runtimes.html#concept-supported-runtimes-node; aws:runtime-get-started-cli-typescript.html#ts-prerequisites -->
- The `codeLocation` directory must contain the entry point, every local module it imports, and the dependency manifest. For this layout, `codeLocation` is `app/<AGENT_NAME>`. Keep the `entrypoint` value the CLI generated. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:135-155 -->

```text
project-root/
├── agentcore/
│   ├── agentcore.json
│   ├── aws-targets.json
│   └── cdk/
└── app/<AGENT_NAME>/
    ├── main.ts          # Runtime handler and Worker
    ├── workflows.ts
    ├── activities.ts
    ├── package.json
    └── tsconfig.json
```

Add the Temporal settings to this Runtime's `envVars` and a named endpoint, exactly as `setup.md` Step 1 describes.

## Versioned Worker

Serverless Workers require Worker Versioning. Pass `workerDeploymentOptions` to `Worker.create()` with the deployment name and Build ID from the Runtime environment: <!-- docs/develop/typescript/workers/serverless-workers/cloud-run.mdx:39-62; docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:157-160 -->

```ts
const worker = await Worker.create({
  connection,
  namespace: requiredEnv('TEMPORAL_NAMESPACE'),
  taskQueue: TASK_QUEUE,
  workflowsPath: fileURLToPath(new URL('./workflows.js', import.meta.url)),
  activities,
  interceptors: { activity: [() => ({ inbound: tracker.interceptor() })] },
  workerDeploymentOptions: {
    version: { deploymentName: DEPLOYMENT_NAME, buildId: BUILD_ID },
    useWorkerVersioning: true,
    defaultVersioningBehavior: 'PINNED',
  },
  shutdownGraceTime: DRAIN,
});
```

`DEPLOYMENT_NAME` and `BUILD_ID` come from `TEMPORAL_DEPLOYMENT_NAME` and `TEMPORAL_BUILD_ID`. They must match the Worker Deployment Version created in `setup.md` Step 5 exactly. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:352-355 --> A new Build ID needs a new Runtime version and a new named endpoint. → `versioning.md`.

`workflowsPath` uses `import.meta.url` because the package is an ES module and `require.resolve` is unavailable. **Not covered by the docs:** whether the AgentCore CLI's esbuild step keeps `workflows.js` as a separate file that the Worker can bundle at startup. AWS states only that the CLI compiles TypeScript and bundles with esbuild. <!-- aws:runtime-get-started-cli-typescript.html#ts-deploy-runtime, #ts-common-issues --> If the Worker fails at startup because it cannot find or bundle the Workflows, prebuild a Workflow bundle with `bundleWorkflowCode()` from `@temporalio/worker`, ship it inside `codeLocation`, and pass it as `workflowBundle` instead of `workflowsPath`.

## Versioning behavior

Every Workflow needs `'PINNED'` or `'AUTO_UPGRADE'`. `defaultVersioningBehavior` covers every Workflow on the Worker. To set it per Workflow, use `setWorkflowOptions()` from `@temporalio/workflow`: <!-- docs/develop/typescript/workers/serverless-workers/cloud-run.mdx:70-79 -->

```ts
import { setWorkflowOptions } from '@temporalio/workflow';

setWorkflowOptions({ versioningBehavior: 'PINNED' }, myWorkflow);
export async function myWorkflow(prompt: string): Promise<string> {
  // ...
}
```

## Runtime handler

`BedrockAgentCoreApp` from `bedrock-agentcore/runtime` serves the AgentCore Runtime HTTP contract: `POST /invocations` and `GET /ping` on `0.0.0.0:8080`. `app.run()` starts the server. <!-- npm:bedrock-agentcore@0.4.5 dist/src/runtime/app.d.ts:3-8,42-50; aws:runtime-http-protocol-contract.html#container-requirements-http, #path-requirements-http --> Do not write these routes by hand.

The `/invocations` payload is not Workflow input. Temporal invokes the endpoint only to add Worker capacity; applications start Workflows through a Temporal Client as usual. <!-- docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx:34-37 --> The handler must therefore:

1. Register an AgentCore async task with `app.addAsyncTask(...)`. While a task is registered, `/ping` reports `HealthyBusy` and AgentCore keeps the session alive. <!-- npm:bedrock-agentcore@0.4.5 dist/src/runtime/app.d.ts:51-65; aws:runtime-http-protocol-contract.html#ping-endpoint -->
2. Start the Worker without awaiting it and return an acknowledgment immediately. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:162-163 -->
3. Keep a module-level reference to the running Worker so a second invocation in the same session does not start a duplicate.
4. Call `app.completeAsyncTask(taskId)` in a `finally`, whether the Worker drained or failed. Without it the session stays `HealthyBusy` until `maxLifetime`. → `diagnostics.md`.

`main.ts`:

```ts
import { fileURLToPath } from 'node:url';
import { BedrockAgentCoreApp } from 'bedrock-agentcore/runtime';
import { NativeConnection, Worker } from '@temporalio/worker';
import { GetSecretValueCommand, SecretsManagerClient } from '@aws-sdk/client-secrets-manager';
import * as activities from './activities.js';

function requiredEnv(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`${name} must be set`);
  return value;
}

const DEPLOYMENT_NAME = requiredEnv('TEMPORAL_DEPLOYMENT_NAME');
const BUILD_ID = requiredEnv('TEMPORAL_BUILD_ID');
const TASK_QUEUE = requiredEnv('TEMPORAL_TASK_QUEUE');

// ActivityTracker, DEBOUNCE_MS and DRAIN: see "Stop and drain the Worker".

// Read the Temporal API key from Secrets Manager; never log it.
async function loadApiKey(): Promise<string | undefined> {
  const secretArn = process.env.TEMPORAL_API_KEY_SECRET_ARN;
  if (!secretArn) return undefined;
  const secrets = new SecretsManagerClient({});
  const { SecretString } = await secrets.send(new GetSecretValueCommand({ SecretId: secretArn }));
  return SecretString;
}

async function runWorker(): Promise<void> {
  const apiKey = await loadApiKey();
  const connection = await NativeConnection.connect({
    address: requiredEnv('TEMPORAL_ADDRESS'),
    apiKey,
    tls: Boolean(apiKey),
  });
  try {
    const tracker = new ActivityTracker();
    console.info(`polling ${TASK_QUEUE} as ${DEPLOYMENT_NAME}/${BUILD_ID}`);
    const worker = await Worker.create({ /* the versioned Worker above */ });
    const running = worker.run();
    try {
      await Promise.race([tracker.waitUntilIdle(DEBOUNCE_MS), running]);
    } finally {
      if (worker.getState() === 'RUNNING') worker.shutdown();
      await running;
    }
    console.info(`worker idle for ${DEBOUNCE_MS / 1000}s; drained`);
  } finally {
    await connection.close();
  }
}

let current: Promise<void> | undefined;

const app: BedrockAgentCoreApp = new BedrockAgentCoreApp({
  invocationHandler: {
    // Start the Worker and acknowledge. The payload is unused.
    process: async () => {
      if (current) {
        console.info(`worker already polling ${TASK_QUEUE}`);
        return { message: 'worker already polling', task_queue: TASK_QUEUE };
      }
      const taskId = app.addAsyncTask('temporal-worker');
      current = runWorker()
        // Nothing awaits this promise, so an error would otherwise be lost.
        .catch((err) => console.error('worker failed in async task', err))
        .finally(() => {
          // Without this the session stays HealthyBusy until maxLifetime.
          app.completeAsyncTask(taskId);
          current = undefined;
        });
      return { message: 'worker starting', task_queue: TASK_QUEUE };
    },
  },
});

app.run();
```

**Not covered by the docs:** this handler is a translation of the Python sample's structure. Temporal has not published or tested it on AgentCore. The `bedrock-agentcore` API names above are taken from the package's type definitions. Verify both in the end-to-end run.

## Stop and drain the Worker

AgentCore cannot tell whether a polling Worker has work, and the async task keeps the Runtime busy, so the handler must decide when the Worker is idle and drain it. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:164; docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx:152-155 --> `worker.shutdown()` stops polling and gives in-flight Activities up to `shutdownGraceTime` before cancellation.

The tracker below follows the Python sample's policy: it is an Activity inbound interceptor that counts running Activities, and `waitUntilIdle` returns once no Activity has started or finished for the debounce period and none is running. Activity interceptors are registered as factory functions in `WorkerOptions.interceptors.activity`. <!-- docs/develop/typescript/workers/interceptors.mdx:118-121 -->

```ts
import type { ActivityInboundCallsInterceptor } from '@temporalio/worker';

// How long the Worker keeps polling after it goes idle.
const DEBOUNCE_MS = Number(process.env.AGENTCORE_DEBOUNCE_SECONDS ?? '60') * 1000;
// How long the drain waits for in-flight Activities.
const DRAIN = '120s';

class ActivityTracker {
  private inflight = 0;
  private lastChange = Date.now();

  interceptor(): ActivityInboundCallsInterceptor {
    return {
      execute: async (input, next) => {
        this.inflight++;
        this.lastChange = Date.now();
        try {
          return await next(input);
        } finally {
          this.inflight--;
          this.lastChange = Date.now();
        }
      },
    };
  }

  async waitUntilIdle(debounceMs: number): Promise<void> {
    for (;;) {
      const quietFor = Date.now() - this.lastChange;
      if (this.inflight === 0 && quietFor >= debounceMs) return;
      await new Promise((resolve) => setTimeout(resolve, Math.max(debounceMs - quietFor, 1000)));
    }
  }
}
```

A long-running Activity keeps the count above zero, so the idle policy does not interrupt it. `AGENTCORE_DEBOUNCE_SECONDS` (set in the Runtime's `envVars`) and `DRAIN` are configured values, not requirements. Choose both for the workload and keep them inside `maxLifetime`. → `constraints.md`.

The TypeScript SDK's default Runtime also shuts the Worker down on `SIGTERM`, so a Worker still running when AgentCore stops the process gets the same drain. If the application installs a custom Runtime, keep `SIGTERM` in its `shutdownSignals`. **Not covered by the docs:** whether AgentCore sends `SIGTERM` before ending compute.

## Connection configuration

Set `TEMPORAL_ADDRESS`, `TEMPORAL_NAMESPACE`, `TEMPORAL_TASK_QUEUE`, `TEMPORAL_DEPLOYMENT_NAME` and `TEMPORAL_BUILD_ID` in the Runtime's `envVars` (`setup.md` Step 1). For Temporal Cloud, also set `TEMPORAL_API_KEY_SECRET_ARN`. The handler above reads the key from that secret each time it starts a Worker, and turns TLS on only when a key is set. That covers Temporal Cloud with an API key and a self-hosted Service without TLS. The AWS SDK client authenticates as the Runtime execution role. **Not covered by the docs:** the Secrets Manager call in TypeScript is a translation of the Python handler, so verify it in the end-to-end run.

To use the shared environment-configuration format and profiles instead (including mTLS settings), call `loadClientConnectConfig()` from `@temporalio/envconfig` and pass its `connectionOptions` and `namespace` to `NativeConnection.connect()` and `Worker.create()`. <!-- docs/develop/typescript/workers/serverless-workers/cloud-run.mdx:88 -->

The API key and any TLS material live in AWS Secrets Manager, never in `agentcore.json` or a Runtime env var. The user stores the key from their own terminal, and the execution role gets `secretsmanager:GetSecretValue` on that one secret ARN. → `setup.md`, [Store the Temporal API key](setup.md#store-the-temporal-api-key). Rotation needs no redeploy, because each new Worker reads the current version. → `iam.md`, [Temporal API key lifecycle](iam.md#temporal-api-key-lifecycle). Never log the value. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:128-131 -->

## Package and deploy

There is no separate packaging step. `agentcore validate` and `agentcore deploy --target <TARGET> -y` compile the TypeScript, bundle it with esbuild, package it as a CodeZip archive, and deploy it. → `setup.md` Step 3. Do not build a container image. <!-- aws:runtime-get-started-cli-typescript.html#ts-deploy-runtime, #ts-common-issues -->

Before deploying, check that:

- `npm run build` succeeds in `app/<AGENT_NAME>/`;
- `agentcore dev` starts the server locally on port 8080, and an invocation returns `worker starting` and starts the Worker against the configured Namespace; <!-- aws:runtime-get-started-cli-typescript.html#ts-test-locally -->
- the zip stays within 250 MB zipped and 750 MB unzipped; and
- any native npm modules are built for Linux arm64. AgentCore Runtime supports only arm64. <!-- aws:runtime-get-started-cli-typescript.html#ts-common-issues -->

**Not covered by the docs:** the Temporal TypeScript Worker depends on `@temporalio/core-bridge`, which ships a native module. Whether `agentcore deploy` packages the Linux arm64 build of it is unverified. If the deployed Worker fails to load the core bridge, follow the AWS native-module guidance above and record the fix.

## Keep Activities safe across Worker termination

AgentCore can end the compute that runs a Worker, and an Activity running at that moment is retried. Set Activity timeouts, and Heartbeat long-running Activities so a retry resumes from its last recorded progress: <!-- docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx:157-159 -->

```ts
import { activityInfo, heartbeat } from '@temporalio/activity';

export async function myActivity(items: string[]): Promise<string> {
  const start = (activityInfo().heartbeatDetails as number | undefined) ?? 0;
  for (let i = start; i < items.length; i++) {
    // ... process items[i]; each step must be safe to repeat
    heartbeat(i + 1);
  }
  return 'done';
}
```

→ `constraints.md`.

## Observability

There is no AgentCore-specific TypeScript helper. Configure metrics and tracing with the TypeScript SDK's standard observability options, as for any long-running Worker. Worker lifecycle logs written to stdout are read with `agentcore logs --runtime <RUNTIME_NAME>`. → `observability.md`. **Not covered by the docs:** Temporal does not publish TypeScript-specific AgentCore observability guidance.
