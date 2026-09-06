# Fork branches and local releases

This is a fork of [dj95/zjstatus](https://github.com/dj95/zjstatus). It exists
both to contribute changes back upstream and to run builds that upstream has not
shipped. Those two purposes pull in opposite directions: contributions need a
history identical to upstream's, while running your own build needs commits
upstream does not have. The branch layout below keeps them apart.

## The branches

**`main`** is a mirror of `upstream/main` and nothing else. No fork-only commit
ever lands here. Keeping it byte-identical to upstream is what makes a pull
request from this fork show only the change it proposes.

**`main-local`** is the integration branch. It carries `main` plus every
fork-only commit: this document, the local release workflow, and any work that
is finished enough to run but not yet accepted upstream. It is merged *into*,
never branched *from* for upstream work. Its history is expected to diverge from
upstream permanently.

**Work branches** are branched from `main`, one per change you intend to
propose. Because they start from an exact copy of upstream, the pull request
they open contains only their own commits. A work branch you also want to run
locally gets merged into `main-local` as well, which is what puts it into a
local release without entangling it with the fork's tooling commits.

```
upstream/main ──▶ main ──┬──▶ work branch ──▶ PR to dj95/zjstatus
                         │         │
                         │         └──▶ merged into main-local (to run it)
                         └──▶ main-local ──▶ fork-v* tag ──▶ fork release
```

## Syncing from upstream

`.github/workflows/sync-upstream.yml` runs daily at 06:00 UTC, and on demand
from the Actions tab. It fast-forwards `main` to `upstream/main` and mirrors any
new upstream tags. If the fast-forward is not possible, it fails rather than
forcing: that means something has been committed to `main` that upstream does
not have, which should be moved to `main-local` so `main` can be reset to
`upstream/main`.

`main-local` is deliberately left alone by that job. Merging `main` into it is
the one step that can conflict, exactly when upstream lands a change the fork
already carries, and that is a decision to make rather than something to
discover from a failed overnight run. Each run's summary says whether
`main-local` has fallen behind, and the merge is a two-line job:

```sh
git switch main-local && git fetch origin
git merge origin/main
```

The scheduled trigger is also why `main-local` is this fork's default branch:
GitHub only runs `schedule` workflows from the default branch, and the sync
workflow is a fork-only file that must never land on `main`.

To sync by hand, upstream is a second remote, added once per clone:

```sh
git remote add upstream https://github.com/dj95/zjstatus.git
git fetch upstream
git switch main && git merge --ff-only upstream/main
```

### Where upstream's tags live

Upstream's tags are mirrored to `refs/upstream-tags/*`, not `refs/tags/*`. They
are kept because `git-cliff` needs a previous tag to measure a release's notes
against, and hiding them from `refs/tags/` buys two things: this fork's tag list
shows its own releases rather than 43 of upstream's, and no `v*.*.*` tag ever
exists here for the inherited `release.yml` to fire on.

They are invisible to a normal `git fetch`. To read them:

```sh
git ls-remote origin | grep refs/upstream-tags/     # list them
git fetch origin "+refs/upstream-tags/*:refs/tags/*"  # fetch as local tags
```

The release workflow does that fetch itself before generating a changelog, so
nothing needs doing by hand for a release.

### When upstream lands something the fork already has

Once a work branch's pull request is accepted, upstream's copy of that change
arrives through the sync above, while `main-local` still carries the copy merged
in earlier. Git compares content, not pull request numbers, so what happens next
depends only on whether the two copies produced the same text:

- Identical content merges silently. Nothing to do.
- Content that drifted apart, because the change was squashed, reformatted or
  revised during review, raises a conflict on the affected lines. Resolve it in
  favour of upstream's version. Staying aligned with upstream is the point of
  the sync, and the fork's copy has served its purpose.

A local commit whose content has landed upstream can then be dropped from
`main-local` entirely by rebasing the branch onto `main`. This is optional
housekeeping, worth doing when the same conflict keeps reappearing on later
syncs.

## Cutting a local release

Releases are built by `.github/workflows/release-local.yml`, which runs on
pushed tags matching `fork-v*`.

```sh
git switch main-local
git push origin main-local          # push the branch before the tag
git tag -a fork-v0.25.1 -m "fork-v0.25.1"
git push origin fork-v0.25.1
```

The workflow runs `cargo nextest run --lib --bins`, and only if that passes
builds `zjstatus.wasm` and `zjframes.wasm` and publishes them as a release on
this fork, with notes generated by `git-cliff` covering the commits since the
previous tag. The notes state that the artifacts are a fork build and name the
exact commit they came from, since the release page is the only context a
reader gets.

Because those notes make a claim about where the code came from, the workflow
checks the claim before building. It requires the tagged commit to be contained
in `origin/main-local` and *not* contained in `origin/main`. That rejects the
two ways of tagging something that is not a fork build: a commit that never
reached `main-local`, such as an unmerged work branch or a `main-local` that was
never pushed, and a commit on `main`, which mirrors upstream and carries no fork
work at all.

A tag that fails the check is already on the remote, since pushing a tag always
succeeds and only the workflow run fails. Remove it before retrying:

```sh
git push --delete origin fork-v0.25.1
git tag -d fork-v0.25.1
```

### Why the tag prefix

`main-local` inherits upstream's `.github/workflows/release.yml`, which triggers
on `v*.*.*` tags. A tag named `v0.25.1` would therefore start two release runs
competing to publish the same tag. The `fork-v` prefix matches only the local
release workflow's filter, which leaves upstream's file untouched and free to
merge cleanly on every sync.

Mirroring upstream's tags out of `refs/tags/` means no `v*.*.*` tag is expected
to exist on this fork at all, so the prefix is now the second of two defences
rather than the only one. It still matters: it is what makes tagging one by hand
harmless.

Upstream's `lint.yml` still runs, since its bare `on: push` fires for tags as
well as branches. Its clippy and test jobs duplicate some of the release
workflow's own test job, which is deliberate: one workflow cannot gate another,
so tests that decide whether a release is published have to live inside the
release workflow.

For the same reason, do not use the `just release` recipe here. It bumps the
version, tags it `v*` and pushes to `main`, all of which is upstream's release
process rather than this one.
