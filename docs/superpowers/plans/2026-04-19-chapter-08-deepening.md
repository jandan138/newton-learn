# Chapter 08 Deepening Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn `08_rigid_solvers` from a template into a teaching-first chapter that explains how the major rigid solver families in Newton consume shared state, contacts, and constraint math differently.

**Architecture:** Reuse chapter 07's bridge by opening from the same solver-facing question, but avoid writing a flat solver catalog. Build chapter 08 around a shared solver contract, then compare four rigid-relevant paths: SemiImplicit, Featherstone, MuJoCo, and Kamino, with Kamino as the direct continuation of the chapter 07 contact-math spine.

**Tech Stack:** Markdown docs, git worktree, Newton source references, `git diff --check`, ripgrep placeholder scans.

---

## File Map

- Modify: `chapters/08_rigid_solvers/README.md`
  - Replace template content with a teaching-first chapter entry, explicit bridge from chapter 07, scope fence, and reading order.
- Create: `chapters/08_rigid_solvers/principle.md`
  - Explain the shared solver contract and the family split across SemiImplicit, Featherstone, MuJoCo, and Kamino.
- Create: `chapters/08_rigid_solvers/source-walkthrough.md`
  - Provide a tiered read path from public solver contract into solver-specific implementations.
- Create: `chapters/08_rigid_solvers/examples.md`
  - Use `example_robot_cartpole.py` and a Kamino contact-first example as observation tasks.
- Create: `docs/superpowers/specs/2026-04-19-chapter-08-deepening-design.md`
  - Keep the finalized design in the worktree for traceability.
- Create: `docs/superpowers/plans/2026-04-19-chapter-08-deepening.md`
  - This implementation plan.

## Task 1: Replace Chapter 08 README

**Files:**
- Modify: `chapters/08_rigid_solvers/README.md`

- [ ] **Step 1: Read the current template and the chapter 07 README back-to-back**

Run: `git diff --no-index -- chapters/07_constraints_contacts_math/README.md chapters/08_rigid_solvers/README.md`

Expected: chapter 08 still looks template-like and needs a real bridge, scope, and reading path.

- [ ] **Step 2: Replace the template with a teaching-first chapter entry**

```md
# 08 刚体求解器家族

`07_constraints_contacts_math` 已经把 `Contacts -> rows -> Jacobian -> Delassus` 讲顺了。第 08 章接着回答：这些对象准备好以后，不同 rigid solver 到底怎样消费它们？

## 文件分工
- `README.md`：本章边界、完成门槛、入口。
- `principle.md`：shared contract 先行，再解释四条 solver 路线怎样不同。
- `source-walkthrough.md`：从 public solver contract 走到 SemiImplicit / Featherstone / MuJoCo / Kamino 的关键源码边界。
- `examples.md`：用一个跨 solver 例子和一个 Kamino contact-first 例子把差异变成可观察现象。
```

- [ ] **Step 3: Add a concrete prerequisite self-check and clear non-goals**

```md
## 前置依赖

- 建议先读完 `07_constraints_contacts_math`。如果你还不能解释 `Contacts -> rows -> Jacobian -> Delassus`，先回看 chapter 07。
- 这章不做完整 CRBA / ABA / PADMM / MuJoCo solver 理论课，也不给“哪个 solver 最好”的调参建议。
```

- [ ] **Step 4: Add a teaching-first comparison table rather than a blank GAMES103 matrix**

```md
| Solver / 家族 | 先怎么想 | 它最先消费哪层对象 | 为什么现在要懂 |
|---------------|-----------|--------------------|----------------|
| SemiImplicit | 力累加 + 积分 baseline | body state、contact force kernels | 帮你看懂“不是所有 solver 都直接解 rows” |
| Featherstone | generalized-coordinate articulation solver | `joint_q / joint_qd`、mass matrix、joint-space solve | 帮你看懂 reduced coordinates 路线 |
| MuJoCo | external backend bridge | Newton -> mujoco_warp bridge | 帮你看懂为什么有些 solver 是“导出再求解” |
| Kamino | constrained dynamics continuation | rows / Jacobians / Delassus / NCP solve | 它是 chapter 07 的直接下一跳 |
```

- [ ] **Step 5: Verify the README formatting**

Run: `git diff --check -- chapters/08_rigid_solvers/README.md`

