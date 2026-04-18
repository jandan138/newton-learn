# Chapters 00-02 Content Upgrade Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rewrite chapters `00`-`02` into a continuous introductory learning path that is easier to read, gives more external prerequisite knowledge in `00`, fully materializes `01`, and strengthens `02` as the first real Newton chapter.

**Architecture:** Keep the existing repo structure and chapter boundaries. Move more prerequisite explanation into `00`, turn `01` from a template into a real Warp chapter with a new `principle.md`, and revise `02` so it clearly receives the reader from `00/01` instead of standing alone as a seed note.

**Tech Stack:** Markdown, existing chapter frontmatter, repository writing conventions, Newton/Warp terminology already established in the repo.

---

### Task 1: Rewrite `00_prerequisites/principle.md` as a readable pre-Newton primer

**Files:**
- Modify: `chapters/00_prerequisites/principle.md`
- Reference: `chapters/00_prerequisites/README.md`, `chapters/02_newton_arch/principle.md`, `docs/superpowers/specs/2026-04-18-chapters-00-02-content-upgrade-design.md`

- [ ] **Step 1: Replace the current outline with a stronger section order**

Write the document with this section skeleton so the reader goes from motivation to intuition to navigation:

```md
## 0. 为什么先有这一章
## 1. 先建立“仿真一步”的通用直觉
## 2. 刚体与机器人动力学最小词汇
## 3. GPU / Warp 最小直觉
## 4. 现在卡住时，应该先跳去 01 还是 02
```

- [ ] **Step 2: Rewrite the opening to explain the chapter's role in plain language**

Write 2-4 short paragraphs that establish:

```md
你不需要先学完整套机器人动力学、CUDA 或 Newton 文档，才有资格读后面的章节。

这一章的作用，只是把后两章马上会撞上的前置知识铺成一条缓坡：
- 看到陌生词时知道它大概在解决什么问题
- 看到 Warp 术语时不至于完全失去执行模型
- 进入 Newton 本体前，先对“仿真一步到底在做什么”有统一直觉
```

- [ ] **Step 3: Add the “simulation step” primer before any Newton-specific layering**

Write a short explanatory section that introduces these ideas without relying on Newton names first:

```md
- 世界里有什么：静态描述
- 这一刻系统在哪、速度多大：当前状态
- 这一拍给系统什么驱动或目标：外部输入
- 做一步更新：根据前面三者算出下一刻
```

Then add one short bridge paragraph connecting this to `02`:

```md
到 `02_newton_arch` 时，你会看到 Newton 把这套通用直觉组织成更明确的对象分层：`Model / State / Control / Solver`。
现在先不用背名字，只要先接受“仿真器每一步都在消费静态描述、当前状态和外部输入，然后产出下一步状态”这件事。
```

- [ ] **Step 4: Expand the rigid-body vocabulary from table-only to paragraph-plus-table**

Keep a compact table, but precede it with short explanations of why each word matters. Cover exactly these terms:

```md
`articulation`
generalized coordinates
mass matrix `M`
Jacobian `J`
ABA
CRBA
```

For each term, explain it by answering “what problem does this concept solve?” rather than giving a formal derivation.

- [ ] **Step 5: Expand the GPU/Warp intuition section around reader confusion points**

Write short paragraphs that translate these concepts into first-pass intuition:

```md
- 从 CPU for-loop 到 kernel：不是“换个语法”，而是“让很多元素做同一段逻辑”
- `wp.array`：不是普通 Python 列表，而是给 kernel 读写的数据容器
- `wp.launch`：把这段批量逻辑按一定规模发出去
- `wp.tid()`：当前线程 / 当前元素索引
- `wp.atomic_add`：多个线程可能同时写同一位置时的保护手段
- `wp.tile`：为了局部复用和并行局部性
- `Graph`：把重复执行的调度关系打包起来
```

Keep `tile` and `Graph` intentionally shallow. The goal is to remove fear, not to teach optimization deeply.

- [ ] **Step 6: End with a navigation section that sends the reader to `01` or `02` based on confusion type**

Write a section that maps likely reader blocks to the next chapter:

```md
- “我不懂 kernel / array / atomic 在说什么” -> 先去 `01_warp_basics`
- “我知道这些词，但不知道 Newton 自己怎么组织” -> 先去 `02_newton_arch`
- “我只是偶尔忘词” -> 先把本章当回查页，不要停太久
```

