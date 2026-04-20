---
chapter: 10
title: 软体、布料与 Cable 原理主线
last_updated: 2026-04-21
source_paths:
  - newton/examples/cloth/example_cloth_hanging.py
  - newton/examples/softbody/example_softbody_hanging.py
  - newton/examples/cable/example_cable_twist.py
  - newton/_src/sim/builder.py
paper_keys: []
newton_commit: 1a230702
---

# 10 软体、布料与 Cable 原理主线

## 0. 先把 chapter 09 的问题换个方向问

chapter 09 的重点是 solver family: **同一块 cloth / softbody 进入一步 `dt` 之后，修正先落在 constraint、block，还是 global system？**

chapter 10 不再先问“谁来修”，而是先问:

```text
被修的东西到底是什么？
```

这个顺序变化很重要，因为在 Newton 里，`cloth`、`softbody`、`cable` 虽然都能表现出柔性，但它们不是同一种内部对象。

## 1. 先给出 chapter 10 的一句总答案

最短答案就是这三行:

```text
cloth    = particles + triangles + bending edges / optional springs
softbody = particles + tetrahedra + generated surface collision mesh
cable    = capsule rigid bodies + cable joints (+ articulation)
```

如果你只看屏幕里的结果，它们都像“会弯、会垂、会晃”的可变形物体。

但如果你看 `builder.py` 里的入口，就会发现 Newton 其实把它们分成了三种完全不同的表示路线:

- `add_cloth_grid(...)` 先造一张表面粒子网格。
- `add_soft_grid(...)` 先造一块体粒子网格，再从体单元边界生成表面碰撞网格。
- `add_rod(...)` 先造一串 capsule rigid bodies，再用 cable joints 把它们连成一条线。

所以 chapter 10 的第一原则是:

```text
不要先按“看起来都可变形”来分，
而要先按“内部是粒子表面、粒子体积，还是刚体链”来分。
```

## 2. 三个对象家族到底差在哪

| 家族 | 主 builder 入口 | 主状态单位 | 拓扑 / 连接元 | 为什么看起来会“变形” | 这会改变什么 |
|------|-----------------|------------|---------------|----------------------|----------------|
| cloth | `add_cloth_grid(...)` | particle | triangles + bending edges + optional springs | 表面三角网格被拉伸、弯折、下垂 | 重点是表面弹性、弯曲和布面碰撞 |
| softbody | `add_soft_grid(...)` | particle | tetrahedra + generated surface triangles / edges | 体单元压缩、拉伸、剪切后整块体积变形 | 重点是体积弹性，表面网格主要服务碰撞 |
| cable | `add_rod(...)` | rigid body | capsules + cable joints + articulation | 一串刚体段在 joint 允许下相对转动 / 拉伸 | 重点是 body transform、joint stiffness 和 articulation 结构 |

这张表里最该注意两件事:

- cloth 和 softbody 都会用到 `particle_q`，但它们的核心元素不是一回事: 一个靠三角形和弯曲边，一个靠四面体。
- cable 连最基础的状态单位都换了，不再是 particle，而是 rigid body transform。

## 3. cloth: 它是表面粒子网格，不是“薄一点的 softbody”

`newton/examples/cloth/example_cloth_hanging.py` 的教学价值，在于它先把 cloth 的身份放在 solver 之前。脚本虽然可以切 `semi_implicit / xpbd / vbd / style3d`，但不管 solver 怎么变，场景入口都在围着一张布网格写。

`builder.add_cloth_grid(...)` 的骨架很直接:

```python
builder.add_cloth_grid(
    pos=...,
    dim_x=...,
    dim_y=...,
    cell_x=...,
    cell_y=...,
    mass=...,
    fix_left=True,
    edge_ke=...,
)
```

而 `add_cloth_grid(...)` 内部会继续 handoff 到 `add_cloth_mesh(...)`，把这张布落实成几层对象:

