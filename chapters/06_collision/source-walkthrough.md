---
chapter: 06
title: 碰撞系统
last_updated: 2026-04-20
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

如果你刚从 `05_rigid_articulation` 走到这里，最自然的问题通常是：

```text
为什么不是 body 直接互相碰撞？为什么要先把 shape 放到 world，再先找 candidate pair？
```

先给一个不带太多算法名词的最短答案：

- body 主要提供运动；真正有几何外形、能拿来做碰撞查询的是 shape。
- 一个 body 上可以挂多个 shape，所以碰撞入口必须先把每个 shape 放到 world 里。
- broad phase 先便宜地做一张 maybe-list，也就是 candidate pairs。
- narrow phase 再把这些 maybe-pair 变成真正的接触几何。
- `write_contact()` 最后把接触几何压进统一的 `Contacts` 缓冲区，交给后面的章节。

这份主 walkthrough 是给第一次追 chapter 06 源码的人准备的。目标不是把所有 branch 都摊平，而是让你不离开这页，也能把 collision 主干读成一条连续 handoff：`body/world state -> candidate pairs -> ContactData -> Contacts`。

如果你想再配一个更慢一点的概念版，可以再看 `principle.md`；但只读这一份，也应该能把 chapter 06 的主线讲顺。

第一次先把下面几个词翻成人话：

- `broad phase`：便宜地筛掉大多数明显不可能接触的 pair，只留下 maybe-list。
- `candidate pair`：值得继续精查的一对 shape；它还不是最终 contact。
- `narrow phase`：对 candidate pair 做真正接触几何查询的阶段。
- `ContactData`：narrow phase 产出的统一接触几何记录，还没写进最终 `Contacts`。
- `margin / gap`：shape 周围的接触包络设置。第一遍先把 `margin` 读成接触几何会参考的表面厚度，把 `gap` 读成“多早开始把接近当回事”的额外余量。比如你完全可以把 `margin=0.01` 暂时脑补成“先把表面看厚 1cm”，而 `gap` 则是在这个基础上再往外留一点提前预警空间。
- `write_contact()`：碰撞链最后的 writer，把统一接触几何改写成 solver 直接读的 `Contacts` 数组。

## What This Walkthrough Follows

只追这一条主线：

```text
body motion + Model.shape_*
-> compute_shape_aabbs(...)
-> world-space shape query objects
-> broad phase candidate pairs
-> narrow phase routing
-> ContactData
-> write_contact(...)
-> Contacts
```

这一页刻意不展开三类东西：

- broad phase 的复杂度对比和实现细枝末节
- GJK / MPR / SDF / hydroelastic 的算法推导
- `07_constraints_contacts_math` 和 `08_rigid_solvers` 的接触数学与 solver 消费

第一遍先守住一句话：chapter 06 真正讲的是 **shape 怎样接上 collision pipeline**，不是 body 直接互相碰撞。

## One-Screen Chapter Map

```text
body gives motion, shape gives geometry
                    |
                    v
Model.shape_transform / shape_body / shape_type / shape_margin / shape_gap / shape_world
                    +
                state.body_q
                    |
                    v
          compute_shape_aabbs(...)
                    |
                    v
 shape world transform + expanded AABB + geom_data
                    |
                    v
               broad phase
                    |
                    v
   broad_phase_shape_pairs + pair_count
                    |
                    v
       narrow phase routing by shape_type
                    |
                    v
                ContactData
                    |
                    v
             write_contact(...)
                    |
                    v
                 Contacts
```

## Beginner Path

1. 先看 Stage 1。
   - 想验证什么：为什么 body 不是直接碰撞对象，为什么 `Model` 里要单独保留 shape 元数据。
   - 看完后应该能说：`Model.collide()` 自己不做几何计算，它只是把 body/world state 和静态 shape 数据接到 `CollisionPipeline`。
2. 再看 Stage 2。
   - 想验证什么：`body_q + shape_transform` 怎样长成每个 shape 的 world transform 和 expanded AABB，以及 `margin / gap` 为什么这么早就出现。
   - 看完后应该能说：broad phase 看到的是 shape 的 world AABB，不是 body 名单；AABB 还会按 `margin + gap` 提前膨胀。
3. 再看 Stage 3。
   - 想验证什么：broad phase 到底写出了什么，为什么它只是一张 maybe-list。
   - 看完后应该能说：它只写 `candidate_pair` 和 `candidate_pair_count`，还没有真正 contact geometry。
