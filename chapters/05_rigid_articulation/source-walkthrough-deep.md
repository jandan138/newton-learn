---
chapter: 05
title: 刚体与关节动力学
last_updated: 2026-04-19
source_paths:
  - docs/concepts/articulations.rst
  - newton/_src/sim/builder.py
  - newton/_src/sim/model.py
  - newton/_src/sim/state.py
  - newton/_src/sim/articulation.py
  - newton/_src/solvers/featherstone/kernels.py
  - newton/_src/solvers/featherstone/solver_featherstone.py
paper_keys:
  - featherstone-book
  - aba-crba-notes
newton_commit: 1a230702
---

# 05 刚体与关节动力学 深读锚点版

这份 deep walkthrough 把 chapter 05 里的精确 articulation / Featherstone 锚点集中起来。第一次读本章时，建议先读完 `source-walkthrough.md`，把 `layout -> FK -> body state -> Featherstone consumption` 主线读顺，再回来补 line-range-heavy 的证据链。

## Fast Deep Index

| ID | repo@commit path | symbol | why it matters |
|----|------------------|--------|----------------|
| D1 | `newton@1a230702 newton/_src/sim/builder.py` | `add_articulation`, `add_joint`, `finalize` | articulation 切片布局的起点 |
| D2 | `newton@1a230702 newton/_src/sim/model.py`, `newton/_src/sim/state.py` | articulation-related fields, `Model.state` | joint-space 与 body-space 状态目录 |
| D3 | `newton@1a230702 newton/_src/sim/articulation.py` | `eval_single_articulation_fk`, `eval_fk` | `joint_q / joint_qd -> body_q / body_qd` 的公开桥 |
| D4 | `newton@1a230702 newton/_src/sim/articulation.py` | `jcalc_motion_subspace`, `eval_articulation_jacobian` | motion subspace / Jacobian 的延长线 |
| D5 | `newton@1a230702 newton/_src/solvers/featherstone/kernels.py` | `compute_spatial_inertia`, `compute_com_transforms`, `eval_rigid_fk`, `compute_link_velocity` | Featherstone 的 spatial buffers |
| D6 | `newton@1a230702 newton/_src/solvers/featherstone/solver_featherstone.py` | `_compute_articulation_indices`, `_allocate_model_aux_vars`, `step` | solver orchestration 与 internal workspaces |
| D7 | `newton@1a230702 docs/concepts/articulations.rst` | articulation public conventions | 概念文档与源码语义对照 |

## Exact Handoff Trace

### 1. articulation layout 从 builder 开始就是 flat slices

- `add_articulation()`：`newton/_src/sim/builder.py:1965-2062`
- `add_joint()` 关键写入：`newton/_src/sim/builder.py:3590-3668`
- `finalize()` sentinel closure：`newton/_src/sim/builder.py:10234-10258`
- exact handoff：
  - `joint_parent / joint_child / joint_X_p / joint_X_c / joint_q_start / joint_qd_start` 在 add_joint 时写入
  - `articulation_start` 在 add_articulation 时记录每条 articulation 的起点
  - `finalize()` 再补上终止 sentinel，方便按 `[start:end)` 切片

### 2. `Model` / `State` 怎样保存 joint-space 与 body-space 两层状态

- `Model` joint/articulation fields：`newton/_src/sim/model.py:445-572`
- `State` body/joint fields：`newton/_src/sim/state.py:72-116`
- `Model.state()`：`newton/_src/sim/model.py:808-859`
- exact handoff：
  - `Model` 持有默认 generalized coordinates 和 body state 初始化值
  - `State` 同时分配 `joint_q / joint_qd` 与 `body_q / body_qd`
  - `Model.state()` 只是 clone，不自动做完整 FK 推进

### 3. public FK 桥：`joint_q / joint_qd -> body_q / body_qd`

- `eval_single_articulation_fk()`：`newton/_src/sim/articulation.py:190-374`
- `eval_fk()` kernel launch：`newton/_src/sim/articulation.py:377-516`
- exact handoff：
  - `joint_q_start / joint_qd_start` 决定每个 joint 从 generalized arrays 哪一段取值
  - 按 joint type 构造 `X_j / v_j`
  - 沿 `X_wpj -> X_wcj -> X_wc` 传播到 child body
  - 再通过 `origin_twist_to_com_twist(...)` 写成公共 `body_qd`

