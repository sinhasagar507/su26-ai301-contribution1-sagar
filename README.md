# Contribution 1: Fix inconsistent URL encoding in @medusajs/file-s3

**Contribution Number:** 1
**Student:** Sagar Sinha
**Issue:** [medusajs/medusa #14957 — "[Bug]: Inconsistent URL encoding in @medusajs/file-s3"](https://github.com/medusajs/medusa/issues/14957)
**Status:** Phase II — Complete

---

## Why I Chose This Issue

I chose [medusajs/medusa #14957](https://github.com/medusajs/medusa/issues/14957) in [medusajs/medusa](https://github.com/medusajs/medusa), a widely-used open-source commerce platform. The issue is a well-scoped bug in the S3 file provider (`@medusajs/file-s3`): public file URLs are built inconsistently across upload methods, which can produce broken links to product images and uploaded assets.

I chose it for three reasons:

1. **Tightly scoped.** The bug lives in a single small TypeScript file and is about correct URL string handling — easy to understand and reason about without learning the whole platform.
2. **Clear definition of done.** The issue states the expected vs. actual behavior precisely, so I have an unambiguous target.
3. **Labeled and claimable.** It is open, unassigned, and labeled `help-wanted` / `type: bug` / `version: 2.0`.

**Issue status note (important and verified):** While reproducing, I confirmed that the underlying bug has **already been fixed upstream** on the `develop` branch by [PR #15109](https://github.com/medusajs/medusa/pull/15109) ("fix(file-s3): encode URL path segments individually to preserve prefix slashes"), merged 2026-05-03. The issue is still shown as "open" only because that PR did not use GitHub's `Fixes #14957` auto-close keyword. I reproduced the bug against the affected v2.5.1 behavior and commented on the issue noting it appears resolved on `develop`. Per the Phase II guidance for an already-fixed issue, this write-up documents a faithful reproduction and a solution plan rather than a new pull request.

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
