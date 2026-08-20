# Supply-chain and scanning

A vulnerability gate that nobody can pass gets disabled. Not maliciously — a
team facing a blocked pipeline and a finding it has no way to fix will reach
for `allow_failure`, a blanket ignore file, or turning the check off, and once
that muscle memory exists it gets used on the next finding too, including the
ones that were genuinely fixable. A gate's whole value is that passing it
means something; a gate that can be permanently unpassable trains people to
stop trying.

## 1. A binary carries what it was built with, not what's newest

Upgrading a compiled tool to its latest release does not necessarily upgrade
the dependencies inside it. A Go binary embeds the exact module versions and
the exact language stdlib it was compiled against at release time. If the
maintainers cut that release before bumping a dependency, the binary ships the
old one — regardless of how new the release itself is.

Concretely: a CLI tool sitting on its newest available version can still fail
a scanner for a stdlib CVE that was patched upstream **after** that release
was built. There may be no newer release of the tool at all yet. "Upgrade to
latest" is not a fix here — it's already been applied, and the finding
remains.

## 2. Building from source doesn't help either

The instinct is to build the tool yourself against current dependencies. It
doesn't work: a source build resolves the *same* pinned dependency graph the
maintainers committed (their lockfile, their `go.mod`/equivalent), not
whatever is newest today. You'd need to fork the tool and edit its own
dependency pins — sustainable for one tool in an emergency, not for the
handful of third-party binaries any CI pipeline invokes.

## 3. Some findings are genuinely unfixable by you, at any version

This is the case a strict "zero HIGH/CRITICAL, no exceptions" gate doesn't
model: a finding that is real, inside a widely-used tool, at its latest
release, with the fix already merged upstream but not yet in a shipped build.
No version bump available to you clears it. A gate that fails the pipeline on
it anyway is not enforcing security — it's enforcing an impossible task, and
the team will route around it.

**The fix is to split the gate, not to weaken it:**

- Fail the build on findings you can actually fix — a version bump, a config
  change, a dependency you control.
- Track, don't silently suppress, findings vendored inside a third-party
  binary you merely invoke. An ignore entry gets: the CVE id, which tool owns
  it, the version it's fixed in upstream, and a **review date** to re-check
  whether that fix has shipped yet.
- Revisit the tracked list on its review date and delete every line whose
  upstream has caught up. An ignore file that only grows is an ignore file
  nobody reads, which is indistinguishable from no gate at all.

An unexplained suppression is indistinguishable from an oversight — from the
next reviewer's seat, a silent ignore-all and a forgotten scan look the same.
Always attach the reasoning to the exception, not just the exception.

## 4. Reading a scanner report: whose dependency graph is this?

Before triaging any finding, place it on one side of this line:

- **In your own dependency graph** (your `go.mod`, `package.json`,
  `pom.xml`, base-image packages you chose) — you can fix this by bumping a
  version or swapping a package. Fail the build on it.
- **Vendored inside a third-party tool you invoke** (the Go stdlib compiled
  into a CLI, a library statically linked into someone else's binary) — you
  cannot fix this without the vendor shipping a rebuild. Track it with a
  review date instead of failing on it.

A finding's severity label doesn't tell you which side it's on — only tracing
where the vulnerable code actually lives does. In practice: check whether the
CVE is against a package that appears in *your* lockfile, or only inside a
tool's own release artifact. The two require completely different responses,
and treating them the same is what turns a useful gate into a permanently
red pipeline someone eventually mutes.

## 5. A cleanup layer can look like it removed something and not have

A related trap in the same territory: deleting a secret or fixture in a
**later** layer of a container build removes it from the final filesystem
view but not from the image. Each layer is an immutable blob; a later `RUN rm`
adds a new layer saying "this file is gone" on top of an earlier layer that
still contains it in full, byte for byte extractable — the same relationship
`git rm` has to git history. `ls` inside the running container shows nothing;
a layer-aware scanner (or anyone who pulls the image and inspects its layers
directly) still finds it.

The fix is to never let the sensitive file exist in a layer that survives:
delete it in the **same** `RUN` instruction that created it, or avoid writing
it into the build context at all. A trailing "cleanup" step is not cleanup
from the image's perspective — check for this pattern anywhere a build
generates and then deletes credentials, test fixtures, or keys.
