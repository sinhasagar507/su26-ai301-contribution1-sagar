# Contribution 1: Fix numpy>=2.3 type-check failures in tqec

**Contribution Number:** 1
**Student:** Sagar Sinha
**Issue:** [tqec/tqec #613 — "Fix mypy failures for numpy>=2.3"](https://github.com/tqec/tqec/issues/613)
**Status:** Phase I — Complete

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

The S3 file provider (`@medusajs/file-s3`, v2.5.1) builds the public file URL as `${fileUrl}/${key}`, but the two upload paths encode the key inconsistently, so the resulting URLs are malformed in different ways.

### Expected Behavior

The file key should be encoded **per path segment** — each segment URL-encoded, with the `/` separators preserved — so a key like `uploads/products/my product-01JRXYZ.jpg` yields:

```
https://cdn.example.com/uploads/products/my%20product-01JRXYZ.jpg
```

### Current Behavior (v2.5.1)

- **`upload()`** runs `encodeURIComponent()` on the **entire** key, so the `/` separators become `%2F` and the path structure breaks:
  ```
  https://cdn.example.com/uploads%2Fproducts%2Fmy%20product-01JRXYZ.jpg
  ```
- **`getUploadStream()`** applies **no** encoding, so spaces and special characters are left raw, producing an invalid URL:
  ```
  https://cdn.example.com/uploads/products/my product-01JRXYZ.jpg
  ```

Malformed URLs stored as the canonical file URL cause 404s or CDN resolution failures.

### Affected Components

- `packages/modules/providers/file-s3/src/services/s3-file.ts` — the `upload()` and `getUploadStream()` methods (and the shared URL-building logic).

---

## Reproduction Process

### Environment Setup

The bug is pure URL-string logic, so a full Medusa monorepo build is not required to reproduce it.

- `gh` CLI was not installed locally, so the fork was created via the GitHub website.
- Cloned the fork as a subfolder and worked from a feature branch.
- Reproduction needs only Node.js (verified with `node v22.22.0`) — no S3 credentials, no `yarn build`.

Working branch: `https://github.com/sinhasagar507/medusa/tree/fix-issue-14957`

### Steps to Reproduce

1. Clone Medusa and check out the affected version's behavior (v2.5.1). The relevant file is `packages/modules/providers/file-s3/src/services/s3-file.ts`.
2. Observe that in v2.5.1 `upload()` builds the URL with `encodeURIComponent(fileKey)` applied to the whole key, while `getUploadStream()` applies no encoding.
3. Run the self-contained reproduction script that mirrors that exact logic:
   ```
   node repro-issue-14957.js
   ```
4. **Expected:** all upload paths produce `.../uploads/products/my%20product-...jpg` (slashes preserved, space encoded).
5. **Actual (v2.5.1):**
   ```
   key              : uploads/products/my product-01JRXYZ.jpg
   upload()         : https://cdn.example.com/uploads%2Fproducts%2Fmy%20product-01JRXYZ.jpg   <-- separators broken (%2F)
   getUploadStream(): https://cdn.example.com/uploads/products/my product-01JRXYZ.jpg   <-- raw space, invalid URL
   expected (fixed) : https://cdn.example.com/uploads/products/my%20product-01JRXYZ.jpg   <-- correct
   ```

Running the script twice produces identical output, confirming the behavior is deterministic.

### Reproduction Evidence

- **Branch:** `https://github.com/sinhasagar507/medusa/tree/fix-issue-14957`
- **Commit showing reproduction:** the `repro-issue-14957.js` commit on that branch (`Reproduce issue #14957: inconsistent URL encoding in @medusajs/file-s3`).
- **My findings:** The bug reproduces exactly as described on the v2.5.1 logic. On the current `develop` branch, both methods already encode per segment — `fileKey.split("/").map(encodeURIComponent).join("/")` (see `s3-file.ts` around lines 164–171 and 208–215) — and a dedicated test `s3-file-url-encoding.spec.ts` already exists. The fix landed in PR #15109. I commented on the issue reporting it appears resolved on `develop`.

---

## Solution Approach

### Analysis

Root cause: `encodeURIComponent()` is the wrong granularity for an entire path. It percent-encodes `/`, which is correct for a single segment but destroys path structure when applied to the whole key. `getUploadStream()` had the opposite problem — no encoding at all. The two paths should share one correct encoding strategy.

### Proposed Solution

Encode each path segment individually and rejoin with `/`, used by every method that builds the public URL — so `/` separators are preserved while segment contents (spaces, `&`, etc.) are encoded.

### Implementation Plan

Using the UMPIRE framework (adapted):

**Understand:** URLs are malformed because `upload()` over-encodes (whole key, `/` → `%2F`) and `getUploadStream()` under-encodes (no encoding). Both should produce per-segment-encoded URLs.

**Match:** The correct pattern is `key.split("/").map(encodeURIComponent).join("/")` — exactly what upstream PR #15109 adopted and what the reproduction script confirms produces the expected URL.

**Plan:**
1. Add a private helper on the service (e.g. `encodeKey(key: string)`) that returns `key.split("/").map(encodeURIComponent).join("/")`.
2. Use the helper in both `upload()` and `getUploadStream()` when constructing `${this.config_.fileUrl}/${encodedKey}`, removing the inconsistent inline encoding.
3. Add unit tests under `src/services/__tests__/` asserting that the resulting URL keeps `/` separators (no `%2F`) and encodes special characters (spaces, `&`) within a segment, for both methods.

**Implement:** Branch `https://github.com/sinhasagar507/medusa/tree/fix-issue-14957` (reproduction committed; the fix itself is already present upstream in PR #15109, so this is documented rather than re-submitted).

**Review:** Follow Medusa's contribution conventions — branch prefix `fix/`, base branch `develop`, PR description structured as What / Why / How / Testing, small isolated commits, squash-merge. Tests live alongside the service in `__tests__/`.

**Evaluate:** The reproduction script prints the correct per-segment-encoded URL after the fix; unit tests modeled on the upstream `s3-file-url-encoding.spec.ts` pass.

---

## Testing Strategy

### Unit Tests

- [ ] `upload()` with a prefix containing `/` keeps the separators (no `%2F`) and encodes the filename's spaces.
- [ ] `getUploadStream()` with the same input produces the same correctly-encoded URL.
- [ ] A segment containing special characters (e.g. `&`) is encoded within the segment while `/` separators are preserved.

### Integration Tests

- [ ] Not required for this fix — the behavior is deterministic string logic covered by unit tests.

### Manual Testing

Ran `node repro-issue-14957.js` (twice) to confirm the v2.5.1 malformed outputs and the corrected output; results captured under Steps to Reproduce.

---

## Implementation Notes

### Code Changes

- **Files involved:** `packages/modules/providers/file-s3/src/services/s3-file.ts` (and a test under `src/services/__tests__/`).
- **Reproduction artifact:** `repro-issue-14957.js` on branch `fix-issue-14957`.
- **Upstream resolution:** PR #15109 (merged 2026-05-03).

---

## Pull Request

No pull request: the underlying fix is already merged upstream (PR #15109). I commented on the issue to report it appears resolved on `develop`.

---

## Resources Used

- Issue: https://github.com/medusajs/medusa/issues/14957
- Upstream fix: https://github.com/medusajs/medusa/pull/15109
- Affected source: `packages/modules/providers/file-s3/src/services/s3-file.ts`
- Medusa contribution guide (CONTRIBUTING.md)
