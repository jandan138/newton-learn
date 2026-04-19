# Walkthrough Redesign Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Pilot and operationalize the repo's walkthrough redesign so chapter 06 proves the new two-layer pattern before wider rollout.

**Architecture:** Split walkthrough responsibilities into two coordinated documents per chapter: `source-walkthrough.md` as the self-contained beginner/main path, and `source-walkthrough-deep.md` as the precise cross-repo anchor map. This implementation plan only executes the `06_collision` pilot and defines the rollout gate; later chapter conversions happen only after the pilot is reviewed and approved.

**Tech Stack:** Markdown docs, git worktrees, commit-pinned upstream Newton excerpts, `git diff --check`, grep-based template and drift scans.

---

## File Map

- Modify: `chapters/06_collision/source-walkthrough.md`
  - Rebuild as the beginner/main walkthrough pilot.
- Create: `chapters/06_collision/source-walkthrough-deep.md`
  - Add the deep anchor companion for the pilot.
- Modify: `chapters/06_collision/README.md`
  - Update reading order to point readers to beginner vs deep walkthroughs.
- Create: `docs/superpowers/specs/2026-04-19-walkthrough-redesign-design.md`
  - Finalized redesign spec.
- Create: `docs/superpowers/plans/2026-04-19-walkthrough-redesign.md`
  - This implementation plan.

## Task 1: Pilot The New Pattern On Chapter 06

**Files:**
- Modify: `chapters/06_collision/source-walkthrough.md`
- Create: `chapters/06_collision/source-walkthrough-deep.md`
- Modify: `chapters/06_collision/README.md`

- [ ] **Step 1: Rewrite chapter 06 main walkthrough into the beginner/main template**

Make `source-walkthrough.md` use this section order:

```md
## What This Walkthrough Follows
## One-Screen Chapter Map
## Beginner Path
## Main Walkthrough
## Object Ledger
## Stop Here
## Go Deeper
```

The beginner/main file must embed the key code excerpts for `Model.collide(...)`, `compute_shape_aabbs(...)`, broad phase candidate writing, and `write_contact(...)` so readers can follow the chapter without opening `/home/zhuzihou/dev/newton`.

- [ ] **Step 2: Move line-number-heavy anchors into a new deep file**

Create `chapters/06_collision/source-walkthrough-deep.md` with this section order:

```md
## Fast Deep Index
## Exact Handoff Trace
## Optional Branches
## Verification Anchors
```

Move most `path:Lx-Ly` density here, preserving commit-pinned anchors and optional side branches such as broad-phase variants.

- [ ] **Step 3: Update chapter 06 README to route readers correctly**

Add explicit reading guidance like:

```md
- 第一次读源码，先看 `source-walkthrough.md`
- 想精确追到上游文件和行号，再看 `source-walkthrough-deep.md`
```

- [ ] **Step 4: Verify the pilot diff**

Run: `git diff --check -- chapters/06_collision/README.md chapters/06_collision/source-walkthrough.md chapters/06_collision/source-walkthrough-deep.md`

Expected: no output.

## Task 2: Define The Rollout Gate After The Pilot

**Files:**
- Modify: `docs/superpowers/plans/2026-04-19-walkthrough-redesign.md`

- [ ] **Step 1: Record explicit pilot acceptance criteria**

Add a short checklist under this task that must be true before `07-09` or `01-05` can start, for example:

```md
- Beginner can follow chapter 06 main walkthrough without opening the upstream repo
- Deep file still preserves precise source credibility
- README routing between beginner and deep files feels natural
- Excerpt maintenance burden is acceptable after review
```

- [ ] **Step 2: Make later chapter rollout explicitly conditional**

Add wording that `07-09` and `01-05` are intentionally out of scope for this plan until the chapter 06 pilot is reviewed.

- [ ] **Step 3: Note the chapter 09 dependency explicitly**

State that `09_variational_solvers` only enters the rollout queue after its base chapter files are merged to `main`.

## Task 3: Document The Future Migration Queue Without Executing It

**Files:**
- Modify: `docs/superpowers/plans/2026-04-19-walkthrough-redesign.md`

- [ ] **Step 1: Record the post-pilot priority queue**

List the intended rollout order only:

```md
1. `07_constraints_contacts_math`
2. `08_rigid_solvers`
3. `09_variational_solvers`
4. `01_warp_basics`
5. `02_newton_arch`
6. `03_math_geometry`
7. `04_scene_usd`
8. `05_rigid_articulation`
```

- [ ] **Step 2: Restate the beginner-file requirement for later chapters**

Add a short rule that every future conversion must preserve:

```md
- `source-walkthrough.md` must remain self-contained and excerpt-backed
- `source-walkthrough-deep.md` must absorb most line-range-heavy anchors
```

## Task 4: Verify The Pilot-Scope Plan Artifacts

**Files:**
- Verify the spec, plan, and pilot chapter files only

- [ ] **Step 1: Run repository diff check**

Run: `git diff --check`

Expected: no output.

- [ ] **Step 2: Scan for old walkthrough assumptions in touched files**

Run: `rg -n "07-09|01-05|later chapter rollout|self-contained and excerpt-backed|source-walkthrough-deep.md" docs/superpowers/plans/2026-04-19-walkthrough-redesign.md`

Expected: the plan should clearly read as chapter-06 pilot first, rollout later.

- [ ] **Step 3: Review final diff by chapter cluster**

Run: `git diff -- chapters/06_collision docs/superpowers/specs/2026-04-19-walkthrough-redesign-design.md docs/superpowers/plans/2026-04-19-walkthrough-redesign.md`

Expected: changes should reflect the chapter 06 pilot plus the repo-wide redesign spec/plan only.

- [ ] **Step 4: Stop before commit unless explicitly requested**

No commit by default. If the user later requests one, use the repo's recent `docs:` commit style and commit the walkthrough redesign in logical batches.
