---
chapter: 04
title: 场景描述与 USD 解析
last_updated: 2026-04-19
source_paths:
  - newton/usd.py
  - newton/_src/sim/builder.py
  - newton/_src/utils/import_usd.py
  - newton/_src/usd/schema_resolver.py
  - newton/_src/usd/schemas.py
  - newton/_src/sim/model.py
paper_keys: []
newton_commit: 1a230702
---

# 04 场景描述与 USD 解析 源码走读

这份 walkthrough 只追 `scene input -> importer -> builder -> Model` 这条线，不展开 runtime stepping，也不把 `_src/utils/import_usd.py` 读成一趟 parser internals 大巡游。第一遍最重要的是看清：scene graph 里的 body / joint / shape / mass attrs，最后怎样压成 `Model` 的静态字段。

概念桥接放在 `principle.md`；这页只负责把那条桥对到真实入口函数、关键调用链和字段落点。

## 涉及路径

| 路径 | 角色 | 先读原因 |
|------|------|----------|
| `newton/_src/sim/builder.py` | 这条链的公开入口和收口都在这里：`ModelBuilder.add_usd()` 薄转发到 importer，见 `newton/_src/sim/builder.py:L2216-L2440`；`add_link()` / `add_joint()` / `add_shape()` 负责把 importer 产物记成 builder 字段，见 `newton/_src/sim/builder.py:L3394-L3445`, `newton/_src/sim/builder.py:L3521-L3678`, `newton/_src/sim/builder.py:L5177-L5336`；`finalize()` 再冻结成 `Model`，见 `newton/_src/sim/builder.py:L9424-L9464`, `newton/_src/sim/builder.py:L9539-L9600`, `newton/_src/sim/builder.py:L10084-L10258`。 | 一页里先拿到“入口在哪”和“最后落到哪”。 |
| `newton/_src/utils/import_usd.py` | 真正的 importer 主干。`parse_usd()` 负责读 stage、body、joint、shape、material 和 MassAPI，再把结果喂给 builder，见 `newton/_src/utils/import_usd.py:L63-L92`, `newton/_src/utils/import_usd.py:L665-L759`, `newton/_src/utils/import_usd.py:L792-L1510`, `newton/_src/utils/import_usd.py:L2393-L2992`。 | 这是 scene graph 真正开始变成 Newton 结构的地方。 |
| `newton/_src/usd/schema_resolver.py` | schema 归一化边界。`SchemaResolverManager.get_value()` 按优先级解析逻辑属性，`collect_prim_attrs()` 负责把相关 authored attrs 收集出来，见 `newton/_src/usd/schema_resolver.py:L120-L147`, `newton/_src/usd/schema_resolver.py:L186-L266`。 | 先看这里，才能把 resolver 读成“属性翻译层”而不是另一套 importer。 |
| `newton/_src/usd/schemas.py` | 具体映射表。Newton / PhysX / MuJoCo 三套命名分别怎样落到同一组逻辑键，在 `SchemaResolverNewton`、`SchemaResolverPhysx`、`SchemaResolverMjc` 里写死，见 `newton/_src/usd/schemas.py:L41-L99`, `newton/_src/usd/schemas.py:L102-L194`, `newton/_src/usd/schemas.py:L279-L380`。 | 这里最能回答“同一个物理意思为什么能从不同命名空间读出来”。 |
| `newton/_src/sim/model.py` | `Model` 静态字段目录。`shape_transform / shape_type / shape_scale`、`body_mass / body_com / body_inertia`、`joint_parent / joint_child / joint_X_p / joint_X_c` 都在这里有最终落点，见 `newton/_src/sim/model.py:L191-L239`, `newton/_src/sim/model.py:L390-L456`。 | 先认最终目的地，再回头看 importer / builder 在往哪里写。 |
| `newton/usd.py` | 不是 importer 主干，而是 chapter 04 的公共目录：把 `SchemaResolver*` 和常用 USD helper 暴露在 public surface，见 `newton/usd.py:L13-L72`。 | 只需要知道 resolver 类型是公开 API，不需要在这章继续往 helper 细节下钻。 |