- 粒子: 网格顶点先变成 particles。
- 三角形: 每个矩形 cell 被拆成两个 triangles。
- 弯曲边: `MeshAdjacency(...)` 从三角网格邻接关系里生成 bending edges。
- 可选 springs: 只有在 `add_springs=True` 时才额外生成 distance-like springs。

这件事最值得 beginner 先记成一句话:

```text
cloth 的核心身份是“表面粒子网格”。
triangle 和 bending edge 是它的基本语言，spring 只是可选补充，不是 cloth 身份本身。
```

这会带来三个直接后果:

- `fix_left=True` 固定的是一整列边界粒子，不是 rigid link。
- 你读源码时最该盯的是 `tri_indices`、`edge_indices`、可选 `spring`，而不是 tet connectivity。
- 同一个 cloth 例子可以换 solver，但 cloth 作为 surface object family 的身份不变。

所以更稳的说法不是“cloth 是一种 solver 支持的东西”，而是:

```text
cloth 先是一种表示，再由不同 solver 去消费这套表示。
```

## 4. softbody: 它是体粒子网格，不是“厚一点的 cloth”

`newton/examples/softbody/example_softbody_hanging.py` 则把另一件事写得很清楚: **softbody 的核心不是表面，而是体积。**

例子里真正的入口是:

```python
builder.add_soft_grid(
    pos=...,
    dim_x=12,
    dim_y=4,
    dim_z=4,
    cell_x=0.1,
    cell_y=0.1,
    cell_z=0.1,
    density=1.0e3,
    k_mu=1.0e5,
    k_lambda=1.0e5,
    k_damp=k_damp,
    fix_left=True,
)
```

`add_soft_grid(...)` 的内部顺序和 cloth 明显不同:

- 先在三维 lattice 上创建 particles。
- 再把每个 hexahedral cell 分解成 5 个 tetrahedra。
- 一边加 tet，一边记录哪些三角面是外露边界。
- 最后只为这些边界面生成 surface triangles，并可选再加 surface edges。

这里最容易误读的地方，是看到 surface triangles 之后就把它重新想成“cloth 加厚”。其实 builder 文档已经说得很直白了:

```text
generated surface triangles and optional edges are for collision purposes
```

也就是说，对 softbody 来说:

- 真正承载体积弹性的核心元素是 tetrahedra。
- 自动生成的表面三角网格主要是为了碰撞和表面表示。
- 默认情况下，这层 surface mesh 的刚度可以是零，它并不是 softbody 的主弹性来源。

所以更稳的记法应该是:

```text
cloth 是表面对象。
softbody 是体对象。
softbody 会长出一层表面碰撞皮肤，但它的核心仍然是 tet volume。
```

这也解释了为什么 cloth 和 softbody 虽然都在 particle family 里，却仍然不该被合并成同一种对象。

## 5. cable: 它不是 particle softbody，而是刚体链

`newton/examples/cable/example_cable_twist.py` 是 chapter 10 最关键的反直觉例子，因为它打破了“凡是能弯的都是粒子物体”这个误会。

它真正的 builder 入口不是 `add_particle_*`，而是:

```python
rod_bodies, _rod_joints = builder.add_rod(
    positions=cable_points,
    quaternions=cable_edge_q,
    radius=cable_radius,
    bend_stiffness=bend_stiffness,
    bend_damping=1.0e-2,
    stretch_stiffness=stretch_stiffness,
    stretch_damping=1.0e-4,
    label=f"cable_{i}",
)
```

`add_rod(...)` 的文档字符串已经直接给出定义: 它会创建一串 capsule rigid bodies，并用 cable joints 把相邻 capsule 连起来；如果需要，还会把这串 joints 包进一个 articulation。

同文件里的 `add_joint_cable(...)` 也写得很清楚:

```text
one linear (stretch) DOF + one angular (bend/twist) DOF
```

这意味着 cable 的“变形”并不是粒子弹性本身，而是:

- 每一段先是一个 rigid capsule。
- 相邻段之间通过 cable joint 允许相对拉伸、弯曲、扭转。
- 整条 cable 的柔性外观来自 joint motion 的累积。

