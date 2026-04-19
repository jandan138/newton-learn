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

如果你是第一次读这一章，最好先配合 `principle.md` 一起读；如果你是直接跳进源码走读也没关系，但一旦发现术语开始变快，就先回到 `principle.md` 把对象关系补齐，再回来追源码。

这份主 walkthrough 是给第一次追 chapter 06 源码的人准备的。目标不是把 every branch 都摊平，而是让你不离开这页，也能把 collision 主干读成一条连续 handoff：`body/world state -> candidate pairs -> ContactData -> Contacts`。

## What This Walkthrough Follows

只追这一条主线：

```text
state.body_q + Model.shape_*
-> compute_shape_aabbs(...)
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
   - 想验证什么：`Model` 里哪些数组是静态 shape 数据，`state` 里哪些是每帧 body/world state。
   - 看完后应该能说：`Model.collide()` 自己不做几何计算，它只是把这两类输入接到 `CollisionPipeline`。
2. 再看 Stage 2。
   - 想验证什么：`body_q + shape_transform` 怎样长成每个 shape 的 world transform 和 expanded AABB。
   - 看完后应该能说：broad phase 看到的是 shape 的 world AABB，不是 body 名单。
3. 再看 Stage 3。
   - 想验证什么：broad phase 到底写出了什么。
   - 看完后应该能说：它只写 `candidate_pair` 和 `candidate_pair_count`，还没有真正 contact geometry。
4. 最后看 Stage 4 和 Stage 5。
   - 想验证什么：narrow phase 怎样把不同几何分支重新压回统一的 `ContactData`，以及 `write_contact()` 怎样写进 `Contacts`。
   - 看完后应该能说：chapter 06 的终点不是某个 GJK 函数，而是 solver 能直接消费的 `Contacts` 数组。

## Main Walkthrough

### Stage 1: 入口先把 runtime body state 和 static shape data 接到一起

**Claim**

`Model` 本身保存的是 shape 的静态描述，`state` 保存的是这一帧 body 的当前位置；`Model.collide()` 只是把这两类东西交给缓存好的 `CollisionPipeline`。

**Why it matters**

这一步最能纠正新手的第一层误会：不是 body 直接拿去做碰撞，而是 body 提供运动，shape 提供几何，pipeline 再把两者接起来。

**Source excerpt**

先看 `Model` 里挂着哪些 shape 级数据：

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

```python
def contacts(self, collision_pipeline: CollisionPipeline | None = None) -> Contacts:
    if collision_pipeline is not None:
        self._collision_pipeline = collision_pipeline
    if self._collision_pipeline is None:
        self._init_collision_pipeline()
    return self._collision_pipeline.contacts()


def collide(self, state: State, contacts: Contacts | None = None, *, collision_pipeline=None) -> Contacts:
    if collision_pipeline is not None:
        self._collision_pipeline = collision_pipeline
    if self._collision_pipeline is None:
        self._init_collision_pipeline()
    if contacts is None:
        contacts = self._collision_pipeline.contacts()
    self._collision_pipeline.collide(state, contacts)
    return contacts
```

**Verification cues**

- `shape_transform / shape_body / shape_type / shape_gap / shape_margin / shape_world` 都挂在 `Model` 上，不在 `State` 上。
- `Model.collide()` 本身没有做 AABB、pair test 或 contact write；它只是把 `state` 和 `contacts` 转交给 `CollisionPipeline.collide()`。
- `Model.contacts()` 负责拿到一个 `Contacts` 缓冲区，所以后面 stage 看到的 writer 都是在往这块缓冲里写。

**Output passed to next stage**

`CollisionPipeline.collide(state, contacts)` 拿到了两类输入：`state.body_q` 这份 runtime body pose，以及 `model.shape_*` 这份静态 shape 元数据。

### Stage 2: `compute_shape_aabbs(...)` 把 body pose 变成每个 shape 的 world-space 表示

**Claim**

碰撞主线的第一跳不是 pair test，而是先把每个 shape 放到 world 里，并顺手准备两份后面都要用的数据：expanded AABB 给 broad phase，`geom_data + geom_transform` 给 narrow phase。

**Why it matters**

这一步把 chapter 05 留下的 `body_q` 真正接到 collision path。读懂这里之后，你就不会再把 broad phase 想成“直接看 body”，而会知道它看的是 shape 的 world-space 盒子。

**Source excerpt**

先看 `CollisionPipeline.collide()` 是怎样调用这一步的：

```python
wp.launch(
    kernel=compute_shape_aabbs,
    dim=model.shape_count,
    inputs=[
        state.body_q,
        model.shape_transform,
        model.shape_body,
        model.shape_type,
        model.shape_scale,
        model.shape_collision_radius,
        model.shape_source_ptr,
        model.shape_margin,
        model.shape_gap,
        model.shape_collision_aabb_lower,
        model.shape_collision_aabb_upper,
    ],
    outputs=[
        self.narrow_phase.shape_aabb_lower,
        self.narrow_phase.shape_aabb_upper,
        self.geom_data,
        self.geom_transform,
    ],
)
```

再看 `compute_shape_aabbs(...)` 里面最关键的几行：

```python
rigid_id = shape_body[shape_id]

