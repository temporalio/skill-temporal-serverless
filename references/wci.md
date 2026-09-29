# Worker Controller Instance (WCI)

<!-- Sources:
  docs/encyclopedia/workers/serverless-workers.mdx
-->

## Role and lifecycle

The Worker Controller Instance (WCI) is a Temporal system Workflow that scales compute for a Worker Deployment Version from Task Queue conditions. One WCI runs for each Worker Deployment Version that has a compute provider configured, in the same Namespace as the Worker Deployment. <!-- docs/encyclopedia/workers/serverless-workers.mdx:66-68 -->

Temporal creates and manages the WCI automatically. Never create, start, or manage it yourself.

## Inputs

The WCI responds to sync match failures and Task Queue backlog or rate information. The selected provider's constraints and diagnostics references describe how those inputs become compute changes, along with the provider-specific lifecycle and failure signatures. <!-- docs/encyclopedia/workers/serverless-workers.mdx:70-72 -->

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

**A Running WCI is not evidence that its scaling Activities work.** The Workflow continues-as-new and remains Running while those Activities fail. Read its history and look for Activity failures.

Diagnose from Temporal's signals before scanning the cloud account or project. Do not enumerate compute resources across regions to reverse-engineer state. After identifying the failed WCI Activity, use the selected provider's `diagnostics.md` to interpret it and inspect provider-specific state.
