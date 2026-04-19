---
chapter: 03
title: 数学与几何基础
last_updated: 2026-04-18
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

# 03 数学与几何基础 源码走读

这份 walkthrough 不再像 chapter 02 那样按 CLI 或 runtime loop 追 orchestration，而是把 chapter 03 里的四条词链钉到真实代码：`frame / transform`、`spatial quantity`、`shape representation`、`inertia / mass property`。阅读重点不是枚举 `_src/math/` 或 `_src/geometry/` 的全部 helper，而是先看“一个概念在代码里怎样接成下一站”。下面所有 `path:line` 引用都以 front matter 里的 `newton_commit: 1a230702` 为准；如果你对照的是别的 revision，行号可能会漂移。

## 涉及路径

| 路径 | 角色 | 先读原因 |
|------|------|----------|
| `newton/_src/sim/builder.py` | chapter 03 的总桥。`shape_transform`、`body_com`、`joint_X_p`、`joint_X_c` 这些词先在 builder 里长成真实字段，见 `newton/_src/sim/builder.py:L886-L887`, `newton/_src/sim/builder.py:L1016-L1017`, `newton/_src/sim/builder.py:L1046-L1049`；`add_shape()` 又把 shape 配置直接接到质量属性上，见 `newton/_src/sim/builder.py:L5265-L5326`；`finalize()` 再把它们冻结成 `Model` arrays，见 `newton/_src/sim/builder.py:L9539-L9576`, `newton/_src/sim/builder.py:L10178-L10198`。 | 一页里同时拿到“局部关系怎样被记录”和“最后落到模型的哪里”。 |
| `newton/_src/math/spatial.py` | spatial vocabulary 的最小词典。`velocity_at_point()`、`transform_twist()`、`transform_wrench()` 分别把“偏移点速度”“twist 换 frame”“wrench 换 frame”包成 Newton layout 下的 helper，见 `newton/_src/math/spatial.py:L54-L130`。 | 先认这三个 helper，后面看 articulation 时就不会把 spatial 量读成黑魔法。 |
| `newton/_src/sim/articulation.py` | `frame / transform / spatial quantity` 真正落地的地方。关节局部锚点怎样拼成 `X_wc`、body twist 怎样在 origin / COM / point 之间换读、motion subspace / Jacobian 怎样复用 spatial helper，都在这里，见 `newton/_src/sim/articulation.py:L15-L33`, `newton/_src/sim/articulation.py:L224-L375`, `newton/_src/sim/articulation.py:L900-L955`, `newton/_src/sim/articulation.py:L1033-L1044`, `newton/_src/sim/articulation.py:L1129-L1192`。 | 这是 chapter 03 词汇第一次变成 articulation 可读代码的地方。 |
| `newton/_src/geometry/inertia.py` | shape 级质量属性的来源。`compute_inertia_shape()` 按 `GeoType` 选择局部质量 / 质心 / 惯量公式或 mesh fallback，`transform_inertia()` 负责换 frame 与平行轴项，见 `newton/_src/geometry/inertia.py:L436-L605`。 | 这里把“geometry 是什么”压缩成“动起来有多重、绕哪更难转”。 |
| `newton/_src/geometry/types.py` | `GeoType` 这张目录图。primitive vs explicit 的第一层分法在 `GeoType.is_primitive` / `is_explicit`，见 `newton/_src/geometry/types.py:L38-L98`；`Mesh.build_sdf()` 说明 mesh 还能继续长成 SDF 表示，见 `newton/_src/geometry/types.py:L720-L785`。 | 先分清“这是哪一类几何”，后面的 support / SDF / primitive collision 才不会混成一团。 |
| `newton/_src/geometry/support_function.py` | convex-style 几何读法。`extract_shape_data()` 把 narrow-phase arrays 还原成 `GenericShapeData`，`support_map()` 再按 `shape_type` 返回局部 support point，见 `newton/_src/geometry/support_function.py:L107-L304`, `newton/_src/geometry/support_function.py:L351-L393`。 | 它把“shape type 只是个枚举”推进到“枚举如何决定可查询几何”。 |
| `newton/_src/geometry/kernels.py` | 一个很值钱的 shape-local 查询切片。这里先做 `X_ws = X_wb * X_bs`、`X_sw = inverse(X_ws)`，再按 `GeoType` 分流到 primitive SDF、mesh query、plane / hfield 采样，见 `newton/_src/geometry/kernels.py:L1053-L1140`。 | 它把 transform chain 和 shape representation chain 在同一处接上。 |
| `newton/_src/geometry/collision_primitive.py` | primitive pair 的小型叶子函数库，例如 `collide_sphere_sphere()`、`collide_sphere_capsule()`、`collide_plane_box()`，见 `newton/_src/geometry/collision_primitive.py:L111-L177`, `newton/_src/geometry/collision_primitive.py:L366-L429`。 | 第一遍只把它当成“primitive 最后会落到这种解析几何叶子”，不要把整章拖进完整碰撞算法。 |

