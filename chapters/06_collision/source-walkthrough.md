---
chapter: 06
title: 碰撞系统
last_updated: 2026-04-19
source_paths:
  - docs/concepts/collisions.rst
  - newton/_src/sim/model.py
  - newton/_src/sim/collide.py
  - newton/_src/geometry/broad_phase_common.py
  - newton/_src/geometry/broad_phase_nxn.py
  - newton/_src/geometry/broad_phase_sap.py
  - newton/_src/geometry/narrow_phase.py
  - newton/_src/geometry/contact_data.py
  - newton/_src/sim/contacts.py
paper_keys: []
newton_commit: 1a230702
---

# 06 碰撞系统 源码走读

这份 walkthrough 只追 `body/world state + shape static data -> candidate shape pairs -> Contacts` 这条线。第一遍不要把注意力放到 GJK / MPR / EPA 推导，也不要提前跳去 solver 约束数学；这一页只保住 chapter 06 最值钱的边界：哪一层在准备 shape world 表示，哪一层只负责筛 candidate pair，哪一层才真正把接触几何写进 `Contacts`。公开语义可以先回看 `docs/concepts/collisions.rst:L32-L119`, `docs/concepts/collisions.rst:L1014-L1053`, `docs/concepts/collisions.rst:L1229-L1255`。

## 先带着哪五个问题读

1. `Model.collide(state, contacts)` 到底把哪份 runtime state 交给碰撞流水线？
2. shape 在进入 broad phase 前，怎样从 `state.body_q + model.shape_transform` 变成 world-space 表示？
3. broad phase 真正写出的是什么: body pair、shape pair，还是已经成型的 contact？
4. narrow phase 怎样按 `shape_type` 分路，但又不把下游接口撕成很多套？
5. chapter 06 的终点到底是哪几个 `Contacts` 字段，chapter 07 / 08 又会从哪里继续接？

## 涉及路径

| 路径 | 角色 | 先读原因 |
|------|------|----------|
| `newton/_src/sim/model.py` | collision 的 public 边界。`Model` 持有 `shape_transform / shape_body / shape_type / shape_gap / shape_margin / shape_collision_group / shape_world` 这些静态 shape 元数据，见 `newton/_src/sim/model.py:L193-L257`, `newton/_src/sim/model.py:L317-L326`；`contacts()` / `collide()` 则把 runtime `state` 接到 `CollisionPipeline`，见 `newton/_src/sim/model.py:L951-L1003`。 | 先看清 chapter 06 的入口不是某个几何算法，而是 `Model` 怎样把 state 和 shape metadata 送进 pipeline。 |
| `newton/_src/sim/collide.py` | 整条流水线的 orchestration。`compute_shape_aabbs(...)` 把 `body_q` 和 shape 局部数据变成 world-space transform + expanded AABB，见 `newton/_src/sim/collide.py:L150-L262`；`CollisionPipeline.__init__()` 负责选 broad phase / narrow phase 并分配缓冲，见 `newton/_src/sim/collide.py:L430-L756`；`CollisionPipeline.collide()` 再把 broad phase、narrow phase 和 `Contacts` 写入串起来，见 `newton/_src/sim/collide.py:L772-L942`。 | 这一页的主线基本都在这里收口。 |
| `newton/_src/geometry/broad_phase_common.py` | broad phase 的共同边界: `check_aabb_overlap()`、`test_world_and_group_pair()`、`is_pair_excluded()`、`write_pair()`，见 `newton/_src/geometry/broad_phase_common.py:L18-L180`；`precompute_world_map()` 还把 world-aware 切片预处理好，见 `newton/_src/geometry/broad_phase_common.py:L183-L216`。 | 先把 broad phase 读成“cheap filter + candidate pair writer”，不要先陷进具体算法。 |
| `newton/_src/geometry/broad_phase_nxn.py` | all-pairs / explicit broad phase 变体。`BroadPhaseAllPairs.launch()` 负责按 AABB overlap、world/group 和排除表写 `candidate_pair`，见 `newton/_src/geometry/broad_phase_nxn.py:L217-L385`；`BroadPhaseExplicit.launch()` 则只检查预给定 pair，见 `newton/_src/geometry/broad_phase_nxn.py:L388-L459`。 | 这里最适合建立“broad phase 输出仍然只是 shape pair 名单”的感觉。 |
| `newton/_src/geometry/broad_phase_sap.py` | SAP broad phase 变体。它先按 world 分段投影和排序，再 sweep 出候选对，见 `newton/_src/geometry/broad_phase_sap.py:L397-L671`。 | 用来对照 `nxn` 看: broad phase 算法可以变，但输出边界不变。 |
| `newton/_src/geometry/narrow_phase.py` | candidate pair 的分诊台。`NarrowPhase` 持有 primitive/GJK/mesh/SDF 几类中间缓冲，见 `newton/_src/geometry/narrow_phase.py:L1406-L1694`；`launch_custom_write()` 先按 shape type 路由，再把各分支结果交给统一 writer，见 `newton/_src/geometry/narrow_phase.py:L1695-L2261`。 | 先看“pair 怎样被分路并重新收束”，不要把这一页读成算法推导页。 |
| `newton/_src/geometry/contact_data.py` | narrow phase 的统一中间表示。`ContactData` 把接触中心、法线、距离、margin、shape id、gap 等信息压成一份公共 struct，见 `newton/_src/geometry/contact_data.py:L17-L56`；`contact_passes_gap_check()` 则说明 contact 仍然要经过 gap 语义检查，见 `newton/_src/geometry/contact_data.py:L86-L117`。 | 看清 narrow phase 内部分支再多，也要回到同一种 contact geometry 语言。 |
| `newton/_src/sim/contacts.py` | chapter 06 的终点对象。`Contacts` 定义了 `rigid_contact_shape0/1`、`rigid_contact_point0/1`、`rigid_contact_normal`、`rigid_contact_margin0/1` 等最终缓冲区，见 `newton/_src/sim/contacts.py:L23-L151`；`clear()` 负责每帧重置计数，见 `newton/_src/sim/contacts.py:L221-L271`。 | 先认清最终 handoff object，前面的 broad / narrow phase 才不会像黑箱。 |

