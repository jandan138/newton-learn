---
chapter: 03
title: 数学与几何基础
last_updated: 2026-04-19
source_paths:
  - newton/_src/sim/builder.py
  - newton/_src/math/spatial.py
  - newton/_src/sim/articulation.py
  - newton/_src/geometry/inertia.py
  - newton/_src/geometry/types.py
  - newton/_src/geometry/support_function.py
  - newton/_src/geometry/kernels.py
  - newton/_src/geometry/collision_primitive.py
paper_keys: []
newton_commit: 1a230702
---

# 03 数学与几何基础 深读锚点版

这份 deep walkthrough 把 chapter 03 里最密集的源码锚点集中起来。第一次读本章时，建议先读完 `source-walkthrough.md`，把 `frame -> transform -> spatial -> shape -> inertia` 的主线读顺，再回来逐段校验具体实现。

## Fast Deep Index

| ID | repo@commit path | symbol | why it matters |
|----|------------------|--------|----------------|
| D1 | `newton@1a230702 newton/_src/sim/builder.py` | builder fields, `add_joint`, `add_shape`, `_update_body_mass`, `finalize` | local frame、shape bookkeeping 和质量累积的起点 |
| D2 | `newton@1a230702 newton/_src/math/spatial.py` | `velocity_at_point`, `transform_twist`, `transform_wrench` | spatial quantity 的最小词典 |
| D3 | `newton@1a230702 newton/_src/sim/articulation.py` | `eval_single_articulation_fk`, `jcalc_motion_subspace`, `eval_articulation_jacobian` | transform / spatial 规则怎样进入 articulation |
| D4 | `newton@1a230702 newton/_src/geometry/types.py` | `GeoType`, `Mesh` | shape representation 的目录图 |
| D5 | `newton@1a230702 newton/_src/geometry/support_function.py` | `support_map`, `extract_shape_data` | convex-style query 的统一入口 |
| D6 | `newton@1a230702 newton/_src/geometry/kernels.py` | particle-shape local query dispatch | `GeoType` 怎样进入具体查询 |
| D7 | `newton@1a230702 newton/_src/geometry/inertia.py` | `transform_inertia`, `compute_inertia_shape` | geometry 到 mass property 的桥 |
| D8 | `newton@1a230702 newton/_src/geometry/collision_primitive.py` | primitive collision leaves | analytic primitive query 的叶子函数 |

## Exact Handoff Trace

### 1. builder 先记局部关系，不先记 world 结论

- 字段目录：`newton/_src/sim/builder.py:883-929`, `newton/_src/sim/builder.py:1008-1049`
- joint 写入：`newton/_src/sim/builder.py:3590-3668`
- shape 写入：`newton/_src/sim/builder.py:5265-5326`
- finalize 到 `Model`：`newton/_src/sim/builder.py:9539-9600`, `newton/_src/sim/builder.py:10178-10258`
- exact handoff：
  - `shape_transform / shape_body / shape_type / shape_scale` 先在 builder 上累积
  - `joint_parent / joint_child / joint_X_p / joint_X_c / joint_q_start / joint_qd_start` 也先在 builder 上累积
  - `finalize()` 再把这些 lists 冻成 `Model` arrays

### 2. 同一套 transform chain 同时服务 articulation 与 geometry query

- articulation FK：`newton/_src/sim/articulation.py:190-374`
- Jacobian / motion subspace：`newton/_src/sim/articulation.py:889-1125`
- geometry local query：`newton/_src/geometry/kernels.py:1044-1140`
- exact handoff：
  - articulation 里先算 `X_wpj -> X_wcj -> X_wc`
  - geometry query 里先算 `X_ws -> X_sw -> x_local`
  - 两边都在复用 builder 留下的 local frame 账本

### 3. spatial quantity helper 的精确入口

- helper 定义：`newton/_src/math/spatial.py:53-130`
- articulation helper 复用：`newton/_src/sim/articulation.py:14-33`, `newton/_src/sim/articulation.py:347-374`, `newton/_src/sim/articulation.py:900-955`
- exact handoff：
  - `velocity_at_point()` 处理 point velocity
  - `transform_twist()` / `transform_wrench()` 处理 frame change 但保持 Newton 的 layout 语义
  - Jacobian / motion subspace 后面直接复用这些 helper

### 4. `GeoType` 怎样变成 query dispatch

- type 目录：`newton/_src/geometry/types.py:38-99`
- support map：`newton/_src/geometry/support_function.py:107-304`
- shape data extraction：`newton/_src/geometry/support_function.py:351-393`
- concrete query dispatch：`newton/_src/geometry/kernels.py:1061-1140`
- primitive analytic leaves：`newton/_src/geometry/collision_primitive.py:111-177`, `newton/_src/geometry/collision_primitive.py:366-439`
- exact handoff：
  - `GeoType` 先把 primitive / explicit / mesh 风格区分开
  - `support_map()` 提供 convex-style furthest-point query
  - `geometry/kernels.py` 再按 type 分流到 SDF、mesh、plane、heightfield
  - primitive pair 更深处会落到 analytic leaf 函数

### 5. geometry 到 mass / inertia 的桥

- inertia transform：`newton/_src/geometry/inertia.py:436-471`
- shape inertia computation：`newton/_src/geometry/inertia.py:474-605`
- builder mass accumulation：`newton/_src/sim/builder.py:5323-5326`, `newton/_src/sim/builder.py:8213-8245`
- exact handoff：
  - `compute_inertia_shape()` 先给 shape-local `mass / com / inertia`
  - `com_body = wp.transform_point(xform, c)` 把 shape COM 拉到 body frame
  - `_update_body_mass()` 再做 COM 合并和平行轴项更新

## Optional Branches

### Branch A: Jacobian / mass matrix 支线

- first pass 可跳过：`newton/_src/sim/articulation.py:958-1239`
- 关键原因：chapter 03 第一遍只需要知道 spatial helper 会继续进 Jacobian，不需要现在就展开矩阵层

### Branch B: mesh / convex / heightfield 的更多查询细节

- first pass 可跳过：`support_map()` 里更长的 CONVEX_MESH / TRIANGLE 分支，和 `geometry/kernels.py` 里的 mesh sampling 细节
- 关键原因：本章先记住 dispatch key，不必提前读完所有叶子分支

### Branch C: primitive analytic leaves

- first pass 可跳过：`collision_primitive.py` 里 plane-box、capsule-capsule 等每个具体叶子公式
- 关键原因：chapter 03 只需要知道 primitive 最后会落到这种 analytic leaf

## Verification Anchors

| 想验证的 claim | 直接打开哪里 |
|----------------|--------------|
| `shape_transform / joint_X_p / joint_X_c / body_com` 是局部关系账本 | `newton/_src/sim/builder.py:883-929`, `newton/_src/sim/builder.py:1008-1049`, `newton/_src/sim/builder.py:3590-3668` |
| articulation 和 geometry query 在复用同一套 transform chain | `newton/_src/sim/articulation.py:224-374`, `newton/_src/geometry/kernels.py:1049-1060` |
| spatial helper 主要在做 frame-aware 6D 量转换 | `newton/_src/math/spatial.py:53-130` |
| `GeoType` 会直接决定查询路径 | `newton/_src/geometry/types.py:38-99`, `newton/_src/geometry/kernels.py:1061-1140` |
| inertia 是 geometry 进入 dynamics 的桥 | `newton/_src/geometry/inertia.py:474-605`, `newton/_src/sim/builder.py:8213-8245` |