## 调用链总览

1. 对这条链来说，公开入口其实是 `ModelBuilder.add_usd()`。它本身很薄，只负责把参数原样转交给 `parse_usd()`；`newton/usd.py` 更像 resolver/helper 的公共目录，不是实际导入 orchestration 的落点，见 `newton/_src/sim/builder.py:L2216-L2440`, `newton/usd.py:L13-L72`。
2. `parse_usd()` 先把 scene input 缩成 Newton 真正关心的几类信息：stage 级单位和 up-axis、PhysicsScene attrs、rigid body、joint、shape/material，以及 path 到整数索引的映射表，见 `newton/_src/utils/import_usd.py:L1512-L1573`, `newton/_src/utils/import_usd.py:L1624-L1734`, `newton/_src/utils/import_usd.py:L3218-L3233`。这里的重点不是“把 USD 树完整保留”，而是把后面会进入 builder 的局部 pose、拓扑关系和属性读出来。
3. schema resolver 在 importer 中间插了一层很重要的统一语义边界。`SchemaResolverManager.get_value()` 会按 resolver 优先顺序查询某个逻辑键，谁先给出 authored 值就用谁；都没有时，再回退到调用方默认值或 mapping 默认值，见 `newton/_src/usd/schema_resolver.py:L214-L266`。具体有哪些别名关系，则来自 `newton/_src/usd/schemas.py` 里对 `newton:*`、`physx*`、`mjc:*` 的映射，见 `newton/_src/usd/schemas.py:L41-L99`, `newton/_src/usd/schemas.py:L102-L194`, `newton/_src/usd/schemas.py:L279-L380`。
4. importer 读清 scene graph 后，并不会直接构造 `Model`。它先把 body、joint、shape 逐步记进 builder：body 通过 `add_link()` 占一个刚体槽位，joint 把 parent/child 和两侧局部 frame 写成 `joint_parent / joint_child / joint_X_p / joint_X_c`，shape 则把几何、局部 pose、scale、material-ish cfg 写成 shape 列表，见 `newton/_src/utils/import_usd.py:L665-L759`, `newton/_src/utils/import_usd.py:L792-L1510`, `newton/_src/utils/import_usd.py:L2393-L2646`, `newton/_src/sim/builder.py:L3394-L3445`, `newton/_src/sim/builder.py:L3521-L3678`, `newton/_src/sim/builder.py:L5177-L5336`。
5. 质量属性也在 builder 阶段收束，而不是留到 runtime 再推。默认路径下，shape 会按 `ShapeConfig.density` 经 `compute_inertia_shape(...)` 和 `_update_body_mass(...)` 累积到 body；如果 body 或 collider author 了 `MassAPI`，importer 会用 authored 或 fallback mass information 覆盖/补齐 `body_mass / body_com / body_inertia`，见 `newton/_src/utils/import_usd.py:L2256-L2391`, `newton/_src/utils/import_usd.py:L2648-L2698`, `newton/_src/utils/import_usd.py:L2850-L2956`, `newton/_src/sim/builder.py:L5323-L5326`, `newton/_src/sim/builder.py:L8213-L8245`。
6. `finalize()` 才是这条线真正闭环的地方。它先做 validation，再把 builder 里的 Python lists 和几何 source 冻成 `Model` 的静态数组：`shape_*`、`body_*`、`joint_*` 都在这里完成定型，见 `newton/_src/sim/builder.py:L9466-L9499`, `newton/_src/sim/builder.py:L9539-L9600`, `newton/_src/sim/builder.py:L10084-L10258`；最终字段目录则在 `newton/_src/sim/model.py:L191-L239`, `newton/_src/sim/model.py:L390-L456`。

