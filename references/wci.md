# Worker Controller Instance (WCI)

<!-- Sources:
  docs/encyclopedia/workers/serverless-workers.mdx
  docs/encyclopedia/workers/serverless-workers/cloud-run.mdx
-->

## What the WCI is

The Worker Controller Instance (WCI) is a Temporal system Workflow that scales Serverless Workers from Task Queue conditions. One WCI runs for each Worker Deployment Version that has a compute provider configured, in the same Namespace as the Worker Deployment. <!-- docs/encyclopedia/workers/serverless-workers.mdx:66-68 -->

Temporal creates the WCI automatically. Never create, start, or manage it yourself.

The WCI turns Task Queue signals and metrics into an action for the configured provider:

| Provider | WCI action |
|---|---|
| AWS Lambda | Invoke the function so a short-lived Worker starts polling. |
| GCP Cloud Run | Update the Worker Pool's manual instance count so long-lived Worker instances start or stop. |

The provider's own reference contains its scaling algorithm, lifecycle constraints, and failure signatures. Do not carry Lambda invocation behavior into Cloud Run, or Cloud Run pool-sizing behavior into Lambda.

## Inspect the WCI

List WCI Workflows in the Namespace: <!-- docs/encyclopedia/workers/serverless-workers.mdx:75 -->

```bash
temporal workflow list \
  --namespace <NAMESPACE> \
  --query 'TemporalNamespaceDivision = "TemporalWorkerControllerInstance"'
```
<!-- docs/encyclopedia/workers/serverless-workers.mdx:77-81 -->

WCI Workflow IDs follow this pattern: <!-- docs/encyclopedia/workers/serverless-workers.mdx:83 -->

```text
temporal-sys-worker-controller-instance:<deployment-name>:<build-id>
```

Inspect the WCI Workflow history for recent Activity results: <!-- docs/encyclopedia/workers/serverless-workers.mdx:83-84 -->

```bash
temporal workflow show \
  --namespace <NAMESPACE> \
  --workflow-id 'temporal-sys-worker-controller-instance:<DEPLOYMENT_NAME>:<BUILD_ID>'
```
<!-- docs/encyclopedia/workers/serverless-workers.mdx:86-90 -->

## Interpret what you find

**A running WCI is not evidence that provider actions work.** The Workflow continues-as-new and remains running while its invocation or scaling Activities fail. Read its history and look for Activity failures.

Diagnose from Temporal's signals before scanning the cloud account or project. Do not enumerate compute resources across regions to reverse-engineer state. After identifying the failed WCI Activity, use the selected provider's `diagnostics.md` to interpret it and inspect provider-specific state.