4. 最后看 Stage 4 和 Stage 5。
   - 想验证什么：narrow phase 怎样把不同几何分支重新压回统一的 `ContactData`，writer 又怎样把它写进 `Contacts`。
   - 看完后应该能说：chapter 06 的终点不是某个 GJK 函数，而是 solver 能直接消费的 `Contacts` 数组。

## Main Walkthrough

### Stage 1: 入口先把 runtime body state 和 static shape data 接到一起

**Claim**

`Model` 本身保存的是 shape 的静态描述，`state` 保存的是这一帧 body 的当前位置；`Model.collide()` 只是把这两类东西交给缓存好的 `CollisionPipeline`。第一遍可以直接把这里读成：body 带来“怎么动”，shape 带来“长什么样”。

**Why it matters**

这一步最能纠正新手的第一层误会：不是 body 直接拿去做碰撞，而是 body 提供运动，shape 提供几何，pipeline 再把两者接起来。只有先接受这个边界，后面“一个 body 对应多个 shape pair”才不会显得奇怪。

**Source excerpt**

先看 `Model` 里挂着哪些 shape 级数据：

这里先保留字段声明块干净，因为这一段的重点是看 shape 元数据整体挂在 `Model` 上，而不是逐行解释语法。

```python
self.shape_transform: wp.array[wp.transform] | None = None
self.shape_body: wp.array[wp.int32] | None = None
self.shape_gap: wp.array[wp.float32] | None = None
self.shape_type: wp.array[wp.int32] | None = None
self.shape_margin: wp.array[wp.float32] | None = None
self.shape_collision_group: wp.array[wp.int32] | None = None
self.shape_world: wp.array[wp.int32] | None = None
```

再看 `Model.contacts()` 和 `Model.collide()` 的入口：

以下摘录为教学注释版，注释非原源码。

```python
def contacts(self, collision_pipeline: CollisionPipeline | None = None) -> Contacts:
    if collision_pipeline is not None:
        self._collision_pipeline = collision_pipeline  # 允许外部替换当前 pipeline
    if self._collision_pipeline is None:
        self._init_collision_pipeline()  # 需要时再懒初始化默认 pipeline
    return self._collision_pipeline.contacts()  # 拿到统一 Contacts 缓冲区


def collide(self, state: State, contacts: Contacts | None = None, *, collision_pipeline=None) -> Contacts:
    if collision_pipeline is not None:
        self._collision_pipeline = collision_pipeline  # 显式传入就覆盖当前 pipeline
    if self._collision_pipeline is None:
        self._init_collision_pipeline()  # 保证后面有可用的 collision pipeline
    if contacts is None:
        contacts = self._collision_pipeline.contacts()  # 没给输出缓冲就现拿一块
    self._collision_pipeline.collide(state, contacts)  # 真正的碰撞主线在 pipeline 里
    return contacts  # 把写好的 Contacts 交还调用者
```

**Verification cues**

- `shape_transform / shape_body / shape_type / shape_gap / shape_margin / shape_world` 都挂在 `Model` 上，不在 `State` 上。
- `Model.collide()` 本身没有做 AABB、pair test 或 contact write；它只是把 `state` 和 `contacts` 转交给 `CollisionPipeline.collide()`。
- `Model.contacts()` 负责拿到一个 `Contacts` 缓冲区，所以后面 stage 看到的 writer 都是在往这块缓冲里写。

**Checkpoint**

如果你现在还会把 collision 读成“body 直接互相碰”，先不要继续。第一遍要先把边界拆开：`State` 带来当前运动，`Model` 带来静态 shape 元数据。

**Output passed to next stage**

`CollisionPipeline.collide(state, contacts)` 拿到了两类输入：`state.body_q` 这份 runtime body pose，以及 `model.shape_*` 这份静态 shape 元数据。

### Stage 2: `compute_shape_aabbs(...)` 把 body pose 变成每个 shape 的 world-space 表示

**Claim**

碰撞主线的第一跳不是 pair test，而是先把每个 shape 放到 world 里，并顺手准备两份后面都要用的数据：expanded AABB 给 broad phase，`geom_data + geom_transform` 给 narrow phase。这里 `margin / gap` 之所以这么早出现，就是因为 broad phase 要先知道“多早该把两边视为值得继续检查”。

