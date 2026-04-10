# K8s from Scratch #0: I've Been Using Managed Kubernetes. Now I'm Building It by Hand.

*I've spent the last 17 posts learning Kubernetes by building [FeedForge on GKE](https://medium.com/@huchka) — deploying apps, configuring networking, setting up monitoring, fighting with YAML. Along the way I kept hitting the same feeling: I understand how to use Kubernetes, but I don't really understand how it works underneath. This is the start of a new mini-series where I find out.*

---

## Why Build from Scratch?

After 17 posts and a working application on GKE, I can write Deployments, debug CrashLoopBackOff, configure NetworkPolicies, and set up HPA autoscaling. But when something breaks at the infrastructure level, my mental model has gaps.

Questions like: why does kubeadm make you load `br_netfilter` before anything else? What actually happens during `kubeadm init`? Why did my kubelet keep restarting after installation, before the node joined a cluster? How do nodes discover each other?

GKE handles all of this. That's its job. But I want to understand what "all of this" actually is.

So I'm building a Kubernetes cluster on local VMs using kubeadm — no managed services, no abstractions hiding the details. Every setup choice understood before moving on.

---

## The Setup

**Hardware:** MacBook Air M2. Everything runs locally.

**VMs:** Three arm64 Ubuntu 22.04 instances on [Multipass](https://multipass.run/) — one control plane, two workers. Multipass is the simplest way to get Linux VMs on macOS. One command, one VM.

| Node    | Role          | CPUs | Memory | Disk |
|---------|---------------|------|--------|------|
| cp1     | control plane | 2    | 2GB    | 20GB |
| worker1 | worker        | 2    | 2GB    | 20GB |
| worker2 | worker        | 2    | 2GB    | 20GB |

**Automation:** cloud-init for OS-level node prep, Ansible for Kubernetes bootstrapping. I could do everything by SSH-ing into each node, but repeating the same commands across three VMs gets old fast. cloud-init applies its configuration on first boot. Ansible handles everything after.

**A lab script** wraps the Multipass lifecycle:

```bash
./lab.sh up       # Launch all 3 VMs
./lab.sh down     # Stop VMs (preserves state)
./lab.sh destroy  # Delete and purge everything
./lab.sh status   # Show VM list
```

Destroy and recreate the whole cluster in minutes. This matters — I've already rebuilt it twice while figuring out cloud-init configs.

> The full project is at [`huchka/k8s-bare-metal`](https://github.com/huchka/k8s-bare-metal) on GitHub.

---

## Treating a Learning Project Like a Real One

Here's the part that might seem like overkill: I'm running this project through a lightweight but complete development workflow.

GitHub issues with labels (`type:feature`, `priority:high`, `phase:design`, `size:M`). A project board with five columns (Backlog → Ready for Dev → In Progress → In Review → Done). Issue templates, PR templates, CONTRIBUTING.md, a wiki with design doc templates. Branches named `infra/1-os-preparation`. PRs that reference issues with `Closes #N`.

For a personal learning project. On local VMs. With an audience of one.

Why?

**Because the process itself is something worth practicing.** I've worked on teams where the SDLC was either nonexistent or so heavy that nobody followed it. I wanted to build a lightweight workflow that I'd actually use — and the only way to test that is to use it on a real project, even a small one.

**Because it forces structure on learning.** Without issues and phases, "learn Kubernetes" is an amorphous blob. With a plan broken into 7 phases, 27 issues, and concrete acceptance criteria, every session has a clear goal. I know what "done" looks like before I start.

**Because it creates a record.** Six months from now, I can look at the project board, the PRs, and the commit history and reconstruct exactly how I learned each concept. That's more useful than scattered notes.

**Because I want to figure out how AI development actually fits into a real workflow.** There's a lot of hype around AI-assisted coding right now — Claude Code, Devin, Cursor, Copilot — but most examples are either toy demos or "watch the AI build an entire app." I want to test something different: what happens when you give AI tools a structured SDLC to work within? Where does AI help most — design, implementation, review, all of it? Where does it get in the way? Having a real process with issues, branches, and PRs gives me a framework to experiment with different AI development approaches and actually compare them. The SDLC isn't just for the Kubernetes learning — it's the control variable for the AI experiment.

The workflow I'm using:

```
Design (wiki or inline) → Create issues (phase:ready)
  → Branch → Implement → PR (Closes #N) → Self-review
  → Merge → Verify → Done
```

Issues start as `phase:design` and move to `phase:ready` once the approach is clear. The board tracks everything. Each phase gets its own branch and PR.

Is it overhead? A little. But it takes less than a minute to create an issue, and the structure pays for itself every time I sit down and wonder "where was I?"

---

## The Plan

Seven phases, from bare OS to a validated cluster:

**Phase 1: OS Preparation** — Kernel modules, sysctl networking params, swap, containerd, kubeadm. The prerequisites that every tutorial tells you to copy-paste. (Already done — [next post](TODO) digs into what each one actually does.)

**Phase 2: Ansible Setup** — Inventory, SSH connectivity, playbook structure. The automation layer that makes the rest repeatable.

**Phase 3: Control Plane Bootstrap** — `kubeadm init`, the API server, etcd, static pods. Where the cluster actually becomes a cluster.

**Phase 4: CNI Plugin** — Pod networking. Without a CNI, nodes stay `NotReady` and CoreDNS remains `Pending` — cluster networking simply doesn't work. Choosing between Flannel, Calico, and Cilium.

**Phase 5: Worker Nodes Join** — `kubeadm join` and verifying all nodes are Ready.

**Phase 6: Validation** — Deploy a workload, test pod-to-pod networking across nodes, verify DNS.

**Phase 7: Extras** — Metrics-server, Ingress, NetworkPolicy, RBAC. Optional, but this is where the real learning happens.

Each phase maps to a parent issue on GitHub with sub-issues for individual steps. The full plan is in [`plan.md`](https://github.com/huchka/k8s-bare-metal/blob/main/plan.md) in the repo.

---

## Things I Learned (Already)

**For this kind of learning, Multipass + cloud-init is the best local setup I found on macOS.** I evaluated kind, minikube, and Vagrant before landing on Multipass. kind and minikube give you a working cluster fast, but they abstract away the node OS — you don't go through the real Linux VM setup path of configuring kernel modules, sysctl, and containerd yourself. Vagrant works but is heavier. Multipass gives you real Ubuntu VMs with cloud-init support in one command. For "build K8s from scratch" learning, it's the right tool.

**cloud-init is more powerful than I expected.** I initially planned to use it just for `apt-get update`. But it handles file creation (`write_files`), arbitrary commands (`runcmd`), package installation, and more — all declaratively in YAML. It's essentially a lightweight provisioning tool built into most cloud images. It's widely used for cloud instance initialization — including cloud-init-compatible images on AWS, GCP, and Azure.

**An SDLC template is worth building once.** I created a reusable template (`docs/sdlc-template.md`) with all the labels, board columns, issue templates, and `gh` CLI commands needed to set up a new project. Setting up k8s-bare-metal's full SDLC took about 10 minutes using the template. Without it, I would have skipped the structure entirely — and lost the benefits.

**The gap between "using K8s" and "understanding K8s" is bigger than I thought.** After 17 posts on GKE, I thought I had a decent understanding. Then I tried to explain why `br_netfilter` is needed and realized I couldn't. Building from scratch surfaces all the things that managed services quietly handle for you.

---

## What's Next

The next post breaks down the node prerequisites for Kubernetes — kernel modules, sysctl params, swap, containerd, and kubeadm. Not just the commands, but what each one does at the Linux level and what breaks if you skip it.

After that: Ansible, then the control plane bootstrap. That's where the real Kubernetes begins.

---

*Building a K8s cluster from scratch on local VMs. Follow along for posts about what's actually happening under the Kubernetes hood.*
