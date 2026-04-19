# Chapter 09 Deepening Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn `09_variational_solvers` from a template into a teaching-first chapter that explains how XPBD, VBD, and Style3D organize implicit/variational correction differently while still addressing a shared class of cloth/softbody problems.

**Architecture:** Reuse chapter 08's shared-contract-first rhythm, but make chapter 09's main axis be “where does each iteration locally compute the correction?” Use `example_cloth_hanging.py` as the cross-solver anchor, introduce XPBD as the easiest entry point, then compare VBD's block solve and Style3D's global PD/PCG route, with `example_softbody_hanging.py` as a secondary VBD-only extension example.

**Tech Stack:** Markdown docs, git worktree, Newton source references, `git diff --check`, ripgrep placeholder scans.

---

## File Map

- Modify: `chapters/09_variational_solvers/README.md`
  - Replace the template with a teaching-first entry that frames chapter 09 around a shared variational problem and three update organizations.
- Create: `chapters/09_variational_solvers/principle.md`
  - Explain the shared implicit/variational problem, then compare XPBD, VBD, and Style3D in increasing solver sophistication.
- Create: `chapters/09_variational_solvers/source-walkthrough.md`
  - Build a tiered read path from shared surface and shared example into XPBD, VBD, and Style3D internals.
- Create: `chapters/09_variational_solvers/examples.md`
  - Use `example_cloth_hanging.py` for cross-solver comparison and `example_softbody_hanging.py` as the VBD extension example.
- Create: `docs/superpowers/specs/2026-04-19-chapter-09-deepening-design.md`
  - Keep the finalized design in the worktree for traceability.
- Create: `docs/superpowers/plans/2026-04-19-chapter-09-deepening.md`
  - This implementation plan.

## Task 1: Replace Chapter 09 README

**Files:**
- Modify: `chapters/09_variational_solvers/README.md`

- [ ] **Step 1: Read the current template and chapter 08 README side by side**

Run: `git diff --no-index -- chapters/08_rigid_solvers/README.md chapters/09_variational_solvers/README.md`

Expected: chapter 09 still looks template-like and needs a real bridge, scope, and reading path.

- [ ] **Step 2: Replace the template with a teaching-first chapter entry**

```md
# 09 变分求解器族

`08_rigid_solvers` 已经把 shared `solver.step(...)` contract 和 rigid solver family split 讲顺了。第 09 章接着回答：当 cloth / softbody 需要更稳定的隐式更新时，XPBD、VBD、Style3D 到底在解什么共同问题？

## 文件分工
- `README.md`：本章边界、完成门槛、入口。
- `principle.md`：先讲 shared variational problem，再比较三种局部更新层级。
- `source-walkthrough.md`：从 shared example 走到 XPBD / VBD / Style3D 的关键源码边界。
- `examples.md`：用 `cloth_hanging` 做跨 solver 对照，再用 `softbody_hanging` 解释 VBD 的延伸。
```

- [ ] **Step 3: Add a concrete prerequisite self-check and non-goals**

```md
## 前置依赖

- 建议先读完 `08_rigid_solvers`。如果你还不能解释为什么同一个 `step(...)` contract 后面会分叉成不同 solver 家族，先回看 chapter 08。
- 这章不做完整 XPBD / VBD / projective dynamics 理论课，也不做性能排行或 AVBD 刚体扩展深挖。
```

- [ ] **Step 4: Add a teaching-first comparison table**

```md
| Solver / 路线 | 先怎么想 | 每次迭代最先修哪层 | 为什么现在要懂 |
|---------------|-----------|--------------------|----------------|
| XPBD | constraint projection | 单条约束 | 最容易上手，帮你建立“隐式修正不一定先建全局矩阵” |
| VBD | local block solve | 单个 vertex / block | 帮你看懂 force + Hessian 的局部 Newton 视角 |
| Style3D | global cloth solve | 全局线性系统 | 帮你看懂 cloth-specialized PD/PCG 路线 |
```

- [ ] **Step 5: Verify README formatting**

Run: `git diff --check -- chapters/09_variational_solvers/README.md`

Expected: no output.

