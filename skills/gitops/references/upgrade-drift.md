# Upgrade drift

Upgrades add and change. They rarely delete, and they never tell you when a
control you still rely on has quietly stopped doing anything.

Four shapes, each seen in production.

## 1. The chart drops a resource and leaves the old one running

A chart at version N creates a `Role` and `RoleBinding`. Version N+1 folds that
functionality elsewhere and removes both from the rendered manifests. The
upgrade applies the new manifests — and leaves the old objects in the namespace,
because applying is not deleting.

Argo now sees live objects with no desired state. Without automated pruning the
Application is **permanently `OutOfSync`**, and the diff never changes:

```
Role        <ns>/<name>-tokenrequest   (live: present, desired: absent)
RoleBinding <ns>/<name>-tokenrequest   (live: present, desired: absent)
```

Syncing does not converge. It cannot: sync adds and updates, and the problem is
a leftover.

**Why it matters beyond the noise.** A permanently-OutOfSync app trains everyone
to ignore that app's status. The next real drift arrives into a signal nobody
reads.

**Fix:** one-time prune of the orphans, then verify the app reaches `Synced`.
**Check:** compare live objects against rendered manifests — `helm template` the
new version and diff resource names against `kubectl get all,role,rolebinding -n <ns>`.

## 2. The deprecated field still parses and stops working

An admission-policy engine moves its enforcement control from the policy level
to the individual rule:

```yaml
# before — policy level
spec:
  validationFailureAction: Enforce

# after — per rule
spec:
  rules:
    - validate:
        failureAction: Enforce
```

The old field still parses. No error, no warning on sync. The policy reports
`Synced / Healthy` — **and enforces nothing.**

This is the worst class in this file. A security control that fails loudly gets
fixed the same day. One that fails quiet is indistinguishable from working, and
you find out when something it should have blocked reaches production.

**Fix:** move the field, and add a test that asserts enforcement rather than
presence — deploy a manifest the policy must reject and check that it *is*
rejected. Asserting the policy exists proves nothing.

**Check, for any upgrade of a policy engine, admission controller or anything
else whose job is to say no:** find one thing it should refuse, and confirm it
refuses. Presence is not enforcement.

## 3. The migration crashes and rolls back to the previous schema

An identity server upgrade runs an automatic model migration on first boot
against an existing database. The migration reads a lazily-loaded collection
outside an open persistence session and throws:

```
LazyInitializationException: failed to lazily initialize a collection of role:
  ...RealmEntity.components, could not initialize proxy - no Session
```

The pod never reaches ready. The migration transaction rolls back, so the schema
stays at the previous version — and every restart re-runs the same migration and
hits the same crash. A clean crash loop with no partial state.

The rollback is genuinely good behaviour: a half-migrated schema would be far
worse. But it means **the upgrade is blocked, not degraded**, and no amount of
restarting helps.

**Before upgrading anything that migrates a schema on boot:**

- Restore a copy of the production database into a scratch environment and run
  the upgrade against it. This is the only test that counts.
- Know your rollback: which image tag, and whether the new version already wrote
  anything the old one cannot read.
- Check `CrashLoopBackOff` logs for migration errors specifically. "Pod not
  ready" and "migration refuses to run" look identical from the outside.

## 4. The bundled chart caps the version you need

A distribution bundles an ingress controller via its own chart. A CVE lands in
that controller. The fix is in `3.7.6`; the bundled chart pins `3.7.4` and
**hard-fails any override**:

```
requirements.yaml: This version of the Chart only supports Traefik Proxy up to v3.7.4
```

Upgrading the distribution does not help when you are already on the newest
release and its bundled chart still carries the old pin. Both obvious paths —
override the image, upgrade the platform — are closed.

**The way out** is to stop the distribution managing that component without
uninstalling it, then point the release at the upstream chart. For k3s, an
adjacent `.skip` file next to the bundled manifest does exactly that: the
helm-controller stops reconciling it, and the running workload plus its CRDs
survive untouched. Then repoint the `HelmChart` CR at the upstream repository
and a chart version whose appVersion carries the fix, staying within the same
minor so the CRDs remain compatible.

**The part that bites later:** that `.skip` file lives on the node, not in your
repository. A rebuilt node has no memory of it and silently returns to the
capped bundled chart, reintroducing the CVE. If you take this route, the node
bootstrap must recreate the file and the replacement `HelmChart` CR must be
committed as IaC — otherwise the fix has a lifespan of exactly one node.

**General rule:** any fix applied to a node rather than to a repository is a fix
with an expiry date. Write it down as infrastructure or accept losing it.