Expected: no output.

## Task 2: Write Chapter 08 Principle

**Files:**
- Create: `chapters/08_rigid_solvers/principle.md`

- [ ] **Step 1: Add front matter consistent with earlier chapters**

```md
---
chapter: 08
title: 刚体求解器家族
last_updated: 2026-04-19
source_paths:
  - newton/solvers.py
  - newton/_src/solvers/solver.py
  - newton/_src/solvers/semi_implicit/solver_semi_implicit.py
  - newton/_src/solvers/featherstone/solver_featherstone.py
  - newton/_src/solvers/mujoco/solver_mujoco.py
  - newton/_src/solvers/kamino/solver_kamino.py
  - newton/_src/solvers/kamino/_src/solver_kamino_impl.py
paper_keys:
  - mujoco-warp-paper
  - kamino-report
  - featherstone-book
newton_commit: 1a230702
---
```

- [ ] **Step 2: Open from chapter 07's handoff rather than from solver names**

```md
## 0. 先把 chapter 07 的问题往后推半步

chapter 07 已经把同一个接触讲成了 rows、Jacobian 和 Delassus。
chapter 08 接着问的不是“还有哪些 solver”，而是“这些对象准备好以后，solver 到底怎样接手？”
```

- [ ] **Step 3: Establish the shared public contract before any family split**

```md
## 1. public contract 看起来一样, 内部数学却完全不同

第一遍先记住这一句：
`solver.step(state_in, state_out, control, contacts, dt)` 看起来一样，不代表内部处理 rows / contacts / articulation 的方式也一样。
```

- [ ] **Step 4: Add the first major split: maximal vs generalized coordinates**

```md
## 2. 第一层大分野: maximal coordinates vs generalized coordinates

- SemiImplicit / Kamino 更接近 maximal coordinates 路线，直接围着 body state 和 body-level constraints 打转。
- Featherstone / MuJoCo 更接近 generalized coordinates 路线，更自然地围着 joint-space state 和 articulation solve 打转。
```

- [ ] **Step 5: Give each solver a teaching role, not equal catalog weight**

```md
## 3. `SemiImplicit`: baseline, but不是 row solver
## 4. `Featherstone`: articulation-native joint-space solve
## 5. `MuJoCo`: external backend bridge
## 6. `Kamino`: chapter 07 contact math 的直接 consumer
```

In those sections, explicitly teach:
- SemiImplicit = force kernels + integration
- Featherstone = generalized coordinates + mass/inertia solve inside Newton
- MuJoCo = bridge into mujoco_warp / MuJoCo-owned solve
- Kamino = explicit constrained dynamics path consuming rows / Jacobians / Delassus

- [ ] **Step 6: End with a compact teaching table and a handoff to examples/walkthrough**

```md
| Solver | 先怎么想 | 最先盯哪层输入 | chapter 07 和它的距离 |
|--------|-----------|----------------|----------------------|
| SemiImplicit | baseline | body/contact forces | 间接，只把 contact 当 force source |
| Featherstone | joint-space solve | `joint_q / joint_qd` | 部分相关，但不直接延续 row solve |
| MuJoCo | bridge | exported backend model/contact data | 更远，重在 bridge |
| Kamino | direct continuation | rows / Jacobians / Delassus | 最近，是 chapter 07 的主延续 |
```

- [ ] **Step 7: Verify principle formatting**

Run: `git diff --check -- chapters/08_rigid_solvers/principle.md`

Expected: no output.

## Task 3: Write Chapter 08 Source Walkthrough

**Files:**
- Create: `chapters/08_rigid_solvers/source-walkthrough.md`

- [ ] **Step 1: Start from a tiered first-read path**

```md
# 08 刚体求解器家族 源码走读

这份 walkthrough 只追一条线：
`public solver contract -> family split -> Kamino continuation path`

## 先带着哪四个问题读
1. 所有 solver 共享的 public contract 到底是什么？
2. 哪些 solver 直接围着 body/contact forces 打转，哪些围着 joint-space articulation 打转？
3. MuJoCo 为什么更像 bridge，而不是 Newton 内部完整 solver 展开？
4. chapter 07 的 rows / Jacobians / Delassus 在哪条 solver 路线上最直接继续出现？
```

- [ ] **Step 2: Build Tier 1 around the public entry and feature matrix**

