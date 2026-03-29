# I Thought Adding Security to My K8s Cluster Would Be Simple YAML. I Was Wrong.

*This is the ninth post in a series about learning Kubernetes by building FeedForge — an RSS feed aggregator with AI summarization on GKE. These posts are learning notes from someone figuring things out in real time. [Previous post here.](https://medium.com/@huchka)*

---

Phase 4 gave FeedForge autoscaling and a daily digest notification. The cluster was functional, but wide open internally — every pod could talk to every other pod, containers ran with full Linux capabilities, and nothing enforced the non-root users that were already in the Dockerfiles.

This post covers two Kubernetes security primitives: **NetworkPolicy** (restricting which pods can talk to which) and **SecurityContext** (restricting what pods can do). Both sound simple. Neither went smoothly.

## What I Built

> Check out the [`phase-5` tag](https://github.com/huchka/feedforge/tree/phase-5) in the FeedForge repo for the full source code at this point.

- **NetworkPolicy** on PostgreSQL and Redis — only authorized workloads can reach the database and message queue
- **SecurityContext** on all 7 workloads — non-root enforcement, read-only filesystems, all Linux capabilities dropped
- **Terraform updates** to enable Calico (NetworkPolicy enforcement) and the HPA metrics-server addon
- **HPA tuning** — reduced maxReplicas from 3 to 2 to fit the cluster's actual capacity

## NetworkPolicy: The Cluster Firewall

By default, every pod in a Kubernetes cluster can reach every other pod. Frontend can connect directly to PostgreSQL. A stray debug pod can hit Redis. There's no network isolation unless you explicitly add it.

NetworkPolicy is how you add it. You select a set of target pods, declare what ingress (or egress) traffic is allowed, and everything else is implicitly denied. The moment a NetworkPolicy targets a pod, that pod switches from "allow all" to "deny all except what's listed."

For FeedForge, the rules are straightforward:

**PostgreSQL (port 5432)** — allow from backend, summarizer, fetcher, and digest. Deny everything else.

**Redis (port 6379)** — allow from backend, summarizer, and fetcher. Deny everything else (digest doesn't use Redis).

The YAML is clean. Each policy targets pods by label and lists the allowed sources:

```yaml
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: postgres
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: backend
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: summarizer
      ports:
        - protocol: TCP
          port: 5432
```

Each `podSelector` under `from` is an OR — any pod matching any of those labels is allowed in.

### The Silent Failure: Policies That Do Nothing

Here's what caught me off guard. I applied the NetworkPolicy manifests, `kubectl get networkpolicy` showed them, but when I tested — frontend could still reach PostgreSQL. The policies existed but weren't being enforced.

GKE Standard clusters don't enforce NetworkPolicy by default. You need a CNI plugin that supports it — **Calico** can be enabled on an existing cluster, while **Dataplane V2** (Cilium) can only be enabled at cluster creation time. Without either, the Kubernetes API happily accepts your NetworkPolicy resources and does absolutely nothing with them.

This is a dangerous silent failure. You think your database is locked down, but it's wide open. Since I was working with an existing cluster, the fix was enabling Calico via Terraform and `gcloud`:

```hcl
network_policy {
  enabled  = true
  provider = "CALICO"
}
```

### The Terraform Chicken-and-Egg

I also needed to enable the `horizontal_pod_autoscaling` addon (the metrics-server for HPA, which had been `disabled = true` since Phase 0). I tried applying both changes in one `terraform apply`. Eight minutes later:

```
Error: googleapi: Error 400: The network policy addon must be enabled
before updating the nodes.
```

GKE requires Calico to be fully running on the control plane before allowing other node-level changes. But enabling Calico is itself a node-level change. Terraform sends the whole cluster config in one API call, so GKE rejects it.

I tried splitting into two applies — just Calico first, then the HPA addon. Same error. The Terraform provider still sends the full desired state, and GKE sees both changes.

The fix was to bypass Terraform entirely for the first step:

```bash
gcloud container clusters update feedforge-cluster \
  --zone us-central1-f \
  --update-addons NetworkPolicy=ENABLED
```

Then `terraform apply` to sync state and enable the HPA addon. This is only an issue on existing clusters — a fresh `terraform apply` on a new cluster handles everything in one shot because it's a create, not an update.

### Calico Has a Resource Cost

Enabling Calico adds a DaemonSet — a `calico-node` pod runs on every node. On a tight 2-node e2-medium cluster (4 vCPU, 8GB total), that's meaningful overhead. This becomes relevant later when SecurityContext changes cause a cascade failure.

### Verification

```bash
# Backend can reach postgres (allowed)
$ kubectl exec deploy/backend -n feedforge -- python -c \
  "from app.database import SessionLocal; db=SessionLocal(); print('OK'); db.close()"
OK

# Frontend cannot reach postgres (blocked — TCP connection times out)
$ kubectl exec deploy/frontend -n feedforge -- \
  sh -c 'nc -zvw3 postgres 5432 || echo "blocked (expected)"'
blocked (expected)
```

The policy works. Only the pods that should talk to the database can.

## SecurityContext: Pod-Level Lockdown

NetworkPolicy controls the network. SecurityContext controls what happens inside the container itself. Three restrictions, each blocking a different attack surface:

**`runAsNonRoot: true`** — Kubernetes refuses to start the pod if the container tries to run as root. The Dockerfiles already use `USER feedforge`, but SecurityContext enforces this at the kubelet level — when the container is about to start on the node, the runtime checks the UID and rejects it if it's 0. If someone accidentally removes the `USER` directive from a Dockerfile, K8s catches it before the container starts.

**`readOnlyRootFilesystem: true`** — Makes the container's filesystem read-only. If an attacker gets code execution inside a container, they can't write malicious scripts to disk. FeedForge's backend only writes to PostgreSQL and Redis over the network — it doesn't need a writable local filesystem.

**`capabilities: drop: ALL`** + **`allowPrivilegeEscalation: false`** — Linux capabilities are fine-grained root powers (raw network access, filesystem mounts, system admin). Dropping all of them means the container has zero special kernel privileges. And `allowPrivilegeEscalation` prevents a process from gaining more privileges than its parent, blocking setuid-style exploits.

### "Cannot Verify User Is Non-Root"

I added `runAsNonRoot: true` to every workload. On deploy, the backend, frontend, and summarizer all crashed immediately:

```
container has runAsNonRoot and image has non-numeric user (feedforge),
cannot verify user is non-root
```

The Dockerfiles use `USER feedforge` — a username, not a numeric UID. Kubernetes can't resolve a username to a UID from image metadata before container startup. It doesn't pull the image and inspect `/etc/passwd` — it just looks at the image config. If the user is specified by name, K8s can't verify it's not root, so it refuses to start the container.

The fix: add explicit `runAsUser` with the numeric UID in the pod's securityContext. But which UID? I had to check the actual images:

```bash
$ docker run --rm --entrypoint id .../backend:0.2.1
uid=999(feedforge) gid=999(feedforge) groups=999(feedforge)

$ docker run --rm --entrypoint id .../frontend:0.3.0
uid=100(feedforge) gid=102(feedforge) groups=102(feedforge)
```

Backend uses `useradd -r` on Debian-slim (gets UID 999). Frontend uses `adduser -S` on Alpine (gets UID 100). Different base images, different UIDs for the same username.

```yaml
# Backend pods
securityContext:
  runAsNonRoot: true
  runAsUser: 999
  runAsGroup: 999

# Frontend pods
securityContext:
  runAsNonRoot: true
  runAsUser: 100
  runAsGroup: 102
```

### Read-Only Filesystem vs Containers That Need to Write

Not every container can go fully read-only. The breakdown across FeedForge's 7 workloads:

**Full lockdown (readOnlyRootFilesystem: true):** Backend, summarizer, fetcher, digest — all four are pure network I/O. They read code from the image and communicate with PostgreSQL/Redis over TCP. No local writes needed.

**Read-only with emptyDir exceptions:** Frontend (nginx) needs writable paths for `/var/cache/nginx` and `/run`. Redis needs `/data`. The solution is `emptyDir` volumes — temporary directories that exist only while the pod is running, mounted at just the paths that need to be writable. The root filesystem stays read-only everywhere else.

```yaml
containers:
  - name: frontend
    securityContext:
      readOnlyRootFilesystem: true
    volumeMounts:
      - name: nginx-cache
        mountPath: /var/cache/nginx
      - name: nginx-run
        mountPath: /run
volumes:
  - name: nginx-cache
    emptyDir: {}
  - name: nginx-run
    emptyDir: {}
```

**Cannot use read-only:** PostgreSQL writes database files, WAL logs, and temp files across multiple paths on the root filesystem. The persistent volume at `/var/lib/postgresql/data` is writable by design, but postgres also writes elsewhere. Skip `readOnlyRootFilesystem` for postgres — everything else still applies.

### The Cascade Failure

This was the most educational failure of the entire phase.

SecurityContext changes forced every pod to restart simultaneously. On a 2-node e2-medium cluster with Calico overhead, there wasn't enough CPU for all pods to run at once. What happened:

Here's what I observed:

1. All pods start restarting simultaneously due to the spec changes.
2. The summarizer was still at 3 replicas — the HPA had scaled it up during the rolling restart's CPU spike, and it hadn't settled back down yet.
3. PostgreSQL gets stuck in `Pending` — no node has enough free CPU to schedule it.
4. Backend's init container (Alembic migrations) starts, tries to connect to PostgreSQL, fails with `failed to resolve host 'postgres.feedforge.svc.cluster.local'` — because the postgres Service has no endpoints while the pod is Pending.
5. Init container exits 1, backend enters `CrashLoopBackOff`.
6. Summarizer pods also crash — same DNS resolution failure.

The root cause wasn't SecurityContext itself — it was **resource pressure**. Three summarizer replicas plus Calico DaemonSet pods plus all other workloads exceeded the cluster's 4 vCPU budget. The fix was two things: scale summarizer down to 1 replica to free CPU, and reduce HPA `maxReplicas` from 3 to 2 so this can't happen again on a 2-node cluster.

The lesson: `maxReplicas` times resource requests must fit within the cluster's total allocatable CPU. If you're adding a DaemonSet (like Calico) that eats per-node resources, recalculate your headroom.

## Things I Learned

### NetworkPolicy Is Silently Ignored Without Enforcement

This is the most dangerous gotcha. Kubernetes accepts NetworkPolicy resources on any cluster, stores them, and reports them in `kubectl get` — but does nothing with them unless a CNI plugin enforces them. On GKE Standard, that means enabling Calico or Dataplane V2. Always test your policies by attempting a blocked connection, not just by confirming the resource exists.

### Terraform and GKE Don't Always Play Nice for In-Place Changes

Some GKE cluster changes have ordering dependencies that the Terraform provider can't express. The network_policy + node pool update conflict required a `gcloud` escape hatch. This is only a problem on existing clusters — fresh creates work fine. When modifying an existing GKE cluster, check if the change has prerequisites that need to be applied separately.

### `runAsNonRoot` Needs Numbers, Not Names

`USER feedforge` in a Dockerfile is not enough for Kubernetes to verify the container won't run as root. K8s needs a numeric UID via `runAsUser` in the securityContext. Different base images assign different UIDs to the same username — `docker run --entrypoint id` is how you find the actual number.

### Read-Only Filesystems Are Achievable for Most Containers

The initial assumption that "containers need writable filesystems" is usually wrong. Most application containers only do network I/O. The ones that do need local writes (nginx cache, redis data) can be handled with targeted `emptyDir` volumes at specific mount points. Only truly stateful workloads like PostgreSQL need a writable root filesystem.

### Know Your Cluster's Resource Ceiling

Adding infrastructure (Calico DaemonSet) and enabling autoscaling (HPA) on a small cluster can compound into resource exhaustion. The failure mode isn't obvious — pods get stuck in `Pending`, dependent services fail DNS resolution, and the whole stack cascades. Always verify that `maxReplicas * resource_requests + system_overhead` fits within total allocatable capacity.

## What's Next

Phase 5 continues with RBAC and ServiceAccounts — giving each workload its own identity instead of sharing the default, and then Workload Identity to bind those K8s identities to GCP IAM for secure cloud API access.

---

*This is part of a series where I build FeedForge, an RSS aggregator with AI summarization, to learn Kubernetes from the ground up. Each phase adds new K8s concepts while building a real application.*
