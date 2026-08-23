# Test environment parity

Green tests are a claim: "this works the way production works." That claim is
only as strong as the environment the tests actually ran in. Test doubles —
embedded databases, in-memory queues, mocked auth, anything standing in for
the real infrastructure — buy speed by being smaller than the real thing, and
"smaller" always means something specific was left out. The question that
matters is whether anyone decided to leave it out, or whether it was simply
never noticed.

## 1. An embedded database ships the server, not the ecosystem

An in-process/embedded database distribution (the kind that downloads a real
server binary and runs it locally for tests, no container required) typically
ships a **vanilla** build of that database — the core server only, no
third-party extensions, and no hook to install one at runtime beyond what the
server itself supports. If a migration does `CREATE EXTENSION <name>` for
anything outside the core distribution, it fails at migrate time:

```
ERROR: could not open extension control file ".../share/extension/<name>.control": No such file or directory
```

Every feature that depends on that extension — vector search, a specific
index type, a language extension, anything not compiled into core — is
untestable in that harness, silently, until the migration is actually run.
The test suite doesn't warn you the coverage gap exists; it just never
exercises that path, and everything reports green because nothing that would
fail ever ran.

## 2. The trade-off is real — it has to be a decision

Choosing an embedded database for test speed is a legitimate choice: no
container startup, no Docker dependency in CI, fast local iteration. None of
that is wrong. What's wrong is not noticing that the choice **narrows what
your tests can prove** the moment a feature needs something outside the
vanilla server. A team that picked the embedded DB for speed and later added
an extension-dependent feature without revisiting that choice has a testing
gap nobody chose — they have a discovery, not a decision, and it surfaces as
a production bug that "all tests passed" a moment before it shipped.

## 3. Options, with their honest costs

- **Containerised real database.** Slower (container startup on every run,
  or a shared long-lived container to amortise it), but it runs the actual
  extension set your production database runs. This is the only option that
  is faithful by default rather than by extra effort.
- **Inject the missing pieces into the embedded distribution.** Extensions
  are typically loaded from a `control` file plus a shared library at
  `CREATE EXTENSION` time, not at server start — so it's often possible to
  drop a matching prebuilt extension binary into the embedded server's
  extension directory after it starts but before migrations run. This keeps
  the speed but adds a real maintenance burden: the extension build has to
  match the database's major version and target platform exactly, and it's
  now something your team owns and has to re-source on every version bump.
- **Skip the dependent tests, explicitly.** Mark them (a skip reason, a tag,
  a `t.Skip("requires extension X, not available in embedded PG")`) so the
  gap shows up in test output and in coverage reports instead of silently
  passing zero assertions. An explicit skip is honest about what wasn't
  proven. A test that quietly never runs the failing path is not.

None of these is universally correct. The point is picking one on purpose and
being able to say, out loud, what production behaviour your test suite does
and doesn't cover as a result.

## 4. The general shape: any smaller stand-in is an assertion about what doesn't matter

This generalises past databases. An in-memory queue instead of the real
broker, a stubbed auth service instead of the real identity provider, a
fake payment gateway instead of the real one — every one of these is "the
real thing, but smaller," and every one of them is implicitly asserting that
whatever got removed to make it smaller doesn't matter to the behaviour under
test. Sometimes that's true. The failure mode is never checking, and finding
out the hard way that the removed part was load-bearing.

When you introduce a test double, or inherit one, write down what it leaves
out relative to the real thing — not just what it provides. "This queue
double doesn't replicate at-least-once redelivery" or "this embedded database
has no vector extension" is one sentence, and it's the sentence that tells
the next person which production bugs this test suite structurally cannot
catch.

## 5. A test that skips is reported as a test that passed

The doubles above narrow what a suite can prove. This one is sharper: the test
does not run at all, and the suite still reports success.

The usual shape is a container-backed test that begins by starting its
dependency and returning early when it cannot:

```go
c := testdb.Start(t)
if c == nil {
    return   // no container runtime here — nothing runs, nothing fails
}
```

That early return is correct. Failing the suite on every machine without a
container runtime would make it unrunnable, and a suite people cannot run
locally is a suite they stop running. The problem is what the runner prints
afterwards: `ok`. Identical to the line printed when the assertions executed and
held.

So the developer machine — the one place a mistake is cheap to fix — is
precisely where the check is absent, and nothing on screen says so. It surfaces
later, in CI, attributed to whoever pushed rather than to whoever wrote it.

A worked example. A migration adds a column with a `NOT NULL DEFAULT`. A
database round-trip test compares the whole struct with a deep equality check,
so the new default arrives on the read side and the expected literal still has
the zero value. Locally: green, because no container, so no test. In CI: a
failure in a file the author never opened.

Three things follow:

- **Know which of your tests are gated on something you do not have**, and
  treat the local run as partial until you have listed them. "The suite passed"
  is a claim about the tests that ran.
- **Run what CI runs, including the build tags.** A suite invoked without the
  tag that unlocks the integration tests is a different suite. This is the
  cheapest of the three and the most commonly skipped.
- **When behaviour can only be asserted behind a gated test, assert it a
  second time somewhere ungated.** A mocked or in-process path that pins the
  same property is not redundant — it is the copy that runs on the machine
  where the code is being written. Say so in a comment, or someone will delete
  it as a duplicate.

The generalisation, and the reason this sits in this skill: **a check that
cannot run in an environment provides no coverage in that environment, and
almost every runner reports "did not run" and "ran and passed" with the same
word.** A whole-struct equality assertion is a good and deliberate choice —
it forces a decision about every new field instead of ignoring it — but it
converts every column addition into a two-file change, and it will only tell
you that in the environment where it is allowed to run.


## 6. A fixture looser than the storage it stands for

A test double is usually judged on whether it behaves like the real thing. There
is a narrower question that catches more: **does it constrain what the real
thing constrains?**

A worked example. An embedding endpoint was stubbed with a three-element vector.
The column it feeds is `vector(1536)`. Every test passed for months. Then an
operator selected a better embedding model in the settings UI — one returning
3072 dimensions — and every memory write began failing, along with a semantic
search feature shipped that same day. The suite could not have caught it: the
fixture was more permissive than the database, so width was the one property
the tests were structurally unable to check.

The same shape appears wherever a double is *smaller* than the real thing in a
dimension that matters: a stub returning two rows where the schema enforces a
unique key, a fake queue that never redelivers, a mock clock that never moves
backwards.

- **Make fixtures as strict as the storage.** If a column is fixed-width, the
  fixture is that width. If a field is `NOT NULL`, the fixture sets it.
- **When a fixture is deliberately smaller, say what that costs.** One sentence
  naming the property it can no longer prove.
- Watch for tests that assert on a *small* returned value — `len(got) != 3` —
  when production values are large. That assertion is usually a fixture
  documenting its own laxity.

## 7. A migration can pass its guard and still break boot

A guard test that every migration file is registered is worth having; it catches
files that silently never run. It does not catch a file that runs and fails.

The failure: a new seed's `INSERT` column list was copied from the OLDEST
migration in the series, naming a column later migrations had renamed. It
registered correctly, passed the guard, and took the service down at startup
with `column "badge" does not exist`.

- **Copy a column list from the most recent insert, not the first one.** Schemas
  drift forward; the oldest example is the least likely to still be correct.
- **A guard that checks registration is not a guard that checks validity.** The
  only thing that proves a migration applies is applying it, which means a
  disposable database in the loop before release.
- If the environment cannot run migrations locally, that is itself a finding —
  see section 5. It means the first execution happens in front of users.