## Task 2: Write Chapter 09 Principle

**Files:**
- Create: `chapters/09_variational_solvers/principle.md`

- [ ] **Step 1: Add front matter consistent with earlier chapters**

```md
---
chapter: 09
title: 变分求解器族
last_updated: 2026-04-19
source_paths:
  - newton/solvers.py
  - newton/examples/cloth/example_cloth_hanging.py
  - newton/_src/solvers/xpbd/solver_xpbd.py
  - newton/_src/solvers/xpbd/kernels.py
  - newton/_src/solvers/vbd/solver_vbd.py
  - newton/_src/solvers/vbd/particle_vbd_kernels.py
  - newton/_src/solvers/style3d/solver_style3d.py
  - newton/_src/solvers/style3d/linear_solver.py
paper_keys:
  - xpbd-paper
  - vbd-paper
  - style3d-paper
newton_commit: 1a230702
---
```

- [ ] **Step 2: Open from a shared hanging-cloth picture, not solver names**

```md
## 0. 先把同一张 hanging cloth 的图带过来

第 09 章不先问“有哪三个 solver”，而先问：同样这块挂着的布，为什么三条 solver 路线都在试图做更稳定的隐式更新？
```

- [ ] **Step 3: Establish the shared problem before the family split**

```md
## 1. shared problem 先行：大家都在试图把这一步修正得更稳定

第一遍只要先接受：XPBD、VBD、Style3D 都不是在瞎调位置，它们都在处理同一类 implicit / variational correction 问题，只是每次迭代把修正组织在不同层级。
```

- [ ] **Step 4: Use XPBD as the easiest canonical entry**

```md
## 2. `XPBD`: 先从单条约束投影开始

- 先怎么想：一次看一条约束，按 compliance / lambda 更新位置修正。
- 为什么它适合当入口：最容易从“修哪条约束”建立直觉。
```

- [ ] **Step 5: Make VBD the richer local-Newton contrast**

```md
## 3. `VBD`: 从单条约束走到单个 vertex / block 的局部 Newton 步

- 先怎么想：不是一条约束一条约束地修，而是把和这个 vertex / block 相关的 force + Hessian 累起来再解。
- 为什么它不是“更复杂的 XPBD”：它换了局部子问题的组织方式。
```

- [ ] **Step 6: Make Style3D the global-solve endpoint**

```md
## 4. `Style3D`: cloth-specialized 的 global PD / PCG 路线

- 先怎么想：把局部更新进一步推成全局线性 solve。
- 为什么它不是“cloth-only VBD”：它最值钱的差别是固定 PD matrix 和 PCG 这条路。
```

- [ ] **Step 7: End with a compact ladder table and clear boundaries**

```md
| Solver | 先怎么想 | 每步局部更新落在哪 | 这条路最值钱的直觉 |
|--------|-----------|--------------------|------------------|
| XPBD | 约束投影 | constraint | 隐式修正不一定先建矩阵 |
| VBD | block Newton | vertex / block | force + Hessian 可以先局部累计 |
| Style3D | global PD | full linear system | 固定 Hessian 近似换来全局 cloth solve |
```

- [ ] **Step 8: Verify principle formatting**

Run: `git diff --check -- chapters/09_variational_solvers/principle.md`

Expected: no output.

## Task 3: Write Chapter 09 Source Walkthrough

**Files:**
- Create: `chapters/09_variational_solvers/source-walkthrough.md`

- [ ] **Step 1: Start from a tiered first-read path**

```md
# 09 变分求解器族 源码走读

这份 walkthrough 只追一条线：
`shared example -> XPBD projection -> VBD local block solve -> Style3D global solve`

## 先带着哪四个问题读
1. 三家到底共享了哪一层问题？
2. XPBD 为什么最像逐约束修正？
3. VBD 为什么更像局部 force + Hessian 的 block solve？
4. Style3D 为什么更像 cloth-specialized 的 global PD/PCG 路线？
```

- [ ] **Step 2: Build Tier 1 around the shared surface and shared example**

