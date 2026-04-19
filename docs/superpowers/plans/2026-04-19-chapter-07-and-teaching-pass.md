# Chapter 07 And Teaching Pass Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Complete `07_constraints_contacts_math` as a teaching-first bridge chapter, then run a minimal high-leverage teaching pass on chapters `03-05`.

**Architecture:** Work in the clean worktree `/.worktrees/ch07-teaching-pass` on top of `main`. Reuse chapter 06's concrete-first style: start chapter 07 from the same contact picture the reader already knows, build from `Contacts` into rows / Jacobian / Delassus, then tighten only the weakest opening sections in chapters 03-05 so the earlier spine better matches chapter 06/07.

**Tech Stack:** Markdown docs, git worktree, Newton source references, `git diff --check`, ripgrep-based placeholder scans.

---

## File Map

- Modify: `chapters/07_constraints_contacts_math/README.md`
  - Replace the template README with a teaching-first chapter entry, explicit `06 -> 07 -> 08` bridge, and a concrete prerequisite self-check.
- Create: `chapters/07_constraints_contacts_math/principle.md`
  - Teach `Contacts -> rows -> Jacobian -> Delassus` with `sphere-ground` first and `box-ground` second.
- Create: `chapters/07_constraints_contacts_math/source-walkthrough.md`
  - Add a tiered first-read path from contact geometry handoff into Kamino constraint rows and Delassus.
- Create: `chapters/07_constraints_contacts_math/examples.md`
  - Turn one or two existing Newton examples into observation tasks that reinforce the principle page.
- Modify: `chapters/03_math_geometry/principle.md`
  - Soften the opening and the shape-representation section with more concrete pictures and a clearer forward pointer to chapter 06.
- Modify: `chapters/04_scene_usd/README.md`
  - Convert the comparison table into a more teaching-oriented table.
- Modify: `chapters/04_scene_usd/principle.md`
  - Start the importer chain from a concrete toy scene, not a five-step abstract chain.
- Modify: `chapters/05_rigid_articulation/principle.md`
  - Add a concrete articulation picture before array names and make the joint-vs-body state distinction misconception-first.
- Create: `docs/superpowers/specs/2026-04-19-chapter-07-deepening-design.md`
  - Keep the refined design in the new worktree for traceability.
- Create: `docs/superpowers/plans/2026-04-19-chapter-07-and-teaching-pass.md`
  - This implementation plan.

## Task 1: Replace Chapter 07 README

**Files:**
- Modify: `chapters/07_constraints_contacts_math/README.md`

- [ ] **Step 1: Read the current template README and chapter 06 README side by side**

Run: `git diff --no-index -- chapters/06_collision/README.md chapters/07_constraints_contacts_math/README.md`

Expected: the chapter 07 file still looks template-like, while chapter 06 has a teaching-first bridge.

- [ ] **Step 2: Replace the template with a teaching-first README skeleton**

```md
# 07 约束与接触数学总论

`06_collision` 已经把球和地面的接触写进 `Contacts`。第 07 章接着回答：这些接触点、法线和距离，怎样继续长成约束行、Jacobian 和 Delassus，成为后续 solver 真正会吃的数学对象。

## 文件分工
- `README.md`：本章边界、完成门槛、入口。
- `principle.md`：从 `sphere-ground` 起手，把 `Contacts -> rows -> Jacobian -> Delassus` 讲顺。
- `source-walkthrough.md`：把 chapter 06 的 handoff object 钉到 Kamino 里的 constraint/Jacobian/Delassus 路径。
- `examples.md`：用 `sphere-ground` 和 `box-ground` 做观察任务。

## 完成门槛
```text
[ ] 我能把 `Contacts` 里的接触点、法线、距离翻译成“这个接触想阻止什么相对运动”
[ ] 我能解释为什么一条接触通常会长成 1 条法向 row + 2 条切向 row
[ ] 我能用人话描述 Jacobian 在这里做的是“哪些速度会影响接触方向”
[ ] 我能把 Delassus / effective mass 先理解成“沿这个接触方向系统有多难被推着动”
```
```

- [ ] **Step 3: Add the concrete prerequisite self-check and explicit chapter boundary**

```md
## 前置依赖

- 建议先读完 `06_collision`；如果你还不能解释 `rigid_contact_normal`、`rigid_contact_point0 / point1` 在球贴地场景里分别代表什么，先回看 chapter 06。
- 这章不讲 solver 怎样解约束；那是 `08_rigid_solvers` 的任务。
```

- [ ] **Step 4: Run a whitespace / conflict check on the README only**

Run: `git diff --check -- chapters/07_constraints_contacts_math/README.md`

Expected: no output.

## Task 2: Write Chapter 07 Principle

**Files:**
- Create: `chapters/07_constraints_contacts_math/principle.md`

- [ ] **Step 1: Add front matter consistent with earlier chapters**

