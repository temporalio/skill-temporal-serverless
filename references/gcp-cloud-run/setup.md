# GCP Cloud Run — Setup (happy path)

<!-- Sources:
  docs/production-deployment/worker-deployments/serverless-workers/cloud-run/index.mdx
  docs/develop/<sdk>/workers/serverless-workers/cloud-run.mdx
-->

End-to-end: write a standard Worker, containerize it, push the image, create a Worker Pool at zero instances, grant Temporal permission to scale it, register a Worker Deployment Version, set it current, verify. For the two service accounts and the Terraform module, see `iam.md`. For what the execution model does and does not bound, see `constraints.md`. For new builds and rollback, see `versioning.md`. If it doesn't work, see `diagnostics.md`.

## Prerequisites

<!-- docs/production-deployment/worker-deployments/serverless-workers/cloud-run/index.mdx:36-51 -->

- **Cloud Run support is Pre-release and access-gated.** The user creates a support ticket or contacts their account team. Confirm this before anything else.
- A Temporal Cloud account with a **GCP-hosted Namespace**, or self-hosted Temporal Service v1.31.0+. The Namespace's cloud provider must match the compute provider — an AWS-hosted Namespace cannot drive Cloud Run. Regions need not match.
- For self-hosted, complete `self-hosted.md` first.
- Every Workflow must declare a versioning behavior, or the Worker must set a default.
- A GCP project with the **Cloud Run and Artifact Registry APIs enabled**, and permission to create Worker Pools, service accounts, and Secret Manager secrets.
- `gcloud` CLI installed and authenticated. The Google Cloud console or Terraform also work.
- **Terraform** installed — Temporal ships the IAM setup as a Terraform module.
- A Temporal SDK. Supported on Cloud Run: Go, Python, TypeScript, Java, .NET, **Ruby, and Rust** — the last two are Cloud Run only and unavailable on Lambda.

**Check the region before deploying.** Google reports high deployment latency creating or updating Cloud Run resources in some regions, including `us-central1`, and recommends another region while the issue is open. → `constraints.md`.

## Step 1: Write Worker code

**There is no Cloud Run Worker package.** Write an ordinary long-lived Worker — same client, same `Worker`/`WorkerFactory`, same registration — and add Worker Versioning, which Serverless Workers require. Do not reach for anything in `../sdk-configuration.md`; that file is AWS Lambda only.

Two things the Worker must do:

1. **Declare its Worker Deployment Version and enable versioning**, with a deployment name and build ID that exactly match the version you register in Step 4.
2. **Read its configuration from the environment** — address, Namespace, Task Queue, credentials — so one image can run against any Namespace. The pool supplies these via `--set-env-vars` and `--set-secrets`.

Per-SDK code lives in the SDK's own Cloud Run guide (`docs/develop/<sdk>/workers/serverless-workers/cloud-run.mdx`). Every SDK page follows the same five sections: versioned Worker, connection, image packaging, scale-in safety, observability.

**The entrypoint must start the Worker process**, so an instance begins polling as soon as it starts. <!-- .../cloud-run/index.mdx:467-468 -->

## Step 2: Containerize the Worker

<!-- .../cloud-run/index.mdx:465-636 -->

Per-runtime notes that matter, from the deployment guide:

| SDK | Notes |
|---|---|
| Go | Multi-stage; `CGO_ENABLED=0` for a static binary, which is what a `distroless/static` base expects. |
| Python | `pip install "temporalio>=1.30.0,<2"`; entrypoint runs the Worker module. |
| TypeScript | **Keep `ca-certificates` installed** — without it the Worker fails at startup with `TransportError: tonic::transport::Error(Transport, NativeCertsNotFound)`. Use a **glibc** image, not Alpine. Set `NODE_OPTIONS=--max-old-space-size=<MB>` to ~80% of the instance memory limit. |
| Java | Fat jar on a JRE image; set `-XX:MaxRAMPercentage=75` — the JVM reads the container limit but defaults max heap to 25% of it. |
| .NET | `dotnet publish` in a build stage, run on the .NET runtime image. |

**The `NativeCertsNotFound` error is the same root cause as the .NET one on Lambda** — a Rust-core SDK that cannot find system root CAs — reached here through a slim base image rather than an overridden `SSL_CERT_FILE`. Any Rust-core SDK (TypeScript, Python, .NET, Ruby) in a minimal image needs CA certificates present.

## Step 3: Build and push the image

```bash
gcloud builds submit \
  --tag <REGION>-docker.pkg.dev/<YOUR_GCP_PROJECT>/<REPOSITORY>/my-temporal-worker:build-1 \
  --project <YOUR_GCP_PROJECT> \
  --region <REGION>
```

Tag the image with the build ID. It keeps image, pool, and Worker Deployment Version aligned, which matters because the compute configuration cannot pin a revision (→ `constraints.md`).

## Step 4: Create the Worker Pool

**Create one pool per Worker Deployment Version, at zero instances.** The WCI raises the count once the version is current and Tasks arrive.

```bash
gcloud run worker-pools deploy my-temporal-worker-pool-build-1 \
  --image <REGION>-docker.pkg.dev/<YOUR_GCP_PROJECT>/<REPOSITORY>/my-temporal-worker:build-1 \
  --region <REGION> \
  --project <YOUR_GCP_PROJECT> \
  --service-account <RUNNER_SERVICE_ACCOUNT> \
  --instances 0 \
  --set-env-vars TEMPORAL_ADDRESS=<address>:7233,TEMPORAL_NAMESPACE=<namespace>,TEMPORAL_TASK_QUEUE=my-task-queue \
  --set-secrets TEMPORAL_API_KEY=<SECRET_NAME>:latest
```

