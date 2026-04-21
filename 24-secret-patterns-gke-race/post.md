# Four Ways to Hand Your Pod a Password on GKE. One of Them Crashloops Your Cold Starts.

*This is the twenty-fourth post in a series about learning Kubernetes by building FeedForge — an RSS feed aggregator with AI summarization on GKE. These posts are learning notes from someone figuring things out in real time. [Previous post here.](https://medium.com/@huchka)*

---

My last post walked the full auth chain from a GKE pod to a secret in GCP Secret Manager. In this one I want to zoom in on a smaller question: once the auth chain is set up, **how do the secret values actually reach your application**? There are four common patterns. Three are startup-deterministic. One has a race condition that only shows up on cold starts — the kind of thing that makes your first deploy logs look like a dumpster fire even when everything is technically fine.

This post compares the four patterns, shows where the race lives, and gives you a decision tree for picking the right one for your situation.

## The four patterns

Before going deep, here's the landscape:

| # | Pattern | Secret lives in | App reads from | Race? | Rotation? |
|---|---|---|---|---|---|
| 1 | **`kubectl create secret` + `envFrom`** | K8s Secret (manual) | Env var | No | Manual restart |
| 2 | **CSI + `secretObjects` + `envFrom`** | GCP Secret Manager, synced to K8s Secret | Env var | **Yes** | Restart required |
| 3 | **CSI + file mount, read at startup** | GCP Secret Manager | File (read once) | No | Restart required |
| 4 | **CSI + file mount, read per auth attempt** | GCP Secret Manager | File (read repeatedly) | No | Hot rotation |

The cliff's-notes: Patterns 1, 3, and 4 are startup-deterministic — the secret is available when the container starts. Pattern 2 is the odd one out: the synced K8s Secret is created asynchronously, so it may or may not exist when your container boots. Pattern 4 additionally handles rotation without restart, at the cost of requiring more careful app code.

## Pattern 1: `kubectl create secret` + `envFrom`

The oldest and simplest:

```bash
kubectl create secret generic postgres-credentials \
  --from-literal=POSTGRES_USER=feedforge \
  --from-literal=POSTGRES_PASSWORD='...'
```

```yaml
env:
  - name: FEEDFORGE_DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: postgres-credentials
        key: POSTGRES_PASSWORD
```

**What's good:**
- No external dependencies — works on any cluster
- No cold-start race, because the Secret exists before the Deployment is applied
- Zero moving parts

**What's bad:**
- The secret lives in etcd. It *can* be encrypted at rest with envelope encryption, but nothing prevents anyone with namespace RBAC from running `kubectl get secret -o yaml` and reading the value
- Rotation means manually updating the Secret and rolling every consumer
- The source of truth is "whoever ran `kubectl create secret` last," and that's probably not written down
- No audit log of reads

**When to use it:** local dev, throwaway clusters, learning. For anything production-facing, you want better guarantees.

## Pattern 2: CSI + `secretObjects` + `envFrom`

This is the pattern most migration guides show, because it keeps your app code unchanged while moving the source of truth into Secret Manager.

```yaml
# SecretProviderClass
spec:
  parameters:
    secrets: |
      - resourceName: "projects/PROJECT/secrets/feedforge-postgres-password/versions/latest"
        path: "POSTGRES_PASSWORD"
  secretObjects:
    - secretName: postgres-credentials
      type: Opaque
      data:
        - objectName: "POSTGRES_PASSWORD"
          key: POSTGRES_PASSWORD
```

```yaml
# Deployment
env:
  - name: FEEDFORGE_DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: postgres-credentials     # created by CSI, not by you
        key: POSTGRES_PASSWORD
volumes:
  - name: postgres-creds
    csi:
      driver: secrets-store.csi.k8s.io
      volumeAttributes:
        secretProviderClass: postgres-credentials
```

The CSI driver mounts the secret as a file. The `secretObjects` block tells it to *also* sync the value into a native K8s Secret named `postgres-credentials`, which your existing `secretKeyRef` then references. The app still reads an env var. Beautiful in theory.

**What's good:**
- Source of truth is Secret Manager — audit logs, IAM, versioning, encryption
- Zero app code changes
- Values never touch your Git repo. The synced K8s Secret is still a regular API object and lives in etcd while it exists (so etcd encryption-at-rest still matters), but its lifecycle is managed by the CSI driver — it's deleted when all consuming pods are gone, not left lying around from a one-time `kubectl create secret`

**What's bad — the cold-start race:**

Here's what actually happens when a fresh pod starts:

```
1. kubelet mounts the CSI volume  ← synchronous, blocks
   └─ Driver calls Secret Manager, writes file
2. kubelet starts containers     ← as soon as mounts complete
3. In parallel:
   a. Container resolves env var from K8s Secret
   b. Sync controller sees the mount and creates the K8s Secret
```

Steps 3a and 3b happen in parallel. There is no ordering guarantee between them. On a first deploy — or any time the Secret doesn't exist yet — the container often wins the race. `secretKeyRef` looks up `postgres-credentials`, it's not there. Without `optional: true`, kubelet won't start the container and the pod sits in `CreateContainerConfigError`. With `optional: true`, the env var is simply omitted (not set, not empty string — just absent), the container starts, the app tries to connect with no password, and exits with an error. Either way, the cycle repeats until the sync controller catches up. On my cluster that was 30–60 seconds; exact recovery time depends on sync controller reconciliation and CrashLoopBackOff behavior.

This is a documented tradeoff, not a bug. The sync controller is intentionally decoupled from the mount lifecycle so it can handle many pods sharing one SPC efficiently. But the racy first deploy is surprising the first time you see it.

**When to use it:** retrofitting existing apps that read env vars, and you're willing to accept ~1 minute of startup noise on fresh deploys. I used this pattern in FeedForge as a pragmatic stepping stone. The upstream driver's sync-to-Kubernetes-Secret is [still documented as alpha](https://secrets-store-csi-driver.sigs.k8s.io/topics/sync-as-kubernetes-secret.html), so for anything production-grade Pattern 3 or 4 is a safer default.

## Pattern 3: CSI + file mount, read at startup

Drop the `secretObjects` block. The app reads the mounted file directly:

```yaml
# SecretProviderClass — same, minus secretObjects
spec:
  parameters:
    secrets: |
      - resourceName: "projects/PROJECT/secrets/feedforge-postgres-password/versions/latest"
        path: "POSTGRES_PASSWORD"
  # No secretObjects — no K8s Secret created
```

```python
# Application code
from pathlib import Path

SECRETS_DIR = Path("/mnt/secrets/postgres")
DB_PASSWORD = (SECRETS_DIR / "POSTGRES_PASSWORD").read_text().strip()

# ... use DB_PASSWORD in the SQLAlchemy URL
```

The mount is synchronous — the file is guaranteed to exist when the container starts. No race.

**What's good:**
- Race-free startup. The file is there when you read it.
- No K8s Secret materializes in the cluster at all. One fewer artifact, one fewer surface for accidental exposure.
- `envFrom` indirection is gone — the path from "secret in Secret Manager" to "value in app" is one hop shorter.

**What's bad:**
- Rotating a secret still requires a pod restart. You read the file once at startup; after that, whatever value was there is baked into your process memory.
- You need app code changes — every workload that reads a secret needs the file-reading helper.
- Local development gets a little trickier — your laptop doesn't have `/mnt/secrets/postgres/` mounted, so the code needs a fallback to env vars or a local file.

**When to use it:** greenfield app code, or when you're willing to make small edits. For most "replace `kubectl create secret`" migrations where rotation is a nice-to-have rather than a requirement, this is the right default.

## Pattern 4: CSI + file mount, read per auth attempt

Identical manifest to Pattern 3. The difference is how the app uses the file:

```python
from pathlib import Path
from contextlib import contextmanager

SECRETS_DIR = Path("/mnt/secrets/postgres")

def get_db_password() -> str:
    """Re-read on every call — supports hot rotation."""
    return (SECRETS_DIR / "POSTGRES_PASSWORD").read_text().strip()

def build_connection_url() -> str:
    return f"postgresql://feedforge:{get_db_password()}@localhost:5432/feedforge"
```

Read the file every time you open a new DB connection. If the CSI driver has rewritten the file (because you pushed a new secret version), new connections get the new password. No pod restart.

To enable rotation, start the CSI driver with `--enable-secret-rotation` and a `--rotation-poll-interval` (default 2 minutes). The driver polls Secret Manager, and when it sees a new version, atomically updates the mounted files so the file path stays stable and the new contents appear in one step.

**What's good:**
- All the benefits of Pattern 3
- Hot rotation: push a new secret version, within `rotation-poll-interval` the pods pick it up for new connections, no restart needed

**What's bad:**
- Rotation is messier than it looks in practice. Existing connections stay authenticated with the old password. For DB rotation, the server side needs to accept the new credential before old connections recycle — rotation policy has to account for connection pool lifetimes
- You read from disk on every auth attempt. Cheap, but not free. For hot paths, cache with a TTL shorter than your rotation poll interval.
- Rotation in the upstream CSI driver is [also documented as alpha](https://secrets-store-csi-driver.sigs.k8s.io/topics/secret-auto-rotation.html)

**When to use it:** when rotation-without-restart actually matters to you. Compliance-driven rotation policies, credentials that might be compromised without warning, systems where rolling restarts are disruptive.

## Rotation scoreboard

Just to make the rotation story explicit:

| Pattern | New secret version published | What happens in the running pod |
|---|---|---|
| 1. `kubectl create secret` + env var | Requires you to edit the Secret manually | Env var unchanged until pod restart |
| 2. CSI + `secretObjects` + env var | K8s Secret updated by sync controller | Env var unchanged until pod restart |
| 3. CSI + file + read once | File rewritten by driver (if rotation enabled) | Process memory unchanged until pod restart |
| 4. CSI + file + read per call | File rewritten by driver | New connections use new value within one poll interval |

Only Pattern 4 gets you real hot rotation. The others require a rolling restart — the question is just whether the Secret or file was updated silently first.

## Decision tree

If you're picking one:

![Decision tree for picking a secret pattern on GKE](decision.png)

Or in one sentence: if you can touch app code, prefer file mounts (Pattern 3 or 4). If you can't, and the cold-start race is acceptable, use CSI + `secretObjects` (Pattern 2). `kubectl create secret` is fine for local dev and nothing else.

## The Python helper I landed on

For FeedForge I ended up halfway between Patterns 2 and 3 — I'm still using `secretObjects` for now because it was the smallest delta during the migration, but I filed a follow-up issue to move to files. The helper I plan to use is pretty short:

```python
from __future__ import annotations
import os
from functools import lru_cache
from pathlib import Path

_SECRETS_DIR = Path("/mnt/secrets/postgres")


def _read_file(key: str) -> str | None:
    path = _SECRETS_DIR / key
    if not path.exists():
        return None
    return path.read_text().strip()


def get_db_credentials() -> tuple[str, str]:
    """
    Prefer mounted files (production). Fall back to env vars (local dev).
    Re-reads on every call so rotation is picked up without restart.
    """
    user = _read_file("POSTGRES_USER") or os.environ["FEEDFORGE_DB_USER"]
    password = _read_file("POSTGRES_PASSWORD") or os.environ["FEEDFORGE_DB_PASSWORD"]
    return user, password
```

Three notes on this:

- The env-var fallback keeps local dev trivial. A `.env` file still works. Mounted files take precedence when they exist.
- `get_db_credentials()` reads on every call. If that's too expensive for your access pattern, cache with a TTL like `lru_cache` + a manual invalidation, or a short-lived memoization. For a typical app opening a handful of connections per minute, the file reads are negligible.
- It returns a tuple because SQLAlchemy needs both user and password in its URL — you can adapt this shape for whatever your connection builder looks like.

## Things I Learned

- **Pattern 2's race condition is real but bounded.** It's not a correctness problem — pods eventually come up fine. It is a noise problem that makes fresh-deploy metrics and logs look worse than they are. Know what you're trading for env var compatibility.

- **File mounts are synchronous, K8s Secret syncs are not.** This is the single most important thing to understand about the CSI driver. The mount is ordered before container start; the `secretObjects` sync is not. If you need determinism, don't use `secretObjects`.

- **Rotation without restart is harder than rotation of files.** Even with Pattern 4, you still have connection pool lifetimes and server-side credential rollover to think about. "The file updates atomically" is one layer; full hot rotation is three.

- **Env vars are only race-free when the Secret exists before the pod starts.** The CSI `secretObjects` pattern is racy specifically because the Secret is created asynchronously on pod mount. An operator that pre-creates Secrets before workloads schedule can avoid this — the race isn't inherent to external secret managers, it's inherent to any reconciliation pattern that creates the Secret lazily. Check when your tool actually materializes the Secret.

- **`lru_cache` is not a rotation strategy.** If you cache the secret value in the app process, rotation stops working even with Pattern 4. Re-read on every call, or use a cache with explicit TTL tied to your rotation policy.

- **Local development is the biggest friction point for file mounts.** Your laptop doesn't have `/mnt/secrets/postgres/`. The env-var fallback is a must-have, not a nice-to-have. Skipping it means every contributor needs to recreate the mount path locally with real values, which is both annoying and a security risk.
