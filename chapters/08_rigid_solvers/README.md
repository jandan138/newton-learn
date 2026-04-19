---
chapter: 08
title: 08 刚体求解器家族
last_updated: 2026-04-19
source_paths: []
paper_keys:
  - mujoco-warp-paper
  - kamino-report
  - featherstone-book
newton_commit: 1a230702
---

# 08 刚体求解器家族

`07_constraints_contacts_math` 已经把 `Contacts -> rows -> Jacobian -> Delassus` 讲顺了。第 08 章接着回答的不是“还有哪些 solver”，而是另一件更关键的事: **这些对象准备好以后，不同 rigid solver 到底怎样消费它们？为什么大家都长得像 `solver.step(state_in, state_out, control, contacts, dt)`，内部路线却完全不同？**

所以这章不是 flat solver catalog。它先建立 shared public contract，再用四条有代表性的 rigid solver 路线做对照:

- `SolverSemiImplicit`: 力累加 + 积分 baseline。
- `SolverFeatherstone`: Newton 内部的 generalized-coordinate articulation 路线。
- `SolverMuJoCo`: 导出到 MuJoCo / `mujoco_warp` 的 external backend bridge。
- `SolverKamino`: Newton 内部的 constrained dynamics solver 路线，也是 chapter 07 contact math 的直接 consumer。

## 文件分工

- `README.md`: 本章边界、完成门槛、阅读入口。
- `principle.md`: 先讲 shared solver contract，再讲 maximal / generalized coordinates 的大分野，以及四条 solver 路线分别怎样处理约束。
- `source-walkthrough.md`: 按 `public contract -> three contrastive paths -> Kamino continuation` 的顺序走源码，不把读者一上来就扔进细节里。
- `examples.md`: 用 `example_robot_cartpole.py` 做跨 solver 对照，再用 Kamino 的 `example_sim_basics_box_on_plane.py` 把 chapter 07 的 contact math 真正落到 solver step。

## 完成门槛

```text
[ ] 我能先说清所有 solver 共享的 `step(state_in, state_out, control, contacts, dt)` contract
[ ] 我能解释 maximal coordinates 和 generalized coordinates 的第一层差别
[ ] 我能说明为什么 `SemiImplicit` 不是 row solver、`Featherstone` 是 joint-space 路线、`MuJoCo` 是 backend bridge、`Kamino` 是 chapter 07 的直接下一跳
[ ] 我能用 `cartpole` 和 `box_on_plane` 两个例子分别观察“跨 solver 对照”和“contact math 落地”
```

## 本章目标

- 把“同一个 public solver contract”先讲清，再比较内部数学分野，而不是先背 solver 名字。
- 让你先会用两条轴来比较 solver: `主状态表示是什么`，`约束 / contacts 在哪一层被处理`。
- 让 chapter 07 留下来的 rows / Jacobians / Delassus 主线，在 Kamino 这条 solver 路线上真正闭环。

## 本章范围

- 只聚焦 Newton 里四条 rigid-relevant solver 路线: `SemiImplicit`、`Featherstone`、`MuJoCo`、`Kamino`。
- 重点比较 shared contract、状态表示、contact handoff 和约束处理层级。
- 重点例子只有两个: `newton/examples/robot/example_robot_cartpole.py` 和 `newton/_src/solvers/kamino/examples/sim/example_sim_basics_box_on_plane.py`。

## 本章明确不做什么

- 不重讲 chapter 07 的 `Contacts -> rows -> Jacobian -> Delassus` 数学。
- 不做完整 CRBA / ABA / PADMM / MuJoCo solver 理论课。
- 不把 `newton/solvers.py` 里的 feature matrix 重写成一张更长的功能表。
- 不回答“哪个 solver 最好”或给调参秘籍。

## 前置依赖

- 建议先读完 `07_constraints_contacts_math`。如果你还不能解释 `Contacts -> rows -> Jacobian -> Delassus`，先回看 chapter 07。
- 建议 chapter 05 的 articulation 结构还在脑子里，因为这章会频繁对照 `body_q / body_qd` 和 `joint_q / joint_qd`。
- 这章默认你已经接受 `Contacts` 只是 collision handoff，不是所有 solver 的最终内部格式。

## 教学型对照表

| Solver / 家族 | 先怎么想 | 它最先消费哪层对象 | 为什么现在要懂 |
|---------------|-----------|--------------------|----------------|
| `SemiImplicit` | 力累加 + 积分 baseline | body state、contact force kernels | 帮你看懂“不是所有 solver 都直接解 rows” |
| `Featherstone` | generalized-coordinate articulation solver | `joint_q / joint_qd`、articulation mass matrix、joint-space solve | 帮你看懂 reduced coordinates 路线 |
| `MuJoCo` | external backend bridge | Newton model/state 到 MuJoCo backend 的桥接，contacts 可由 backend 或 bridge 提供 | 帮你看懂为什么有些 solver 是“导出再求解” |
| `Kamino` | Newton 内部的 constrained dynamics continuation | `Contacts -> ContactsKamino -> rows / Jacobians / Delassus / NCP solve` | 它是 chapter 07 的直接下一跳 |

## 阅读顺序

1. 先读 `principle.md`，把 shared contract 和 solver family split 建起来。
2. 再读 `source-walkthrough.md`，确认同样的 `step(...)` 在源码里怎样分叉成三条对照路径和一条 chapter 07 continuation path。
3. 然后读 `examples.md`，先用 `cartpole` 把“同一 public scene 可切换 solver”看顺，再用 `box_on_plane` 把 chapter 07 的 contact math 接进 Kamino solver。
4. 如果你读完还会把四个 solver 想成“只是名字不同”，就回到 `principle.md` 里的两条比较轴重新看一遍。

## 预期产出

- `principle.md`: 一条 beginner-safe 主线，先稳住 shared contract，再说明四条 rigid solver 路线为什么不同。
- `source-walkthrough.md`: 一份 tiered read path，告诉你先看 public contract，再看对照路径，最后看 Kamino 的 chapter 07 continuation。
- `examples.md`: 两个观察任务，分别守住“外层调用一致”与“contact math 真正进入 solver”这两层理解。