**Why it matters**

这一步把 chapter 05 留下的 `body_q` 真正接到 collision path。读懂这里之后，你就不会再把 broad phase 想成“直接看 body”，而会知道它看的是 shape 的 world-space 盒子；也会知道 `margin / gap` 不是后面才凭空冒出来的补丁。

**Source excerpt**

先看 `CollisionPipeline.collide()` 是怎样调用这一步的：

以下摘录为教学注释版，注释非原源码。

```python
wp.launch(
    kernel=compute_shape_aabbs,  # 先把每个 shape 放到 world 并算 AABB
    dim=model.shape_count,  # 每个 shape 一次
    inputs=[
        state.body_q,  # 来自刚体状态的 world pose
        model.shape_transform,  # shape 相对 body 的局部 pose
        model.shape_body,  # 这个 shape 挂在哪个 body 上
        model.shape_type,  # narrow phase 后面还要按类型分路
        model.shape_scale,
        model.shape_collision_radius,
        model.shape_source_ptr,
        model.shape_margin,  # 接触厚度设置
        model.shape_gap,  # 提前预警的额外 gap
        model.shape_collision_aabb_lower,
        model.shape_collision_aabb_upper,
    ],
    outputs=[
        self.narrow_phase.shape_aabb_lower,  # broad phase 直接消费的 world AABB
        self.narrow_phase.shape_aabb_upper,
        self.geom_data,  # narrow phase 复用的几何参数
        self.geom_transform,  # narrow phase 复用的 world transform
    ],
)
```

再看 `compute_shape_aabbs(...)` 里面最关键的几行：

```python
rigid_id = shape_body[shape_id]  # 先看这个 shape 是否挂在某个 body 上

if rigid_id == -1:
    X_ws = shape_transform[shape_id]  # world shape 直接用自己的 pose
else:
    X_ws = wp.transform_multiply(body_q[rigid_id], shape_transform[shape_id])  # body pose * local pose = shape world pose

margin = shape_margin[shape_id]  # shape 自己的接触厚度
effective_gap = margin + shape_gap[shape_id]  # broad phase 实际参考的包络半径
margin_vec = wp.vec3(effective_gap, effective_gap, effective_gap)  # 用来膨胀 AABB 的向量

...

aabb_lower[shape_id] = aabb_min_world - margin_vec  # 写入膨胀后的 AABB 下界
aabb_upper[shape_id] = aabb_max_world + margin_vec  # 写入膨胀后的 AABB 上界
geom_data[shape_id] = wp.vec4(geom_scale[0], geom_scale[1], geom_scale[2], margin)  # 打包 narrow phase 还要用的几何参数
geom_xform[shape_id] = X_ws  # 把 shape 的 world pose 留给 narrow phase
```

**Verification cues**

- `shape_body == -1` 的 shape 不挂在任何 body 上，所以直接用 `shape_transform`；这就是地面这类 world shape 的入口。
- AABB 不是只包原始几何，而是额外按 `margin + gap` 膨胀；第一遍先把这件事读成“接触包络会提前放大 maybe-list 的搜索范围”。
- `geom_xform` 和 `geom_data` 不是 broad phase 用的，它们是 narrow phase 后面直接复用的几何输入。

**Checkpoint**

如果你现在还会把 broad phase 想成“直接看 body 有没有碰撞”，先停一下。chapter 06 真正先算的是每个 shape 的 world-space 表示和 expanded AABB。

**Output passed to next stage**

每个 shape 的 `world transform + expanded AABB + geom_data`。其中 AABB 交给 broad phase，`geom_*` 交给 narrow phase。

### Stage 3: broad phase 只写 candidate pairs，不写 contact geometry

**Claim**

broad phase 的工作就是便宜、保守地筛出“值得继续看一眼”的 shape pair，然后把 pair id 写进缓冲区；它的输出仍然只是 candidate 名单。这里的 `candidate pair` 第一遍就直接读成“也许值得进 narrow phase 的一对 shape”。

**Why it matters**

如果这里不读清楚，后面就很容易把“包围盒重叠”和“已经产生接触点”混为一谈。chapter 06 里，这一层必须被读成 cheap filter，而不是 contact generator。

**Source excerpt**

`BroadPhaseAllPairs` 的 kernel 核心就是 world/group 过滤、AABB overlap 和 pair write：

