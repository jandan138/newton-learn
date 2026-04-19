---
chapter: 09
title: 变分求解器族 源码走读
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

# 09 变分求解器族 源码走读

如果你是第一次读这一章，最好先配合 `principle.md` 一起读；如果你是直接跳进源码走读也没关系，但一旦发现术语开始变快，就先回到 `principle.md` 把对象关系补齐，再回来追源码。

这份主 walkthrough 是给第一次追 chapter 09 源码的人准备的。目标不是把 cloth / softbody solver 的所有变体摊平，而是让你不离开这页，也能把本章主线读成一条连续 handoff：`shared variational problem -> XPBD -> VBD -> Style3D`。

## What This Walkthrough Follows

只追这一条主线：

```text
same hanging-cloth problem
-> XPBD: per-constraint projection
-> VBD: per-vertex / per-block local solve
-> Style3D: global PD / PCG solve
```

这一页刻意不展开三类东西：

- chapter 08 的 rigid solver family 对照；这里只把 `solver.step(...)` 当共同外壳。
- XPBD / VBD / projective dynamics 的完整理论推导。
- VBD 的 AVBD 刚体分支、softbody 体网格延伸、Style3D 更细的碰撞与预条件分支；这些放到 `source-walkthrough-deep.md`。

第一遍先守住一句话：chapter 09 真正讲的是 **同一个稳定更新问题，可以按 constraint、vertex/block、global system 三个层级去组织修正**。

## One-Screen Chapter Map

```text
same cloth_hanging scene + same collide -> step loop
                       |
                       v
             predicted state / inertia target
                       |
         +-------------+--------------+
         |             |              |
         v             v              v
      XPBD           VBD          Style3D
  per-constraint   per-vertex     global PD matrix
   projection      / per-block    + PCG linear solve
    + lambdas      force+hessian
```

## Beginner Path

1. 先看 Stage 1。
   - 想验证什么：三家为什么能被放在同一章里比较。
   - 看完后应该能说：它们面对的是同一个 hanging-cloth 稳定更新问题，只是内部修正层级不同。
2. 再看 Stage 2。
   - 想验证什么：XPBD 为什么最像“逐约束投影”。
   - 看完后应该能说：XPBD 的第一语言是 constraint projection 和 `lambda` 更新。
3. 再看 Stage 3。
   - 想验证什么：VBD 为什么开始围着 vertex / block 的局部子问题组织。
   - 看完后应该能说：VBD 会先累计 force + Hessian，再做局部 `h_inv * f` 更新。
4. 最后看 Stage 4。
   - 想验证什么：Style3D 为什么更像 global PD / PCG 路线。
   - 看完后应该能说：它先准备固定 PD 矩阵，再反复解全局线性子问题。

## Main Walkthrough

### Stage 1: 三家先共享同一个 hanging-cloth 问题和外层 loop

**Claim**

chapter 09 把 `XPBD / VBD / Style3D` 放在同一章里，不是因为它们内部长得像，而是因为它们都在回答同一个问题：同一块 cloth 在同一个 `collide -> solver.step -> swap` 外层 loop 里，怎样被更稳定地往下一步推进。

**Why it matters**

这一步最能纠正新手的第一层误会：chapter 09 不是“又多了三个 solver 名字”，而是“同一问题的三种修正组织方式”。

**Source excerpt**

`newton/examples/cloth/example_cloth_hanging.py` 先把 shared problem 立得很清楚：

```python
if self.solver_type == "semi_implicit":
    self.sim_substeps = 32
elif self.solver_type == "style3d":
    self.sim_substeps = 2
else:
    self.sim_substeps = 10

...

if self.solver_type == "style3d":
    self.solver = newton.solvers.SolverStyle3D(model=self.model, iterations=self.iterations)
elif self.solver_type == "xpbd":
    self.solver = newton.solvers.SolverXPBD(model=self.model, iterations=self.iterations)
else:
    self.solver = newton.solvers.SolverVBD(model=self.model, iterations=self.iterations, ...)

...

self.model.collide(self.state_0, self.contacts)
self.solver.step(self.state_0, self.state_1, self.control, self.contacts, self.sim_dt)
```

**Verification cues**

- 三家共享同一个 cloth 场景和同一个外层时间推进骨架，这就是 chapter 09 的比较基线。
- `sim_substeps` 一开始就不同，说明它们虽然处理同一问题，但稳定性和内部更新方式已经有明显差别。
- `SemiImplicit` 也会出现在这个例子里，但这里只是 baseline 对照；真正的主角是 `XPBD / VBD / Style3D`。

**Output passed to next stage**

同一个 hanging-cloth 稳定更新问题。接下来要看的不是“谁也会 step”，而是“谁把每轮修正首先落在哪一层对象上”。

### Stage 2: `XPBD` 把每轮修正先压回单条约束

**Claim**

