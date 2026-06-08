# Contribution #1: Model loading fails on Windows when path contains non-ASCII (Cyrillic) characters

**Contribution Number:** 1
**Student:** Preetham
**Issue:** https://github.com/ggml-org/llama.cpp/issues/18571
**Status:** Phase I (Issue Selection) — Complete · Phase II (Reproduce & Plan) — Next

---

## Why I Chose This Issue

I'm moving from a hardware design-verification / RTL background into AI/ML systems engineering, and `llama.cpp` is close to the ideal bridge between those two worlds: it's the de facto local LLM inference engine, written in C/C++, where a lot of the work is exactly the low-level, cross-platform, correctness-and-performance reasoning I already do. This particular issue is a Windows file-I/O and character-encoding bug — `fopen` can't open model paths that contain non-ASCII characters because it goes through the system ANSI codepage instead of UTF-8/UTF-16. Fixing it means understanding narrow-vs-wide Windows file APIs (`_wfopen` / `CreateFileW`), UTF-8 → UTF-16 conversion, and how to make a change cleanly platform-conditional. That's systems work I can reason about confidently, while still being inside the AI inference stack I'm trying to learn.

It's also well-shaped for the course. It's a genuine, reported, reproducible bug (not a feature or RFC), so Phase II has something concrete to reproduce. A maintainer (CISC) is already engaged in the thread and has pointed at the fix direction, which is the kind of responsiveness Phase IV depends on. And there's a distinctive angle here: three earlier PRs (#16611, #16609, #16589) attempted this fix and stalled, and the maintainer explicitly asked for someone to revive/consolidate them. Adopting that work — reproducing the bug, diagnosing *why* those PRs stalled, and consolidating the viable approach into one clean, reviewable PR with attribution — is a stronger engineering story than a greenfield fix, and it's the kind of "make a stuck thing land" contribution that's genuinely useful to the project.

