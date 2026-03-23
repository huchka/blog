# From `terraform apply` to `curl`: Standing Up a GKE Cluster from Scratch

*This is the second post in a series about learning Kubernetes by building FeedForge — an RSS feed aggregator on GKE. These posts are learning notes and progress reports from someone figuring things out in real time. [First post here.](https://medium.com/@huchka/before-writing-a-single-line-of-code-i-spent-a-session-configuring-my-ai-pair-programmer-c743b0886cc5)*

---

Phase 0 of FeedForge has one goal: get a GKE cluster running, deploy nginx, and hit it from the internet. No application code — just infrastructure.

Here's what I built, the decisions I made, and what I actually learned along the way.

---

## What I Built

- Custom VPC with secondary IP ranges for pods and services
- GKE Standard cluster (zonal) with a dedicated node pool (2x `e2-medium`)
- Least-privilege IAM service account for the nodes
- Artifact Registry repository with a cleanup policy
- An nginx deployment + LoadBalancer service for verification
- Everything managed by Terraform with GCS remote state

Four Terraform modules (`network`, `gke`, `iam`, `artifact-registry`), composed by a `dev` environment layer.

> Want to see the full Terraform modules, k8s manifests, and project structure? Check out the [`phase-0` tag](https://github.com/huchka/feedforge/tree/phase-0) in the FeedForge repo — it's a snapshot of the codebase at exactly this point.

---

## Key Design Decisions

**GKE Standard over Autopilot.** Autopilot manages nodes for you, which is great for production but defeats the purpose when you're learning. I want to understand node pools, machine types, autoscaling, and scheduling firsthand.

**Zonal over Regional.** Regional runs the control plane across three zones — more available, more expensive. Zonal is fine for learning and the management fee is covered by the free tier.

**Custom VPC with `auto_create_subnetworks = false`.** GKE needs VPC-native networking with secondary IP ranges for pods (`10.1.0.0/16`) and services (`10.2.0.0/20`). I want explicit control over the network, not GCP's auto-created defaults. Pods get a `/16` (65k IPs) and services get a `/20` (4k IPs) — generous, but you can't resize these later without recreating the subnet.

**Dedicated IAM service account.** The default Compute Engine SA has way too many permissions. I created a dedicated one with just five roles: log writing, metrics, monitoring, Artifact Registry reader, and GCS object viewer (needed for image pulls). The `cloud-platform` OAuth scope on nodes looks broad, but actual permissions are controlled by the IAM roles — this is the GCP-recommended pattern.

**Artifact Registry cleanup: keep 5 most recent images.** Old images pile up fast on a personal project. This saves storage costs without manual housekeeping.

---

## Verification

```bash
$ kubectl apply -f k8s/phase0-verify/nginx.yaml
$ kubectl get svc nginx-hello
NAME          TYPE           EXTERNAL-IP     PORT(S)
nginx-hello   LoadBalancer   34.xxx.xxx.xx   80:31234/TCP

$ curl 34.xxx.xxx.xx
<!DOCTYPE html>
<html>
<head><title>Welcome to nginx!</title></head>
...
```

Phase 0: done.

---

## Things I Learned

### Working with AI: Know When to Take the Wheel

I'm using Claude Code as my pair programmer for this project. When I started Phase 0, it generated a plan, wrote all the Terraform modules, ran `terraform plan` and `apply`, deployed the nginx verification manifest, and created the LoadBalancer service — all in one go, fully automated.

I had to stop it.

My goal is to *learn* Kubernetes, not watch an AI deploy things for me. The Terraform and GCP setup? Sure, automate that — it's infrastructure plumbing. But `kubectl apply`, understanding what a Deployment actually creates, watching pods come up, debugging why a Service has no external IP — that's the stuff I need to do myself.

The takeaway: AI pair programming is powerful, but if your goal is learning, you need to be intentional about which parts you delegate and which parts you do with your own hands. I now explicitly tell Claude to stop before any Kubernetes operations so I can run them myself.

### GKE LoadBalancer Service = GCP Network Load Balancer

When you create a Kubernetes Service with `type: LoadBalancer` on GKE, it doesn't just do something inside the cluster. GKE provisions an actual GCP Network Load Balancer — a Layer 3/4 load balancer with a real external IP address. That external IP shows up in your GCP console under Network Services.

This was obvious in hindsight, but it clicked for me when I saw the GCP resource appear. Kubernetes abstractions like "Service" feel like cluster-internal concepts until you realize they're creating real cloud infrastructure underneath.

### Free Trial $300 Credit: Region Matters More Than You Think

GCP's free trial gives you $300 over 90 days, which sounds like plenty. But there's a catch I didn't expect: **free trial workloads run at low priority for resource reservation.**

I originally planned to run the cluster in `asia-northeast1` (Tokyo) since I'm based in Japan. But I couldn't reliably get `e2-medium` instances there — the free trial's low-priority status meant resource reservations would fail in a popular region.

The fix: I moved the cluster to `us-central1` — GCP's largest region with the most capacity. Resources there are plentiful even at low priority. The latency doesn't matter for a learning project.

If you're on the free trial and hitting resource availability issues, try the largest region available. It's not in the docs, but it makes a real difference.

---

## What's Next

Phase 1: deploying the actual application. FastAPI backend, PostgreSQL as a StatefulSet, and my first real encounter with ConfigMaps, Secrets, PersistentVolumeClaims, and init containers.

That's where the Kubernetes learning really starts.

---

*Building FeedForge in public. Follow along for practical Kubernetes and GCP posts based on actually building things, not reading docs.*
