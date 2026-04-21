# Three AI Agents Wrote, Reviewed, and Re-Reviewed My Infra PR. I Still Had to Catch the Worst Bugs Myself.

*This is the twenty-fifth post in a series about learning Kubernetes by building FeedForge — an RSS feed aggregator with AI summarization on GKE. These posts are learning notes from someone figuring things out in real time. [Previous post here.](https://medium.com/@huchka)*

---

> The PR this post is about: [feedforge#25](https://github.com/huchka/feedforge/pull/25). Every diff, every review comment, every revision commit is browsable there.

A few weeks ago I wanted to ship a security hardening PR for FeedForge: turn off the `automountServiceAccountToken` defaults that many Kubernetes audit tools flag, and migrate Postgres credentials from a manually-created Kubernetes Secret to GCP Secret Manager via the Secrets Store CSI Driver. Real, non-trivial work that touches Terraform, Kubernetes manifests, documentation, and cluster bootstrap scripts.

I didn't write a single line of it. I orchestrated three AI agents against the task and watched them hand work off to each other. And at the end — after two of them had agreed the PR was ready to merge — I still had to pull the branch, deploy it by hand, and discover the bugs that only show up at runtime.

This post is about that workflow: what each agent contributed, what each one missed, and what the experience taught me about where humans still have to stay in the loop on infrastructure changes.

## The workflow

Here's what actually happened:

1. **I wrote two GitHub issues** describing the security hardening work (`#20: disable automountServiceAccountToken`, `#21: migrate secrets to GCP Secret Manager`), both as subtasks of an epic (`#24: security hardening`). Each issue had acceptance criteria, labels, priorities, and size estimates.
2. **I connected the repo to Perplexity** and ran it in their agentic sandbox environment. The entire prompt was:
   > as senior software engineer, please work on feedforge's issue #20, and #21 which are subtasks of #24. please work on the issues and then create PR
3. **Perplexity's agent read the issues, wrote the code, and opened [PR #25](https://github.com/huchka/feedforge/pull/25).** Twenty-one files changed, 458 lines added, 3 lines deleted. Full `docs/secret-manager.md`. The whole thing.
4. **Claude Code reviewed the PR.** I opened a new session, pointed Claude Code at PR #25, and asked for a thorough review. It posted a structured review comment with four blockers and two non-blocking gaps.
5. **I sent the review back to Perplexity** — same agent, new task, "please address the review feedback." Six commits landed within an hour, each addressing one finding.
6. **Claude Code re-reviewed.** All blockers resolved. PR marked ready to merge.
7. **Then I pulled the branch and tried to actually deploy it.** And that's when the bugs the AIs hadn't caught came out.

There was also a fourth AI in the mix that I didn't invite. I had recently connected Devin.ai to the repo to try it out, and its auto-review triggered on PR #25 the moment it was opened. It posted its own review, flagged one issue in-line and five more on its platform. I'll discuss it at a high level later (most of its findings live behind its platform paywall, so I can't reproduce them in detail here) and explain why I ended up turning off its auto-review.

## Perplexity: the author

The prompt was one sentence. It specified two issues and a parent epic, and it said "work on the issues and then create PR." Nothing about the existing project structure, no pointers to the relevant files, no constraints about Terraform vs Helm vs kustomize.

The agent came back with code that *matched the existing project style*. It used the same Terraform module structure. It added labels consistent with the rest of the manifests. It wrote a `docs/secret-manager.md` with architecture diagrams and a troubleshooting section. The PR description enumerated acceptance criteria it had satisfied, listed follow-up actions for the deployer, and linked the closed issues via `Closes #20, Closes #21`.

Much of the per-file code looked plausible on a first read. The `google_secret_manager_secret_iam_member` Terraform resources used a `flatten` + nested `for_each` pattern — the canonical "bind many to many" idiom. The CSI driver install commands were current. The `SecretProviderClass` manifests used the right API version.

This part of the workflow genuinely worked. I was not paying attention to any of it — I was doing something else — and when I came back there was a branch with a structurally plausible PR on it.

That last fact is also the thing that worries me about Perplexity a little. In the agent mode I used, it never asked me anything. No "I see two ways to wire this up, which one do you want?", no "I'm about to delete the `k8s/base/postgres/` directory, is that intentional?", no mid-task check-in. It read the issues, made every design decision — which CSI driver to use, whether to add `secretObjects`, whether to retain the old backup `Job`, how to split the Terraform modules — and handed me a finished PR. The author experience felt genuinely black-box in a way Claude Code sessions don't, where I'm used to a conversational back-and-forth with proposed tradeoffs and clarifying questions. It worked out fine this time because my issues had thorough acceptance criteria and the agent happened to pick reasonable defaults. But the takeaway is uncomfortable: the less interactive the agent, the more load-bearing the initial prompt becomes. For the mode I used, I'd now write a much more explicit prompt up front — "use the OSS CSI driver not the managed add-on," "don't touch files under `k8s/base/postgres/` or `k8s/base/backup/`," "put new platform dependencies in `k8s/bootstrap/`" — because the agent didn't pause to ask me about any of those choices.

## Claude Code: the reviewer

Then the gap showed up. I opened a Claude Code session and asked it to review PR #25 carefully. Within a few minutes it had flagged six issues across two categories:

**Four blockers** that would have made the PR fail to deploy:

1. The Kubernetes ServiceAccount patch annotated `feedforge/db-backup` with the GCP SA `feedforge-db-backup@...`, but **that GSA didn't exist in Terraform**. No `google_service_account` resource, no IAM grants, nothing. The KSA→GSA mapping chain was incomplete, so the workload would fail to obtain a valid GCP identity and any Secret Manager call would error out with `PermissionDenied`.
2. The Terraform `google_secret_manager_secret_iam_member` resources granted access to secrets **by name** — but no one had created the actual secrets in GCP Secret Manager yet. `terraform apply` would error out trying to add IAM to resources that didn't exist.
3. The Secret Manager API wasn't enabled. No `google_project_service` resource to enable `secretmanager.googleapis.com`. On a fresh project, the apply would fail on the first Secret Manager call.
4. The CSI driver install was documentation-only. Commands in a markdown file, no script, no Helm values, nothing in `k8s/bootstrap/`. Fine on the first deploy if you read carefully; bad for repeatability.

**Two non-blocking gaps:**

5. The `PROJECT_ID` placeholder in the `SecretProviderClass` manifests required manual find-replace per environment — no kustomize substitution strategy.
6. The PR edited `k8s/base/postgres/` files that were orphan dead code (postgres was migrated to Cloud SQL three phases ago). Harmless diffs, but noise.

Every single one of these was at a **seam between two systems**: K8s annotation ↔ Terraform GSA definition, Terraform IAM grant ↔ Secret Manager resource existence, Terraform ↔ project-level API enablement, documentation ↔ bootstrap automation, manifest file ↔ kustomize build graph.

Within any one file the Perplexity agent had been accurate. Across boundaries, the chain of references hadn't been validated. Claude Code, reviewing fresh, had no trouble pattern-matching these as standard cross-layer omissions.

I posted the review as a PR comment with the structure: `summary → verified → blockers → gaps → recommended path`.

## Round two: Perplexity addresses the feedback

I copied the review URL back into Perplexity and said "please address the feedback." An hour later there were six new commits on the branch:

```
2be1392 fix: add WI bindings and Secret Manager IAM grants in Terraform
d705fc1 fix: add CSI Secrets Store bootstrap script
7ac8948 fix: use kustomize overlay patches for PROJECT_ID
92a2bfb fix: remove orphaned k8s/base/postgres directory
caed888 fix: add optional: true to secretKeyRef for CSI-synced secrets
77762b2 docs: update secret-manager.md and README for review feedback
```

One commit per finding. A response comment on the PR broke down which commit solved which finding. Claude Code re-reviewed and confirmed all blockers were resolved. This loop — AI authors, AI reviews, AI fixes, AI re-reviews — took about two hours of wall-clock time and roughly five minutes of my attention.

On paper the PR was ready to merge.

## The human lap

Then I tried to deploy it.

First `kubectl apply -k k8s/overlays/dev/`:

```
error: no resource matches strategic merge patch "ServiceAccount.v1.[noGrp]/db-backup.feedforge":
no matches for Id ServiceAccount.v1.[noGrp]/db-backup.feedforge
```

The overlay was patching a `db-backup` ServiceAccount that wasn't in the base kustomization. Everyone had agreed `k8s/base/postgres/` was dead code and deleted it. Nobody had noticed that `k8s/base/backup/` — a sibling orphan directory — was still being referenced by an overlay patch without being in `kustomization.yaml`.

Reading the code: the backup `Job` pointed at `PGHOST: postgres` (the in-cluster service that had been deleted in the Cloud SQL migration), had no `cloud-sql-proxy` sidecar, and hadn't been functional for months. It was broken dead code next to other dead code. The hardening PR had applied the same automount/CSI changes to *all* of it uniformly, including the bits that don't run.

Fix: delete `k8s/base/backup/`, remove the overlay patch, remove the Terraform GSA that had been added for it. A 9-file cleanup commit.

Second bug, next apply:

```
Error: notification-credentials volume mount failed:
rpc error: code = NotFound desc = secret version "projects/.../feedforge-line-channel-token/versions/latest"
does not exist
```

Three of the five Secret Manager secrets had been created (Perplexity's docs had walked me through that) but no versions had been added to them. The mount failed because the referenced secrets existed but had no enabled versions yet — the CSI driver can't fetch content from a secret that has no version to read. The `digest` CronJob mounts the notification volume, so on first run it crashed.

Fix: add a version to each of the three notification secrets — even a placeholder value works, as long as an enabled version exists. Or populate them with real LINE / webhook values. Either unblocked the mount.

Third issue, not a bug but surprising: cold-start crashloop. The CSI mount creates a K8s Secret asynchronously via the driver's sync controller. The app container starts at roughly the same time. For about 30 seconds after a fresh deploy, pods flapped between `Error` and `CrashLoopBackOff` because the synced Secret didn't exist yet when kubelet tried to resolve the env var. The `optional: true` flag that Perplexity had added in response to my review softened the failure mode but didn't actually eliminate it — the container starts with an empty password, fails at DB connect, and loops. I filed a follow-up issue to read from the mounted file directly, which is the truly race-free pattern.

None of these three showed up in any AI review. You only see them when the pod actually boots.

## Devin: the uninvited reviewer

While all this was happening, Devin.ai had auto-reviewed the PR. Its comment appeared minutes after the PR was opened:

> **Devin Review** found 1 potential issue.
> View 5 additional findings in Devin Review.

The full findings were behind a link to Devin's platform, not inlined on GitHub. I only looked after everything else was settled. One of the findings was a **real bug** — something neither Perplexity's author nor Claude Code's reviewer had caught. I'm keeping the specifics intentionally vague because the full Devin review is paywalled and I'd rather not paraphrase it with errors, but it was a legitimate pre-existing code bug that the hardening PR happened to touch adjacent code to.

The fact that Devin caught something both other AIs missed is the interesting part. Three AIs, three different failure profiles.

The less-fun part: I had to turn Devin's auto-review off. I hadn't asked it to review every PR on my user activity — it had started triggering on any PR in the repo. For a personal learning project where I'm intentionally sending work through Perplexity and then Claude Code, an unrequested third review stream adds noise. The signal was valuable; the cadence wasn't configurable in a way that matched my workflow. So I turned it off, figuring I'd re-enable it for specific PRs I wanted a fourth opinion on.

## What each of them did well, and what each missed

Putting the four perspectives on one table:

| Perspective | Strong at | Missed |
|---|---|---|
| **Perplexity (author)** | Following the spec, matching project style, writing docs, producing structurally coherent code inside each file | Cross-layer name references; orphan code awareness; prerequisite ordering; repeatability concerns |
| **Claude Code (review)** | Spotting cross-layer mismatches, system-boundary issues, the Terraform ↔ K8s ↔ IAM chain | Runtime-only behavior (the "works on paper, crashloops on first deploy" class); the backup-dir orphan sibling |
| **Devin (auto-review)** | Code-pattern bugs in adjacent code that the hardening touched | N/A to me — I didn't invite it and the full review lived off-GitHub |
| **Human (me)** | Actually running the thing. Integration testing. The "is this a workable deploy flow for a fresh environment" check | Structured code review (it's faster to have an LLM read 458 lines than for me to); Terraform idioms I'm not fluent in |

The interesting pattern: **each AI added value but each had a blind spot, and at least one bug was caught by only one of the four reviewers**. Not by the "best" one, just by whichever happened to pattern-match the issue. That implies the ensemble is genuinely more thorough than any single agent — and also that no number of AI reviewers replaces the human act of running the code.

## A checklist for reviewing AI-written infra PRs

Six rules that would have caught every bug I found. Many platform teams already do some of these, but they're easy to skip in normal code review and even easier to skip when an AI wrote the PR:

1. **Cross-reference every name.** For every string identifier the PR adds — ServiceAccount names, GCP SA emails, Secret names, ConfigMap references — grep the repo. Does the referenced resource exist in code? If it's external, is there a reproducible step that creates it?
2. **Walk a clean-environment apply mentally.** Imagine a fresh GCP project and empty cluster. Does every resource reference something that will exist at the moment it's created? Are any `depends_on` missing?
3. **Check the build graph.** For every K8s manifest the PR edits or adds, confirm it's actually in a `kustomization.yaml` (or your deploy tool's equivalent). Orphan edits are waste at best and confusing noise at worst.
4. **Is every new platform dependency captured as code?** Prose rots. Install commands buried in documentation get skipped on the second deploy.
5. **Scrutinize behavior claims in the PR description as much as the code.** "This flag fixes the race" deserves a trace through actual pod startup sequence. Correct-sounding claims don't always hold up.
6. **Pull the branch and actually run it.** Especially on infra PRs, static review can catch some pre-deploy problems but not runtime behavior. The backup-sa-patch error was trivially obvious at apply time and invisible in the diff. You find those by running the thing, not by reading it.

## Things I Learned

- **In this case, the authoring agent benefited from an external review much more than from its own first pass.** Perplexity wrote a coherent PR. When I gave it the review, it fixed everything I'd flagged. But it didn't catch the same issues on its own the first time. The value of a second pair of eyes — even AI eyes from a different system — was real here. I'd want to test the pattern on more PRs before calling it a general rule.

- **Cross-system reference checks are the single highest-value review step for AI-generated PRs.** Within a file, AI-generated code is often better than mine. Across files and across systems (Terraform ↔ Kubernetes ↔ cloud IAM), the hit rate drops sharply. That's the spot where a human grepping the repo still matters.

- **Runtime-only bugs are invisible to static review.** The backup-SA orphan, the missing Secret Manager version causing a mount failure, the cold-start crashloop — none of these appear in the diff. Static review can catch some pre-deploy problems, but it can't prove that `kubectl apply` will succeed or that pods will actually come up healthy. Those are runtime questions, and the only way to answer them is to deploy.

- **In this workflow, multiple reviewers caught non-overlapping issues that any one reviewer alone missed.** Devin surfaced something both Perplexity and Claude Code hadn't flagged. Claude Code caught cross-layer issues Perplexity hadn't noticed. Perplexity produced the original draft neither of the others would have. Three imperfect reviewers with non-overlapping blind spots was more thorough than any one alone — though I want to see more data points before generalizing this as "always use multiple agents."

- **But more review isn't free.** Devin's auto-review was valuable and also noisy. For a project where I was already orchestrating two other AIs, a third uninvited agent was friction. I'd love a "review when I ask" mode more than a "review everything" mode. Knowing when to disable helpful automation is part of using it well.

- **The prompt was one sentence.** "as senior software engineer, please work on feedforge's issue #20, and #21 which are subtasks of #24. please work on the issues and then create PR." No project context, no file pointers, no constraints. That produced a 458-line PR that was mostly right. This is wild to me. The failure modes are real — that's what the rest of this post is about — but the magnitude of what a one-sentence prompt can do is worth naming.

- **Non-interactive agents amplify the weight of the prompt.** Perplexity never asked a clarifying question. It made every design decision autonomously and handed me a finished PR. That worked because the issues had thorough acceptance criteria and the defaults it picked happened to be reasonable. But the failure mode is obvious: if the agent doesn't pause to ask, and the prompt doesn't nail down your constraints, you'll discover disagreements only after the work is done — when changing direction is expensive. Interactive agents like Claude Code trade speed for confidence; they propose tradeoffs, pause, and check in. Know which mode you're in before you kick off. A one-sentence prompt was sufficient in this case, but non-interactive agents make omissions more expensive because they may not pause to clarify. Front-load the constraints.

- **The human is still doing the integration test.** I don't think this is going away soon. Pulling the branch, running `terraform apply`, watching pods boot — that's the part where the abstract "code looks right" becomes "system works." No review agent does that today. I expect that to remain the last thing to automate, because it requires a real cluster with real credentials doing real things.
