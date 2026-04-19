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

# 04 场景描述与 USD 解析 深读锚点版

这份 deep walkthrough 把 chapter 04 里密集的 importer / resolver / finalize 锚点集中起来。第一次读本章时，建议先读完 `source-walkthrough.md`，把 `scene input -> importer -> builder -> Model` 主线读顺，再回来逐段核对实现细节。

## Fast Deep Index

| ID | repo@commit path | symbol | why it matters |
|----|------------------|--------|----------------|
| D1 | `newton@1a230702 newton/_src/sim/builder.py` | `ModelBuilder.add_usd` | public entry 到 importer 的接缝 |
| D2 | `newton@1a230702 newton/usd.py` | public resolver exports | `SchemaResolver*` 怎样进入 public surface |
| D3 | `newton@1a230702 newton/_src/utils/import_usd.py` | `parse_usd`, `add_body`, `parse_body`, `resolve_joint_parent_child`, `parse_joint` | importer 主干与 body/joint handoff |
| D4 | `newton@1a230702 newton/_src/usd/schema_resolver.py` | `SchemaResolverManager.get_value` | 逻辑键解析顺序 |
| D5 | `newton@1a230702 newton/_src/usd/schemas.py` | `SchemaResolverNewton`, `SchemaResolverPhysx`, `SchemaResolverMjc` | authored names 到逻辑键的映射表 |
| D6 | `newton@1a230702 newton/_src/utils/import_usd.py` | shape parsing loop | shape/material attrs 怎样变成 `ShapeConfig` |
| D7 | `newton@1a230702 newton/_src/utils/import_usd.py` | MassAPI override path | authored 与 geometry-derived mass property 的优先关系 |
| D8 | `newton@1a230702 newton/_src/sim/builder.py`, `newton/_src/sim/model.py` | `finalize`, `Model` fields | `builder -> Model` 的冻结终点 |

## Exact Handoff Trace

### 1. public entry: `add_usd()` 和 `newton.usd`

- `add_usd()` wrapper：`newton/_src/sim/builder.py:2216-2440`
- public resolver exports：`newton/usd.py:13-72`
- exact handoff：
  - `add_usd()` 只是把参数转给 `parse_usd()`
  - `newton.usd` 把 `SchemaResolverNewton / Physx / Mjc` 暴露在 public surface

### 2. importer 如何抓 body / joint / articulation graph

- `parse_usd()` 入口：`newton/_src/utils/import_usd.py:63-92`
- body parse helpers：`newton/_src/utils/import_usd.py:665-760`
- joint parent/child resolve：`newton/_src/utils/import_usd.py:761-824`
- articulation/body collection：`newton/_src/utils/import_usd.py:1717-1849`
- joint parsing：`newton/_src/utils/import_usd.py:792-1063`
- exact handoff：
  - body path 先经过 `parse_body()` / `add_body()` 进入 `builder.add_link()`
  - joint path 先经过 `resolve_joint_parent_child()` 变成 `parent_id / child_id / parent_tf / child_tf`
  - articulation loop 再组织 `articulatedBodies`、`joint_descriptions`、`body_ids`

### 3. resolver 如何把 authored names 统一成逻辑键

- base resolver + manager：`newton/_src/usd/schema_resolver.py:46-147`, `newton/_src/usd/schema_resolver.py:186-266`
- Newton mapping：`newton/_src/usd/schemas.py:41-99`
- PhysX mapping：`newton/_src/usd/schemas.py:102-194`
- MuJoCo mapping：`newton/_src/usd/schemas.py:279-361`
- exact handoff：
  - importer 通过 `SchemaResolverManager.get_value()` 按 resolver 优先级查值
  - 每个 resolver 只负责把 authored attr 名字映射到统一逻辑键
  - 调用方可以继续用 `margin`、`gap`、`armature`、`self_collision_enabled` 这种统一键名

### 4. shape / material / mass info 怎样写进 builder

- shape parsing loop：`newton/_src/utils/import_usd.py:2393-2709`
- body-level mass override：`newton/_src/utils/import_usd.py:2850-2992`
- builder shape sink：`newton/_src/sim/builder.py:5177-5336`
- exact handoff：
  - shape attrs 和 material attrs 会先被压成 `ShapeConfig`
  - 再按具体 USD shape type 调到 `builder.add_shape_box / sphere / mesh / plane ...`
  - authored `MassAPI` 会在 body 级别覆盖或补齐 `body_mass / body_com / body_inertia`

### 5. finalize 到 `Model`

- builder finalize top：`newton/_src/sim/builder.py:9424-9639`
- body/joint freeze：`newton/_src/sim/builder.py:10167-10258`
- `Model` shape fields：`newton/_src/sim/model.py:191-256`
- `Model` joint/body fields：`newton/_src/sim/model.py:390-572`
- exact handoff：
  - builder lists 被统一搬到 `Model.shape_* / body_* / joint_*`
  - `Model` 不再保留 scene graph 树结构，只保留 runtime 要消费的静态数组

## Optional Branches

### Branch A: mesh approximation / remeshing

- first pass 可跳过：`newton/_src/utils/import_usd.py:2617-2709`
- 关键原因：chapter 04 第一遍只需要知道 mesh shape 会进入 `builder.add_shape_mesh()`，不必现在展开所有 approximation policy

### Branch B: custom attributes and schema attr collection

- first pass 可跳过：`usd.get_custom_attribute_values(...)` 与 `collect_schema_attrs` 的更长支线
- 关键原因：这些更偏扩展机制，不影响本章的主 handoff

### Branch C: body mass fallback 的边缘分支

- first pass 可跳过：`ComputeMassProperties` 失败后的密集回退逻辑和默认惯量补丁
- 关键原因：本章先守住 authored-vs-derived 的优先关系就够了

## Verification Anchors

| 想验证的 claim | 直接打开哪里 |
|----------------|--------------|
| `add_usd()` 只是薄入口 | `newton/_src/sim/builder.py:2410-2440` |
| body / joint handoff 先落在 importer helper | `newton/_src/utils/import_usd.py:665-824` |
| resolver 解决的是命名归一化，不是另一套 importer | `newton/_src/usd/schema_resolver.py:214-266`, `newton/_src/usd/schemas.py:41-194`, `newton/_src/usd/schemas.py:279-361` |
| shape attrs 会先被压成 `ShapeConfig` 再进 builder | `newton/_src/utils/import_usd.py:2453-2645`, `newton/_src/sim/builder.py:5177-5336` |
| authored MassAPI 会覆盖或补齐 geometry-derived mass properties | `newton/_src/utils/import_usd.py:2850-2992` |
| `finalize()` 才是 scene graph 到 `Model` 的真正封口点 | `newton/_src/sim/builder.py:9539-9600`, `newton/_src/sim/builder.py:10167-10258`, `newton/_src/sim/model.py:191-256` |
