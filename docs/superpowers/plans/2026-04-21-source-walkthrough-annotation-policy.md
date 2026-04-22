# Source Walkthrough Annotation Policy Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Apply a repo-wide annotation policy so mainline `source-walkthrough.md` files default to Chinese inline comments for beginner-critical source excerpts, while preserving clean excerpts for math-heavy and structurally dense blocks.

**Architecture:** First update the walkthrough template so future chapters inherit the policy. Then update existing `source-walkthrough.md` files in chapter groups, adding concise Chinese inline comments to beginner-critical excerpts and explicit block-level notes where the policy allows clean code. Finish with repo-wide review and diff hygiene checks so the style becomes consistent without turning every code block into prose.

**Tech Stack:** Markdown docs, existing walkthrough structure, upstream Newton source excerpts, `git diff --check`, ripgrep placeholder scan.

---

## File Map

- Modify:
  - `templates/source-walkthrough.md.tmpl`
  - `chapters/01_warp_basics/source-walkthrough.md`
  - `chapters/02_newton_arch/source-walkthrough.md`
  - `chapters/03_math_geometry/source-walkthrough.md`
  - `chapters/04_scene_usd/source-walkthrough.md`
  - `chapters/05_rigid_articulation/source-walkthrough.md`
  - `chapters/06_collision/source-walkthrough.md`
  - `chapters/07_constraints_contacts_math/source-walkthrough.md`
  - `chapters/08_rigid_solvers/source-walkthrough.md`
  - `chapters/09_variational_solvers/source-walkthrough.md`
  - `chapters/10_softbody_cloth_cable/source-walkthrough.md`
  - `chapters/11_mpm/source-walkthrough.md`
  - `chapters/12_sensors_ik/source-walkthrough.md`
- Create:
  - `docs/superpowers/specs/2026-04-21-source-walkthrough-annotation-policy-design.md`
  - `docs/superpowers/plans/2026-04-21-source-walkthrough-annotation-policy.md`

## Task 1: Update The Source Walkthrough Template

**Files:**
- Modify: `templates/source-walkthrough.md.tmpl`

- [ ] **Step 1: Replace the old skeleton with the current main walkthrough structure**

The template should reflect the repo’s current walkthrough shape, not the old table-heavy skeleton. It should include these sections in order:

```text
What This Walkthrough Follows
One-Screen Chapter Map
Beginner Path
Main Walkthrough
Object Ledger
Stop Here
Go Deeper
```

- [ ] **Step 2: Add annotation-policy guidance to the template comments / scaffolding**

The template should explicitly instruct future writers:

```text
Default to Chinese inline comments for beginner-critical source excerpts.
Keep math-heavy / dense excerpts clean when inline comments would damage readability.
Use a block note like “以下摘录为教学注释版，注释非原源码。” when the excerpt is teaching-annotated.
```

- [ ] **Step 3: Add one annotated excerpt example and one clean-excerpt example**

The template should show both policy branches:

```python
self.model = builder.finalize()  # builder 冻结成可运行的 Model
self.state_0 = self.model.state()  # 当前状态缓冲区
```

and:

```python
J = build_contact_jacobians(...)
D = J M^-1 J^T
```

with surrounding text clarifying why the second block stays clean.

## Task 2: Annotate Chapters 01-04

**Files:**
- Modify:
  - `chapters/01_warp_basics/source-walkthrough.md`
  - `chapters/02_newton_arch/source-walkthrough.md`
  - `chapters/03_math_geometry/source-walkthrough.md`
  - `chapters/04_scene_usd/source-walkthrough.md`

- [ ] **Step 1: Annotate beginner-critical control-flow and assembly excerpts**

For each file, add concise Chinese inline comments to the excerpts that carry the chapter’s first-pass story:

```text
01: loop -> kernel -> launch handoff
02: launcher / Example.__init__ / simulate chain
03: transform-chain and pose/state excerpts where single-line role is not obvious
04: scene import / builder handoff excerpts
```

- [ ] **Step 2: Leave dense math / declaration blocks clean where needed**

If a block is mostly declarations or frame/math structure, keep the code clean and add a block note or stronger `Verification cues` instead of line-by-line comments.

- [ ] **Step 3: Add block-level note where annotated excerpt is clearly teaching-edited**

Use:

```text
以下摘录为教学注释版，注释非原源码。
```

before excerpt blocks that are both shortened and line-commented.

## Task 3: Annotate Chapters 05-08

**Files:**
- Modify:
  - `chapters/05_rigid_articulation/source-walkthrough.md`
  - `chapters/06_collision/source-walkthrough.md`
  - `chapters/07_constraints_contacts_math/source-walkthrough.md`
  - `chapters/08_rigid_solvers/source-walkthrough.md`

- [ ] **Step 1: Add inline comments to the excerpts that define object roles and route splits**

Focus inline comments on:

```text
05: articulation assembly / FK / state flow
06: collision-pipeline handoff excerpts
07: geometry-to-solver handoff blocks, but not the densest math identities
08: solver-family route split excerpts and public step contract excerpts
```

- [ ] **Step 2: Preserve clean code for row/Jacobian/Delassus math-heavy excerpts**

