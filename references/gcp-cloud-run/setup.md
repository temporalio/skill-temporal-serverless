# GCP Cloud Run — Setup (happy path)

End-to-end: write a standard Worker, containerize it, push the image, create a Worker Pool at zero instances, grant Temporal permission to scale it, register a Worker Deployment Version, set it current, verify. For the two service accounts and the Terraform module, see `iam.md`. For what the execution model does and does not bound, see `constraints.md`. For new builds and rollback, see `versioning.md`. If it doesn't work, see `diagnostics.md`.

## Prerequisites

- **Cloud Run — Public Preview.** Available to all Temporal Cloud customers. There is no access request, support ticket, or manual toggle to enable; select GCP Cloud Run as the compute provider and proceed with the deployment.
- A Temporal Cloud account with a **GCP-hosted Namespace**, or self-hosted Temporal Service v1.31.0+. The Namespace must be hosted on GCP; its region need not match the pool's.
- For self-hosted, complete `self-hosted.md` first.
- Every Workflow must declare a versioning behavior, or the Worker must set a default.
- A GCP project with billing enabled and permission to enable service APIs and create Worker Pools, Artifact Registry repositories, Cloud Build jobs, service accounts, and Secret Manager secrets.
- `gcloud` CLI and `jq` installed; authenticate `gcloud` before the preflight. The Google Cloud console or Terraform also work.
- **Terraform** installed — Temporal ships the IAM setup as a Terraform module.
- A Temporal SDK supported by this skill: Go, Python, TypeScript, Java, or .NET.
- A Temporal Cloud API key that can access the target Namespace. **The user creates it**, either in the Temporal Cloud UI under **Settings → API Keys** or by running `tcld apikey create --name <NAME> --duration <DURATION>` in their own terminal. Never run `tcld apikey create` from the agent's shell: it prints the new key. The command creates a key for the current user; add `--service-account-id <ID>` to create it for an existing service account instead. Prefer a service-account-owned key for shared or long-lived Workers and a short-lived user key for a personal test. The key is needed twice, in the operator's Temporal CLI profile and in the Worker's Secret Manager runtime secret; the [API-key hand-off](#hand-off-the-temporal-api-key) sets both from one read.

<!-- docs/production-deployment/worker-deployments/serverless-workers/cloud-run/index.mdx:36-51 -->

The `temporal` CLI commands in Steps 6–8 authenticate through a named CLI profile that the user creates during the [API-key hand-off](#hand-off-the-temporal-api-key). An exported variable in the user's terminal does not reach the agent's shell, so do not rely on `TEMPORAL_API_KEY` there. Never append `--api-key <value>` or put the key in an inline assignment. On macOS the default profile file is `~/Library/Application Support/temporalio/temporal.toml`. The examples below pass `--profile <PROFILE>` explicitly; omit it only when using a different already-configured authentication mechanism, including self-hosted mTLS.

**Use the endpoint Temporal Cloud shows for the Namespace.** Copy the gRPC endpoint from the Namespace page in the Cloud UI, or from the endpoint fields in `tcld namespace get --namespace <NAMESPACE>` output when `tcld` is signed in. Do not construct it from the Namespace name or region. Use the same value, written `<ENDPOINT>` below, for the CLI profile and for the pool's `TEMPORAL_ADDRESS`.

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

Create the Secret Manager secret before deploying the pool. This creates only the secret; the key itself is added during the hand-off below:

```bash
gcloud secrets create <SECRET_NAME> \
  --replication-policy automatic \
  --project <YOUR_GCP_PROJECT>
```

### Hand off the Temporal API key

The user runs this once, after the resource list is approved and the secret exists, **in their own interactive terminal**. It reads the key once, then writes both the CLI profile and the secret version. Do not run it through the agent's shell or a `!`-prefixed command: `read -s` needs a terminal, and the key would pass through the agent's session. Work that needs no key, such as writing the Worker and building the image, can continue meanwhile.

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
  temporal --profile <PROFILE> config set --prop address --value "<ENDPOINT>" &&
  temporal --profile <PROFILE> config set --prop namespace --value "<NAMESPACE>" &&
  temporal --profile <PROFILE> config set --prop api_key --value "$key" &&
  printf %s "$key" | gcloud secrets versions add <SECRET_NAME> \
    --data-file=- --project <YOUR_GCP_PROJECT>
)
```

The parentheses run the script in a subshell, so the key does not stay in the user's shell afterwards. The key is piped with `printf %s` because `gcloud` stores standard input byte for byte: pasting it, pressing Enter, and sending EOF would store a trailing newline and corrupt the credential. `temporal config set` has no standard-input option, so the key is briefly visible in the process list while that command runs; on a shared machine, say so before the user runs it.

Then verify without reading the key back. The agent may run these:

```bash
temporal --profile <PROFILE> config get --prop address
temporal --profile <PROFILE> config get --prop namespace
temporal --profile <PROFILE> worker deployment list --namespace <NAMESPACE>
gcloud secrets versions list <SECRET_NAME> --project <YOUR_GCP_PROJECT> \
  --format='table(name,state,createTime)'