## 调用链总览

这章更适合按“概念怎样在代码里移动”来读，而不是按 runtime 时间线来读。

1. `frame / transform` 词汇先在 builder 里登记成局部关系：`shape_transform` 说明 shape 相对 body 怎么挂，`joint_X_p / joint_X_c` 说明 joint anchor 分别从 parent / child 哪一侧看，`body_com` 说明质心相对 body frame 在哪，见 `newton/_src/sim/builder.py:L886-L887`, `newton/_src/sim/builder.py:L1016-L1017`, `newton/_src/sim/builder.py:L1046-L1049`。`add_joint()` 和 `add_shape()` 只是把这些局部关系写进 builder，见 `newton/_src/sim/builder.py:L3563-L3602`, `newton/_src/sim/builder.py:L5265-L5273`。
2. 这些局部关系在两个方向继续外推。articulation FK 里，关节状态先生成 `X_j`，再一路接成 `X_wpj -> X_wcj -> X_wc`，把 child body pose 放到 world，见 `newton/_src/sim/articulation.py:L235-L337`；geometry kernel 里，同一个思路变成 `X_ws = X_wb * X_bs` 与 `X_sw = inverse(X_ws)`，把 world query 拉回 shape local frame，见 `newton/_src/geometry/kernels.py:L1053-L1060`。所以 transform chain 既服务运动学，也服务碰撞几何查询。
3. spatial quantity 这条链先在 `newton/_src/math/spatial.py` 被包成 Newton-friendly helper：`velocity_at_point()` 负责把 body twist 读到某个偏移点上，`transform_twist()` / `transform_wrench()` 负责换 frame 但保持 Newton 的 `(linear, angular)` 与 `(force, torque)` 语义顺序，见 `newton/_src/math/spatial.py:L54-L130`。articulation 紧接着复用这些 helper，把 `body_qd` 在 origin / COM / point velocity 之间互转，并把 joint axis 变成 world-space motion subspace 与 Jacobian 列，见 `newton/_src/sim/articulation.py:L15-L33`, `newton/_src/sim/articulation.py:L347-L374`, `newton/_src/sim/articulation.py:L900-L955`, `newton/_src/sim/articulation.py:L1039-L1044`。
4. shape representation 这条链先从 `GeoType` 起步：primitive 和 explicit geometry 的第一层分法在 `newton/_src/geometry/types.py:L38-L98`。builder 把 `shape_type / shape_scale / shape_source` 写下来，并在 `finalize()` 里变成 `shape_type / shape_source_ptr / shape_scale` 这些可查询 arrays，见 `newton/_src/sim/builder.py:L5287-L5307`, `newton/_src/sim/builder.py:L9539-L9576`。之后 collision 代码再根据类型走向不同 representation：`support_map()` 给 convex-style 支持点，`geometry/kernels.py` 给 primitive SDF / mesh / hfield 查询，primitive pair 则可直接落到 `collision_primitive.py` 的解析几何叶子，见 `newton/_src/geometry/support_function.py:L107-L304`, `newton/_src/geometry/kernels.py:L1061-L1140`, `newton/_src/geometry/collision_primitive.py:L111-L177`。
5. inertia / mass-property 这条链并不是 geometry 的边角料，而是 geometry 通向 articulation 的压缩接口。`compute_inertia_shape()` 先把 shape 的 `type + scale + src + density + is_solid` 变成 shape-local 的 `mass / com / inertia`，见 `newton/_src/geometry/inertia.py:L474-L605`；builder 再用 `wp.transform_point(xform, c)` 和 `_update_body_mass()` 把它搬到 body frame 并累加成 `body_mass / body_com / body_inertia`，见 `newton/_src/sim/builder.py:L5323-L5326`, `newton/_src/sim/builder.py:L8213-L8245`；articulation 最后把 body inertia 提升成 world-frame spatial inertia，见 `newton/_src/sim/articulation.py:L1161-L1192`。