- [ ] **Step 7: Verify the rewritten chapter is still intentionally bounded**

Read the full file and delete any passage that turns into:

```md
- 机器人动力学系统教程
- CUDA/Warp API 全量导读
- 提前展开 `03+` 章节内容
```

Expected result: `00` feels like a readable prerequisite mini-lesson, not a detached glossary or a full textbook.

---

### Task 2: Turn `00` README and `01` chapter entrypoints into a continuous handoff

**Files:**
- Modify: `chapters/00_prerequisites/README.md`
- Modify: `chapters/01_warp_basics/README.md`
- Create: `chapters/01_warp_basics/principle.md`
- Reference: `chapters/00_prerequisites/principle.md`, `chapters/02_newton_arch/README.md`, `templates/principle.md.tmpl`

- [ ] **Step 1: Rewrite `00` README as the entrance to the first three chapters**

Replace the current wording so the file clearly says:

```md
- 本章不是把所有前置知识讲完
- 本章的工作是把你安全送到 `01` 和 `02`
- 如果你已经能读懂这些前置词，可以快速通过，不必久留
```

Keep the frontmatter. Rewrite the completion criteria, goals, and reading order to sound actionable rather than abstract.

- [ ] **Step 2: Rewrite `01` README from template to real chapter entry**

Remove the placeholder questions and fill these sections with concrete content:

```md
## 本章目标
- 为什么 Newton 学习者需要先补 Warp 的最小执行模型
- 这一章学到什么程度就足以进入 `02`

## 前置依赖
- 只要求 `00` 的最小前置
- 不要求完整 CUDA 背景

## GAMES103 已有 vs 本章新增
- 明确 GAMES103 对 GPU/Warp 的空白
- 明确本章新增的是执行模型而不是通用 GPU 工程大全

## 预期产出
- 读者能稳定解释 `for-loop`、`kernel`、`array`、`launch`、`tid`、`atomic`
```

- [ ] **Step 3: Create `01_warp_basics/principle.md` with the same frontmatter style as neighboring chapters**

Use the existing chapter style and include:

```md
---
chapter: 01
title: Warp 编程模型
last_updated: 2026-04-18
source_paths: []
paper_keys:
  - warp-paper
  - mujoco-warp-paper
newton_commit: 1a230702
---
```

- [ ] **Step 4: Write the body of `01` as an approachable Warp mental-model chapter**

Use this section order:

```md
## 0. 为什么在 Newton 之前先讲 Warp
## 1. 从 CPU for-loop 到 kernel
## 2. `wp.array` 在心智模型里是什么
## 3. `wp.launch`、`wp.tid()` 和批量执行
## 4. 为什么会遇到 `wp.atomic_add`
## 5. `wp.tile` 与局部复用的第一印象
## 6. `Graph` 解决的是什么问题
## 7. 带着这些概念进入 Newton 时，先看什么，不必急着看什么
```

Each section must answer both:

```md
- 这个概念本身是什么意思？
- Newton 学习者为什么现在就要懂它？
```

- [ ] **Step 5: Add an ending bridge from `01` into `02`**

End `01` with a short handoff paragraph like:

```md
读完这一章后，你还不需要会写复杂 Warp kernel，也不需要理解所有性能优化细节。
你只需要带着一个稳定直觉进入 `02`：Newton 的很多对象和求解器之所以长成现在这样，是因为它们最终要落到 Warp 的批量执行模型上。
```

- [ ] **Step 6: Verify `00` and `01` do not duplicate each other excessively**

Read `chapters/00_prerequisites/README.md`, `chapters/00_prerequisites/principle.md`, `chapters/01_warp_basics/README.md`, and `chapters/01_warp_basics/principle.md` together.

Expected result:

```md
- `00` gives prerequisite intuition and navigation
- `01` gives the real Warp chapter
- The same explanation is not pasted twice in full
```

---

### Task 3: Revise `02_newton_arch` so it clearly receives the reader from `00/01`

**Files:**
- Modify: `chapters/02_newton_arch/README.md`
- Modify: `chapters/02_newton_arch/principle.md`
- Reference: `chapters/00_prerequisites/principle.md`, `chapters/01_warp_basics/principle.md`

