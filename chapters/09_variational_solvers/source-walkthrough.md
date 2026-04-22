---
chapter: 09
title: 变分求解器族 源码走读
last_updated: 2026-04-21
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

这份主 walkthrough 先回答一个真正的新手问题：同一块 hanging cloth，外层 loop 还是同一个 `collide -> step -> swap`，但每轮迭代里到底先纠正谁？是一条 constraint、一个局部 vertex / block，还是整张 cloth？

chapter 09 里说的 `variational`，第一遍先把它读成一句人话：solver 先给 cloth 一份只顺着惯性和外力往前走的预测，再把 stretch / bend / contact 这些要求组织成一个“怎样更稳地拉回去”的修正问题。三家差别不在场景换了，而在 **每轮迭代最先被反复更新的容器换了**。

第一遍先把最核心的三个词钉住，这样后面基本不用再跳去 `principle.md`；`lambda`、`Hessian`、`PCG` 放到对应 stage 再补：

- `variational`：把“下一步怎样更稳”写成一个修正问题，而不是只做一次显式前推。
- `predicted state`：还没经过约束或能量修正前，那份先积分出来的临时状态。
- `inertia target`：如果这一步主要顺着惯性和外力走，粒子最想去的位置；它常常和 predicted state 同向，但 solver 里出现的字段名不一定一样。

你当然可以再配合 `principle.md`，但只读这一页，也应该能把 chapter 09 的主线讲顺。

## What This Walkthrough Follows

只追这一条主线：

```text
same hanging-cloth problem
-> same collide -> step -> swap shell
-> same predicted state / inertia target idea
-> ask one question: who gets corrected first each iteration?
-> XPBD: deltas / lambdas
-> VBD: local force + Hessian containers / local displacements
-> Style3D: rhs / dx around a fixed PD matrix
```

这一页刻意不展开三类东西：

- chapter 08 的 rigid solver family 对照；这里只把 `solver.step(...)` 当共同外壳。
- XPBD / VBD / projective dynamics 的完整理论推导。
- VBD 的 AVBD 刚体分支、`softbody_hanging` 这条 VBD 延伸、Style3D 更细的碰撞与预条件分支；这些放到 `source-walkthrough-deep.md`。这里的 `softbody_hanging` 只负责给 VBD 补一句“这条局部 block 路线还能继续走到 softbody”，不是第二条主线。

第一遍先守住一句话：chapter 09 真正讲的是 **同一块 cloth 在同一个 outer loop 里，每轮先反复更新哪个容器，于是修正层级就落在 constraint、local block 或 whole-cloth global system 上**。

## One-Screen Chapter Map

```text
same cloth_hanging scene + same collide -> step -> swap
                       |
                       v
            predicted state / inertia target
                       |
                       v
       who gets corrected first each iteration?
         /                  |                   \
        v                   v                    v
      XPBD                VBD               Style3D
  deltas / lambdas   local force +         rhs / dx
  one constraint     Hessian containers    around fixed PD matrix
                     + displacements       + PCG
                     one local block       whole cloth
```

## Beginner Path

1. 先看 Stage 1。
   - 想验证什么：为什么同一个 hanging cloth 和同一个 `collide -> step -> swap` loop 就足够把三家放在一起比较。
   - 看完后应该能说：它们面对的是同一个 cloth 修正问题，先别背 solver 名字，先问“每轮先纠正谁”。
2. 再看 Stage 2。
   - 想验证什么：XPBD 为什么最像“逐约束投影”。
   - 看完后应该能说：XPBD 每轮先反复更新的是 `particle_deltas` / `lambdas`，所以它的第一语言是 constraint projection。
3. 再看 Stage 3。
   - 想验证什么：VBD 为什么开始围着 vertex / block 的局部子问题组织。
   - 看完后应该能说：VBD 每轮先反复更新 local force + Hessian 容器，再把它们解成局部 `h_inv * f` 位移。
4. 最后看 Stage 4。
   - 想验证什么：Style3D 为什么更像 global PD / PCG 路线。
   - 看完后应该能说：它围着固定 PD 矩阵，反复更新 `rhs` / `dx`，所以是在解整张 cloth 的全局线性子问题。

## Main Walkthrough

### Stage 1: 同一块 hanging cloth，同一个 `collide -> step -> swap` loop，问题只剩“每轮先纠正谁”

**Definitions**