把这一页压成一句话就是：`add_usd()` 并不是“直接读出一个 Model”，而是先让 importer 解析 scene graph 和 attrs，再让 resolver 统一命名，再让 builder 记成 Newton 的 body/joint/shape/mass 结构，最后由 `finalize()` 把这些结构冻成 `Model`。

## 数据流切片

| 切片 | 读入 | 中间接力 | 写出 | 证据 |
|------|------|----------|------|------|
| authored transform / local pose -> builder frame fields | rigid body 的 `position / rotation`，joint 的 `localPose0 / localPose1`，shape 的 `localPos / localRot`，再加 stage up-axis 与外部 `xform`。 | `parse_body()` 先把 body pose 组合成导入后的 `origin`；`resolve_joint_parent_child()` 把 joint 两侧局部 frame 解析出来；shape 侧再把 `shape_spec.localPos / localRot` 变成相对 body 或 world 的 `shape_xform`。 | builder 里的 `body_q`、`joint_X_p / joint_X_c`、`shape_transform`，并在 `finalize()` 后落成 `Model.body_q`、`Model.joint_X_p / joint_X_c`、`Model.shape_transform`。 | `newton/_src/utils/import_usd.py:L710-L714`, `newton/_src/utils/import_usd.py:L761-L788`, `newton/_src/utils/import_usd.py:L1810-L1818`, `newton/_src/utils/import_usd.py:L2454-L2458`, `newton/_src/sim/builder.py:L3431-L3436`, `newton/_src/sim/builder.py:L3600-L3602`, `newton/_src/sim/builder.py:L5271-L5273`, `newton/_src/sim/builder.py:L9540-L9541`, `newton/_src/sim/builder.py:L10178-L10180`, `newton/_src/sim/builder.py:L10197-L10198` |
| scene body / joint graph -> builder topology | articulation 里的 `articulatedBodies / articulatedJoints`，joint 的 `body0 / body1`，以及 orphan joint 关系。 | importer 先建 `body_ids`、`joint_edges`、`merged_joint_groups` 和 `path_body_map / path_joint_map`，再按可选的 topological ordering 决定 body/joint 插入顺序。builder `add_joint()` 随后把 parent-child 关系记成自己的拓扑缓存。 | builder 的 `joint_parent / joint_child`、`joint_parents / joint_children`、articulation 分组，以及最终 `Model.joint_parent / joint_child / joint_ancestor`。 | `newton/_src/utils/import_usd.py:L1717-L1729`, `newton/_src/utils/import_usd.py:L1788-L1887`, `newton/_src/utils/import_usd.py:L1952-L1986`, `newton/_src/utils/import_usd.py:L2148-L2161`, `newton/_src/sim/builder.py:L3590-L3600`, `newton/_src/sim/builder.py:L10194-L10213` |
| authored or inferred mass properties -> `body_mass / body_com / body_inertia` | physics material density、collider `MassAPI`、body `MassAPI`，以及 shape geometry 本身。 | shape 导入先把 density、friction、restitution 等收进 `ShapeConfig`；默认路径下 `add_shape()` 用 `compute_inertia_shape(...)` 和 `_update_body_mass(...)` 累积 body 质量属性；如果 body/collider author 了 `MassAPI`，importer 再用 authored 值或 `ComputeMassProperties` fallback 覆盖/补齐。 | builder 里的 `body_mass / body_com / body_inertia / body_inv_*`，以及 finalize 后的同名 `Model` arrays。 | `newton/_src/utils/import_usd.py:L1655-L1680`, `newton/_src/utils/import_usd.py:L2516-L2537`, `newton/_src/utils/import_usd.py:L2648-L2698`, `newton/_src/utils/import_usd.py:L2850-L2956`, `newton/_src/sim/builder.py:L5323-L5326`, `newton/_src/sim/builder.py:L8213-L8245`, `newton/_src/sim/builder.py:L10124-L10128`, `newton/_src/sim/builder.py:L10167-L10180` |
| authored shape / material-ish attrs -> shape arrays / config | shape type、local transform、scale、material friction/restitution/density、resolver 归一后的 `margin / gap / ke / kd`，以及 visibility / collision group 一类 authoring 结果。 | importer 先解析 material，再用 resolver 统一 `margin / gap / ke / kd` 等逻辑键，最后组装成 `ModelBuilder.ShapeConfig(...)` 并分发到各个 `add_shape_*()`。builder `add_shape()` 把 cfg 和几何参数拆到 shape 列表。 | `shape_type / shape_transform / shape_scale / shape_flags / shape_material_* / shape_gap / shape_collision_group`，并在 `finalize()` 后变成 `Model` 的 shape arrays。 | `newton/_src/utils/import_usd.py:L1655-L1680`, `newton/_src/utils/import_usd.py:L2466-L2539`, `newton/_src/utils/import_usd.py:L2542-L2645`, `newton/_src/sim/builder.py:L5287-L5307`, `newton/_src/sim/builder.py:L9539-L9600`, `newton/_src/sim/model.py:L191-L239` |

