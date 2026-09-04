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

**And a tag is not a deploy.** Creating a tag starts a pipeline; it does not
publish an image, and an image updater can only roll what was published. When a
gate fails, the tag exists, the release notes exist, and the cluster keeps
running the previous image indefinitely.

Nothing announces this. The pipeline result is one click away in a system
nobody is watching at that moment, and the symptom surfaces somewhere else
entirely — a version string that looks stale, an endpoint still exhibiting a
bug you fixed. A stale version number is a deploy question, not a version
question. Verify the pipeline passed and the image was published before
believing anything shipped.

**Build-time requirements drift too, and only a tag reveals it.** A language
manifest can raise its required toolchain — a `go` directive, an `engines`
field, a target framework — while the builder image stays on the old one. Where
the toolchain is pinned to exactly what the image provides, the build refuses
rather than fetching a newer one, and the error names a version rather than a
cause.

The gap is that the image is usually built **only on a tag**. If the
requirement rose in an ordinary commit and no release was cut for weeks, the
build has been broken the whole time and nothing said so. The next release
inherits a failure it did not cause, which makes it look like the release broke
it. Check the manifest's required toolchain against the builder image whenever
either moves, and treat "the last tag predates this change" as the signal that
nothing has actually compiled it yet.

**A default in code is not the running configuration.** Where a value is
seeded into a database at first boot and read from there afterwards, changing
the seed changes nothing for any row that already exists. The code is correct,
the release ships, the deploy succeeds — and the behaviour does not move,
because the seed path is only reached when the row is absent.

This one is worse than most because every signal says success: the commit is
right, the pipeline is green, the image is deployed, the binary genuinely
contains the new default. The only thing that would have shown it is reading
the value from where the runtime actually reads it.

- **Before reporting a config change as done, query the store the running
  system reads.** Not the source, not the manifest — the row.
- **Seeding is a first-boot concern.** If a value must change for existing
  installs, that is a migration or an admin action, and saying so is part of
  the change rather than a follow-up someone else discovers.

**The version lives in more places than you bumped, and the guard runs in a
job the publisher does not wait for.** A repo with a desktop shell or a
packaged client fans the release version across several manifests — a VERSION
file, a language manifest, a Tauri or Electron config, a lock file. Drift tests
that compare them are the right defence, and they only defend the jobs that
depend on them.

Observed: VERSION and a CLI copy were bumped by hand, the tag was cut, and the
test job went red on three remaining manifests. Meanwhile the macOS desktop job
built and published, because it declared `needs: [build:web]` and nothing else.
The published artefact was tagged 2.151.0 and reported 2.150.0 from inside —
the exact failure the sync script had been written to end — while the jobs that
*did* depend on the tests were correctly skipped.

- **Every job that publishes must depend on the job that validates.** Skipped
  is a safe outcome; built-and-shipped-anyway is not. Read the `needs` of each
  publishing job and ask what it is allowed to outrun.
- **Look for the repo's own sync script before hand-editing any version.** It
  usually exists, and the drift test's failure message usually names it.
- **A sync script that covers three of four targets makes the fourth look
  optional.** If the guard has to tell the reader to run a second command,
  fold that command into the script. Partial automation is how the missing
  target survives every review — the script was run, so the job looks done.
- **Verify a version fix by running the drift tests, not by reading the
  files.** They are the only thing that knows the full list.


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
- **In a polyglot repo, "I ran the tests locally" names a language, not a
  pipeline.** A Go service with a TypeScript client has two independent local
  gates, and passing the one you think of as the real suite proves nothing
  about the other. Two consecutive releases here built no image at all: the Go
  suite, `go vet` and `golangci-lint` were all green, and `build:web` had been
  failing on four lint errors since the first of them — errors in a file added
  in that same release.

  The failure is durable because nothing corrects it. The tag exists, the
  release notes exist, the local suite is green, and the cluster quietly keeps
  the previous image. The next release inherits the break and looks like its
  cause.

  Read the CI file and run **every** job's script, in its order, for every
  language the repo contains — not the subset you remember. Where a job runs
  four steps, run four; a lint that fails on step two means steps three and four
  were never exercised either.
- **An intermittently failing test in a gating suite is usually the CODE being
  nondeterministic, not the test being badly written.** The reflex is to retry,
  quarantine, or add a sleep. Measure instead: run it a few dozen times and get
  a rate. One here failed 2 in 40 because the websocket write raced its own
  context and could complete a frame for a caller that had already cancelled —
  the test was asserting a property the code only usually had. An explicit check
  made the behaviour definite, which was the better behaviour anyway, and 200
  runs went clean. A flake you cannot explain is an unreproduced bug with a
  known trigger.