- `variational`：不是“凭感觉改位置”，而是把这一步写成一个稳定修正问题。
- `predicted state`：还没处理 stretch / bend / contact 之前，那份先往前走出来的临时状态。
- `inertia target`：如果这一步先顺着惯性继续前进，粒子最想去的目标位置；后面 VBD 和 Style3D 会更显式地围着它回拉。

**Claim**

chapter 09 把 `XPBD / VBD / Style3D` 放在同一章里，不是因为它们内部长得像，而是因为它们都在回答同一个问题：同一块 cloth 在同一个 `collide -> solver.step -> swap` 外层 loop 里，怎样被更稳定地往下一步推进；真正分叉的是每轮修正先落到哪一个容器上。

**Why it matters**

这一步最能纠正新手的第一层误会：chapter 09 不是“又多了三个 solver 名字”，而是“同一块 cloth 的同一步修正，先从哪一层开始改”。

**Source excerpt**

`newton/examples/cloth/example_cloth_hanging.py` 先把 shared problem 立得很清楚：

以下摘录为教学注释版，注释非原源码。

```python
if self.solver_type == "semi_implicit":
    self.sim_substeps = 32  # baseline 需要更多 substeps 才稳
elif self.solver_type == "style3d":
    self.sim_substeps = 2  # Style3D 每步更重，所以外层子步更少
else:
    self.sim_substeps = 10  # XPBD / VBD 落在中间这一档

...

if self.solver_type == "style3d":
    self.solver = newton.solvers.SolverStyle3D(model=self.model, iterations=self.iterations)  # 选整张 cloth 的全局 PD / PCG 路线
elif self.solver_type == "xpbd":
    self.solver = newton.solvers.SolverXPBD(model=self.model, iterations=self.iterations)  # 选逐约束投影路线
else:
    self.solver = newton.solvers.SolverVBD(model=self.model, iterations=self.iterations, ...)  # 选局部 block solve 路线

...

self.model.collide(self.state_0, self.contacts)  # 先用当前 cloth state 刷新 contacts
self.solver.step(self.state_0, self.state_1, self.control, self.contacts, self.sim_dt)  # 再让选中的 solver 推进到下一拍
self.state_0, self.state_1 = self.state_1, self.state_0  # 交换双缓冲，准备下个 substep
```

**Verification cues**

- 三家共享同一个 cloth 场景和同一个外层时间推进骨架，这就是 chapter 09 的比较基线。
- `sim_substeps` 一开始就不同，说明它们虽然处理同一问题，但稳定性和内部更新方式已经有明显差别。
- `SemiImplicit` 也会出现在这个例子里，但这里只是 baseline 对照；真正的主角是 `XPBD / VBD / Style3D`。
- 到这里先不要急着比完整理论，先把比较轴收缩成一句话：预测状态已经有了以后，solver 每轮先反复更新谁？

**Checkpoint**

如果你现在还会把 chapter 09 读成“三张 solver 说明页”，先停一下。第一遍只要守住：同一块 hanging cloth、同一个 outer loop、同一个稳定修正问题，接下来只是要看三家先动的是哪种容器。

**Output passed to next stage**

同一个 hanging-cloth 稳定更新问题。接下来要看的不是“谁也会 step”，而是“预测状态已经摆出来以后，谁把每轮修正首先落在哪个容器上”。

### Stage 2: `XPBD` 把每轮修正先压回单条约束

**Definition**

`lambda`：XPBD 给单条约束记的累计修正量。第一遍可以把它当成“这条约束已经帮我往回拉了多少”的小账本。

**Claim**

`SolverXPBD` 的第一语言是 per-constraint projection：先预测状态，再让每类约束 kernel 逐条更新 `particle_deltas` 和对应的 `lambdas`，最后把这些 delta 应回状态。

**Why it matters**

XPBD 是 chapter 09 最容易上手的一层，因为它最接近“我有一条约束，就先修这条约束”的直觉。这里最该盯的不是所有数学细节，而是第一个被反复清零、累计、再应用的容器就是 `particle_deltas`，旁边还跟着 `lambda` 账本。

**Source excerpt**

`newton/_src/solvers/xpbd/solver_xpbd.py` 的主骨架就是“预测 -> 约束投影 -> apply delta”：

以下摘录为教学注释版，注释非原源码。

