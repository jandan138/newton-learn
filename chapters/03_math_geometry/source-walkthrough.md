---
chapter: 03
title: 数学与几何基础
last_updated: 2026-04-20
source_paths:
  - newton/_src/sim/model.py
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

如果你已经读完 `02_newton_arch`，下一步最容易冒出的困惑通常不是“Newton 里还有哪些对象”，而是：为什么 `04/05/06` 里总在反复出现 `transform`、`frame`、`body_q`、`shape_transform`、`body_qd`、`inertia`？

先给最短答案，但不要一次背完这些词：

- Newton 先记 `body / joint / shape` 之间“谁相对谁”的局部关系。
- 这些局部关系再一路组合成 `body_q` 这样的 world pose。
- 刚体一旦真的动起来，又要用 `body_qd` 这种 6D 量去写“整体怎么动、怎么受力”。
- shape 还要按 `GeoType` 分流到不同几何表示。
- 最后几何再被压成 `body_mass / body_com / body_inertia`，交给动力学代码。

整章就是在把这五步的来龙去脉讲顺。

后面整份文件都会反复回到同一条故事线：`local relationships -> transform chains -> spatial quantities -> shape representation -> inertia`。如果你想配一个更慢一点的概念版，可以再看 `principle.md`；但只读这一份，也应该能把 chapter 03 的主线讲顺。

## What This Walkthrough Follows

这一页只追后面章节最常复用的那条“图例”主线：

```text
builder 先记 local frame / 局部关系（也就是“谁相对谁写”）
-> 同一条 transform chain 组合成 body_q 和 world/local query pose
-> body_qd 和 spatial helper 保持 6D 运动 / 受力量的 frame 语义
-> GeoType 决定 shape representation 和 query path
-> inertia 把 geometry 压成 dynamics 要消费的质量属性
```

这一页刻意不展开三类东西：

- 完整的 Lie group / quaternion 课程
- GJK、EPA、hydroelastic 这类碰撞算法推导
- ABA / CRBA / Featherstone 的完整动力学递推

第一遍先守住一句话：chapter 03 讲的是 **后面 `04/05/06` 会反复复用的几何语言和空间量语言**。

## One-Screen Chapter Map

```text
scene / builder 留下局部关系：joint_X_p、joint_X_c、shape_transform、body_com
                              |
                              v
同一条 transform chain 把这些局部关系接成 body_q 和 shape world pose
                              |
                              v
body_qd 和 spatial helper 把“刚体怎么动、怎么受力”写成 frame-aware 的 6D 量
                              |
                              v
GeoType 回答“这到底是哪一类 shape，要走哪条 query path”
                              |
                              v
compute_inertia_shape + _update_body_mass 把 geometry 压成 mass / com / inertia
```

## Beginner Path

1. 先看 Stage 1。
   - 想验证什么：`shape_transform`、`joint_X_p / joint_X_c`、`body_com` 为什么会在 builder 里先出现。
   - 看完后应该能说：Newton 先记“谁相对谁”的局部关系，而不是一开始就写死 world 坐标。这是在给 `04_scene_usd` 之后的所有章节打底。
2. 再看 Stage 2。
   - 想验证什么：为什么后面章节会反复出现 `body_q`。
   - 看完后应该能说：`body_q` 就是局部关系一路组合后的 body world pose；`05` 的 FK 和 `06` 的几何查询都在复用同一条 transform chain。
3. 再看 Stage 3。
   - 想验证什么：为什么后面章节又会冒出 `body_qd`、twist、wrench、`velocity_at_point`。
   - 看完后应该能说：这些都是在写同一个东西的不同侧面，也就是“挂在 frame 上的 6D 运动 / 受力量”。它们会直接进入 `05`，也会在 `06` 的点速度问题里再次出现。
4. 再看 Stage 4。
   - 想验证什么：为什么 `GeoType` 不只是个类型名。
   - 看完后应该能说：它决定 shape 走哪种表示和查询路径，所以 `06_collision` 不会把所有 shape 当成同一种东西处理。
5. 最后看 Stage 5。
   - 想验证什么：为什么 chapter 03 还要突然讲质量、质心、惯量。
   - 看完后应该能说：它们是 geometry 进入 `05_rigid_articulation` 和动力学代码前的压缩接口。

## Main Walkthrough

### Stage 1: Newton 先记局部关系，不先记 world 答案

**Definition**

- `frame`：一个带原点和朝向的参考系。第一遍只把它读成“谁在拿谁当参考系”。

**Claim**

