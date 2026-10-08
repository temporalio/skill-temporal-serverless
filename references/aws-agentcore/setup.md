# AgentCore — Setup (happy path)

<!-- Sources:
  docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx
  docs/guides/durable-agent-on-agentcore.mdx
  docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx
  aws: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/agent-runtime-versioning.html
  samples-python@d67795f: bedrock_agentcore/strands_agent/
-->

This page covers the full path. You configure the Runtime in `agentcore/agentcore.json`, add the Worker entry point, deploy with the AgentCore CLI using the **CodeZip** build, grant Temporal an invocation role, register a Worker Deployment Version against a named endpoint, set it current, and verify. Related references: `iam.md` for the three identities, `constraints.md` for lifecycle limits, `versioning.md` for new builds and rollback, `diagnostics.md` when something fails, and `self-hosted.md` for a self-hosted Service, which you complete first.

This skill deploys and operates the Worker. To design the agent itself (Workflow and Activity boundaries, Strands, Bedrock model calls), link the user to [Build a durable agent on Amazon Bedrock AgentCore](https://docs.temporal.io/guides/durable-agent-on-agentcore). Do not reproduce that guide. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:30-34 -->

## Prerequisites

<!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:39-53 -->

- **Temporal:** either a Temporal Cloud account with an **AWS-hosted Namespace that has AgentCore Pre-release access**, or a self-hosted Temporal Service v1.32.0+ with `self-hosted.md` completed.
- [Temporal CLI v1.8.3](https://github.com/temporalio/cli/releases/tag/v1.8.3) or later, configured for the Namespace.
- An AWS account in an [AgentCore-supported Region](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/agentcore-regions.html). The AWS CLI must be configured for that account, and the operator needs the permissions listed in `iam.md`.
- Node.js 20+, the [AgentCore CLI](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/agentcore-get-started-cli.html) (`npm install -g @aws/agentcore`), and the AWS CDK, bootstrapped in the target account and Region. <!-- samples-python/bedrock_agentcore/strands_agent/README.md:26-30 -->
- For Python: `uv` on the operator's machine. The CDK build uses it to install dependencies for the Runtime's platform, and without it the build fails with `uv install failed ... with exit code null`. <!-- samples-python/bedrock_agentcore/strands_agent/bin/create-runtime.sh:16-26 -->
- **No Docker.** With CodeZip, the CLI uploads a zip of the code directory and AgentCore runs it on a managed runtime. The CLI also handles AgentCore's ARM64 (Graviton) requirement. Do not build container images or create ECR repositories for this path. <!-- samples-python/bedrock_agentcore/strands_agent/README.md:40-42; docs/guides/durable-agent-on-agentcore.mdx:112-114 -->

### Select the Namespace

For Temporal Cloud, list the user's Namespaces, for example with `tcld namespace list`. Then show **every** Namespace that is hosted on AWS, not just the first match, and let the user choose. Then ask the user to confirm that the chosen Namespace has AgentCore Pre-release access. If it does not, stop and point them to a [support ticket](https://docs.temporal.io/evaluate/cloud/support#support-ticket) or their account team. Mention self-hosting (`self-hosted.md`) as the alternative. <!-- docs/encyclopedia/workers/serverless-workers/serverless-workers-agentcore.mdx:19-24 -->

Use the gRPC endpoint that Temporal Cloud shows for the Namespace as `TEMPORAL_ADDRESS`. Do not construct it from the Namespace name.

## Step 1: Configure the Runtime

`agentcore/agentcore.json` is the AgentCore CLI project file. Its `runtimes` array defines the Runtimes that the CLI deploys, and `agentcore/aws-targets.json` names the target account and Region. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:57-59; docs/guides/durable-agent-on-agentcore.mdx:187-199 -->

**No project yet:** `agentcore create` generates one, including the `agentcore/cdk/` scaffold that `agentcore deploy` synthesizes. The sample runs it once to obtain only the CDK scaffold, and keeps its own checked-in `agentcore.json`: <!-- samples-python/bedrock_agentcore/strands_agent/bin/create-runtime.sh:28-40; samples-python/bedrock_agentcore/strands_agent/README.md:123-125 -->

```bash
agentcore create --project-name <PROJECT_NAME> --no-agent \
  --output-dir <TMP_DIR> --skip-git --skip-python-setup
mv <TMP_DIR>/<PROJECT_NAME>/agentcore/cdk agentcore/cdk
```

The Runtime object needs `"build": "CodeZip"`, an `entrypoint`, a `codeLocation`, and a **named endpoint** pinned to a version. Never rely on `DEFAULT` (see `versioning.md`):

```json
{
  "name": "<RUNTIME_NAME>",
  "build": "CodeZip",
  "entrypoint": "<ENTRY_POINT_FILE>",
  "codeLocation": ".",
  "runtimeVersion": "PYTHON_3_12",
  "networkMode": "PUBLIC",
  "protocol": "HTTP",
  "authorizerType": "AWS_IAM",
  "endpoints": {
    "temporal": { "version": 1, "description": "Invoked by Temporal Serverless Workers" }
  }
}
```

<!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:67-89 -->

`runtimeVersion` above is the Python value from the sample. Take the value for other languages from the selected `sdk-<language>.md`. Add `lifecycleConfiguration` (`idleRuntimeSessionTimeout`, `maxLifetime`) only after you read `constraints.md`. The sample sets the AgentCore defaults explicitly. <!-- samples-python/bedrock_agentcore/strands_agent/agentcore/agentcore.json:54-57 -->

Add the Temporal settings to the Runtime's `envVars`:

| Variable | Value |
|---|---|
| `TEMPORAL_ADDRESS` | Namespace gRPC endpoint (Cloud) or the frontend address (self-hosted) |
| `TEMPORAL_NAMESPACE` | Namespace |
| `TEMPORAL_API_KEY` | Temporal Cloud API key. See the secret rule below. |
| `TEMPORAL_TASK_QUEUE` | Task Queue the application uses |
| `TEMPORAL_DEPLOYMENT_NAME` | Worker Deployment name. It must match Step 5. |
| `TEMPORAL_BUILD_ID` | Build ID. It must match Step 5. |

<!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:91-127 -->

**Do not commit a populated API key.** In production, store it in AWS Secrets Manager, grant the Runtime **execution** role permission to read it, and load it in the entry point. The key is the user's to handle: never print it or echo it through the agent's shell. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:129-131 -->

If the Runtime uses a VPC instead of `PUBLIC`, configure outbound access from the VPC to the Temporal Service. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:287-288 -->

## Step 2: Add the Worker entry point

For CodeZip, AgentCore packages the `codeLocation` directory, and `entrypoint` is relative to it. That directory must contain the entry point, every local module it imports, and the language's dependency manifest. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:135-155 -->

The entry point must: <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:157-164 -->

- connect a Client and build a standard long-running Worker with Worker Versioning, using the deployment name and Build ID from the environment;
- serve the AgentCore Runtime HTTP contract (`/invocations`, `/ping`);
- start the Worker as background work on `/invocations` and acknowledge immediately; and
- stop polling and drain when its idle policy decides to release the Runtime.

The language-specific code is in the selected `sdk-<language>.md`.

## Step 3: Deploy the Runtime

From the project directory, run the following, using the `name` of the target in `aws-targets.json`:

```bash
agentcore validate
agentcore deploy --target <TARGET> -y
```

The CLI packages the code, deploys the Runtime, creates its execution role, and creates the named endpoint. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:244-254; samples-python/bedrock_agentcore/strands_agent/README.md:107-110 --> Re-running the same commands deploys changes, which creates a new Runtime version. Read `versioning.md` before you redeploy a Runtime that serves a live Worker Deployment Version.

Record both ARNs: <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:256-285 -->

```bash
agentcore status --runtime <RUNTIME_NAME> --json
agentcore status --type runtime-endpoint --json
```

- **Runtime ARN** ends in `/runtime/<runtime-id>`. It is used only for IAM, with a trailing `*`.
- **Endpoint ARN** ends in `/runtime/<runtime-id>/runtime-endpoint/<endpoint-name>`. It is used only for `--aws-agentcore-endpoint-arn`. **Never pass the Runtime ARN there.**

Do not continue until both ARNs are printed and the endpoint state is `READY`. <!-- docs/guides/durable-agent-on-agentcore.mdx:256; aws:agent-runtime-versioning.html#endpoint-lifecycle -->

## Step 4: Grant Temporal permission to invoke the Runtime

For Temporal Cloud, deploy the Cloud invocation-role template with the Runtime ARN plus `*` and an External ID the user chooses, then read the stack's `RoleARN` output. → `iam.md`. For a self-hosted Service, use the role from `self-hosted.md` step 4 and skip this step. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:290-348 -->

## Step 5: Create the Worker Deployment Version

The deployment name and Build ID must match `TEMPORAL_DEPLOYMENT_NAME` and `TEMPORAL_BUILD_ID`. If they differ, Temporal can start compute that never registers as the version waiting for work. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:352-355; docs/guides/durable-agent-on-agentcore.mdx:211-214 -->

**Temporal Cloud UI:** go to **Workers → Create Worker Deployment** and set Name, Build ID, Compute Provider **Amazon Bedrock AgentCore Runtime**, Runtime endpoint ARN, IAM role ARN and External ID. A version created in the UI is current automatically. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:358-371 -->

**Temporal CLI** (required for self-hosted):

```bash
temporal worker deployment create \
  --namespace <TEMPORAL_NAMESPACE> \
  --name <DEPLOYMENT_NAME>

temporal worker deployment create-version \
  --namespace <TEMPORAL_NAMESPACE> \
  --deployment-name <DEPLOYMENT_NAME> \
  --build-id <BUILD_ID> \
  --aws-agentcore-endpoint-arn <RUNTIME_ENDPOINT_ARN> \
  --aws-agentcore-assume-role-arn <INVOCATION_ROLE_ARN> \
  --aws-agentcore-assume-role-external-id <EXTERNAL_ID>
```

<!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:376-396 -->

Creating the version makes Temporal invoke the Runtime and wait for the Worker to register. <!-- docs/guides/durable-agent-on-agentcore.mdx:350-352 --> On Cloud, use **Actions → Validate Connection** to check the role, the endpoint lookup and the invocation. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:401-403 --> Then confirm registration with the check in [`diagnostics.md`](diagnostics.md#start-here-did-the-expected-worker-register). Wait on that observable state, not on a fixed delay.

## Step 6: Set the version current

```bash
temporal worker deployment set-current-version \
  --namespace <TEMPORAL_NAMESPACE> \
  --deployment-name <DEPLOYMENT_NAME> \
  --build-id <BUILD_ID> \
  --yes
```

The command asks for confirmation because it changes which version receives new Tasks. `--yes` skips the prompt. Skip this step if the version was created in the Cloud UI. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:405-417 -->

## Step 7: Verify

Submit work to the Task Queue. When no Worker is polling, Temporal invokes the endpoint, the Runtime starts the Worker, and the Worker processes Tasks. Confirm all three: <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:419-430 -->

- **Temporal UI:** the Worker Deployment Version shows a Worker polling the Task Queue. On Cloud, the connection also shows as valid.
- **AgentCore logs:** `agentcore logs --runtime <RUNTIME_NAME>` shows the Worker starting and processing Tasks.
- **Temporal CLI:** `temporal worker deployment describe --name <DEPLOYMENT_NAME>` shows the deployment and its current version.

The Workflow's Event History shows what ran, and the AgentCore logs show which Worker process ran it and when it drained. <!-- docs/guides/durable-agent-on-agentcore.mdx:384-391 -->

## Resource inventory

Record each resource as you create it: the AgentCore project directory and target, the Runtime name and ARN, the endpoint name and ARN, the CloudFormation stack and role ARN, the External ID, the Region, and the deployment name and Build ID. Hand the inventory to the user at the end. AgentCore teardown commands are not covered by this skill's sources, so before you delete any AgentCore resource, ask the user how they want to remove it. Never unset or delete a Temporal version while Pinned Workflows still need it (see `versioning.md`).
