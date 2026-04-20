# Chapters 02 03 07 Walkthrough Refinement Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Refine the main walkthroughs for chapters 02, 03, and 07 so they better explain their backstory and concept chain to true beginners, using the refined chapter 01 standard as the benchmark.

**Architecture:** Keep the walkthrough rollout's main/deep split intact. The primary target is the three `source-walkthrough.md` files, preserving their chapter roles while making them more self-contained, more learning-order-driven, and more anchored in a single narrative question per chapter. If a chapter package develops wording drift because of the main-file rewrite, allow the smallest possible README / principle / examples consistency edits.

**Tech Stack:** Markdown docs, git worktree, excerpt-backed main walkthroughs, `git diff --check`, multi-round doc review.

---

## File Map

- Modify:
  - `chapters/02_newton_arch/source-walkthrough.md`
  - `chapters/03_math_geometry/source-walkthrough.md`
  - `chapters/07_constraints_contacts_math/source-walkthrough.md`
- Optional consistency edits if needed:
  - `chapters/02_newton_arch/README.md`
  - `chapters/02_newton_arch/principle.md`
  - `chapters/02_newton_arch/examples.md`
- Create:
  - `docs/superpowers/specs/2026-04-20-chapters-02-03-07-walkthrough-refinement-design.md`
  - `docs/superpowers/plans/2026-04-20-chapters-02-03-07-walkthrough-refinement.md`

## Task 1: Refine Chapter 02 Main Walkthrough

**Files:**
- Modify: `chapters/02_newton_arch/source-walkthrough.md`

- [ ] **Step 1: Add a stronger opening anchored in the user-facing entry story**

The file should answer, before object names pile up:

```text
当我运行一个 Newton example 时，到底是谁把我带到 Model / State / Control / Contacts / Solver？
```

- [ ] **Step 2: Reorder the file around the single entry-to-runtime storyline**

Make the chapter feel like one path:

```text
example short name
-> launcher / example module
-> Example object
-> Model / State / Control / Contacts / Solver
-> simulate loop
```

instead of a source-layout-first tour of exports and routing.

- [ ] **Step 3: Add beginner-safe first-use definitions for the core runtime objects**

At first mention, briefly define:

```md
- Model
- State
- Control
- Contacts
- Solver
```

- [ ] **Step 4: Add clearer checkpoint-style lines**

After the key stages, add short “如果你现在还答不上来 X，就先不要继续往下读” style checks.

## Task 2: Refine Chapter 03 Main Walkthrough

**Files:**
- Modify: `chapters/03_math_geometry/source-walkthrough.md`

- [ ] **Step 1: Replace the terminology-bundle opening with a why-story**

The file should begin from a beginner question like:

```text
为什么 04/05/06 里总在出现 transform、frame、body_q、shape_transform、body_qd、inertia？
```

- [ ] **Step 2: Rebuild the chapter as a “legend for later chapters” rather than a math inventory**

Give it one coherent storyline:

```text
local relationships
-> transform chains
-> spatial quantities
-> shape representation
-> inertia as geometry-to-dynamics bridge
```

- [ ] **Step 3: Add beginner-safe first-use explanations for the hardest objects**

Especially define in plainer language:

```md
- frame
- spatial quantity / 6D quantity
- GeoType
- inertia
```

- [ ] **Step 4: Add more “why this matters later” linking sentences**

Each main stage should explicitly say which later chapters it is preparing the reader for.

## Task 3: Refine Chapter 07 Main Walkthrough

**Files:**
- Modify: `chapters/07_constraints_contacts_math/source-walkthrough.md`

- [ ] **Step 1: Add a stronger opening anchored in “why contact is not enough”**

Before rows/Jacobians/Delassus appear, the file should answer:

```text
chapter 06 已经给了接触点和法线，为什么 solver 还不能直接停在这里？
```

- [ ] **Step 2: Reorder the chapter around the necessity chain**

The main line should feel like:

```text
contact geometry
-> solver needs motion restrictions
-> those restrictions become rows
-> rows need Jacobians
-> Jacobians induce Delassus
```

- [ ] **Step 3: Delay the densest math nouns until the physical story is established**

Make sure `solver-facing contact`, `rows`, `Jacobian`, and `Delassus` each arrive after a plain-language justification.

- [ ] **Step 4: Add stronger “can you now explain this?” checkpoints**

Readers should be able to stop and restate:

```text
为什么光有接触点和法线还不够。
为什么一条接触要长成 row 结构。
```

## Task 4: Final Verification

**Files:**
- Verify the three refined walkthroughs and supporting spec/plan files.

- [ ] **Step 1: Run file-level diff check**

Run: `git diff --check -- chapters/02_newton_arch/source-walkthrough.md chapters/03_math_geometry/source-walkthrough.md chapters/07_constraints_contacts_math/source-walkthrough.md docs/superpowers/specs/2026-04-20-chapters-02-03-07-walkthrough-refinement-design.md docs/superpowers/plans/2026-04-20-chapters-02-03-07-walkthrough-refinement.md`

Expected: no output.

- [ ] **Step 2: Re-read for self-containment**

Check that each main walkthrough can stand on its own for the core backstory, even if `principle.md` still exists as the gentler concept layer.

- [ ] **Step 3: Review the final diff**

Run: `git diff -- chapters/02_newton_arch/source-walkthrough.md chapters/03_math_geometry/source-walkthrough.md chapters/07_constraints_contacts_math/source-walkthrough.md docs/superpowers/specs/2026-04-20-chapters-02-03-07-walkthrough-refinement-design.md docs/superpowers/plans/2026-04-20-chapters-02-03-07-walkthrough-refinement.md`

Expected: changes stay focused on beginner clarity and backstory, not on deep-walkthrough structure.

- [ ] **Step 4: Stop before commit unless explicitly requested**

No commit by default. If the user later requests one, use the repo's recent `docs:` commit style and commit only the refinement files.
