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

## Diagrams

- Generate architecture diagrams using Mermaid syntax
- Write the Mermaid code in a `.mmd` file in the post directory (e.g., `architecture.mmd`)
- Render to PNG automatically by visiting [mermaid.live](https://mermaid.live) via browser tools — paste the Mermaid code, export as PNG, and save to the post directory
- Save the PNG in the same directory (e.g., `architecture.png`)
- **Always include rendered PNGs in git commits** — the `.mmd` source alone is not enough since Medium needs the image files

## Workflow

1. Draft in Markdown in the numbered directory
2. Review and edit
3. Generate diagrams: write Mermaid code → render at mermaid.live via browser → save PNG to post directory
4. Commit all files including PNGs (post.md, .mmd, .png, screenshots) to git
5. Publish to Medium
6. Update the table above with the Medium title