以下摘录为教学注释版，注释非原源码。

```python
if not test_world_and_group_pair(world1, world2, collision_group1, collision_group2):
    return  # 先过滤掉不该互相碰撞的 world/group 组合

if check_aabb_overlap(
    shape_bounding_box_lower[shape1],
    shape_bounding_box_upper[shape1],
    gap1,
    shape_bounding_box_lower[shape2],
    shape_bounding_box_upper[shape2],
    gap2,
):
    if num_filter_pairs > 0 and is_pair_excluded(wp.vec2i(shape1, shape2), filter_pairs, num_filter_pairs):
        return  # 显式排除的 pair 不再继续
    write_pair(wp.vec2i(shape1, shape2), candidate_pair, candidate_pair_count, max_candidate_pair)  # 只把 shape id 写进 maybe-list
```

真正写入时也只是把 pair id 追加到数组里：

```python
def write_pair(pair, candidate_pair, candidate_pair_count, max_candidate_pair):
    pairid = wp.atomic_add(candidate_pair_count, 0, 1)  # 先抢一个输出槽位
    if pairid >= max_candidate_pair:
        return  # 超出容量就丢弃
    candidate_pair[pairid] = pair  # 这里只记录 pair id，还没有接触几何
```

**Verification cues**

- `test_world_and_group_pair(...)` 说明 broad phase 先看 world 和 collision group，而不是先跑重几何。
- `write_pair(...)` 只写 `(shape_a, shape_b)`，完全没有法线、点、距离。
- 在 `CollisionPipeline.collide()` 里，无论 broad phase 是 `nxn` 还是 `sap`，最后都写到同一个 `self.broad_phase_shape_pairs` 和 `self.broad_phase_pair_count`。

**Checkpoint**

如果你现在还会把 broad phase candidate pair 当成“已经有接触点了”，先不要继续。这里写出来的 still 只是 maybe-list，不是 contact geometry。

**Output passed to next stage**

一张 candidate shape pair 名单：`broad_phase_shape_pairs + broad_phase_pair_count`。

### Stage 4: narrow phase 可以分很多支，但会重新收束成统一的 `ContactData`

**Claim**

narrow phase 的确会按 `shape_type` 分路，但 chapter 06 最重要的不是背下每条支路，而是看清：这些分支最后都要把结果压回同一种接触语言，然后才能交给 writer。这里的 `ContactData` 可以先直接读成“某一条接触几何记录”。

**Why it matters**

只有把这一层读成“分路再收束”，你才会明白为什么 solver 根本不想知道 contact 来自 plane-sphere、box-box，还是 mesh/SDF 分支；solver 只想看到统一格式的接触几何。

**Source excerpt**

先看 `NarrowPhase.launch_custom_write()` 的 stage 注释，Newton 自己就把这层写得很直白：

这里先保留路由注释块干净，因为这一段的重点是看 narrow phase 先分流、再收束的整体形状。

```python
# Stage 1: Launch primitive kernel for fast analytical collisions
# This handles sphere-sphere, sphere-capsule, capsule-capsule, plane-sphere, plane-capsule
# and routes remaining pairs to gjk_candidate_pairs and mesh buffers
wp.launch(kernel=self.primitive_kernel, ...)

# Stage 2: Launch GJK/MPR kernel for remaining convex pairs
wp.launch(kernel=self.narrow_phase_kernel, ...)
```

再看一个 primitive path 里怎样把结果装进 `ContactData`：

以下摘录为教学注释版，注释非原源码。

```python
contact_data = ContactData()  # 先创建统一接触记录容器
contact_data.contact_normal_a_to_b = contact_normal  # 写入 A->B 法线
contact_data.radius_eff_a = radius_eff_a  # A 侧有效半径语义
contact_data.radius_eff_b = radius_eff_b  # B 侧有效半径语义
contact_data.margin_a = margin_offset_a  # A 侧 margin / offset
contact_data.margin_b = margin_offset_b  # B 侧 margin / offset
contact_data.shape_a = shape_a  # 记录这条接触属于哪对 shape
contact_data.shape_b = shape_b
contact_data.gap_sum = gap_sum  # 两边 gap 语义也一起带下去

contact_data.contact_point_center = contact_pos_0  # world-space 接触中心
contact_data.contact_distance = contact_dist_0  # 当前 signed distance / penetration
contact_0_valid = contact_passes_gap_check(contact_data)  # 最后再过一次 gap 检查
```