```

Never read back `api_key` with `temporal config get`, and never run `gcloud secrets versions access`. The `worker deployment list` call proves the profile authenticates to the Namespace. Expect exactly one `ENABLED` secret version. If the hand-off ran more than once, keep the newest version, confirm the pool uses `latest`, and disable the older ones with `gcloud secrets versions disable <VERSION> --secret <SECRET_NAME> --project <YOUR_GCP_PROJECT>`. Secret versions are immutable: replace a bad one by adding a correct version, never by editing it.

Grant only the runner access to that secret:

```bash
gcloud secrets add-iam-policy-binding <SECRET_NAME> \
  --member="serviceAccount:<RUNNER_SERVICE_ACCOUNT_ID>@<YOUR_GCP_PROJECT>.iam.gserviceaccount.com" \
  --role roles/secretmanager.secretAccessor \
  --project <YOUR_GCP_PROJECT>
```

Before each create, use the corresponding `describe` command from `iam.md` to avoid colliding with shared resources. The invoker service account is created later by Temporal's Terraform module; do not substitute it for the runner.

## Step 1: Write Worker code

**There is no Cloud Run Worker package.** Write an ordinary long-lived Worker — same client, same `Worker`/`WorkerFactory`, same registration — and add Worker Versioning, which Serverless Workers require.

Two things the Worker must do:

1. **Declare its Worker Deployment Version and enable versioning**, with a deployment name and build ID that exactly match the version you register in Step 6.
2. **Read its configuration from the environment** — address, Namespace, Task Queue, credentials — so one image can run against any Namespace. The pool supplies these via `--set-env-vars` and `--set-secrets`.

**The entrypoint must start the Worker process**, so an instance begins polling as soon as it starts. <!-- docs/production-deployment/worker-deployments/serverless-workers/cloud-run/index.mdx:465-468 -->

## Step 2: Containerize the Worker

Follow the selected SDK's Cloud Run guidance for the container image, runtime, entrypoint, certificate requirements, and memory settings.

The basic path uses one Worker container. If the user asks for OpenTelemetry export to Google Cloud, use the multi-container Worker Pool manifest and Collector sidecar described in `observability.md` instead of trying to add the sidecar with this file's single-container `deploy` command.

## Step 3: Build and push the image

```bash
gcloud builds submit \
  <APP_DIR> \
  --tag <REGION>-docker.pkg.dev/<YOUR_GCP_PROJECT>/<REPOSITORY>/my-temporal-worker:build-1 \
  --project <YOUR_GCP_PROJECT>
```

Tag the image with the build ID. It keeps image, pool, and Worker Deployment Version aligned, which matters because the compute configuration cannot pin a revision (→ `constraints.md`).

Record the immutable digest after the build:

```bash
gcloud artifacts docker images describe \
  <REGION>-docker.pkg.dev/<YOUR_GCP_PROJECT>/<REPOSITORY>/my-temporal-worker:build-1 \
  --format='value(image_summary.digest)'
```

Deploy the recorded digest rather than the mutable tag so a later tag update cannot change what the pool runs.

Confirm the image targets the platform Cloud Run runs, `linux/amd64`, before deploying it. With Docker available, inspect the digest rather than the tag; on a multi-platform index the same format prints one entry per platform instead of a single value:

```bash
gcloud auth configure-docker <REGION>-docker.pkg.dev   # once, if Docker is not yet authorized for this registry
docker buildx imagetools inspect \
  <REGION>-docker.pkg.dev/<YOUR_GCP_PROJECT>/<REPOSITORY>/my-temporal-worker@sha256:<DIGEST> \
  --format '{{.Image.OS}}/{{.Image.Architecture}}'