### 4. motion subspace / Jacobian 的延长线

- `jcalc_motion_subspace()`：`newton/_src/sim/articulation.py:889-955`
- `eval_articulation_jacobian()`：`newton/_src/sim/articulation.py:958-1125`
- exact handoff：
  - `joint_axis` 经过 `transform_twist(...)` 变成 world-space motion subspace `joint_S_s`
  - Jacobian 再按 joint ancestor 链把这些列装配出来

### 5. Featherstone 的 spatial precompute 与 velocity propagation

- spatial inertia/com transforms：`newton/_src/solvers/featherstone/kernels.py:19-49`
- `eval_rigid_fk()`：`newton/_src/solvers/featherstone/kernels.py:641-681`
- `compute_link_velocity()`：`newton/_src/solvers/featherstone/kernels.py:717-801`
- exact handoff：
  - `body_mass / body_inertia / body_com` 先变成 `body_I_m / body_X_com`
  - `eval_rigid_fk()` 刷新 `body_q / body_q_com`
  - `compute_link_velocity()` 再把 joint motion 变成 `joint_S_s / body_v_s / body_f_s / body_I_s`

### 6. `SolverFeatherstone.step()` 的完整 orchestration

- articulation index bookkeeping：`newton/_src/solvers/featherstone/solver_featherstone.py:229-331`
- step 前半：`newton/_src/solvers/featherstone/solver_featherstone.py:379-658`
- matrix build / solve / integration：`newton/_src/solvers/featherstone/solver_featherstone.py:659-936`
- exact handoff：
  - step 先 `eval_rigid_fk()` 保证 body poses 与当前 generalized state 对齐
  - 中间构建 `joint_S_s`、`body_I_s`、`J / M / H / L` 等 solver 工作区
  - 最后 `integrate_generalized_joints()` 更新 `joint_q`
  - `eval_fk_with_velocity_conversion(...)` 再把更新后的 generalized state 重建成公共 `body_q / body_qd`

## Optional Branches

### Branch A: FREE / DISTANCE joints 的 public/internal 速度转换

- first pass 可跳过：`convert_free_distance_joint_qd_public_to_internal`、`convert_free_distance_joint_qd_internal_to_public` 相关支线
- 关键原因：chapter 05 第一遍先守住“有一层 internal conversion”，不必展开每个细节

### Branch B: mass matrix / Cholesky 工作区

- first pass 可跳过：`eval_rigid_jacobian`、`eval_rigid_mass`、tile GEMM、dense Cholesky 的完整矩阵支线
- 关键原因：这些更适合在 `08_rigid_solvers` 里系统看

### Branch C: kinematic body / joint 特殊处理

- first pass 可跳过：`zero_kinematic_body_forces`、`zero_kinematic_joint_qdd`、`copy_kinematic_joint_state`
- 关键原因：不影响 articulation 主 handoff 的第一层理解

## Verification Anchors

| 想验证的 claim | 直接打开哪里 |
|----------------|--------------|
| articulation 在源码里主要表现为切片布局 | `newton/_src/sim/builder.py:1965-2062`, `newton/_src/sim/builder.py:3590-3668`, `newton/_src/sim/builder.py:10234-10258` |
| `State` 同时保留 generalized 与 body-space 两层状态 | `newton/_src/sim/state.py:72-116`, `newton/_src/sim/model.py:808-859` |
| FK 真正负责 `joint_q / joint_qd -> body_q / body_qd` | `newton/_src/sim/articulation.py:227-374` |
| motion subspace 和 Jacobian 是这条主线的延长线 | `newton/_src/sim/articulation.py:889-1125` |
| Featherstone 先把 body mass property 变成 spatial buffers | `newton/_src/solvers/featherstone/kernels.py:19-49`, `newton/_src/solvers/featherstone/kernels.py:717-801` |
| solver 最终还是把结果写回公共 `State` | `newton/_src/solvers/featherstone/solver_featherstone.py:828-936` |