## GPU 并行切片

- `-` 这章刻意不展开 runtime kernel。`parse_usd()`、resolver、`add_link()`、`add_joint()`、`add_shape()` 的高价值信息主要是 host-side 导入与字段组织，不是并行策略。
- 唯一需要记住的冻结点是 `finalize()`：它把 builder 列表搬成 `wp.array`，并在需要时 finalize geometry source；GPU 细节到这里为止，见 `newton/_src/sim/builder.py:L9499-L9600`, `newton/_src/sim/builder.py:L10084-L10258`。

## 回指原理

| 源码点 | 对应 chapter-04 原理 | 第一遍应该怎么翻译 |
|--------|---------------------|--------------------|
| `newton/_src/sim/builder.py:L2216-L2440`, `newton/_src/utils/import_usd.py:L63-L92` | `scene input -> importer -> builder -> finalize() -> Model` 不是口号，而是一条明确的接力链。 | 先翻译成“`add_usd()` 只是入口，真正的导入工作在 `parse_usd()`，真正的定型工作在 `finalize()`”。 |
| `newton/_src/usd/schema_resolver.py:L214-L266`, `newton/_src/usd/schemas.py:L41-L99`, `newton/_src/usd/schemas.py:L102-L194`, `newton/_src/usd/schemas.py:L279-L380` | resolver 是“属性翻译边界”，不是另一套几何 parser。 | 第一遍只需要记住：不同命名空间先被翻译成统一逻辑键，importer 后面消费的是这些逻辑键，不是原始名字本身。 |
| `newton/_src/utils/import_usd.py:L2850-L2956`, `newton/_src/sim/builder.py:L5323-L5326`, `newton/_src/sim/builder.py:L8213-L8245` | `body_mass / body_com / body_inertia` 是 resolve 后的静态结果，不是“原始 scene attr 原封不动抄进来”。 | 先翻译成“shape 几何会先给 baseline mass properties，body/collider authored MassAPI 再按优先关系覆盖或补齐”。 |
| `newton/_src/sim/builder.py:L9539-L9600`, `newton/_src/sim/builder.py:L10084-L10258`, `newton/_src/sim/model.py:L191-L239`, `newton/_src/sim/model.py:L390-L456` | `Model` 是冻结后的仿真静态结构，不再保留 scene graph 的 authoring 树和命名空间噪音。 | 第一遍先把它翻译成“到 `Model` 为止，只剩 solver 真正要消费的 body/joint/shape 数组”，下一跳自然就是 `05_rigid_articulation` 和 `06_collision`。 |

带着这四个回指再看 `principle.md`，chapter 04 的桥就会比较稳：scene graph 不会直接等于 `Model`，它中间一定要经过一次解析、归一、累积和冻结。源码走读做到这里就够了。