把整章压成一句话就是：builder 先保存局部几何与质量语义，`spatial.py` 负责 frame-aware 读法，articulation 把这些局部量接到 world / COM 运动学上，geometry collision 代码则把同一批 shape 信息变成可碰撞、可查询的 representation。

## 数据流切片

| 切片 | 读入 | 中间接力 | 写出 | 证据 |
|------|------|----------|------|------|
| shape config -> inertia properties | `add_shape()` 看到的 `type / scale / src / density / is_solid / margin` 与 `xform`。 | `compute_inertia_shape()` 先给 shape-local `mass / com / inertia`；`com_body = wp.transform_point(xform, c)` 把局部质心搬到 body frame；`_update_body_mass()` 再用 `transform_inertia()` 处理旋转和平行轴项，把多个 shape 的贡献并到同一个 body。 | `body_mass / body_com / body_inertia / body_inv_mass / body_inv_inertia`，并在 `finalize()` 后落成 `Model` arrays。 | `newton/_src/sim/builder.py:L5323-L5326`, `newton/_src/geometry/inertia.py:L436-L605`, `newton/_src/sim/builder.py:L8213-L8245`, `newton/_src/sim/builder.py:L10173-L10180` |
| local transforms -> body/world transforms | builder 里的 `shape_transform`、`joint_X_p`、`joint_X_c`，再加关节坐标 `joint_q / joint_qd`。 | articulation 先从关节类型生成 `X_j`，再组合 `X_wpj = X_wp * X_pj`、`X_wcj = X_wpj * X_j`、`X_wc = X_wcj * inverse(X_cj)`；geometry kernel 继续用 `X_ws = X_wb * X_bs` 和 `X_sw = inverse(X_ws)` 把 world query 拉回 shape local frame。 | `body_q` 的 world transforms、`body_qd` 的 COM/world twist，以及 shape-local 查询所需的 `x_local`。 | `newton/_src/sim/builder.py:L1046-L1049`, `newton/_src/sim/builder.py:L3563-L3602`, `newton/_src/sim/articulation.py:L235-L375`, `newton/_src/geometry/kernels.py:L1053-L1060`, `newton/_src/sim/builder.py:L10197-L10198` |
| spatial twist/wrench helpers -> frame-aware motion reading | Newton layout 的 twist / wrench，加上某个 `transform` 或点偏移 `r`。 | `velocity_at_point()` 把整体 twist 读到某个具体点；`transform_twist()` / `transform_wrench()` 在不丢掉 Newton 语义顺序的前提下接 Warp helper；articulation 再把这些 helper 组合成 `com_twist_to_point_velocity()`、origin/COM twist 互转，以及 `transform_twist()` 驱动的 motion subspace / Jacobian。 | 点速度、world-space joint motion subspace、Jacobian 列，以及一套稳定的“这个 6D 量现在在哪个 frame 里表达”的读法。 | `newton/_src/math/spatial.py:L54-L130`, `newton/_src/sim/articulation.py:L15-L33`, `newton/_src/sim/articulation.py:L347-L374`, `newton/_src/sim/articulation.py:L900-L955`, `newton/_src/sim/articulation.py:L1039-L1044` |
| geometry type -> collision-ready representation | `GeoType`、`shape_scale`、`shape_source`、`shape_transform`。 | `GeoType` 先把 shape 分成 primitive / explicit；builder `finalize()` 把类型、缩放和 source 指针冻结成 arrays；`extract_shape_data()` / `support_map()` 把它们变成可做 support query 的 `GenericShapeData`；`geometry/kernels.py` 按类型切到 primitive SDF、mesh query、plane / hfield 采样；如果还是 primitive-pair，最后落到 `collision_primitive.py` 的解析叶子。 | support point、SDF 距离 / 法向 / 接触样本，或 primitive-specific contact geometry。 | `newton/_src/geometry/types.py:L38-L98`, `newton/_src/sim/builder.py:L5287-L5307`, `newton/_src/sim/builder.py:L9539-L9576`, `newton/_src/geometry/support_function.py:L107-L304`, `newton/_src/geometry/support_function.py:L351-L393`, `newton/_src/geometry/kernels.py:L1061-L1140`, `newton/_src/geometry/collision_primitive.py:L111-L177` |