```

Expect exactly `linux/amd64`. `gcloud auth configure-docker` edits the user's Docker configuration, so mention it in the approval list if it has not been run before. A platform mismatch surfaces only when the instance starts, as a container that never logs.

The happy path deliberately uses Cloud Build's global endpoint. Supplying `--region` can require additional regional build and staging-bucket setup, depending on the project's Cloud Build bucket policy. If regional builds are required for a private pool or data-residency policy, pre-create or select the regional source/log buckets, grant the build identity access, and then add `--region <REGION>`.

## Step 4: Create the Worker Pool

**Create one pool per Worker Deployment Version, initially at zero instances.**

```bash
gcloud run worker-pools deploy my-temporal-worker-pool-build-1 \
  --image <REGION>-docker.pkg.dev/<YOUR_GCP_PROJECT>/<REPOSITORY>/my-temporal-worker@sha256:<DIGEST> \
  --region <REGION> \
  --project <YOUR_GCP_PROJECT> \
  --service-account <RUNNER_SERVICE_ACCOUNT> \
  --instances 0 \
  --set-env-vars TEMPORAL_ADDRESS=<ENDPOINT>,TEMPORAL_NAMESPACE=<NAMESPACE>,TEMPORAL_TASK_QUEUE=my-task-queue,TEMPORAL_DEPLOYMENT_NAME=my-app,TEMPORAL_BUILD_ID=build-1 \
  --set-secrets TEMPORAL_API_KEY=<SECRET_NAME>:latest
```

| Parameter | Description |
|---|---|
| `--image` | The digest recorded after Step 3. |
| `--service-account` | The **runner** service account instances run as. **Not** the invoker Temporal impersonates. → `iam.md`. |
| `--instances` | Set to `0`; the WCI takes ownership of the count when the Worker Deployment Version is registered. |
| `--set-env-vars` | Non-secret configuration. |
| `--set-secrets` | Maps a Secret Manager secret to an env var — use it for `TEMPORAL_API_KEY` or TLS material. |

**Put the API key in Secret Manager from the start.** Do not introduce a plaintext environment-variable deployment step.

Wait for Cloud Run to report the pool ready and record the ready revision:

```bash
gcloud run worker-pools describe my-temporal-worker-pool-build-1 \
  --region <REGION> --project <YOUR_GCP_PROJECT> \
  --format='yaml(status.conditions,status.latestReadyRevisionName)'

gcloud run worker-pools revisions describe <LATEST_READY_REVISION> \
  --region <REGION> --project <YOUR_GCP_PROJECT> \
  --format='yaml(status.imageDigest)'
```

Require the `Ready` condition to be true and the revision digest to match the recorded image digest. A pool can be Ready at zero instances without ever starting the image; registration is the first runtime test.

## Step 5: Grant Temporal permission to scale the pool

Cloud Run has no invocation grant. Temporal **impersonates an invoker service account** and drives the Cloud Run admin API. Create it with Temporal's Terraform module — the Cloud UI supplies a filled-in template under **Workers → Create Worker Deployment → Access**. → `iam.md` for the module, its variables, and the two-service-account distinction.

Terraform's `invoker_email` output is what Step 6 needs.

**Wait for IAM propagation before Step 6.** The module's `serviceAccountTokenCreator` grants take time to reach Temporal's impersonation path. In test runs, `create-version` issued 41–60 seconds after the apply was rejected with a 403 on `iam.serviceAccounts.getAccessToken`. Allow a few minutes after the apply, and rely on the read-back in Step 6 rather than on the clock. → `iam.md`.

## Step 6: Register the Worker Deployment Version

```bash
temporal --profile <PROFILE> worker deployment create --namespace <NS> --name my-app

temporal --profile <PROFILE> worker deployment create-version \
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
  --gcp-cloud-run-utilization-target 0.8 \
  --gcp-cloud-run-scale-down-stabilization-duration 90s
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
| `--gcp-cloud-run-scale-down-stabilization-duration` | How long the scaler waits after the most recent sync match failure before scaling in; defaults to `90s`. Set it to `0s` to disable the wait. |

The accepted scaler group depends on the installed CLI. **Read `temporal worker deployment create-version --help` before constructing the command. This is the canonical compatibility guidance for scaler flags:**

- CLI v1.8.2 accepts four coupled flags: minimum, maximum, initial, and utilization target. Remove `--gcp-cloud-run-scale-down-stabilization-duration` from the example above.
- CLI v1.8.3 and later accept all five flags shown above.
- On either version, omit the whole supported group to accept the defaults (`0`, `30`, `0`, `0.8`, and a server-side `90s` stabilization duration).

Supplying only part of the supported group fails CLI validation. On an existing version, omitting the scaler flags leaves its current settings unchanged.

### Confirm the version was created

Do not truncate `create-version` output; its error is the only record of a rejected create. Then read the version back immediately:

```bash
temporal --profile <PROFILE> worker deployment describe-version \
  --namespace <NS> --deployment-name my-app --build-id build-1
```

If it reports the version as not found, **the create was rejected**, even if the command appeared to succeed. During IAM propagation the rejection can read `worker pool … not found` although the pool name is correct, because the WCI's `ValidateSpec` failed with a 403 on `iam.serviceAccounts.getAccessToken`. Read the `ValidateSpec` result in the WCI history (`../wci.md`) before changing any names. → [`diagnostics.md`](diagnostics.md#start-here-did-the-expected-worker-bind).

