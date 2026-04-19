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

# 03 数学与几何基础 源码走读

如果你是第一次读这一章，最好先配合 `principle.md` 一起读；如果你是直接跳进源码走读也没关系，但一旦发现术语开始变快，就先回到 `principle.md` 把对象关系补齐，再回来追源码。

这份主 walkthrough 只追 chapter 03 最核心的那条桥：`frame / transform / spatial quantity / shape representation / inertia` 这些词，怎样在 Newton 源码里互相接上。目标不是把你变成一轮完整的数学课，而是让你在后面的 scene、articulation、collision 章节里再看到这些词时，不会只剩术语压力。

## What This Walkthrough Follows

这一页只追下面这条词汇主线：

```text
local frames in builder
-> transform composition in articulation / geometry queries
-> twist & wrench helpers
-> GeoType-driven representation dispatch
-> geometry compressed into mass / inertia
```

这一页刻意不展开三类东西：

- 完整的 Lie group / quaternion 课程
- GJK、EPA、hydroelastic 这类碰撞算法推导
- ABA / CRBA / Featherstone 的完整动力学递推

第一遍先守住一句话：chapter 03 讲的是 **后面各章都会复用的几何语言和空间量语言**。

## One-Screen Chapter Map

```text
builder 里的 local frame / shape / mass bookkeeping
                    |
                    v
 articulation FK 和 geometry query 都在组合 transform
                    |
                    v
    spatial helpers 让 twist / wrench 换 frame 还保留语义
                    |
                    v
        GeoType 决定 shape 走哪种 query / representation
                    |
                    v
   compute_inertia_shape + _update_body_mass 把几何压成质量属性
```

## Beginner Path

1. 先看 Stage 1。
   - 想验证什么：`shape_transform`、`joint_X_p / joint_X_c`、`body_com` 这些词到底先落在哪。
   - 看完后应该能说：Newton 先把局部关系记下来，而不是一开始全写成 world 坐标。
2. 再看 Stage 2。
   - 想验证什么：同样的 transform chain 为什么会同时出现在 articulation 和 geometry query 里。
   - 看完后应该能说：这不是两套数学，而是同一套局部到 world 的组合规则。
3. 再看 Stage 3。
   - 想验证什么：`twist / wrench / velocity_at_point` 为什么会看起来像一组 helper。
   - 看完后应该能说：它们本质上是在做 frame-aware 的 6D 量转换。
4. 再看 Stage 4。
   - 想验证什么：为什么 `GeoType` 不只是个标签。
   - 看完后应该能说：shape type 会直接决定查询路径和几何表示。
5. 最后看 Stage 5。
   - 想验证什么：质量、质心、惯量为什么会被这一章拿出来讲。
   - 看完后应该能说：它们是 geometry 进入 articulation / dynamics 的压缩接口。

## Main Walkthrough

### Stage 1: Newton 先把局部几何关系记成显式字段

**Claim**

chapter 03 里的 frame 词汇不是抽象装饰，而是 builder 一开始就明确存下来的对象关系：shape 相对 body 怎么挂、joint 两侧的锚点在哪、body 的质心在哪。

**Why it matters**

如果不先把这些字段认成“局部关系账本”，后面看到 transform 组合时就会误以为 Newton 在凭空玩数学体操。

**Source excerpt**

builder 先把这些关键字段记成 lists：

```python
self.shape_transform: list[Transform] = []
self.shape_body: list[int] = []
...
self.body_com: list[Vec3] = []
...
self.joint_X_p: list[Transform] = []
self.joint_X_c: list[Transform] = []
```

真正写入时也很直白：

```python
self.joint_child.append(child)
self.joint_X_p.append(parent_xform)
self.joint_X_c.append(child_xform)
...
self.shape_body.append(body)
self.shape_transform.append(xform)
```

**Verification cues**

- `shape_transform` 明确是 shape-to-body transform，不是 world pose。
- `joint_X_p / joint_X_c` 明确把 joint 两侧都保留下来，所以 parent / child frame 不会混成一层。
- `body_com` 作为单独字段存在，说明 body origin 和 COM 也不是默认重合。

**Output passed to next stage**

一份局部关系账本。后面所有 FK、shape query、mass update 都要从这里接着读。

### Stage 2: 同一套 transform 组合规则同时服务 articulation 和 geometry

**Claim**

articulation FK 和 geometry query 并不是两套不同数学，它们都在做同一件事：把局部关系一层层组合到 world，再在需要时反过来拉回 local frame。

**Why it matters**

这一步能把 chapter 03 真正变成后续章节的共同底座。你一旦看懂这条 transform chain，`05` 和 `06` 的阅读阻力都会小很多。

**Source excerpt**

在 articulation FK 里，joint 两侧 frame 会一路拼到 child body：