```md
---
chapter: 07
title: 约束与接触数学总论
last_updated: 2026-04-19
source_paths:
  - docs/concepts/collisions.rst
  - newton/_src/geometry/contact_data.py
  - newton/_src/sim/collide.py
  - newton/_src/sim/contacts.py
  - newton/_src/solvers/kamino/_src/geometry/contacts.py
  - newton/_src/solvers/kamino/_src/kinematics/constraints.py
  - newton/_src/solvers/kamino/_src/kinematics/jacobians.py
  - newton/_src/solvers/kamino/_src/dynamics/delassus.py
paper_keys: []
newton_commit: 1a230702
---
```

- [ ] **Step 2: Write the bridge and `sphere-ground` opening before any formulas**

```md
## 0. 先把 chapter 06 的球贴地画面带过来

chapter 06 里，你关心的是“接触点在哪、法线朝哪、还隔多少”。
chapter 07 换的问题是：既然球已经贴到地面，这次接触到底希望系统阻止哪种相对运动？

先别想矩阵。第一遍先想一件朴素的事：如果球还在沿法线方向继续往地里钻，系统就得想办法阻止这件事。
```

- [ ] **Step 3: Add the row-growth section before Jacobian terminology**

```md
## 1. 为什么一条接触会长成三条 row

- 法向 row：先阻止继续互相压进。
- 两条切向 row：给摩擦或切向限制预留方向。

所以“一条 contact”在这里不是一个标量，而是一组接触局部坐标轴下的约束方向。
```

- [ ] **Step 4: Add a teaching table for contact frame / Jacobian / Delassus**

```md
| 词 / 对象 | 先怎么想 | 球贴地时在干嘛 | 在源码里第一次看哪里 |
|-----------|-----------|----------------|-------------------------|
| contact frame | 接触点自己的小坐标系 | `z` 轴沿法线，`x/y` 轴沿切向 | `geometry/contacts.py` |
| Jacobian row | 哪些速度会改变这个方向上的接触状态 | 球往上/往下、切着滑、绕偏心点转都会被映射进来 | `kinematics/jacobians.py` |
| Delassus | 沿这些 row 推系统时有多难动 | 质量大、惯量大或杠杆臂大时，响应会不同 | `dynamics/delassus.py` |
```

- [ ] **Step 5: Add the second example and chapter handoff**

```md
## 5. 再看 `box-ground`

球最适合建立“单接触 -> 三条 row”的直觉。箱子贴地则提醒你两件更难的事：
- 一个 shape pair 可能长出多个 contact。
- 接触点离 COM 有偏移时，Jacobian 会把平移和转动一起带进来。

## 6. 这一章到这里停，下一步去 `08`
```

- [ ] **Step 6: Run a file-level sanity check**

Run: `git diff --check -- chapters/07_constraints_contacts_math/principle.md`

Expected: no output.

## Task 3: Write Chapter 07 Source Walkthrough

**Files:**
- Create: `chapters/07_constraints_contacts_math/source-walkthrough.md`

- [ ] **Step 1: Start with a "what question to carry" intro and tiered read path**

```md
# 07 约束与接触数学总论 源码走读

这份 walkthrough 只追一条线：`Contacts -> solver-facing contact -> constraint rows -> Jacobian -> Delassus`。

## 先带着哪四个问题读
1. chapter 06 写进 `Contacts` 的几何，到 chapter 07 还剩下哪些信息？
2. 一个 contact 为什么会变成 3 条 row？
3. Jacobian 到底是在把哪种运动映射到接触方向？
4. Delassus 为什么可以先理解成“沿这个方向有多难推”？
```

- [ ] **Step 2: Build Tier 1 from chapter 06 handoff files**

```md
## Tier 1: 先确认 handoff object 里到底有什么

| 路径 | 角色 | 先读原因 |
|------|------|----------|
| `docs/concepts/collisions.rst` | 接触字段的公开语义 | 先把 normal / point / distance / margin 的人话读顺 |
| `newton/_src/geometry/contact_data.py` | narrow phase 中间 contact 语义 | 看 contact geometry 最初长什么样 |
| `newton/_src/sim/collide.py` | `write_contact()` 把几何写进 `Contacts` | 看 chapter 06 的真正 handoff 边界 |
| `newton/_src/sim/contacts.py` | runtime `Contacts` 布局 | 认清哪些字段后来会被继续消费 |
```

- [ ] **Step 3: Build Tier 2 around `ContactsKamino`, row packing, and Jacobians**

```md
## Tier 2: contact 怎样长成 rows 和 Jacobian

| 路径 | 角色 | 先读原因 |
|------|------|----------|
| `.../geometry/contacts.py` | `Contacts` 到 solver-facing contact 的桥 | 解释为何 chapter 07 仍从 `Contacts` 出发 |
| `.../kinematics/constraints.py` | 每个 contact 分配多少 row | 把“1 contact -> 3 rows”钉到代码 |
| `.../kinematics/jacobians.py` | 构造 contact Jacobians | 看接触 frame 和杠杆臂怎样进入 row |
| `.../core/math.py` | contact wrench / lever arm 辅助数学 | 把角向部分的来源讲人话 |
```