例子后面把第一段 capsule 设成 kinematic 也再次证明了这点:

```python
builder.body_mass[first_body] = 0.0
builder.body_inv_mass[first_body] = 0.0
builder.body_inertia[first_body] = wp.mat33(0.0)
builder.body_inv_inertia[first_body] = wp.mat33(0.0)
```

被固定的是第一段 rigid body，不是某个 particle pin。

所以 chapter 10 对 cable 的最短表述必须是:

```text
cable 在 Newton 里首先是一条刚体链，
不是一条“长得像线的 particle softbody”。
```

## 6. 为什么 chapter 10 必须是 representation-first

如果你还沿用 chapter 09 的 solver-first 视角，很容易在 chapter 10 里犯两个错:

- 看到 `cloth_hanging.py` 支持多种 solver，就误以为 cloth 的身份来自 solver。
- 看到 `softbody_hanging.py` 和 `cable_twist.py` 都能走 `SolverVBD`，就误以为 softbody 和 cable 是同一种对象。

这两种看法都不稳。因为 solver 名字相同，不代表被更新的内部对象相同。

最典型的对照就是:

- `example_softbody_hanging.py` 使用 `SolverVBD`，但核心状态是 particles + tetrahedra。
- `example_cable_twist.py` 也使用 `SolverVBD`，但核心状态却是 rigid bodies + cable joints。

所以 chapter 10 最重要的一层桥接是:

```text
chapter 09 问“修正怎样组织？”
chapter 10 问“被组织、被修正的对象到底是哪一类？”
```

先把这层对象家族分清楚，你后面再看 solver、contacts、coloring、state arrays 才不会串线。

## 7. 这三种表示具体会改变什么

| 你接下来会看到的东西 | cloth | softbody | cable |
|----------------------|-------|----------|-------|
| 主要状态数组 | `particle_q` 为主 | `particle_q` 为主 | `body_q` 为主 |
| 核心拓扑 | triangles + bending edges | tetrahedra | bodies + joints |
| 固定方式 | 边界粒子置零质量 / inactive | 边界粒子置零质量 | 某段 rigid body 置零质量 / 惯量，或由 joint / articulation 组织 |
| 碰撞表面来源 | 表面网格本身 | 从 tet 边界生成 | capsule shapes 本身 |
| 你读 builder 时最该盯的函数 | `add_cloth_grid` / `add_cloth_mesh` | `add_soft_grid` | `add_rod` / `add_joint_cable` / `add_articulation` |

这张表就是 chapter 10 的真正“后果表”。

它告诉你:

- 你不能再用同一套拓扑词汇去描述三者。
- 你也不能只靠 solver 名字来判断内部对象。
- 一旦对象家族变了，builder handoff、state layout、碰撞入口和固定方式都会跟着变。

## 8. 第一遍最容易犯的四个混淆

- cloth 和 softbody 都有 particles，所以它们一定是同一种对象。
- softbody 的表面三角网格已经出现，所以它本质上就是厚 cloth。
- cable 看起来会弯，所以它一定也是 particle deformable。
- 两个例子都能走 `SolverVBD`，所以它们内部表示应该差不多。

更稳的改写应该是:

- cloth 和 softbody 同属 particle family，但分别是 surface family 与 volumetric family。
- softbody 的 surface mesh 是从 tet boundary 生成的碰撞 / 表面层，不是它的核心体表示。
- cable 的主对象是 rigid capsule chain，柔性来自 cable joints。
- solver 可以跨对象家族复用，但不会抹掉对象家族本身。

## 9. 这一章最后只带走一句话也够

```text
在 Newton 里，
cloth 是表面粒子网格，softbody 是体粒子网格，cable 是由 capsule rigid bodies 和 cable joints 组成的链。
```

如果这句话已经稳定了，下一步就去看 `source-walkthrough.md`，把这三条 builder handoff 真正串进源码。
