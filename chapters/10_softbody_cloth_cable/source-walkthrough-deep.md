---
chapter: 10
title: 软体、布料与 Cable 源码走读深读
last_updated: 2026-04-21
source_paths:
  - newton/examples/cloth/example_cloth_hanging.py
  - newton/examples/softbody/example_softbody_hanging.py
  - newton/examples/cable/example_cable_twist.py
  - newton/_src/sim/builder.py
paper_keys: []
newton_commit: 1a230702
---

# 10 软体、布料与 Cable 深读锚点版

这份 deep walkthrough 保留精确 cross-repo anchors、可选分支和 verification-heavy trace。第一次读 chapter 10 时，建议先把 `source-walkthrough.md` 读完，再回来用这一页把 builder handoff 钉死。

## Fast Deep Index

| ID | repo@commit path | symbol | why it matters |
|----|------------------|--------|----------------|
| D1 | `newton@1a230702 newton/examples/cloth/example_cloth_hanging.py` | `common_params`, `builder.add_cloth_grid` | cloth 入口从一开始就是 2D surface-grid contract |
| D2 | `newton@1a230702 newton/_src/sim/builder.py` | `add_cloth_grid` | 先造平面顶点和三角索引，再 handoff 给 `add_cloth_mesh` |
| D3 | `newton@1a230702 newton/_src/sim/builder.py` | `add_cloth_mesh` | cloth 真正落地成 particles + triangles + bending edges (+ optional springs) |
| D4 | `newton@1a230702 newton/examples/softbody/example_softbody_hanging.py` | `builder.add_soft_grid`, `builder.color` | softbody 入口直接是 3D grid，而不是“厚一点的 cloth” |
| D5 | `newton@1a230702 newton/_src/sim/builder.py` | `add_soft_grid` | 先加 particles 和 tetrahedra，再生成 boundary surface mesh |
| D6 | `newton@1a230702 newton/examples/cable/example_cable_twist.py` | `create_cable_geometry_with_turns`, `builder.add_rod` | cable 先走 geometry path，再交给 rigid-body rod builder |
| D7 | `newton@1a230702 newton/_src/sim/builder.py` | `add_rod`, `add_rod_graph` | 每条 edge 变 capsule body，再通过 cable joints 组织成链 |
| D8 | `newton@1a230702 newton/_src/sim/builder.py` | `add_joint_cable`, `add_articulation` | cable 的柔性连接和 articulation-safe 组织在这里真正落地 |

## Exact Handoff Trace

### 1. Builder 已经先把家族分开

- cloth 入口：`newton/examples/cloth/example_cloth_hanging.py:66-80`, `97-118`
- softbody 入口：`newton/examples/softbody/example_softbody_hanging.py:38-68`
- cable 入口：`newton/examples/cable/example_cable_twist.py:50-107`, `159-196`

精确阅读点：

- cloth 例子在 `common_params` 里先说二维 grid、边界固定和 surface-style 参数，然后调用 `builder.add_cloth_grid(...)`。
- softbody 例子从一开始就多出 `dim_z / cell_z`，直接调用 `builder.add_soft_grid(...)`。
- cable 例子先用 `create_cable_geometry_with_turns(...)` 生成折线路径和 per-edge quaternion，再调用 `builder.add_rod(...)`。
- 三条路都在 `builder.color(...)` 和 `builder.finalize()` 之前就已经分家，所以 chapter 10 的 first split 发生在 builder，不发生在 solver。

这一层要钉住的事实是：chapter 10 的 main comparison 不是“谁更会解”，而是 builder 先把对象家族造成了什么。

### 2. Cloth: `add_cloth_grid -> add_cloth_mesh`

- example：`newton/examples/cloth/example_cloth_hanging.py:66-80`, `97-114`
- builder：`newton/_src/sim/builder.py:7488-7606`, `7608-7740`

关键 handoff：

- `common_params` 已经把 cloth 写成二维 surface-grid contract：`dim_x / dim_y / cell_x / cell_y / fix_left`。
- `add_cloth_grid(...)` 先在 `builder.py:7542-7557` 生成平面 `vertices` 和 triangle `indices`，再在 `builder.py:7565-7587` 把这批数据 handoff 给 `add_cloth_mesh(...)`。
- `add_cloth_mesh(...)` 真正把 cloth 落成三层核心对象：`add_particles(...)` 在 `builder.py:7671-7684`，`add_triangles(...)` 在 `builder.py:7686-7699`，`MeshAdjacency(...) -> add_edges(...)` 在 `builder.py:7709-7723`。
- 边界固定不是通过 joint，而是在 `builder.py:7589-7606` 里把边界粒子的 `particle_flag` 清成非 active，并把 `particle_mass` 设成 `0.0`。
- 如果走 XPBD cloth branch，`add_springs=True` 才会在 `builder.py:7725-7739` 额外生成 spring graph；这说明 spring 是 cloth 的可选支路，不是 cloth 身份本体。