if rigid_id == -1:
    X_ws = shape_transform[shape_id]
else:
    X_ws = wp.transform_multiply(body_q[rigid_id], shape_transform[shape_id])

margin = shape_margin[shape_id]
effective_gap = margin + shape_gap[shape_id]
margin_vec = wp.vec3(effective_gap, effective_gap, effective_gap)

...

aabb_lower[shape_id] = aabb_min_world - margin_vec
aabb_upper[shape_id] = aabb_max_world + margin_vec
geom_data[shape_id] = wp.vec4(geom_scale[0], geom_scale[1], geom_scale[2], margin)
geom_xform[shape_id] = X_ws
```

**Verification cues**

- `shape_body == -1` 的 shape 不挂在任何 body 上，所以直接用 `shape_transform`；这就是地面这类 world shape 的入口。
- AABB 不是只包原始几何，而是额外按 `margin + gap` 膨胀；这和文档里“`margin` 决定 where，`gap` 决定 when”是同一套语义。
- `geom_xform` 和 `geom_data` 不是 broad phase 用的，它们是 narrow phase 后面直接复用的几何输入。

**Output passed to next stage**

每个 shape 的 `world transform + expanded AABB + geom_data`。其中 AABB 交给 broad phase，`geom_*` 交给 narrow phase。

### Stage 3: broad phase 只写 candidate pairs，不写 contact geometry

**Claim**

broad phase 的工作就是便宜、保守地筛出“值得继续看一眼”的 shape pair，然后把 pair id 写进缓冲区；它的输出仍然只是 candidate 名单。

**Why it matters**

如果这里不读清楚，后面就很容易把“包围盒重叠”和“已经产生接触点”混为一谈。chapter 06 里，这一层必须被读成 cheap filter，而不是 contact generator。

**Source excerpt**

`BroadPhaseAllPairs` 的 kernel 核心就是 world/group 过滤、AABB overlap 和 pair write：

```python
if not test_world_and_group_pair(world1, world2, collision_group1, collision_group2):
    return

if check_aabb_overlap(
    shape_bounding_box_lower[shape1],
    shape_bounding_box_upper[shape1],
    gap1,
    shape_bounding_box_lower[shape2],
    shape_bounding_box_upper[shape2],
    gap2,
):
    if num_filter_pairs > 0 and is_pair_excluded(wp.vec2i(shape1, shape2), filter_pairs, num_filter_pairs):
        return
    write_pair(wp.vec2i(shape1, shape2), candidate_pair, candidate_pair_count, max_candidate_pair)
```

真正写入时也只是把 pair id 追加到数组里：

```python
def write_pair(pair, candidate_pair, candidate_pair_count, max_candidate_pair):
    pairid = wp.atomic_add(candidate_pair_count, 0, 1)
    if pairid >= max_candidate_pair:
        return
    candidate_pair[pairid] = pair
```

**Verification cues**

- `test_world_and_group_pair(...)` 说明 broad phase 先看 world 和 collision group，而不是先跑重几何。
- `write_pair(...)` 只写 `(shape_a, shape_b)`，完全没有法线、点、距离。
- 在 `CollisionPipeline.collide()` 里，无论 broad phase 是 `nxn` 还是 `sap`，最后都写到同一个 `self.broad_phase_shape_pairs` 和 `self.broad_phase_pair_count`。

**Output passed to next stage**

一张 candidate shape pair 名单：`broad_phase_shape_pairs + broad_phase_pair_count`。

### Stage 4: narrow phase 可以分很多支，但会重新收束成统一的 `ContactData`

**Claim**

narrow phase 的确会按 `shape_type` 分路，但 chapter 06 最重要的不是背下每条支路，而是看清：这些分支最后都要把结果压回同一种接触语言，然后才能交给 writer。

**Why it matters**

只有把这一层读成“分路再收束”，你才会明白为什么 solver 根本不想知道 contact 来自 plane-sphere、box-box，还是 mesh/SDF 分支；solver 只想看到统一格式的接触几何。

**Source excerpt**

先看 `NarrowPhase.launch_custom_write()` 的 stage 注释，Newton 自己就把这层写得很直白：

```python
# Stage 1: Launch primitive kernel for fast analytical collisions
# This handles sphere-sphere, sphere-capsule, capsule-capsule, plane-sphere, plane-capsule
# and routes remaining pairs to gjk_candidate_pairs and mesh buffers
wp.launch(kernel=self.primitive_kernel, ...)

