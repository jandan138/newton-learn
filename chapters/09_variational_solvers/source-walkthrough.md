---
chapter: 09
title: 变分求解器族 源码走读
last_updated: 2026-04-19
source_paths:
  - newton/solvers.py
  - newton/examples/cloth/example_cloth_hanging.py
  - newton/examples/softbody/example_softbody_hanging.py
  - newton/_src/solvers/xpbd/solver_xpbd.py
  - newton/_src/solvers/xpbd/kernels.py
  - newton/_src/solvers/vbd/solver_vbd.py
  - newton/_src/solvers/vbd/particle_vbd_kernels.py
  - newton/_src/solvers/style3d/solver_style3d.py
  - newton/_src/solvers/style3d/kernels.py
  - newton/_src/solvers/style3d/linear_solver.py
paper_keys:
  - xpbd-paper
  - vbd-paper
  - style3d-paper
newton_commit: 1a230702
---

# 09 变分求解器族 源码走读

这份 walkthrough 只追一条线:

```text
shared solver surface
-> shared cloth example
-> XPBD per-constraint path
-> VBD per-vertex / per-block path
-> Style3D global PD / PCG path
-> VBD softbody extension
```

第 08 章教你同一个 `step(...)` contract 后面可以接不同 solver family。第 09 章的源码走读继续保留这种节奏，但把主问题换成: **同一块 hanging cloth 的稳定修正，在源码里到底怎样分叉成三条路线？**

如果你只想做一次最小 first pass，先看这五站就够了:

1. `newton/solvers.py:L91-L114`
2. `newton/examples/cloth/example_cloth_hanging.py:L40-L45`、`L111-L144`、`L165-L175`
3. `newton/_src/solvers/xpbd/solver_xpbd.py:L245-L309` 和 `L350-L609`
4. `newton/_src/solvers/vbd/solver_vbd.py:L1338-L1381` 和 `L1704-L1888`
5. `newton/_src/solvers/style3d/solver_style3d.py:L155-L333`

## 先带着哪四个问题读

1. 三家共享的 outer problem 到底是什么？
2. `XPBD` 为什么最像逐约束修正，而不是局部 Hessian solve？
3. `VBD` 为什么更像围着一个 vertex / block 累积 force + Hessian？
4. `Style3D` 为什么更像 cloth-specialized 的 global PD / PCG 路线？

## Tier 1: 先看 shared surface 和 shared example

这一层只做一件事: 先把三家重新拉回同一个 hanging-cloth 画面里。你先不要急着看哪家更“高级”。

| 路径 | 角色 | 先读原因 |
|------|------|----------|
| `newton/solvers.py` | solver 全景、feature matrix、家族边界 | `L91-L114` 直接把 `Style3D / VBD / XPBD` 放在同一张 feature table 里；`L129-L135` 又提醒你 joints / coordinates 的边界在哪里。 |
| `newton/examples/cloth/example_cloth_hanging.py` | shared example | 同一个 scene 里同时出现 `semi_implicit / xpbd / vbd / style3d`，是 chapter 09 最省力的共同锚点。 |

### 先看 `solvers.py`: 三家为什么会被放到同一章

第一遍只要看两件事就够了:

- `L91-L114`: `SolverStyle3D`、`SolverVBD`、`SolverXPBD` 都被归在 implicit solver 视角下，而且都和 cloth / particles 有明确关联。
- `L129-L135`: `XPBD` 属于 maximal-coordinate solver，`VBD` 有自己的有限 joint 支持，`Style3D` 不处理 joints。

这一层的教学结论不是“谁支持什么功能更多”，而是:

```text
这三家之所以能放在一章里, 是因为它们都在处理 cloth / particle 世界里的稳定更新问题。
```

### 再看 `example_cloth_hanging.py`: shared problem 在哪里

这份例子最值得按三段读:

- `L40-L45`: 不同 solver 的 `sim_substeps` 一上来就不同。
- `L52-L57`、`L111-L117`、`L124-L144`: builder 和 solver setup 根据 solver family 分叉。
- `L165-L175`: 但真正的 simulation loop 仍然保持同样的 `collide -> solver.step -> swap states`。

如果你读完这一层还会把 chapter 09 想成“又有三个 solver 名字”，就说明你漏掉了最值钱的画面:

```text
同一块布
-> 同一个外层 loop
-> 三种不同的内部修正组织方式
```

顺手记住: `SemiImplicit` 也在这个例子里，但它在 chapter 09 的角色只是 baseline 对照。真正的源码主线从这里开始要转向 `XPBD / VBD / Style3D`。

## Tier 2: 三条 variational path

这一层才进入 chapter 09 的三条主路线。推荐顺序不要反过来，因为 `XPBD -> VBD -> Style3D` 正好对应局部更新层级从最直观到最全局的梯子。