这里最该稳住的一句话是：cloth 的 handoff 主线不是 “solver -> cloth”，而是 `grid -> mesh -> particles + triangles + bending edges`。

### 3. Softbody: `add_soft_grid -> particles -> tetrahedra -> generated surface mesh`

- example：`newton/examples/softbody/example_softbody_hanging.py:38-68`
- builder：`newton/_src/sim/builder.py:7843-8005`

关键 handoff：

- `add_soft_grid(...)` 的参数层已经暴露 volume contract：`dim_x / dim_y / dim_z` 和 `cell_x / cell_y / cell_z` 同时出现。
- builder 先在 `builder.py:7917-7937` 把 `(dim_x + 1) * (dim_y + 1) * (dim_z + 1)` 个节点加入 `particle` state；边界固定仍然是把对应粒子的质量设成 `0.0`。
- 之后 `faces` ledger 和 `add_tet(...)` 在 `builder.py:7939-7956` 一边调用 `self.add_tetrahedron(...)`，一边用开闭面字典记录哪些面是体网格边界。
- 每个 hexahedral cell 在 `builder.py:7961-7985` 被分解成 5 个 tetrahedra；奇偶分支只是在选哪一组 5-tet pattern，不会把主表示从 volume 改回 surface。
- 直到 `builder.py:7987-8005`，builder 才把剩下的 boundary faces 生成 surface triangles，并可选地加 surface edges。
- 这段 helper 自己的 docstring 也直接写明：`builder.py:7908-7911` 里的 generated surface triangles and optional edges are for collision purposes。

这一层要钉住的事实是：softbody 的主表示是 tetrahedral volume，surface mesh 是从 tet boundary 长出来的后续表层。

### 4. Cable: geometry path -> `add_rod` -> `add_joint_cable` -> `add_articulation`

- example geometry：`newton/examples/cable/example_cable_twist.py:50-107`
- example build path：`newton/examples/cable/example_cable_twist.py:159-207`
- builder：`newton/_src/sim/builder.py:6344-6529`, `6531-6912`, `4162-4238`, `1965-2044`

关键 handoff：

- 例子不会直接“加一条 soft cable”。它先在 `create_cable_geometry_with_turns(...)` 里生成 zigzag polyline points，再用 `create_parallel_transport_cable_quaternions(...)` 给每个 segment 准备朝向。
- `add_rod(...)` 的 docstring 在 `builder.py:6358-6363` 直接定义 rod 是 `capsule bodies connected by cable joints`，所以 cable family 的语言一开始就是 rigid body language。
- `add_rod(...)` 在 `builder.py:6457-6477` 先把 centerline 变成线性 edge list，然后委托给 `add_rod_graph(...)`。
- `add_rod_graph(...)` 在 `builder.py:6640-6705` 为每条 edge 创建一个 capsule rigid body：`add_link(...)` 决定 body frame，`add_shape_capsule(...)` 把几何挂到 local `+Z` 上。
- 接下来它在 `builder.py:6754-6779` 或 `6817-6845` 里按共享节点创建 joints；真正的 joint 类型是 `add_joint_cable(...)`。
- `add_joint_cable(...)` 在 `builder.py:4178-4233` 明说 cable joint 只有两层 DOF：one linear (stretch) DOF + one angular (bend/twist) DOF。
- 默认情况下，`add_rod(...)` 还会在 `builder.py:6479-6482` 把整串 joints 交给 `add_articulation(...)`；而 `add_articulation(...)` 在 `builder.py:1971-2015` 又要求 joint list 必须 contiguous and monotonically increasing。
- 例子里固定 cable 头段的方法也完全暴露了刚体身份：`example_cable_twist.py:177-183` 直接把 `body_mass / body_inv_mass / body_inertia / body_inv_inertia` 清零，而不是去改 particle 质量。

这一层要钉住的事实是：cable 的 handoff 主线不是 `particles -> elastic volume/surface`，而是 `geometry path -> capsule bodies -> cable joints -> articulation-safe chain`。

### 5. Cross-family state ledger

- cloth state：`newton/examples/cloth/example_cloth_hanging.py:111-150`
- softbody state：`newton/examples/softbody/example_softbody_hanging.py:67-86`
- cable state：`newton/examples/cable/example_cable_twist.py:177-207`, `231-237`

