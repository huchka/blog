# Blog

Technical blog posts published on Medium. Part of a series documenting the FeedForge project (learning Kubernetes by building a real app on GKE).

## Structure

```
blog/
├── 01-{slug}/post.md
├── 02-{slug}/post.md
├── 03-{slug}/post.md
└── ...
```

- Each post lives in a numbered directory: `{NN}-{short-slug}/post.md`
- Numbering matches publication order
- Post content is in `post.md` (Markdown)
- Images/assets go in the same directory if needed

## Published Posts

| # | Directory | Medium Title |
|---|-----------|-------------|
| 01 | `01-setting-up-claude-code-for-kubernetes-learning` | Before Writing a Single Line of Code, I Spent a Session Configuring My AI Pair Programmer |
| 02 | `02-from-terraform-apply-to-curl` | From `terraform apply` to `curl`: Standing Up a GKE Cluster from Scratch |
| 03 | `03-deploying-fastapi-to-gke` | Deploying a FastAPI App to GKE: What I Actually Learned About Kubernetes |
| 09 | `09-networkpolicy-securitycontext-hardening` | I Thought Adding Security to My K8s Cluster Would Be Simple YAML. I Was Wrong. |

## Writing Conventions

- Series intro paragraph at the top of each post (italic, links to previous posts)
- Tone: technical, practical, first-person narrative based on real experience
- Target audience: developers learning similar technologies
- Show real problems and debugging, not just the happy path
- Code snippets should be focused — enough to understand, not full file dumps
- Each post maps to a FeedForge phase or milestone
- Every post MUST end with a "Things I Learned" section — personal takeaways, surprises, and lessons from the phase
- After a PR is merged, tag the merge commit on GitHub (e.g., `phase-2`) and link to it in the post's "What I Built" section so readers can browse the exact code at that point in time
- **Always include images referenced in the post in the same commit** — don't commit post.md without its images

## Diagrams

- Generate architecture diagrams using Mermaid syntax
- Write the Mermaid code in a `.mmd` file in the post directory (e.g., `architecture.mmd`)
- Render to PNG using mermaid-cli (`/opt/homebrew/bin/mmdc`):
  ```bash
  mmdc -i <post-directory>/diagram.mmd -o <post-directory>/diagram.png -b white -s 2
  ```
- Save the PNG in the same directory (e.g., `architecture.png`)
- **Always include rendered PNGs in git commits** — the `.mmd` source alone is not enough since Medium needs the image files

## Workflow

1. Draft in Markdown in the numbered directory
2. Review and edit
3. Generate diagrams: write Mermaid code → render with `mmdc -i input.mmd -o output.png -b white -s 2` (white background ensures readability in dark mode on Medium)
4. Run Codex review and auto-apply fixes (see below)
5. Commit all files including PNGs (post.md, .mmd, .png, screenshots) to git
6. Publish to Medium
7. Update the table above with the Medium title

## Codex Review Automation

After drafting a blog post, run a Codex CLI review and apply fixes automatically:

1. Run the review non-interactively:
   ```bash
   codex exec -o /tmp/codex-review.md "Review this blog post draft for technical accuracy, clarity, and readability. Flag any factual errors, unclear explanations, or improvements. Be specific about line-level issues. File: <post-directory>/post.md"
   ```
2. Read the review output and assess each finding
3. **Auto-apply** fixes that are clearly correct (factual errors, misleading explanations, terminology mistakes, readability improvements)
4. **Ask the user** only when the fix involves a subjective judgment call, a major structural change, or when you're unsure whether the review finding is valid
5. After applying, summarize what was changed and why