chapter 03 里的 `frame` 词汇不是抽象装饰，而是 builder 一开始就明确存下来的对象关系：shape 相对 body 怎么挂、joint 两侧的锚点在哪、body 的质心在哪。

**Why it matters**

如果不先把这些字段认成“局部关系账本”，后面看到 transform 组合时就会误以为 Newton 在凭空玩数学体操。`04_scene_usd` 的导入代码，本质上就是把场景输入先翻译成这本账本；`05` 和 `06` 再继续消费它。

**Source excerpt**

builder 先把这些关键字段记成 lists：

这段本质上是局部关系账本的字段表，先保持干净代码，重点只看它确实把 shape / body / joint 两侧 frame 分开存。

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

以下摘录为教学注释版，注释非原源码。

```python
self.joint_child.append(child)  # 先记这条 joint 指向哪个 child body
self.joint_X_p.append(parent_xform)  # 记录 joint 在 parent 侧的局部锚点
self.joint_X_c.append(child_xform)  # 记录 joint 在 child 侧的局部锚点
...
self.shape_body.append(body)  # 记录这个 shape 挂在哪个 body 上
self.shape_transform.append(xform)  # 记录 shape 相对 body 的局部位姿
```

**Verification cues**

- `shape_transform` 明确是 shape-to-body transform，不是 world pose。
- `joint_X_p / joint_X_c` 明确把 joint 两侧都保留下来，所以 parent / child frame 不会混成一层。
- `body_com` 作为单独字段存在，说明 body origin 和 COM 也不是默认重合。

**Checkpoint**

如果你现在还答不上来“`shape_transform` 是谁相对谁写的”“`joint_X_p` 和 `joint_X_c` 为什么要分两边存”，先不要继续。第一步的核心不是记字段名，而是把它们都认成局部关系。

**Output passed to next stage**

一份局部关系账本。后面所有 FK、shape query、mass update 都要从这里接着读。

### Stage 2: 同一条 transform chain 把局部关系接成 `body_q`

**Definition**

- `transform`：第一遍先读成“平移 + 旋转”，也就是把一个 frame 里的东西搬到另一个 frame 里。
- `body_q`：每个 body 当前的 world pose buffer。后面看到它时，先翻译成“这个 body 现在在世界里放在哪、朝哪”。

**Claim**

articulation FK 和 geometry query 并不是两套不同数学，它们都在做同一件事：把局部关系一层层组合到 world，得到 `body_q` 这类 world pose；然后在需要时再反过来拉回 local frame。

**Why it matters**

这一步能把 chapter 03 真正变成后续章节的共同底座。你一旦看懂这条 transform chain，`04` 里“scene 输入怎样落到 body / shape 上”、`05` 里 FK 怎样长成 `body_q`、`06` 里 query point 怎样被拉回 shape local frame，都会顺很多。

**Source excerpt**

`Model` 先把 `body_q` 声明成 body world pose 的长期 buffer：

```python
self.body_q: wp.array[wp.transform] | None = None  # 每个 body 当前的 world pose buffer
```

在 articulation FK 里，joint 两侧 frame 会一路拼到 child body：

以下摘录为教学注释版，注释非原源码。

```python
X_wpj = X_pj  # 先从 joint 在 parent 侧的局部 frame 开始
if parent >= 0:
    X_wp = body_q[parent]  # 取 parent body 当前的 world pose
    X_wpj = X_wp * X_wpj  # 把 joint parent frame 接到 world

X_wcj = X_wpj * X_j  # 再沿关节自由度走到 joint child frame
X_wc = X_wcj * wp.transform_inverse(X_cj)  # 从 child 侧锚点还原出 child body 的 world pose
```

在 geometry query 里，同一个想法只是换成 shape：

```python
X_wb = body_q[rigid_index]  # 先取这个 rigid body 当前的 world pose
X_bs = shape_transform[shape_index]  # 再取 shape 相对 body 的局部 pose

X_ws = wp.transform_multiply(X_wb, X_bs)  # 把 shape 从 body frame 放到 world
X_sw = wp.transform_inverse(X_ws)  # 准备把 world query 拉回 shape local

x_local = wp.transform_point(X_sw, px)  # 把 world-space query point 转成 shape local
```

**Verification cues**

- `body_q` 的类型本身就是 `wp.transform`，所以它不是任意状态，而是 body pose。
- articulation 那边是在算 child body 的 world pose。
- geometry 那边是在把 world-space query point 拉回 shape local frame。
- 两边共同依赖的其实都是“局部关系先记好，再按需要组合 / 取逆”。

**Checkpoint**

如果你现在还会把 `body_q` 看成一个需要死背的缩写，先停一下。第一遍只要把它翻译成“局部关系链最后接出来的 body world pose”就够了。