| family | builder 真正造出的核心对象 | 固定/驱动单位 | finalize 之后最先盯什么 |
|--------|----------------------------|---------------|--------------------------|
| cloth | particles + triangles + bending edges (+ optional springs) | 边界粒子 | `particle_q`、`tri_indices`、`edge_indices` |
| softbody | particles + tetrahedra + generated surface triangles/edges | 体网格边界粒子 | `particle_q`、tet connectivity、generated surface mesh |
| cable | capsule rigid bodies + cable joints (+ articulation) | 刚体段与 joint anchors | `body_q`、joint list、articulation grouping |

这张表最重要的作用不是背字段名，而是提醒你：cloth 和 softbody 虽然都盯 `particle_q`，但一个的核心拓扑是 triangle/edge，另一个的核心拓扑是 tetrahedra；cable 则连 state 单位都切到了 rigid bodies。

## Optional Branches

### Branch A: `add_springs` cloth branch

- example 开关：`newton/examples/cloth/example_cloth_hanging.py:97-102`
- builder 落点：`newton/_src/sim/builder.py:7725-7739`

这条分支值得看，但只该当成 optional augmentation。它解释的是 XPBD cloth 为什么会额外长出 spring graph，不会改写 chapter 10 的主定义：cloth 仍然先是 surface particle mesh。

### Branch B: 5-tet decomposition / generated surface edges softbody branch

- 5-tet 说明：`newton/_src/sim/builder.py:7872-7876`
- parity 分解：`newton/_src/sim/builder.py:7973-7985`
- surface edges：`newton/_src/sim/builder.py:7993-8005`

这条分支帮助你核对两件事：每个 cell 为什么是 5-tet 而不是单一固定切法，以及 generated surface edges 为什么默认更像 collision robustness helper，而不是 softbody 主弹性来源。

### Branch C: `add_rod_graph` / stiffness normalization / closed loop cable branch

- graph builder：`newton/_src/sim/builder.py:6531-6912`
- stiffness normalization：`newton/_src/sim/builder.py:6754-6760`, `6817-6823`, `6509-6511`
- closed loop：`newton/_src/sim/builder.py:6484-6527`, `6857-6863`

这条分支解释 cable family 的三个 second-pass 细节：

- `add_rod_graph(...)` 说明 cable builder 不只支持直链，也支持 junction / graph topology。
- stiffness 会按 segment length 归一化，所以 API 参数更像 material-like stiffness，而不是直接的 per-joint raw stiffness。
- closed loop 不能简单塞进 articulation tree；builder 会额外加 loop-closing joint，并明确区分它和 articulation-safe joint forest。

## Verification Anchors

| 想验证的 claim | 直接打开哪里 |
|----------------|--------------|
| chapter 10 的家族分裂发生在 builder，而不是 solver | `newton/examples/cloth/example_cloth_hanging.py:66-80`, `111-118`; `newton/examples/softbody/example_softbody_hanging.py:38-68`; `newton/examples/cable/example_cable_twist.py:50-107`, `159-196` |
| cloth 的 builder handoff 是 `add_cloth_grid -> add_cloth_mesh -> particles + triangles + bending edges` | `newton/_src/sim/builder.py:7542-7587`, `7671-7723` |
| cloth 的 spring graph 只是 optional branch | `newton/examples/cloth/example_cloth_hanging.py:97-102`; `newton/_src/sim/builder.py:7725-7739` |
| softbody 的核心是 tetrahedra，surface mesh 是后生成的 | `newton/_src/sim/builder.py:7939-8005` |
| softbody 的 generated surface triangles / edges 默认是 collision-purpose layer | `newton/_src/sim/builder.py:7908-7911`, `7987-8005` |
| cable 的主表示是 capsule rigid-body chain，不是 particle deformable | `newton/examples/cable/example_cable_twist.py:159-183`; `newton/_src/sim/builder.py:6358-6363`, `6640-6705` |
| cable joint 的主语言是 stretch DOF + bend/twist DOF | `newton/_src/sim/builder.py:4178-4233` |
| cable 的 joints 默认还会被组织进 articulation，而不是散落的 pairwise joints | `newton/_src/sim/builder.py:6479-6482`, `1965-2015` |
| cross-family state ledger 里，cloth/softbody 先盯 `particle_q`，cable 先盯 `body_q` | `newton/examples/cloth/example_cloth_hanging.py:146-150`; `newton/examples/softbody/example_softbody_hanging.py:82-86`; `newton/examples/cable/example_cable_twist.py:177-183`, `199-207`, `231-237` |
