# Temporal Serverless Workers Skill

Deploy and operate [Temporal](https://temporal.io/) Workers on serverless compute with help from a coding agent. The skill guides an agent through the complete AWS Lambda lifecycle: scoping, access checks, Worker implementation, packaging, deployment, Temporal registration, verification, troubleshooting, updates, and rollback.

> [!WARNING]
> This skill is in Public Preview and will continue to evolve. Pin the Temporal SDK, serverless Worker package, and CLI versions for long-lived projects.

> [!NOTE]
> Temporal Serverless Workers on AWS Lambda are in Public Preview and are available to all Temporal Cloud customers without an access request. AWS Lambda is currently the only compute provider supported by this skill.

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

## Support

| Area | Supported |
|---|---|
| Compute | AWS Lambda — Public Preview |
| Temporal | Temporal Cloud and self-hosted Temporal Service |
| SDKs | Go, Python, TypeScript, Java, .NET |
| Other compute providers | Not currently supported |

For Temporal Cloud, the Namespace must be hosted on AWS. The Namespace and Lambda function may be in different AWS regions.

## Before you start

Before starting, make sure you can sign in to:

- An AWS account with permission to inspect and create the required Lambda, IAM, CloudFormation, and logging resources.
- A Temporal Cloud Namespace hosted on AWS, or a compatible self-hosted Temporal Service.

You do not need to install or configure the AWS CLI, `tcld`, or the Temporal CLI before you begin. The skill checks what is already available and can help set up the tools and supported login flows needed for the task. If you prefer not to install a CLI, or a login method is unavailable, it can guide you through the corresponding Temporal Cloud UI or AWS console steps instead. It never asks you to paste credentials or secrets into the conversation.

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

For a new deployment, the skill follows five stages:

1. **Scope** — confirm the SDK, compute provider, Namespace, region, and resource-naming prefix.
2. **Access** — verify AWS and Temporal identities and permissions, then present the exact billable resources for approval.
3. **Build** — install the serverless Worker package, inspect its current API, author the Worker, and deploy it.
4. **Connect** — configure Temporal's invocation role, register the Worker Deployment Version, validate the Task Queue binding, and set the version current.
5. **Verify and hand back** — run a Workflow, confirm two independent health signals, inventory every created resource, and offer teardown.

Nothing is created before you approve the resource list. Troubleshooting and inspection requests skip the deployment walkthrough and begin with read-only diagnostics.

## Important operating constraints

- Serverless Workers and their APIs are Public Preview, not generally available.
- Every Workflow must use a Worker Versioning behavior: `Pinned` or `AutoUpgrade`.
- The deployment name and build ID in Worker code must exactly match the registered Worker Deployment Version.
- Production releases should map each build ID to one immutable Lambda version.
- Activities must finish within the compute provider's execution bound and configured shutdown buffer; Workflow duration remains unbounded. On AWS Lambda that bound is the invocation limit — see [`references/aws-lambda/constraints.md`](references/aws-lambda/constraints.md).
- Secrets belong in a secret store for shared or production deployments, not plaintext environment variables.
- Temporal creates and manages the Worker Controller Instance (WCI); this skill never creates or manages it directly.

## Repository guide

| Path | Contents |
|---|---|
| [`SKILL.md`](SKILL.md) | Core workflow, safety gates, provider rules, and reference routing |
| [`references/concepts.md`](references/concepts.md) | Architecture, invocation flow, autoscaling, lifecycle, constraints, and use cases |
| [`references/sdk-configuration.md`](references/sdk-configuration.md) | Go, Python, TypeScript, Java, and .NET packages, entry points, versioning behavior, and tuned defaults |
| [`references/aws-lambda/setup.md`](references/aws-lambda/setup.md) | End-to-end deployment, verification, and teardown workflow |
| [`references/aws-lambda/iam.md`](references/aws-lambda/iam.md) | Operator permissions, Lambda execution role, and Temporal invocation role |
| [`references/aws-lambda/constraints.md`](references/aws-lambda/constraints.md) | What follows from Lambda's per-invocation execution model — Worker lifetime, invocation deadline, timeout triple, Activity duration bounds — and what does not generalize to other providers |
| [`references/aws-lambda/diagnostics.md`](references/aws-lambda/diagnostics.md) | Diagnostic decision tree and WCI inspection |
| [`references/aws-lambda/versioning.md`](references/aws-lambda/versioning.md) | Immutable releases, updates, and rollback |
| [`references/aws-lambda/observability.md`](references/aws-lambda/observability.md) | OpenTelemetry and ADOT configuration |
| [`references/aws-lambda/self-hosted.md`](references/aws-lambda/self-hosted.md) | Self-hosted Temporal prerequisites and configuration |
| [`assets/`](assets/) | CloudFormation templates for Temporal invocation roles |

### Provenance comments

Factual lines in the reference files carry an HTML comment naming their source, so claims stay auditable. There are three forms:

| Form | Meaning |
|---|---|
| `<!-- docs/<path>.mdx:NN -->` | Taken from Temporal's documentation, with the source line |
| `<!-- verified against <artifact>:<version> -->` | Read from the published package itself (`javap`, sources jar, XML docs) |
| `<!-- measured -->` / `<!-- inferred … -->` | Observed in a real deployment, or reasoned from evidence — not documented |

The second form exists because the documentation and the shipped artifact can disagree: as of `temporal-aws-lambda` 1.38.0 the docs' Java entry point (`LambdaWorker.run`) does not exist in the artifact. **Where they conflict, the artifact wins and the disagreement is stated in the text.** Add `(line unverified)` when citing a documentation page whose line numbers have not been checked against the source tree, rather than guessing a number.

## Feedback

Feedback is welcome in the [Temporal Community Slack](https://t.mp/slack), in the [`#topic-ai` channel](https://temporalio.slack.com/archives/C0818FQPYKY), or through [GitHub issues](https://github.com/temporalio/skill-temporal-serverless/issues).
