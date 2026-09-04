# GCP Cloud Run — versioning, updates, and rollback

<!-- Sources:
  docs/encyclopedia/workers/serverless-workers/cloud-run.mdx
  docs/production-deployment/worker-deployments/serverless-workers/cloud-run/index.mdx
-->

## One Worker Pool per Build ID

**The compute configuration names a project, region, and Worker Pool — it does not name a [revision](https://cloud.google.com/run/docs/managing/revisions).** Temporal runs whichever revision the pool happens to serve. That ties a pool to a single build, so **a new build needs a new pool.** Carry the Build ID in the pool name (`my-worker-pool-build-1`) so the mapping stays visible. <!-- docs/encyclopedia/workers/serverless-workers/cloud-run.mdx:82-89 -->

This is the structural difference from Lambda, where one function holds many published versions and the ARN pins which one Temporal invokes. Temporal's Cloud Run compute configuration has no revision selector, so **durable isolation has to come from creating separate pools.** A pool-level instance split can temporarily hold a particular revision, but that split remains mutable state outside Temporal.

## The hazard: redeploying into a live pool

> Deploying a new image into a pool that a live Worker Deployment Version points at creates a new revision, and Cloud Run promotes it to every instance by default. The version does not change, but the code behind it does. Deploying replay-unsafe code this way causes non-determinism errors for in-flight Workflows, **including Pinned ones**. <!-- docs/encyclopedia/workers/serverless-workers/cloud-run.mdx:94-100 -->

Same failure as pointing a Lambda Worker Deployment Version at a mutable `$LATEST`, reached through a different mechanism. Worth stating precisely because the intuition differs: on Lambda you have to *choose* the mutable path by registering an unqualified ARN. **On Cloud Run the mutable path is what a normal `gcloud run worker-pools deploy` does** — the durable Temporal-aligned path is a new pool, while `--no-promote` is an explicit same-pool guardrail.

`Pinned` does not protect you. Pinning routes Workflows to a *version*; it cannot pin the code behind a pool whose revision moved underneath it.

## Guardrail and recovery when a pool is reused

Pool-per-build remains the production rule because Temporal identifies a Worker Pool, not one of its revisions. If an exceptional workflow must deploy a new revision into a pool that still serves a live version, `--no-promote` prevents the new revision from receiving the pool's instances automatically:

```bash
gcloud run worker-pools deploy <POOL_NAME> \
  --image <REGION>-docker.pkg.dev/<PROJECT>/<REPOSITORY>/<IMAGE>:<TAG> \
  --region <REGION> \
  --project <PROJECT> \
  --no-promote
```

This is a guardrail, not immutable versioning: the pool's revision split remains mutable state outside Temporal. If a new revision was already promoted accidentally, send all instances back to the known-good revision explicitly:

```bash
gcloud run worker-pools update-instance-split <POOL_NAME> \
  --to-revisions=<KNOWN_GOOD_REVISION>=100 \
  --region <REGION> \
  --project <PROJECT>
```

`--to-latest` is **not** that rollback: it assigns instances to the current and future `LATEST` revision. Use it only when deliberately removing the sticky `--no-promote` behavior and restoring automatic promotion of future revisions:

```bash
gcloud run worker-pools update-instance-split <POOL_NAME> \
  --to-latest \
  --region <REGION> \
  --project <PROJECT>
```

After emergency recovery, return to one pool per Build ID for the next release so Temporal version routing and deployed code cannot drift independently.

## Rolling out a new build

1. Build and push a new image tagged with the new build ID.
2. **Create a new Worker Pool** at zero instances for that build ID, with the same runner service account.
3. Register a new Worker Deployment Version pointing at the new pool, with a build ID matching the new Worker code.
4. Confirm registration bootstrapped the pool: its `lastModifier` shows the invoker and the expected Task Queue types are bound. Treat the separate UI Validate Connection action as a read-only check. → `diagnostics.md`.
5. Set the new version current, or ramp to it.
6. **Leave the old pool in place** while Pinned Workflows still run on it. It can sit at zero instances; its WCI scales it back up when a Task arrives for that version. <!-- docs/encyclopedia/workers/serverless-workers/cloud-run.mdx:91-92 -->

Only the invoker's permissions are shared across pools, so a new pool usually needs no IAM change — provided the invoker's `deploy_roles` are project-level, which is the module's default. Check `iam.md` if you scoped them to individual pools instead.

## Rollback

Set the previous version current again. Its pool is still there (step 6 above), and its WCI scales it back up on the next Task. Nothing needs rebuilding, and no image or revision has to be reverted — which is the payoff for keeping one pool per build.

Rolling back through Temporal is therefore straightforward if you followed the pool-per-build discipline. If you redeployed into the same pool, setting the previous Temporal version current is not enough because both versions still address mutable pool state. Restore the known-good Cloud Run revision split as described above, provided that revision still exists; if it was deleted, there is nothing left to route back to and the old image must be redeployed deliberately.

## Cost of the discipline

A pool per build means pools accumulate. They cost nothing while at zero instances, but they are real resources with real names, and stale ones make the project harder to reason about. Delete a pool once its version is deleted and no Pinned Workflow can route to it — that ordering is in `setup.md`'s teardown section.
