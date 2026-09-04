# GCP Cloud Run — self-hosted Temporal Service setup

<!-- Sources:
  docs/production-deployment/worker-deployments/serverless-workers/cloud-run/self-hosted-setup.mdx
-->

Serverless Workers require **Temporal Service v1.31.0 or later**. Complete this page before following `setup.md`.

Four prerequisites: network reachability, enable the WCI, give the server a GCP identity, create the invoker service account.

## 1. Cloud Run instances must reach the Temporal Service

The frontend must be reachable **from the Worker Pool instances**. If the Service runs on a private network, that likely means [Direct VPC egress](https://cloud.google.com/run/docs/configuring/vpc-direct-vpc) or a [Serverless VPC Access connector](https://cloud.google.com/run/docs/configuring/vpc-connectors).

Note the direction: instances dial *out* to Temporal. Nothing needs to reach *into* Cloud Run — Temporal drives the pool through the Cloud Run admin API, not by connecting to your instances.

## 2. Enable the Worker Controller Instance

The WCI is **disabled by default** and enabled through [Temporal Service dynamic configuration](https://docs.temporal.io/references/dynamic-configuration):

```yaml
workercontroller.enabled:
  - value: true

workercontroller.compute_providers.enabled:
  - value:
      - gcp-cloud-run

workercontroller.scaling_algorithms.enabled:
  - value:
      - rate-based
```

**Cloud Run requires the `rate-based` algorithm.** Because a pool is a set of long-lived instances, the WCI resizes it from arrival and backlog rates. The `no-sync` algorithm applies only to providers invoked once per Task, and **pairing it with `gcp-cloud-run` is rejected** — a concrete way the two providers' execution models surface in server configuration.

To enable per Namespace instead of globally:

```yaml
workercontroller.enabled:
  - value: true
    constraints:
      namespace: 'your-namespace'
```

The Service watches the file and applies updates **without a restart**.

Two optional global keys cover multi-hop impersonation:

| Key | Purpose |
|---|---|
| `workercontroller.compute_providers.gcp.intermediary_service_accounts` | Service accounts to impersonate in sequence before the invoker. Unset means the server impersonates the invoker directly. |
| `workercontroller.compute_providers.gcp.first_delegate_as_base` | When `true`, the first entry is the identity the server's ambient credentials impersonate directly and the rest are passed as token-creator delegates. Defaults to `false`, passing the whole chain as delegates. |

## 3. Give the Temporal Service a GCP identity

The server impersonates the invoker service account, so it must first run as a GCP identity permitted to do so.

**On GCP (GCE, GKE):** the attached service account is used automatically through [Application Default Credentials](https://cloud.google.com/docs/authentication/application-default-credentials). No extra credential configuration — you grant *that* account impersonation rights in step 4.

**Outside GCP:** use [Workload Identity Federation](https://cloud.google.com/iam/docs/workload-identity-federation) and point `GOOGLE_APPLICATION_CREDENTIALS` at the credential configuration file it produces. That variable also accepts a service account key file, but **Google recommends against long-lived keys** — prefer federation, and never ask a user to paste a key.

## 4. Create the invoker service account

The Service scales the pool as an **invoker** service account, which only reads and scales the pool. The identity instances *run as* is the separate **runner** service account set on the pool in `setup.md`. → `iam.md` for the full distinction.

Two grants:

- The GCP identity the Service runs as (step 3) gets **`roles/iam.serviceAccountTokenCreator`** on the invoker.
- The invoker gets a project-level Cloud Run role with at least **`run.workerPools.get`** and **`run.workerPools.update`**. `roles/run.developer` includes both.

The same Terraform module works — pass the server's GCP identity as the impersonator instead of Temporal Cloud's accounts:

```hcl
module "serverless-worker-cloud-run" {
  source = "github.com/temporalio/terraform-modules//modules/serverless-workers/gcp/cloud-run"

  project_id         = "<YOUR_GCP_PROJECT>"
  invoker_account_id = "temporal-serverless-worker"

  runner_service_account_email = "<WORKER-POOL-RUNNER-SERVICE-ACCOUNT-EMAIL>"

  impersonator_service_account_emails = [
    "<TEMPORAL_SERVICE_GCP_IDENTITY>",
  ]
}
```

Use the module's `invoker_email` output as `--gcp-cloud-run-service-account` when registering the Worker Deployment Version.

**This is the one place self-hosted is simpler than Cloud:** there is no UI-provided template to copy, because you already know the impersonating identity — it is your own server's.

## Then

Follow `setup.md` from Step 1. The read/update permission trap in `iam.md` applies identically, and `diagnostics.md`'s `lastModifier` check is still the fastest way to confirm the server can actually scale the pool.
