# Walkthrough Rollout Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Execute the repo-wide walkthrough redesign by piloting the new beginner/deep split on chapter 06, then rolling the same pattern through chapters 07-09 and backfilling 01-05.

**Architecture:** Keep the redesign spec merged to `main` as the governing contract. Implement in one clean rollout branch, but preserve the rollout order the user requested: first prove the `06_collision` pilot, then port the same two-layer walkthrough pattern to `07/08/09`, then backfill `01-05`, keeping README routing and commit-pinned source conventions consistent throughout.

**Tech Stack:** Markdown docs, git worktree, commit-pinned upstream Newton excerpts, `git diff --check`, ripgrep-based walkthrough scans, multi-round doc review.

---

## File Map

- Modify/Create walkthrough pairs under:
  - `chapters/06_collision/`
  - `chapters/07_constraints_contacts_math/`
  - `chapters/08_rigid_solvers/`
  - `chapters/09_variational_solvers/`
  - `chapters/01_warp_basics/`
  - `chapters/02_newton_arch/`
  - `chapters/03_math_geometry/`
  - `chapters/04_scene_usd/`
  - `chapters/05_rigid_articulation/`
- Modify touched chapter `README.md` files where they route readers to walkthrough files.
- Reuse merged spec:
  - `docs/superpowers/specs/2026-04-19-walkthrough-redesign-design.md`
- Create this rollout plan:
  - `docs/superpowers/plans/2026-04-19-walkthrough-rollout.md`

## Task 1: Pilot The Split On Chapter 06

**Files:**
- Modify: `chapters/06_collision/source-walkthrough.md`
- Create: `chapters/06_collision/source-walkthrough-deep.md`
- Modify: `chapters/06_collision/README.md`

- [ ] **Step 1: Rebuild chapter 06 main walkthrough as the beginner/main version**

Apply the redesign spec literally:

```md
## What This Walkthrough Follows
## One-Screen Chapter Map
## Beginner Path
## Main Walkthrough
## Object Ledger
## Stop Here
## Go Deeper
```

Embed the critical excerpts for `Model.collide(...)`, `compute_shape_aabbs(...)`, broad-phase candidate writing, `ContactData`, and `write_contact(...)` so a beginner can read the file without opening the upstream repo.

- [ ] **Step 2: Create chapter 06 deep walkthrough**

Create `chapters/06_collision/source-walkthrough-deep.md` with:

```md
## Fast Deep Index
## Exact Handoff Trace
## Optional Branches
## Verification Anchors
```

Move the dense line-range-heavy anchors and optional broad-phase branches here.

- [ ] **Step 3: Update chapter 06 README routing**

Add explicit guidance like:

```md
- 第一次读源码，先看 `source-walkthrough.md`
- 想精确追到上游文件和行号，再看 `source-walkthrough-deep.md`
```

- [ ] **Step 4: Verify the pilot chapter only**

Run: `git diff --check -- chapters/06_collision/README.md chapters/06_collision/source-walkthrough.md chapters/06_collision/source-walkthrough-deep.md`

Expected: no output.

- [ ] **Step 5: Review the pilot before continuing**

Run a focused documentation review after the pilot lands in the branch. If the review says the beginner/main file still depends too much on cross-repo jumps, fix chapter 06 before touching 07-09.

## Task 2: Port The Same Pattern To Chapters 07-09

**Files:**
- Modify/Create:
  - `chapters/07_constraints_contacts_math/source-walkthrough*.md`
  - `chapters/08_rigid_solvers/source-walkthrough*.md`
  - `chapters/09_variational_solvers/source-walkthrough*.md`
- Modify:
  - `chapters/07_constraints_contacts_math/README.md`
  - `chapters/08_rigid_solvers/README.md`
  - `chapters/09_variational_solvers/README.md`

- [ ] **Step 1: Convert chapter 07 after the pilot passes**

Preserve the `Contacts -> rows -> Jacobians -> Delassus` beginner storyline in `source-walkthrough.md`. Move Kamino-heavy exact anchors into `source-walkthrough-deep.md`.

- [ ] **Step 2: Convert chapter 08 with the same split**

Keep `shared contract -> family split -> Kamino continuation` in the beginner/main file. Move dense solver file anchors into the deep file.

- [ ] **Step 3: Convert chapter 09 with the same split**

Keep `shared variational problem -> XPBD -> VBD -> Style3D` in the beginner/main file. Move exact kernel/solver anchor ranges into the deep file.

- [ ] **Step 4: Update README routing for 07-09**

Every touched README should consistently direct first-time readers to `source-walkthrough.md` and deeper readers to `source-walkthrough-deep.md`.

- [ ] **Step 5: Verify the 07-09 rollout cluster**

Run: `git diff --check -- chapters/07_constraints_contacts_math chapters/08_rigid_solvers chapters/09_variational_solvers`

Expected: no output.

## Task 3: Backfill Chapters 01-05

**Files:**
- Modify/Create walkthrough pairs in:
  - `chapters/01_warp_basics/`
  - `chapters/02_newton_arch/`
  - `chapters/03_math_geometry/`
  - `chapters/04_scene_usd/`
  - `chapters/05_rigid_articulation/`
- Modify corresponding `README.md` files if walkthrough routing is mentioned there.

- [ ] **Step 1: Convert 01-02 first**

These chapters are closest to the learning ramp and should adopt the new beginner/main vs deep split before the more technical middle chapters are considered stable.

- [ ] **Step 2: Convert 03-05 second**

These should follow the exact same file role split after 01-02 are updated.

- [ ] **Step 3: Standardize README routing language across 01-05**

Each touched README should use the same wording pattern introduced in chapters 06-09.

- [ ] **Step 4: Verify the 01-05 backfill cluster**

Run: `git diff --check -- chapters/01_warp_basics chapters/02_newton_arch chapters/03_math_geometry chapters/04_scene_usd chapters/05_rigid_articulation`

Expected: no output.

## Task 4: Global Verification

**Files:**
- Verify all touched walkthrough pairs, README routing updates, and this rollout plan.

- [ ] **Step 1: Run repository diff check**

Run: `git diff --check`

Expected: no output.

- [ ] **Step 2: Scan main walkthroughs for old line-range-heavy style**

Run: `rg -n "L[0-9]+-L[0-9]+|如果你只想做一次最小 first pass|推荐的一遍读法" chapters/0{1,2,3,4,5,6,7,8,9}_*/source-walkthrough.md`

Expected: the main walkthrough files should now contain only rare line-range mentions; most dense anchor usage should live in `source-walkthrough-deep.md`.

- [ ] **Step 3: Scan deep walkthroughs for existence and routing coverage**

Run: `rg -n "source-walkthrough-deep.md" chapters/0{1,2,3,4,5,6,7,8,9}_*/README.md`

Expected: every touched chapter README should route readers to the deep file.

- [ ] **Step 4: Review the final diff by rollout scope**

Run: `git diff -- chapters/01_warp_basics chapters/02_newton_arch chapters/03_math_geometry chapters/04_scene_usd chapters/05_rigid_articulation chapters/06_collision chapters/07_constraints_contacts_math chapters/08_rigid_solvers chapters/09_variational_solvers docs/superpowers/plans/2026-04-19-walkthrough-rollout.md`

Expected: changes should reflect the two-layer walkthrough rollout and consistent README routing only.

- [ ] **Step 5: Stop before commit unless explicitly requested**

No commit by default. If the user later requests one, use the repo's recent `docs:` commit style and commit the rollout in logical batches.
