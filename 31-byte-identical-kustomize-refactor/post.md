# I Wanted Byte-Identical kustomize Output. Strategic Merge Had Other Plans.

*This is the thirty-first post in a series about learning Kubernetes by building FeedForge — an RSS feed aggregator with AI summarization on GKE. These posts are learning notes from someone figuring things out in real time. [Previous post here.](https://medium.com/@huchka)*

*This one is about a kustomize refactor where my "the cluster sees zero change" guarantee leaked in two specific places, and the fix was switching one patch from strategic merge to JSON 6902.*

---

> Source: [PR #36 in the FeedForge repo](https://github.com/huchka/feedforge/pull/36).

I had a kustomize base that was lying.

It claimed to hold the shared, environment-agnostic manifests. But the backend Deployment had a `cloud-sql-proxy` sidecar baked in, the Workload Identity annotation pointed at a specific GCP service account, and a `postgres-credentials` volume assumed the Secrets Store CSI driver was running in the cluster. None of that is "shared" — it's all "production-on-GCP."

FeedForge has two overlays: `dev` (deploys to a managed GKE cluster) and `local` (deploys to a kind cluster on my laptop). Both consume the same base. The `local` overlay had to use `$patch: delete` directives to strip the GCP-specific bits out before deploying — which is the smell that something belongs in a Component instead of the base.

The refactor was straightforward in shape:

- Strip the GCP-specific bits out of `k8s/base/`.
- Move them into a new `k8s/components/postgres-cloudsql/` Component.
- Tell the dev overlay to `components: [postgres-cloudsql]` instead of getting it for free from base.

But the dev overlay deploys to a real GKE cluster running real traffic. The whole point of doing this as a refactor — not a feature — is that **the cluster should see zero change**. Whatever kustomize renders today, it must render tomorrow. Same bytes.

That was my verification gate. I'll explain why it's so cheap and so useful, then show two specific places where it cracked.

## The refactor and the byte-identical gate

The contract of a refactor is "behavior unchanged." For Kubernetes manifests, behavior comes from the rendered YAML — what `kubectl apply` actually sends to the API server. If `kubectl kustomize k8s/overlays/dev` produces the same output before and after my changes, the manifests are *render-equivalent*. The cluster could still see differences from admission mutation, defaulting, or server-side apply field management — but the largest, most controllable source of drift, the manifest itself, is provably unchanged.

That's not just a theoretical guarantee — it's a review aid. "Look at this 21-file diff" is a chore. "Look at this 21-file diff that produces byte-identical render" is a one-liner: `diff before.yaml after.yaml` returns nothing, and the review can focus on whether the new layout is *better organized* without arguing about whether it's *behaviorally equivalent*.

So before touching anything, I captured the baseline:

```bash
kubectl kustomize k8s/overlays/dev > /tmp/dev-baseline.yaml
shasum /tmp/dev-baseline.yaml
# 730d31120c6503cca1c2457d9b72f864fb391ee1  /tmp/dev-baseline.yaml
```

The hash is the contract. After my changes, the same command must return the same hash.

I did the work — wrote the Component patches, stripped base, updated the overlay — and ran the diff:

```
$ diff /tmp/dev-baseline.yaml /tmp/dev-after.yaml | wc -l
17
```

Not byte-identical. Two distinct problems, hiding in three diff chunks.

## Leak one: `$setElementOrder` in the rendered Deployment

The first chunk:

```diff
+       $setElementOrder/initContainers:
+       - name: cloud-sql-proxy
+       - name: run-migrations
+       - name: log-exporter
```

That `$setElementOrder/initContainers:` is a strategic-merge directive. It's supposed to be consumed by kustomize during patch processing — kustomize reads it, reorders the list, then strips the directive from the output. The Kubernetes API server doesn't know what to do with a `$setElementOrder` key on a `PodSpec` and would reject the manifest if you tried to apply it.

It was in my output anyway.

I'd added the directive deliberately. The base, after my strip, had `initContainers: [run-migrations, log-exporter]`. My patch added `cloud-sql-proxy` and used `$setElementOrder` to push it to position 0:

```yaml
spec:
  template:
    spec:
      $setElementOrder/initContainers:
        - name: cloud-sql-proxy
        - name: run-migrations
        - name: log-exporter
      initContainers:
        - name: cloud-sql-proxy
          image: gcr.io/cloud-sql-connectors/cloud-sql-proxy:2.15.2
          # ... rest of the spec
```

The merge worked — `cloud-sql-proxy` ended up at index 0, where it needed to be. But kustomize 5.5.0 (the version embedded in `kubectl` 1.32) didn't strip the directive afterward. It just carried it into the rendered Deployment as if it were a regular field.

This isn't documented behavior. In my environment (`kubectl v1.32.2`, the embedded kustomize is `v5.5.0`), the directive was preserved in the rendered output. Whether that's a kustomize bug, a limitation specific to the unified `patches:` field, or something I was using wrong, I never pinned down — the behavior was reproducible enough that I stopped digging once I had a working alternative.

That's leak one. The merge produced the right list order, and the rendered YAML carried a directive that doesn't belong in a Deployment.

## Leak two: new list entries land at the front

The other two diff chunks were both list-order flips. Baseline volumeMounts on the backend container were:

```yaml
volumeMounts:
- mountPath: /var/log/app
  name: app-logs
- mountPath: /mnt/secrets/postgres
  name: postgres-creds
  readOnly: true
```

My render produced:

```yaml
volumeMounts:
- mountPath: /mnt/secrets/postgres
  name: postgres-creds
  readOnly: true
- mountPath: /var/log/app
  name: app-logs
```

`postgres-creds` and `app-logs` were flipped. Same data, different order. The same flip happened in the pod's `volumes:` list.

My instinct was that strategic merge should *append* new entries to a list. `app-logs` exists in the base, `postgres-creds` is added by the patch, the merge result should be `[app-logs, postgres-creds]`. That's what I expected. That's what I remembered kustomize doing.

What kustomize 5.5 actually did was put the patch's new entries *first* and the base's untouched entries after. So `[postgres-creds, app-logs]`. Same elements, opposite end of the list.

I don't know if this counts as a bug or a deliberate change. In this specific manifest the mounts don't overlap and the volumes don't depend on each other, so the runtime behavior is the same either way. But for *byte-identity*, "behaviorally equivalent" isn't good enough. The whole point is that `diff` returns nothing.

## Why list order isn't always cosmetic

Here's the part where this stops being a cosmetic concern.

The backend's init containers, in the original base, were:

```yaml
initContainers:
  - name: cloud-sql-proxy   # restartPolicy: Always — native sidecar
  - name: run-migrations    # alembic upgrade head
  - name: log-exporter      # restartPolicy: Always — native sidecar
```

`run-migrations` connects to Postgres on `localhost:5432` to apply the alembic migrations. The connection only works because `cloud-sql-proxy` — running as a Kubernetes "native sidecar" (an init container with `restartPolicy: Always`; the feature shipped behind a gate in 1.28, on by default since 1.29, stable in 1.33) — is already up and listening on port 5432 by then.

Native sidecars start sequentially with the rest of the init containers, but because they have `restartPolicy: Always`, the kubelet treats them as `started` as soon as their container is running (or, if they declare a `startupProbe`, when that probe succeeds) and proceeds to the next init container without waiting for them to exit. So:

1. `cloud-sql-proxy` starts, opens `localhost:5432` → marked ready.
2. `run-migrations` starts, dials `localhost:5432`, runs alembic, exits cleanly.
3. `log-exporter` sidecar starts (background log tail).
4. The main `backend` container starts.

If `cloud-sql-proxy` ends up in slot 2 instead of slot 0, the sequence becomes:

1. `run-migrations` starts. Dials `localhost:5432`. Nothing's listening. Connection refused. Container exits with error.
2. Pod restarts. Loops forever.

For volumeMount and volume order, the strategic-merge flip was harmless in this specific case — the mounts don't overlap and no volume depends on another. But for `initContainers`, ordering is functional. Wrong order = broken pod. A "non-byte-identical" render that happens to also reorder init containers isn't just a cosmetic diff; it's a pending crashloop the next time anything restarts.

That made the byte-identical gate suddenly load-bearing for the right reason. Without it I might have squinted at the diff, said "the elements are the same, who cares about order," and shipped a regression. The diff being non-empty was the warning bell.

## The fix: JSON 6902 with explicit positions

kustomize supports two patch flavors. **Strategic merge** is the default. It knows Kubernetes-aware semantics like list merge keys (`name` for containers, `name` for volumes, `containerPort` for ports). It reads naturally because the patch *looks* like a Deployment with the fields you want to add or change.

**JSON 6902** (named after [RFC 6902](https://datatracker.ietf.org/doc/html/rfc6902)) is dumber but more precise. It's a list of operations like `{op: add, path: /spec/.../initContainers/0, value: {...}}`. No Kubernetes knowledge — just JSON pointer paths against the document.

For the backend deployment patch, JSON 6902 was the right tool because every place I needed to add something was a place where the position mattered. I rewrote `backend-patch.yaml` from a strategic-merge file into a JSON 6902 file:

```yaml
- op: add
  path: /spec/template/spec/initContainers/0   # insert at index 0
  value:
    name: cloud-sql-proxy
    image: gcr.io/cloud-sql-connectors/cloud-sql-proxy:2.15.2
    restartPolicy: Always
    args:
      - "--structured-logs"
      - "--private-ip"
      - "--port=5432"
      - "$(CLOUD_SQL_INSTANCE)"
    # ... rest of the container spec
- op: add
  path: /spec/template/spec/containers/0/volumeMounts/-   # /- = append
  value:
    name: postgres-creds
    mountPath: /mnt/secrets/postgres
    readOnly: true
- op: add
  path: /spec/template/spec/volumes/-
  value:
    name: postgres-creds
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: postgres-credentials
```

The `/0` says "insert at index 0, push everything after it right one slot." The `/-` says "append after the last element." No ambiguity, no merge directives, no leaked output.

Strategic merge files are recognized by their content shape (a Kubernetes object). JSON 6902 files are recognized by their content shape (a JSON Patch array). But for kustomize to know *which Kubernetes resource* a JSON 6902 patch targets, you have to specify a `target` block in the kustomization entry — strategic merge gets that for free from the patch file's `metadata.name` and `kind`:

```yaml
patches:
  - path: backend-patch.yaml
    target:
      group: apps
      version: v1
      kind: Deployment
      name: backend
```

I re-rendered:

```bash
kubectl kustomize k8s/overlays/dev > /tmp/dev-after.yaml
shasum /tmp/dev-after.yaml
# 730d31120c6503cca1c2457d9b72f864fb391ee1  /tmp/dev-after.yaml
```

Same hash as the baseline. `diff` returned nothing. The contract held.

## When to reach for strategic merge, when for JSON 6902

I'm not abandoning strategic merge. The other patches in the same Component — for `fetcher`, `summarizer`, `digest`, and the four ServiceAccounts — are still strategic merge. They read more naturally and the things they add don't have ordering constraints I have to defend, so they're not at risk of either leak.

The heuristic I came out of this with:

| Reach for | When |
|---|---|
| Strategic merge | Adding/modifying fields where order doesn't matter; when matching by merge key (`name`) is what you want; when you want the patch to *look like* the resource being patched so reviewers can grok it without learning JSON Pointer |
| JSON 6902 | List position is functional (init container ordering, env var precedence); strategic merge produces wrong or leaky output for your specific case; when you need to remove a single element by index without touching siblings |

The other thing JSON 6902 wins at is "I want this exact thing in this exact place, full stop." Strategic merge will sometimes do too much — merge a list when you wanted to replace it, leak a directive into the output, prepend when you expected append. JSON 6902 does the literal operation you wrote. Verbose, but unambiguous.

The mental model shift was treating JSON 6902 as a precision tool, not as the verbose escape hatch you reach for when strategic merge fails. After this refactor it's the first thing I'd reach for whenever I need explicit positional control.

## Things I Learned

- **Byte-identical render is a cheap, powerful refactor proof.** When you want to prove a refactor didn't change what gets sent to the API server, `shasum` on the rendered output is your test suite. It catches bugs no schema validator would — the wrong list order in a merge, a leaked directive — because it tests the property you actually care about (manifest render-equivalence) instead of structural validity.
- **Strategic merge has loose corners in modern kustomize.** `$setElementOrder` directives can leak into rendered output in 5.5. New list entries can land before existing entries instead of after. Both are recoverable, but you only find them if you have a baseline to compare against.
- **`initContainers` order is functional, not cosmetic.** A native sidecar (`restartPolicy: Always` init container) at index 0 is a load-bearing piece of the pod startup sequence. Anything that depends on it must come after. List ordering bugs that would be harmless for `volumes:` are crashloop-inducing for `initContainers:`.
- **JSON 6902 is the precision tool, not the ugly escape hatch.** I'd been treating it as the "verbose but powerful" alternative to strategic merge. After this, I see it as the right choice whenever I need explicit positional control. Strategic merge for "merge these fields in"; JSON 6902 for "put exactly this thing in exactly this place."
- **Capture the baseline before you start.** Three minutes of `kubectl kustomize ... > /tmp/baseline.yaml && shasum` would have saved me an hour of "is this diff a real change or a kustomize quirk?" if I'd done it first instead of last. Always record what you're refactoring *from*, not just what you're refactoring *to*.
