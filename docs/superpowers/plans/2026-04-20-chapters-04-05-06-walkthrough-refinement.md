# Chapters 04 05 06 Walkthrough Refinement Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Refine the main walkthroughs for chapters 04, 05, and 06 so they better explain their backstory and concept chain to true beginners, completing the `02-07` walkthrough refinement pass.

**Architecture:** Keep the walkthrough rollout's main/deep split intact. Only rewrite the three `source-walkthrough.md` files, preserving their chapter roles while making their opening questions, stage order, and beginner vocabulary more human and more necessity-driven.

**Tech Stack:** Markdown docs, git worktree, excerpt-backed main walkthroughs, `git diff --check`, multi-round doc review.

---

## File Map

- Modify:
  - `chapters/04_scene_usd/source-walkthrough.md`
  - `chapters/05_rigid_articulation/source-walkthrough.md`
  - `chapters/06_collision/source-walkthrough.md`
- Create:
  - `docs/superpowers/specs/2026-04-20-chapters-04-05-06-walkthrough-refinement-design.md`
  - `docs/superpowers/plans/2026-04-20-chapters-04-05-06-walkthrough-refinement.md`

## Task 1: Refine Chapter 04 Main Walkthrough

**Files:**
- Modify: `chapters/04_scene_usd/source-walkthrough.md`

- [ ] **Step 1: Add a stronger beginner question at the top**

The file should answer up front:

```text
我在 USD / 场景文件里写了一个 body / joint / shape，Newton 到底怎样把它变成 Model 里的数组？
```

- [ ] **Step 2: Reorder the chapter around importer -> resolver -> builder -> finalize**

The chapter should feel like one necessity chain, not a list of parser terms.

- [ ] **Step 3: Add beginner-safe first-use definitions**

Especially define:

```md
- importer
- authored attrs
- schema resolver
- MassAPI
- finalize()
```

## Task 2: Refine Chapter 05 Main Walkthrough

**Files:**
- Modify: `chapters/05_rigid_articulation/source-walkthrough.md`

- [ ] **Step 1: Start from a smallest articulation picture before slices**

The opening should first ground the reader in “two links + one joint” before introducing flat layout fields.

- [ ] **Step 2: Reframe the chapter around the state handoff**

Make the storyline clearly feel like:

```text
minimal tree
-> flat layout
-> joint-space state
-> FK / velocity propagation
-> body-space state
-> Featherstone continues that chain
```

- [ ] **Step 3: Add plain-language first-use definitions where the chapter is still too slice-heavy**

Especially for `articulation_start`, `joint_q_start`, FK, and generalized coordinates.

## Task 3: Refine Chapter 06 Main Walkthrough

**Files:**
- Modify: `chapters/06_collision/source-walkthrough.md`

- [ ] **Step 1: Strengthen the opening question**

Answer clearly:

```text
为什么不是 body 直接互相碰撞？为什么要先把 shape 放到 world，再先找 candidate pair？
```

- [ ] **Step 2: Make broad phase / narrow phase / writer read like a necessity chain**

The chapter should feel like:

```text
body provides motion
-> shape becomes world query object
-> broad phase cheaply filters possibilities
-> narrow phase computes actual geometry
-> writer converts it into Contacts
```

- [ ] **Step 3: Add clearer first-use explanations**

Especially for:

```md
- broad phase
- narrow phase
- candidate pair
- ContactData
- margin / gap
```

## Task 4: Final Verification

**Files:**
- Verify the three refined walkthroughs and supporting spec/plan files.

- [ ] **Step 1: Run file-level diff check**

Run: `git diff --check -- chapters/04_scene_usd/source-walkthrough.md chapters/05_rigid_articulation/source-walkthrough.md chapters/06_collision/source-walkthrough.md docs/superpowers/specs/2026-04-20-chapters-04-05-06-walkthrough-refinement-design.md docs/superpowers/plans/2026-04-20-chapters-04-05-06-walkthrough-refinement.md`

Expected: no output.

- [ ] **Step 2: Re-read for self-containment**

Check that each main walkthrough can stand on its own for the core backstory.

- [ ] **Step 3: Review the final diff**

Run: `git diff -- chapters/04_scene_usd/source-walkthrough.md chapters/05_rigid_articulation/source-walkthrough.md chapters/06_collision/source-walkthrough.md docs/superpowers/specs/2026-04-20-chapters-04-05-06-walkthrough-refinement-design.md docs/superpowers/plans/2026-04-20-chapters-04-05-06-walkthrough-refinement.md`

Expected: changes stay focused on beginner clarity and backstory.