```md
## Tier 1: 先看 shared surface 和 shared example

| 路径 | 角色 | 先读原因 |
|------|------|----------|
| `newton/solvers.py` | solver family 全景 | 看 XPBD / VBD / Style3D 在全景里的位置 |
| `newton/examples/cloth/example_cloth_hanging.py` | 共享例子 | 一次性把三家摆进同一个 hanging-cloth 场景 |
```

- [ ] **Step 3: Build Tier 2 around XPBD and VBD**

```md
## Tier 2: 两条 local 路线

| 路径 | 角色 | 先读原因 |
|------|------|----------|
| `xpbd/solver_xpbd.py` | XPBD 主流程 | 看 predict / iterate / velocity recovery |
| `xpbd/kernels.py` | XPBD constraint projection 核心 | 看 compliance / lambda / projection 味道 |
| `vbd/solver_vbd.py` | VBD 主流程 | 看 coloring、iteration scaffold、block solve 外壳 |
| `vbd/particle_vbd_kernels.py` | VBD particle force + Hessian 核心 | 看为什么它不是“复杂版 XPBD” |
```

- [ ] **Step 4: Build Tier 3 around Style3D and VBD extension**

```md
## Tier 3: global solve 和 VBD beyond cloth

| 路径 | 角色 | 先读原因 |
|------|------|----------|
| `style3d/solver_style3d.py` | Style3D 主流程 | 看 nonlinear loop + fixed PD matrix workflow |
| `style3d/linear_solver.py` | PCG / sparse linear solve | 看 global solve 在哪里真正发生 |
| `examples/softbody/example_softbody_hanging.py` | VBD beyond cloth | 看 VBD 不只处理 cloth |
```

- [ ] **Step 5: Add a final ladder summary and verify formatting**

Run: `git diff --check -- chapters/09_variational_solvers/source-walkthrough.md`

Expected: no output.

## Task 4: Write Chapter 09 Examples

**Files:**
- Create: `chapters/09_variational_solvers/examples.md`

- [ ] **Step 1: Use `example_cloth_hanging.py` as the cross-solver anchor**

```md
## 主例子: `newton/examples/cloth/example_cloth_hanging.py`

- 先验证什么：同一张 hanging cloth 图，XPBD / VBD / Style3D 分别怎样组织修正。
- 最值钱的改动：切换 `--solver semi_implicit / xpbd / vbd / style3d`。
- 最值得观察：substeps、iterations、额外 builder/solver 要求的差异。
```

- [ ] **Step 2: Use `example_softbody_hanging.py` as the VBD extension anchor**

```md
## 对照例子: `newton/examples/softbody/example_softbody_hanging.py`

- 先验证什么：VBD 不只是 cloth solver。
- 最值钱的观察：tet softbody + damping + coloring 要求。
```

- [ ] **Step 3: Add explicit misconception checks**

```md
如果你把 VBD 只想成“更复杂的 XPBD”，或把 Style3D 只想成“只适合衣服的特例”，就该回到 `principle.md` 重读局部更新层级那张梯子表。
```

- [ ] **Step 4: Verify examples formatting**

Run: `git diff --check -- chapters/09_variational_solvers/examples.md`

Expected: no output.

## Task 5: Final Verification

**Files:**
- Verify all chapter 09 files and supporting spec/plan files

- [ ] **Step 1: Run repository diff check for current chapter 09 work**

Run: `git diff --check`

Expected: no output.

- [ ] **Step 2: Scan touched chapter 09 files for template leftovers**

Run: `rg -n "本章要回答什么问题|读完后应该能独立讲清哪些对象|本章最终应有哪些文件|二选一先开写" chapters/09_variational_solvers`

Expected: no matches.

- [ ] **Step 3: Review the final diff before any optional git action**

Run: `git diff -- chapters/09_variational_solvers docs/superpowers/specs/2026-04-19-chapter-09-deepening-design.md docs/superpowers/plans/2026-04-19-chapter-09-deepening.md`

Expected: only the planned documentation changes appear.

- [ ] **Step 4: Stop before commit unless the user explicitly asks for one**

No commit by default. If the user later requests one, use the repo's recent `docs:` commit style and commit only the planned files.