```python
self.integrate_particles(model, state_in, state_out, dt)  # 先得到还没被约束拉回的预测状态

spring_constraint_lambdas = wp.empty_like(model.spring_rest_length)  # 给每条 spring 约束准备 lambda 账本

for i in range(self.iterations):
    particle_deltas.zero_()  # 每轮先清空本轮累计的位置修正

    if model.spring_count:
        spring_constraint_lambdas.zero_()  # 这一类约束的 lambda 从零开始累计
        wp.launch(
            kernel=solve_springs,
            dim=model.spring_count,
            inputs=[..., dt, spring_constraint_lambdas],  # kernel 一边读约束数据，一边读写 lambda
            outputs=[particle_deltas],  # 所有约束先把修正写进共享 delta 容器
        )

    particle_q, particle_qd = self._apply_particle_deltas(model, state_in, state_out, particle_deltas, dt)  # 这一轮结束时再把 delta 应回粒子状态
```

而 `solve_springs(...)` 直接把 XPBD 的味道写在了 `lambda` 更新里：

下面这段公式密集，先保持代码形状，只盯“先算 `dlambda`，再把它落实成两端粒子的 delta”这条主线：

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

**Checkpoint**

如果你现在还不能一句话说出“XPBD 每轮先动的是哪两个东西”，就先停在这里：答案不是抽象的“约束数学”，而是很具体的 `particle_deltas` 和 `lambdas`。

**Output passed to next stage**

XPBD 给出了 chapter 09 的第一层答案：稳定更新可以先落在“单条约束怎么修”，并通过 `deltas / lambdas` 把这件事记下来。

### Stage 3: `VBD` 把修正升级成 per-vertex / per-block 局部子问题

**Definition**

`Hessian`：局部刚度或曲率摘要。第一遍不必背推导，只要记住它告诉 solver“如果这里再动一点，力会怎样变”，所以它能把很多项组织进同一个局部线性化子问题。

**Claim**

`SolverVBD` 不再主要问“这一条约束怎么投影”，而是问“围着这个 vertex / block 的所有力学项先累成什么局部系统，再怎么解”。所以它每轮最先反复更新的容器变成了 `particle_forces`、`particle_hessians` 和 `particle_displacements`。

**Why it matters**

这是 chapter 09 最关键的中间台阶。只有把 VBD 读成局部 block solve，你后面看到 softbody 延伸或 AVBD 刚体分支时才不会迷路；而且 `softbody_hanging` 在这章里也只该被读成这条局部 block 视角的延伸，不是新的主线。

**Source excerpt**

`newton/_src/solvers/vbd/solver_vbd.py` 先把整个 timestep 写成三阶段骨架：

以下摘录为教学注释版，注释非原源码。

```python
self._initialize_rigid_bodies(state_in, control, contacts, dt, update_rigid_history)  # 先把刚体支线的本轮输入准备好
self._initialize_particles(state_in, state_out, dt)  # 再把粒子 predicted state / workspace 准备好

for iter_num in range(self.iterations):
    self._solve_rigid_body_iteration(state_in, state_out, control, contacts, dt)  # 先走刚体支线这一轮
    self._solve_particle_iteration(state_in, state_out, contacts, dt, iter_num)  # 再走 cloth 粒子的局部 block 修正

self._finalize_rigid_bodies(state_out, dt)  # 收尾刚体状态
self._finalize_particles(state_out, dt)  # 收尾粒子状态
```

而 `_solve_particle_iteration(...)` 的粒子主线又是“先累计，再局部求解”：

以下摘录为教学注释版，注释非原源码。

```python
self.particle_forces.zero_()  # 每轮先清空局部力累计桶
self.particle_hessians.zero_()  # 同时清空局部刚度 / Hessian 累计桶

for color in range(len(self.model.particle_color_groups)):
    wp.launch(kernel=accumulate_spring_force_and_hessian, ..., outputs=[self.particle_forces, self.particle_hessians])  # 先把这一批 vertex / block 的力和 Hessian 累起来
    ...
    wp.launch(kernel=solve_elasticity, ..., outputs=[self.particle_displacements])  # 再把累计结果解成局部位移
```

`solve_elasticity(...)` 最后则非常直白地把局部子问题解成 `h_inv * f`：

以下摘录为教学注释版，注释非原源码。

