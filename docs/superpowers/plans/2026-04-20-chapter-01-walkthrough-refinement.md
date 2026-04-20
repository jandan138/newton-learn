# Chapter 01 Walkthrough Refinement Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rewrite `chapters/01_warp_basics/source-walkthrough.md` so that a true beginner can follow the whole Warp backstory and execution chain without already knowing why Warp exists in Newton.

**Architecture:** Keep the existing beginner/deep walkthrough split, but upgrade only the main walkthrough. Rebuild the file around a single “ordinary Python loop -> Warp -> Newton” story, add explicit definition/checkpoint scaffolding, and reorder the stages so the reader learns Warp first and only then sees how Newton's `Model / State / Control` fit into that execution model.

**Tech Stack:** Markdown docs, git worktree, embedded source excerpts, `git diff --check`, multi-round doc review.

---

## File Map

- Modify: `chapters/01_warp_basics/source-walkthrough.md`
  - Rewrite the main walkthrough into a more beginner-complete narrative.
- Create: `docs/superpowers/specs/2026-04-20-chapter-01-walkthrough-refinement-design.md`
  - Refinement design for this pass.
- Create: `docs/superpowers/plans/2026-04-20-chapter-01-walkthrough-refinement.md`
  - This implementation plan.

## Task 1: Rebuild The Warp Backstory At The Top

**Files:**
- Modify: `chapters/01_warp_basics/source-walkthrough.md`

- [ ] **Step 1: Add a stronger opening before the walkthrough sections**

Insert an opening that explains, in plain language:

```md
- Newton 为什么需要 Warp
- 普通 Python `for` 循环在仿真里代表什么
- Warp 怎样把“单元素规则”和“整批 dispatch”拆开
```

This must answer “Why are we here?” before the file starts discussing `wp.array`, `wp.launch`, or `wp.tid()`.

- [ ] **Step 2: Add one simple running example**

Introduce a tiny ordinary-loop example such as:

```python
for i in range(n):
    out[i] = f(x[i])
```

Then state how the rest of the file will repeatedly map this to Warp.

- [ ] **Step 3: Verify the opening reads as self-contained**

Re-read the first ~60 lines and check that a reader who has not opened `principle.md` can still answer:

```text
为什么 Warp 会出现在 Newton 里？
它大体是在替代什么样的普通写法？
```

## Task 2: Reorder The Main Stages Into Learning Order

**Files:**
- Modify: `chapters/01_warp_basics/source-walkthrough.md`

- [ ] **Step 1: Move from “buffers first” to “loop -> kernel/launch first”**

Rewrite the stage flow so the reader encounters the concepts in this order:

```text
Why Warp
-> kernel as per-element rule
-> launch and tid as batch execution
-> wp.array back in Newton's Model / State / Control
-> atomic shared writeback
-> Graph / tile as execution organization
```

- [ ] **Step 2: Make each stage solve a clearly stated problem**

Each stage should begin by telling the reader what problem is being solved, for example:

```md
在普通 `for` 循环里，索引 `i` 是循环变量。
Warp 里没有这个显式循环，所以线程需要一种方式知道“我是哪一个 i”。
这就是 `wp.tid()` 在解决的问题。
```

- [ ] **Step 3: Preserve the existing real Newton examples, but subordinate them to the single story**

`selection_materials`, `SolverXPBD`, `mpm_twoway_coupling`, and `diffsim_bear` should remain, but each must be introduced as “the same idea now put back into a real Newton file” rather than as a new topic.

## Task 3: Add Definition And Checkpoint Scaffolding

**Files:**
- Modify: `chapters/01_warp_basics/source-walkthrough.md`

- [ ] **Step 1: Add definition-style explanations at first use**

At first mention, define in plain language:

```md
- kernel
- wp.launch
- wp.tid()
- wp.array
- wp.atomic_add
- Graph
- tile
```

These should be compact, beginner-safe, and immediately tied to what problem each construct solves.

- [ ] **Step 2: Add at least one concrete conflict picture for atomic writeback**

Show a tiny scenario where two threads both want to update the same slot, then explain atomic accumulation as the fix.

- [ ] **Step 3: Add checkpoint-style lines after the key stages**

Use short lines such as:

```md
如果你现在还说不清 X，就先不要继续往下读。
```

These should help the reader notice when they have lost the thread.

## Task 4: Keep The File Within The Walkthrough System

**Files:**
- Modify: `chapters/01_warp_basics/source-walkthrough.md`

- [ ] **Step 1: Preserve the redesign structure**

The file must still retain:

```md
## What This Walkthrough Follows
## One-Screen Chapter Map
## Beginner Path
## Main Walkthrough
## Object Ledger
## Stop Here
## Go Deeper
```

- [ ] **Step 2: Do not collapse the deep file back into the main file**

Precise file/symbol/line anchors should remain mostly in `source-walkthrough-deep.md`; the main file should stay excerpt-driven and beginner-oriented.

- [ ] **Step 3: Verify that the `Stop Here` summary still matches the new story order**

The final summary should now read like the conclusion of the rewritten “ordinary loop -> Warp -> Newton” storyline.

## Task 5: Final Verification

**Files:**
- Verify the refined main walkthrough and supporting spec/plan files.

- [ ] **Step 1: Run file-level diff check**

Run: `git diff --check -- chapters/01_warp_basics/source-walkthrough.md docs/superpowers/specs/2026-04-20-chapter-01-walkthrough-refinement-design.md docs/superpowers/plans/2026-04-20-chapter-01-walkthrough-refinement.md`

Expected: no output.

- [ ] **Step 2: Re-read for beginner self-containment**

Check that the main walkthrough no longer depends on `principle.md` for the core backstory.

- [ ] **Step 3: Review the final diff**

Run: `git diff -- chapters/01_warp_basics/source-walkthrough.md docs/superpowers/specs/2026-04-20-chapter-01-walkthrough-refinement-design.md docs/superpowers/plans/2026-04-20-chapter-01-walkthrough-refinement.md`

Expected: changes stay focused on chapter 01 beginner readability only.

- [ ] **Step 4: Stop before commit unless explicitly requested**

No commit by default. If the user later requests one, use the repo's recent `docs:` commit style and commit only the refinement files.
