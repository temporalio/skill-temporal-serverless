# GCP Cloud Run — IAM & permissions

<!-- Sources:
  docs/production-deployment/worker-deployments/serverless-workers/cloud-run/index.mdx
  docs/production-deployment/worker-deployments/serverless-workers/cloud-run/self-hosted-setup.mdx
  docs/troubleshooting/serverless-workers/cloud-run.mdx
-->

Three identities, as on AWS Lambda — the **operator** (whose credentials run the commands), the **runner service account** (what the pool runs as), and the **invoker service account** (what Temporal impersonates). The mapping to Lambda's roles is close enough to be useful and different enough to be dangerous if assumed.

| Concept | AWS Lambda | GCP Cloud Run |
|---|---|---|
| The compute's own identity | Execution role, trusted by `lambda.amazonaws.com` | **Runner service account**, set with `--service-account` |
| Temporal's identity | Invocation role, assumed via `sts:AssumeRole` + External ID | **Invoker service account**, reached by **impersonation** |
| Mechanism | `sts:AssumeRole` with a confused-deputy guard | `roles/iam.serviceAccountTokenCreator` — **no External ID equivalent** |
| What Temporal does with it | Invokes the function | Reads and **updates the pool's instance count** via the Cloud Run admin API |
| Infrastructure as code | CloudFormation template (shipped in `assets/`) | **Terraform module**, `serverless-workers/gcp/cloud-run` |

**The two service accounts are not interchangeable**, and confusing them is the single most likely IAM mistake here. The runner runs the pool and never scales it; the invoker scales the pool and never runs it. <!-- .../cloud-run/index.mdx:710-721 -->

## Runner service account

The runtime identity the pool's instances use to reach other Google Cloud services. Set in `setup.md` Step 4 with `gcloud run worker-pools deploy --service-account`. It may be an account that already exists; a dedicated one is preferred.

**It needs no baseline role to run the Worker.** Cloud Run collects `stdout` and `stderr` into Cloud Logging through its own infrastructure, and the Cloud Run *service agent* — not the runner — pulls the container image. Grant only what your code actually reaches: <!-- .../cloud-run/index.mdx:670-677 -->

- `roles/secretmanager.secretAccessor` on each secret you mount, including the Temporal API key.
- `roles/logging.logWriter` **only if** the Worker writes through the Cloud Logging API rather than stdout/stderr.
- Whatever else your Workflows and Activities call.

This differs from Lambda, where `AWSLambdaBasicExecutionRole` is effectively mandatory because the execution role is what creates the log group. Here, logging works with no grant at all.

## Invoker service account

The identity Temporal Cloud impersonates to read and scale the pool. Two grants make it work: <!-- .../cloud-run/self-hosted-setup.mdx:108-113 -->

- Temporal's identity receives **`roles/iam.serviceAccountTokenCreator`** on the invoker, so it can impersonate it.
- The invoker receives a project-level Cloud Run role with at least **`run.workerPools.get`** (read) and **`run.workerPools.update`** (scale). `roles/run.developer` includes both.

The invoker also needs **`roles/iam.serviceAccountUser` on the runner service account**, which Cloud Run requires in order to attach that identity when it scales the pool. The Terraform module applies this.

### The read/update split is a real trap

`run.workerPools.get` alone is enough for **Validate Connection to pass**. Scaling needs `run.workerPools.update`, which validation never exercises. An invoker that can read but not update **validates successfully and then silently fails to scale** — the pool sits at zero while Tasks pile up. <!-- docs/troubleshooting/serverless-workers/cloud-run.mdx:94-99 -->

There is no Lambda analogue: there, the validation invocation exercises the same permission real traffic uses. Check the update path explicitly (→ `diagnostics.md`, `lastModifier`) rather than trusting a green validation.

## The Terraform module