```md
## Tier 1: 先看 shared contract

| 路径 | 角色 | 先读原因 |
|------|------|----------|
| `newton/solvers.py` | solver 全景和 feature matrix | 先建立“有哪些 solver 家族” |
| `newton/_src/solvers/__init__.py` | public exports | 看名字是怎样接到 public API 的 |
| `newton/_src/solvers/solver.py` | `SolverBase` 和 shared helper | 看看为什么 public 调用表面上很统一 |
```

- [ ] **Step 3: Build Tier 2 around the three contrastive paths**

```md
## Tier 2: 三条重要对照路径

| 路径 | 角色 | 先读原因 |
|------|------|----------|
| `semi_implicit/solver_semi_implicit.py` | maximal-coordinate baseline | 看 contact force kernels + integration |
| `featherstone/solver_featherstone.py` | generalized-coordinate articulation solver | 看 joint-space state 和 mass-matrix 路线 |
| `mujoco/solver_mujoco.py` | external backend bridge | 看 Newton 和 MuJoCo 之间的 handoff |
```

- [ ] **Step 4: Build Tier 3 around the Kamino continuation path**

```md
## Tier 3: chapter 07 的直接延续

| 路径 | 角色 | 先读原因 |
|------|------|----------|
| `kamino/solver_kamino.py` | Newton-facing Kamino wrapper | 看 public API 怎样接到 Kamino |
| `kamino/_src/solver_kamino_impl.py` | 真正的 solver 主路径 | 看 rows / Jacobians / Delassus 怎样进入 constrained dynamics solve |
```

- [ ] **Step 5: Add a final dataflow summary and verify formatting**

Run: `git diff --check -- chapters/08_rigid_solvers/source-walkthrough.md`

Expected: no output.

## Task 4: Write Chapter 08 Examples

**Files:**
- Create: `chapters/08_rigid_solvers/examples.md`

- [ ] **Step 1: Use `example_robot_cartpole.py` as the cross-solver comparison anchor**

```md
## 主例子: `newton/examples/robot/example_robot_cartpole.py`

- 先验证什么：同一个 scene / model，solver 可以切换，但内部状态表示和推进方式不同。
- 最值钱的改动：切换 `SolverMuJoCo` / `SolverSemiImplicit` / `SolverFeatherstone` 这几行。
- 最值得观察：哪些 solver 需要 `contacts`、哪些更围着 articulation state 打转。
```

- [ ] **Step 2: Use `example_sim_basics_box_on_plane.py` as the chapter-07-continuation anchor**

```md
## 对照例子: `kamino/examples/sim/example_sim_basics_box_on_plane.py`

- 先验证什么：chapter 07 的 contact math 怎样真正进入 solver。
- 最值钱的观察：box-on-plane 这组接触，不再只是 `Contacts -> rows`，而是真的进入 Kamino solver step。
```

- [ ] **Step 3: Add explicit misconception checks**

```md
如果你把所有 solver 都想成“只是不同实现的同一套 row solver”，或把 MuJoCo 也想成在 Newton 内部完整展开的 solver，你就该回到 `principle.md` 重读 family split 那一节。
```

- [ ] **Step 4: Verify examples formatting**

Run: `git diff --check -- chapters/08_rigid_solvers/examples.md`

Expected: no output.

## Task 5: Final Verification

**Files:**
- Verify all chapter 08 files and supporting spec/plan files

- [ ] **Step 1: Run repository diff check for current chapter 08 work**

Run: `git diff --check`

Expected: no output.

- [ ] **Step 2: Scan touched chapter 08 files for template leftovers**

Run: `rg -n "本章要回答什么问题|读完后应该能独立讲清哪些对象|本章最终应有哪些文件|二选一先开写" chapters/08_rigid_solvers`

Expected: no matches.

- [ ] **Step 3: Review the final diff before any optional git action**

Run: `git diff -- chapters/08_rigid_solvers docs/superpowers/specs/2026-04-19-chapter-08-deepening-design.md docs/superpowers/plans/2026-04-19-chapter-08-deepening.md`

Expected: only the planned documentation changes appear.

- [ ] **Step 4: Stop before commit unless the user explicitly asks for one**

No commit by default. If the user later requests one, use the repo's recent `docs:` commit style and commit only the planned files.