**Output passed to next stage**

一个统一读法：先用 transform chain 得到正确 pose，再决定后面要读速度、受力，还是几何查询。

### Stage 3: `body_qd` 和 spatial helper 解释了为什么运动要写成 6D

**Definition**

- `spatial quantity` / `6D quantity`：把线性部分和角部分绑在一起的刚体量。第一遍你可以把它读成“挂在某个 frame 上的 6D 运动量或受力量”。
- `body_qd`：每个 body 当前的 spatial velocity buffer。它不是单独一个 `vec3` 速度，而是刚体整体的 6D 速度描述。

**Claim**

`velocity_at_point()`、`transform_twist()`、`transform_wrench()` 这些 helper 的核心任务，不是引入新概念，而是让 `body_qd` 这种 6D 量在换 frame、换参考点时仍然保持一致语义。

**Why it matters**

如果你把 spatial quantity 读成“另一本教材”，chapter 03 就会突然变得很重。第一遍其实只要看懂：这些 helper 在帮你维护“这个 6D 量现在在哪个 frame、按什么顺序表达”。这会直接帮到 `05` 的 articulation helper，也会帮你在 `06` 里理解“某个接触点速度从哪来”。

**Source excerpt**

`Model` 把 body 速度存成 spatial vector：

```python
self.body_qd: wp.array[wp.spatial_vector] | None = None  # 每个 body 的 6D spatial velocity buffer
```

`spatial.py` 里的三个关键 helper 都在做 layout / frame 的适配：

这组三个 adapter 主要靠并排结构理解，先保持干净代码，重点看它们都在做“拆 layout -> 调 Warp helper -> 再装回”。

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

**Why these adapters exist**

如果你第一次看到 `spatial_bottom / spatial_top`，最自然的反应通常是：为什么这里像在搬运数据？

第一遍最稳的答案是：Newton 想把 `body_qd` 一直当成统一的 6D 量来传，而 Warp 底层又已经有一套现成的 spatial helper。于是这些 wrapper 的职责，就是在“Newton 自己怎么存”和“Warp helper 期待怎么吃”之间做一次适配。这样后面的 articulation、collision、Jacobian 代码就不必每次手写这层拆装。

**Verification cues**

- `body_qd` 的类型不是 `wp.vec3`，而是 `wp.spatial_vector`，说明 Newton 从一开始就把刚体速度当成 6D 量。
- 这几段代码都在显式处理 Newton 和 Warp 的 spatial layout 差异。
- `velocity_at_point()` 说明 twist 不只是“一个 6D 向量”，还和参考点有关。
- `transform_twist()` 和 `transform_wrench()` 是一对对称操作，分别服务运动量和受力量。

**Checkpoint**

如果你现在还会把 `body_qd` 读成“body 的某个普通速度数组”，先不要继续。第一遍最值钱的理解是：它在保存刚体整体的 6D 运动语义，而不是只保存一个点的线速度。

**Output passed to next stage**

一套稳定心智模型：当后面章节里出现 `body_qd`、twist、wrench、Jacobian 列时，你知道它们本质上都离不开 frame-aware 的 6D 转换。

### Stage 4: `GeoType` 不只是标签，它决定 shape 走哪条表示路径

**Definition**

- `GeoType`：shape 的类型枚举，但它不只是“这个 shape 叫什么名字”。更准确地说，它是在回答“这到底是哪一类几何，后面该走哪条 query / representation path”。

**Claim**

在 Newton 里，shape type 不是展示信息，而是几何表示和查询路径的分流键：primitive、mesh、convex mesh、heightfield 会走不同的 query contract。

**Why it matters**

这一步是 chapter 03 到 `04` 和 `06` 的桥。`04_scene_usd` 会把场景输入翻译成具体 shape 类型和参数；`06_collision` 则会根据 `GeoType` 分流到不同查询路径。如果不把 `GeoType` 读成 dispatch key，后面看 collision 分支时会很容易迷路。

**Source excerpt**

`GeoType` 先给出第一层分类：

这段更像类型目录，先保持干净代码，重点看它后面如何决定 query 分支。

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

其中这两个谓词第一遍可以先这样翻译：

- `is_primitive`：这种 shape 能不能直接走解析式 primitive 路线，比如 sphere / box / capsule。
- `is_explicit`：这种 shape 的几何是不是已经显式给出来了，而不是要走更间接的场或采样表示。

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

**Checkpoint**

如果你现在还会把 `GeoType` 看成“只是导入阶段顺手留的标签”，先停一下。chapter 06 里真正的几何查询分支，就是靠它起跳的。

