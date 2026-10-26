# AgentCore — TypeScript SDK

Use this reference for the TypeScript entry point of an AgentCore Worker: dependencies, project layout, the versioned Worker, the Runtime handler, the idle and drain policy, and connection settings. For the shared deploy flow, permissions, lifecycle limits, versioning, observability and diagnostics, see `setup.md`, `iam.md`, `constraints.md`, `versioning.md`, `observability.md` and `diagnostics.md`. This file supplies the code for `setup.md` Step 2 and the `runtimeVersion` for Step 1; every other step is the same for every SDK.

**Coverage.** Temporal publishes AgentCore Worker code only for Python, and there is no TypeScript sample. The Temporal deploy guide defines what any entry point must do, language-neutrally. This guide implements that contract with the AWS TypeScript `bedrock-agentcore` package and the standard Temporal TypeScript SDK, following the structure of `sdk-python.md`. Statements marked **Not covered by the docs** have no published source: tell the user they are unverified, and confirm them in `agentcore dev` or the first deploy rather than presenting them as fact. To design the agent itself (Strands, Bedrock model calls, Workflow and Activity boundaries), link the user to [Build a durable agent on Amazon Bedrock AgentCore](https://docs.temporal.io/guides/durable-agent-on-agentcore); this skill does not cover it.

## Install and scaffold

Scaffold the TypeScript app with the AgentCore CLI rather than writing `agentcore.json` entries by hand. Use the **Container** build. The CLI generates a TypeScript agent under `app/<AGENT_NAME>/` with `main.ts`, `package.json`, `tsconfig.json`, a `Dockerfile` and a `.dockerignore`, and adds its Runtime to `agentcore/agentcore.json`:

```bash
agentcore add agent --name <AGENT_NAME> --type create --build Container \
  --language TypeScript --framework Strands --model-provider Bedrock --memory none
```

**Do not use the CodeZip build for TypeScript.** The CLI's Node CodeZip packager runs esbuild on `main.ts` alone, writes a single `main.js`, and copies no `node_modules` except a fixed list of `bedrock-agentcore` helpers. It has no option to take a prebuilt zip. `@temporalio/worker` cannot be bundled that way: it bundles Workflow code at startup with webpack and swc, and it loads the native `@temporalio/core-bridge` module. `agentcore deploy` then fails during CDK synthesis with `No loader is configured for ".node" files`. The Container build installs `node_modules` in the image instead. The CLI builds that image in AWS CodeBuild, so no local Docker is needed. → `setup.md` Prerequisites.

The generated starter is a Strands agent. Replace `main.ts` with the Worker entry point below, and remove starter files (`model/`, `mcp_client/`) that the application does not import. Remove the starter's Strands, MCP and `zod` dependencies from `package.json` if nothing imports them. Then add the Temporal and AgentCore packages in `app/<AGENT_NAME>/`:

```bash
npm install bedrock-agentcore@latest @temporalio/worker @temporalio/workflow @temporalio/activity \
  @aws-sdk/client-secrets-manager
```

The scaffold pins an older `bedrock-agentcore` range (`^0.3.0` with CLI 0.28.1), and a plain `npm install bedrock-agentcore` keeps it. `@latest` moves it to the current release.

- `bedrock-agentcore` requires Node.js 20 or later and is ESM-only (`"type": "module"`, `import` exports only). Keep `"type": "module"` in the app's `package.json`; AWS lists its absence as a TypeScript build failure cause.
- Keep the generated `runtimeVersion` (`NODE_22`, the only Node.js identifier AgentCore lists) and use Node.js 22 locally; the Dockerfile's base image sets the Node.js version the Runtime actually runs.
- The `codeLocation` directory (`app/<AGENT_NAME>` in this layout) is the image's build context. It must contain the `Dockerfile`, the entry point, every local module it imports, `package.json` and `package-lock.json`. Put the secret's policy file here too, because `additionalPolicies` paths resolve relative to `codeLocation` (`setup.md`, Store the Temporal API key).

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

Serverless Workers require Worker Versioning. Pass `workerDeploymentOptions` to `Worker.create()` with the deployment name and Build ID from the Runtime environment:

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

`DEPLOYMENT_NAME` and `BUILD_ID` come from `TEMPORAL_DEPLOYMENT_NAME` and `TEMPORAL_BUILD_ID`. They must match the Worker Deployment Version created in `setup.md` Step 5 exactly. A new Build ID needs a new Runtime version and a new named endpoint. → `versioning.md`.

`workflowsPath` uses `import.meta.url` because the package is an ES module and `require.resolve` is unavailable. It points at the compiled `workflows.js` next to the compiled `main.js`, so the image must run the `tsc` output, not `main.ts` (see [Package and deploy](#package-and-deploy)).

## Versioning behavior

Every Workflow needs `'PINNED'` or `'AUTO_UPGRADE'`. `defaultVersioningBehavior` covers every Workflow on the Worker. To set it per Workflow, use `setWorkflowOptions()` from `@temporalio/workflow`:

```ts
import { setWorkflowOptions } from '@temporalio/workflow';

setWorkflowOptions({ versioningBehavior: 'PINNED' }, myWorkflow);
export async function myWorkflow(prompt: string): Promise<string> {
  // ...
}
```

## Runtime handler

`BedrockAgentCoreApp` from `bedrock-agentcore/runtime` serves the AgentCore Runtime HTTP contract: `POST /invocations` and `GET /ping` on `0.0.0.0:8080`. `app.run()` starts the server. Do not write these routes by hand.

The `/invocations` payload is not Workflow input. Temporal invokes the endpoint only to add Worker capacity; applications start Workflows through a Temporal Client as usual. The handler must therefore:

1. Register an AgentCore async task with `app.addAsyncTask(...)`. While a task is registered, `/ping` reports `HealthyBusy` and AgentCore keeps the session alive.
2. Start the Worker without awaiting it and return an acknowledgment immediately.
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

AgentCore cannot tell whether a polling Worker has work, and the async task keeps the Runtime busy, so the handler must decide when the Worker is idle and drain it. `worker.shutdown()` stops polling and gives in-flight Activities up to `shutdownGraceTime` before cancellation.

The tracker below follows the Python sample's policy: it is an Activity inbound interceptor that counts running Activities, and `waitUntilIdle` returns once no Activity has started or finished for the debounce period and none is running. Activity interceptors are registered as factory functions in `WorkerOptions.interceptors.activity`.

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

To use the shared environment-configuration format and profiles instead (including mTLS settings), call `loadClientConnectConfig()` from `@temporalio/envconfig` and pass its `connectionOptions` and `namespace` to `NativeConnection.connect()` and `Worker.create()`.

The API key and any TLS material live in AWS Secrets Manager, never in `agentcore.json` or a Runtime env var. The user stores the key from their own terminal, and the execution role gets `secretsmanager:GetSecretValue` on that one secret ARN. → `setup.md`, [Store the Temporal API key](setup.md#store-the-temporal-api-key). Rotation needs no redeploy, because each new Worker reads the current version. → `iam.md`, [Temporal API key lifecycle](iam.md#temporal-api-key-lifecycle). Never log the value.

## Package and deploy

There is no local packaging step. `agentcore validate` and `agentcore deploy --target <TARGET> -y` upload `codeLocation`, build the image from its `Dockerfile` in AWS CodeBuild, push it to the ECR repository the CLI creates, and deploy the Runtime. → `setup.md` Step 3.

The generated `Dockerfile` runs `npx tsx main.ts`. Change it to compile with `tsc` and run `dist/main.js`, so that `workflowsPath` finds `dist/workflows.js`, and install `ca-certificates`: the generated `node:22-slim` base has no CA bundle, and the TypeScript SDK's Rust core then fails TLS to Temporal Cloud with `NativeCertsNotFound`. Keep the generated base image, user and port:

```dockerfile
FROM public.ecr.aws/docker/library/node:22-slim

WORKDIR /app
ENV DOCKER_CONTAINER=1

# The Temporal core verifies TLS against the system trust store; the slim image has none.
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates \
    && rm -rf /var/lib/apt/lists/*

RUN userdel -r node 2>/dev/null || true
RUN useradd -m -u 1000 bedrock_agentcore

COPY package.json package-lock.json* ./
RUN npm ci

COPY . .
# Compile so './workflows.js' exists next to main.js, then drop dev dependencies.
RUN npm run build && npm prune --omit=dev && chown -R bedrock_agentcore:bedrock_agentcore /app

ENV NODE_ENV=production
USER bedrock_agentcore

# AgentCore Runtime HTTP contract
EXPOSE 8080

CMD ["node", "dist/main.js"]
```

Keep the generated `.dockerignore`, which excludes `node_modules/` and `dist/`, so the image installs its own Linux arm64 dependencies, including the native `@temporalio/core-bridge` module.

Before deploying, check that:

- `npm run build` succeeds in `app/<AGENT_NAME>/`;
- `agentcore validate` reports `Valid`; and
- the Runtime in `agentcore.json` has `"build": "Container"`.

## Keep Activities safe across Worker termination

AgentCore can end the compute that runs a Worker, and an Activity running at that moment is retried. Set Activity timeouts, and Heartbeat long-running Activities so a retry resumes from its last recorded progress:

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

## Logging and diagnostic signatures

Write Worker lifecycle logs to stdout and stderr with `console`, as the handler above does, and read them with the endpoint's log group (`aws logs tail /aws/bedrock-agentcore/runtimes/<RUNTIME_ID>-<ENDPOINT_NAME>`). The handler logs the four points `observability.md` requires, with the same messages as the Python sample, so the signatures match across SDKs. Never log the API key or the secret value.

| Log signature | Meaning / action |
|---|---|
| `polling <TASK_QUEUE> as <DEPLOYMENT_NAME>/<BUILD_ID>` | A Worker started in this session. If the deployment name or Build ID differs from the Worker Deployment Version, the Runtime's env vars are wrong. → `diagnostics.md`, [Invoked, but no Worker registers](diagnostics.md#invoked-but-no-worker-registers). |
| `worker already polling <TASK_QUEUE>` | Another invocation reached a session whose Worker is still running. No second Worker started. Expected. |
| `worker idle for <N>s; drained` | The idle policy fired and the Worker drained. The session emits no further Worker logs; this is not a failure. → `observability.md`. |
| `worker failed in async task`, followed by an error | `runWorker` rejected, and the async task was released. Diagnose from the error that follows. |
| `worker failed in async task TransportError: tonic::transport::Error(Transport, NativeCertsNotFound)` | The image has no CA certificates, so the Worker cannot open a TLS connection to Temporal. Install `ca-certificates` in the `Dockerfile` ([Package and deploy](#package-and-deploy)) and redeploy. |
| `<NAME> must be set` at startup | A required variable is missing from the Runtime's `envVars`. Set it (`setup.md` Step 1) and redeploy. |

These are the messages this guide's handler writes; like the handler, they are **not covered by the docs**. Temporal documents no AgentCore-specific TypeScript log signatures. For anything else, follow `diagnostics.md`.

## Observability

There is no AgentCore-specific TypeScript helper. Configure metrics and tracing with the TypeScript SDK's standard observability options, as for any long-running Worker. Worker lifecycle logs written to stdout are read with the endpoint's log group (`aws logs tail /aws/bedrock-agentcore/runtimes/<RUNTIME_ID>-<ENDPOINT_NAME>`). → `observability.md`. **Not covered by the docs:** Temporal does not publish TypeScript-specific AgentCore observability guidance.