`ContactData` 自己定义的字段也很清楚：

```python
class ContactData:
    contact_point_center: wp.vec3
    contact_normal_a_to_b: wp.vec3
    contact_distance: float
    radius_eff_a: float
    radius_eff_b: float
    margin_a: float
    margin_b: float
    shape_a: int
    shape_b: int
    gap_sum: float
```

**Verification cues**

- `launch_custom_write()` 的第一层任务是 triage：primitive pair 先处理，剩余 convex pair 送去 GJK/MPR，mesh / SDF / heightfield 再走更重的分支。
- 不管前面怎么分路，`ContactData` 都在描述同一件事：接触中心、法线、距离、两边 shape id、margin/gap 语义。
- `contact_passes_gap_check(contact_data)` 说明 even after narrow phase，contact 还要再经过一次 gap 语义检查。

**Checkpoint**

如果你现在还会把 narrow phase 理解成“一条分支对应一种完全不同的输出”，先停一下。chapter 06 在这里最重要的动作是“先分路，再收束回统一的 `ContactData`”。

**Output passed to next stage**

一条条统一格式的 `ContactData` 记录，已经足够让 writer 把它们写进最终 `Contacts` 缓冲区。

### Stage 5: `write_contact()` 把 `ContactData` 变成 solver 能直接消费的 `Contacts`

**Claim**

`write_contact()` 是 chapter 06 真正的终点：它把 world-space contact geometry 转成 `Contacts` 里的 shape id、body-frame 接触点、body-frame offset、world-frame 法线和 margin 信息。第一遍直接把它读成 collision pipeline 的 writer 即可。

**Why it matters**

读到这里，你就能明白 `07` 和 `08` 为什么都从 `Contacts` 继续，而不是各自重新理解 mesh triangle、GJK simplex 或其它 narrow-phase 内部状态。broad phase、narrow phase、writer 之所以分三段，就是因为它们在负责三种完全不同的工作：筛选、求几何、写统一 handoff。

**Source excerpt**

`write_contact()` 里最关键的 handoff 就是下面这一段：

以下摘录为教学注释版，注释非原源码。

```python
gap_a = writer_data.shape_gap[contact_data.shape_a]  # 取 A 侧 shape 的 gap 设置
gap_b = writer_data.shape_gap[contact_data.shape_b]  # 取 B 侧 shape 的 gap 设置
contact_gap = gap_a + gap_b  # 两边合起来的写入门槛

if index < 0:
    if d > contact_gap:
        return  # 还没近到需要写 contact
    index = wp.atomic_add(writer_data.contact_count, 0, 1)  # 抢一个输出 contact 槽位

writer_data.out_shape0[index] = contact_data.shape_a  # 先写这条 contact 属于哪对 shape
writer_data.out_shape1[index] = contact_data.shape_b

body0 = writer_data.shape_body[contact_data.shape_a]  # 找到 A 侧挂在哪个 body 上
body1 = writer_data.shape_body[contact_data.shape_b]  # 找到 B 侧挂在哪个 body 上
X_bw_a = wp.transform_identity() if body0 == -1 else wp.transform_inverse(writer_data.body_q[body0])  # world -> body0 frame
X_bw_b = wp.transform_identity() if body1 == -1 else wp.transform_inverse(writer_data.body_q[body1])  # world -> body1 frame

writer_data.out_point0[index] = wp.transform_point(X_bw_a, a_contact_world)  # world 接触点 -> A 侧 body frame
writer_data.out_point1[index] = wp.transform_point(X_bw_b, b_contact_world)  # world 接触点 -> B 侧 body frame
writer_data.out_normal[index] = contact_normal  # 法线仍保留在 world frame
writer_data.out_margin0[index] = offset_mag_a  # A 侧实际 margin / offset
writer_data.out_margin1[index] = offset_mag_b  # B 侧实际 margin / offset
```

而 `Contacts` 里提前分配好的，就是 solver 后面要读的这些数组：

这里保留字段声明块的整洁外观，因为读者只需要一眼看出 writer 最终会写哪些公共槽位。

