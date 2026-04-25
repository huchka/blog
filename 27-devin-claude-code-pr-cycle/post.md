# Devin Wrote the PR. Claude Code Reviewed It. I Merged. Here's What Surprised Me About the Cycle.

*This is the twenty-seventh post in a series about learning Kubernetes by building FeedForge — an RSS feed aggregator with AI summarization on GKE. These posts are learning notes from someone figuring things out in real time. [Previous post here.](https://medium.com/@huchka)*

---

> The PR this post is about: [feedforge#26](https://github.com/huchka/feedforge/pull/26). Merge commit `ae00a99`. Six commits, one design doc on the wiki, two AI agents, one human (me) approving, deploying, and verifying.

A few weeks ago I wrote about [shipping a security hardening PR through three AI agents](https://medium.com/@huchka). The author was Perplexity, the reviewer was Claude Code, and Devin auto-reviewed uninvited from the side. That post ended on the conclusion that **the human still does the integration test**.

This post is the next step. Same structure — AI authors a PR, Claude Code reviews, the human verifies in a real cluster — but two things changed:

1. The author this time is **Devin**, and the workflow with Devin is meaningfully different from Perplexity's.
2. I ran the cycle on a more boring change (CI/CD migration, not security hardening), which let me notice the *workflow ergonomics* as opposed to the *correctness gaps*.

The change itself was migrating FeedForge from Cloud Build to GitHub Actions with Workload Identity Federation. I wrote about the technical depth of WIF in [the previous post](https://medium.com/@huchka). This post is about *how the work got done*: who did what, what surprised me, and where the seams are.

## The cycle on this PR

Telegraphing the whole shape so the rest reads cleanly:

1. **I filed issue #22** with acceptance criteria, labels, and a `phase:design` flag.
2. **I tagged Devin on the issue.** It immediately asked me a clarifying question (more on this below). After I answered, it produced a design doc on the wiki, opened a PR, and told me what manual prerequisites I'd need to do to actually deploy.
3. **Claude Code reviewed the PR** — flagged two blockers (unpinned skaffold version, no deploy timeout) and five nice-to-haves.
4. **I posted the review to the PR as a comment.** Devin auto-detected the comment, addressed the blockers and the easy nice-to-haves in commit `a23ec5e`, and replied within minutes.
5. **Claude Code re-verified** the changes matched the description.
6. **I applied Terraform** (5 added, 5 destroyed — old Cloud Build module out, WIF infrastructure in), wired three GitHub repo variables, and merged the PR.
7. **The deploy workflow ran end-to-end in 4 minutes 2 seconds.** I verified the rollout by SHA. I opened a trivial smoke-test PR to confirm the PR-time CI workflow worked too.

Total wall-clock time: about 90 minutes, most of it waiting for the deploy job and reading. Total of-my-attention time: maybe 30 minutes across the day.

## What's different about Devin

In the [Perplexity post](https://medium.com/@huchka) I noted that Perplexity was **non-interactive**: it took a one-sentence prompt and handed back a finished PR, never asking a clarifying question. Devin in this workflow was much more interactive — the whole thing was a conversation, sometimes one I didn't initiate.

### It asks where the design doc goes

When I tagged Devin into an issue labeled `phase:design`, the *first* thing it did was ask me a question:

> Where should the design doc live — wiki page, or as a comment on the issue/PR?

That's a small thing but it's a sign of a deliberate workflow. My repo has a `phase:design` label specifically because I want a design pass before code. Devin noticed the label, inferred that "produce a design doc" was part of the task, and asked where to put it before proceeding. I picked the wiki because design docs that outlive the PR they're attached to belong on the wiki.

Devin then wrote a [Migrate-CI-to-GitHub-Actions](https://github.com/huchka/feedforge/wiki/Migrate-CI-to-GitHub-Actions) wiki page with: problem statement, approach, alternatives considered (a real comparison table), decision rationale, acceptance criteria, and open questions. That's a good shape for a design doc and matched my [Design-Doc-Template](https://github.com/huchka/feedforge/wiki/Design-Doc-Template). I didn't ask it to follow the template; it found it.

### It has guardrails I appreciated

Devin couldn't push to the wiki itself. It told me so:

> Push the wiki page. I couldn't push it from the session (guardrail against pushing to `master`). Run:
> ```bash
> cd /tmp/feedforge.wiki
> git push origin master
> ```

In this session, Devin cloned the wiki, committed the file locally, and asked me to push it myself. The framing in its message ("guardrail against pushing to `master`") is its own; I haven't tried to verify whether that's a documented platform-wide rule or session-specific behavior. Either way, the practical effect was: one 30-second `git push` from me, with a moment to look at what was about to land before it did.

Not a bad trade for a learning project. A small manual step in exchange for a clear policy boundary is reasonable, and the alternative — an agent pushing to anything called `master` with no human in the loop — isn't a default I'd want.

### It picks up PR comments automatically

After Claude Code reviewed PR #26, I posted the review as a single PR comment. I didn't tag Devin, didn't @-mention it, didn't open a new session. Within minutes Devin had replied:

> Thanks for the thorough review. Pushed `a23ec5e` addressing both blockers and two of the nice-to-haves: ...

It had detected the new comment, opened a session against the PR, addressed the actionable feedback, and replied. The reply itself was structured: blockers handled, nice-to-haves addressed, items deliberately deferred with reasoning. It accepted some feedback and pushed back politely on the rest.

Once the cycle is "Claude Code reviews → I post the review → Devin acts on it," the whole loop is observable on the GitHub PR page itself. No Devin tab needed; the PR is the canonical state.

Devin's intro comment on the PR mentioned an opt-out — adding `(aside)` to a comment makes it skip that one. I haven't tried it yet, but it's a nice touch — a way to write internal notes on a PR Devin is watching without triggering work.

### It auto-reviews PRs (whether you asked or not)

This was the friction. In my repo's setup, Devin's auto-review was triggering on every new PR I opened and posting a review summary. On PRs I authored myself (or that Claude Code wrote), I didn't want a third opinion every time — especially because most of Devin's review findings live behind a link to its own platform rather than inline on the PR.

I had to find the setting and remove my user from the auto-review enrollment list. Once I did, Devin only triggers when I explicitly tag it on an issue or comment on a PR it's already authored. That's the cadence I want. The lesson, scoped to my setup: **the defaults I had configured assumed I wanted maximum involvement.** Worth checking what your enrollment looks like before assuming the cadence will match your workflow.

### A note on free-tier credits

I'm using Devin on the free tier. Anecdotally, my available credit has reappeared a few times without me topping up — so something is replenishing on a schedule I haven't tracked. I haven't dug into the policy. The practical effect for a hobby project: I don't think about credit cost most of the time, but I also can't predict when I'll hit a wall. Fine for personal learning; I'd want to read the actual policy before depending on it for anything serious.

## Where Claude Code added value in this cycle

Below is what Claude Code did in my local environment with my configured tools — `gh`, `kubectl`, my project memory files, the slash commands I've installed. Other setups will be different. Three things stood out:

**1. The review.** I asked it to review PR #26. It walked the diff, checked the skaffold config to confirm the deploy assumptions matched, ran `gh pr checks` to confirm CI was green, and posted a structured review with strengths, blockers, and nice-to-haves. The two blockers were both real: an unpinned `latest` skaffold install URL (vendor breakage = next prod deploy fails) and no `timeout-minutes` on a job that could hang for the runner's 6h default. Devin had not caught these on its own first pass.

**2. The K8s explanation.** After Devin's PR was technically ready, I asked Claude Code to explain the whole change "from scratch as a Kubernetes learner." It produced a long walkthrough — what a deploy on Kubernetes actually does, why kustomize image-tag drift is a real failure mode, what skaffold owns vs what kubectl owns, what each WIF resource in the Terraform module does. That kind of explanation is hard to ask Devin for, because Devin is wired to *ship code*, not to teach. Different tools, different jobs.

**3. Operational walkthrough during deploy.** When I was ready to apply Terraform and wire repo variables, Claude Code wrote out the step-by-step: what `terraform plan` should report (5 added, 5 destroyed, with specific module names), what to verify in the WIF outputs, the `gh variable set` command shape, what the post-merge `kubectl` verification should show. When something didn't fit on screen (the `gh run watch` race after PR creation), it queued a background poll and notified me when both checks passed.

What Claude Code didn't do: write the original PR. I never asked it to. The split that emerged organically was: **Devin authors and iterates against feedback; Claude Code reviews, explains, and operationally handholds the human.**

## The handoffs that worked

The seam between agents that worked best was the PR comment as the message bus. Specifically: I would have a conversation with Claude Code about the PR, Claude Code would summarize a review, I would post that review verbatim as a PR comment, and Devin would pick it up and respond on its own.

This means the canonical record of *why something changed* lives on GitHub, not in any agent's session. If I come back to this PR in six months, I can read the review comment, see Devin's response, and see the diff that resulted. That's more durable than transcripts in any AI tool.

The other handoff that worked: Claude Code explaining a Devin-written PR in detail. Devin's commit messages and PR description are good but not pedagogical. They describe *what changed* but not *why this mechanism instead of that one*. Asking Claude Code to explain it is the part that turned the PR into something I learned from instead of just merged.

## The handoffs that didn't

A few rough edges worth naming:

- **Devin's auto-review noise** until I turned it off. Already covered above.
- **Wiki guardrail required a manual push.** Fine, but worth knowing in advance — if you tag Devin on a `phase:design` issue, expect to do one manual git push to publish the design doc. Not a big deal, just a context switch.
- **The PR was the only persistent state I could rely on.** When I came back to Devin in a new session it didn't reference earlier conversations, and I didn't see useful cross-session memory in this workflow. This is also broadly true of Claude Code across sessions — but I've configured `CLAUDE.md` memory files in the repo, so Claude Code at least has consistent project context. I'd want to read Devin's docs on persistent memory before claiming this is universal; in my workflow it didn't show up.
- **Most of Devin's review findings live on its platform, not on GitHub.** I disabled auto-reviews partly for this reason — for hobby use I want everything reviewable on GitHub itself. Pay-tier features may improve this.

## Verdict

The shape of the workflow that worked, for reference:

- Tag Devin on issues that have clear acceptance criteria.
- Use Claude Code to review what Devin produces.
- Post the review as a PR comment so Devin can respond on its own loop.
- Use Claude Code (separately) to explain and operationally handhold during the actual deploy.
- Keep human eyes on the integration test in a real cluster. (Same conclusion as last time.)

For a PR in a domain I already know reasonably well — like a CI/CD migration where the failure modes are familiar — this split was faster than me doing all of it myself. For something I don't understand, I'd want to write more of it myself just to learn it.

## Things I Learned

- **Different agents reward different prompt styles.** Perplexity (last post) was non-interactive and rewarded a heavily-front-loaded prompt with constraints baked in. Devin is interactive — it asks where to put design docs, asks about scope, picks up PR comments — and rewards *thin* initial prompts paired with willingness to answer its clarifying questions. Trying to use one style on the other agent mismatches the model.

- **The PR comment as a message bus is genuinely useful.** When the AI on the other end actually reads PR comments and acts on them, the GitHub PR page becomes the canonical audit trail of "why this code is shaped this way." That's more valuable than agent transcripts because it survives session ends and is browsable by anyone — including future me.

- **An unrequested guardrail wasn't a problem.** Devin refusing to push to `master` and asking me to do it myself added a small manual step I didn't mind — it signaled a clear policy boundary. For a personal project, a default of "pause and confirm before touching something called master" trades a few seconds of friction for a useful pause.

- **Auto-features that watch your repo are a different ergonomic from agents you tag.** Both can be useful. I prefer the latter for personal projects, where the watched-repo cadence doesn't always match my workflow. For a team setting where many PRs land per day and an extra reviewer is genuinely valuable, I'd reconsider.

- **The split that emerged organically: Devin authors, Claude Code reviews and explains.** I didn't plan this — I just gravitated to it after a few PRs. The split tracks how each tool felt in my workflow: Devin produced code with a clear audit trail on the PR; Claude Code worked best as a conversational pair-programmer who walked me through what was happening. Mixing them lets each play to type, at least the way I had each set up.

- **The integration test is still mine.** Last post said this and I'm saying it again because nothing about the new cycle changed it. `terraform plan`, `terraform apply`, watch the rollout, `kubectl get pods`, verify image tags — that is still the part where the abstract "code looks right" becomes "system works." Both Devin and Claude Code can prepare you for it (Claude Code very actively walked me through it). Neither does it for you. I expect this to be the last thing to automate, because it requires real cloud credentials doing real things in a real cluster — and that's exactly the trust boundary humans should sit on.