| Parameter | Description |
|---|---|
| `--image` | The image pushed in Step 3. |
| `--service-account` | The **runner** service account instances run as. **Not** the invoker Temporal impersonates. → `iam.md`. |
| `--instances` | Set to `0`; the WCI manages the count once the version is current. |
| `--set-env-vars` | Non-secret configuration. |
| `--set-secrets` | Maps a Secret Manager secret to an env var — use it for `TEMPORAL_API_KEY` or TLS material. |

**Secrets are the documented default here, not an upgrade.** Unlike Lambda's guide, the Cloud Run path puts the API key in Secret Manager from the start, so there is no "acceptable for development only" plaintext step to warn about.

## Step 5: Grant Temporal permission to scale the pool

Cloud Run has no invocation grant. Temporal **impersonates an invoker service account** and drives the Cloud Run admin API. Create it with Temporal's Terraform module — the Cloud Cloud UI supplies a filled-in template under **Workers → Create Worker Deployment → Access**. → `iam.md` for the module, its variables, and the two-service-account distinction.

Terraform's `invoker_email` output is what Step 6 needs.

## Step 6: Register the Worker Deployment Version

```bash
temporal worker deployment create --namespace <NS> --name my-app

temporal worker deployment create-version \
  --namespace <NS> \
  --deployment-name my-app \
  --build-id build-1 \
  --gcp-cloud-run-project <YOUR_GCP_PROJECT> \
  --gcp-cloud-run-region <REGION> \
  --gcp-cloud-run-worker-pool my-temporal-worker-pool-build-1 \
  --gcp-cloud-run-service-account <INVOKER_SERVICE_ACCOUNT>
```

| Flag | Description |
|---|---|
| `--deployment-name` / `--build-id` | Must match the Worker code exactly. |
| `--gcp-cloud-run-project` | Project containing the pool. |
| `--gcp-cloud-run-region` | Pool region. |
| `--gcp-cloud-run-worker-pool` | Pool name from Step 4. |
| `--gcp-cloud-run-service-account` | The **invoker** — Terraform's `invoker_email`. |

Through the UI, the version is set current automatically; through the CLI it is a separate step.

### Checkpoint: the Cloud Run checkpoint is weaker than Lambda's

On Lambda, creating a version triggers a validation *invocation*, and a bound Task Queue proves the whole path works. **Cloud Run has no such invocation** — nothing starts an instance at registration.

**Validate Connection** (Workers → Deployments → select → Actions) impersonates the invoker and reads the pool. It confirms the pool exists, that Temporal can impersonate the invoker, and that the invoker can *read*. It **starts no instance and does not exercise the update permission scaling needs**, so passing validation is *not* proof Temporal can scale the pool. <!-- .../cloud-run/index.mdx:824-827 -->

So the two useful checks are:

```bash
# has any Worker ever polled under this version?
temporal worker deployment describe-version \
  --deployment-name my-app --build-id build-1 --report-task-queue-stats

# has Temporal ever actually written to the pool?
gcloud run worker-pools describe my-temporal-worker-pool-build-1 \
  --region <REGION> --project <PROJECT> --format=yaml
```

In the pool's `metadata.annotations`, `serving.knative.dev/lastModifier` becoming the invoker service account is the real proof that scaling works. → `diagnostics.md`.

## Step 7: Set the version current

```bash
temporal worker deployment set-current-version \
  --namespace <NS> --deployment-name my-app --build-id build-1 --yes
```

Without this, Tasks do not route to the version and the WCI starts no instances. The command prompts for confirmation; **run non-interactively without `--yes` it exits having changed nothing**, which reads as success. Read the state back with `temporal worker deployment describe`.

## Step 8: Verify

```bash
temporal workflow start \
  --namespace <NS> --task-queue my-task-queue \
  --type MyWorkflow --input '"Hello, serverless!"'
```

Tasks arriving with no active pollers cause the WCI to raise the instance count; Cloud Run starts an instance, the Worker connects and processes the Task.

Confirm from two independent signals:

- **Temporal** — Task completions in the Workflow's event history.
- **Cloud Run** — pool logs showing Worker startup and Task processing:
  ```bash
  gcloud run worker-pools logs read my-temporal-worker-pool-build-1 \
    --region <REGION> --project <YOUR_GCP_PROJECT>
  ```
  **The pool produces no logs while scaled to zero**, so read them while an instance is up. An empty log is not evidence of failure.

## Teardown

Record what you create as you go: pool name, image tag and Artifact Registry repository, runner and invoker service accounts, the Terraform state, secrets, deployment name and build ID, project and region.

**The ordering problem is milder than Lambda's**, where the function had to go before the version or the delete deadlocked on active pollers. Here, scaling the pool to zero stops the pollers without destroying anything.

1. Unset the current version — a Current version cannot be deleted:
   ```bash
   temporal worker deployment set-current-version \
     --deployment-name my-app --unversioned --yes
   ```
2. Scale the pool to zero, which ends polling:
   ```bash
   gcloud run worker-pools update <POOL_NAME> --instances 0 --region <REGION> --project <PROJECT>
   ```
3. Wait for drainage, then delete the version, then the deployment:
   ```bash
   temporal worker deployment describe-version --deployment-name my-app --build-id build-1
   temporal worker deployment delete-version --deployment-name my-app --build-id build-1
   temporal worker deployment delete --name my-app
   ```
4. Delete the Worker Pool:
   ```bash
   gcloud run worker-pools delete <POOL_NAME> --region <REGION> --project <PROJECT>
   ```
5. `terraform destroy` the IAM module — **only if this deployment created it.** One invoker service account can serve several pools, so a shared one may still be in use. → `iam.md`.
6. Delete the container image from Artifact Registry, and any Secret Manager secrets created for this deployment. Ask before revoking a Temporal Cloud API key: it is account-scoped, not deployment-scoped.
