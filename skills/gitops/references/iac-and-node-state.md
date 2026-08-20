# IaC and node state

Infrastructure-as-code has a quiet precondition: the tool has to know what
already exists before it can reason about what should exist. When that
precondition is unmet — no prior state, a plan built against a state that has
since moved, a fix applied by hand instead of through the repo — the tool
either fails in a way that misdirects you, or it succeeds and quietly stops
being true. Three shapes of the same problem.

## 1. Import before create

Bringing an existing resource under a state-based tool (Terraform, Pulumi,
anything that diffs desired-vs-state rather than desired-vs-live) for the
first time, with no prior state, makes the tool plan a **create** — because as
far as it knows, the resource doesn't exist yet. Applying that plan against a
resource that already exists the other side of the API returns a permission or
conflict error:

```
error creating realm: POST /admin/realms: 403 Forbidden
```

That error reads like a permissions problem. It is a state problem. The
service account is very likely scoped correctly for an **update** — the
request that fails is a **create**, which is not the operation you meant to
run. Widening the credential to allow creates "just to get the apply through"
fixes the symptom by breaking least privilege, and the next mis-ordered apply
will use the widened scope to do something worse than 403.

**The fix is ordering, not permissions:**

```sh
terraform import 'thing.name' existing-id   # 1. import BEFORE apply
terraform plan  -out tfplan                  # 2. regenerate plan AFTER import
terraform apply tfplan                       # 3. apply the fresh plan
```

Import populates state from the live object. Only then does a plan diff
correctly to an update. If a 403/409 shows up on what should be a routine
apply against something that already exists, check for an import step before
touching credentials.

## 2. A saved plan expires the moment state moves

```
Error: Saved plan is stale
```

A plan file captures the state serial at the moment it was generated. If
anything mutates state afterwards — an import, a concurrent apply, a drift
refresh — applying that saved plan is refused. This looks like friction. It is
protection: the plan was computed against a state that no longer exists, so
applying it blind could re-apply decisions made against stale assumptions
(including, in the case above, plan a create that import has since made
unnecessary).

Two practical rules follow:

- Never apply a saved plan across any operation that mutates state. Re-plan
  after import, after a manual `refresh`, after anything else touches the
  backend.
- In CI, generate and consume the plan inside the **same job**, holding the
  state lock the whole time, so nothing else can slip a mutation into the
  window between plan and apply.

## 3. A fix applied to the node dies with the node

Anything changed by hand on a running node — a config flag, a file dropped
next to a manifest, a chart installed outside the pipeline that manages
everything else — is invisible to the repository. It works, it keeps working
right up until the node is rebuilt, and then it is simply gone, with no error
and no diff to explain why the old problem is back.

The test that catches this before it bites: **if this node were rebuilt from
the repository tonight, what would be missing?** Walk every manual change you
have made to a live node or cluster member and ask whether the repo alone
would reproduce it. If the answer is no, the fix has an expiry date — the
date of the next rebuild, restart, or replacement — whether or not anyone
remembers to write it down as infrastructure before then.

This is not only a "did you forget to commit something" check. Some fixes are
*inherently* node-local by construction — a marker file that tells a
controller to stop managing a resource, an override staged outside the normal
render path — and those need the node-bootstrap process itself updated to
recreate them, not just a commit of the artifact. A fix that only a human
remembers to reapply after a rebuild is not fixed.

## The general principle

Drift is usually described as live diverging from desired state. That is only
half of it. The other half is **desired state that exists nowhere written
down** — a resource the tool doesn't know about yet, a plan computed against
a state that has since changed, a change that lives only on a machine that
will eventually be replaced. In every case the fix is the same: make the
tool's record of "what should exist" match reality *before* asking it to act,
and make sure that record survives the next time reality gets rebuilt.