```python
X_wpj = X_pj
if parent >= 0:
    X_wp = body_q[parent]
    X_wpj = X_wp * X_wpj

X_wcj = X_wpj * X_j
X_wc = X_wcj * wp.transform_inverse(X_cj)
```

在 geometry query 里，同一个想法只是换成 shape：

```python
X_wb = body_q[rigid_index]
X_bs = shape_transform[shape_index]

X_ws = wp.transform_multiply(X_wb, X_bs)
X_sw = wp.transform_inverse(X_ws)

x_local = wp.transform_point(X_sw, px)
```

**Verification cues**

- articulation 那边是在算 child body 的 world pose。
- geometry 那边是在把 world-space query point 拉回 shape local frame。
- 两边共同依赖的其实都是“局部关系先记好，再按需要组合 / 取逆”。

**Output passed to next stage**

一个统一读法：frame 组合先得到正确的 pose，再决定后面要读速度、受力，还是几何查询。

### Stage 3: `twist / wrench` helper 本质上是 frame adapter

**Claim**

`velocity_at_point()`、`transform_twist()`、`transform_wrench()` 这些 helper 的核心任务，不是引入新概念，而是让 Newton 的 6D 量在换 frame 时仍然保持一致语义。

**Why it matters**

如果你把 spatial quantity 读成“另一本教材”，chapter 03 就会突然变得很重。第一遍其实只要看懂：这些 helper 在帮你维护“这个 6D 量现在在哪个 frame、按什么顺序表达”。

**Source excerpt**

`spatial.py` 里的三个关键 helper 都在做 layout / frame 的适配：

```python
def velocity_at_point(qd: wp.spatial_vector, r: wp.vec3) -> wp.vec3:
    qd_wp = wp.spatial_vector(wp.spatial_bottom(qd), wp.spatial_top(qd))
    return wp.velocity_at_point(qd_wp, r)

def transform_twist(t: wp.transform, x: wp.spatial_vector) -> wp.spatial_vector:
    x_wp = wp.spatial_vector(wp.spatial_bottom(x), wp.spatial_top(x))
    y_wp = wp.transform_twist(t, x_wp)
    return wp.spatial_vector(wp.spatial_bottom(y_wp), wp.spatial_top(y_wp))

def transform_wrench(t: wp.transform, x: wp.spatial_vector) -> wp.spatial_vector:
    x_wp = wp.spatial_vector(wp.spatial_bottom(x), wp.spatial_top(x))
    y_wp = wp.transform_wrench(t, x_wp)
    return wp.spatial_vector(wp.spatial_bottom(y_wp), wp.spatial_top(y_wp))
```

**Verification cues**

- 这几段代码都在显式处理 Newton 和 Warp 的 spatial layout 差异。
- `velocity_at_point()` 说明 twist 不只是“一个 6D 向量”，还和参考点有关。
- `transform_twist()` 和 `transform_wrench()` 是一对对称操作，分别服务运动量和受力量。

**Output passed to next stage**

一套稳定心智模型：当 later chapters 里出现 twist / wrench / Jacobian 列时，你知道它们本质上都离不开 frame-aware 转换。

### Stage 4: `GeoType` 不只是标签，它决定 shape 走哪条表示路径

**Claim**

在 Newton 里，shape type 不是展示信息，而是几何表示和查询路径的分流键：primitive、mesh、convex mesh、heightfield 会走不同的 query contract。

**Why it matters**

这一步是 chapter 03 到 chapter 06 的桥。如果不把 `GeoType` 读成 dispatch key，后面看 collision 分支时会很容易迷路。

**Source excerpt**

`GeoType` 先给出第一层分类：

```python
class GeoType(enum.IntEnum):
    PLANE = 1
    HFIELD = 2
    SPHERE = 3
    CAPSULE = 4
    ELLIPSOID = 5
    CYLINDER = 6
    BOX = 7
    MESH = 8
    CONE = 9
    CONVEX_MESH = 10

    @property
    def is_primitive(self) -> bool:
        ...

    @property
    def is_explicit(self) -> bool:
        ...
```

真正查询时，它会直接决定要走哪种几何分支：

```python
if geo_type == GeoType.SPHERE:
    d = sdf_sphere(x_local, geo_scale[0])
    n = sdf_sphere_grad(x_local, geo_scale[0])

if geo_type == GeoType.BOX:
    d = sdf_box(x_local, geo_scale[0], geo_scale[1], geo_scale[2])
    n = sdf_box_grad(x_local, geo_scale[0], geo_scale[1], geo_scale[2])

if geo_type == GeoType.MESH or geo_type == GeoType.CONVEX_MESH:
    ...

if geo_type == GeoType.HFIELD:
    d, n = sample_sdf_grad_heightfield(hfd, heightfield_elevations, x_local)
```