Temporal publishes [`serverless-workers/gcp/cloud-run`](https://github.com/temporalio/terraform-modules/tree/main/modules/serverless-workers/gcp/cloud-run), which creates the invoker service account and applies the grants.

**Get the template from the Cloud UI, not from here.** Under **Workers → Create Worker Deployment → Access**, Temporal Cloud emits a template with `impersonator_service_account_emails` already filled in for your account. Those values are account-specific, which is why every published snippet shows a placeholder. <!-- .../cloud-run/index.mdx:723-744 -->

```hcl
module "serverless-worker-cloud-run" {
  source = "github.com/temporalio/terraform-modules//modules/serverless-workers/gcp/cloud-run"

  project_id         = "<YOUR_GCP_PROJECT>"
  invoker_account_id = "temporal-worker-pool-invoker"

  impersonator_service_account_emails = [
    "<provided by Temporal Cloud>",
  ]

  runner_service_account_email = "temporal-worker-pool-runner@<YOUR_GCP_PROJECT>.iam.gserviceaccount.com"
}
```

| Variable | Required | Description |
|---|---|---|
| `project_id` | Yes | Project hosting the pool and the invoker. |
| `invoker_account_id` | Yes | Name for the invoker the module creates; email becomes `<invoker_account_id>@<project_id>.iam.gserviceaccount.com`. The template supplies one. |
| `impersonator_service_account_emails` | Yes | Temporal Cloud's service accounts, granted `serviceAccountTokenCreator` on the invoker. **From the UI template.** For self-hosted, the GCP identity the server runs as. |
| `runner_service_account_email` | Yes | The runner from `setup.md` Step 4. The module grants the invoker `roles/iam.serviceAccountUser` on it. |
| `invoker_display_name` | No | Defaults to `Temporal Serverless Worker Pool Invoker`. |
| `deploy_roles` | No | Project-level Cloud Run roles for the invoker. Defaults to `roles/run.developer`. A substitute must include `run.workerPools.get` and `run.workerPools.update`. |

```bash
terraform init
terraform apply
```

Use the **`invoker_email`** output as `--gcp-cloud-run-service-account` when registering the version.

### Treat the module as shared, pre-existing infrastructure

Same discipline as Lambda's CloudFormation stack, different tool. One invoker can serve several pools, so before creating a second one, look for an existing account and consider reusing it. **Do not `terraform destroy` state you did not create**, and note that the module's default `invoker_account_id` collides the same way Lambda's default `RoleName` does — an earlier deployment in the project may already own it.

## Operator GCP permissions

The identity running the `gcloud`/Terraform commands needs, at minimum:

| Step | Operator needs |
|---|---|
| Build and push the image (`setup.md` Step 3) | Cloud Build submit, Artifact Registry write |
| Create the Worker Pool (Step 4) | `run.workerPools.create`/`update`, and `iam.serviceAccounts.actAs` on the **runner** to attach it |
| Create secrets | Secret Manager admin on the secrets used |
| Apply the Terraform module (Step 5) | Service-account creation plus IAM policy binding on the project and on the runner |
| Read pool state and logs (verify, diagnose) | `run.workerPools.get`, Cloud Logging read |

`iam.serviceAccounts.actAs` on the runner is the Cloud Run counterpart of Lambda's `iam:PassRole` on the execution role, and fails the same way — a pool create that is denied despite having Cloud Run permissions.

### Preflight

Run before anything that creates or modifies GCP resources. None should fail:

```bash
gcloud auth list                                   # which identity
gcloud config get-value project                    # which project
gcloud services list --enabled --filter='run.googleapis.com OR artifactregistry.googleapis.com'
gcloud run worker-pools list --region <REGION> >/dev/null && echo "cloud run: ok"
gcloud iam service-accounts list >/dev/null && echo "iam read: ok"
terraform version
```

**Classify an authentication failure before acting on it.** An absent or expired credential (`gcloud auth login`, or `gcloud auth application-default login` for Terraform) is recoverable in a minute; an identity that resolves but is denied a specific action is a real permissions problem. Never collect credentials in conversation and never ask the user to paste a service account key — Google recommends against long-lived keys outright.

**Confirm the project explicitly before creating anything.** `gcloud config get-value project` is ambient state that is easy to be wrong about, exactly like an AWS profile pointing at an unintended account. Name the project in the approval list and verify it, rather than trusting the default.