`SolverXPBD` 的第一语言是 per-constraint projection：先预测状态，再让每类约束 kernel 逐条产生 delta，最后把这些 delta 应回状态。

**Why it matters**

XPBD 是 chapter 09 最容易上手的一层，因为它最接近“我有一条约束，就先修这条约束”的直觉。用它开场，最容易建立后面 `constraint -> block -> global` 的梯子。

**Source excerpt**

`newton/_src/solvers/xpbd/solver_xpbd.py` 的主骨架就是“预测 -> 约束投影 -> apply delta”：

```python
self.integrate_particles(model, state_in, state_out, dt)

spring_constraint_lambdas = wp.empty_like(model.spring_rest_length)

for i in range(self.iterations):
    particle_deltas.zero_()

    if model.spring_count:
        spring_constraint_lambdas.zero_()
        wp.launch(
            kernel=solve_springs,
            dim=model.spring_count,
            inputs=[..., dt, spring_constraint_lambdas],
            outputs=[particle_deltas],
        )

    particle_q, particle_qd = self._apply_particle_deltas(model, state_in, state_out, particle_deltas, dt)
```

而 `solve_springs(...)` 直接把 XPBD 的味道写在了 `lambda` 更新里：

```python
c = l - rest
alpha = 1.0 / (ke * dt * dt)
gamma = kd / (ke * dt)

dlambda = -1.0 * (c + alpha * lambdas[tid] + gamma * grad_c_dot_v) / ((1.0 + gamma) * denom + alpha)

lambdas[tid] = lambdas[tid] + dlambda
wp.atomic_add(delta, i, dxi)
wp.atomic_add(delta, j, dxj)
```

**Verification cues**

- `particle_deltas` 是每轮迭代真正被反复覆盖和应用的对象，所以 solver 的主语言是 position correction。
- `solve_springs(...)` 里先算 `dlambda` 再写 `delta`，说明约束乘子不是旁枝，而是 projection 核心。
- rigid contact 和 joint 在这条路线上也会被组织成类似“约束先行”的修正方式，所以 XPBD 的统一感很强。

**Output passed to next stage**

XPBD 给出了 chapter 09 的第一层答案：稳定更新可以先落在“单条约束怎么修”上。

### Stage 3: `VBD` 把修正升级成 per-vertex / per-block 局部子问题

**Claim**

`SolverVBD` 不再主要问“这一条约束怎么投影”，而是问“围着这个 vertex / block 的所有力学项先累成什么局部系统，再怎么解”。

**Why it matters**

这是 chapter 09 最关键的中间台阶。只有把 VBD 读成局部 block solve，你后面看到 softbody 延伸或 AVBD 刚体分支时才不会迷路。

**Source excerpt**

`newton/_src/solvers/vbd/solver_vbd.py` 先把整个 timestep 写成三阶段骨架：

```python
self._initialize_rigid_bodies(state_in, control, contacts, dt, update_rigid_history)
self._initialize_particles(state_in, state_out, dt)

for iter_num in range(self.iterations):
    self._solve_rigid_body_iteration(state_in, state_out, control, contacts, dt)
    self._solve_particle_iteration(state_in, state_out, contacts, dt, iter_num)

self._finalize_rigid_bodies(state_out, dt)
self._finalize_particles(state_out, dt)
```

而 `_solve_particle_iteration(...)` 的粒子主线又是“先累计，再局部求解”：

```python
self.particle_forces.zero_()
self.particle_hessians.zero_()

for color in range(len(self.model.particle_color_groups)):
    wp.launch(kernel=accumulate_spring_force_and_hessian, ..., outputs=[self.particle_forces, self.particle_hessians])
    ...
    wp.launch(kernel=solve_elasticity, ..., outputs=[self.particle_displacements])
```

`solve_elasticity(...)` 最后则非常直白地把局部子问题解成 `h_inv * f`：

```python
f = mass[particle_index] * (inertia[particle_index] - pos[particle_index]) * dt_sqr_reciprocal
h = mass[particle_index] * dt_sqr_reciprocal * wp.identity(n=3, dtype=float)

h = h + particle_hessians[particle_index]
f = f + particle_forces[particle_index]

if abs(wp.determinant(h)) > 1e-8:
    h_inv = wp.inverse(h)
    particle_displacements[particle_index] = particle_displacements[particle_index] + h_inv * f
```

**Verification cues**

- `particle_forces` 和 `particle_hessians` 是 VBD 这条路线上最该盯的两个容器，因为它们把约束、接触、弹性项都拉到了同一个局部 solve 视角下。
- color groups 说明 solver 的迭代单位已经变成“按可并行的 vertex/block 子集更新”，而不是按单条约束顺序扫。
- `h_inv * f` 是 chapter 09 最值得记住的一句代码：它直接说明这里不是投影单条约束，而是在解局部线性化子问题。

**Output passed to next stage**