```python
f = mass[particle_index] * (inertia[particle_index] - pos[particle_index]) * dt_sqr_reciprocal  # 惯性目标给出的右端项
h = mass[particle_index] * dt_sqr_reciprocal * wp.identity(n=3, dtype=float)  # 质量项先提供基础局部矩阵

h = h + particle_hessians[particle_index]  # 把累计 Hessian 并进局部系统
f = f + particle_forces[particle_index]  # 把累计力并进同一个右端项

if abs(wp.determinant(h)) > 1e-8:
    h_inv = wp.inverse(h)
    particle_displacements[particle_index] = particle_displacements[particle_index] + h_inv * f  # 用 h_inv * f 得到这次局部位移修正
```

**Verification cues**

- `particle_forces` 和 `particle_hessians` 是 VBD 这条路线上最该盯的两个容器，因为它们把约束、接触、弹性项都拉到了同一个局部 solve 视角下。
- color groups 说明 solver 的迭代单位已经变成“按可并行的 vertex / block 子集更新”，而不是按单条约束顺序扫。
- `h_inv * f` 是 chapter 09 最值得记住的一句代码：它直接说明这里不是投影单条约束，而是在解局部线性化子问题。
- `f = mass * (inertia - pos) * dt_sqr_reciprocal` 这句也很重要，因为它把前面说的 inertia target 明确拉进了同一个局部系统。

**Checkpoint**

如果你现在还会把 VBD 读成“更复杂的 XPBD”，就先不要往后跳。第一遍只要抓住：VBD 每轮先清空并累计 local force + Hessian 容器，再把它们解成局部位移。

**Output passed to next stage**

VBD 给出了 chapter 09 的第二层答案：稳定更新可以先落在“某个 vertex / block 的局部系统怎么解”上。

这里顺手把主线和延伸分清：`softbody_hanging` 在本章只负责验证这套 local force + Hessian / block-solve 视角还能继续走到 tet softbody。它不是和 `cloth_hanging` 并列的第二主线，而是给 chapter 10 留的一座桥。到了 chapter 10，镜头会改成“这块 softbody 到底是什么对象家族”，而不再只追 VBD 这条 solver 路线。

### Stage 4: `Style3D` 把修正推进成全局 PD / PCG solve

**Definition**

- `PD matrix`：projective dynamics 这条路线里预先搭好的固定刚度骨架。第一遍先把它读成“这张 cloth 这一类弹性结构的固定线性框架”。
- `PCG`：一种专门拿来迭代解大稀疏线性系统的方法。第一遍先把它读成“我不直接求整个矩阵的逆，而是反复逼近这次全局解”。

**Claim**

`SolverStyle3D` 继续往上走了一层：它先准备固定的 projective-dynamics 矩阵结构，再在每轮非线性迭代里围着 `rhs` 和 `dx` 解一个全局线性子问题。

**Why it matters**

这一步把 chapter 09 的梯子封口。你现在可以很清楚地比较三家：XPBD 先反复写 `deltas / lambdas`，VBD 先反复写 local force + Hessian 容器，Style3D 则围着固定 PD 矩阵反复写 `rhs / dx`，所以它处理的是整张 cloth 的全局系统。

**Source excerpt**

`newton/_src/solvers/style3d/solver_style3d.py` 的文档字符串先把它的问题写明了：

```python
Implicit-Euler method solves the following non-linear equation:

(M / dt^2 + H(x)) * dx = (M / dt^2) * (x_inertia - x) + f_int(x)
```

构造器和 `step()` 又把这件事落实成固定 PD 矩阵加 PCG：

以下摘录为教学注释版，注释非原源码。

```python
self.pd_matrix_builder = PDMatrixBuilder(model.particle_count)  # 先准备整张 cloth 共用的 PD 矩阵骨架
self.linear_solver = PcgSolver(model.particle_count, self.device)  # 再准备专门解全局线性系统的 PCG

for _iter in range(self.nonlinear_iterations):
    wp.launch(init_rhs_kernel, ..., outputs=[self.rhs])  # 先清并初始化这一轮全局 rhs
    wp.launch(eval_stretch_kernel, ..., outputs=[self.rhs])  # stretch 项往全局 rhs 里加贡献
    wp.launch(eval_bend_kernel, ..., outputs=[self.rhs])  # bend 项继续往同一个 rhs 里加贡献
    ...
    self.linear_solver.solve(
        self.pd_non_diags,
        self.static_A_diags,
        self.dx if _iter == 0 else None,  # 首轮可用旧 dx 当 warm start
        self.rhs,  # 这一轮累计出来的全局右端项
        self.inv_A_diags,
        self.dx,  # PCG 把求出的全局位移增量写回这里
        self.linear_iterations,
        ...,
    )
    wp.launch(nonlinear_step_kernel, ..., outputs=[state_out.particle_q, self.dx])  # 再把 dx 应回 cloth 顶点位置
```

