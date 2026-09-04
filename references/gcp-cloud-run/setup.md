# GCP Cloud Run — Setup (happy path)

End-to-end: write a standard Worker, containerize it, push the image, create a Worker Pool at zero instances, grant Temporal permission to scale it, register a Worker Deployment Version, set it current, verify. For the two service accounts and the Terraform module, see `iam.md`. For what the execution model does and does not bound, see `constraints.md`. For new builds and rollback, see `versioning.md`. If it doesn't work, see `diagnostics.md`.

## Prerequisites

- **Cloud Run support is Pre-release and access-gated.** The user creates a support ticket or contacts their account team. Confirm this before anything else.
- A Temporal Cloud account with a **GCP-hosted Namespace**, or self-hosted Temporal Service v1.31.0+. The Namespace must be hosted on GCP; its region need not match the pool's.
- For self-hosted, complete `self-hosted.md` first.
- Every Workflow must declare a versioning behavior, or the Worker must set a default.
- A GCP project with billing enabled and permission to enable service APIs and create Worker Pools, Artifact Registry repositories, Cloud Build jobs, service accounts, and Secret Manager secrets.
- `gcloud` CLI installed and authenticated. The Google Cloud console or Terraform also work.
- **Terraform** installed — Temporal ships the IAM setup as a Terraform module.
- A Temporal SDK. Supported on Cloud Run: Go, Python, TypeScript, Java, .NET, **Ruby, and Rust**.

The `temporal` CLI commands in Steps 6 and 7 must inherit authentication from an existing profile or from the process environment. **Never append `--api-key <value>` or put the key in an inline assignment.** If `TEMPORAL_API_KEY` is not already populated, set it privately in the user's own terminal without putting the value in shell history:

```bash
export TEMPORAL_ADDRESS="<address>:7233"
export TEMPORAL_NAMESPACE="<namespace>"
printf 'Temporal API key: ' >&2
IFS= read -r -s TEMPORAL_API_KEY
printf '\n' >&2
export TEMPORAL_API_KEY
```

Do not run the secret-reading commands through an agent shell, ask the user to paste the key into conversation, or inspect the resulting variable. A configured Temporal CLI profile is equally valid and avoids a session environment variable.

## Prepare a clean GCP project

After the resource list is approved, create only what is missing. First enable every API used by the commands below:

```bash
gcloud services enable \
  run.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com \
  secretmanager.googleapis.com \
  iam.googleapis.com \
  iamcredentials.googleapis.com \
  cloudresourcemanager.googleapis.com \
  --project <YOUR_GCP_PROJECT>
```

Create a regional Docker repository for the Worker image:

```bash
gcloud artifacts repositories create <REPOSITORY> \
  --repository-format docker \
  --location <REGION> \
  --project <YOUR_GCP_PROJECT> \
  --description "Temporal Serverless Worker images"
```

Create the runner service account that pool instances use:

```bash
gcloud iam service-accounts create <RUNNER_SERVICE_ACCOUNT_ID> \
  --display-name "Temporal Cloud Run Worker runner" \
  --project <YOUR_GCP_PROJECT>
```

Create the Secret Manager secret before deploying the pool:

```bash
gcloud secrets create <SECRET_NAME> \
  --replication-policy automatic \
  --project <YOUR_GCP_PROJECT>

gcloud secrets versions add <SECRET_NAME> \
  --data-file=- \
  --project <YOUR_GCP_PROJECT>
```

Run the `versions add` command only in the user's own terminal, provide the Temporal API key on standard input, and then send EOF. Never ask for the value in conversation or run it through an agent shell where input or output may be captured.

Grant only the runner access to that secret:

```bash
gcloud secrets add-iam-policy-binding <SECRET_NAME> \
  --member="serviceAccount:<RUNNER_SERVICE_ACCOUNT_ID>@<YOUR_GCP_PROJECT>.iam.gserviceaccount.com" \
  --role roles/secretmanager.secretAccessor \
  --project <YOUR_GCP_PROJECT>
```