| 路径 | 角色 | 先读原因 |
|------|------|----------|
| `newton/_src/solvers/xpbd/solver_xpbd.py` | XPBD 主流程 | 看预测、迭代、apply delta 怎样围着“逐约束修正”组织。 |
| `newton/_src/solvers/xpbd/kernels.py` | XPBD constraint kernels | 看 compliance / lambda / projection 的具体味道。 |
| `newton/_src/solvers/vbd/solver_vbd.py` | VBD 主流程 | 看 initialize / iterate / finalize 的三阶段骨架，以及 particle iteration 怎样围着 color group 展开。 |
| `newton/_src/solvers/vbd/particle_vbd_kernels.py` | VBD 粒子局部 block solve | 看 force + Hessian 怎样在局部累计，再变成 `h_inv * f`。 |
| `newton/_src/solvers/style3d/solver_style3d.py` | Style3D 主流程 | 看 fixed PD matrix、nonlinear iterations 和 PCG handoff 怎样串起来。 |
| `newton/_src/solvers/style3d/linear_solver.py` | 全局 PCG 子求解器 | 看 global linear solve 真正发生在哪里。 |

### 路径 A: `XPBD` 先把每轮修正压回“单条约束”

推荐先盯 `newton/_src/solvers/xpbd/solver_xpbd.py:L245-L309` 和 `L350-L609`。

这份 `step()` 最适合 beginner 的读法是四段:

1. `L281-L299`: 先对 particles 做前向积分，并准备可能需要的粒子邻域结构。
2. `L300-L341`: 再对 bodies 做前向积分，把 joint forces 先并进临时 body force。
3. `L343-L348`: 为 spring / bending 约束分配 `lambda` 缓冲。
4. `L350-L609`: 进入主迭代循环，按 contact、spring、bending、tet、joint 的顺序求修正。

最值钱的理解不是“它支持很多约束”，而是这条数据流:

```text
预测状态
-> 逐类约束 kernel 产生 delta
-> apply delta
-> 下一轮再继续投影
```

再去看 `xpbd/kernels.py`，你会发现它的隐式味道不是口号，而是直接写在 kernel 里。更准确地说，它依赖的是 compliance 和当前这轮约束乘子更新，而不是跨 outer iteration 保存一整段 lambda 历史:

- `L293-L355`: `solve_springs(...)` 持续维护 `lambdas[tid]`，并通过 `alpha`、`gamma` 算 `dlambda`。
- `L359-L456`: `bending_constraint(...)` 也是同样的结构，只是约束变成了二面角。
- `L460-L562`: `solve_tetrahedra(...)` 把体积和拉伸误差转成局部修正。
- `L1442` 的 `solve_body_joints(...)` 和 `L2055` 的 `solve_body_contact_positions(...)`: 刚体 joint / contact 也被组织成这种“约束先行”的修正方式。

如果你只想带走一句话，那就是:

```text
XPBD 的第一语言是 constraint projection, 不是 block Hessian 或 global matrix。
```

### 路径 B: `VBD` 的第一语言已经变成 vertex / block

`newton/_src/solvers/vbd/solver_vbd.py` 第一遍建议只抓三段:

- `L1338-L1381`: `step()` 总骨架，三阶段结构一眼就能看懂。
- `L1445-L1482`: `_initialize_particles(...)` 先做 forward step，得到惯性目标和初始位移。
- `L1704-L1890`: `_solve_particle_iteration(...)` 才是 chapter 09 的 cloth 主线。

注意这里有一个初学者非常值得先建立的区别:

```text
XPBD 是“遍历约束”。
VBD 是“遍历 color group 里的 vertex / block”。
```

在 `_solve_particle_iteration(...)` 里，这件事非常具体:

- `L1735-L1737`: 先清空当前轮的 `particle_forces` 和 `particle_hessians`。
- `L1739-L1740`: 进入 color loop。
- `L1742-L1822`: 当前颜色的 body contact、spring、自碰撞等项都先累计成局部 force + Hessian。
- `L1823-L1888`: 再调用 `solve_elasticity_tile(...)` 或 `solve_elasticity(...)` 计算位移更新。

真正让你看懂 “block solve” 的地方在 `particle_vbd_kernels.py`:

- `L2970-L3132`: `solve_elasticity_tile(...)` 围着单个粒子收集相邻三角形、边和四面体的贡献，最后解一个局部系统。
- `L3135-L3272`: `solve_elasticity(...)` 的普通版本更直白，最后就是把惯性项、弹性项、接触项汇总，再做 `h_inv * f`。
- `L860-L1022`: 三角形 StVK force / Hessian。
- `L1055-L1187`: bending force / Hessian。
- `L335-L462`: tet Neo-Hookean force / Hessian。

读 VBD 时最该保护的直觉是:

```text
它不是一条约束一条约束地投影，
而是把“和这个 vertex / block 相关的所有力学项”先凑成一个局部子问题再解。
```

顺手提醒一个 chapter 09 边界: `solver_vbd.py` 里还有大量刚体 AVBD 路径。它们说明 VBD 家族的外延很广，但本章第一遍不需要深挖这些 rigid details。对 cloth 主线来说，优先把 particle path 读顺就够了。

如果你是第一次读这章，可以把 AVBD 刚体路径直接当成“已知存在，但先跳过”的分支，不要让它把你从 `constraint -> block -> global` 这条主梯子上拽走。

### 路径 C: `Style3D` 把 cloth 修正推进成全局 PD / PCG solve

`newton/_src/solvers/style3d/solver_style3d.py` 的好处是，它把自己的算法意图写得很公开。

推荐先按四段读:

1. `L42-L57`: 文档字符串先说明 implicit Euler 非线性方程，以及 `P` 作为固定 PD 近似 Hessian。
2. `L119-L147`: 构造器预分配 PD matrix、PCG solver、`rhs`、`dx`、预条件器等缓存。
3. `L155-L333`: `step()` 把每次 timestep 写成非常清楚的 nonlinear solve workflow。
4. `L388-L417`: `_precompute(builder)` 负责把固定 PD matrix 真正搭出来。

如果你已经走过 `XPBD` 和 `VBD`，这里最该读出来的是主角又换了:

```text
XPBD 主角是单条约束。
VBD 主角是单个 vertex / block。
Style3D 主角是整张 cloth 的线性系统。
```

`step()` 里可以很容易看见这件事:

- `L173-L194`: `init_step_kernel(...)` 先建立 `x_prev`、`x_inertia` 和静态对角项。
- `L211-L279`: 每轮非线性迭代里，stretch、bend、drag、contact 都会把贡献累到 `rhs` 上。
- `L280-L299`: 准备 Jacobi 预条件器。
- `L300-L309`: `PcgSolver.solve(...)` 真正做这轮线性求解。
- `L314-L323`: `nonlinear_step_kernel(...)` 把 `dx` 写回位置。

`style3d/linear_solver.py:L236-L379` 又把这件事进一步剖开了。`PcgSolver` 并不是“附带小工具”，而是 `Style3D` 真正完成 global solve 的核心机械装置。

所以这一条路线更适合压成下面这句:

```text
Style3D 读起来最不像“再换一组局部约束 kernel”，
而更像“先准备固定矩阵, 再反复解全局线性子问题”。
```

## Tier 3: VBD beyond cloth

这一层不再做三家并排比较，只补 chapter 09 必须讲清的一件事: `VBD` 不只是一条 cloth 路线。这里仍然站在同一张梯子上，只是把 “per-block local solve” 这一级从布片延伸到了 tet softbody。

| 路径 | 角色 | 先读原因 |
|------|------|----------|
| `newton/examples/softbody/example_softbody_hanging.py` | VBD 的 softbody 延伸例子 | 让你看到同一套局部 block / Hessian 思路怎样继续走到 tet softbody。 |

### 先看 `example_softbody_hanging.py`: 为什么它只是 Tier 3

第一遍只要盯四处:

- `L32-L33`: 例子明确禁止非 `vbd` solver。
- `L48-L65`: 场景由四组 `add_soft_grid(...)` 体网格组成，而且有不同 damping。
- `L68-L80`: `builder.color()` 和 `SolverVBD(...)` 仍然保留。
- `L100-L111`: 外层 loop 仍然是 `collide -> solver.step -> swap states`。

它被放在 Tier 3，而不是一上来就和 `cloth_hanging` 并排，是因为它承担的教学任务更单一:

```text
不是比较三家, 而是补一句: VBD 的局部 block solve 视角还能走到 volumetric softbody。
```

## 最后压成一张总图

```text
shared cloth example
-> XPBD: per-constraint projection
-> VBD: per-vertex / per-block force + Hessian solve
-> Style3D: global PD / PCG solve
-> softbody_hanging: VBD beyond cloth
```

推荐的一遍读法是:

1. `newton/solvers.py`
2. `newton/examples/cloth/example_cloth_hanging.py`
3. `newton/_src/solvers/xpbd/solver_xpbd.py`
4. `newton/_src/solvers/xpbd/kernels.py`
5. `newton/_src/solvers/vbd/solver_vbd.py`
6. `newton/_src/solvers/vbd/particle_vbd_kernels.py`
7. `newton/_src/solvers/style3d/solver_style3d.py`
8. `newton/_src/solvers/style3d/linear_solver.py`
9. `newton/examples/softbody/example_softbody_hanging.py`

读完这一遍，你就不该再把 chapter 09 读成“三个 solver 说明页”，而会开始用 shared problem 和局部更新层级去比较它们。