# Stage 2: Launch GJK/MPR kernel for remaining convex pairs
wp.launch(kernel=self.narrow_phase_kernel, ...)
```

再看一个 primitive path 里怎样把结果装进 `ContactData`：

```python
contact_data = ContactData()
contact_data.contact_normal_a_to_b = contact_normal
contact_data.radius_eff_a = radius_eff_a
contact_data.radius_eff_b = radius_eff_b
contact_data.margin_a = margin_offset_a
contact_data.margin_b = margin_offset_b
contact_data.shape_a = shape_a
contact_data.shape_b = shape_b
contact_data.gap_sum = gap_sum

contact_data.contact_point_center = contact_pos_0
contact_data.contact_distance = contact_dist_0
contact_0_valid = contact_passes_gap_check(contact_data)
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

**Output passed to next stage**

一条条统一格式的 `ContactData` 记录，已经足够让 writer 把它们写进最终 `Contacts` 缓冲区。

### Stage 5: `write_contact()` 把 `ContactData` 变成 solver 能直接消费的 `Contacts`

**Claim**

`write_contact()` 是 chapter 06 真正的终点：它把 world-space contact geometry 转成 `Contacts` 里的 shape id、body-frame 接触点、body-frame offset、world-frame 法线和 margin 信息。

**Why it matters**

读到这里，你就能明白 `07` 和 `08` 为什么都从 `Contacts` 继续，而不是各自重新理解 mesh triangle、GJK simplex 或其它 narrow-phase 内部状态。

**Source excerpt**

`write_contact()` 里最关键的 handoff 就是下面这一段：

```python
gap_a = writer_data.shape_gap[contact_data.shape_a]
gap_b = writer_data.shape_gap[contact_data.shape_b]
contact_gap = gap_a + gap_b

if index < 0:
    if d > contact_gap:
        return
    index = wp.atomic_add(writer_data.contact_count, 0, 1)

writer_data.out_shape0[index] = contact_data.shape_a
writer_data.out_shape1[index] = contact_data.shape_b

body0 = writer_data.shape_body[contact_data.shape_a]
body1 = writer_data.shape_body[contact_data.shape_b]
X_bw_a = wp.transform_identity() if body0 == -1 else wp.transform_inverse(writer_data.body_q[body0])
X_bw_b = wp.transform_identity() if body1 == -1 else wp.transform_inverse(writer_data.body_q[body1])

writer_data.out_point0[index] = wp.transform_point(X_bw_a, a_contact_world)
writer_data.out_point1[index] = wp.transform_point(X_bw_b, b_contact_world)
writer_data.out_normal[index] = contact_normal
writer_data.out_margin0[index] = offset_mag_a
writer_data.out_margin1[index] = offset_mag_b
```

而 `Contacts` 里提前分配好的，就是 solver 后面要读的这些数组：

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
body_q 先把挂在 body 上的 shape 放到 world 里；
broad phase 再写出“可能相撞”的 shape pair；
narrow phase 把这些 pair 变成统一的 ContactData；
write_contact() 最后把它们压进 Contacts，交给后面的 contact math 和 solver。
```

这时你已经不需要频繁跳回上游 repo，也能解释 chapter 06 的主干为什么成立。

## Go Deeper

如果你还想继续精确追源码，再去 `source-walkthrough-deep.md`：

- 想保留 cross-repo 精确锚点：看 `Fast Deep Index`
- 想逐跳追 `Model.collide()` 到 `Contacts` 的 exact handoff：看 `Exact Handoff Trace`
- 想分清哪些 branch 第一遍可以跳过：看 `Optional Branches`
- 想逐条核对这里的 claim：看 `Verification Anchors`