`PcgSolver` 本身也把这条路线写得很公开：

```python
class PcgSolver:
    """A Customized PCG implementation for efficient cloth simulation"""

    def solve(self, A_non_diag, A_diag, x0, b, inv_M, x1, iterations, additional_multiplier=None):
        ...
```

**Verification cues**

- `PDMatrixBuilder` 和 `PcgSolver` 在构造器里就是一级公民，这已经和 XPBD / VBD 的代码气质完全不同。
- 每轮非线性迭代里，stretch / bend / contact 都先被累计到全局 `rhs`，然后再一起交给 PCG。
- `_precompute(...)` 会提前把固定 PD 矩阵搭出来，所以这条路线非常强调“先搭全局结构，再反复线性求解”。
- 第一遍最该盯住的容器就是 `rhs` 和 `dx`：前者收集这轮全局右端项，后者保存这轮线性解出来的位移增量。

**Checkpoint**

如果你现在还说不清 Style3D 这轮里“哪些东西大体固定、哪些东西在反复更新”，先停一下。固定的是 PD 矩阵骨架，反复刷新的则是 `rhs` 和 `dx`。

**Output passed to next stage**

Style3D 给出了 chapter 09 的第三层答案：稳定更新可以先落在“整张 cloth 的全局线性系统怎么解”上。到这里，本章主梯子就封口了。

## Object Ledger

| 对象 | 谁生产 | 谁消费 | 第一遍盯什么 |
|------|--------|--------|--------------|
| shared cloth scene / loop | `example_cloth_hanging.py` | `SolverXPBD`、`SolverVBD`、`SolverStyle3D` | `sim_substeps`、solver 选择、`collide -> step -> swap` |
| predicted state / inertia target | `integrate_*()`、`_initialize_particles()`、`init_step_kernel(...)` | 三条 solver path 的内层迭代 | 还没被修正前，cloth 这一步先想往哪走 |
| `particle_deltas` / `spring_constraint_lambdas` | `SolverXPBD.step()`、`solve_springs(...)` | XPBD 的 apply-delta 主线 | 每轮先清什么、写什么、`dlambda` 怎样带动 delta |
| `particle_forces` / `particle_hessians` / `particle_displacements` | `SolverVBD._solve_particle_iteration()` | `solve_elasticity*` | local force + Hessian 怎样累计，再怎样解成位移 |
| `rhs` / `dx` / `pd_non_diags` / `static_A_diags` | `SolverStyle3D.step()`、`_precompute()` | `PcgSolver.solve()` | 哪些是固定 PD 矩阵，哪些是每轮刷新的全局容器 |
| `contacts` | `model.collide(...)` | 三条 solver path | cloth-ground / body-particle 交互怎样进入各自的修正层级 |

## Stop Here

读到这里就已经够 chapter 09 的 80-90% 了。

如果你现在能用自己的话讲顺下面这句话，这一章的 beginner 目标就完成了：

```text
三家共享同一个 hanging-cloth 稳定更新问题，也共享同一个 `collide -> step -> swap` 外壳；
XPBD 每轮先反复更新 `deltas / lambdas`，所以像逐约束投影；
VBD 每轮先反复更新 local force + Hessian 容器和局部位移，所以像 vertex / block solve；
Style3D 围着固定 PD 矩阵反复更新 `rhs / dx`，所以是在解整张 cloth 的全局线性子问题。
```

这时你已经不会再把 chapter 09 读成“三个 solver 说明页”，也不容易把 `softbody_hanging` 误读成第二条主线。它在这里只是 VBD 的延伸入口，也是往 chapter 10 过桥的地方。

## Go Deeper

如果你还想继续精确追源码，再去 `source-walkthrough-deep.md`：

- 想保留 cross-repo 精确锚点：看 `Fast Deep Index`
- 想逐跳追 `shared problem -> XPBD -> VBD -> Style3D`：看 `Exact Handoff Trace`
- 想补 `SemiImplicit` baseline、VBD 的 `softbody_hanging` 延伸和 tile solve 分支：看 `Optional Branches`
- 想逐条核对这里的 claim：看 `Verification Anchors`
