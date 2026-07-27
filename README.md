# Contribution 1: Fix numpy>=2.3 type-check failures in tqec

**Contribution Number:** 1
**Student:** Sagar Sinha
**Issue:** [tqec/tqec #613 — "Fix mypy failures for numpy>=2.3"](https://github.com/tqec/tqec/issues/613)
**Status:** ✅ **COMPLETE (2026-07-06)** — tqec PR [#977](https://github.com/tqec/tqec/pull/977) **merged by @nelimee** (all 11 checks green), closing [#613](https://github.com/tqec/tqec/issues/613) end-to-end. The cross-repo unblocker [tqecd#71](https://github.com/tqec/tqecd/pull/71) was also **merged** and **tqecd 0.2.1 released** on PyPI — two merged contributions across two repos.

---

## Why I Chose This Issue

I chose [tqec/tqec #613](https://github.com/tqec/tqec/issues/613) in [tqec/tqec](https://github.com/tqec/tqec), a design-automation library for representing, constructing, and compiling large-scale fault-tolerant quantum computations (Topological Quantum Error Correction, based on the surface code and lattice surgery). The issue asks to fix static type-checking failures that appear when `numpy>=2.3` is installed: numpy 2.3 tightened the type stubs for its scalar types (`signedinteger`, `floating`), and the type checker now flags places in `tqec` that iterate over, index with, or pass those scalars where the new stubs no longer allow it.

I chose it for three reasons:

1. **Tightly scoped and well-specified.** The issue lists the exact files and line numbers where the errors occur (`subtemplates.py`, `display.py`, `_testing.py`, `template.py`, `generation.py`, `database.py`, `rotations.py`), so the surface area is small and unambiguous.
2. **Non-quantum.** It is labeled `non-quantum` and `good first issue` — it is about Python typing, not surface-code physics, so I can contribute correctly without deep QEC domain expertise.
3. **Clear definition of done.** The fix is complete when the project's type checker passes against `numpy>=2.3`.

**Issue status note (important and verified):** The issue title says "mypy," but the project currently type-checks with **`ty`** (Astral's type checker) via `uv run ty check` in CI — there is no `mypy` configured anywhere in the repo. `numpy` is also currently capped at `<2.3` in [`pyproject.toml`](https://github.com/tqec/tqec/blob/main/pyproject.toml) (upper bound added in PR #659), so the failing version is not installed by default; reproducing the errors requires lifting that cap. The reporter (`purva-thakre`) further notes the errors "may fix themselves once a stable numpy 2.3 is released," and there is live upstream work in PR #659. I am officially assigned to the issue (`sinhasagar507`). I will verify the exact current state — which errors still reproduce under `numpy>=2.3`, and against `ty` vs. `mypy` — in Phase II before writing the fix.

---

## Understanding the Issue

### Problem Description

numpy 2.3 tightened the type stubs for its scalar types (e.g. `np.signedinteger`, `np.floating`). Code in tqec that iterates over, indexes into, or passes those scalars in ways the new stubs no longer permit fails the static type checker. The issue asks to fix all such failures so `uv run ty check` passes cleanly when numpy>=2.3 is installed.

### Expected Behavior

`uv run ty check` exits 0 with numpy>=2.3 installed, and all existing tests still pass.

### Current Behavior

`pyproject.toml` caps numpy at `<2.3` (`"numpy>=1.22,<2.3"`, line 44) specifically to avoid these type failures. The cap was added conservatively in PR #659 — the type errors themselves were never fixed.

### Affected Components

The issue originally listed these files as containing numpy scalar type errors:

- `src/tqec/templates/subtemplates.py`
- `src/tqec/display.py`
- `src/tqec/_testing.py`
- `src/tqec/templates/template.py`
- `src/tqec/compile/generation.py`
- `src/tqec/compile/database.py`
- `src/tqec/templates/rotations.py`

---

## Reproduction Process

### Environment Setup

- Python 3.13, `uv` (installed to `~/.local/bin`)
- Cloned `tqec/tqec` into `/Applications/saggydev/projects_learning/su26-ai301-contribution1-sagar/tqec`
- Created working branch `fix/613-numpy-2.3-type-check`
- Lifted the `numpy<2.3` cap in `pyproject.toml` to install numpy 2.4.6 (latest available)

```bash
git clone https://github.com/tqec/tqec.git
cd tqec
git checkout -b fix/613-numpy-2.3-type-check
# edit pyproject.toml: remove "<2.3" from the numpy constraint
uv sync
uv run ty check
uv run pytest --tb=short -q
```

### Steps to Reproduce

1. Lift the `numpy<2.3` cap in `pyproject.toml` and run `uv sync` — this installs numpy 2.4.6.
2. Run `uv run ty check` to observe type-checker output.
3. Run `uv run pytest --tb=short -q` to verify the test suite.

### Reproduction Evidence

**`ty check` result (numpy 2.4.6, Python 3.13):** exit 0 — only 3 pre-existing `redundant-cast` warnings unrelated to the issue. No numpy scalar type errors.

**Test suite result:** 788 passed, 0 failed.

**Key finding — the true blocker:** The type errors from the original issue report are essentially gone under `ty` (the repo migrated from mypy to Astral's `ty` checker; the issue title still says "mypy" but there is no mypy config in the repo). The real blocker is a cross-repo dependency cap: `tqecd==0.2.0` (tqec's sibling decoder package) pins `numpy>=1.22,<2.3`, and it is the latest published release. tqec mirrors this cap at `pyproject.toml:44`. The cap cannot be lifted in tqec until `tqecd` ships a new release without the `<2.3` upper bound.

**Runtime note:** tqecd runs fine with numpy 2.4.6 at runtime — the cap is conservative metadata, not a real incompatibility.

**Maintainer contact (2026-06-20):** A maintainer replied on the issue inviting a fresh PR coupling the numpy version bump with any remaining type fixes, superseding the stale PR #659.

### Phase III — Refined Reproduction (Build, 2026-06-22)

The Phase II evidence above was correct as far as it went, but Phase III tightened it by forcing numpy ≥ 2.3 deterministically and testing each minor version. Two things changed the picture.

**Method.** With the `<2.3` cap still present, `uv` resolves numpy to **2.2.6** (not 2.4.6) — the `tqecd 0.2.0` transitive `numpy<2.3` cap drags it below 2.3. To exercise the real numpy ≥ 2.3 scenario I added a temporary, repro-only override (removed before any commit) that beats *both* caps:

```toml
# pyproject.toml — TEMPORARY, reproduction only
[tool.uv]
override-dependencies = ["numpy>=2.3; python_version >= '3.11'"]
```

The `python_version >= '3.11'` marker is required: **numpy ≥ 2.3 dropped Python 3.10 support**, and tqec still targets 3.10 (`requires-python = ">=3.10,<3.14"`). A bare `numpy>=2.3` override is *unsatisfiable* on the 3.10 leg, so the real fix must lift the cap without forcing 2.3 onto 3.10.

**`ty check` per numpy version (Python 3.13):**

| numpy | `ty check` | Notes |
| --- | --- | --- |
| 2.2.6 (capped default) | ✅ clean | what the `<2.3` cap actually installs today |
| 2.3.5 | ✅ `All checks passed!` | all 7 issue-listed files clean |
| 2.4.6 | ✅ `All checks passed!` | all 7 issue-listed files clean |
| 2.5.0 | ❌ **1 error** | new overload error, see below |

**The one real error (numpy 2.5.0 only):** numpy 2.5.0 tightened the `ufunc.reduce` type stubs, surfacing a single failure in a file the issue never listed:

```text
error[no-matching-overload]: No overload of bound method `_UFunc_Nin2_Nout1.reduce` matches arguments
  --> src/tqec/compile/blocks/layers/merge.py:177
  num_internal_layers = numpy.lcm.reduce(considered_timesteps)
```

`considered_timesteps` is a `list[int]`, so the minimal, idiomatic fix is `math.lcm(*considered_timesteps)` — a pure-`int` result, no numpy scalar, available on Python ≥ 3.9 (tqec targets ≥ 3.10).

**Test suite at numpy 2.5.0:** **788 passed**, 257 slow-deselected — i.e. this is *purely* a static-typing issue; runtime is correct. (The run emits numpy-2.5 deprecation warnings, but only from the third-party `pycollada` dependency, not from tqec.)

**Refined conclusion.** Supporting numpy ≥ 2.3 is *two* coupled changes, not one: (a) lift tqec's `<2.3` cap — clean through numpy 2.4 with **zero** source edits; (b) one-line `merge.py` type fix to also stay clean on numpy 2.5. The *resolution* blocker is unchanged: `tqecd 0.2.0`'s transitive `numpy<2.3` cap still has to be lifted (tqecd release) or overridden in tqec for CI to actually install numpy ≥ 2.3.

---

## Solution Approach

### Analysis

There are two parts to the fix:

1. **tqecd (upstream blocker):** `tqecd` must release a version that drops the `numpy<2.3` upper bound. Without this, `uv sync` in tqec cannot resolve numpy>=2.3 regardless of what tqec's own `pyproject.toml` says.
2. **tqec (the actual PR):** Once tqecd publishes an uncapped release, the tqec PR lifts the matching cap and relocks so CI exercises numpy 2.3+.

### Proposed Solution

Lift the numpy version cap (a dependency-metadata change) **and** apply one small source fix. *Updated by Phase III ([Refined Reproduction](#phase-iii--refined-reproduction-build-2026-06-22)):* the original "no source rewrites needed" assumption holds through numpy 2.4, but numpy 2.5.0 surfaces one real `ty` error in `compile/blocks/layers/merge.py` (`numpy.lcm.reduce` → `math.lcm`). The PR therefore couples the cap-lift with that one-line fix, exactly the bump-plus-type-fix the maintainer requested.

### Implementation Plan

Using the UMPIRE framework (adapted):

**Understand:** The `numpy<2.3` cap in tqec exists solely because tqecd has not published an uncapped release. The type-checker errors that motivated the cap are no longer present under `ty` with numpy 2.4.6.

**Match:** The fix is a `pyproject.toml` + `uv.lock` change. Pattern: lift the cap, relock, confirm CI passes.

**Plan:**

1. Raise the tqecd blocker with the maintainer — ask whether tqecd 0.2.1 (uncapped) is planned, or whether tqec should override the cap temporarily.
2. Once tqecd is released: edit `pyproject.toml` — change `"tqecd>=0.2.0"` to `"tqecd>=0.2.1"` and remove the `<2.3` upper bound from the numpy constraint.
3. Run `uv lock` to relock; run `uv run ty check` and `uv run pytest` to confirm clean.
4. Open a PR against `main` on branch `fix/613-numpy-2.3-type-check`.

**Review:** tqec uses `uv` + `ty` + `ruff`; follow `CONTRIBUTING.md`. Only `pyproject.toml` and `uv.lock` change.

**Evaluate:** `uv run ty check` exits 0, `uv run pytest` passes 788+ tests, CI exercises numpy 2.3+.

---

## Testing Strategy

All gates are the project's real CI commands (`uv run ty check`, `uv run ruff check`, `uv run pytest`). numpy ≥ 2.3 was forced locally via a temporary, uncommitted `uv` override (see [Refined Reproduction](#phase-iii--refined-reproduction-build-2026-06-22)).

### Static Type Check

- [x] `uv run ty check` — **clean at numpy 2.3.5, 2.4.6, and 2.5.1** (numpy 2.5 was the only failing version; clean after the `merge.py` fix), and in the committed state (numpy 2.2.6). Re-verified on the maintainer-merged base (2026-07-05).

### Unit / Integration Tests

- [x] `uv run pytest` — **788 passed** at both the default lock (numpy 2.2.6) and forced numpy 2.5.1 (`merge_test.py`: 9 passed). No runtime behavior change.
- [x] **Line coverage 92%** overall — above the project's 80% bar (re-run on the merged base, 2026-07-05).

### Code Health

- [x] `uv run ruff check` — clean (import swap is tidy, no unused import left behind).

### Regression Guard

The CI `ty check` job is the guard: once `tqecd` lifts its `numpy<2.3` cap and the lockfile resolves numpy ≥ 2.3, the existing matrix job exercises this automatically. No CI change was added — to keep the PR minimal and avoid committing a uv-only override into CI (discussed in the PR).

---

## Implementation Notes

### Implementation Progress

Phase III is **complete** on the tqec side — two small, independent commits on branch `fix/613-numpy-2.3-type-check`:

1. **Type fix** (`compile/blocks/layers/merge.py`): `numpy.lcm.reduce(considered_timesteps)` → `math.lcm(*considered_timesteps)`. numpy 2.5 tightened the `ufunc.reduce` stubs, making the old call fail `ty`; the input is a non-empty `list[int]`, so `math.lcm` is equivalent, returns a plain `int`, and removes the file's only numpy import.
2. **Dependency cap-lift** (`pyproject.toml` + `uv.lock`): removed the `numpy<2.3` upper bound (added in #659) and relocked.

The original `signedinteger` errors from #613 are already gone under `ty` (clean at numpy 2.3.5/2.4.6); the only remaining numpy ≥ 2.3 error was the numpy-2.5 `lcm.reduce` one fixed above.

### Code Changes

- **Branch:** [`fix/613-numpy-2.3-type-check`](https://github.com/sinhasagar507/tqec/tree/fix/613-numpy-2.3-type-check) (fork `sinhasagar507/tqec`)
- **Commits:**
  - [`2b026fb8` — Replace numpy.lcm.reduce with math.lcm](https://github.com/sinhasagar507/tqec/commit/2b026fb88)
  - [`663d03b8` — Lift the numpy<2.3 upper bound](https://github.com/sinhasagar507/tqec/commit/663d03b80)
  - [`2b82c11b5` — Require tqecd>=0.2.1 to allow numpy>=2.3](https://github.com/sinhasagar507/tqec/commit/2b82c11b5) (2026-07-05, on top of the maintainer's `main` merge)
  - `Update pyproject.toml` — applied @nelimee's review suggestion, 2026-07-06
- **PR:** [tqec/tqec#977](https://github.com/tqec/tqec/pull/977) — **merged 2026-07-06**

### Challenges Faced

- **The literal task had shifted.** The issue says "mypy," but the repo type-checks with `ty`, and the originally-listed `signedinteger` errors had already resolved once numpy 2.3 stabilized. Confirming this required forcing numpy ≥ 2.3 deterministically (a `uv` override scoped to Python ≥ 3.11, since numpy ≥ 2.3 dropped 3.10) and testing each minor version — which surfaced the real, current error (numpy 2.5 / `merge.py`) the issue never listed.
- **The real blocker is cross-repo.** Lifting tqec's own cap doesn't resolve numpy ≥ 2.3 because `tqecd 0.2.0` transitively pins `numpy<2.3`. That kept the PR a draft pending a tqecd release (or a temporary `uv` override) — raised with the maintainer on #613, then fixed at the source (see [Blocker Cleared](#blocker-cleared--un-block-2026-07-05)).
- **The bump rewrites a lot of `uv.lock`, but only one version changed.** Bumping `tqecd>=0.2.1` touched ~144 lock lines; the only version change is tqecd 0.2.0→0.2.1. Removing tqecd's numpy cap shifts uv's resolution forks, so uv re-derives per-fork `python_full_version` markers across the transitive graph. Verified genuine, not noise: `uv lock --locked` passes on it, while a hand-minimized diff would fail that check (and CI).

---

## Pull Request

**PR Link:** [tqec/tqec#977 — Lift numpy<2.3 cap and fix numpy 2.5 ty error](https://github.com/tqec/tqec/pull/977) — **merged 2026-07-06** (supersedes the stale #659).

**PR Description (summary):** Three changes to support `numpy>=2.3`: (a) a one-line type fix in `compile/blocks/layers/merge.py` (`numpy.lcm.reduce(...)` → `math.lcm(*...)`, the only `ty` error surfaced by numpy 2.5), (b) lifting tqec's `numpy<2.3` upper bound in `pyproject.toml` + relocking, and (c) bumping `tqecd>=0.2.1` (first tqecd release without its own `numpy<2.3` cap). `ty` clean at numpy 2.3.5 / 2.4.6 / 2.5.1; 788 tests pass; 92% coverage. Closes #613.

**Status:** ✅ **Merged 2026-07-06** by @nelimee, closing #613 — see [Merged](#merged-2026-07-06) for the approve → suggestion → merge sequence.

### Maintainer Feedback

The feedback loop on [issue #613](https://github.com/tqec/tqec/issues/613) (maintainer **@nelimee**, cc **@purva-thakre**):

- **Jun 19** — @nelimee confirmed PR #659 was stalled (waiting on a `stim` release that has since shipped) and invited a fresh PR; I was assigned to the issue.
- **Jun 23** — I posted the full investigation: the original `signedinteger` errors are gone under `ty` (clean at numpy 2.3.5 / 2.4.6), numpy 2.5 surfaces exactly one new error in `merge.py` (fixed), and the real blocker is cross-repo — `tqecd 0.2.0` transitively pins `numpy<2.3`, so lifting tqec's own cap alone still resolves numpy to 2.2.x.
- **Jun 23** — @nelimee replied: *"I'll try to bump `tqecd`, good catch. Keeping you updated."* (path A — an uncapped `tqecd` release).
- **Jun 23** — I confirmed the plan: once the new `tqecd` is on PyPI I'll bump `tqecd>=0.2.x`, relock, and flip #977 to ready-for-review.
- **Jul 1** — polite follow-up (LT4 etiquette, ~1 week after his last reply) confirming the `tqecd` bump was still on his radar.
- **Jul 3** — @nelimee replied: *"It is, but I won't have the time to do it in the next few days. Seems like that could be a good first contribution to `tqecd` from you too :) Sorry for flagging that as 'I'll do it' and blocking you."* — **the path flipped from A to B: the maintainer invited me to open the `tqecd` cap-lift myself.**
- **Jul 4** — acknowledged on #613, then opened the parallel `tqecd` contribution (see below).
- **Jul 6** — @nelimee **approved** #977 and offered to carry it the rest of the way himself: *"You have nothing more to do, I'll click on merge as soon as CI passes."* (mechanics in [Merged](#merged-2026-07-06) below).

### Parallel Contribution — `tqecd` cap-lift (the actual unblocker)

The cross-repo blocker is fixed where it actually lives — in `tqecd`:

- **Issue:** [tqec/tqecd#70 — Lift the numpy<2.3 upper bound](https://github.com/tqec/tqecd/issues/70) (`enhancement`; follows tqecd's issue-first contributor guide).
- **PR:** [tqec/tqecd#71 — Lift the numpy<2.3 upper bound](https://github.com/tqec/tqecd/pull/71) (`Closes #70`) — one-line metadata change: `numpy>=1.22,<2.3` → `numpy>=1.22`.
- **Verified:** unlike the tqec side, `tqecd` needed **zero source edits** — `mypy src/tqecd` is clean and all **123 tests** pass at numpy 2.3.5 / 2.4.6 / 2.5.1 (Python ≥ 3.11; Python 3.10 keeps numpy 2.2.x automatically). `pre-commit run --all-files` green.
- **Scope note:** this is a *second* contribution in a different repo. #613's Phase IV milestone was already met independently; the `tqecd` PR is what finally lets numpy ≥ 2.3 resolve end-to-end (merge + release dates in [Blocker Cleared](#blocker-cleared--un-block-2026-07-05) below).

### Blocker Cleared — Un-block (2026-07-05)

The cross-repo blocker closed out in a single day, all on the maintainer's side, then finalized on the tqec PR:

- **12:42 UTC** — @nelimee merged [tqecd#71](https://github.com/tqec/tqecd/pull/71) and closed [tqecd#70](https://github.com/tqec/tqecd/issues/70).
- **12:59 UTC** — @nelimee released **tqecd 0.2.1** to PyPI (drops the `numpy<2.3` cap).
- **~14:02 UTC** — @nelimee merged `main` *into* the PR branch (via "Allow edits from maintainers"), bringing #977 up to date — commits preserved, not rebased.
- **Un-block commit** — on top of that merge, [`2b82c11b5`](https://github.com/sinhasagar507/tqec/commit/2b82c11b5) bumps `tqecd>=0.2.1` and relocks, re-verified green on the merged base (see [Testing Strategy](#testing-strategy)). Proved the fix directly — `tqecd==0.2.0` makes `numpy>=2.3` unsatisfiable, `tqecd>=0.2.1` resolves it.
- **#977 un-drafted** and a ready-for-review note posted to @nelimee.

### Merged (2026-07-06)

- @nelimee's one review suggestion — remove the now-redundant inline comment on the numpy constraint — was applied via GitHub's "Commit suggestion" (commit `Update pyproject.toml`), which auto-dismissed his approval (branch protection dismisses stale approvals on a new push).
- The lone failing check was **"Check for broken links"** (`make linkcheck` in `docs/`, a flaky external-URL validator), **unrelated** to the pyproject/`merge.py` diff. @nelimee fixed it himself in [#998](https://github.com/tqec/tqec/pull/998) (`fix: ignore stargazer list`) and merged #977 once CI passed.
- **#977 squash-merged to `main` as [`4aed39c99`](https://github.com/tqec/tqec/commit/4aed39c99)**, all 11 checks green. Local + fork clones resynced to merged `main`; the merged feature branches were deleted on both `tqec` and `tqecd`.

---

## Learnings & Reflections

- **Verify the task before fixing it.** The issue said "mypy" and listed seven files; the repo had since migrated to Astral's `ty`, and those errors had already resolved once numpy 2.3 stabilized. Reproducing deterministically (a `uv` override scoped to Python ≥ 3.11, because numpy ≥ 2.3 dropped 3.10) surfaced the *actual* current error — numpy 2.5 / `merge.py` — that the issue never listed. The hardest part was distinguishing "already fixed" from "not yet reproduced."
- **Some blockers aren't in the repo you're editing.** The cleanest local fix still couldn't fully land because a *sibling* package (`tqecd`) transitively capped numpy. Recognizing this early and raising it with the maintainer — rather than forcing a fragile uv-only override into CI — kept the PR honest and got the maintainer to own the dependency bump.
- **Communicating the blocker clearly was as valuable as the code.** A concise, evidence-backed comment on #613 turned a stuck PR into an active collaboration: the maintainer agreed to bump `tqecd` within hours.
- **A polite follow-up can hand you the next contribution.** A one-week nudge on #613 turned the maintainer's "I'll do it" into "this could be a good first contribution to `tqecd` from you too" — a second, invited PR in a sibling repo. Respecting the target repo's *own* rules mattered: `tqecd` has a separate issue-first contributor guide and uses `mypy` (not `ty`) with `pre-commit` hooks, none of which could be assumed from the `tqec` clone.
- **Merging isn't shipping.** `tqecd`'s PyPI publish is tag-gated, so even a merged cap-lift doesn't unblock downstream until the maintainer cuts a release — a step outside my control, documented honestly in the PR rather than glossed over. In this case the wait was short: the maintainer merged, released 0.2.1, and synced `main` into my PR branch all within ~90 minutes.
- **Reconcile, don't force-push.** When I went to push the un-block commit, the maintainer had already pushed a `main`-merge to my fork branch (via "Allow edits from maintainers"), so my push was correctly rejected as non-fast-forward. The right move was to fetch, reset onto his merge, re-apply the one-line bump, regenerate the lock, and re-verify on the merged base — never `--force`. A large-looking `uv.lock` diff turned out to be a genuine, `uv lock --locked`-clean consequence of the dependency change, not noise to hand-trim away.
- **A red check isn't always your bug — read *which* check.** Applying the maintainer's review suggestion auto-dismissed his approval (branch protection dismisses stale approvals on any push — expected, not a mistake), and the re-run then showed one red check. Reading it carefully mattered: it was the docs `linkcheck` job (`make linkcheck`, which hits live external URLs and is inherently flaky), not any test or type-check touched by the diff. Rather than "fixing" something I didn't break, I flagged it as unrelated — and the maintainer confirmed, patched it himself in #998, and merged. Distinguishing a CI red herring from a real regression saved a pointless scramble.

---

## Resources Used

- Issue: [tqec/tqec #613](https://github.com/tqec/tqec/issues/613)
- Stale upstream PR (reference only): [tqec/tqec #659](https://github.com/tqec/tqec/pull/659)
- tqec PR (merged): [tqec/tqec #977](https://github.com/tqec/tqec/pull/977)
- tqecd package: [tqec/tqecd](https://github.com/tqec/tqecd)
- tqecd cap-lift issue: [tqec/tqecd #70](https://github.com/tqec/tqecd/issues/70)
- tqecd cap-lift PR: [tqec/tqecd #71](https://github.com/tqec/tqecd/pull/71)
- tqec `pyproject.toml`: line 44 (numpy cap), line 59 (tqecd dep)
- tqec contribution guide: [`CONTRIBUTING.md`](https://github.com/tqec/tqec/blob/main/CONTRIBUTING.md)
- tqecd contributor guide (issue-first; `mypy` + `pre-commit`): [contributor_guide](https://tqec.github.io/tqecd/contributor_guide.html)

---
---

# Contribution 2: Surface non-convergence in `SVC` with `max_iter=-1` (scikit-learn)

**Contribution Number:** 2
**Student:** Sagar Sinha
**Issue:** [scikit-learn/scikit-learn #13557 — "`cross_validate` hang randomly when training svc with polynomial kernel"](https://github.com/scikit-learn/scikit-learn/issues/13557)
**Status:** 🟡 **Phase I — claimed, awaiting maintainer scoping reply.** Claim comment posted 2026-06-18, follow-up posted 2026-07-27; no maintainer response to either. Issue remains open and unassigned with no linked PR. **Proceed/withdraw gate: 2026-08-03** — see [Next Steps](#next-steps).

---

## Why I Chose This Issue

[scikit-learn](https://github.com/scikit-learn/scikit-learn) is the reference machine-learning library for Python. Issue #13557 reports that `cross_validate` appears to **hang indefinitely** when fitting an `SVC` with a polynomial kernel at extreme hyper-parameters (`degree=7–8`, `gamma≈900–4300`). The underlying cause is that libsvm's SMO solver fails to converge, and with scikit-learn's default `max_iter=-1` (unlimited iterations) `fit()` simply spins with no output — the user cannot tell a slow fit from a stuck one.

Reasons for choosing it:

1. **Pure Python / CPU.** No GPU, no compiled-extension changes needed for the likely fix (a warning or documentation improvement) — the same non-domain-specialist profile that made tqec #613 workable.
2. **Maintainer-reproduced with an exact trigger.** Core maintainer @NicolasHug confirmed it on master and pinned the trigger (`random_state=1` in `train_test_split`, oversized `gamma`), and the reporter linked the related [#4648](https://github.com/scikit-learn/scikit-learn/issues/4648).
3. **Gold-standard contributor infrastructure.** scikit-learn has an OS-specific `CONTRIBUTING.md`, a code of conduct, and an extremely active review pipeline — the governance gaps I had to work around in tqec/tqecd do not exist here.
4. **Labeled `help wanted`, unassigned, no linked PR.** No courtesy conflict with another contributor.

**Checklist score: 5 / 6** (see `SHORTLIST.md` — the one weak check is #4 "active & claimable": the issue itself has had no maintainer engagement since 2019, even though the repo is highly active).

---

## Understanding the Issue

### Problem Description

`SVC(kernel='poly', degree=7, gamma=4178, C=0.66)` fed into `cross_validate` never returns. The reporter's `gdb` backtrace shows execution parked inside the libsvm solver, i.e. the SMO optimizer is not converging rather than the code deadlocking.

### Expected Behavior

The user gets either results or an actionable signal — a warning that the solver did not converge and that the features should be scaled — instead of a silent, indefinite spin.

### Current Behavior (verified on scikit-learn 1.9.0, 2026-06-18)

The original behavior is **largely mitigated already**:

- With a **finite `max_iter`**, scikit-learn now emits `ConvergenceWarning: Solver terminated early (max_iter=N). Consider pre-processing your data with StandardScaler or MinMaxScaler.`
- With the **default `max_iter=-1`** and the reporter's extreme parameters, the fit *converged* in my runs (n_iter ≈ 70k) rather than hanging indefinitely.

### Remaining Gap (the actual candidate scope)

There is **no signal at all on the `max_iter=-1` path**. A user who never sets `max_iter` — the default — still gets a fit that can run for tens of thousands of iterations with no indication that the parameters are pathological. Candidate fixes, in order of decreasing invasiveness:

1. Emit a `ConvergenceWarning`-style hint when `max_iter=-1` and the solver blows past an iteration threshold.
2. Document the failure mode (unscaled features + large `gamma`/`degree` → effectively unbounded runtime) in the SVM user guide / `SVC` docstring, pointing to `StandardScaler`.

### ⚠️ Primary risk: "already fixed upstream"

This is the same risk that ended the earlier medusa #14957 attempt and that nearly sank tqec #613. The issue is 7 years old and the reported symptom no longer reproduces as written. **This contribution proceeds only if a maintainer confirms option 1 or 2 is still wanted.** If the answer is "nothing left to do," the issue should be closed as resolved and I move to the next shortlisted candidate.

---

## Status Log

| Date | Event |
|---|---|
| 2026-06-18 | Reproduced on scikit-learn 1.9.0; posted a claim comment on #13557 documenting that the hang no longer reproduces and asking whether the `max_iter=-1` guidance gap is still worth fixing. |
| 2026-07-21 | No maintainer reply (issue last updated = my own comment). Issue still open, unassigned, no linked PR. Next step: a polite one-line follow-up ping, and if still silent, raise the scoping question on the scikit-learn Discord / mailing list per `CONTRIBUTING.md`. |
| 2026-07-27 | Follow-up posted on #13557 (18:40 UTC). Rather than a bare ping it adds the measured `n_iter ≈ 70_000` for the reporter's exact setup and narrows the ask to a binary question: is a **docs-only** PR welcome (SVM user guide + `SVC` docstring note pointing at `StandardScaler` and a finite `max_iter`), or is the behavior considered adequately covered today — in which case the issue can be closed as resolved. Option 1 (a warning past an iteration threshold) was deliberately left out of the ask, since choosing the threshold is the contentious part and is a maintainer's call. Still unassigned; no reply at time of writing. |

---

## Next Steps

1. **Escalate the scoping question.** The follow-up of 2026-07-27 is now on the issue; the thread has no active maintainer subscriber, so if it goes unanswered the question moves to the scikit-learn Discord / mailing list per `CONTRIBUTING.md`. The tqec #613 lesson applies directly — a polite, evidence-backed nudge is what converted that issue from stuck to active.
2. **Draft the docs-only change in parallel, don't wait.** The docs note is the subset common to both acceptable outcomes and needs no maintainer scoping to be defensible, so it can be written while the question is open. Nothing in `sklearn/svm/` gets touched — i.e. no warning-threshold work — until a maintainer confirms option 1 is wanted.
3. **Proceed/withdraw gate — 2026-08-03.** If there is still no reply by then (six weeks after the original claim), treat the silence as an answer: withdraw from #13557 and switch to a fallback rather than let the contribution drift a second month. This is the "already fixed upstream" risk above, made time-bounded.
4. **Fallback candidates** if #13557 is declined, closed as fixed, or hits the gate — re-verified 2026-07-27:
   - [home-assistant/core #143041](https://github.com/home-assistant/core/issues/143041) — migrate `ubus` `DeviceScanner` → `ScannerEntity` (6/6, `SHEET_RESULTS.md`). Open, unassigned, untouched since 2025-04-15.
   - [zarr-developers/zarr-python #1328](https://github.com/zarr-developers/zarr-python/issues/1328) — additional `fsspec` tests (4/6, `SHEET_RESULTS.md` Backup 1). Open, unassigned. Scope is fuzzy by design — it needs a maintainer reply pinning the exact tests before it is claimable, so it is a slower fallback than #143041.
   - ~~[papra-hq/papra #691](https://github.com/papra-hq/papra/issues/691) — add Docker `HEALTHCHECK`~~ — **no longer available**: claimed by another contributor on 2026-07-26 ("taking a crack at this"). Dropped from the list.

---

## Resources Used — Contribution 2

- Issue: [scikit-learn/scikit-learn #13557](https://github.com/scikit-learn/scikit-learn/issues/13557)
- Related issue (same non-convergence family): [scikit-learn #4648](https://github.com/scikit-learn/scikit-learn/issues/4648)
- Reporter's libsvm backtrace: [gist](https://gist.github.com/sam-yusuke/7099829b1797d6867eb928264597979d)
- scikit-learn contribution guide: [CONTRIBUTING.md](https://github.com/scikit-learn/scikit-learn/blob/main/CONTRIBUTING.md)
- Candidate evaluation: [`SHORTLIST.md`](SHORTLIST.md) (Candidate 1, 5/6)