- **A pipeline reports the LAST command's status, not the failing one.**
  `run-tests | grep -v boring | head -30` exits 0 when `head` succeeds, and
  `head` succeeds whatever the test binary did — including panicking. The
  filtering idiom people reach for to make output readable is the same idiom
  that discards the result, and it is invisible precisely because the surviving
  output looks like a normal short report. Set `pipefail`, or check the status
  of the stage you care about, or do not pipe the command whose exit code is
  the answer. This one bites hardest in a wrapper or CI summary that prints
  only "exit 0" — at that point the real failure is two layers away from
  anything anyone reads.

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

## 7. A workload that never starts is not monitored by anything watching outcomes

The usual worry is a failure that is too quiet. This one is the opposite: it is
loud, continuous, and still invisible, because everything watching is watching
the wrong layer.

A scheduled job referenced a secret by a name nothing created — a rename left
the consumer pointing at the old one. The container therefore never started. The
runtime retried it **thousands of times over hours**, burning CPU the whole
while. And the job controller never recorded a single failed run, because a
failure is a container that ran and exited non-zero. Nothing ran. There was no
outcome to record.

So every layer reported honestly and the sum was silence:

- the controller had no failed jobs, because nothing completed
- the deployment was untouched and healthy
- the alerting watched job outcomes, and there were none
- the only trace was a pod stuck in a config error and a retry counter climbing

It went unnoticed for weeks.

**What to take from it:**

- **Alert on pods not in a running or completed state, with an age bound.**
  Anything failing to start for more than a few minutes is worth a signal,
  independent of what any controller says about outcomes.
- **A scheduled job with no run history is a finding, not a quiet success.**
  Check last-schedule against last-successful; a gap means it is not running,
  and "no failures" reads identically to "never ran".
- **A rename must find every reader.** Grep the whole repository for the old
  name, including manifests owned by a different application than the one you
  changed. This one was in a bootstrap tree, not with the service.
- **A retry counter is a metric.** Restart and retry counts climbing on
  something nothing else reports is often the only place the problem is
  visible.

**Fixing the schedule does not fix the run that is stuck.** This is the part
that catches people twice. A job's pod template is immutable once the job
exists, so correcting the CronJob changes what the *next* job will look like
and nothing about the one already running. The stuck job keeps recreating pods
from the old, broken template — which means deleting the pod achieves nothing,
because the job immediately makes another.

Worse, if the concurrency policy is `Forbid`, that one stuck job **suppresses
every future run**. The schedule does not fire, no new job is created, and the
last-schedule timestamp freezes on the day it broke. A daily job can sit like
this for a month while its own status field quietly reports the date it stopped.

So the check is a pair of fields, not one:

```sh
kubectl get cronjob -A -o custom-columns=\
NS:.metadata.namespace,NAME:.metadata.name,SCHED:.spec.schedule,\
LAST:.status.lastScheduleTime,LASTOK:.status.lastSuccessfulTime
```

`lastSuccessfulTime: <none>` on a job that has existed for weeks means it has
never once worked. A `lastScheduleTime` far older than the schedule implies
means something is blocking it — look for an active job, not a failing one.

**Delete the job, not the pod**, and confirm `.status.active` is empty
afterwards.

Generalised: monitoring built around *results* cannot see work that never
produced one. Ask what your alerting would do if a component simply stopped
being invoked — if the answer is nothing, that is a blind spot, not an
absence of problems.

## Adjacent skills

- **diagnosis** — the general search method when the cause is not in this
  list, or when the symptom spans systems. This skill is the specific case;
  that one is the class.
- **integrations** — when the failing dependency is a third-party API or feed
  rather than your own infrastructure.
- **auth** — when the question is an ingress gate, a token, or who may touch
  what.

## Reporting

Order findings by when they bite: at deploy, at next sync, at rollback, later.
Say what you could not check and why — "no access to the Application spec"
belongs in the report. State plainly when a change looks correct; a review that
must find something will invent something.

## References

Deeper dives for specific failure classes — read the relevant one when a
finding matches its theme:

- `references/upgrade-drift.md` — a chart, policy engine, or migration upgrade
  that leaves an orphaned resource, a control that still parses but stops
  enforcing, a crash-looping migration, or a bundled dependency version cap.
- `references/iac-and-node-state.md` — a state-based IaC tool (Terraform and
  similar) failing on import ordering or a stale saved plan, or any fix
  applied by hand to a live node instead of through the repository.
- `references/supply-chain-and-scanning.md` — a vulnerability scan gate with
  findings vendored inside a third-party binary rather than your own
  dependency graph, or a container layer that looks cleaned up but still
  contains a deleted secret.
- `references/test-environment-parity.md` — a test double (embedded database,
  mocked service, in-memory queue) that quietly narrows what the test suite
  can prove, especially missing extensions/features silently untestable in a
  lighter-weight harness.