## 调用链总览

1. 公开入口先把 runtime body/world state 和静态 shape 元数据接起来。`Model.collide(state, contacts)` 本身很薄，只是确保 `CollisionPipeline` 存在并把 `state` 交过去，见 `newton/_src/sim/model.py:L977-L1003`。真正关键的输入是两半: `state.body_q` 提供 body 当前 world pose，`Model.shape_transform / shape_body / shape_type / shape_scale / shape_margin / shape_gap` 提供 shape 相对 body 的静态定义，见 `newton/_src/sim/model.py:L193-L257`, `newton/_src/sim/model.py:L317-L326`。
2. `CollisionPipeline.collide()` 的第一跳不是做 pair test，而是先把每个 shape 摆到世界里。`compute_shape_aabbs(...)` 对动态 shape 计算 `body_q[shape_body] * shape_transform`，对 `shape_body == -1` 的 world shape 直接使用 `shape_transform`；同一趟 kernel 里还会把 local AABB 变成 world AABB，并把 narrow phase 还要用的 `geom_transform` 和 `geom_data=(scale, margin)` 一起写好，见 `newton/_src/sim/collide.py:L150-L262`, `newton/_src/sim/collide.py:L820-L845`。所以 chapter 05 留下的 `body_q` 到这里第一次真的长成“shape 在 world 里现在在哪”。
3. broad phase 接下来只做一件事: 把 shape world 表示压成一张 candidate pair 名单。`CollisionPipeline` 在构造时可以选择 `explicit`、`nxn` 或 `sap` broad phase，见 `newton/_src/sim/model.py:L947-L975`, `newton/_src/sim/collide.py:L596-L678`；到了运行时，三种 broad phase 都只消费 AABB、world、collision group 和排除表，然后把 surviving shape pair 写进 `broad_phase_shape_pairs` / `broad_phase_pair_count`，见 `newton/_src/sim/collide.py:L847-L886`, `newton/_src/geometry/broad_phase_common.py:L18-L180`, `newton/_src/geometry/broad_phase_nxn.py:L217-L459`, `newton/_src/geometry/broad_phase_sap.py:L510-L671`。这一步的产物仍然只是“值得继续看”的 shape pair，不是最终 contact。
4. narrow phase 才把 candidate pair 变成真正的接触几何。`NarrowPhase.launch_custom_write()` 先用 primitive kernel 处理轻量 primitive pair，并把剩余 pair 按 `shape_type` 路由到 GJK/MPR 或 mesh/heightfield/SDF 相关缓冲，见 `newton/_src/geometry/narrow_phase.py:L1512-L1529`, `newton/_src/geometry/narrow_phase.py:L1752-L2006`。这页最重要的不是每条支路内部怎样算，而是它们最后都会收束成同一种 `ContactData` 边界: 接触中心、A->B 法线、signed distance、两侧 margin、shape id、gap 语义，见 `newton/_src/geometry/contact_data.py:L17-L56`。
5. `ContactData` 还不是 chapter 06 的终点，`Contacts` 才是。`CollisionPipeline.collide()` 会先把 `Contacts` 数组绑进 `ContactWriterData`，再让 narrow phase 直接调用 `write_contact()` 把 world-space contact geometry 变成 body-frame `point0 / point1`、`offset0 / offset1`、`normal`、`margin0 / margin1` 和 `shape0 / shape1`，见 `newton/_src/sim/collide.py:L33-L145`, `newton/_src/sim/collide.py:L888-L942`。`docs/concepts/collisions.rst` 也明确把这些 `Contacts` 数组定义成 solver 消费的统一接触数据，见 `docs/concepts/collisions.rst:L1229-L1255`。所以 chapter 06 到这里就该收口: `Contacts` 是本章终点，也是 chapter 07 / 08 继续读 contact math 与 solver consumption 的共同起点。