```python
self.rigid_contact_shape0 = wp.full(rigid_contact_max, -1, dtype=wp.int32)
self.rigid_contact_shape1 = wp.full(rigid_contact_max, -1, dtype=wp.int32)
self.rigid_contact_point0 = wp.zeros(rigid_contact_max, dtype=wp.vec3)
self.rigid_contact_point1 = wp.zeros(rigid_contact_max, dtype=wp.vec3)
self.rigid_contact_offset0 = wp.zeros(rigid_contact_max, dtype=wp.vec3)
self.rigid_contact_offset1 = wp.zeros(rigid_contact_max, dtype=wp.vec3)
self.rigid_contact_normal = wp.zeros(rigid_contact_max, dtype=wp.vec3)
self.rigid_contact_margin0 = wp.zeros(rigid_contact_max, dtype=wp.float32)
self.rigid_contact_margin1 = wp.zeros(rigid_contact_max, dtype=wp.float32)
```

**Verification cues**

- `out_point0 / out_point1` 是 body frame 点，不是 world frame 点；writer 在这里显式做了 `body_q` 的逆变换。
- `out_normal` 仍然保留 world-frame 法线方向，这和文档里的 contact geometry 描述对得上。
- `Contacts.clear()` 每帧先把计数清零，所以 writer 只需要覆盖 `[0, rigid_contact_count)` 这段活跃区间。

**Checkpoint**

如果你现在还会把 collision pipeline 的终点想成某个 primitive/GJK 内部结构，先不要继续。chapter 06 的真正终点是统一的 `Contacts` 公共缓冲区。

**Output passed to next stage**

`Contacts.rigid_contact_count` 以及 `rigid_contact_shape0/1`、`rigid_contact_point0/1`、`rigid_contact_offset0/1`、`rigid_contact_normal`、`rigid_contact_margin0/1`。这就是 `07` 和 `08` 的共同输入。

## Object Ledger

| 对象 | 谁生产 | 谁消费 | 盯哪些字段 |
|------|--------|--------|------------|
| `state.body_q` | `State` / 上一章的刚体状态更新 | `compute_shape_aabbs(...)`、`write_contact()` | body 当前 world pose |
| `model.shape_transform / shape_body / shape_type / shape_margin / shape_gap / shape_world / shape_collision_group` | `ModelBuilder.finalize()` 后挂到 `Model` | `compute_shape_aabbs(...)`、broad phase、narrow phase、writer | 局部位姿、父 body、几何类型、margin/gap、world/group 过滤 |
| `shape_aabb_lower / shape_aabb_upper` | `compute_shape_aabbs(...)` | broad phase | 这是按 `margin + gap` 膨胀后的 world AABB |
| `geom_transform / geom_data` | `compute_shape_aabbs(...)` | narrow phase | world transform、scale、margin |
| `broad_phase_shape_pairs` | broad phase (`explicit` / `nxn` / `sap`) | `NarrowPhase.launch_custom_write()` | 这里只存 shape pair id |
| `ContactData` | narrow phase 各分支 | `write_contact()` | `contact_point_center`、`contact_normal_a_to_b`、`contact_distance`、`shape_a/b`、`gap_sum` |
| `Contacts` | `write_contact()` 写入，`CollisionPipeline.contacts()` 分配 | `07_constraints_contacts_math`、`08_rigid_solvers` | `rigid_contact_count`、`shape0/1`、`point0/1`、`offset0/1`、`normal`、`margin0/1` |

## Stop Here

读到这里就已经够 chapter 06 的 80-90% 了。

如果你现在能用自己的话讲顺下面这句话，这一章的 beginner 目标就完成了：

```text
body 提供运动，真正直接参加碰撞的是 shape；
body_q 先把挂在 body 上的 shape 放到 world 里；
broad phase 只写出“可能相撞”的 candidate pairs；
narrow phase 再把这些 pair 变成统一的 ContactData；
write_contact() 最后把它们压进 Contacts，交给后面的 contact math 和 solver。
```

这时你已经不需要频繁跳回上游 repo，也能解释 chapter 06 的主干为什么成立。

## Go Deeper

如果你还想继续精确追源码，再去 `source-walkthrough-deep.md`：

- 想保留 cross-repo 精确锚点：看 `Fast Deep Index`
- 想逐跳追 `Model.collide()` 到 `Contacts` 的 exact handoff：看 `Exact Handoff Trace`
- 想分清哪些 branch 第一遍可以跳过：看 `Optional Branches`
- 想逐条核对这里的 claim：看 `Verification Anchors`