Before each create, use the corresponding `describe` command from `iam.md` to avoid colliding with shared resources. The invoker service account is created later by Temporal's Terraform module; do not substitute it for the runner.

## Step 1: Write Worker code

**There is no Cloud Run Worker package.** Write an ordinary long-lived Worker — same client, same `Worker`/`WorkerFactory`, same registration — and add Worker Versioning, which Serverless Workers require. Use `sdk-<language>.md` in this directory.

Two things the Worker must do:

1. **Declare its Worker Deployment Version and enable versioning**, with a deployment name and build ID that exactly match the version you register in Step 6.
2. **Read its configuration from the environment** — address, Namespace, Task Queue, credentials — so one image can run against any Namespace. The pool supplies these via `--set-env-vars` and `--set-secrets`.

Per-SDK code lives in `sdk-<language>.md` in this directory, one file per SDK. Each covers the versioned Worker, connection, image packaging, graceful shutdown, scale-in safety, and observability. The Java, Python, and .NET references also include their logging setup and diagnostic signatures.

**The entrypoint must start the Worker process**, so an instance begins polling as soon as it starts.

## Step 2: Containerize the Worker

Follow the selected `sdk-<language>.md` reference for the container image, runtime, entrypoint, certificate requirements, and memory settings.

## Step 3: Build and push the image

```bash
gcloud builds submit \
  --tag <REGION>-docker.pkg.dev/<YOUR_GCP_PROJECT>/<REPOSITORY>/my-temporal-worker:build-1 \
  --project <YOUR_GCP_PROJECT>
```

Tag the image with the build ID. It keeps image, pool, and Worker Deployment Version aligned, which matters because the compute configuration cannot pin a revision (→ `constraints.md`).

The happy path deliberately uses Cloud Build's global endpoint. Supplying `--region` can require additional regional build and staging-bucket setup, depending on the project's Cloud Build bucket policy. If regional builds are required for a private pool or data-residency policy, pre-create or select the regional source/log buckets, grant the build identity access, and then add `--region <REGION>`.

## Step 4: Create the Worker Pool

**Create one pool per Worker Deployment Version, initially at zero instances.** Registering the version starts the WCI, which temporarily raises the pool to the scaler's planned count—at least one instance—to register its Task Queues. The version does not need to be current for this bootstrap.

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
| `--instances` | Set to `0`; the WCI takes ownership of the count when the Worker Deployment Version is registered. |
| `--set-env-vars` | Non-secret configuration. |
| `--set-secrets` | Maps a Secret Manager secret to an env var — use it for `TEMPORAL_API_KEY` or TLS material. |

**Put the API key in Secret Manager from the start.** Do not introduce a plaintext environment-variable deployment step.

## Step 5: Grant Temporal permission to scale the pool

Cloud Run has no invocation grant. Temporal **impersonates an invoker service account** and drives the Cloud Run admin API. Create it with Temporal's Terraform module — the Cloud UI supplies a filled-in template under **Workers → Create Worker Deployment → Access**. → `iam.md` for the module, its variables, and the two-service-account distinction.

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
  --gcp-cloud-run-service-account <INVOKER_SERVICE_ACCOUNT> \
  --gcp-cloud-run-min-instances 0 \
  --gcp-cloud-run-max-instances 30 \
  --gcp-cloud-run-initial-instances 0 \
  --gcp-cloud-run-utilization-target 0.8