**Selection diligence (verified during Phase I, as of 2026-06-08):**
- Issue is **open**, opened 2026-01-03; labels: `bug`, `help wanted`.
- **No assignee** and **no linked PR** — technically claimable.
- One contributor (AvinashDwivedi) proposed the fix approach and offered to implement, and the reporter endorsed it, but no PR has been submitted; the comment "@CISC please respond" indicates they're waiting on a maintainer nod rather than holding an active claim.
- Maintainer steer: revive the three stalled PRs (#16611, #16609, #16589) by pinging the original authors.
- **Open question to resolve before/at claim:** read those three PRs to determine whether they are *work-stalled* (authors went quiet → revivable) or *design-disagreement-stalled* (maintainers split on approach → higher risk).

**Honest risk note (part of choosing thoughtfully):**
- `llama.cpp` is a very high-traffic repo; review can be fast and terse, and good-first bugs get picked over quickly. Mitigation: claim promptly, keep the PR tightly scoped and the description crisp.
- The bug is **Windows-only**, so reproduction requires a Windows environment (or VM).
- **Fallback identified:** a fresh, unclaimed `pytorch/ignite` bug (lower variance, more responsive newcomer review), pending confirmation from the instructor on whether an off-list issue is allowed.

---

## Understanding the Issue

### Problem Description

On Windows, `llama.cpp` opens GGUF model files with `fopen`. `fopen` takes a narrow (`char*`) path that Windows interprets in the system ANSI codepage, so any path containing characters outside that codepage — Cyrillic, Chinese, Korean, etc. — fails to open. The result is a model-load failure for any non-English user whose model path includes such characters. The reporter saw:

```
gguf_init_from_file: failed to open GGUF file 'D:\Загрузки\models\llm\olmOCR-2-7B-1025-Q4_K_M.gguf'
```

### Expected Behavior

A GGUF model located at a path containing non-ASCII characters should load successfully on Windows, the same as it does on Linux/macOS. Unicode file paths should be fully supported.

### Current Behavior

Model loading fails with "failed to open GGUF file" whenever the path contains non-ASCII characters. ASCII-only paths work; the failure is specific to the narrow-`fopen` path-encoding limitation on Windows. Reported on build `b7539`, Windows, Vulkan backend (the bug is backend- and hardware-independent), with models `olmOCR-2-7B-1025-Q4_K_M` and `Qwen2.5-VL-7B-Instruct-Q4_K_M`.

### Affected Components

- **The GGUF loader** — `gguf_init_from_file` and the underlying file-open call in the ggml GGUF implementation, where `fopen` is used.
- **Potentially other file-open call sites** across `llama.cpp` / `ggml` that use narrow `fopen` and would share the same limitation; the cleanest fix may hoist a Unicode-safe open helper into a shared location rather than patching one call site. (Scope to be confirmed during reproduction.)
- **Prior art / lineage to study:** PR #10960 (slaren — using `wstring` for backend search paths) establishes a maintainer-accepted precedent for Unicode path handling on Windows; sibling issue #22361 covers non-ASCII argv mangling; the three stalled PRs (#16611, #16609, #16589) attempted this specific model-path fix.

---

## Reproduction Process

> Phase II — to be completed. The documented repro from the issue is captured below as the starting point.

### Environment Setup

[Phase II — pending.] Note: this is a **Windows-only** bug, so reproduction needs a Windows machine or VM. Plan: build `llama.cpp` from source on Windows (record the build hash; the report used `b7539`), then exercise the model-load path. Record toolchain (compiler/CMake versions), backend used, and any build friction here.

### Steps to Reproduce

1. On Windows, create a directory whose name contains non-ASCII characters, e.g. `D:\Загрузки\models\`.
2. Place a `.gguf` model file inside it.
3. Load the model with `llama.cpp` (e.g. `llama-cli -m "D:\Загрузки\models\<model>.gguf"`).
4. **Observed result:** load fails with `failed to open GGUF file`.

### Reproduction Evidence

- **Commit showing reproduction:** [Phase II — pending: link to repro commit/notes in fork]
- **Screenshots/logs:** [Phase II — pending: capture the `gguf_init_from_file` failure]
- **My findings:** [Phase II — pending: confirm the exact `fopen` call site on current `master`; confirm ASCII paths succeed and non-ASCII fail; check whether other tools (server, etc.) share the failure]

### Stalled-PR assessment (issue-specific Phase II step)

- [ ] #16611 — state, last activity, approach, stall reason (work vs. design disagreement)
- [ ] #16609 — same
- [ ] #16589 — same
- [ ] Decision: revive/finish an existing PR, or open one consolidated PR with attribution

---

## Solution Approach

> Draft plan — to be confirmed/revised after Phase II reproduction and the stalled-PR review.

### Analysis

Root cause (agreed in the issue thread): on Windows, narrow `fopen` cannot open paths whose bytes fall outside the active ANSI codepage, so UTF-8/Unicode model paths fail. This is a known Windows limitation, unrelated to the GGUF format, backend, or hardware.

### Proposed Solution

On Windows only: convert the incoming UTF-8 path to UTF-16 (`wchar_t`) and open the file with a Unicode-safe API (`_wfopen` or `CreateFileW`); leave the existing POSIX `fopen` path unchanged on Linux/macOS. Where practical, route file opens through a single cross-platform helper so other call sites benefit and the platform branch lives in one place. Final shape depends on which of the three stalled PRs is closest to mergeable and on maintainer preference (`_wfopen` vs `CreateFileW` vs `std::filesystem`).

### Implementation Plan (UMPIRE)

**Understand:** Narrow `fopen` can't open non-ASCII paths on Windows; need a Unicode-safe open path.

**Match:** Study PR #10960 (accepted `wstring` precedent) and the three stalled PRs to identify the maintainer-preferred pattern and existing helpers.

**Plan:** [Phase III — pending: lock after reproduction + stalled-PR review; choose consolidate-existing vs new PR.]

**Implement:** [Phase III — pending: link branch/commits.]

**Review:** [Phase III — pending: follow llama.cpp CONTRIBUTING; keep the diff tightly scoped and platform-conditional.]

**Evaluate:** [Phase III — pending: verify non-ASCII path loads on Windows, ASCII path still works, Linux/macOS unaffected.]

---

## Testing Strategy

> Phase III — to be completed.

### Manual / Platform Tests

- [ ] [Pending] Windows: load a model from a Cyrillic path (e.g. `D:\Загрузки\...`) — succeeds.
- [ ] [Pending] Windows: load a model from an ASCII path — still succeeds (no regression).
- [ ] [Pending] Windows: additional scripts (CJK characters) to confirm general Unicode coverage.
- [ ] [Pending] Linux/macOS: behavior unchanged.

### Notes

[Pending: how the fix is exercised — CLI load, and any other affected file-open entry points found during reproduction.]

---

## Implementation Notes

### Week [X] Progress

[Pending — document as work proceeds.]

### Code Changes

- **Files modified:** [Pending]
- **Key commits:** [Pending]
- **Approach decisions:** [Pending — including consolidate-vs-new-PR rationale and attribution to original PR authors]

---

## Pull Request

**PR Link:** [Phase IV — pending]

**PR Description:** [Phase IV — pending; the "Understanding the Issue" and "Solution Approach" sections above can be adapted, with attribution to #16611/#16609/#16589 authors as appropriate.]

**Maintainer Feedback:**
- [Date]: [Pending]

**Status:** [Phase IV — not yet submitted]

---

## Learnings & Reflections

> Phase IV — to be completed.

### Technical Skills Gained

[Pending — expected: Windows Unicode file APIs, UTF-8/UTF-16 conversion, cross-platform conditional code, working a fix through a high-traffic OSS review process.]

### Challenges Overcome

[Pending]

### What I'd Do Differently Next Time

[Pending]

---

## Resources Used

- Issue #18571: https://github.com/ggml-org/llama.cpp/issues/18571
- Stalled prior PRs: #16611, #16609, #16589
- Related precedent — PR #10960 (use `wstring` for backend search paths)
- Sibling issue — #22361 (non-ASCII argv mangling on Windows)
- llama.cpp CONTRIBUTING / build docs: https://github.com/ggml-org/llama.cpp
- [Pending: Microsoft docs on `_wfopen` / `CreateFileW`, any threads consulted during reproduction]