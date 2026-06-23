# Contribution 1: Fix numpy>=2.3 type-check failures in tqec

**Contribution Number:** 1
**Student:** Sagar Sinha
**Issue:** [tqec/tqec #613 — "Fix mypy failures for numpy>=2.3"](https://github.com/tqec/tqec/issues/613)
**Status:** Phase II — Complete · Phase III — In Progress (Build)

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

### Static Type Check

- [x] `uv run ty check` exits 0 with numpy 2.4.6 installed (confirmed locally)

### Unit / Integration Tests

- [x] Full test suite (`uv run pytest`) passes — 788 tests with numpy 2.4.6

### Manual Testing

Ran `uv run ty check` and `uv run pytest --tb=short -q` after lifting the numpy cap locally — both pass. Results captured under Reproduction Evidence above.

---

## Implementation Notes

### Code Changes

- **Files to modify:** `pyproject.toml` (lift numpy cap, bump tqecd lower bound), `uv.lock` (relock)
- **Working branch:** `fix/613-numpy-2.3-type-check` in the tqec clone at `/Applications/saggydev/projects_learning/su26-ai301-contribution1-sagar/tqec`
- **Blocker:** Waiting on `tqecd` to publish a release without the `numpy<2.3` cap. Raised with the maintainer.

---

## Pull Request

Not yet opened — blocked on `tqecd` publishing an uncapped release. The PR will be opened on branch `fix/613-numpy-2.3-type-check` once the blocker is resolved.

---

## Resources Used

- Issue: [tqec/tqec #613](https://github.com/tqec/tqec/issues/613)
- Stale upstream PR (reference only): [tqec/tqec #659](https://github.com/tqec/tqec/pull/659)
- tqecd package: [tqec/tqecd](https://github.com/tqec/tqecd)
- tqec `pyproject.toml`: line 44 (numpy cap), line 59 (tqecd dep)
- tqec contribution guide: `CONTRIBUTING.md`
