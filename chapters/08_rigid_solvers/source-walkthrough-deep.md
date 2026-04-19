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
  - newton/examples/robot/example_robot_cartpole.py
  - newton/_src/solvers/kamino/examples/sim/example_sim_basics_box_on_plane.py
paper_keys:
  - mujoco-warp-paper
  - kamino-report
  - featherstone-book
newton_commit: 1a230702
---

# 08 刚体求解器家族 深读锚点版

这份 deep walkthrough 保留精确 cross-repo anchors、对照路径和 verification-heavy trace。第一次读 chapter 08 时，建议先把 `source-walkthrough.md` 读完，再回来用这一页钉住细节。

## Fast Deep Index

| ID | repo@commit path | symbol | why it matters |
|----|------------------|--------|----------------|
| D1 | `newton@1a230702 newton/solvers.py` | module doc, feature tables | shared solver contract 和 family split 的官方总览 |
| D2 | `newton@1a230702 newton/_src/solvers/solver.py` | `SolverBase.step`, `SolverBase.update_contacts` | 统一 public API 边界 |
| D3 | `newton@1a230702 newton/_src/solvers/semi_implicit/solver_semi_implicit.py` | `SolverSemiImplicit.step` | contacts as force kernels 的 baseline |
| D4 | `newton@1a230702 newton/_src/solvers/featherstone/solver_featherstone.py` | `SolverFeatherstone.step` | generalized-coordinate articulation 路线 |
| D5 | `newton@1a230702 newton/_src/solvers/mujoco/solver_mujoco.py` | `SolverMuJoCo.__init__`, `step`, `update_contacts` | external backend bridge 和 contact round-trip |
| D6 | `newton@1a230702 newton/_src/solvers/kamino/solver_kamino.py` | `SolverKamino.__init__`, `step` | Newton-facing Kamino wrapper |
| D7 | `newton@1a230702 newton/_src/solvers/kamino/_src/solver_kamino_impl.py` | `SolverKaminoImpl.__init__`, `_solve_forward_dynamics`, `_forward` | chapter 07 continuation 的内部 solve |
| D8 | `newton@1a230702 newton/examples/robot/example_robot_cartpole.py` | `Example.__init__` | shared contract + family switch 的最小例子 |
| D9 | `newton@1a230702 newton/_src/solvers/kamino/examples/sim/example_sim_basics_box_on_plane.py` | `control_callback`, `Example.__init__`, `step` | Kamino continuation 的最小 box-ground 例子 |

## Exact Handoff Trace

### 1. Shared public contract 先把 family 比较拉回同一个平面

- contract 总览：`newton/solvers.py:5-33`, `43-117`, `129-135`
- base API：`newton/_src/solvers/solver.py:177-316`, `347-356`
- 最小对照例子：`newton/examples/robot/example_robot_cartpole.py:54-71`

精确阅读点：

- `Model / State / Control / Contacts / dt -> solver.step() -> State`
- `SolverBase.step(...)` 的统一签名
- `SolverBase.update_contacts(...)` 说明 solver 也可能回写或转换 contact 数据
- `cartpole` 里并排切换 `SolverMuJoCo / SolverSemiImplicit / SolverFeatherstone`
- `contacts = None` 和 `eval_fk(...)` 暗示不同 family 的输入假设已经不同

### 2. `SemiImplicit`: contacts 进入 force accumulation

- 主路径：`newton/_src/solvers/semi_implicit/solver_semi_implicit.py:121-185`

关键 handoff：

- spring / triangle / bending / tet 力先累计
- `eval_body_joint_forces(...)` 把 joint 也先转成力
- `eval_body_contact_forces(...)`、`eval_particle_body_contact_forces(...)` 是 rigid/particle contact 的主要入口
- 最后统一 `integrate_particles(...)`、`integrate_bodies(...)`

这条路径最适合验证主 walkthrough 的一句话：`SemiImplicit` 真正想守住的是 `contacts -> forces -> integration`。

### 3. `Featherstone`: contact 只是 articulation solve 的一部分输入

- FK 和外力累积：`newton/_src/solvers/featherstone/solver_featherstone.py:404-585`
- mass matrix / solve / integrate：`newton/_src/solvers/featherstone/solver_featherstone.py:639-848`

关键 handoff：

- `eval_rigid_fk(...)` 先把 `joint_q / joint_qd` 对到 body pose
- `eval_body_contact(...)` 把 rigid contacts 转成 body force contribution
- `eval_rigid_jacobian(...)`、`eval_rigid_mass(...)` 生成 `J` 和 `M`
- `eval_dense_solve_batched(...)` 解出 `joint_qdd`
- `integrate_generalized_joints(...)` 再写回 generalized state

如果你想核对“Featherstone 是 joint-space 路线，而不是 row solver”，这一段是最直接证据。

### 4. `MuJoCo`: contact ownership 和状态推进都可以跨 backend

- constructor 和 contact ownership：`newton/_src/solvers/mujoco/solver_mujoco.py:2763-2995`
- main step：`newton/_src/solvers/mujoco/solver_mujoco.py:3003-3026`
- contact 回写：`newton/_src/solvers/mujoco/solver_mujoco.py:3681-3730`