Where code shape matters more than single-line labels, keep the excerpt clean and make the surrounding prose do the teaching.

## Task 4: Annotate Chapters 09-12

**Files:**
- Modify:
  - `chapters/09_variational_solvers/source-walkthrough.md`
  - `chapters/10_softbody_cloth_cable/source-walkthrough.md`
  - `chapters/11_mpm/source-walkthrough.md`
  - `chapters/12_sensors_ik/source-walkthrough.md`

- [ ] **Step 1: Add inline comments to first-pass route excerpts**

Focus on code blocks where readers most need one-line role labels:

```text
09: solver route setup and top-level update chain
10: builder/object-family assembly and top-level simulation chain
11: persistent-particle / grid-workspace / P2G-G2P route excerpts
12: read-side / write-side backbone excerpts and IK loop skeleton
```

- [ ] **Step 2: Keep dense kernels, transfer code, and optimizer internals selective**

In chapter 11 and chapter 12 especially, avoid turning structurally dense blocks into over-commented prose walls. Use only the minimum inline comments needed to preserve the beginner story.

## Task 5: Consistency Sweep

**Files:**
- Modify any of the walkthroughs above if the first pass reveals inconsistent wording or policy drift.

- [ ] **Step 1: Standardize the teaching-edit note wording**

When a block is annotated and clearly not raw source, use the same sentence:

```text
以下摘录为教学注释版，注释非原源码。
```

- [ ] **Step 2: Standardize the exception behavior**

Where a file keeps dense excerpts clean, ensure the surrounding prose explicitly explains why the block is being read cleanly instead of annotated line by line.

## Task 6: Verification And Review

**Files:**
- Verify all modified walkthrough files plus template/spec/plan.

- [ ] **Step 1: Run file-level diff checks**

Run:

`git diff --check -- templates/source-walkthrough.md.tmpl chapters/01_warp_basics/source-walkthrough.md chapters/02_newton_arch/source-walkthrough.md chapters/03_math_geometry/source-walkthrough.md chapters/04_scene_usd/source-walkthrough.md chapters/05_rigid_articulation/source-walkthrough.md chapters/06_collision/source-walkthrough.md chapters/07_constraints_contacts_math/source-walkthrough.md chapters/08_rigid_solvers/source-walkthrough.md chapters/09_variational_solvers/source-walkthrough.md chapters/10_softbody_cloth_cable/source-walkthrough.md chapters/11_mpm/source-walkthrough.md chapters/12_sensors_ik/source-walkthrough.md docs/superpowers/specs/2026-04-21-source-walkthrough-annotation-policy-design.md docs/superpowers/plans/2026-04-21-source-walkthrough-annotation-policy.md`

Expected: no output.

- [ ] **Step 2: Run placeholder scan**

Run:

`rg -n "TODO|TBD" templates/source-walkthrough.md.tmpl chapters/01_warp_basics/source-walkthrough.md chapters/02_newton_arch/source-walkthrough.md chapters/03_math_geometry/source-walkthrough.md chapters/04_scene_usd/source-walkthrough.md chapters/05_rigid_articulation/source-walkthrough.md chapters/06_collision/source-walkthrough.md chapters/07_constraints_contacts_math/source-walkthrough.md chapters/08_rigid_solvers/source-walkthrough.md chapters/09_variational_solvers/source-walkthrough.md chapters/10_softbody_cloth_cable/source-walkthrough.md chapters/11_mpm/source-walkthrough.md chapters/12_sensors_ik/source-walkthrough.md docs/superpowers/specs/2026-04-21-source-walkthrough-annotation-policy-design.md docs/superpowers/plans/2026-04-21-source-walkthrough-annotation-policy.md`

Expected: no output.

- [ ] **Step 3: Re-read for policy compliance**

Confirm these two statements are simultaneously true:

```text
beginner-critical excerpts now usually carry concise Chinese inline comments
math-heavy / structurally dense excerpts are still allowed to stay clean
```

- [ ] **Step 4: Review the final diff**

Run:

`git diff -- templates/source-walkthrough.md.tmpl chapters/01_warp_basics/source-walkthrough.md chapters/02_newton_arch/source-walkthrough.md chapters/03_math_geometry/source-walkthrough.md chapters/04_scene_usd/source-walkthrough.md chapters/05_rigid_articulation/source-walkthrough.md chapters/06_collision/source-walkthrough.md chapters/07_constraints_contacts_math/source-walkthrough.md chapters/08_rigid_solvers/source-walkthrough.md chapters/09_variational_solvers/source-walkthrough.md chapters/10_softbody_cloth_cable/source-walkthrough.md chapters/11_mpm/source-walkthrough.md chapters/12_sensors_ik/source-walkthrough.md docs/superpowers/specs/2026-04-21-source-walkthrough-annotation-policy-design.md docs/superpowers/plans/2026-04-21-source-walkthrough-annotation-policy.md`

Expected: the diff shows consistent teaching annotation, not blanket comment spam.

- [ ] **Step 5: No commit unless explicitly requested**

Do not create a git commit unless the user explicitly asks for one.