- [ ] **Step 4: Build Tier 3 around Delassus and close the dataflow**

```md
## Tier 3: 最后看 Delassus

| 路径 | 角色 | 先读原因 |
|------|------|----------|
| `.../dynamics/delassus.py` | `D = J M^{-1} J^T` 及其对角/块结构 | 只保住第一层物理意义 |
```

- [ ] **Step 5: Add a dataflow table ending at `D = J M^{-1} J^T` and verify formatting**

Run: `git diff --check -- chapters/07_constraints_contacts_math/source-walkthrough.md`

Expected: no output.

## Task 4: Write Chapter 07 Examples

**Files:**
- Create: `chapters/07_constraints_contacts_math/examples.md`

- [ ] **Step 1: Reuse the same `sphere-ground` mental model from chapter 06**

```md
## 主例子：球贴地

- 先验证什么：`1 contact -> 3 rows`
- 改哪里最值钱：高度、半径、初始水平速度
- 最值得观察：contact 是否出现、法线是否稳定、切向滑动是否让你更容易理解两条 tangent row 的存在
```

- [ ] **Step 2: Add the `box-ground` contrast case**

```md
## 对照例子：箱子贴地

- 先验证什么：一个 pair 可能长出多个 contact
- 改哪里最值钱：轻微倾斜、宽度、COM 偏移
- 最值得观察：接触数变化、接触点是否远离 COM、为什么这会让角向部分更重要
```

- [ ] **Step 3: Add explicit misconception checks**

```md
如果你把“contact 数”直接等同于“shape pair 数”，或把“Jacobian”直接等同于“接触力”，这页就该回到 `principle.md` 对照着重读。
```

- [ ] **Step 4: Run a markdown sanity check**

Run: `git diff --check -- chapters/07_constraints_contacts_math/examples.md`

Expected: no output.

## Task 5: Run The Small Teaching Pass On Chapters 03-05

**Files:**
- Modify: `chapters/03_math_geometry/principle.md`
- Modify: `chapters/04_scene_usd/README.md`
- Modify: `chapters/04_scene_usd/principle.md`
- Modify: `chapters/05_rigid_articulation/principle.md`

- [ ] **Step 1: Soften chapter 03's opening and shape-representation entry**

```md
## 0. 为什么 `02` 之后还会卡住

先想一个最简单的画面：你刚打开 builder 或 `Model`，看到一堆 `shape_transform`、`joint_X_p`、`body_com`，像是在看一张没有图例的工程图。
chapter 03 的任务，就是先把这张图的图例补上。
```

And in `## 5`, start from a concrete contrast such as “球和箱子是参数化形状，地形和 mesh 则更像一份几何数据”。

- [ ] **Step 2: Make chapter 04 start from a toy scene, not a five-step abstract chain**

```md
先想一个最小 scene：一个地面、一个 box、一个 revolute joint。读者最先该问的不是 schema resolver 叫什么，而是“这些 authored prim 最后怎样进到 `Model` 的 body / joint / shape 数组里”。
```

Also rewrite the README comparison table into a teaching table like:

```md
| 词 / 对象 | 先怎么想 | 一个最小 scene 里它像什么 | 第一次去哪里看 |
```

- [ ] **Step 3: Make chapter 05 more misconception-first**

```md
## 1. 先看一条最小两连杆

先别想 `articulation_start`。先想两个 body 用一个关节连起来：运行时既要知道这是一条树，也要知道每个关节参数最后怎样长成 body 的世界位姿。
```

And in `## 2`, open with: “最常见的误会，是把 `joint_qd` 直接当成 `body_qd`。” Put the table after that sentence and after one revolute-joint example.

- [ ] **Step 4: Check only the touched legacy files for whitespace issues**

Run: `git diff --check -- chapters/03_math_geometry/principle.md chapters/04_scene_usd/README.md chapters/04_scene_usd/principle.md chapters/05_rigid_articulation/principle.md`

Expected: no output.

## Task 6: Final Verification

**Files:**
- Verify all modified chapter files

- [ ] **Step 1: Run repository diff check for all current changes**

Run: `git diff --check`

Expected: no output.

- [ ] **Step 2: Scan for leftover template markers in chapter 07 and touched chapter 03-05 files**

Run: `rg -n "本章要回答什么问题|读完后应该能独立讲清哪些对象|本章最终应有哪些文件|二选一先开写" chapters/07_constraints_contacts_math chapters/03_math_geometry chapters/04_scene_usd chapters/05_rigid_articulation`

Expected: no matches in the files touched by this plan.

- [ ] **Step 3: Review the final diff before any optional git action**

Run: `git diff -- chapters/07_constraints_contacts_math chapters/03_math_geometry chapters/04_scene_usd chapters/05_rigid_articulation docs/superpowers/specs docs/superpowers/plans`

Expected: only the planned documentation changes appear.

- [ ] **Step 4: Stop before commit unless the user explicitly asks for one**

No commit in this plan by default. If the user later requests a commit, use the repo's recent docs commit style and only include the planned files.