关键 handoff：

- `use_mujoco_contacts=True` 时直接启用 backend collision detection
- `use_mujoco_contacts=False` 时，`step()` 里先 `_convert_contacts_to_mjwarp(...)`
- backend 前进一步之后，再 `_update_newton_state(...)`
- `update_contacts(...)` 可以把 MuJoCo contact 再写回 Newton `Contacts`

这也是 chapter 08 里最值得深挖的一条 bridge 路线：solver 核心可以主要留在 backend，不必在 Newton 仓里重写。

### 5. `Kamino`: chapter 07 objects 真正进入 solve

- wrapper 构造：`newton/_src/solvers/kamino/solver_kamino.py:403-434`
- wrapper step：`newton/_src/solvers/kamino/solver_kamino.py:511-579`
- internal init：`newton/_src/solvers/kamino/_src/solver_kamino_impl.py:194-250`
- internal solve：`newton/_src/solvers/kamino/_src/solver_kamino_impl.py:1000-1119`

关键 handoff：

- `SolverKamino.step()` 如果拿到 Newton `Contacts`，先 `convert_contacts_newton_to_kamino(...)`
- 如果没有外部 contacts，则改走 internal collision detector
- `SolverKaminoImpl.__init__()` 里已经把 `make_unilateral_constraints_info(...)`、`Dense/SparseSystemJacobians`、`DualProblem`、`PADMMSolver` 都接上
- `_solve_forward_dynamics()` 里再按 `intermediates -> detector/limits -> constraint info -> Jacobians -> actuation -> _forward()` 推进
- `_forward()` 里实际做 `solve(problem)`、`compute_constraint_body_wrenches(...)`、`unpack_constraint_solutions(...)`

如果你想验证 chapter 07 不是“讲完数学就结束”，这就是最直接的 continuation。

### 6. 例子怎样帮你把 family split 和 continuation 对到可运行场景

- `cartpole`：`newton/examples/robot/example_robot_cartpole.py:54-71`
- `box_on_plane` 控制回调：`newton/_src/solvers/kamino/examples/sim/example_sim_basics_box_on_plane.py:50-89`
- `box_on_plane` solver config：`newton/_src/solvers/kamino/examples/sim/example_sim_basics_box_on_plane.py:145-163`
- `box_on_plane` 外层 step：`newton/_src/solvers/kamino/examples/sim/example_sim_basics_box_on_plane.py:240-255`

这两份例子分工很明确：

- `cartpole` 负责证明共享 contract 真的允许切 solver family
- `box_on_plane` 负责把 chapter 07 的接触数学推进成 Kamino 每一步真的会消费的对象

## Optional Branches

### Branch A: `cartpole` 里 `eval_fk(...)` 为什么只对某些 solver 重要

- 例子位置：`newton/examples/robot/example_robot_cartpole.py:67-71`
- 对照说明：`newton/solvers.py:129-135`

first pass 只要守住一句话：maximal-coordinate solver 往往需要显式 FK 或显式 body pose 维护；generalized-coordinate 路线会把这部分内嵌到自己的 step 里。

### Branch B: `MuJoCo` CPU backend vs `mujoco_warp`

- main switch：`newton/_src/solvers/mujoco/solver_mujoco.py:2787-2795`, `3003-3026`

first pass 可以跳过 CPU / Warp 的实现差异。只要先守住“MuJoCo 路线是 backend bridge”即可。

### Branch C: `Kamino` internal detector vs external Newton contacts

- wrapper 决策：`newton/_src/solvers/kamino/solver_kamino.py:542-548`

这条分支很值得在 chapter 07 和 chapter 08 之间回看一次：同一个 Kamino solver 既能吃 Newton `Contacts`，也能自己跑 detector。first pass 先看 external `Contacts` 路线最顺。

## Verification Anchors

| 想验证的 claim | 直接打开哪里 |
|----------------|--------------|
| shared contract 统一的是 API，不是内部数学 | `newton/solvers.py:5-33`, `newton/_src/solvers/solver.py:301-316` |
| `SemiImplicit` 把 contacts 当 force kernels | `newton/_src/solvers/semi_implicit/solver_semi_implicit.py:141-180` |
| `Featherstone` 真正围着 joint-space 求解打转 | `newton/_src/solvers/featherstone/solver_featherstone.py:404-428`, `639-848` |
| `MuJoCo` 的关键是 backend handoff | `newton/_src/solvers/mujoco/solver_mujoco.py:2832-2834`, `3003-3026`, `3681-3730` |
| `Kamino` 是 chapter 07 的直接 continuation | `newton/_src/solvers/kamino/solver_kamino.py:542-566`, `newton/_src/solvers/kamino/_src/solver_kamino_impl.py:1000-1119` |
| `cartpole` 和 `box_on_plane` 分别负责 family split 与 contact continuation | `newton/examples/robot/example_robot_cartpole.py:60-71`, `newton/_src/solvers/kamino/examples/sim/example_sim_basics_box_on_plane.py:145-163`, `240-255` |
