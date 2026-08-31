# AWS Lambda — execution-model constraints

Consequences of Lambda's execution model: **Temporal invokes a function per unit of work, and the Worker exits when the invocation ends.** Everything here follows from that, and none of it generalizes to a provider that keeps long-lived instances alive.

This file is the provider "diff surface." To understand what changes when moving between compute providers, compare this file with the equivalent one for the other provider — the workflow in `SKILL.md` stays the same; these constraints do not.

## Worker lifetime is one invocation

A Worker starts, connects, polls until its shutdown deadline buffer, drains, and exits. The invocation is the Worker's *lifetime*, not one unit of work: a single invocation can serve many Workflow and Activity Tasks from however many executions arrive in its window. Billing follows the invocation, not the work done inside it.

## Set the invocation deadline high enough

Providers often default to a very short timeout — Lambda's default is 3 seconds. If the first invocation times out before the Worker registers the Task Queue, the binding is never created and the Worker is never invoked again. → `setup.md` for the exact defaults, the per-SDK examples, and the GB-seconds trade-off behind choosing a value.

## Tune the timeout triple together for long-running Activities

1. worker stop timeout > longest Activity runtime
2. shutdown deadline buffer > worker stop timeout + shutdown hook time
3. invocation deadline > longest Activity runtime + shutdown deadline buffer

Raising one alone does not help. Raising only the shutdown deadline buffer makes the Worker stop polling earlier but gives in-flight Activities no more time; raising only the worker stop timeout doesn't make it stop polling earlier, so the provider may terminate the Worker first.

*Symptom of getting this wrong:* Activities abandoned mid-execution and retried on a later invocation.

If the longest Activity exceeds half the maximum invocation deadline, recommend Activity Heartbeats. → `../concepts.md`, `../sdk-configuration.md`.

## Activities are bounded by the invocation limit

An Activity must finish within the invocation deadline minus the shutdown deadline buffer. Workflow duration is unbounded and can span many invocations. Flag Activities that approach the provider's limit early — Lambda's ceiling is 15 minutes.

**An Activity that cannot fit needs a different hosting strategy**, not a larger timeout: a long-lived Worker on a separate Task Queue, or a compute provider without a per-invocation ceiling. → `../concepts.md`.

## Eager Activities are always disabled

Every SDK's Lambda Worker package sets this and it cannot be overridden, because eager Activity execution requires a persistent connection that per-invocation Workers don't maintain. Don't suggest it as an optimization. → `../sdk-configuration.md` for the per-SDK setting names.

## Pitfalls specific to this execution model

These are the invocation-shaped members of the pitfall list in `SKILL.md`; the rest apply to any provider.

1. **Failed first invocation.** When a version is created, the WCI invokes the Worker once to validate. If that invocation fails — missing env vars, bad TLS/auth config, missing dependencies, or an invocation deadline too short for the Worker to start and register the Task Queue — the Worker never connects, never polls, the binding is never created, and the Worker is never automatically invoked again. *Fix:* diagnose by manually invoking the function, and confirm the invocation deadline is set high. A successful manual invoke also establishes the binding. → `diagnostics.md`.

2. **Timeout tuning mismatch.** See the timeout triple above. *Fix:* tune the three values together.

3. **Invoke permission scoped to a single build.** *Symptom:* the deployment works, then the *next* release cannot be invoked, with an error that looks like a connection or configuration problem rather than a permissions one. *Cause:* the grant named one immutable build, and the new release is a different resource. *Fix:* scope the grant to cover the base function ARN **and** its published versions (`function:name` and `function:name:*` — the wildcard form does not cover the unqualified ARN). → `iam.md`.

## What does *not* follow from this model

Stated explicitly, because these are easy to over-generalize from Lambda:

- **"Serverless Worker" does not imply a per-invocation lifetime.** A provider that scales a pool of long-lived instances is still a Serverless Worker driven by the same WCI, with the same Worker Deployment Versioning, and none of the constraints above apply to it in the same form.
- **The shutdown deadline buffer is a property of the Lambda Worker packages**, not of Temporal.
- **Connection-per-invocation is a Lambda property.** Anything reasoning from "the connection is not persistent" — eager Activities being the example above — needs rechecking against a provider that holds the connection for an instance's lifetime.
