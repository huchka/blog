# After a Month, My Claude Code Config Was a Mess. The Cleanup Followed a Pattern.

*This is the thirtieth post in a series about learning Kubernetes by building FeedForge — an RSS feed aggregator with AI summarization on GKE. These posts are learning notes from someone figuring things out in real time. [Previous post here.](https://medium.com/@huchka)*

---

> Source: [commit `7631714`](https://github.com/huchka/feedforge/commit/76317145742be739decad90434ba97f094da6b7d) on the FeedForge repo. That commit is just the path-scoped rules part of the cleanup. The permissions and memory work happened in an umbrella `~/Projects/personal/.claude/` directory that isn't a git repo, so it can only show up here as before-and-after numbers.

A month ago I spent a full session configuring Claude Code before writing any FeedForge code. That was [post #1 in this series](https://medium.com/@huchka). Then I shipped a real application on GKE, wrote 28 more posts about it, and barely touched my Claude Code config since.

This week I opened `.claude/settings.local.json` and counted **104 entries**.

I'd been auto-approving permissions for a month — `git push`, `terraform plan`, `mmdc -i some-specific-file.mmd`, one-shot debugging `awk` patterns. Each one made sense in the moment. None of them was a deliberate config decision.

That's a familiar shape. It's the same drift you see in any long-running system: a `.bashrc` that grew over years of "this works, paste it in," a CI config with a dozen jobs no one remembers writing, a feature-flag table where half the flags are dead. My AI tooling had quietly accumulated the same kind of cruft, and the cleanup followed the same pattern: surface the noise, classify it, prune in a single pass.

The discomfort wasn't that the config was bad. It was that it had drifted out of conscious design without me noticing.

## What 104 permission entries actually look like

Here's the shape of the rot, with real examples from my file:

**One was syntactically broken.** Probably the most embarrassing one to find:

```
"Bash(/dev/null --format=\"value\\(account,status\\)\")"
```

That's a fragment of a `gcloud auth list` invocation that got mis-parsed when Claude Code tried to extract a permission pattern from it. It had been in my allowlist for weeks, matching nothing.

**Many were full-path duplicates of bare-name wildcards.**

```
"Bash(gh pr:*)"
"Bash(/opt/homebrew/bin/gh pr:*)"
"Bash(/opt/homebrew/bin/gh pr *)"
```

Three entries doing the same job because each session approved a different shape — sometimes Claude ran `gh pr view` directly, sometimes via the full Homebrew path, and the permission prompt offered each as a separate approve-once option.

**Two were nvm-version-pinned npm paths.**

```
"Bash(/Users/huchka/.nvm/versions/node/v22.14.0/bin/npm install *)"
"Bash(/Users/huchka/.nvm/versions/node/v22.14.0/bin/npm run *)"
```

Brittle. The next nvm bump and these stop matching anything. There's a more important problem too: `npm run *` is a wildcard for arbitrary code execution — any script defined in `package.json` can run any shell command. `npm install *` is in the same category, since `npm install` triggers a project's lifecycle scripts (`preinstall`, `install`, `postinstall`). Neither should have been blanket-approved. I dropped `npm run *` and replaced the install rule with a narrower `Bash(npm install *)` — still imperfect for the same reason, but at least bounded to install operations in projects I'm working on. A stricter version would require approval each time.

**Ten were one-shot `curl` calls** to GitHub's API for repos I was researching during a single decision (kind vs k3d vs minikube — that became [post #29](https://medium.com/@huchka)):

```
"Bash(curl -s \"https://api.github.com/repos/GoogleContainerTools/skaffold\")"
"Bash(curl -s \"https://api.github.com/repos/tilt-dev/tilt\")"
"Bash(curl -s \"https://api.github.com/repos/devspace-sh/devspace\")"
... (seven more)
```

Each was approved during a single research session. Each was one `GET` to a URL I will never hit again. They sit in my allowlist forever now, ten dead entries.

**Six were absolute-path `mmdc` commands** to render specific Mermaid diagrams:

```
"Bash(/opt/homebrew/bin/mmdc -i /Users/huchka/Projects/personal/blog/23-secret-manager-csi-auth-chain/architecture.mmd -o /Users/huchka/Projects/personal/blog/23-secret-manager-csi-auth-chain/architecture.png -b white -s 2)"
... (five more for other blog posts)
```

These should have been one wildcard: `Bash(mmdc *)`. Instead I'd approved each render individually because the permission prompt offered the literal command, not a generalization.

The patterns:

- **Approve-the-literal-string defaults** — every session writes a new specific entry instead of generalizing.
- **Path-prefix duplicates** — `/opt/homebrew/bin/X` and `X` are different strings; both get approved separately.
- **Version-pinned brittleness** — entries that go stale on the next package update.
- **Research one-shots** — tasks that recur once become permission entries forever.
- **Exact-string captures of one-off shell snippets** — the `awk '/pattern/{print}'` you ran to debug one thing, sitting in your allowlist permanently.

Once I categorized them, the cleanup was mechanical.

## The two-pass cleanup: skill, then manual

**The first pass was a built-in skill called `/fewer-permission-prompts`** that ships with Claude Code. The documented contract: it scans your recent transcripts, counts which read-only commands you've been approving, and proposes a wildcard allowlist for `.claude/settings.json`. In my run it also de-duplicated against commands that Claude Code already auto-allows out of the box — things like `ls`, `git status`, `gh pr view` — so the proposed list was tight rather than padded with rules I didn't need. I'm not sure whether that filtering is part of the documented behavior or just what mine did; either way the output was small and useful.

In my case it scanned 35 sessions in this project — 1019 Bash calls, 1684 MCP tool calls — and proposed eight patterns:

```json
{
  "permissions": {
    "allow": [
      "Bash(kubectl kustomize *)",
      "Bash(gh project item-list *)",
      "Bash(gh project list *)",
      "Bash(gh project field-list *)",
      "Bash(gh label list *)",
      "Bash(gcloud auth print-access-token)",
      "Bash(gcloud builds connections list *)",
      "Bash(gcloud builds repositories list *)"
    ]
  }
}
```

Eight entries. All read-only. All tested against actual usage history, not what I imagined I needed. These went into a new `.claude/settings.json` (the canonical, "shareable" file).

**The second pass was a manual prune of `settings.local.json`** — the messy 104-entry one. Most entries were already covered either by the eight canonical patterns or by Claude Code's built-in auto-allow list. The rest fell into the rot categories above, which made the cleanup decision-by-category rather than entry-by-entry:

- Drop the broken one.
- Drop full-path duplicates of bare-name wildcards.
- Drop version-pinned nvm paths; replace with `Bash(npm install *)` (and explicitly drop `npm run *` because it's arbitrary execution).
- Drop the ten `curl` one-shots and the four `awk '/.../{...}'` debugging patterns.
- Consolidate six absolute-path `mmdc` entries into one `Bash(mmdc *)`.
- Drop entries auto-allowed by Claude Code (`git --version`, `node --version`, basic git read-only).
- Keep the mutating-but-frequent ones I actually want pre-approved (`git push *`, `git commit *`, `terraform plan *`, `kubectl apply *`).

Result: 45 entries, organized into logical groups (web fetches → tmp reads → gh subcommands → gcloud → git → infra/k8s → tools). I kept a backup of the old file as `.bak.2026-04-26` in case I'd dropped something I needed.

Total time for both passes: about 25 minutes.

## Then I checked memory — and found a different kind of rot

Claude Code has a memory system separate from `CLAUDE.md`. It lives in `~/.claude/projects/<sanitized-cwd>/memory/`, with a `MEMORY.md` index loaded at session start and individual topic files referenced from it. It's meant for things like persistent user preferences and durable feedback ("don't auto-commit," "use first-person in writing"). I had six files in there:

```
MEMORY.md                            (the index)
user_profile.md
feedback_k8s_commands.md
feedback_no_autocommit.md
feedback_blog_images.md
feedback_blog_codex_review.md
feedback_skaffold_deploy.md
```

I read each one.

Four were duplicates of content already in a `CLAUDE.md` somewhere in the repo:

- `feedback_skaffold_deploy.md` was a one-paragraph rule about always using `skaffold run` instead of `kubectl apply -k`. The same rule, almost word-for-word, was already in `feedforge/CLAUDE.md` (visible in the [linked commit](https://github.com/huchka/feedforge/blob/76317145742be739decad90434ba97f094da6b7d/CLAUDE.md) under Kubernetes rules).
- `feedback_blog_images.md` told Claude to commit blog images alongside `post.md`. The same rule was in `blog/CLAUDE.md`.
- `feedback_blog_codex_review.md` instructed Claude to run a Codex CLI review after drafting any post. The same workflow was already documented in `blog/CLAUDE.md` as numbered steps.
- `feedback_k8s_commands.md` said the user runs `kubectl` commands manually as a learning exercise. That rule was in `feedforge/CLAUDE.md` and in `k8s-bare-metal/CLAUDE.md`.

The fifth, `feedback_no_autocommit.md`, was unique — it said don't auto-commit; wait for me to verify and ask. Not in any `CLAUDE.md`. Worth keeping.

The sixth, `user_profile.md`, was thin and partly stale. It listed projects ("feedforge on GKE, technical blog on Medium") that were now visible in the filesystem itself (the directories exist), and missed the two projects I'd added in the past month (`k8s-bare-metal/`, `apple-calendar-mcp/`).

So a different kind of rot from the permissions one: not bloat, but the right idea filed in the wrong place. The four duplicate memories had been useful when I wrote them — each was a feedback I'd given Claude that I wanted to persist. The mistake was that as I wrote things into a new `CLAUDE.md` later, I never went back to retire the corresponding memory file. They drifted from "a single source of truth" to "two sources of truth that could disagree" without any single moment where it was obviously wrong.

The cleanup: deleted the four duplicates, kept `feedback_no_autocommit.md` as-is, slimmed `user_profile.md` to just the things that aren't on disk and aren't in any `CLAUDE.md` (working style, the fact that I read Japanese tech blogs about LLM workflows, the build-then-blog feedback loop). Six files down to two.

## Memory or CLAUDE.md? The rule that emerged

After the consolidation I had two memory files (`user_profile.md`, `feedback_no_autocommit.md`) and three `CLAUDE.md` files (root, feedforge, blog), each with their own rules. The decision of where to file each fact, in retrospect, was clean:

**Memory holds what's hard to re-derive about the human.** Preferences. Working style. The fact that I read Japanese tech blogs about LLM workflows. Things that aren't in the filesystem and that no `CLAUDE.md` captures because they're about *me*, not the project.

**CLAUDE.md holds how the project works.** Rules, conventions, infrastructure decisions, things any contributor — human or AI — needs to know to work in this codebase. Loaded with the project. Lives next to the code.

The trap is that both *feel* like instructions. "Use first-person in writing" sounds like the same kind of fact as "use `apps/v1` not `apps/v1beta`." But they're not — one is about me as a user, one is about the project's conventions. Mixing them is the bug.

A simpler test: **if a fact is true regardless of which project I open, it goes in memory. If it's true because *this project* works this way, it goes in `CLAUDE.md`.**

That test would have caught all four of the duplicates I deleted. They were project rules — about a specific blog series, about a specific deployment tool, about a specific project's learning approach. None were about me as a user. They had been written into memory because at the time I gave the feedback the relevant `CLAUDE.md` either didn't exist yet or didn't cover that rule. As soon as it did, the memory should have moved.

I don't know how to enforce this automatically. Probably I won't try — a periodic reflective pass is easier than building tooling to detect duplication. But the rule itself is now explicit, which means I'll catch the next one in roughly an hour rather than a month.

## Things I Learned

- **Hygiene is a recurring task, not a one-off.** I configured Claude Code carefully on day one, then didn't audit it for a month. Both the permissions file and the memory directory needed pruning by the end. Plan for a half-hour audit every few weeks; treat it like dependency upgrades, not like a setup step you complete once.

- **Approve-once defaults bias toward duplication.** Every time the permission prompt offered the literal command I'd just run, I clicked "approve." The same conceptual permission — "run `gh pr` subcommands" — ended up encoded three different ways: `gh pr:*`, `/opt/homebrew/bin/gh pr:*`, `/opt/homebrew/bin/gh pr *`. Knowing the wildcard form up front lets you approve once at the right level of generality.

- **The duplicate-memory pattern is sneaky because both copies are useful when written.** I didn't put a rule in two places on purpose. I put it in memory when I gave the feedback, then into `CLAUDE.md` later when I was writing project documentation. I never noticed I'd done both. The drift only became visible when I pulled all six memory files into one view. Periodic consolidation is the only fix I see; there's no detection-time check, because the duplication isn't wrong in any single moment.

- **A skill (or any automated audit) beats willpower.** I would not have manually written a script to scan 35 transcripts and propose an allowlist. `/fewer-permission-prompts` did it in seconds. Same observation applies to memory — a memory-consolidation skill (the one I used ships in Anthropic's official skills plugin, not in core Claude Code) did the actual sorting; my job was to confirm what got dropped. When the audit is automatable, the cost goes from "an afternoon" to "a coffee break" — which is the threshold where it actually happens regularly.

- **Cruft has a shape.** Once I categorized the 104 permission entries, the same five buckets covered the rot: broken parses, full-path duplicates, version-pinned brittle paths, research one-shots, and exact-string captures of one-off shell snippets. That makes the next audit faster — I'll know what to look for. I suspect the same patterns show up in any system with approve-once defaults; the patterns aren't really about Claude Code, they're about how humans accumulate config under "fix this in the moment" pressure.
