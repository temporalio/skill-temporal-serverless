# AgentCore — IAM and permissions

<!-- Sources:
  docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx
  docs/production-deployment/worker-deployments/serverless-workers/agentcore-self-hosted-setup.mdx
  docs/guides/durable-agent-on-agentcore.mdx
  s3: https://temporal-server-scaled-workers-config.s3.us-west-2.amazonaws.com/cloudformation/temporal-worker-role-agentcore.yaml
  samples-python@d67795f: bedrock_agentcore/strands_agent/
-->

This file covers three separate identities. Never merge them:

| Identity | Who uses it | Created by |
|---|---|---|
| **Operator** | The person or CI running `agentcore`, `aws` and CloudFormation commands | Already exists |
| **Runtime execution role** | AgentCore, inside the Runtime session, when Worker code calls AWS (Bedrock, Code Interpreter, Secrets Manager) | `agentcore deploy` |
| **Invocation role** | Temporal, to get the named endpoint and invoke the Runtime. It never runs Worker code. | The CloudFormation templates below |

<!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:129-131,347-348; docs/guides/durable-agent-on-agentcore.mdx:312-318; samples-python/bedrock_agentcore/strands_agent/README.md:110-116 -->

## Operator permissions

The operator needs permission to create AgentCore resources, CloudFormation stacks and IAM roles. See [IAM permissions for AgentCore Runtime](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-permissions.html). The CDK must also be bootstrapped in the target account and Region. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:47-53 --> The CloudFormation deploys below create named IAM roles, so they need `--capabilities CAPABILITY_NAMED_IAM`. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:317-327 -->

## Runtime execution role

`agentcore deploy` creates the execution role. To grant more permissions to Worker code, list policy files under the Runtime's `additionalPolicies` in `agentcore.json`. The sample uses this to add Code Interpreter access. <!-- samples-python/bedrock_agentcore/strands_agent/README.md:110-116 --> In production, store the Temporal API key in AWS Secrets Manager and grant **this** role permission to read it, not the invocation role. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:129-131 -->

## Invocation role

The invocation role grants exactly `bedrock-agentcore:InvokeAgentRuntime` and `bedrock-agentcore:GetAgentRuntimeEndpoint` on the Runtime ARNs you pass in. Its trust policy requires an External ID, which prevents [confused deputy](https://docs.aws.amazon.com/IAM/latest/UserGuide/confused-deputy.html) attacks. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:300-303,347-348 -->

**The External ID.** The user chooses it, and it must be 5–45 characters from `[a-zA-Z0-9_+=,.@-]`. Use the same value in the stack and in the Worker Deployment Version. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:300-301; assets/temporal-cloud-serverless-worker-agentcore-role.yaml:6-11 --> A fixed value is fine for a tutorial. Use a unique value in production. <!-- docs/guides/durable-agent-on-agentcore.mdx:285 -->

**`AgentRuntimeARNs` takes the Runtime ARN with a trailing `*`, not the endpoint ARN.** The wildcard makes the policy cover the Runtime and all of its endpoints, which `GetAgentRuntimeEndpoint` requires. As a result, new endpoints for later builds need no IAM change. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:266-283,305-306; assets/temporal-cloud-serverless-worker-agentcore-role.yaml:13-19 -->

```text
Runtime ARN  (IAM, plus *):     arn:aws:bedrock-agentcore:<region>:<account>:runtime/<runtime-id>
Endpoint ARN (Temporal only):   arn:aws:bedrock-agentcore:<region>:<account>:runtime/<runtime-id>/runtime-endpoint/<endpoint-name>
```

### Temporal Cloud

Template: [`assets/temporal-cloud-serverless-worker-agentcore-role.yaml`](../../assets/temporal-cloud-serverless-worker-agentcore-role.yaml). It is vendored unchanged from Temporal's S3 copy. The trust policy lists Temporal Cloud's principals and requires `sts:ExternalId`. <!-- assets/temporal-cloud-serverless-worker-agentcore-role.yaml:45-68 -->

```bash
aws cloudformation create-stack \
  --stack-name <STACK_NAME> \
  --template-body file://assets/temporal-cloud-serverless-worker-agentcore-role.yaml \
  --parameters \
    ParameterKey=AssumeRoleExternalId,ParameterValue=<EXTERNAL_ID> \
    ParameterKey=AgentRuntimeARNs,ParameterValue='<AGENT_RUNTIME_ARN>*' \
    ParameterKey=RoleName,ParameterValue=<ROLE_NAME> \
  --capabilities CAPABILITY_NAMED_IAM \
  --region <AWS_REGION>

aws cloudformation wait stack-create-complete --stack-name <STACK_NAME> --region <AWS_REGION>

aws cloudformation describe-stacks --stack-name <STACK_NAME> \
  --query 'Stacks[0].Outputs[?OutputKey==`RoleARN`].OutputValue' \
  --output text --region <AWS_REGION>
```

<!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:317-345 -->

**Check the role name length before you create the stack.** The template names the role `<ROLE_NAME>-<STACK_NAME>`, and an IAM role name has at most 64 characters, counting the hyphen. If the name is longer, CloudFormation cannot create the role. With the default `RoleName` (`Temporal-Cloud-Serverless-Worker`, 32 characters), keep `<STACK_NAME>` to 31 characters or fewer. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:308-315; docs/guides/durable-agent-on-agentcore.mdx:287-293 -->

### Self-hosted Temporal Service

Template: [`assets/temporal-self-hosted-serverless-worker-agentcore-role.yaml`](../../assets/temporal-self-hosted-serverless-worker-agentcore-role.yaml). It is vendored from the Temporal docs. Instead of Temporal Cloud's principals, it trusts the single AWS identity in `TemporalIamRoleArn`, which is the identity the Temporal Service runs as. Unlike the Cloud template, it uses `RoleName` as given (default `Temporal-AgentCore-Worker`). Set a different `RoleName` when you deploy more than one copy of the stack. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore-self-hosted-setup.mdx:114-147 -->

The deploy command, parameters and the server-side `sts:AssumeRole` grant are in `self-hosted.md`, [Step 4](self-hosted.md#4-create-the-invocation-role).

## Validate Connection

On Temporal Cloud, **Actions → Validate Connection** on the Worker Deployment Version checks that Temporal can assume the invocation role, get the named endpoint, and invoke the Runtime. A successful check does not show that the Worker registered and polled. Confirm that separately, as described in `diagnostics.md`. <!-- docs/production-deployment/worker-deployments/serverless-workers/agentcore.mdx:401-403 -->

## Rules

- Never paste or echo AWS secret keys or the Temporal API key in commands the agent runs.
- When an existing invocation stack is in use, add the new Runtime ARN to its `AgentRuntimeARNs` (comma-separated) instead of creating a parallel role. Ask the user before you change a stack this run did not create.
