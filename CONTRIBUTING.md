# Contributing to the temporal-serverless skill

These are lessons from adding GCP Cloud Run next to AWS Lambda in [skill-temporal-serverless](https://github.com/temporalio/skill-temporal-serverless), written for anyone changing the skill.

The rules apply to every change: a fix to an existing provider, a new SDK guide, or an edit to `SKILL.md`. The last section adds what you need when you add a compute provider. If you work through the [checklist](#before-you-request-review) at the end before asking for review, most review back-and-forth can be skipped.

**In short**

1. Write rules for the agent, not notes about your test runs or your edits.
2. Wait on state you can observe, and stop after a bounded number of retries. Give an exact duration only when it is documented or configured.
3. Never encode one agent harness, one menu shape, or a link to something unmerged.
4. The agent never reads, prints or passes secrets.
5. Teardown removes only what this run created and nothing still uses.
6. When a reviewer flags a pattern, fix every occurrence, not only the flagged line.
7. Put provider material in `references/<provider>/`. Shared files hold only what is true for every provider.

## Writing durable instructions

Most review comments fell here. A skill reference is read by an agent on every future run, so each sentence should be an instruction that will still be true later.

### 1. No authoring narration

> **Flagged:** "now that there are two", "keep the existing Lambda path unchanged", "observed with the xxx tcld build".

These describe your edit, not the system. Put them in the commit message or PR description.

### 2. Gate on observable state, not timings or exit codes

> **Flagged:** Setup and diagnostics read like a record of individual runs: "binding takes one to five minutes", "WCI closes after about 40 seconds", "IAM failures at 41–60 seconds", "drainage takes about three minutes", and exporter timings in an SDK guide. Readers take these as supported thresholds.

Wait on something the agent can read back (the Task Queue binding, WCI state, the pool's requested count, an IAM read-back, drainage status), and set a stopping point: "if it hasn't changed after several minutes, stop and inspect X". A stopping point can be loose like this: it says when to stop waiting and look, not how long the operation takes. Exact durations stay only when they are documented or configured, such as a 90-second stabilization default. Keep the run evidence in the commit.

The same applies to every checkpoint a guide defines. Name the state that proves the stage worked, the command that reads it, and what its output looks like; a zero exit code is not enough. Say what a check does not prove: a pool can be Ready at zero instances, so Ready proves neither that the container started nor that the Worker registered (see `references/gcp-cloud-run/setup.md`). Check the image by digest, the Task Queue types actually bound, and a startup line with the deployment name, build ID and Task Queue. Label a warm-path check and a scale-from-zero check separately, and mark slow checks optional.

### 3. Don't encode one harness or one UI shape

> **Flagged:** "(in Claude Code: `! tcld login`)", "the question tool takes at most four questions", and later "the three most likely Namespaces plus Other".

State the intent and let the agent's environment decide the mechanics: "ask the user to run `tcld login` in their own terminal". For choices, offer all eligible options. If they don't all fit in a structured question, print the full numbered list and ask for a number or name. Never show only a subset.

### 4. Link only to stable, merged sources

> **Flagged:** "a complete sample is under review" in three SDK guides, and a link to an open docs PR.

Link merged samples and published docs, or leave the sentence out until they exist. When you reuse a sample, say exactly which part to take. The Cloud Run guide takes only the Collector config from the Go sample, because the sample's Worker Pool manifest uses a mutable tag and is unversioned.

### 5. Reduce debugging notes to the rule

> **Flagged:** One `SKILL.md` paragraph covered a possible browser sign-in, an agent-shell hang, a 20-second cutoff, the missing `timeout` command on macOS, and version flags. "It reads more like accumulated debugging notes than a crisp workflow instruction."

Keep the behavior: warn that a login may open, have the user log in first, and stop and ask if a call stalls. Move any CLI or platform detail that is still needed into the provider's `setup.md`.

### 6. Say exactly where a value comes from

> **Flagged (offline):** "Take `impersonator_service_account_emails` from the UI template" was too vague: which page, paste what, and should the whole template be applied?

Give the click path. Say which field to take and which values not to apply, and why. Give a format check, such as `serverless-<account>@…`, and have the agent show the result back to the user to confirm. Say whether the pasted content contains credentials.

### Rewriting these lines

When you find a line like this, keep the instruction it carries and move the rest to the commit message. If nothing is left once the narration is gone, delete the line.

| Kind | Before | After |
|---|---|---|
| Authoring narration | "The inventory above is the unchanged AWS Lambda path; now that there are two providers…" | A separate `### GCP Cloud Run` subsection that only lists what Cloud Run needs. The narration goes in the PR description. |
| Test-run timing as a wait | "Initial binding took one to five minutes in test runs." | "If the count stays at `1` and no startup log appears for several minutes, stop waiting and inspect the pool's provisioning state." |
| Test-run timing inside a rule | "The exporter took about 7 s, and 29 s with the default…" | "`shutdown(timeout)` bounds only the force-flush. This exporter reads `OTEL_EXPORTER_OTLP_TIMEOUT` in seconds; set it to `1`." |
| Harness-specific wording | "(in Claude Code: `! tcld login`)" | "Ask the user to run `tcld login` in their own terminal." |
| Tool limit or menu shape | "The question tool takes at most four questions… offer the three most likely Namespaces plus Other." | "If all eligible Namespaces fit in a structured question, offer all of them. Otherwise print the complete numbered list and ask for the number or name." |
| Unstable link or status | "A complete sample is under review in `<open PR link>`." | Link the merged sample, or delete the sentence until one exists. |
| Build-specific observation | "Observed with the 2026-02-17 `tcld` build." | Delete it. Keep a version only when it pins a fact you checked: "CLI v1.8.2 reports the current version under these field names." |

**What to keep**

- **Durations:** keep exact values only when they are documented or configured, such as the 90 s stabilization default. A loose stopping point on a state check ("several minutes") is fine.
- **Versions:** keep only when they pin a verified fact about output or flags.
- **Run evidence** (timings, run IDs, dates): put it in the commit message, where reviewers can still see it.

## Correctness and safety

### 7. Teardown removes only what this run owns

> **Flagged (high priority):** Setup allows reusing an existing runner identity and secret, and the secret-access grant is idempotent. Teardown still removed the binding, which would have cut secret access for every other pool using that runner.

Before each grant or create, check whether it already exists, and record whether this run made it in the deployment inventory. At teardown, list what still depends on the resource. Remove it only if this run created it and nothing else uses it. This applies to any shared identity, role, binding or secret.

### 8. Scope diagnostic queries to the exact resource

Log and audit queries filter on the resource name and its region or location. A project can have same-named resources in several regions, and a query on the name alone mixes their logs into the diagnosis. Check the label names in the provider's monitored-resource docs.

### 9. Check field names and output shapes against the source

Commands that parse CLI output (`jq` paths, field names) must match the provider's documented API shape, which can differ between API versions. The documented worker-pool shape differs between the v1 and v2 APIs, so the teardown filter accepts both paths. Note the CLI version you checked against in the commit. Test filters on real output before merging.

### 10. Secrets stay out of the agent and have a full lifecycle

Creating an API key and handing it off happen in the user's own terminal, straight into the secret store or CLI profile. Never tell the agent to print, read back or pass key material, including through "access secret version" or "get the profile's key" commands.

Document each credential's whole lifecycle: where it comes from, how it reaches the Worker, and how it is rotated or revoked. Secret-entry commands must store the exact bytes, with no trailing newline. Where the secret store has immutable versions, a fix is a new version, not an edit. Keep the Worker's runtime identity separate from the identity Temporal uses to manage the compute; they need different permissions and fail differently.

### 11. Design diagnostics around evidence

Give diagnosis one entry point: `SKILL.md`, the shared WCI reference and the provider's `diagnostics.md` should not each name a different first command. Write the decision table so the available evidence tells the rows apart. A nonzero requested count with no container startup log is a provisioning problem, not an image mismatch, so send the reader to image or identity checks only once the evidence for that branch exists. Test at least one failure path by injecting a fault, such as an image whose build ID does not match the registered version, and record what the healthy and failing states look like. Do not say a repair re-runs registration or a bootstrap unless you tested it.

## Validation and review process

### 12. Run every SDK guide you change, as written

Follow the guide exactly: build, image, deploy, register, check the Task Queue binding, run a Workflow, then scale in or shut down. Each run on the Cloud Run guides found SDK-specific problems no read-through caught: a missing `defaultVersioningBehavior`, a builder image older than `go.mod`, an exporter timeout in the wrong unit, a misleading Java log signature. Start the Workflow type the deployed sample actually registers; the type names differ by SDK.

Inspect the installed CLI and SDK APIs before documenting them. Preview APIs and flags change, and a plausible snippet is not evidence until it builds and runs at the versions the guide pins. For a new provider, run the evaluation with a fresh agent given only the skill files, and record every outside lookup it needed as a documentation gap. Cover a Workflow on a running instance, a scale-from-zero run where practical, an update and rollback, and one deliberate failure diagnosed from the docs.

### 13. Review sentence by sentence, not only for facts

Checking that facts agree, links resolve and code builds did not catch narration or test-run timings. Several of those came back in fix commits. Before asking for review, read `SKILL.md` and your references top to bottom and ask of each sentence: is this a lasting instruction for the agent? When you search for a flagged pattern (lesson 14), read every matching line in full: a configured value at the start of a line can hide test-run timings later in the same line.

### 14. Fix the pattern, not the line

> **Flagged twice:** After the four-question limit was removed, the same rule came back as "three most likely plus Other". After narration was removed in one place, a Claude Code parenthetical was left in another.

When a comment names a type of problem, search every file for it, fix all matches, and say in the reply that you did.

### 15. Keep the PR scoped

> **Flagged:** "There are a lot of changes to SKILL.md and README.md and others that are not related to the GCP updates. Are they intentionally there?"

Leave another provider's wording alone unless your change needs it. If a shared refactor is needed, put it in its own PR or commit and say why in the description. Reviewers can then check it without reading a new provider at the same time.

### 16. Stacked PRs: fix low, merge up

For a large change, a stack works well: provider references first, then SDK guides, then the SKILL and README wiring, with a shared-structure cleanup on top. Commit each fix on the lowest branch that owns the file. Merge it up the stack with merge commits and push fast-forward only, so every PR shows the same state. After merging a fix up, re-check each branch for stale support statements, broken links and duplicated blocks. Reply to each comment with what changed in one or two lines, and resolve the comment after the push.

## Adding a compute provider

A provider is supported when a fresh agent, using only the skill, can deploy, verify, diagnose, update and safely remove a Worker on it, without borrowing assumptions from another provider.

### The layout you are extending

Before you add anything, read how the existing providers are wired. Copy that shape, and say why if you depart from it.

| Path | Holds | Rule |
|---|---|---|
| `SKILL.md` | Workflow, safety gates, routing | Provider-neutral steps. Provider-specific parts go under `### <Provider>` subsections (steps 3–8, principles, troubleshooting, pitfalls, routing). Add a row to the support table. |
| `references/concepts.md` | Shared serverless concepts | Shared content only. The Lambda invocation model lives in `aws-lambda/constraints.md`. |
| `references/wci.md` | WCI lifecycle and inspection | Shared by every provider. Each command must work under each provider's CLI connection. |
| `references/<provider>/` | `constraints`, `setup`, `iam`, `versioning`, `diagnostics`, `observability`, `self-hosted`, plus one `sdk-<lang>.md` for each SDK the provider supports | Same file set as the other providers. `constraints.md` is the execution model, and it is the first file the agent reads for the provider. |
| `README.md` | Human entry point | Prerequisites and support rows split by provider. |

### 17. Shared files must stay provider-neutral

> **Flagged:** `concepts.md` was described as shared but was mostly the Lambda invocation model. Later, a shared "compute provider" definition kept the word "invoke", which only fits Lambda. Cloud Run sizes a pool; it does not invoke.

If a section is only true for one provider, move it to that provider's directory. Don't keep it in a shared file with a "this does not apply to X" caveat. When you move text into a shared section, reread every term for whether it holds for all providers.

### 18. Keep one source for shared mechanics

> **Flagged:** WCI behavior was described in both providers' references. The reviewer asked to pull it into one shared reference that both link to, which also keeps the context the agent loads small.

Before writing something, search `SKILL.md`, the shared references and the other provider directories for the same subject. Decide whether it is a Temporal fact (shared reference), a provider fact (`references/<provider>/`), or a provider exception (a short note next to a link to the shared rule). Don't copy command blocks between files; copies drift.

### 19. Shared commands must work for every provider's connection

> **Flagged:** The WCI inspection commands left out `--profile`, but the Cloud Run setup writes credentials to a named `temporal` CLI profile. The obvious fix, hard-coding `--profile`, would have broken Lambda users who configure the CLI through environment variables: the CLI fails with "unable to find profile".

Keep shared commands neutral. Add one sentence that says how each provider connects. Before changing anything in `wci.md` or `SKILL.md`, check that it still works on the existing providers.

### 20. Use provider subsections, not prose about the other provider

> **Flagged:** Sentences like "the inventory above is the unchanged AWS Lambda path" and "Cloud Run does not carry the Lambda invocation rules".

Format it as `### AWS Lambda` … `### GCP Cloud Run` … and in each section say only what that provider needs. Don't describe your provider by contrast with another one.

### 21. Find every "only provider" claim

> **Flagged:** Three separate "this is wrong" comments: `README.md`, `SKILL.md` and `concepts.md` each still said Lambda was the only supported provider. The README prerequisites also read as if AWS were needed for both providers.

Search the whole repo for statements about which providers exist, and update the support table, the overview text and the prerequisites together. Split prerequisites into per-provider sub-lists. Turn support on last: change the support table, routing and README in the final change, once the whole path works, and use one support status term (such as Public Preview) everywhere the provider is described.

### 22. Triggers should be Temporal-specific

> **Flagged:** Frontmatter triggers like "Worker Pool", "gcloud run worker-pools" and "invoker service account" would fire the skill on general provider work that has nothing to do with Temporal.

Add only triggers that pair the provider with Temporal, for example "deploy Temporal worker on Cloud Run".

### 23. Derive the provider; don't ask for it

The Namespace decides the provider: `tcld namespace get` gives `.spec.regionId.provider`, and the region prefix (`aws-` or `gcp-`) works as a fallback. The skill states the derived provider and asks only for self-hosted Temporal. A new provider should fit this rule: name the Namespace field or region prefix that identifies it, and add any eligibility filter, such as Cloud Run's API-key-only Namespaces. Give the reason for each ineligible Namespace.

## Before you request review

**Every change**

Ordered by impact: safety first, then whether the steps work, then how they read.

- [ ] **No secrets pass through the agent** (lesson 10). No step has the agent read, print or pass key material, including through "access secret version" or "get the profile's key" commands. Key creation and hand-off happen in the user's own terminal, straight into the secret store or CLI profile.
- [ ] **Teardown removes only what this run owns** (lesson 7). Before each create or grant, setup checks whether it already exists and records in the inventory whether this run made it. Teardown lists what still uses a shared identity, role, binding or secret, and removes it only if this run created it and nothing else depends on it.
- [ ] **Every SDK guide you changed was run end to end as written** (lesson 12): build, image, deploy, register, Task Queue binding, a Workflow of the type the sample registers, then scale-in or shutdown.
- [ ] **Waits and checkpoints read state** (lesson 2). Every step that waits says what to check, such as "the version shows both Task Queue bindings", and when to stop waiting and go to diagnostics. It never says "wait two minutes". Every "did this work" step reads back state, such as a binding, a startup log line or an image digest, rather than trusting an exit code.
- [ ] **Queries and parsing match reality** (lessons 8 and 9). Every log or audit query filters on the resource name and its region, so a same-named resource in another region can't mix into the results. Every `jq` path or field name was tested against real CLI output, not only written from the API docs.
- [ ] **Nothing goes stale** (lessons 1–5). There's no narration about your edit ("now that there are two"), no test-run durations, no harness names ("in Claude Code"), no fixed menu shapes ("four questions", "three most likely plus Other"), no "under review" text and no links to open PRs.
- [ ] **Instructions are complete and exact** (lessons 6, 13 and 14). You read every changed sentence and asked whether it is a lasting instruction for the agent. For each pattern a reviewer flagged, you searched every file and fixed every match. Wherever the agent copies a value from somewhere, such as a field in a Cloud UI template, the doc gives the exact place, which field to take, and what not to take.
- [ ] **The change stays in scope** (lesson 15). No other provider's text changed unless your change needs it, and any shared refactor is called out in the PR description.

**Also, for a new provider**

- [ ] **Every credential has a full lifecycle** (lesson 10). The provider's `setup.md` and `iam.md` say where each credential comes from, how it reaches the Worker without passing through the agent, and how it is rotated or revoked. Secret entry stores the exact bytes, with no trailing newline. The Worker's runtime identity and the identity Temporal uses to manage the compute are separate, each with only the permissions it needs.
- [ ] **Existing providers still work** (lesson 19). Every command in a shared file (`SKILL.md`, `wci.md`) still runs under the existing providers' CLI connections. For example, a hard-coded `--profile` would break a CLI configured through environment variables. Any provider-specific connection detail is one sentence beside the neutral command.
- [ ] **Every advertised SDK guide was run end to end** (lesson 12). Each SDK the support table lists for the provider was deployed by following its guide exactly, at least once by a fresh agent given only the skill files, with every outside lookup it needed fixed as a documentation gap. The runs covered a Workflow on a running instance, a scale-from-zero run where practical, an update and rollback, and one deliberate failure.
- [ ] **Each SDK guide is complete** (lessons 2 and 12). It pins the SDK version; registers the Workflow and Activity that verification starts; sets the deployment name and build ID so they cannot stay at sample values; has a Dockerfile or equivalent with target architecture and an ignore file; prints a startup line with deployment name, build ID and Task Queue; and sets Activity heartbeat, timeout and retry where scale-in can interrupt an Activity.
- [ ] **Diagnostics lead with evidence** (lesson 11). There is one entry point that `SKILL.md`, `wci.md` and the provider's `diagnostics.md` agree on. Each decision-table row can be told apart from the evidence the agent has at that point. At least one failure path, such as an image whose build ID doesn't match the registered version, was tested by injecting the fault.
- [ ] **The agent picks the provider correctly** (lessons 22 and 23). The skill derives the provider from the Namespace's cloud provider field or region prefix, and gives the reason for each ineligible Namespace. New frontmatter triggers pair the provider with Temporal, so the skill doesn't fire on general work with that provider.
- [ ] **Shared files stay shared** (lessons 17 and 18). `concepts.md`, `wci.md` and the provider-neutral parts of `SKILL.md` contain no provider-only terms and no caveats about one provider. Shared mechanics, such as WCI commands, are written once in a shared reference, and provider files link to it rather than copying it.
- [ ] **The provider is wired in like the others, and support is turned on last** (lessons 20 and 21). `references/<provider>/` has the full file set. `SKILL.md` has `### <Provider>` subsections wherever the existing providers have them. The README has per-provider prerequisites. The support table, routing and README change in the final change, once everything above passes. The support row names the supported SDKs if they are fewer than all of them, and the provider's support status uses the same term everywhere.
