# Contribution 1: Add more custom types to Outlines

**Contribution Number:** 1  
**Student:** Sagar Sinha  
**Issue:** [dottxt-ai/outlines #1303 — "Add more custom types"](https://github.com/dottxt-ai/outlines/issues/1303)  
**Status:** Phase I — Complete

---

## Why I Chose This Issue

I chose [outlines #1303 "Add more custom types"](https://github.com/dottxt-ai/outlines/issues/1303) in [dottxt-ai/Outlines](https://github.com/dottxt-ai/outlines), a widely-used Python library for *structured generation* — constraining an LLM's output to match a required format. "Custom types" are reusable, regex-backed types (such as `email`, `isbn`, `ipv4`, and `semver`) that guarantee a model produces a valid value. The maintainer (rlouf, the project's creator) wants to expand this collection to cover more common patterns like phone numbers and postal codes.

I'm interested in this for four reasons:

1. **Skill match.** The work is core Python plus regular expressions and unit tests — areas I'm comfortable in — so I can be productive immediately instead of spending the cycle ramping up on an unfamiliar stack.
2. **AI/LLM relevance.** Structured generation is foundational to modern LLM applications (reliable tool-calling, data extraction, RAG pipelines). Contributing here teaches me how a real library enforces output constraints, which is exactly the area I want to grow in.
3. **A clear, bounded definition of "done."** A finished contribution is a new custom type — regex + type definition + edge-case tests + docs — that mirrors the recently merged [PR #1863](https://github.com/dottxt-ai/outlines/pull/1863) (which added `semver` and `mac_address`). I have a concrete template to follow and a clear acceptance bar.
4. **Active and genuinely claimable.** The issue is open, unassigned, labeled `help wanted`, and *additive* by design — different contributors add different types — so I can claim specific ones without racing an in-flight pull request. Maintainers merged a related PR as recently as May 2026.

**What I plan to build:** I reviewed the library's current types and will add three that are currently missing — `hex_color` (e.g. `#1a2b3c`), `slug` (e.g. `my-post-title`), and `credit_card` (common card-number formats) — each with parametrized tests and a docs entry, following the merged [PR #1863](https://github.com/dottxt-ai/outlines/pull/1863) (`semver`/`mac_address`) as a template. (I first considered `ipv6`, but found it was already being added in open PR #1867, so I deferred to that work and chose `credit_card` instead — a good reminder to check existing issues *and* PRs for each specific type before claiming.) I've commented on the issue introducing myself and claiming these specific types so my work doesn't overlap with other contributors.

---

## Understanding the Issue

### Problem Description

[In your own words, what's broken or missing?]

### Expected Behavior

[What should happen?]

### Current Behavior

[What actually happens?]

### Affected Components

[Which parts of the codebase are involved?]

---

## Reproduction Process

### Environment Setup

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. [Step 1]
2. [Step 2]
3. [Observed result]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
