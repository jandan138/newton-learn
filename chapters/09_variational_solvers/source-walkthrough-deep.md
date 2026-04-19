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
  - newton/examples/softbody/example_softbody_hanging.py
paper_keys:
  - xpbd-paper
  - vbd-paper
  - style3d-paper
newton_commit: 1a230702
---

# 09 变分求解器族 深读锚点版

这份 deep walkthrough 保留精确 cross-repo anchors、可选分支和 verification-heavy trace。第一次读 chapter 09 时，建议先把 `source-walkthrough.md` 读完，再回来用这一页固定细节。

## Fast Deep Index

| ID | repo@commit path | symbol | why it matters |
|----|------------------|--------|----------------|
| D1 | `newton@1a230702 newton/solvers.py` | feature table / joint notes | 为什么 chapter 09 聚焦 `XPBD / VBD / Style3D` |
| D2 | `newton@1a230702 newton/examples/cloth/example_cloth_hanging.py` | `Example.__init__`, `simulate` | shared hanging-cloth problem 和外层 loop |
| D3 | `newton@1a230702 newton/_src/solvers/xpbd/solver_xpbd.py` | `SolverXPBD.step` | predicted state + iterative projection 骨架 |
| D4 | `newton@1a230702 newton/_src/solvers/xpbd/kernels.py` | `solve_springs` | XPBD 的 `lambda` / compliance 更新核心 |
| D5 | `newton@1a230702 newton/_src/solvers/vbd/solver_vbd.py` | `SolverVBD.step`, `_initialize_particles`, `_solve_particle_iteration` | VBD 三阶段骨架和 particle 主线 |
| D6 | `newton@1a230702 newton/_src/solvers/vbd/particle_vbd_kernels.py` | `solve_elasticity_tile`, `solve_elasticity` | local force+hessian -> `h_inv * f` |
| D7 | `newton@1a230702 newton/_src/solvers/style3d/solver_style3d.py` | `SolverStyle3D.step`, `_precompute` | global PD / PCG 路线 |
| D8 | `newton@1a230702 newton/_src/solvers/style3d/linear_solver.py` | `PcgSolver.solve` | 全局线性子问题真正在哪里解 |
| D9 | `newton@1a230702 newton/examples/softbody/example_softbody_hanging.py` | `Example.__init__`, `simulate` | VBD beyond cloth 的最小延伸 |

## Exact Handoff Trace

### 1. Shared problem: 同一块 cloth，三条内部路线

- solver 全景：`newton/solvers.py:91-114`, `129-135`
- shared example：`newton/examples/cloth/example_cloth_hanging.py:40-48`, `52-64`, `97-144`, `165-175`

精确阅读点：

- `semi_implicit / xpbd / vbd / style3d` 共享同一个 cloth scene
- 不同 solver 的 `sim_substeps` 一开始就不同
- `Style3D` 需要 `register_custom_attributes(...)`
- `VBD` 需要 `builder.color(include_bending=True)`
- 外层 loop 仍然是 `model.collide(...) -> solver.step(...) -> swap states`

这一层要钉住的事实是：chapter 09 的比较基准来自同一个问题实例，而不是来自一张更长的 feature matrix。

### 2. `XPBD`: predicted state -> per-constraint projection -> apply delta

- step 主骨架：`newton/_src/solvers/xpbd/solver_xpbd.py:281-341`, `343-609`
- spring kernel：`newton/_src/solvers/xpbd/kernels.py:293-355`

关键 handoff：

- `integrate_particles(...)` / `integrate_bodies(...)` 先给出预测状态
- `spring_constraint_lambdas`、`edge_constraint_lambdas` 为约束投影分配缓冲
- 每轮迭代都先清 `particle_deltas` / `body_deltas`
- `solve_springs(...)`、`bending_constraint(...)`、`solve_tetrahedra(...)`、`solve_body_contact_positions(...)` 依次产生日志 delta
- `_apply_particle_deltas(...)`、`_apply_body_deltas(...)` 再把修正写回状态

如果你想核对“XPBD 的第一语言是 constraint projection”，就盯 `dlambda`、`lambdas[tid] += dlambda` 和 `wp.atomic_add(delta, ...)` 这组三连。

### 3. `VBD`: initialize -> iterate -> finalize，然后每轮都围着 vertex/block 解局部系统

- step 总骨架：`newton/_src/solvers/vbd/solver_vbd.py:1338-1381`
- particle initialization：`newton/_src/solvers/vbd/solver_vbd.py:1445-1482`
- particle iteration：`newton/_src/solvers/vbd/solver_vbd.py:1704-1888`
- local tile solve：`newton/_src/solvers/vbd/particle_vbd_kernels.py:2970-3132`
- local scalar solve：`newton/_src/solvers/vbd/particle_vbd_kernels.py:3135-3272`

关键 handoff：