- [ ] **Step 1: Rewrite the dependency and reading-order language in `02` README**

Adjust the README so it explicitly says:

```md
- `00` 解决的是“读不懂这些前置词怎么办”
- `01` 解决的是“Warp 批量执行模型到底是什么”
- `02` 从这里开始真正进入 Newton 本体
```

Keep the current core goals: `basic_pendulum`, four-layer model, 8-solver overview.

- [ ] **Step 2: Strengthen the opening of `02` principle to reference the first two chapters**

Rewrite the first section so it feels like the next step in a staircase, not a separate note. It should establish:

```md
如果 `00` 解决了前置词汇焦虑，`01` 解决了 Warp 执行模型焦虑，那么这一章开始回答另一个问题：Newton 自己到底是怎样把例子入口、公共对象和 solver 家族组织起来的？
```

- [ ] **Step 3: Tighten the “example -> four layers” explanation**

Keep `basic_pendulum` as the quick-win example, but revise the explanatory flow so the reader can naturally follow:

```md
examples 入口 -> 公共 API / 基础对象 -> `Model / State / Control / Solver` -> 一次 step
```

Use short bridge sentences between bullets so the prose reads like a guided walkthrough instead of isolated definitions.

- [ ] **Step 4: Keep the solver overview broad, not deep**

Retain the 8-solver map and table, but make sure each row stays at the “what role does this solver family play in the map?” level.

Delete or shorten any sentence that drifts into details better reserved for:

```md
`05_rigid_articulation`
`07_constraints_contacts_math`
`08_rigid_solvers`
`09_variational_solvers`
`11_mpm`
```

- [ ] **Step 5: Add a closing paragraph that points forward cleanly**

End the file with a short paragraph that tells the reader what comes next after this architectural overview:

```md
接下来如果你想继续沿主线往下走，先去补 `03` 和 `04` 的数学/几何与模型构建基础；如果你更想顺着刚体主线深入，就带着这一章的对象地图进入 `05` 和 `08`。
```

- [ ] **Step 6: Verify the full `00 -> 01 -> 02` staircase reads smoothly**

Read the six target files in order and check for these outcomes:

```md
- `00` is more external and prerequisite-oriented
- `01` is more execution-model-oriented
- `02` is clearly Newton-oriented
- terminology and tone stay consistent
- later chapters are referenced, not pre-empted
```

If any section feels redundant or out of order, tighten it before stopping.

---

### Task 4: Run the internal review and verification loop on the upgraded docs

**Files:**
- Review: `chapters/00_prerequisites/README.md`
- Review: `chapters/00_prerequisites/principle.md`
- Review: `chapters/01_warp_basics/README.md`
- Review: `chapters/01_warp_basics/principle.md`
- Review: `chapters/02_newton_arch/README.md`
- Review: `chapters/02_newton_arch/principle.md`

- [ ] **Step 1: Run a structure scan for unfinished template language**

Search the six files for remnants like:

```text
本章要回答什么问题？
读完后应该能独立讲清哪些对象、流程和边界？
本章最终应有哪些文件：
未填写的占位词
空白模板段落
```

Expected result: no leftover template placeholders remain in chapters `00`-`02`.

- [ ] **Step 2: Run a consistency scan for chronology and terminology**

Check these conventions across all six files:

```md
- `00` = prerequisite chapter outside Newton proper
- `01` = Warp execution model chapter
- `02` = first Newton architecture chapter
- `Model / State / Control / Solver` wording is consistent
- `kernel / array / launch / tid / atomic` wording is consistent
```

- [ ] **Step 3: Run a readability scan focused on the user's request**

Review each section and cut or rewrite any paragraph that is:

```md
- too dense for a first pass
- too much like a glossary
- too much like API documentation
- too deep for chapters `00`-`02`
```

Expected result: the first three chapters feel more approachable, more ordered, and more beginner-safe.

- [ ] **Step 4: Produce a short implementation summary for the final handoff**

Summarize the result in three bullets:

```md
- what changed in `00`
- what changed in `01`
- what changed in `02`
```

Also note any intentional gaps that remain for later work, such as `source-walkthrough.md`, `examples.md`, and execution evidence.
