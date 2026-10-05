# Temporal Serverless Workers Skill

Deploy and operate [Temporal](https://temporal.io/) Workers on serverless compute with help from a coding agent. The skill guides an agent through the complete AWS Lambda lifecycle: scoping, access checks, Worker implementation, packaging, deployment, Temporal registration, verification, troubleshooting, updates, and rollback.

It also supports GCP Cloud Run Worker Pools through a separate pool-based path, without changing the Lambda workflow.

> [!WARNING]
> This skill is in Public Preview and will continue to evolve. Pin the Temporal SDK and CLI versions for long-lived projects, plus any provider-specific package you use, such as the AWS Lambda serverless Worker package or a Cloud Run OpenTelemetry helper.

> [!NOTE]
> Temporal Serverless Workers on AWS Lambda are in Public Preview and are available to all Temporal Cloud customers without an access request.
> GCP Cloud Run is also in Public Preview and available without an access request.

## What the skill can do

- Build Serverless Workers with the Go, Python, TypeScript, Java, or .NET SDK.
- Package and deploy Workers to AWS Lambda with the correct architecture, timeout, and shutdown settings.
- Configure the separate AWS roles used by the Lambda function and by Temporal.
- Register a Worker Deployment Version, validate its Task Queue binding, and set it current.
- Verify a deployment from both Temporal Workflow history and Lambda logs.
- Diagnose Workers that are not invoked or do not complete Tasks.
- Publish immutable Lambda versions, update deployments, and roll back safely.
- Add OpenTelemetry observability with the AWS Distro for OpenTelemetry.
- Configure self-hosted Temporal deployments that meet the serverless prerequisites.
- Build ordinary long-lived Workers for GCP Cloud Run Worker Pools without a provider-specific package or handler.
- Containerize and deploy one immutable Worker Pool per build ID.
- Configure separate Cloud Run runner and invoker service accounts.
- Diagnose WCI-driven pool resizing, scale-in interruption, quotas, and pool annotations.

## Support

| Area | Supported |
|---|---|
| Compute | AWS Lambda and GCP Cloud Run — Public Preview |
| Temporal | Temporal Cloud and self-hosted Temporal Service |
| SDKs | Go, Python, TypeScript, Java, .NET |
| Other compute providers | Not currently supported by this skill |

For Temporal Cloud, the Namespace must be hosted on AWS. The Namespace and Lambda function may be in different AWS regions.

For Cloud Run, the Namespace must be hosted on GCP. The Namespace and Worker Pool may be in different GCP regions.

## Before you start

Before starting, make sure you can sign in to:

- An AWS account with permission to inspect and create the required Lambda, IAM, CloudFormation, and logging resources.
- A Temporal Cloud Namespace hosted on AWS, or a compatible self-hosted Temporal Service.
- For Cloud Run instead: a GCP project with permission to inspect and create Worker Pools, Artifact Registry images, IAM bindings, Secret Manager secrets, and logs, plus a GCP-hosted Namespace or compatible self-hosted Temporal Service.

You do not need to install or configure the AWS CLI, `gcloud`, `tcld`, or the Temporal CLI before you begin. The skill checks what is already available and can help set up the tools and supported login flows needed for the task. If you prefer not to install a CLI, or a login method is unavailable, it can guide you through the corresponding Temporal Cloud UI, AWS console, or Google Cloud console steps instead. It never asks you to paste credentials or secrets into the conversation.

## Installation

Clone the repository into your coding agent's skills directory.

### Codex

```bash
mkdir -p ~/.codex/skills
git clone https://github.com/temporalio/skill-temporal-serverless.git \
  ~/.codex/skills/temporal-serverless
```

If `CODEX_HOME` is set, install the repository under `$CODEX_HOME/skills/temporal-serverless` instead.

### Claude Code

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/temporalio/skill-temporal-serverless.git \
  ~/.claude/skills/temporal-serverless
```

For another coding agent that supports skills, use its documented skills directory and keep [`SKILL.md`](SKILL.md) at the root of the installed folder. Restart or open a new agent session after installation if the skill is not discovered immediately.

To update a Git-based installation:

```bash
git -C /path/to/temporal-serverless pull --ff-only
```

## Usage

Ask your coding agent for the outcome you want. For example:

```text
Deploy a Python Temporal Serverless Worker to AWS Lambda.
```

```text
My Serverless Worker is not being invoked. Diagnose it without changing resources.
```

```text
Publish an immutable build of this TypeScript Worker and roll the deployment forward.
```

```text
Add OpenTelemetry tracing to my Go Serverless Worker on Lambda.
```

```text
Package this Java Worker as a shaded jar and deploy it to Lambda.
```

```text
Deploy this .NET Worker to Lambda with a runtime-specific publish.
```

```text
Deploy a Go Temporal Worker to a GCP Cloud Run Worker Pool.
```

For a new deployment, the skill follows five stages.

### AWS Lambda

1. **Scope** — confirm the SDK, compute provider, Namespace, region, and resource-naming prefix.
2. **Access** — verify AWS and Temporal identities and permissions, then present the exact billable resources for approval.
3. **Build** — install the serverless Worker package, inspect its current API, author the Worker, and deploy it.
4. **Connect** — configure Temporal's invocation role, register the Worker Deployment Version, validate the Task Queue binding, and set the version current.
5. **Verify and hand back** — run a Workflow, confirm two independent health signals, inventory every created resource, and offer teardown.

### GCP Cloud Run

1. **Scope** — confirm the SDK, GCP-hosted Namespace, project, region, and resource-naming prefix.
2. **Access** — verify GCP and Temporal identities and permissions, then present the exact billable resources for approval.
3. **Build** — author an ordinary long-lived Worker, containerize it, and deploy a dedicated Worker Pool for the build ID.
4. **Connect** — configure the invoker identity, register the Worker Deployment Version, verify Task Queue bindings, and set the version current.
5. **Verify and hand back** — run a Workflow, confirm its history and pool logs, inventory every created resource, and offer teardown.

Nothing is created before you approve the resource list. Troubleshooting and inspection requests skip the deployment walkthrough and begin with read-only diagnostics.

## Important operating constraints

### AWS Lambda

- Serverless Workers and their APIs are Public Preview, not generally available.
- Every Workflow must use a Worker Versioning behavior: `Pinned` or `AutoUpgrade`.
- The deployment name and build ID in Worker code must exactly match the registered Worker Deployment Version.
- Production releases should map each build ID to one immutable Lambda version.
- Activities must finish within the Lambda invocation limit and configured shutdown buffer; Workflow duration remains unbounded.
- Secrets belong in a secret store for shared or production deployments, not plaintext environment variables.
- Temporal creates and manages the Worker Controller Instance (WCI); this skill never creates or manages it directly.

### GCP Cloud Run

- Cloud Run uses ordinary long-lived Worker APIs; there is no per-invocation handler or serverless Worker package.
- Every Workflow must use `Pinned` or `AutoUpgrade`, and each build ID maps to a dedicated immutable Worker Pool.
- Cloud Run has no invocation deadline, but scale-in can interrupt Activities; use graceful shutdown and Heartbeats for resumable work.
- Do not share the Task Queue with an independently managed long-lived fleet.
- A minimum instance count of zero permits scaling to zero; a nonzero minimum intentionally keeps capacity running.

## Repository guide

| Path | Contents |
|---|---|
| [`SKILL.md`](SKILL.md) | Core workflow, safety gates, provider rules, and reference routing |
| [`references/concepts.md`](references/concepts.md) | AWS Lambda architecture, invocation flow, autoscaling, lifecycle, constraints, and use cases |
| [`references/wci.md`](references/wci.md) | Shared WCI lifecycle, inputs, Workflow ID pattern, inspection commands, and health interpretation |
| [`references/aws-lambda/sdk-go.md`](references/aws-lambda/sdk-go.md) | Go package, API, handler, build, packaging, Lambda deployment values, tuned defaults, connection configuration, and OpenTelemetry integration |
| [`references/aws-lambda/sdk-python.md`](references/aws-lambda/sdk-python.md) | Python package, API, handler, build, packaging, Lambda deployment values, tuned defaults, connection configuration, OpenTelemetry integration, and diagnostics |
| [`references/aws-lambda/sdk-typescript.md`](references/aws-lambda/sdk-typescript.md) | TypeScript package, API, handler, Workflow pre-bundling, build, packaging, Lambda deployment values, tuned defaults, connection configuration, and OpenTelemetry integration |
| [`references/aws-lambda/sdk-java.md`](references/aws-lambda/sdk-java.md) | Java artifact, API, handler, callbacks, build, packaging, Lambda deployment values, tuned defaults, connection configuration, OpenTelemetry integration, logging, and diagnostics |
| [`references/aws-lambda/sdk-dotnet.md`](references/aws-lambda/sdk-dotnet.md) | .NET package, API, handler, build, RID-specific publish, Lambda deployment values, tuned defaults, connection configuration, OpenTelemetry integration, logging, and diagnostics |
| [`references/aws-lambda/setup.md`](references/aws-lambda/setup.md) | Shared AWS and Temporal deployment lifecycle, verification, and teardown workflow |
| [`references/aws-lambda/iam.md`](references/aws-lambda/iam.md) | Operator permissions, Lambda execution role, and Temporal invocation role |
| [`references/aws-lambda/diagnostics.md`](references/aws-lambda/diagnostics.md) | Lambda diagnostic decision tree and provider-specific failure interpretation |
| [`references/aws-lambda/versioning.md`](references/aws-lambda/versioning.md) | Immutable releases, updates, and rollback |
| [`references/aws-lambda/observability.md`](references/aws-lambda/observability.md) | Shared ADOT Collector configuration, X-Ray enablement, and IAM permissions |
| [`references/aws-lambda/self-hosted.md`](references/aws-lambda/self-hosted.md) | Self-hosted Temporal prerequisites and configuration |
| [`references/gcp-cloud-run/sdk-go.md`](references/gcp-cloud-run/sdk-go.md) | Go Worker construction, versioning behavior, connection configuration, image packaging, scale-in safety, and observability |
| [`references/gcp-cloud-run/sdk-python.md`](references/gcp-cloud-run/sdk-python.md) | Python Worker construction, versioning behavior, connection configuration, image packaging, scale-in safety, and observability |
| [`references/gcp-cloud-run/sdk-typescript.md`](references/gcp-cloud-run/sdk-typescript.md) | TypeScript Worker construction, versioning behavior, connection configuration, image packaging, scale-in safety, and observability |
| [`references/gcp-cloud-run/sdk-java.md`](references/gcp-cloud-run/sdk-java.md) | Java Worker construction, versioning behavior, connection configuration, image packaging, scale-in safety, and observability |
| [`references/gcp-cloud-run/sdk-dotnet.md`](references/gcp-cloud-run/sdk-dotnet.md) | .NET Worker construction, versioning behavior, connection configuration, image packaging, scale-in safety, and observability |
| [`references/gcp-cloud-run/setup.md`](references/gcp-cloud-run/setup.md) | End-to-end Worker Pool deployment, registration, verification, and teardown |
| [`references/gcp-cloud-run/iam.md`](references/gcp-cloud-run/iam.md) | Operator permissions, runner and invoker service accounts, and Terraform IAM setup |
| [`references/gcp-cloud-run/constraints.md`](references/gcp-cloud-run/constraints.md) | Pool lifecycle, autoscaling, scale-in interruption, and mixed-fleet constraints |
| [`references/gcp-cloud-run/diagnostics.md`](references/gcp-cloud-run/diagnostics.md) | Worker Pool scaling, WCI Activity failures, annotations, quotas, and Worker logs |
| [`references/gcp-cloud-run/versioning.md`](references/gcp-cloud-run/versioning.md) | One Worker Pool per build ID, immutable releases, and rollback |
| [`references/gcp-cloud-run/observability.md`](references/gcp-cloud-run/observability.md) | Cloud Run OpenTelemetry helpers, Google-built Collector sidecar, Cloud Logging, and provider-specific scaling signals |
| [`references/gcp-cloud-run/self-hosted.md`](references/gcp-cloud-run/self-hosted.md) | Self-hosted Temporal Service prerequisites and GCP identity configuration |
| [`assets/`](assets/) | CloudFormation templates for Temporal invocation roles |

## Feedback

Feedback is welcome in the [Temporal Community Slack](https://t.mp/slack), in the [`#topic-ai` channel](https://temporalio.slack.com/archives/C0818FQPYKY), or through [GitHub issues](https://github.com/temporalio/skill-temporal-serverless/issues).