- `_initialize_particles(...)` 先用 `forward_step` 得到惯性目标和初始位移
- `_solve_particle_iteration(...)` 每轮先清 `particle_forces / particle_hessians`
- 之后按 color group 累计 body contact、spring、自碰撞等项
- `solve_elasticity_tile(...)` / `solve_elasticity(...)` 再对当前 vertex/block 解局部子问题
- 最终局部更新的核心就是把 inertia 项和累计项合成 `h_inv * f`

这一步是 chapter 09 最值得反复看的地方：它直接把“约束视角”抬升成了“局部 block 子问题视角”。

### 4. `Style3D`: fixed PD structure + nonlinear iterations + PCG

- equation / constructor：`newton/_src/solvers/style3d/solver_style3d.py:35-55`, `119-147`
- step 主体：`newton/_src/solvers/style3d/solver_style3d.py:155-333`
- precompute：`newton/_src/solvers/style3d/solver_style3d.py:388-417`
- PCG solver：`newton/_src/solvers/style3d/linear_solver.py:236-379`

关键 handoff：

- 构造器预分配 `pd_non_diags`、`pd_diags`、`rhs`、`dx`、`PcgSolver`
- `init_step_kernel(...)` 先建立 `x_prev / x_inertia / static_A_diags`
- 每轮非线性迭代里，stretch、bend、drag、contact 都往 `rhs` 上累计
- `prepare_jacobi_preconditioner_*` 生成预条件器
- `PcgSolver.solve(...)` 真正解线性子问题
- `nonlinear_step_kernel(...)` 再把 `dx` 写回粒子位置

如果你想验证“Style3D 的主角是整张 cloth 的全局系统”，就盯 `PDMatrixBuilder`、`PcgSolver` 和 `_precompute(...)` 这三个入口。

### 5. `VBD` 为什么还能延伸到 softbody

- example：`newton/examples/softbody/example_softbody_hanging.py:48-80`, `100-108`

关键 handoff：

- `builder.add_soft_grid(...)` 把问题从 cloth 网格换成体网格
- `builder.color()` 和 `SolverVBD(...)` 仍然保留
- 外层 loop 还是 `collide -> solver.step -> swap`

这条分支并不是 chapter 09 的主梯子，而是帮你验证：VBD 的“局部 block solve”视角不只适用于 cloth，也能延伸到 volumetric softbody。

## Optional Branches

### Branch A: `SemiImplicit` baseline 为什么仍然值得看一眼

- example 中的 baseline：`newton/examples/cloth/example_cloth_hanging.py:40-45`, `124-145`

它的作用不是和 main trio 平分篇幅，而是帮你看到：如果没有更强的隐式修正，你就得靠更高的 `sim_substeps` 撑稳定性。

### Branch B: XPBD 的 rigid/body contact 与 joint 分支

- rigid contacts：`newton/_src/solvers/xpbd/solver_xpbd.py:490-567`
- joints：`newton/_src/solvers/xpbd/solver_xpbd.py:569-609`

first pass 可以跳过。主 walkthrough 只需要先守住 `spring / bending / tetrahedra` 这些最容易看出 projection 风格的路径。

### Branch C: VBD 的 tile solve、非 tile solve 和 AVBD rigid path

- tile vs non-tile：`newton/_src/solvers/vbd/solver_vbd.py:1823-1888`, `newton/_src/solvers/vbd/particle_vbd_kernels.py:2970-3272`
- rigid AVBD 分支入口：`newton/_src/solvers/vbd/solver_vbd.py:1484-1703`, `1892-2401`

chapter 09 第一遍建议只守住 particle path。AVBD 刚体路径和 tile 优化都属于第二遍再挖的内容。

### Branch D: Style3D 的 contact/no-contact 预条件器分支

- contact 分支：`newton/_src/solvers/style3d/solver_style3d.py:269-309`
- no-contact 分支：`newton/_src/solvers/style3d/solver_style3d.py:291-299`

first pass 只要守住“它要解全局线性子问题”即可，不必先纠缠具体预条件器路径。

## Verification Anchors

| 想验证的 claim | 直接打开哪里 |
|----------------|--------------|
| chapter 09 比较的是同一个 hanging-cloth 稳定更新问题 | `newton/examples/cloth/example_cloth_hanging.py:40-48`, `111-144`, `165-175` |
| XPBD 的主语言是 per-constraint projection | `newton/_src/solvers/xpbd/solver_xpbd.py:343-609`, `newton/_src/solvers/xpbd/kernels.py:330-355` |
| VBD 的主语言是 local force+hessian block solve | `newton/_src/solvers/vbd/solver_vbd.py:1735-1888`, `newton/_src/solvers/vbd/particle_vbd_kernels.py:3168-3272` |
| Style3D 的主语言是全局 PD / PCG solve | `newton/_src/solvers/style3d/solver_style3d.py:42-57`, `211-323`, `388-417`, `newton/_src/solvers/style3d/linear_solver.py:340-379` |
| `SemiImplicit` 在这章里只是 baseline，而不是主角 | `newton/examples/cloth/example_cloth_hanging.py:40-45`, `124-145` |
| VBD 的 block-solve 视角还能延伸到 softbody | `newton/examples/softbody/example_softbody_hanging.py:48-80`, `100-108` |