Through the UI, the version is set current automatically; through the CLI it is a separate step.

### Checkpoint: verify registration

Creating the Worker Deployment Version starts its WCI, which temporarily raises the pool to at least one instance so the Worker can bind its Task Queues. This occurs even when the minimum and initial instance counts are `0`. Wait for the expected Task Queue types before setting the version current. The separate UI **Validate Connection** action checks pool read access only and is not a substitute for this registration check. <!-- docs/production-deployment/worker-deployments/serverless-workers/cloud-run/index.mdx:824-827 -->

First run the machine-checkable [Task Queue binding check](diagnostics.md#start-here-did-the-expected-worker-bind). Do not continue until it finds the expected Task Queue and types. Then verify that Temporal wrote to the pool:

```bash
gcloud run worker-pools describe my-temporal-worker-pool-build-1 \
  --region <REGION> --project <PROJECT> --format=yaml
```

In the pool's `metadata.annotations`, `serving.knative.dev/lastModifier` should show the invoker service account after the WCI updates the pool. That proves only that Temporal wrote the requested pool size; it does not prove that the intended image started or that the Worker bound the Task Queue. Confirm the pool's deployed digest and a Worker startup log that announces the expected deployment name, build ID, and Task Queue. → `diagnostics.md`.

## Step 7: Set the version current

```bash
temporal --profile <PROFILE> worker deployment set-current-version \
  --namespace <NS> --deployment-name my-app --build-id build-1 --yes
```

Without this, new traffic does not route to the version. The registration instance may already have started and bound the Task Queue, but that bootstrap does not make the version current. The command prompts for confirmation. **When run non-interactively without `--yes`, it exits having changed nothing**, which reads as success. Read the state back with `temporal worker deployment describe`.

## Step 8: Verify

```bash
temporal --profile <PROFILE> workflow start \
  --namespace <NS> --task-queue my-task-queue \
  --type MyWorkflow --input '"Hello, serverless!"'
```

Tasks arriving with no active pollers cause the WCI to raise the instance count; Cloud Run starts an instance, the Worker connects and processes the Task.

The scaler minimum is `0`, and the pool normally returns to zero within a few minutes after registration or work completes. Do not wait for scale-to-zero during normal verification: the Workflow above verifies routing and execution, but registration scale-down can race it, so do not classify that run as reliably warm or cold.

Test scale-from-zero only when the user explicitly asks for a cold-path test. It adds roughly four to five minutes per run in observed tests. Wait until the requested count is zero and the logs show the previous instance received `SIGTERM`, then start a second Workflow and measure until the Worker startup/polling log appears.

Confirm from two independent signals:

- **Temporal** — Task completions in the Workflow's event history.
- **Cloud Run** — pool logs showing Worker startup and Task processing, read with the [pool log query](diagnostics.md#read-the-pool-logs). **A scaled-to-zero pool emits no new logs.**

## Teardown

Record what you create as you go: project and region; the Artifact Registry repository, image tag and digest; the pool name; the runner and invoker service accounts, and whether this run created the invoker; the Terraform state directory; the secret and its versions; the deployment name, build ID, and Task Queue; the local CLI profile name; and who created the Temporal API key.

Scale the pool to zero before deleting the version so its pollers stop without destroying the pool prematurely.

1. Unset the current version — a Current version cannot be deleted:
   ```bash
   temporal --profile <PROFILE> worker deployment set-current-version \
     --namespace <NS> --deployment-name my-app --unversioned --yes
   ```
2. Scale the pool to zero, which ends polling:
   ```bash
   gcloud run worker-pools update <POOL_NAME> --instances 0 --region <REGION> --project <PROJECT>
   ```
3. Wait for drainage, then delete the version, then the deployment:
   ```bash
   temporal --profile <PROFILE> worker deployment describe-version --namespace <NS> --deployment-name my-app --build-id build-1
   temporal --profile <PROFILE> worker deployment delete-version --namespace <NS> --deployment-name my-app --build-id build-1
   temporal --profile <PROFILE> worker deployment delete --namespace <NS> --name my-app
   ```
4. Delete the Worker Pool:
   ```bash
   gcloud run worker-pools delete <POOL_NAME> --region <REGION> --project <PROJECT>
   ```
5. `terraform destroy` the IAM module — **only if this deployment created it.** One invoker service account can serve several pools, so a shared one may still be in use. → `iam.md`.
6. Delete the container image from Artifact Registry, and any Secret Manager secrets created for this deployment. Delete the Artifact Registry repository too if this run created it. Ask before revoking a Temporal Cloud API key: it is account-scoped, not deployment-scoped. Once nothing else uses the local CLI profile, remove it with `temporal config delete-profile --profile <PROFILE>`.