**Output passed to next stage**

一条更清楚的表示链：shape type 决定 query contract，而 query contract 再决定后面怎样进入 collision 或 inertia 相关代码。

### Stage 5: inertia 是 geometry 进入 dynamics 的压缩接口

**Definition**

- `inertia`：第一遍先读成“几何对运动和转动有多难被改变”的压缩描述。它通常和质量、质心一起出现，因为 Newton 真正要交给动力学的不是“这个 shape 长什么样”，而是“这个 body 动起来的代价是什么”。

**Claim**

质量、质心、惯量不是 geometry 旁边的注释，而是 geometry 最终压缩给 articulation / dynamics 使用的接口。

**Why it matters**

只有把这一步读顺，chapter 03 才真的连上 `05_rigid_articulation`。否则 shape 只是“长得不一样”，还没有变成“动起来的代价不一样”。这也是为什么 `04` 里导入的 density / geometry 设置，最后会在 `05` 里变成真实动力学差异。

**Source excerpt**

`compute_inertia_shape()` 先按 shape type 给出局部质量属性：

前后两段 dispatch / 合并公式都偏结构化，先保持干净代码；真正最值得盯的是 shape 级质量属性怎样被交给 body。

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

以下摘录为教学注释版，注释非原源码。

```python
(m, c, inertia) = compute_inertia_shape(type, scale, src, cfg.density, cfg.is_solid, cfg.margin)  # 先把这个 shape 压成局部质量属性
com_body = wp.transform_point(xform, c)  # 再把 shape 局部 COM 变到 body frame
self._update_body_mass(body, m, inertia, com_body, xform.q)  # 最后把这份质量贡献并进 body
```

而 `_update_body_mass()` 真正在做的是重新计算 body COM 和 shifted inertia：

这里的 COM / inertia 合并公式更适合保持干净代码，避免注释打断它的结构。

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

**Checkpoint**

如果你现在还会把 inertia 想成“geometry 章节里突然插进来的另一门课”，先停一下。第一遍只要记住：它是 geometry 被打包之后，真正交给 dynamics 的接口。

**Output passed to next stage**

`body_mass / body_com / body_inertia` 这组三件套。它们会在 `05` 的 articulation 与 dynamics 代码里继续被消费。

## Object Ledger

| 名字 | 第一遍先怎么翻译 | 后面最常在哪再出现 |
|------|------------------|----------------------|
| `frame` | “谁相对谁写”的参考系。 | `04` 的 scene parsing，`05` 的 FK / Jacobian，`06` 的 geometry query |
| `shape_transform` | shape 相对 body 的局部位姿。 | `04` 导入 shape placement，`06` 把 shape 放到 world |
| `joint_X_p / joint_X_c` | joint 在 parent / child 两侧的局部锚点。 | `05` 的 articulation FK 和关节相关 helper |
| `body_q` | body 当前的 world pose buffer。 | `05` 的刚体位姿更新，`06` 的碰撞 / 查询入口 |
| `body_qd` | body 当前的 6D spatial velocity buffer。 | `05` 的 motion / Jacobian，`06` 的点速度问题 |
| `GeoType` | shape 类型，也是 query dispatch key。 | `04` 的 shape 导入分型，`06` 的 collision 分支 |
| `body_mass / body_com / body_inertia` | geometry 压出来的 body 级质量属性。 | `05` 的 articulation 与 dynamics |

## Stop Here

读到这里就已经够 chapter 03 的 80-90% 了。

如果你现在能用自己的话讲顺下面这句话，这一章的 beginner 目标就完成了：

```text
后面章节反复出现 transform、frame、body_q、shape_transform、body_qd、inertia，
不是因为 Newton 在不断换数学话题；
而是因为它先把局部关系记账，再用 transform chain 得到 body_q，
再用 6D spatial quantity 写 body_qd 和受力，
再用 GeoType 决定 shape query 路径，
最后用 inertia 把 geometry 压成 dynamics 要消费的质量属性。
```

这时你已经可以更稳地进入 `04_scene_usd`、`05_rigid_articulation` 和 `06_collision`。

## Go Deeper

如果你还想继续精确追源码，再去 `source-walkthrough-deep.md`：

- 想保留所有 file/symbol/line anchors：看 `Fast Deep Index`
- 想逐跳追 frame、shape、inertia 的 exact handoff：看 `Exact Handoff Trace`
- 想知道哪些 mesh / Jacobian / primitive 细节第一遍可以跳过：看 `Optional Branches`
- 想逐条核对这里的 claim：看 `Verification Anchors`