```

| Flag | Description |
|---|---|
| `--deployment-name` / `--build-id` | Must match the Worker code exactly. |
| `--gcp-cloud-run-project` | Project containing the pool. |
| `--gcp-cloud-run-region` | Pool region. |
| `--gcp-cloud-run-worker-pool` | Pool name from Step 4. |
| `--gcp-cloud-run-service-account` | The **invoker** — Terraform's `invoker_email`. |
| `--gcp-cloud-run-min-instances` | Floor the scaler maintains; `0` allows scale-to-zero. |
| `--gcp-cloud-run-max-instances` | Ceiling the scaler may request; defaults to `30`. |
| `--gcp-cloud-run-initial-instances` | Initial planned count; must be between min and max. |
| `--gcp-cloud-run-utilization-target` | Target average utilization in `(0, 1]`; defaults to `0.8`. |

The four scaler flags are a coupled group: **either omit all four and accept the defaults (`0`, `30`, `0`, `0.8`), or provide all four together.** Supplying only one—even only a higher maximum—fails CLI validation.

Through the UI, the version is set current automatically; through the CLI it is a separate step.

### Checkpoint: distinguish registration from Validate Connection

Creating the Worker Deployment Version starts its WCI. The WCI validates the configuration by reading the Worker Pool, then asks the rate-based algorithm for its Task Queue registration action. For a new Cloud Run version, that action updates the pool's manual instance count to at least one. This special registration floor applies even when `--gcp-cloud-run-min-instances` and `--gcp-cloud-run-initial-instances` are both `0`; those values govern the scaler's normal operating range and initial plan, not whether it can bootstrap Task Queue registration. This exercises both `run.workerPools.get` and the `run.workerPools.update` path used by real scaling.

The update completing proves that Temporal reached Cloud Run, not that the container started or the Worker connected. Wait for the expected Task Queue types to appear; that binding is proof the instance started and polled under the registered deployment name and build ID.

The separate UI **Validate Connection** action (Workers → Deployments → select → Actions) only impersonates the invoker and reads the pool. It starts no instance and does not exercise `run.workerPools.update`, so a green manual validation is weaker than a successful registration bootstrap.

Use both views when diagnosing the checkpoint:

```bash
# has any Worker ever polled under this version?
temporal worker deployment describe-version \
  --namespace <NS> --deployment-name my-app --build-id build-1 --report-task-queue-stats

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

Without this, new traffic does not route to the version. The registration instance may already have started and bound the Task Queue, but that bootstrap does not make the version current. The command prompts for confirmation; **run non-interactively without `--yes` it exits having changed nothing**, which reads as success. Read the state back with `temporal worker deployment describe`.

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
  **A scaled-to-zero pool emits no new logs.** Use `logs read` for historical entries and `logs tail` while an instance is running.

## Teardown

Record what you create as you go: pool name, image tag and Artifact Registry repository, runner and invoker service accounts, the Terraform state, secrets, deployment name and build ID, project and region.

Scale the pool to zero before deleting the version so its pollers stop without destroying the pool prematurely.

1. Unset the current version — a Current version cannot be deleted:
   ```bash
   temporal worker deployment set-current-version \
     --namespace <NS> --deployment-name my-app --unversioned --yes
   ```
2. Scale the pool to zero, which ends polling:
   ```bash
   gcloud run worker-pools update <POOL_NAME> --instances 0 --region <REGION> --project <PROJECT>
   ```
3. Wait for drainage, then delete the version, then the deployment:
   ```bash
   temporal worker deployment describe-version --namespace <NS> --deployment-name my-app --build-id build-1
   temporal worker deployment delete-version --namespace <NS> --deployment-name my-app --build-id build-1
   temporal worker deployment delete --namespace <NS> --name my-app
   ```
4. Delete the Worker Pool:
   ```bash
   gcloud run worker-pools delete <POOL_NAME> --region <REGION> --project <PROJECT>
   ```
5. `terraform destroy` the IAM module — **only if this deployment created it.** One invoker service account can serve several pools, so a shared one may still be in use. → `iam.md`.
6. Delete the container image from Artifact Registry, and any Secret Manager secrets created for this deployment. Ask before revoking a Temporal Cloud API key: it is account-scoped, not deployment-scoped.
