# Contribution 1: Fix numpy>=2.3 type-check failures in tqec

**Contribution Number:** 1
**Student:** Sagar Sinha
**Issue:** [tqec/tqec #613 — "Fix mypy failures for numpy>=2.3"](https://github.com/tqec/tqec/issues/613)
**Status:** Phase IV — Complete (PR submitted & under maintainer review)

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

- [x] `uv run ty check` — **clean at numpy 2.3.5, 2.4.6, and 2.5.0** (2.5.0 was the only failing version; clean after the `merge.py` fix), and in the committed state (numpy 2.2.6).

### Unit / Integration Tests

- [x] `uv run pytest $(git ls-files '*_test.py')` — **788 passed** at numpy 2.5.0 (`merge_test.py`: 13 passed). No runtime behavior change.
- [x] **Line coverage 92%** overall (`merge.py` 93%; the changed line is exercised) — above the project's 80% bar.

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
- **Draft PR:** [tqec/tqec#977](https://github.com/tqec/tqec/pull/977)

### Challenges Faced

- **The literal task had shifted.** The issue says "mypy," but the repo type-checks with `ty`, and the originally-listed `signedinteger` errors had already resolved once numpy 2.3 stabilized. Confirming this required forcing numpy ≥ 2.3 deterministically (a `uv` override scoped to Python ≥ 3.11, since numpy ≥ 2.3 dropped 3.10) and testing each minor version — which surfaced the real, current error (numpy 2.5 / `merge.py`) the issue never listed.
- **The real blocker is cross-repo.** Lifting tqec's own cap doesn't resolve numpy ≥ 2.3 because `tqecd 0.2.0` transitively pins `numpy<2.3`. That makes the PR a draft pending a tqecd release (or a temporary `uv` override) — raised with the maintainer on #613.

---

## Pull Request

**PR Link:** [tqec/tqec#977 — Lift numpy<2.3 cap and fix numpy 2.5 ty error](https://github.com/tqec/tqec/pull/977) (supersedes the stale #659).

**PR Description (summary):** Couples two changes to support `numpy>=2.3`: (a) a one-line type fix in `compile/blocks/layers/merge.py` (`numpy.lcm.reduce(...)` → `math.lcm(*...)`, the only `ty` error surfaced by numpy 2.5), and (b) lifting tqec's `numpy<2.3` upper bound in `pyproject.toml` + relocking. `ty` clean at numpy 2.3.5 / 2.4.6 / 2.5.0; 788 tests pass; 92% coverage. Closes #613.

**Status:** **Phase IV Complete — PR submitted; iterating with maintainer.** Open as an honest **draft** pending one external, maintainer-owned step (see below). Per the program, a review-ready submitted PR is the Phase IV milestone; the work is complete and green on the tqec side, with the remaining blocker outside this repo.

### Maintainer Feedback

The feedback loop on [issue #613](https://github.com/tqec/tqec/issues/613) (maintainer **@nelimee**, cc **@purva-thakre**):

- **Jun 19** — @nelimee confirmed PR #659 was stalled (waiting on a `stim` release that has since shipped) and invited a fresh PR; I was assigned to the issue.
- **Jun 23** — I posted the full investigation: the original `signedinteger` errors are gone under `ty` (clean at numpy 2.3.5 / 2.4.6), numpy 2.5 surfaces exactly one new error in `merge.py` (fixed), and the real blocker is cross-repo — `tqecd 0.2.0` transitively pins `numpy<2.3`, so lifting tqec's own cap alone still resolves numpy to 2.2.x.
- **Jun 23** — @nelimee replied: *"I'll try to bump `tqecd`, good catch. Keeping you updated."* (path A — an uncapped `tqecd` release).
- **Jun 23** — I confirmed the plan: once the new `tqecd` is on PyPI I'll bump `tqecd>=0.2.x`, relock, and flip #977 to ready-for-review.
- **Jul 1** — polite follow-up (LT4 etiquette, ~1 week after his last reply) confirming the `tqecd` bump was still on his radar.
- **Jul 3** — @nelimee replied: *"It is, but I won't have the time to do it in the next few days. Seems like that could be a good first contribution to `tqecd` from you too :) Sorry for flagging that as 'I'll do it' and blocking you."* — **the path flipped from A to B: the maintainer invited me to open the `tqecd` cap-lift myself.**
- **Jul 4** — acknowledged on #613, then opened the parallel `tqecd` contribution (see below).

### Parallel Contribution — `tqecd` cap-lift (the actual unblocker)

The cross-repo blocker is fixed where it actually lives — in `tqecd`, at @nelimee's invitation:

- **Issue:** [tqec/tqecd#70 — Lift the numpy<2.3 upper bound](https://github.com/tqec/tqecd/issues/70) (`enhancement`; follows tqecd's issue-first contributor guide).
- **PR:** [tqec/tqecd#71 — Lift the numpy<2.3 upper bound](https://github.com/tqec/tqecd/pull/71) (`Closes #70`). One-line metadata change: `numpy>=1.22,<2.3` → `numpy>=1.22`.
- **Verified:** unlike the tqec side, `tqecd` needed **zero source edits** — `mypy src/tqecd` is clean and all **123 tests** pass at numpy 2.3.5 / 2.4.6 / 2.5.1 (Python ≥ 3.11; Python 3.10 keeps numpy 2.2.x automatically). `pre-commit run --all-files` green.
- **Scope note:** this is a *second, maintainer-invited* contribution in a different repo. #613's Phase IV milestone was already met independently; the `tqecd` PR is what will finally let numpy ≥ 2.3 resolve end-to-end.

**Next step:** once tqecd#71 merges **and** @nelimee cuts a `tqecd` release (tag-gated PyPI publish — a maintainer-owned step; merging alone doesn't publish), I'll bump `tqecd>=<new>` in tqec's `pyproject.toml`, relock so numpy ≥ 2.3 actually resolves, re-run `ty`/tests, and un-draft #977.

---

## Learnings & Reflections

- **Verify the task before fixing it.** The issue said "mypy" and listed seven files; the repo had since migrated to Astral's `ty`, and those errors had already resolved once numpy 2.3 stabilized. Reproducing deterministically (a `uv` override scoped to Python ≥ 3.11, because numpy ≥ 2.3 dropped 3.10) surfaced the *actual* current error — numpy 2.5 / `merge.py` — that the issue never listed. The hardest part was distinguishing "already fixed" from "not yet reproduced."
- **Some blockers aren't in the repo you're editing.** The cleanest local fix still couldn't fully land because a *sibling* package (`tqecd`) transitively capped numpy. Recognizing this early and raising it with the maintainer — rather than forcing a fragile uv-only override into CI — kept the PR honest and got the maintainer to own the dependency bump.
- **Communicating the blocker clearly was as valuable as the code.** A concise, evidence-backed comment on #613 turned a stuck PR into an active collaboration: the maintainer agreed to bump `tqecd` within hours.
- **A polite follow-up can hand you the next contribution.** A one-week nudge on #613 turned the maintainer's "I'll do it" into "this could be a good first contribution to `tqecd` from you too" — a second, invited PR in a sibling repo. Respecting the target repo's *own* rules mattered: `tqecd` has a separate issue-first contributor guide and uses `mypy` (not `ty`) with `pre-commit` hooks, none of which could be assumed from the `tqec` clone.
- **Merging isn't shipping.** `tqecd`'s PyPI publish is tag-gated, so even a merged cap-lift doesn't unblock downstream until the maintainer cuts a release — a step outside my control, documented honestly in the PR rather than glossed over.

---

## Resources Used

- Issue: [tqec/tqec #613](https://github.com/tqec/tqec/issues/613)
- Stale upstream PR (reference only): [tqec/tqec #659](https://github.com/tqec/tqec/pull/659)
- tqecd package: [tqec/tqecd](https://github.com/tqec/tqecd)
- tqec `pyproject.toml`: line 44 (numpy cap), line 59 (tqecd dep)
- tqec contribution guide: `CONTRIBUTING.md`
