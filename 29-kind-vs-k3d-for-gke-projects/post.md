# Local Kubernetes for Cloud-Targeted Projects: Match the Policy Engine, Not the Startup Time

*This is a post in a series about learning Kubernetes by building FeedForge — an RSS feed aggregator with AI summarization on GKE. These posts are learning notes from someone figuring things out in real time. [Previous posts on Medium.](https://medium.com/@huchka)*

---

> Source: [PR #34 in the FeedForge repo](https://github.com/huchka/feedforge/pull/34).

I needed a local Kubernetes for FeedForge. Most "kind vs k3d vs minikube" comparisons rank by startup time, memory footprint, and "does NetworkPolicy work out of the box." Those are the wrong criteria for a project that deploys to a managed cloud cluster.

The criterion that actually matters: **which NetworkPolicy enforcement engine runs locally, and does it match what runs in prod.** That's a different question from "does NetworkPolicy work" — recent versions of both kind and k3d enforce policies out of the box. The question is *whose implementation*.

For FeedForge on GKE Standard, the answer is `kind + Calico`. The reasoning generalizes.

## Two layers, often confused

A K8s cluster has two networking layers:

- **Pod networking CNI**: assigns pod IPs, routes pod-to-pod traffic. Examples: kindnet, Flannel, Calico, Cilium, AWS VPC CNI, Azure CNI.
- **NetworkPolicy enforcement engine**: reads `NetworkPolicy` resources and enforces them with iptables/eBPF/whatever. Examples: kube-network-policies (used by kindnet since v0.24), Calico, Cilium, kube-router's netpol library.

These can be the same thing (Calico does both) or different (k3s uses Flannel for pod networking and embeds kube-router's netpol library only for policy enforcement — see [K3s networking docs](https://docs.k3s.io/networking/networking-services)).

Most comparison tables collapse them into one column labeled "CNI." That's the source of a lot of confusion.

## Where the popular advice goes wrong

The popular take used to be "kind doesn't enforce NetworkPolicy, k3d does, so use k3d if you care about policies." That advice is stale. [kind v0.24.0](https://github.com/kubernetes-sigs/kind/releases/tag/v0.24.0) added built-in NetworkPolicy support via [`sigs.k8s.io/kube-network-policies`](https://github.com/kubernetes-sigs/kube-network-policies). As of recent kind releases, both kind and k3d enforce policies out of the box.

So if both enforce, why does the local distro choice still matter?

Because the engines are *different implementations* of the same Kubernetes spec. The [NetworkPolicy spec](https://kubernetes.io/docs/concepts/services-networking/network-policies/) is loose in well-known places: SCTP support, `endPort` support, behavior with host-networked pods, behavior on connections that existed before the policy was applied, logging. Different engines make different choices in those gaps.

If your local engine differs from your prod engine, your local tests can pass on policies that fail (or fail differently) in prod. That's the class of bug local dev is supposed to prevent.

## Mapping prod engines to local options

For GKE specifically, the engine depends on which dataplane you chose at cluster creation:

| Cluster | Pod networking | NetworkPolicy engine | Source |
|---|---|---|---|
| GKE Standard (legacy dataplane) + `--enable-network-policy=true` | Calico | Calico | [GKE NetworkPolicy docs](https://cloud.google.com/kubernetes-engine/docs/how-to/network-policy) |
| GKE Standard or Autopilot with Dataplane V2 | Cilium | Cilium | [GKE Dataplane V2 docs](https://cloud.google.com/kubernetes-engine/docs/concepts/dataplane-v2) |
| AKS w/ Azure CNI Powered by Cilium | Azure CNI (overlay or VNet) | Cilium | [AKS Cilium docs](https://learn.microsoft.com/en-us/azure/aks/azure-cni-powered-by-cilium) |
| EKS + managed Calico addon | AWS VPC CNI | Calico | (varies by addon choice) |
| Local: kind default | kindnet | kube-network-policies | [kind v0.24 release](https://github.com/kubernetes-sigs/kind/releases/tag/v0.24.0) |
| Local: k3s / k3d default | Flannel | kube-router (embedded library) | [K3s networking](https://docs.k3s.io/networking/networking-services) |
| Local: kind + Calico | Calico | Calico | (manual install) |
| Local: kind + Cilium | Cilium | Cilium | (manual install) |

Note FeedForge is on GKE Standard with `--enable-network-policy=true`, the legacy dataplane. So the matching local engine is Calico, and the cluster type that lets me install Calico cleanly is kind. (k3d theoretically works too, but the swap-default-CNI dance is messier than disabling kindnet in a kind config flag.)

If FeedForge ran on Dataplane V2, the right local choice would be `kind + Cilium`. If it ran on AKS Cilium, same. The pattern is "local engine matches prod engine," not "kind always wins."

## What "matches" actually buys you

I want to be careful not to overclaim. Self-managed Calico in kind is not byte-for-byte identical to GKE-managed Calico — they're different versions, different host kernels, different surrounding glue. But running the same policy engine eliminates one specific class of bug: behavioral divergence that comes from the engine itself implementing the spec's loose corners differently.

Concrete examples of where engines diverge (from the [NetworkPolicy spec's "implementation-defined" sections](https://kubernetes.io/docs/concepts/services-networking/network-policies/) and engine docs):

- **Protocol support beyond TCP/UDP.** SCTP support is feature-gated and not all engines implement it.
- **`endPort` field.** Optional in the spec; engines vary on whether it's supported.
- **Host-networked pods.** Spec doesn't fully define how policies apply to them.
- **Connections that existed before the policy.** Some engines drop them, some don't.
- **Logging.** Out of spec entirely; only some engines support it.
- **FQDN-based egress rules.** Out of base spec; Cilium has them, Calico has them, kindnet doesn't.

A policy that works on kindnet and fails on Calico (or vice versa) usually trips on one of those.

## The setup for kind + Calico

```bash
# kind-config.yaml — disable kindnet, multi-node so you can test cross-node policy
cat > kind-config.yaml <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: feedforge-local
networking:
  disableDefaultCNI: true
  podSubnet: "192.168.0.0/16"
nodes:
  - role: control-plane
  - role: worker
  - role: worker
EOF

kind create cluster --config kind-config.yaml

# Install Calico via the Tigera operator (matches the GKE managed setup conceptually)
kubectl apply --server-side -f \
  https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/tigera-operator.yaml
kubectl apply -f \
  https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/custom-resources.yaml

# Wait for the operator to create calico-system, then for pods to be Ready
kubectl wait --for=condition=Available --timeout=120s \
  deployment/tigera-operator -n tigera-operator
until kubectl get namespace calico-system >/dev/null 2>&1; do sleep 3; done
kubectl wait --for=condition=Ready pods --all -n calico-system --timeout=300s
```

Subtlety: a naive `kubectl wait --for=condition=Ready pods --all -n calico-system` right after applying the manifests fails with "no matching resources found" because the operator hasn't created the namespace yet. The poll loop above is what makes the bootstrap reliable.

## Verifying enforcement (regardless of which engine)

This is the step that turns "I installed Calico" into "I trust it." Two pods, one allowed, one blocked, both run from inside the cluster.

I had a NetworkPolicy that allowed only `backend`, `summarizer`, and `feed-fetcher` pods to reach Redis on port 6379:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: redis-allow-clients
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: redis
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: backend
        # ... summarizer, feed-fetcher
      ports:
        - protocol: TCP
          port: 6379
```

**Allowed path** (backend pod → redis):

```bash
kubectl exec -n feedforge -it deploy/backend -- python -c \
  "import redis; r = redis.Redis(host='redis.feedforge.svc.cluster.local'); print(r.ping())"
# Expected: True
```

**Blocked path** (random pod with non-matching label → redis):

```bash
kubectl run -n feedforge tmp --image=alpine --rm -it --restart=Never \
  --labels=app.kubernetes.io/name=intruder \
  -- timeout 5 nc -z -v redis.feedforge.svc.cluster.local 6379
# Expected: nc: ... Operation timed out
```

The `timeout 5` matters — `nc` and `redis-cli` will sit forever waiting for a connect, and Ctrl-C through `kubectl run -it` doesn't propagate cleanly. Wrap inner commands with a timeout.

If both behave as expected, your engine is enforcing the policy you wrote. Same test works against Calico, Cilium, kindnet — it's a property test, engine-agnostic.

## When you should pick a different local

This isn't a "kind always wins" post. The takeaway is "match your prod engine," and that maps to different choices:

| Prod target | Local pick |
|---|---|
| GKE Standard (legacy dataplane) w/ NetworkPolicy | kind + Calico |
| GKE w/ Dataplane V2 (Standard or Autopilot) | kind + Cilium |
| AKS w/ Azure CNI Powered by Cilium | kind + Cilium |
| EKS w/ managed Calico addon | kind + Calico |
| On-prem k3s | k3d (matches the kube-router netpol library you're already running) |
| Project doesn't use NetworkPolicy in prod | k3d for speed; engine doesn't matter |

`kind + Cilium` is the underrated combination — Cilium has clean kind installation docs, and it's the right local match for an increasing chunk of managed clusters as Dataplane V2 becomes the default for new GKE Autopilot clusters and AKS Cilium adoption grows.

## Caveats I'm punting on

A few things this post doesn't cover but are real for full local-prod parity:

- **metrics-server** isn't in kind by default. CPU-based HPAs won't work locally. Either install it, or accept that local doesn't exercise HPA scaling. (For FeedForge, I just delete the HPA in the local kustomize overlay.)
- **Workload Identity, Cloud SQL Proxy sidecar injection, GKE-specific webhooks.** Not replicable locally. Accepted divergences.
- **Image architecture.** Building only `linux/amd64` (the GKE node arch) and running on an Apple Silicon host where kind nodes are arm64 fails at pull time with a platform mismatch. Generic to any local arm64 cluster, not kind-specific. Either build multi-arch, or set the platform per-environment in your build tool (e.g. a skaffold local profile with `platforms: ["linux/arm64"]`).

Those are each their own gotcha. The engine choice is the one that bites first and bites silently — NetworkPolicy bugs look like security and behave like nothing until prod traffic disagrees with you.

## What I'd tell my past self

If you're picking a local Kubernetes for a project that deploys to a managed cloud cluster, look up your prod cluster's NetworkPolicy engine and match it locally. Both kind and k3d enforce policies now, so "does it work" is a non-question — the question is whose implementation, and the answer should be "the same one prod runs."

The 15 seconds k3d saves on cluster boot is gone the first time a NetworkPolicy passes locally and fails differently in prod.

---

*Notes for engineers reading this:*

- *kind has had built-in NetworkPolicy support since [v0.24.0](https://github.com/kubernetes-sigs/kind/releases/tag/v0.24.0); pre-v0.24 advice that "kindnet ignores policies" is outdated.*
- *K3s's default CNI is Flannel; NetworkPolicy is enforced by an [embedded controller using kube-router's library](https://docs.k3s.io/networking/networking-services), not kube-router-as-CNI.*
- *GKE Standard supports both legacy dataplane (Calico) and Dataplane V2 (Cilium); the engine you get depends on cluster creation flags. Same goes for [AKS](https://learn.microsoft.com/en-us/azure/aks/azure-cni-powered-by-cilium).*
- *"Same engine" ≠ "byte-identical behavior." Self-managed Calico v3.28 on kind isn't bit-for-bit GKE-managed Calico. It eliminates implementation-difference bugs, not all behavioral differences.*
