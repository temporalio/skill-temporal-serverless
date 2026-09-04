# Temporal Serverless Workers Skill

Deploy and operate [Temporal](https://temporal.io/) Workers on serverless compute with help from a coding agent. The skill guides an agent through the complete lifecycle on AWS Lambda and GCP Cloud Run: scoping, access checks, Worker implementation, packaging, deployment, Temporal registration, verification, troubleshooting, updates, and rollback.

> [!WARNING]
> This skill is in Public Preview and will continue to evolve. Pin the Temporal SDK, serverless Worker package, and CLI versions for long-lived projects.

> [!NOTE]
> **AWS Lambda** is in Public Preview and available to all Temporal Cloud customers without an access request.
> **GCP Cloud Run** is in Pre-release, its APIs may change in backwards-incompatible ways, and access is granted on request — create a support ticket or contact your account team.

> [!IMPORTANT]
> The two providers have different execution models. Lambda invokes a function per unit of work and the Worker exits when the invocation ends. Cloud Run resizes a pool of long-lived instances, scaling to zero when idle. That changes what the Worker code is, what bounds an Activity, what there is to tune, and how failures present — guidance does not transfer between them. Each provider directory carries a `constraints.md` describing what follows from its model.

## What the skill can do

- Build Serverless Workers with the Go, Python, TypeScript, Java, or .NET SDK on either provider, plus Ruby and Rust on Cloud Run.
- Package and deploy to AWS Lambda with the correct architecture, timeout, and shutdown settings — or containerize and deploy to a Cloud Run Worker Pool.
- Configure the two distinct identities each provider needs: execution and invocation roles on AWS, runner and invoker service accounts on GCP.
- Register a Worker Deployment Version, validate its Task Queue binding, and set it current.
- Verify a deployment from both Temporal Workflow history and the provider's logs.
- Diagnose Workers that are never started, or that start but do not complete Tasks.
- Keep each build immutable, update deployments, and roll back safely.
- Add OpenTelemetry observability — the AWS Distro for OpenTelemetry on Lambda, the SDK's standard setup plus Cloud Logging on Cloud Run.
- Configure self-hosted Temporal deployments that meet the serverless prerequisites.

## Support

| Area | Supported |
|---|---|
| Compute | AWS Lambda — Public Preview; GCP Cloud Run — Pre-release, access-gated |
| Temporal | Temporal Cloud and self-hosted Temporal Service |
| SDKs | Lambda: Go, Python, TypeScript, Java, .NET. Cloud Run: those plus Ruby and Rust |
| Other compute providers | Not currently supported |

For Temporal Cloud, the Namespace must be hosted on the same cloud provider as the compute — AWS for Lambda, GCP for Cloud Run. Regions need not match.

## Before you start

Before starting, make sure you can sign in to:

- For AWS Lambda: an AWS account with permission to inspect and create the required Lambda, IAM, CloudFormation, and logging resources.
- For GCP Cloud Run: a GCP project with the Cloud Run and Artifact Registry APIs enabled, and permission to create Worker Pools, service accounts, and Secret Manager secrets. Cloud Run access must also be enabled on your Temporal Cloud account.
- A Temporal Cloud Namespace hosted on the same cloud provider as your compute, or a compatible self-hosted Temporal Service.

You do not need to install or configure the AWS CLI, `gcloud`, Terraform, `tcld`, or the Temporal CLI before you begin. The skill checks what is already available and can help set up the tools and supported login flows needed for the task. If you prefer not to install a CLI, or a login method is unavailable, it can guide you through the corresponding Temporal Cloud UI, AWS console, or Google Cloud console steps instead. It never asks you to paste credentials or secrets into the conversation.

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
Deploy this Go Worker to a GCP Cloud Run Worker Pool.
```

```text
My Cloud Run Worker Pool is stuck at zero instances. Find out why.
```

For a new deployment, the skill follows five stages:

1. **Scope** — confirm the SDK, compute provider, Namespace, region, and resource-naming prefix.
2. **Access** — verify cloud-provider and Temporal identities and permissions, then present the exact billable resources for approval.
3. **Build** — author the Worker and deploy it. On Lambda that means installing the serverless Worker package and inspecting its current API; on Cloud Run it means an ordinary long-lived Worker in a container image.
4. **Connect** — grant Temporal access to the compute (an invocation role on AWS, an impersonated invoker service account on GCP), register the Worker Deployment Version, confirm it is reachable, and set the version current.
5. **Verify and hand back** — run a Workflow, confirm two independent health signals, inventory every created resource, and offer teardown.

Nothing is created before you approve the resource list. Troubleshooting and inspection requests skip the deployment walkthrough and begin with read-only diagnostics.

## Important operating constraints

- Serverless Workers are not generally available: AWS Lambda is Public Preview, GCP Cloud Run is Pre-release and access-gated.
- Every Workflow must use a Worker Versioning behavior: `Pinned` or `AutoUpgrade`.
- The deployment name and build ID in Worker code must exactly match the registered Worker Deployment Version.
- Production releases should map each build ID to one immutable build: a published Lambda version, or a dedicated Cloud Run Worker Pool.
- Activities must finish within the compute provider's execution bound and configured shutdown buffer; Workflow duration remains unbounded. On AWS Lambda that bound is the invocation limit — see [`references/aws-lambda/constraints.md`](references/aws-lambda/constraints.md).
- Secrets belong in a secret store for shared or production deployments, not plaintext environment variables.
- Temporal creates and manages the Worker Controller Instance (WCI); this skill never creates or manages it directly.

## Repository guide

| Path | Contents |
|---|---|
| [`SKILL.md`](SKILL.md) | Core workflow, safety gates, provider rules, and reference routing |
| [`references/concepts.md`](references/concepts.md) | Architecture, invocation flow, autoscaling, lifecycle, constraints, and use cases |
| [`references/aws-lambda/sdk-go.md`](references/aws-lambda/sdk-go.md) | Go package, API, handler, build, packaging, Lambda deployment values, tuned defaults, connection configuration, and OpenTelemetry integration |
| [`references/aws-lambda/sdk-python.md`](references/aws-lambda/sdk-python.md) | Python package, API, handler, build, packaging, Lambda deployment values, tuned defaults, connection configuration, OpenTelemetry integration, and diagnostics |
| [`references/aws-lambda/sdk-typescript.md`](references/aws-lambda/sdk-typescript.md) | TypeScript package, API, handler, Workflow pre-bundling, build, packaging, Lambda deployment values, tuned defaults, connection configuration, and OpenTelemetry integration |
| [`references/aws-lambda/sdk-java.md`](references/aws-lambda/sdk-java.md) | Java artifact, API, handler, callbacks, build, packaging, Lambda deployment values, tuned defaults, connection configuration, OpenTelemetry integration, logging, and diagnostics |
| [`references/aws-lambda/sdk-dotnet.md`](references/aws-lambda/sdk-dotnet.md) | .NET package, API, handler, build, RID-specific publish, Lambda deployment values, tuned defaults, connection configuration, OpenTelemetry integration, logging, and diagnostics |
| [`references/aws-lambda/setup.md`](references/aws-lambda/setup.md) | Shared AWS and Temporal deployment lifecycle, verification, and teardown workflow |
| [`references/aws-lambda/iam.md`](references/aws-lambda/iam.md) | Operator permissions, Lambda execution role, and Temporal invocation role |
| [`references/aws-lambda/constraints.md`](references/aws-lambda/constraints.md) | What follows from Lambda's per-invocation execution model — Worker lifetime, invocation deadline, timeout triple, Activity duration bounds — and what does not generalize to other providers |
| [`references/aws-lambda/diagnostics.md`](references/aws-lambda/diagnostics.md) | Diagnostic decision tree and WCI inspection |
| [`references/aws-lambda/versioning.md`](references/aws-lambda/versioning.md) | Immutable releases, updates, and rollback |
| [`references/aws-lambda/observability.md`](references/aws-lambda/observability.md) | Shared ADOT Collector configuration, X-Ray enablement, and IAM permissions |
| [`references/aws-lambda/self-hosted.md`](references/aws-lambda/self-hosted.md) | Self-hosted Temporal prerequisites and configuration |
| [`references/gcp-cloud-run/sdk-go.md`](references/gcp-cloud-run/sdk-go.md) | Go versioned Worker, versioning behavior, connection configuration, image packaging, scale-in safety, and observability on Cloud Run |
| [`references/gcp-cloud-run/sdk-python.md`](references/gcp-cloud-run/sdk-python.md) | Python versioned Worker, versioning behavior, connection configuration, image packaging, scale-in safety, and observability on Cloud Run |
| [`references/gcp-cloud-run/sdk-typescript.md`](references/gcp-cloud-run/sdk-typescript.md) | TypeScript versioned Worker, versioning behavior, connection configuration, image packaging, scale-in safety, and observability on Cloud Run |
| [`references/gcp-cloud-run/sdk-java.md`](references/gcp-cloud-run/sdk-java.md) | Java versioned Worker, versioning behavior, connection configuration, image packaging, scale-in safety, and observability on Cloud Run |
| [`references/gcp-cloud-run/sdk-dotnet.md`](references/gcp-cloud-run/sdk-dotnet.md) | .NET versioned Worker, versioning behavior, connection configuration, image packaging, scale-in safety, and observability on Cloud Run |
| [`references/gcp-cloud-run/setup.md`](references/gcp-cloud-run/setup.md) | End-to-end Cloud Run deployment: container image, Worker Pool, registration, verification, teardown |
| [`references/gcp-cloud-run/iam.md`](references/gcp-cloud-run/iam.md) | Operator permissions, runner vs invoker service accounts, and the Terraform module |
| [`references/gcp-cloud-run/constraints.md`](references/gcp-cloud-run/constraints.md) | What follows from Cloud Run's pool-of-instances model — instance lifetime, autoscaling, scale-in interrupting Activities — and what does not generalize |
| [`references/gcp-cloud-run/versioning.md`](references/gcp-cloud-run/versioning.md) | One Worker Pool per build ID, the redeploy-into-a-live-pool hazard, and rollback |
| [`references/gcp-cloud-run/diagnostics.md`](references/gcp-cloud-run/diagnostics.md) | Pool annotations, scaling failures, and Worker-side errors |
| [`references/gcp-cloud-run/observability.md`](references/gcp-cloud-run/observability.md) | Logs, memory sizing per runtime, and the scaling signals to watch |
| [`references/gcp-cloud-run/self-hosted.md`](references/gcp-cloud-run/self-hosted.md) | Self-hosted prerequisites: dynamic config, the server's GCP identity, invoker creation |
| [`assets/`](assets/) | CloudFormation templates for Temporal invocation roles |

## Feedback

Feedback is welcome in the [Temporal Community Slack](https://t.mp/slack), in the [`#topic-ai` channel](https://temporalio.slack.com/archives/C0818FQPYKY), or through [GitHub issues](https://github.com/temporalio/skill-temporal-serverless/issues).