VBD 给出了 chapter 09 的第二层答案：稳定更新可以先落在“某个 vertex / block 的局部系统怎么解”上。

### Stage 4: `Style3D` 把修正推进成全局 PD / PCG solve

**Claim**

`SolverStyle3D` 继续往上走了一层：它先准备固定的 projective-dynamics 矩阵结构，再在每轮非线性迭代里解一个全局线性子问题。

**Why it matters**

这一步把 chapter 09 的梯子封口。你现在可以很清楚地比较三家：XPBD 修单条约束，VBD 修局部 block，Style3D 修整张 cloth 的全局系统。

**Source excerpt**

`newton/_src/solvers/style3d/solver_style3d.py` 的文档字符串先把它的问题写明了：

```python
Implicit-Euler method solves the following non-linear equation:

(M / dt^2 + H(x)) * dx = (M / dt^2) * (x_inertia - x) + f_int(x)
```

构造器和 `step()` 又把这件事落实成固定 PD 矩阵加 PCG：

```python
self.pd_matrix_builder = PDMatrixBuilder(model.particle_count)
self.linear_solver = PcgSolver(model.particle_count, self.device)

for _iter in range(self.nonlinear_iterations):
    wp.launch(init_rhs_kernel, ..., outputs=[self.rhs])
    wp.launch(eval_stretch_kernel, ..., outputs=[self.rhs])
    wp.launch(eval_bend_kernel, ..., outputs=[self.rhs])
    ...
    self.linear_solver.solve(
        self.pd_non_diags,
        self.static_A_diags,
        self.dx if _iter == 0 else None,
        self.rhs,
        self.inv_A_diags,
        self.dx,
        self.linear_iterations,
        ...,
    )
    wp.launch(nonlinear_step_kernel, ..., outputs=[state_out.particle_q, self.dx])
```

`PcgSolver` 本身也把这条路线写得很公开：

```python
class PcgSolver:
    """A Customized PCG implementation for efficient cloth simulation"""

    def solve(self, A_non_diag, A_diag, x0, b, inv_M, x1, iterations, additional_multiplier=None):
        ...
```

**Verification cues**

- `PDMatrixBuilder` 和 `PcgSolver` 在构造器里就是一级公民，这已经和 XPBD/VBD 的代码气质完全不同。
- 每轮非线性迭代里，stretch/bend/contact 都先被累计到全局 `rhs`，然后再一起交给 PCG。
- `_precompute(...)` 会提前把固定 PD 矩阵搭出来，所以这条路线非常强调“先搭全局结构，再反复线性求解”。

**Output passed to next stage**

Style3D 给出了 chapter 09 的第三层答案：稳定更新可以先落在“整张 cloth 的全局线性系统怎么解”上。到这里，本章主梯子就封口了。

## Object Ledger

| 对象 | 谁生产 | 谁消费 | 盯哪些字段 |
|------|--------|--------|------------|
| shared cloth scene / loop | `example_cloth_hanging.py` | `SolverXPBD`、`SolverVBD`、`SolverStyle3D` | `sim_substeps`、solver 选择、`collide -> step -> swap` |
| `particle_deltas` / `spring_constraint_lambdas` | `SolverXPBD.step()`、`solve_springs(...)` | XPBD 的 apply-delta 主线 | 每轮 delta、`dlambda`、projection 顺序 |
| `particle_forces` / `particle_hessians` / `particle_displacements` | `SolverVBD._solve_particle_iteration()` | `solve_elasticity*` | force+hessian 累积、color groups、局部 `h_inv * f` |
| `rhs` / `dx` / `pd_non_diags` / `static_A_diags` | `SolverStyle3D.step()`、`_precompute()` | `PcgSolver.solve()` | 全局系统右端项、线性增量、固定 PD 矩阵 |
| `contacts` | `model.collide(...)` | 三条 solver path | cloth-ground / body-particle 交互怎样进入各自的修正层级 |

## Stop Here

读到这里就已经够 chapter 09 的 80-90% 了。

如果你现在能用自己的话讲顺下面这句话，这一章的 beginner 目标就完成了：

```text
三家共享同一个 hanging-cloth 稳定更新问题；
XPBD 先逐约束投影，VBD 先围着 vertex / block 累局部 force+hessian，Style3D 则先搭全局 PD 矩阵再用 PCG 解线性子问题。
```

这时你已经不会再把 chapter 09 读成“三个 solver 说明页”。

## Go Deeper

如果你还想继续精确追源码，再去 `source-walkthrough-deep.md`：

- 想保留 cross-repo 精确锚点：看 `Fast Deep Index`
- 想逐跳追 `shared problem -> XPBD -> VBD -> Style3D`：看 `Exact Handoff Trace`
- 想补 `SemiImplicit` baseline、VBD softbody 延伸和 tile solve 分支：看 `Optional Branches`
- 想逐条核对这里的 claim：看 `Verification Anchors`