**Verification cues**

- primitive 类型直接走解析 SDF / gradient。
- mesh / convex mesh 需要 source pointer 和 mesh query。
- 你不用第一遍背完所有 query 细节，只要先把“分流权在 `GeoType` 手里”记住。

**Output passed to next stage**

一条更清楚的表示链：shape type 决定 query contract，而 query contract 再决定后面怎样进入 collision 或 inertia 相关代码。

### Stage 5: inertia 是 geometry 进入 dynamics 的压缩接口

**Claim**

质量、质心、惯量不是 geometry 旁边的注释，而是 geometry 最终压缩给 articulation / dynamics 使用的接口。

**Why it matters**

只有把这一步读顺，chapter 03 才真的连上 `05_rigid_articulation`。否则 shape 只是“长得不一样”，还没有变成“动起来的代价不一样”。

**Source excerpt**

`compute_inertia_shape()` 先按 shape type 给出局部质量属性：

```python
if type == GeoType.SPHERE:
    solid = compute_inertia_sphere(density, scale[0])
    return solid
elif type == GeoType.BOX:
    solid = compute_inertia_box(density, scale[0], scale[1], scale[2])
    return solid
elif type == GeoType.MESH or type == GeoType.CONVEX_MESH:
    ...
    return m_new, c_new, I_new
```

builder 再把 shape 级质量属性合并到 body 上：

```python
(m, c, inertia) = compute_inertia_shape(type, scale, src, cfg.density, cfg.is_solid, cfg.margin)
com_body = wp.transform_point(xform, c)
self._update_body_mass(body, m, inertia, com_body, xform.q)
```

而 `_update_body_mass()` 真正在做的是重新计算 body COM 和 shifted inertia：

```python
new_com = (self.body_com[i] * self.body_mass[i] + p * m) / new_mass

new_inertia = transform_inertia(
    self.body_mass[i], self.body_inertia[i], com_offset, wp.quat_identity()
) + transform_inertia(m, inertia, shape_offset, q)

self.body_mass[i] = new_mass
self.body_inertia[i] = new_inertia
self.body_com[i] = new_com
```

**Verification cues**

- inertia 不是从 body 凭空出现的，而是从 shape geometry 推上来的。
- `com_body = wp.transform_point(xform, c)` 说明连质心也要先过局部 frame 变换。
- `_update_body_mass()` 明确在做多 shape 合并，所以 body mass property 是汇总结果，不是单一 shape 属性。

**Output passed to next stage**

`body_mass / body_com / body_inertia` 这组三件套。它们会在 `05` 的 articulation 与 dynamics 代码里继续被消费。

## Object Ledger

| 对象 | 谁生产 | 谁消费 | 盯哪些字段 |
|------|--------|--------|------------|
| `shape_transform` | builder | geometry query、collision pipeline | shape 相对 body 的局部位姿 |
| `joint_X_p / joint_X_c` | builder | articulation FK、Jacobian 相关 helper | joint 两侧 frame |
| `body_com` | builder mass accumulation | articulation、solver kernels | COM 相对 body origin 的偏移 |
| `X_wc / X_ws / X_sw` | transform composition | articulation / geometry kernels | world pose 与 local query frame |
| `wp.spatial_vector` | state / helper functions | spatial transforms、Jacobian、dynamics | linear 与 angular 部分的语义 |
| `GeoType` | shape config / finalize | support / SDF / mesh query | primitive vs explicit dispatch |
| `body_mass / body_inertia` | `compute_inertia_shape()` + `_update_body_mass()` | articulation / dynamics | geometry 被压缩后的质量属性 |

## Stop Here

读到这里就已经够 chapter 03 的 80-90% 了。

如果你现在能用自己的话讲顺下面这句话，这一章的 beginner 目标就完成了：

```text
builder 先把 shape、joint、COM 这些局部关系记下来；
articulation 和 geometry query 再用同一套 transform 规则组合到 world；
spatial helper 负责 frame-aware 的 6D 量转换；
GeoType 决定查询路径；
inertia 则把几何压成后面 dynamics 要消费的质量属性。
```

这时你已经可以更稳地进入 `04_scene_usd`、`05_rigid_articulation` 和 `06_collision`。

## Go Deeper

如果你还想继续精确追源码，再去 `source-walkthrough-deep.md`：

- 想保留所有 file/symbol/line anchors：看 `Fast Deep Index`
- 想逐跳追 frame、shape、inertia 的 exact handoff：看 `Exact Handoff Trace`
- 想知道哪些 mesh / Jacobian / primitive 细节第一遍可以跳过：看 `Optional Branches`
- 想逐条核对这里的 claim：看 `Verification Anchors`
