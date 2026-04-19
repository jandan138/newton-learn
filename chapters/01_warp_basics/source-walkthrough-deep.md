---
chapter: 01
title: Warp 编程模型
last_updated: 2026-04-19
source_paths:
  - newton/_src/sim/model.py
  - newton/_src/solvers/xpbd/solver_xpbd.py
  - newton/_src/solvers/xpbd/kernels.py
  - newton/examples/selection/example_selection_materials.py
  - newton/examples/mpm/example_mpm_twoway_coupling.py
  - newton/examples/diffsim/example_diffsim_bear.py
paper_keys:
  - warp-paper
  - mujoco-warp-paper
newton_commit: 1a230702
---

# 01 Warp 编程模型 深读锚点版

这份 deep walkthrough 保留精确的上游路径、symbol 和行号。第一次读 chapter 01 时，建议先读完 `source-walkthrough.md`，建立起 `buffer -> launch -> tid -> atomic -> execution organization` 的主线，再回来用这一页核对细节。

## Fast Deep Index

| ID | repo@commit path | symbol | why it matters |
|----|------------------|--------|----------------|
| D1 | `newton@1a230702 newton/_src/sim/model.py` | `Model` fields, `Model.state`, `Model.control` | `wp.array` 在 Newton 里的长期归属关系 |
| D2 | `newton@1a230702 newton/examples/selection/example_selection_materials.py` | `compute_middle_kernel`, `capture`, `simulate`, `step` | 最干净的 launch-first 与 graph capture 例子 |
| D3 | `newton@1a230702 newton/_src/solvers/xpbd/solver_xpbd.py` | `SolverXPBD.step` | 真实 solver 的 launch site |
| D4 | `newton@1a230702 newton/_src/solvers/xpbd/kernels.py` | `solve_particle_shape_contacts` | `tid` 怎样翻成 contact / particle / body 槽位 |
| D5 | `newton@1a230702 newton/examples/mpm/example_mpm_twoway_coupling.py` | `compute_body_forces`, `simulate` | `wp.atomic_add` 的 many-to-one 写回模式 |
| D6 | `newton@1a230702 newton/examples/diffsim/example_diffsim_bear.py` | `network`, `forward` | `tile` 和 `launch_tiled` 的高级锚点 |

## Exact Handoff Trace

### 1. `wp.array` 先长成 `Model / State / Control` 的长期缓冲区

- `Model` 字段目录：`newton/_src/sim/model.py:390-492`
- runtime clone 入口：`newton/_src/sim/model.py:808-902`
- exact handoff：
  - `Model` 先持有 `body_q / body_qd / body_com / joint_q / joint_qd / joint_X_p / joint_X_c`
  - `Model.state()` 把初始 `body_q / body_qd / joint_q / joint_qd` clone 到 `State`
  - `Model.control()` 把 `joint_target_* / joint_act / joint_f` clone 到 `Control`

第一遍只要守住：Warp arrays 在 Newton 里经常是对象背后的长期存储，不是零散临时量。

### 2. launch-first 读法：`selection_materials`

- kernel 定义：`newton/examples/selection/example_selection_materials.py:37-40`
- launch site：`newton/examples/selection/example_selection_materials.py:119-131`
- capture/replay：`newton/examples/selection/example_selection_materials.py:153-182`
- exact handoff：
  - `dim=self.default_ant_dof_positions.shape` 决定了 3D batch
  - kernel 里的 `world, arti, dof = wp.tid()` 只是把这 3 个 batch 维度命名出来
  - 后面的 `capture()` / `step()` 则把整段 `simulate()` 变成 graph replay

### 3. 真实 solver 里的同一读法：`SolverXPBD.step -> solve_particle_shape_contacts`

- launch site：`newton/_src/solvers/xpbd/solver_xpbd.py:364-397`
- kernel 入口：`newton/_src/solvers/xpbd/kernels.py:143-152`
- shared writeback：`newton/_src/solvers/xpbd/kernels.py:214-222`
- exact handoff：
  - `dim=contacts.soft_contact_max` 说明这批线程按 soft-contact slot 发起
  - `shape_index = contact_shape[tid]`、`particle_index = contact_particle[tid]` 把 slot 继续翻成具体对象
  - `particle_deltas / body_deltas` 是输出缓冲区，不是 kernel 内局部变量

### 4. `wp.atomic_add` / `wp.atomic_sub` 的第一层证据链

- MPM kernel：`newton/examples/mpm/example_mpm_twoway_coupling.py:25-55`
- 对应 launch：`newton/examples/mpm/example_mpm_twoway_coupling.py:188-205`
- XPBD contact 累加：`newton/_src/solvers/xpbd/kernels.py:218-222`
- exact handoff：
  - MPM 那边，许多 collider impulse 线程都可能命中同一 `body_index`
  - XPBD 那边，许多 contact slot 线程也可能命中同一 `particle_index` 或 `body_index`
  - 原子操作因此是执行正确性的保障，不是新的物理章节

### 5. `Graph` 和 `tile` 的精确入口

- graph capture：`newton/examples/selection/example_selection_materials.py:153-182`
- tiled kernel：`newton/examples/diffsim/example_diffsim_bear.py:69-81`
- tiled launch：`newton/examples/diffsim/example_diffsim_bear.py:222-233`
- exact handoff：
  - graph 路径把一整段 `simulate()` capture 成 replayable graph
  - tile 路径把 kernel 内的矩阵乘和激活组织成 tile-local 计算
  - 两者都没有改写 Newton 的对象边界，只是换了执行方式

## Optional Branches

### Branch A: `selection_materials` 的 Torch 路径

- first pass 可跳过：`USE_TORCH=True` 的分支
- 关键原因：chapter 01 真正需要的是 launch、tid、graph 的读法，不是 Torch interop 细节

### Branch B: XPBD 里更长的约束序列

- first pass 可跳过：`solve_particle_particle_contacts`、`solve_springs` 等更多 kernels
- 关键原因：chapter 01 只需要一个真实 launch-to-kernel 例子就够了

### Branch C: `tile` 的更多性能含义

- first pass 可跳过：为什么这里要用 tiled GEMM、block 维度为什么这么选
- 关键原因：chapter 01 只需要先把 `tile` 认成“高级执行组织方式”

## Verification Anchors

| 想验证的 claim | 直接打开哪里 |
|----------------|--------------|
| `wp.array` 在 Newton 里常是长期对象字段 | `newton/_src/sim/model.py:390-492`, `newton/_src/sim/model.py:808-902` |
| launch 比 kernel 本体更先决定线程语义 | `newton/examples/selection/example_selection_materials.py:37-40`, `newton/examples/selection/example_selection_materials.py:119-131` |
| 真实 solver 里也应该先看 launch | `newton/_src/solvers/xpbd/solver_xpbd.py:364-397`, `newton/_src/solvers/xpbd/kernels.py:143-152` |
| `wp.atomic_add` 是共享写回保护 | `newton/examples/mpm/example_mpm_twoway_coupling.py:25-55`, `newton/_src/solvers/xpbd/kernels.py:218-222` |
| `Graph` / `tile` 改的是执行组织 | `newton/examples/selection/example_selection_materials.py:153-182`, `newton/examples/diffsim/example_diffsim_bear.py:69-81`, `newton/examples/diffsim/example_diffsim_bear.py:222-233` |