## 数据流切片

| 切片 | 读入 | 中间接力 | 写出 | 证据 |
|------|------|----------|------|------|
| body/world pose + shape transform -> shape world representation | `state.body_q`，以及 `Model.shape_transform / shape_body / shape_type / shape_scale / shape_margin / shape_gap / shape_collision_aabb_lower / shape_collision_aabb_upper`。 | `Model.collide()` 把 `state` 交给 `CollisionPipeline.collide()`；`compute_shape_aabbs(...)` 再把 `body_q` 和 shape 局部 pose 合成 `X_ws`，并在同一趟 kernel 里写 world-space AABB、`geom_transform`、`geom_data=(scale, margin)`。 | `NarrowPhase.shape_aabb_lower / shape_aabb_upper`，以及 narrow phase 后续直接消费的 `geom_transform`、`geom_data`。 | `newton/_src/sim/model.py:L193-L257`, `newton/_src/sim/model.py:L317-L326`, `newton/_src/sim/model.py:L977-L1003`, `newton/_src/sim/collide.py:L150-L262`, `newton/_src/sim/collide.py:L820-L845` |
| AABB / world / group filtering -> candidate pairs | 每个 shape 的 world AABB，再加上 `shape_world`、`shape_collision_group`、显式排除 pair 或显式允许 pair。 | `check_aabb_overlap()` 做便宜的 overlap test，`test_world_and_group_pair()` 处理 world/group 过滤，`is_pair_excluded()` 处理禁碰表；`BroadPhaseAllPairs`、`BroadPhaseSAP`、`BroadPhaseExplicit` 只是用不同办法找到满足这些条件的 shape pair，然后统一调用 `write_pair()`。 | `CollisionPipeline.broad_phase_shape_pairs` 和 `CollisionPipeline.broad_phase_pair_count`，也就是 narrow phase 的 candidate shape pair 名单。 | `newton/_src/geometry/broad_phase_common.py:L18-L180`, `newton/_src/geometry/broad_phase_common.py:L183-L216`, `newton/_src/geometry/broad_phase_nxn.py:L217-L459`, `newton/_src/geometry/broad_phase_sap.py:L510-L671`, `newton/_src/sim/collide.py:L847-L886` |
| candidate pair -> shape-specific contact geometry | `candidate_pair`、`candidate_pair_count`、`shape_types`、`geom_data`、`geom_transform`、`shape_source`、`shape_gap`，以及 mesh / SDF / heightfield 所需的静态几何缓存。 | `NarrowPhase.launch_custom_write()` 先清内部 counters；primitive kernel 先吃掉解析几何就能处理的 pair，并把剩余 pair 路由到 GJK/MPR、mesh-plane、mesh-triangle、mesh-mesh、SDF/hydroelastic 等分支。对这页来说，真正要守住的边界是: 各条分支最后都把结果整理成 `ContactData`，而不是让下游继续理解每种算法的内部状态。 | 统一的 `ContactData` 语义: `contact_point_center`、`contact_normal_a_to_b`、`contact_distance`、`margin_a / margin_b`、`shape_a / shape_b`、`gap_sum`。 | `newton/_src/geometry/narrow_phase.py:L1406-L1529`, `newton/_src/geometry/narrow_phase.py:L1695-L2006`, `newton/_src/geometry/contact_data.py:L17-L56`, `newton/_src/geometry/contact_data.py:L86-L117` |
| contact geometry -> `Contacts` fields | `ContactData`，再加上 `state.body_q`、`shape_body`、`shape_gap` 和已经分配好的 `Contacts` 缓冲。 | `CollisionPipeline.contacts()` 先分配 `Contacts`；`CollisionPipeline.collide()` 再组装 `ContactWriterData`。`write_contact()` 把 `ContactData` 中的 world-space 几何关系变成 body-frame `point0 / point1` 和 friction offset，写入 `shape0 / shape1`、`normal`、`margin0 / margin1`、计数以及可选的 per-contact properties / sort key。 | `Contacts.rigid_contact_count`、`rigid_contact_shape0 / shape1`、`rigid_contact_point0 / point1`、`rigid_contact_offset0 / offset1`、`rigid_contact_normal`、`rigid_contact_margin0 / margin1`。这些数组就是后续章节继续消费的统一接触记录。 | `newton/_src/sim/collide.py:L33-L145`, `newton/_src/sim/collide.py:L730-L756`, `newton/_src/sim/collide.py:L804-L942`, `newton/_src/sim/contacts.py:L23-L151`, `newton/_src/sim/contacts.py:L221-L271`, `docs/concepts/collisions.rst:L1229-L1255` |

这页到这里就停。第一次读 collision path，只要把 `body/world state -> shape world representation -> candidate shape pairs -> ContactData -> Contacts` 这条桥读顺就够了；一旦 narrow phase 已经把结果写进 `Contacts`，chapter 06 的源码边界就闭环了，下一跳自然是 chapter 07 的 contact math 和 chapter 08 的 solver consumption。
