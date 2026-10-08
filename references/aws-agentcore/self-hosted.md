# AgentCore — Self-hosted Temporal Service setup

<!-- Sources:
  docs/production-deployment/worker-deployments/serverless-workers/agentcore-self-hosted-setup.mdx
  docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx
-->

AgentCore Serverless Workers require **Temporal Service v1.32.0 or later**. A self-hosted Service needs no Pre-release access request. Complete this page, then follow `setup.md`. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore-self-hosted-setup.mdx:22-34 -->

Confirm the server version first. If the Service is older than v1.32.0, stop and tell the user that it must be upgraded. Do not attempt a partial setup.

## 1. Configure network access in both directions

- **Runtime → Temporal.** The Temporal Service frontend must be reachable from the AgentCore Runtime. If the frontend is public, use `"networkMode": "PUBLIC"` on the Runtime. If it is reachable only on a private network, configure the Runtime for [VPC access](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/agentcore-vpc.html) and connect that VPC to the network that hosts the Service. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore-self-hosted-setup.mdx:36-42 -->
- **Temporal → AgentCore APIs.** The Service must reach the AgentCore control-plane and data-plane APIs. If it runs in an AWS VPC without internet access, configure [AgentCore interface VPC endpoints](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/vpc-interface-endpoints.html) for both. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore-self-hosted-setup.mdx:44-47 -->

Ask the user which case applies. Do not infer it from the address format.

## 2. Enable the Worker Controller Instance

The WCI is disabled by default. Enable it, the `aws-agentcore` compute provider and the `no-sync` scaling algorithm in the Service's [dynamic configuration](https://docs.temporal.io/references/dynamic-configuration):

```yaml
workercontroller.enabled:
  - value: true

workercontroller.compute_providers.enabled:
  - value:
      - aws-agentcore

workercontroller.scaling_algorithms.enabled:
  - value:
      - no-sync
```

<!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore-self-hosted-setup.mdx:49-72 -->

Both lists are allowlists. **If either already holds values for other providers or algorithms, such as `aws-lambda` or `rate-based`, append to the list. Do not replace it.** <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore-self-hosted-setup.mdx:70-75 -->

To enable the WCI for specific Namespaces only:

```yaml
workercontroller.enabled:
  - value: true
    constraints:
      namespace: 'your-namespace'
```

The Service watches the file and applies changes **without a restart**. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore-self-hosted-setup.mdx:77-87 -->

Keep `workercontroller.compute_providers.aws.require_role_and_external_id` at its default (enabled). With that default, every Worker Deployment Version must carry an invocation role and External ID. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore-self-hosted-setup.mdx:89-91 -->

## 3. Give the Temporal Service AWS credentials

The Service assumes the invocation role, so it needs an AWS identity with `sts:AssumeRole` on that role. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore-self-hosted-setup.mdx:95-112 -->

- **On AWS (EC2, ECS, EKS):** the server uses the attached instance, task or pod role automatically. Grant that role `sts:AssumeRole` on the invocation role from step 4.
- **Outside AWS:** use [IAM Roles Anywhere](https://aws.amazon.com/iam/roles-anywhere/). Static `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` / `AWS_REGION` in the server environment also work, but they are not recommended. Never ask the user to paste access keys into the session.

To identify the ARN of the identity the server runs as, run `aws sts get-caller-identity` **in the server environment**. That ARN becomes `TemporalIamRoleArn` in step 4. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore-self-hosted-setup.mdx:144 -->

## 4. Create the invocation role

Deploy [`assets/temporal-self-hosted-serverless-worker-agentcore-role.yaml`](../../assets/temporal-self-hosted-serverless-worker-agentcore-role.yaml). Its trust policy allows only `TemporalIamRoleArn`, and only with the External ID. Pass each Runtime ARN with a trailing `*` so the policy covers the Runtime and its endpoints. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore-self-hosted-setup.mdx:114-120 --> The Runtime must exist first, so run this after `setup.md` Step 3 has produced the Runtime ARN.

```bash
aws cloudformation create-stack \
  --stack-name <STACK_NAME> \
  --template-body file://assets/temporal-self-hosted-serverless-worker-agentcore-role.yaml \
  --parameters \
    ParameterKey=TemporalIamRoleArn,ParameterValue=<TEMPORAL_SERVER_ROLE_ARN> \
    ParameterKey=AssumeRoleExternalId,ParameterValue=<EXTERNAL_ID> \
    ParameterKey=AgentRuntimeARNs,ParameterValue='<AGENT_RUNTIME_ARN>*' \
  --capabilities CAPABILITY_NAMED_IAM \
  --region <AWS_REGION>

aws cloudformation wait stack-create-complete --stack-name <STACK_NAME> --region <AWS_REGION>

aws cloudformation describe-stacks --stack-name <STACK_NAME> \
  --query 'Stacks[0].Outputs[?OutputKey==`RoleARN`].OutputValue' \
  --output text --region <AWS_REGION>
```

<!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore-self-hosted-setup.mdx:130-140,224-236 -->

| Parameter | Value |
|---|---|
| `TemporalIamRoleArn` | ARN of the IAM role or user the Temporal Service runs as (step 3). |
| `AssumeRoleExternalId` | A string of 5–45 characters that the user chooses. Use the same value in the Worker Deployment Version. |
| `AgentRuntimeARNs` | Comma-separated Runtime ARNs, each with a trailing `*`. |
| `RoleName` | Optional. Defaults to `Temporal-AgentCore-Worker`, and the full name must be 64 characters or fewer. Set a different name for each extra copy of the stack. |

<!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore-self-hosted-setup.mdx:122-147 -->

## 5. Continue with `setup.md`, with these differences

<!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore-self-hosted-setup.mdx:240-245; docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:292-298,374-376 -->

- **Runtime environment:** set `TEMPORAL_ADDRESS` and `TEMPORAL_NAMESPACE` to the self-hosted frontend and Namespace. Configure the self-hosted Service's authentication settings in the Worker entry point. The Temporal Cloud API-key variables do not apply unless the Service uses API keys.
- **IAM:** skip the Temporal Cloud IAM step. Use the role ARN and External ID from step 4.
- **Worker Deployment Version:** create it with the Temporal CLI. The Cloud UI and **Validate Connection** are Temporal Cloud only. Confirm registration with the checks in `diagnostics.md` instead.