## GPU 并行切片

这页的重点不是 launch 维度，而是语义桥接，所以 GPU 只保留三个最小结论：

- `builder.py` 和 `spatial.py` 的高价值锚点大多是 host-side 组织或 `@wp.func` helper，它们决定的是数据和 frame 语义，不是并行策略。
- 真正进入批量执行的地方，主要是 articulation kernels 与 shape 查询 kernels：`eval_articulation_fk` / `eval_articulation_jacobian` 说明同一套 frame 与 spatial 规则会被成批应用到整条 articulation，见 `newton/_src/sim/articulation.py:L377-L389`, `newton/_src/sim/articulation.py:L958-L977`；`geometry/kernels.py` 则说明同一套 `GeoType` dispatch 会被成批应用到 shape-local 查询，见 `newton/_src/geometry/kernels.py:L1053-L1140`。
- 第一遍先把这些 kernel 读成“相同规则的批量化”，不要在 chapter 03 提前展开 launch policy、cache 或更细的 GPU 优化细节。

## 回指原理

| 源码点 | 对应原理 | 第一遍应该怎么翻译 |
|--------|----------|--------------------|
| `newton/_src/sim/builder.py:L886-L887`, `newton/_src/sim/builder.py:L1016-L1017`, `newton/_src/sim/builder.py:L1046-L1049`, `newton/_src/sim/articulation.py:L328-L337` | frame / transform 不是抽象数学装饰，而是局部关系怎样一层层接到 world 的结构描述。 | 先翻译成“shape 挂在 body 上，joint anchor 分别站在 parent / child 两边，FK 只是把这条局部链接起来”。 |
| `newton/_src/math/spatial.py:L54-L130`, `newton/_src/sim/articulation.py:L15-L33`, `newton/_src/sim/articulation.py:L900-L955` | twist / wrench helper 的核心问题始终是“在哪个 frame、在哪个点上读这个 6D 量”。 | 先把它们翻译成“frame adapter”，不是新的物理章节；`transform_wrench()` 则是和 `transform_twist()` 对称的受力侧版本。 |
| `newton/_src/geometry/types.py:L38-L98`, `newton/_src/geometry/support_function.py:L107-L304`, `newton/_src/geometry/kernels.py:L1061-L1140`, `newton/_src/geometry/collision_primitive.py:L111-L177` | `shape type` 不只是标签，而是后续 collision representation 与 query path 的分流键。 | 第一遍先问“这是什么类几何，要走 support map、SDF/mesh query，还是 primitive analytic leaf”，不要先问完整 narrow phase 算法。 |
| `newton/_src/geometry/inertia.py:L474-L605`, `newton/_src/sim/builder.py:L5323-L5326`, `newton/_src/sim/builder.py:L8213-L8245`, `newton/_src/sim/articulation.py:L1161-L1192` | 惯量是 geometry 通向 articulation 的桥，不是碰撞之外的附属物。 | 先翻译成“shape 先给局部质量属性，builder 把它们搬到 body 上，articulation 再把 body inertia 变成 spatial inertia”。 |

带着这四条回指去读后续章节时，路线会自然分叉：去 `04_scene_usd` 继续追“这些局部关系最初从哪里来”，去 `05_rigid_articulation` 继续追“frame / spatial / inertia 怎样进入刚体动力学”，去 `06_collision` 继续追“这些 representation 怎样变成真实接触生成”。chapter 03 的源码走读做到这里就够了。
