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
| `TEMPORAL_API_KEY_SECRET_ARN` | ARN of the Secrets Manager secret that holds the Temporal Cloud API key (Temporal Cloud with an API key only). This is the ARN, never the key. |
| `TEMPORAL_TASK_QUEUE` | Task Queue the application uses |
| `TEMPORAL_DEPLOYMENT_NAME` | Worker Deployment name. It must match Step 5. |
| `TEMPORAL_BUILD_ID` | Build ID. It must match Step 5. |

<!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:91-127 -->

**The API key never goes in `agentcore.json`, an env var, or any command the agent runs.** It lives only in AWS Secrets Manager. The Runtime **execution** role reads it, and the entry point loads it before it connects. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:129-131 --> Set up the secret, the grant and the user's hand-off as described in [Store the Temporal API key](#store-the-temporal-api-key) before Step 3. A self-hosted Service without API-key auth needs no secret, so leave `TEMPORAL_API_KEY_SECRET_ARN` unset.

### Store the Temporal API key

Skip this for a self-hosted Service that does not use API keys. The full key lifecycle (source, rotation, revocation) is in [`iam.md`](iam.md#temporal-api-key-lifecycle).

1. **Create the secret, with no value.** First check whether it exists: `aws secretsmanager describe-secret --secret-id <SECRET_NAME> --region <AWS_REGION> --query ARN --output text`. If it returns `ResourceNotFoundException`, create it and record it in the inventory as *created*. If it exists, reuse it only when the user confirms it is theirs to use for this Worker, and record it as *reused*. A secret can exist without a version; the hand-off adds the first one. <!-- aws-cli 2.35.15: secretsmanager describe-secret, create-secret, put-secret-value help -->

   ```bash
   aws secretsmanager create-secret --name <SECRET_NAME> \
     --description "Temporal API key for <DEPLOYMENT_NAME>" \
     --region <AWS_REGION> --query ARN --output text
   ```

   Put the printed ARN in `TEMPORAL_API_KEY_SECRET_ARN`.

2. **Grant the execution role read access to this one secret.** Add a policy file next to `agentcore/`, as the sample does for Code Interpreter, and list it in the Runtime's `additionalPolicies`. `agentcore deploy` attaches it to the execution role. Never grant it to the invocation role, and never use `"Resource": "*"`. <!-- samples-python/bedrock_agentcore/strands_agent/agentcore/agentcore.json:64-66; samples-python/bedrock_agentcore/strands_agent/README.md:110-116 -->

   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "ReadTemporalApiKey",
         "Effect": "Allow",
         "Action": "secretsmanager:GetSecretValue",
         "Resource": "<SECRET_ARN>"
       }
     ]
   }
   ```

   ```json
   "additionalPolicies": ["temporal-api-key-secret-policy.json"]
   ```

3. **Hand off the key.** The user runs this once, **in their own interactive terminal**, after the secret exists. It reads the key once and writes it as the secret's first version. Do not run it through the agent's shell: `read -s` needs a terminal, and the key would pass through the agent's session. Steps 2–4 need no key and can continue meanwhile. **Step 5 must wait:** creating the version makes Temporal start the Worker, which fails to connect until the secret has an `AWSCURRENT` version.

   ```bash
   (
     if [ ! -t 0 ]; then
       printf 'Run this in an interactive terminal; nothing was changed\n' >&2; exit 1
     fi
     printf 'Temporal API key: ' >&2
     IFS= read -r -s key
     printf '\nlen=%s\n' "${#key}" >&2
     dots=${key//[^.]/}
     if [ "${#dots}" -ne 2 ]; then
       printf 'Expected a JWT-shaped key with two dots; nothing was changed\n' >&2; exit 1
     fi
     printf %s "$key" | aws secretsmanager put-secret-value \
       --secret-id <SECRET_ARN> --secret-string file:///dev/stdin \
       --region <AWS_REGION> --query VersionId --output text
   )
   ```

   The parentheses run the script in a subshell, so the key does not stay in the user's shell. `printf %s` pipes the key without a trailing newline; pasting it into a file or a here-doc would store one and corrupt the credential. The key never appears on a command line, so it is not visible in the process list.

   Write the script to a file outside the code directory, with every placeholder filled in, such as `<WORK_DIR>/<RUNTIME_NAME>-handoff.sh`. It contains no key. Then end your turn with the user's action first: one line saying the run is waiting on them; the file's absolute path and the command `bash <ABSOLUTE_PATH>` to run in their own terminal; and what to reply when it finishes. Put status and notes after that. While the run is blocked on the hand-off, repeat these steps in full rather than pointing back to them.

4. **Verify without reading the key back.** When the user replies, check the secret's versions rather than trusting the reply:

   ```bash
   aws secretsmanager describe-secret --secret-id <SECRET_ARN> \
     --region <AWS_REGION> --query VersionIdsToStages
   ```

   Expect exactly one version labelled `AWSCURRENT`. Never run `aws secretsmanager get-secret-value`, and never print, log or pass the key in a command. The Worker registering in Step 5 is the proof that the key authenticates. <!-- aws-cli 2.35.15: secretsmanager describe-secret help -->

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

First check that no Runtime with this name is already deployed: `agentcore status --runtime <RUNTIME_NAME> --json`. If one is, it belongs to an earlier run or another user. Ask before you redeploy it, and record it as *reused*. <!-- agentcore 0.28.1: status --help -->

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

Also record the execution role that `agentcore deploy` created, for the inventory and teardown: `aws bedrock-agentcore-control get-agent-runtime --agent-runtime-id <RUNTIME_ID> --region <AWS_REGION> --query roleArn --output text`. <!-- aws-cli 2.35.15: bedrock-agentcore-control get-agent-runtime help -->

Do not continue until both ARNs are printed and the endpoint state is `READY`. <!-- docs/guides/durable-agent-on-agentcore.mdx:256; aws:agent-runtime-versioning.html#endpoint-lifecycle -->

## Step 4: Grant Temporal permission to invoke the Runtime

For Temporal Cloud, deploy the Cloud invocation-role template with the Runtime ARN plus `*` and an External ID the user chooses, then read the stack's `RoleARN` output. Before you create a stack, check whether the user already has one (`aws cloudformation describe-stacks --stack-name <STACK_NAME>`). If they do, add this Runtime to it as `iam.md` describes, and record it as *reused*. → `iam.md`. For a self-hosted Service, use the role from `self-hosted.md` step 4 and skip this step. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:290-348 -->

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

Record each resource as you go, and mark each one **created** (this run made it) or **reused** (it existed before this run). Each step above checks for an existing resource before it creates or grants one. Teardown removes only what is marked *created*.

| Resource | Record |
|---|---|
| AgentCore project | Directory, target name, account, Region |
| Runtime | Name, Runtime ID, Runtime ARN |
| Named endpoint | Name, endpoint ARN, Runtime version it points to |
| Runtime execution role | Role ARN (created by `agentcore deploy` with the Runtime) |
| Invocation role | Cloud: CloudFormation stack name and `RoleARN`, and if reused, the Runtime ARN this run added to `AgentRuntimeARNs`. Self-hosted: the role from `self-hosted.md`. External ID. |
| Secret | Secret ARN, and the policy file in `additionalPolicies` |
| Temporal | Worker Deployment name, Build ID, Task Queue, and whether this run created the Worker Deployment |
| Temporal API key | Who created it. The user owns it; it is never in the inventory. |

Hand the inventory to the user at the end.

## Teardown

Remove resources in this order: Temporal first, so that nothing invokes the Runtime, then the Runtime and endpoint, then shared resources. Remove a shared resource (invocation role, stack, secret) only if this run created it **and** nothing else uses it. Ask before changing or deleting anything marked *reused*. Never unset or delete a version while Pinned Workflows still need it (see `versioning.md`).

1. **Unset the current version.** A current version cannot be deleted. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:405-417; temporal 1.8.3: worker deployment set-current-version --help -->

   ```bash
   temporal worker deployment set-current-version \
     --namespace <TEMPORAL_NAMESPACE> --deployment-name <DEPLOYMENT_NAME> \
     --unversioned --yes
   ```

2. **Delete the version once it has no pollers and is not draining, then the deployment.** Running Workers stop on their idle policy (see the selected `sdk-<language>.md`). `delete-version` refuses a draining version, so read the drainage status and retry rather than adding `--skip-drainage`. Delete the deployment only if this run created it and it has no other versions. <!-- temporal 1.8.3: worker deployment describe-version, delete-version, delete --help -->

   ```bash
   temporal worker deployment describe-version \
     --namespace <TEMPORAL_NAMESPACE> --deployment-name <DEPLOYMENT_NAME> \
     --build-id <BUILD_ID> -o json | jq -r '.drainageInfo.drainageStatus // empty'
   temporal worker deployment delete-version \
     --namespace <TEMPORAL_NAMESPACE> --deployment-name <DEPLOYMENT_NAME> --build-id <BUILD_ID>
   temporal worker deployment delete --namespace <TEMPORAL_NAMESPACE> --name <DEPLOYMENT_NAME>
   ```

3. **Remove the Runtime and its endpoint.** The AgentCore CLI removes resources from `agentcore.json`, and the next deploy deletes them from AWS. To remove one endpoint of an earlier build and keep the Runtime, use `agentcore remove runtime-endpoint --name <ENDPOINT_NAME> -y` instead. Never remove an endpoint that a live version still points to. <!-- agentcore 0.28.1: remove agent --help, remove runtime-endpoint --help, deploy --help -->

   ```bash
   agentcore remove agent --name <RUNTIME_NAME> -y
   agentcore deploy --target <TARGET> -y
   ```

   Verify by reading state back. `agentcore status --runtime <RUNTIME_NAME> --json` must no longer list the Runtime as deployed. `aws bedrock-agentcore-control get-agent-runtime --agent-runtime-id <RUNTIME_ID> --region <AWS_REGION>` must return `ResourceNotFoundException`. Then check the execution role with `aws iam get-role --role-name <EXECUTION_ROLE_NAME>`. If it still exists, report it to the user instead of deleting it, because the CLI's stack owns it. <!-- agentcore 0.28.1: status --help; aws-cli 2.35.15: bedrock-agentcore-control get-agent-runtime, iam get-role help -->

4. **Invocation role (Temporal Cloud).** Delete the stack only if this run created it. First read its Runtime list:

   ```bash
   aws cloudformation describe-stacks --stack-name <STACK_NAME> --region <AWS_REGION> \
     --query 'Stacks[0].Parameters[?ParameterKey==`AgentRuntimeARNs`].ParameterValue' --output text
   ```

   - It lists only this Runtime's ARN with `*`: delete the stack and wait on its state. `describe-stacks` then reports that the stack does not exist.

     ```bash
     aws cloudformation delete-stack --stack-name <STACK_NAME> --region <AWS_REGION>
     aws cloudformation wait stack-delete-complete --stack-name <STACK_NAME> --region <AWS_REGION>
     ```

   - It lists other Runtime ARNs, or the stack was *reused*: keep it. With the user's approval, update it to drop only this Runtime's ARN, keeping every other parameter value.

   <!-- aws-cli 2.35.15: cloudformation describe-stacks, delete-stack, wait stack-delete-complete help -->

   For a self-hosted Service, apply the same rule to the stack from `self-hosted.md` Step 4. Leave the server-side `sts:AssumeRole` grant to the user, because other Runtimes may rely on it.

5. **Secret.** If this run created the secret and no other Runtime lists it in `additionalPolicies`, schedule its deletion. Keep the recovery window so that a mistake can be undone, and do not add `--force-delete-without-recovery` unless the user asks. Keep a *reused* secret. Delete the policy file from the project. <!-- aws-cli 2.35.15: secretsmanager delete-secret help -->

   ```bash
   aws secretsmanager delete-secret --secret-id <SECRET_ARN> \
     --recovery-window-in-days 7 --region <AWS_REGION>
   ```

   Verify with `aws secretsmanager describe-secret --secret-id <SECRET_ARN> --query DeletedDate`, which must print a date.

6. **Temporal API key.** Ask before you suggest revoking it, because a Temporal Cloud API key is account-scoped, not deployment-scoped. The user revokes it, as described in [`iam.md`](iam.md#temporal-api-key-lifecycle).

Leave the local AgentCore project directory to the user.
