---
name: gitops
description: Review Kubernetes and GitOps changes for the failure modes that pass CI, deploy green, and break later — silent Argo drift, orphaned resources, secret paths that 403, queue topology owned by the wrong repo, and tags that ship the wrong artifact. Use when reviewing Helm values, Argo Application/ApplicationSet manifests, ExternalSecrets, CI release pipelines, or a deploy that "succeeded" but did not take effect.
---

# GitOps review

Most Kubernetes advice covers the happy path. This skill covers the failures that
survive a green pipeline — the ones where every check passes, the deploy reports
success, and the cluster is quietly wrong.

They share a shape: **the system reports on the action it took, not on the state
it produced.** Argo says "Synced" because it applied what it was told, not
because the workload is right. A lint step exits 0 because it started, not
because it finished. A registry push succeeds because the coordinate was
writable, not because the coordinate was new. Every check below is a way of
asking about state instead of action.

## How to use this

Work through the sections relevant to the diff. For each finding, report:

- **what breaks**, concretely — not "this is risky"
- **when it surfaces** — at deploy, at the next sync, on rollback, weeks later
- **what proves it** — the command whose output settles the question

Uncertainty is a finding too. "This looks like the prune case but I cannot see
the Application spec" is useful. A confident guess is not.

## 1. Sync is not convergence

`Synced` means Argo applied the manifests. It does not mean the cluster matches
your intent.

**Removal without prune.** If an Application sets `prune: false` and you delete
a resource from git, Argo leaves the live object running. The app disappears
from your IaC while the namespace, Deployment, Service and Ingress keep serving.
This is the standard way a decommission leaves a live orphan behind — and it is
worse than not deleting, because now nothing in git describes what is running.

- Deleting from git is step one of a decommission, never the whole thing.
- Enumerate what remains afterwards and delete it explicitly.
- A decommission is finished when a fresh cluster built from the repo would
  produce the same result as the live one.

**Write-back loops.** An image updater that commits `image.tag` back to the same
repo Argo watches will fight any process that also writes that field. Two
writers on one field is drift by construction. Pick one: either the updater owns
the tag, or a human does.

**Churning a generator.** An ApplicationSet generator regenerates *every* child
Application. Editing its template or generator list is not a local change — it
is a fleet-wide operation, and a mistake takes out apps you did not touch.
Change a single child app directly; treat generator edits as their own reviewed
change with a rollback plan.

**Check:**

```sh
kubectl get application -A -o custom-columns=\
NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status,\
PRUNE:.spec.syncPolicy.automated.prune
```

`Synced + Healthy` with `prune: false` and a recent deletion in git is the
orphan case. Compare against the namespace list, not against git.

## 2. Secret paths fail closed and look like absence

A secrets operator asking for a path it lacks permission to read gets a 403.
Depending on the controller, that surfaces as an unfilled key, an empty
`Secret`, or a pod stuck waiting — rarely as "permission denied".

The KV-v2 trap specifically: the API path and the policy path are not the same
string. A read of `secret/data/<path>` is **not** matched by a policy rule
written `path "secret/<path>/*"`. The rule needs the `data/` segment. The policy
looks correct, reads fail, and the error surfaces far from the cause.

- Policy rules for KV-v2 need `secret/data/...` for reads and
  `secret/metadata/...` for read+list.
- A glob does not match the bare prefix. `secret/data/apps/x/*` does not grant
  `secret/data/apps/x`. Grant both if both are read.
- Role and policy are different objects. "The X role" and "the policy attached
  to the X role" get conflated in comments and then in reasoning.
- After changing roles or policies, restart the operator. Many cache their auth
  and will keep failing with stale credentials long after the fix landed.

**Check:** resolve one secret end to end — the ExternalSecret's status, the
target `Secret`'s keys, and the pod's mounted env. Do not stop at "the
ExternalSecret exists".

## 3. Topology belongs to whoever declares it

A message broker deployed by the platform repo does not create queues. If a
consumer subscribes to a queue nobody declares, the broker returns NOT_FOUND,
the consumer fails its readiness probe, and the pod never becomes ready — for a
reason that reads like a broker problem and is actually a missing declaration in
the service.

- The service that consumes a queue declares it, in code, next to the consumer.
- The platform repo owns the broker, not the topology.
- Test it: a test that asserts every declared consumer has a matching
  declaration catches this before deploy. Nothing else does.

The general rule: when two repos could own a piece of configuration, the one
that *fails* without it should own it.

## 4. A tag is not a version

A release tag that disagrees with the version inside the manifest —
`package.json`, `pom.xml`, a `VERSION` file, `Chart.yaml` — republishes the
coordinate the manifest names. Downstream then resolves a stale artifact under a
name that implies the new one.

Worse, it is inconsistent: per-runner or per-machine caches decide which copy a
given build sees, so the same commit can succeed on one runner and fail on
another. That inconsistency is the signature — chase it as a version problem,
not a flaky-runner problem.

- Fail the pipeline when the tag and the manifest version disagree. This is
  three lines of CI and it removes the whole class.
- Republishing an already-published coordinate does not reliably overwrite it.
  Recovery is a new, unused version — not a retry.

**Embedded version drift.** If a binary embeds a version file, confirm the
embed reads the file you are bumping. A comment naming "the VERSION file at the
repo root" while the embed reads a package-local copy means bumping the root
does nothing, and the deployed service reports a version from several releases
ago. A test comparing the two files costs nothing and catches it permanently.

## 5. The environment differs from your machine in specific ways

These are container and CI defaults that do not exist locally, so they surface
only after deploy.

- **A local env file masks a missing variable.** If `.env.local` (or similar)
  supplies a value the deployed config lacks, everything works locally and fails
  in the cluster. Build once with that file renamed before shipping.
- **`HOSTNAME` is set inside containers.** Frameworks that bind to
  `process.env.HOSTNAME` bind to the container ID, and the service answers on an
  address nothing routes to. Bind `0.0.0.0` explicitly.
- **Standalone build outputs need their own entrypoint and directories.** A
  build mode that emits a self-contained server has a different start command
  and may require directories the build does not create.
- **A tool exiting 0 has not necessarily run.** Some linters exit 0 when they
  cannot acquire a lock, so a sequential chain reports clean while doing
  nothing. Assert on the tool's own output line, not the exit code.

## 6. Rollback is a claim until it is tested

Backups, replicas and snapshots are not recovery. Recovery is a restore you have
performed.

- Restore into a scratch namespace and check row counts and a known record. An
  untested backup is a hypothesis.
- Verify the dump is complete. A truncated dump compresses, stores and lists
  identically to a good one — check for the terminator its format writes.
- Store the copy somewhere the failure cannot reach. A backup on the node you
  are about to change is not a backup.
- Digest-pin images if you need to reproduce a past state. A moving tag makes
  rollback a different build than the one you ran.

## Reporting

Order findings by when they bite: at deploy, at next sync, at rollback, later.
Say what you could not check and why — "no access to the Application spec"
belongs in the report. State plainly when a change looks correct; a review that
must find something will invent something.
