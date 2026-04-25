# Antigravity Wrote the PR. I Reviewed It. Then I Gave the Agent Bad Advice.

*This is the twenty-eighth post in a series about learning Kubernetes by building FeedForge — an RSS feed aggregator with AI summarization on GKE. These posts are learning notes from someone figuring things out in real time. [Previous post here.](https://medium.com/@huchka)*

---

> The PR this post is about: [feedforge#33](https://github.com/huchka/feedforge/pull/33). Author: Antigravity, running Gemini 3.1 Pro. Reviewer: Claude Code. Verifier: me, with the cluster.

In the last two posts I wrote about an AI authoring infra PRs and Claude Code reviewing them — first with Perplexity, then with Devin. The conclusion both times was that **the human still does the integration test**. This post is the same loop again, with a different AI on the author side and a twist I didn't see coming: the worst bug in the merged code wasn't the AI's. It was mine. Caught not by me, not by the AI agent, but by `terraform validate`.

The PR addressed three Cloud SQL findings flagged by GCP Security Command Center, refactored a tangled `kustomization.yaml`, and moved Secret Manager containers into Terraform. Three issues bundled, 176 lines added, 71 deleted. Realistic, multi-domain infra work.

## The cycle on this PR

Telegraphing the shape so the rest reads cleanly:

1. **I filed three issues** (`#27`, `#28`, `#29`) with acceptance criteria and labels. They were all `size:S` cleanup work — small enough to bundle.
2. **Antigravity (with Gemini 3.1 Pro) wrote the PR**, but only after walking me through a plan and asking my approval before executing.
3. **I had to ask for a more detailed PR description.** The first draft was thin.
4. **Claude Code reviewed PR #33** and posted a structured comment with five issues plus minor nits.
5. **Antigravity revised the PR**, addressing six of the seven actionable items in commit `3218a34`.
6. **The verification step caught my own mistake.** `terraform validate` failed because of advice *I* had given the agent. The agent had taken my advice exactly as written, and broken the config.
7. **I fixed the regression**, ran `terraform apply`, hit a partial-failure scenario I'd warned the user about myself in the review, recovered with `terraform import`, and verified end-to-end.

Total wall-clock: about 2 hours, most of it waiting for `terraform apply` and reading. About 40 minutes of actual attention.

## What the PR actually changed

Three issues, three different domains, all in one PR. Worth covering each because the technical content is the part you'll search for later.

### Issue #27 — Cloud SQL hardening (defense in depth)

GCP Security Command Center had flagged the Cloud SQL Postgres instance for three things:

```hcl
ip_configuration {
  ipv4_enabled    = false
  private_network = var.network_id
  ssl_mode        = "ENCRYPTED_ONLY"   # ← new
}

database_flags {
  name  = "cloudsql.enable_pgaudit"    # ← new
  value = "on"
}
database_flags {
  name  = "pgaudit.log"                # ← new (and more on this below)
  value = "ddl,write"
}

password_validation_policy {           # ← new
  min_length                  = 12
  complexity                  = "COMPLEXITY_DEFAULT"
  reuse_interval              = 2
  disallow_username_substring = true
  enable_password_policy      = true
}
```

Three things worth knowing about this block:

**SSL enforcement is independent of the proxy.** All FeedForge connections go through the Cloud SQL Auth Proxy sidecar, which already encrypts. So why force `ssl_mode = "ENCRYPTED_ONLY"` on the instance? Because the proxy is one layer; if a future bug or misconfig ever lets a non-proxy client reach the DB, the instance itself should still reject the unencrypted connection. Defense in depth is "make every layer enforce its own invariants."

**pgAudit needs more than one knob.** This is the exact bug I caught in the original PR. The first version only set `cloudsql.enable_pgaudit = on`, which loads the extension's shared library but tells it to log nothing. To actually capture an audit trail you also need `pgaudit.log = ddl,write` (or wider). And on Cloud SQL, the standard pattern is also to run `CREATE EXTENSION pgaudit;` in each database — for the FeedForge instance the extension was already registered, so the missing log flag was the only blocker. After both flags landed and the instance restarted, Cloud Logging showed:

```
LOG: parameter "pgaudit.log" changed to "ddl,write"
LOG: pgaudit extension initialized
```

The flag change triggered ~54 seconds of Cloud SQL update time during apply (the instance restarts to pick up the change to `shared_preload_libraries`).

**Why `ddl,write` and not `all`?** Reads are vastly more frequent than writes — logging every `SELECT` would balloon your Cloud Logging bill. `ddl,write` is a pragmatic low-noise baseline that captures anything that mutates the database. Some compliance frameworks want broader coverage; pick the level that fits your audit requirements.

### Issue #28 — Kustomize refactor

Before: every YAML file listed individually in `k8s/base/kustomization.yaml`.

```yaml
resources:
  - redis/serviceaccount.yaml
  - redis/deployment.yaml
  - redis/service.yaml
  - redis/networkpolicy.yaml
  - backend/serviceaccount.yaml
  - backend/configmap.yaml
  - backend/service.yaml
  ... 40+ entries ...
```

After: directory references with per-component kustomizations.

```yaml
resources:
  - namespace.yaml
  - namespace-quota.yaml
  - namespace-limitrange.yaml
  - redis
  - backend
  - frontend
  ...
```

When kustomize sees `- redis` in `resources:`, it looks for `redis/kustomization.yaml` and recursively builds it. The output gets merged into the parent. Adding a new file to a component now only requires editing that component's kustomization — not finding the right slot in a 60-line root file.

**The verification that proves this refactor is safe**: `kubectl kustomize k8s/base` before and after should produce identical output. I ran it both ways: 2431 lines, 75 resources, byte-for-byte the same. An empty diff in the rendered manifests is a strong safety check for this kind of refactor. (For larger refactors, semantic equivalence is what really matters — render order can shift without changing behavior — so don't take "byte-identical" as a universal rule, just a clean one when you can get it.)

### Issue #29 — Secret Manager containers in Terraform

Before this PR, the secret container lifecycle was split:

1. Run `gcloud secrets create feedforge-postgres-user ...` manually.
2. Then `terraform apply` would grant IAM bindings on those secrets.

This is the **split-lifecycle problem**: Terraform managed *part* of a resource's life (IAM) but not the resource itself (the container). On a fresh project `terraform apply` would fail because the IAM grants reference secrets that don't exist yet. `terraform destroy` would leave orphaned secret containers behind. Drift was invisible to Terraform.

After:

```hcl
locals {
  all_secrets = toset(concat(local.postgres_secret_names, local.notification_secret_names))
}

resource "google_secret_manager_secret" "feedforge_secrets" {
  for_each = local.all_secrets

  project   = var.project_id
  secret_id = each.value

  replication {
    auto {}
  }

  depends_on = [google_project_service.secretmanager]
}
```

Two patterns worth understanding here:

- **`replication { auto {} }`** — GCP requires you to declare replication. `auto {}` means "let Google replicate globally," vs. user-managed replication where you pick regions. Fine for a learning project.
- **No `secret_data` field.** This is intentional. The container goes in Terraform; the *value* is added separately via `gcloud secrets versions add`. If the value were in Terraform, it would sit in plaintext in `terraform.tfstate` — and anyone with read access to local or remote state could retrieve it. Splitting governance (in IaC) from values (in Secret Manager) keeps the secret out of the file you're most likely to copy around carelessly.

The IAM bindings then changed from referencing strings to referencing the resource's `.id`:

```hcl
# Before
secret_id = each.value
# After
secret_id = google_secret_manager_secret.feedforge_secrets[each.value].id
```

This change looks cosmetic. It is not — because the secret resources were new and therefore "known after apply" at plan time, the binding's `secret_id` field couldn't be resolved during planning, which would later cause every IAM binding to be destroyed and recreated. More on that further down.

## What's different about Antigravity

I had not used Antigravity before this PR. The agent is built into an IDE-style environment and runs Gemini 3.1 Pro. Three things stood out about the workflow.

**It plans first, then asks before executing.** When I tagged it on the issues, it didn't immediately start writing code. It produced a plan: "I'll modify these files in this order, here's the strategy for each issue, here's how I'll verify." Only after I approved did it run anything. When it needed to test something, it proposed the command and let me run it. The cadence is closer to pair-programming than to a "submit job, get PR" pattern.

**It feels closer to code than Claude Code does, because it lives in the IDE.** Claude Code in my terminal is conversational; Antigravity is in the file tree, the diff view, the editor. The same model conversation feels different in different surfaces. Neither is better — it's a different ergonomic. Antigravity's surface made me think about specific files and lines more readily. Terminal Claude Code makes me think about the project as a whole more readily.

**I had to nudge it on the PR description.** The first version of the PR description was a one-line "addresses issues 27, 28, 29." I had to specifically ask for a per-issue breakdown of what changed and why. The final description (the one in the PR now) is much more useful — but the agent's default was minimal. Worth knowing if you use this tool: the PR description is something you'll want to ask for explicitly, the same way you'd specify acceptance criteria.

## Round 1: the review

Claude Code's review on PR #33 surfaced five issues:

🔴 **Orphaned `k8s/base/postgres/kustomization.yaml`** — a new file that wasn't actually wired into the parent. Dead code.

🟡 **Doc/code mismatch** — `docs/cloud-sql-security.md` referenced `require_ssl = true` (the deprecated boolean), but the code used `ssl_mode = "ENCRYPTED_ONLY"` (the current enum).

🟡 **pgAudit needs the log flag.** Just `enable_pgaudit` produces an audit trail of nothing. Add `pgaudit.log = ddl,write`.

🟡 **Password substring policy can break rotations.** With `disallow_username_substring = true`, any password containing `feedforge` would be rejected. Worth documenting.

🟡 **Cloud SQL changes trigger an instance update** — verify the plan is `~` (in-place), not `-/+` (replacement), before applying.

And, embarrassingly, a "minor nit" buried at the bottom:

> `enable_password_policy = true` inside `password_validation_policy` is redundant — setting any fields in that block enables the policy. Harmless, just noise.

## Round 2: the agent revised. I broke it.

Antigravity addressed six of the seven items in one commit. Including, faithfully, my "minor nit" — it removed `enable_password_policy = true` because I'd called it redundant.

`terraform validate`:

```
Error: Missing required argument
on .../cloud-sql/main.tf line 32, in resource "google_sql_database_instance" "postgres":
  32:     password_validation_policy {
The argument "enable_password_policy" is required, but no definition was found.
```

The `hashicorp/google` provider (v7.24.0) treats `enable_password_policy` as required inside `password_validation_policy`. My nit was wrong. The agent took my wrong advice exactly as I gave it, the regression went into the PR, and `terraform validate` was the only thing standing between that bug and a broken `apply`.

This is the part of AI-assisted development that's least talked about. Most AI-coding posts focus on the failure mode where the AI is wrong. The failure mode I keep finding more interesting is when the **human reviewer** is wrong, the AI agent dutifully complies, and now both of you are wrong in the same direction. Without an external check — a compiler, a validator, a type system, a test — there's nothing in the loop to catch you. You confirm each other.

`terraform validate` is free, fast, and catches a class of bugs that `terraform plan` won't catch until you've authenticated to GCP and refreshed state. Run it after every refactor.

## The apply: 409s, imports, and a brief permission gap

After the fix, `terraform plan` showed:

```
Plan: 12 to add, 1 to change, 7 to destroy.
```

The 1 change was the Cloud SQL update (in-place — good). The 12 adds were the new secret resources and the IAM bindings being recreated. The 7 destroys were the old IAM bindings. Why were they being destroyed and recreated?

```
~ secret_id = "projects/.../feedforge-postgres-user"  # forces replacement -> (known after apply) # forces replacement
```

Refactoring `secret_id` from a literal string to `google_secret_manager_secret.feedforge_secrets[each.value].id` looks cosmetic. After both apply, the resolved string is identical. But at *plan time* Terraform doesn't know what the new resource's `.id` will be — it shows `(known after apply)`. The provider has marked `secret_id` as a `ForceNew` field, so any change at plan time triggers replacement.

This matters because **destroy-then-recreate is not atomic**. There's a window between the destroy and the recreate where the service accounts technically lack permission to read those secrets.

`terraform apply` then partially failed: Cloud SQL updated successfully (54s, restart for pgaudit), the IAM bindings destroyed, but creating the secret containers failed:

```
Error: Error creating Secret: googleapi: Error 409:
  Secret [projects/.../feedforge-postgres-user] already exists.
```

Of course they did. They were the secrets I'd created manually with `gcloud secrets create` months ago. Terraform now wanted to own them, but state had nothing for them.

The recovery is `terraform import` — tell Terraform "this real-world resource maps to this address in your config":

```bash
terraform import 'google_secret_manager_secret.feedforge_secrets["feedforge-postgres-user"]' \
  projects/${PROJECT_ID}/secrets/feedforge-postgres-user
# repeat for each of the 5 secrets
```

After importing all five, the second `terraform apply` ran clean: the 7 IAM bindings recreated, no secret-creation conflicts, end state consistent. `terraform plan` after that: "No changes. Your infrastructure matches the configuration."

**The brief permission gap turned out not to matter.** The Cloud SQL Auth Proxy reads its identity from the Workload Identity token, not from these specific Secret Manager bindings — its access continued to work. The application pods' DB credentials had already been mounted by the Secrets Store CSI driver into pod filesystems. In this cluster, with rotation not enabled and pods not restarting during the gap, the existing mounts stayed usable throughout.

In a stricter setup — auto-rotation enabled, or a pod restart landing inside the gap — this would have caused a real outage. Worth keeping in mind for any future IAM refactor: read every `forces replacement` line in a plan, and know which of your IAM resources have `ForceNew` fields.

## The GCP Console keeps lying for a day

After `apply` completed, the Cloud SQL Instance Health panel in the GCP console still showed all four warnings:

- Allows unencrypted direct connections
- Auditing not enabled
- No password policy
- No user password policy

But `gcloud sql instances describe` showed every setting active:

```yaml
ipConfiguration:
  sslMode: ENCRYPTED_ONLY
databaseFlags:
- name: pgaudit.log
  value: ddl,write
- name: cloudsql.enable_pgaudit
  value: on
passwordValidationPolicy:
  enablePasswordPolicy: true
  minLength: 12
  ...
```

In my case, three of the four warnings cleared on their own within ~24h.

More generally, the Instance Health panel is powered by the **Recommender API / Security Health Analytics**, whose findings are detector outputs with their own latency and scope: some detectors update on relevant config changes in near-real-time, some are batch-only, and the Standard tier can be much slower than Premium. The takeaway isn't a fixed cadence — it's that you should verify state directly rather than trust the panel.

The fourth warning ("No user password policy") is genuinely a different setting:

| Setting | Controls | Where it lives |
|---|---|---|
| `passwordValidationPolicy` (instance) | Password content rules (length, complexity) | `google_sql_database_instance.settings.password_validation_policy` |
| `passwordPolicy` (per-user) | Account behavior (failed-login lockout, expiration) | `google_sql_user.password_policy` |

If I were to add per-user `password_policy`, a typical config would set something like `allowed_failed_attempts = 5`, which means five failed logins lock the account. For a service-account-style DB user where the password rarely rotates, that's more outage-prone than helpful — if your K8s Secret ever drifts from the DB user, the app will retry on a tight loop and lock itself out within seconds. For FeedForge I left this finding intentionally unaddressed.

The lesson generalizes: **SCC findings are detector outputs with their own latency and scope; verify the underlying resource directly, and decide which findings actually fit your threat model before chasing them all to zero.**

## Things I Learned

- **The most dangerous bug was mine, not the AI's.** I gave the agent a confidently-wrong "nit." The agent accepted it. Without `terraform validate`, that bug would have shipped. The lesson generalizes: when reviewing AI code, your reviewing voice carries more weight than usual because the agent doesn't push back on authoritative-sounding wrong advice. Run a separate check.

- **Plan-first agents change the conversation shape.** Antigravity's plan-then-execute pattern made me read its plan more carefully than I usually read AI output. When the agent proposes the plan, you're reviewing the design before any code exists — which is the cheaper time to catch design problems. Worth pushing other tools toward this cadence too.

- **PR description quality is a deliberate prompt.** The default Antigravity PR description was thin. Asking for a per-issue breakdown got a description I'd actually want to read in six months. The PR description is one of the longest-lived artifacts of a change — it deserves the same care as the code. Ask the agent for it explicitly.

- **`terraform validate` is the cheap preflight.** Free, no provider auth, no state refresh. `terraform plan` does its own schema validation too, but `validate` is the fast first check — it would have caught my mistake instantly if I'd run it before pushing my review comment. After every refactor: `validate` first, then `plan`.

- **Read every `forces replacement` line.** The IAM-binding-refactor case here didn't cause an outage because of CSI mount survivability. In a different setup it absolutely could have. Anything `-/+` deserves a moment of "what's the blast radius if this is destroy-then-create with a gap?"

- **`terraform import` is the right answer when reality already exists.** When `apply` fails with 409 on resources that pre-exist in your cloud, importing is the safe move — no data lost, state catches up to reality. Memorize the syntax; you'll need it on any project that grew before its IaC.

- **SCC findings are detector outputs, not facts about your current state.** Latency and scope vary by detector and tier. Verify resource state directly with `gcloud`, and decide which findings fit your threat model before chasing them all to zero.

- **Two AIs and one human is still the right shape.** Antigravity authored, Claude Code reviewed, I verified in the cluster. The split that emerged organically across the last few PRs — author / reviewer / human-with-the-cluster — keeps producing better outcomes than any single agent doing the whole thing. Each role catches what the others miss. And as this PR showed, the human role is necessary not just to verify the code but to verify their own review.
