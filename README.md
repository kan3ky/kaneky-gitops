# claude-gitops

A Claude Code skill for reviewing Kubernetes and GitOps changes — aimed at the
failures that **pass CI, deploy green, and break anyway**.

Most Kubernetes material covers the happy path. This covers the other one: the
deploy reported success, Argo says `Synced`, every check is green, and the
cluster is quietly wrong.

## Install

```sh
git clone https://github.com/kan3ky/claude-gitops
cp -r claude-gitops/skills/gitops ~/.claude/skills/
```

No configuration. No dependencies. Ask Claude Code to review a Helm values file,
an Argo `Application`, an `ExternalSecret`, or a release pipeline and the skill
loads itself.

## What it catches

| Failure | Why it survives a green pipeline |
|---|---|
| **Orphaned resources after a decommission** | `prune: false` means deleting from git leaves the workload running. It vanishes from your IaC and keeps serving. |
| **Tag/manifest version mismatch** | A tag disagreeing with `package.json` / `pom.xml` / `VERSION` republishes the old coordinate. Per-runner caches then decide which artifact a build sees — so the same commit passes on one runner and fails on another. |
| **Secret policy paths that 403 silently** | KV-v2 reads hit `secret/data/<path>`, so a policy written `secret/<path>/*` never matches. The rule looks right; the read fails; the error surfaces far from the cause. |
| **Queue declared by the wrong repo** | A consumer subscribing to a queue nobody declares gets NOT_FOUND, fails readiness, and never starts — looking like a broker fault. |
| **ApplicationSet churn** | Editing a generator regenerates every child Application. It is a fleet-wide operation wearing the clothes of a one-line change. |
| **Image-updater write-back loops** | Two writers on one `image.tag` field is drift by construction. |
| **`HOSTNAME` inside containers** | Frameworks binding to `process.env.HOSTNAME` bind to the container ID and answer on an address nothing routes to. |
| **A linter that exits 0 without running** | Some tools exit 0 when they cannot take a lock. A sequential chain reports clean having done nothing. |
| **Embedded version drift** | If the embed reads a package-local `VERSION` but your comment says "repo root", bumping the root changes nothing and the service reports a version from several releases ago. |

## The idea behind it

Every item shares one shape:

> **The system reports on the action it took, not the state it produced.**

Argo reports `Synced` because it applied what it was given. A linter exits 0
because it started. A registry push succeeds because the coordinate was
writable, not because it was new. A backup exists because a file was created,
not because a restore works.

The skill is a set of ways to ask about **state** instead of **action**, with
the command that settles each question.

## Scope

Reviews and explains. It does not mutate a cluster, and it says what it could
not check rather than assuming. A finding names what breaks, when it surfaces,
and what proves it — and it will tell you plainly when a change looks correct,
because a review that must find something will invent something.

## Contributing

Failure reports are the most useful contribution — especially ones where the
pipeline stayed green. Include the symptom, the root cause, and the check that
would have caught it.

## Licence

MIT.
