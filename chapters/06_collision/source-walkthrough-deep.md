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

# 06 碰撞系统 深读锚点版

这份 deep walkthrough 保留精确 cross-repo anchors、可选分支和 exact handoff trace。第一次读 chapter 06 时，建议先把 `source-walkthrough.md` 读完，再回来用这一页验证细节。

## Fast Deep Index

| ID | repo@commit path | symbol | why it matters |
|----|------------------|--------|----------------|
| D1 | `newton@1a230702 newton/_src/sim/model.py` | `Model.contacts`, `Model.collide` | 公开入口，决定 `Contacts` 何时分配、pipeline 何时被调用 |
| D2 | `newton@1a230702 newton/_src/sim/collide.py` | `CollisionPipeline.__init__` | broad phase mode 选择、narrow phase 构造、主缓冲分配 |
| D3 | `newton@1a230702 newton/_src/sim/collide.py` | `compute_shape_aabbs` | `body_q + shape_transform` 到 world transform / expanded AABB / `geom_*` |
| D4 | `newton@1a230702 newton/_src/geometry/broad_phase_common.py` | `check_aabb_overlap`, `test_world_and_group_pair`, `write_pair` | broad phase 的共同 contract |
| D5 | `newton@1a230702 newton/_src/geometry/broad_phase_nxn.py` | `_nxn_broadphase_kernel`, `BroadPhaseAllPairs.launch`, `BroadPhaseExplicit.launch` | all-pairs / explicit 路径怎样写 candidate pairs |
| D6 | `newton@1a230702 newton/_src/geometry/broad_phase_sap.py` | `_sap_broadphase_kernel`, `BroadPhaseSAP.launch` | SAP 路径怎样在不同算法下保持同样输出边界 |
| D7 | `newton@1a230702 newton/_src/geometry/narrow_phase.py` | `NarrowPhase.launch_custom_write` | pair triage、branch fan-out、writer fan-in |
| D8 | `newton@1a230702 newton/_src/geometry/contact_data.py` | `ContactData`, `contact_passes_gap_check` | narrow phase 和 writer 之间的统一中间表示 |
| D9 | `newton@1a230702 newton/_src/sim/collide.py` | `write_contact` | `ContactData -> Contacts` 的最终 handoff |
| D10 | `newton@1a230702 newton/_src/sim/contacts.py` | `Contacts`, `Contacts.clear` | solver-facing buffer layout 和每帧 reset 语义 |
| D11 | `newton@1a230702 docs/concepts/collisions.rst` | `Contact Geometry`, `margin-gap-semantics`, `Collision Pipeline Details` | 概念文档和源码 handoff 的对照证据 |

## Exact Handoff Trace

### 1. Public entry: `Model.contacts()` 和 `Model.collide()`

- 入口符号：`newton/_src/sim/model.py` 的 `Model.contacts`、`Model.collide`
- 精确锚点：`newton/_src/sim/model.py:951-1003`
- 前置初始化：默认 pipeline 在 `_init_collision_pipeline()` 里用 `CollisionPipeline(self, broad_phase="explicit")` 创建，见 `newton/_src/sim/model.py:947-949`
- 真正 handoff：`Model.collide()` 如果没传 `contacts`，先 `self._collision_pipeline.contacts()` 分配一块，再调用 `self._collision_pipeline.collide(state, contacts)`

这里最关键的边界是：`Model` 不做碰撞算法，它只决定 “用哪条 pipeline” 和 “把哪份 `state` / `contacts` 交过去”。

### 2. Pipeline construction: broad phase mode、narrow phase、主缓冲

- 入口符号：`newton/_src/sim/collide.py` 的 `CollisionPipeline.__init__`
- 精确锚点：`newton/_src/sim/collide.py:596-705`, `newton/_src/sim/collide.py:730-756`
- exact handoff：
  - `broad_phase == "explicit"` 时走 `BroadPhaseExplicit()`，消费 `shape_contact_pairs`
  - `broad_phase == "nxn"` 时构造 `BroadPhaseAllPairs(shape_world, shape_flags=...)`
  - `broad_phase == "sap"` 时构造 `BroadPhaseSAP(shape_world, shape_flags=...)`
  - 无论哪种 broad phase，都会接着构造 `NarrowPhase(..., contact_writer_warp_func=write_contact, ...)`
  - 最后统一分配 `self.broad_phase_pair_count`、`self.broad_phase_shape_pairs`、`self.geom_data`、`self.geom_transform`

深读时要特别注意：broad phase 是可插拔的，但 `NarrowPhase` 和 `write_contact` 组成的下游 contract 是共享的。

### 3. AABB/worldization: `compute_shape_aabbs`

- 入口符号：`newton/_src/sim/collide.py` 的 `CollisionPipeline.collide`、`compute_shape_aabbs`
- 精确锚点：调用点 `newton/_src/sim/collide.py:820-845`，kernel 本体 `newton/_src/sim/collide.py:150-262`
- exact handoff：
  - `state.body_q`、`model.shape_transform`、`model.shape_body`、`model.shape_type`、`model.shape_margin`、`model.shape_gap` 被一起送进 kernel
  - `shape_body == -1` 时直接用 `shape_transform`；否则用 `body_q[rigid_id] * shape_transform[shape_id]`
  - kernel 同时写两组输出：
    - `self.narrow_phase.shape_aabb_lower/upper`
    - `self.geom_data`、`self.geom_transform`
  - AABB 膨胀是按 `margin + gap` 做的，见 `effective_gap = margin + shape_gap[shape_id]` 和 `aabb_lower/upper = ... +/- margin_vec`

这一步是 chapter 05 到 chapter 06 的真正桥：body pose 第一次长成 “shape 现在在 world 哪儿”。

### 4. Broad phase contract: candidate shape pairs only

- 共同 helper：`newton/_src/geometry/broad_phase_common.py:18-180`
- `nxn` 写 pair：`newton/_src/geometry/broad_phase_nxn.py:175-214`
- `explicit` 写 pair：`newton/_src/geometry/broad_phase_nxn.py:35-66`, `newton/_src/geometry/broad_phase_nxn.py:402-459`
- `sap` 写 pair：`newton/_src/geometry/broad_phase_sap.py:248-281`, `newton/_src/geometry/broad_phase_sap.py:368-392`, `newton/_src/geometry/broad_phase_sap.py:510-671`
- exact handoff：
  - 先过 `test_world_and_group_pair(...)`
  - 再过 `check_aabb_overlap(...)`
  - 再过 `is_pair_excluded(...)`（如果有 exclusion list）
  - 最后统一 `write_pair(pair, candidate_pair, candidate_pair_count, max_candidate_pair)`

这也是主 walkthrough 里“broad phase 只写 candidate pairs”的精确证据链。

### 5. Narrow phase triage: `candidate_pair -> ContactData`

- 入口符号：`newton/_src/geometry/narrow_phase.py` 的 `NarrowPhase.launch_custom_write`
- 精确锚点：`newton/_src/geometry/narrow_phase.py:1695-2123`
- triage 关键段：
  - primitive fast path：`newton/_src/geometry/narrow_phase.py:1752-1785`
  - GJK/MPR path：`newton/_src/geometry/narrow_phase.py:1789-1812`
  - mesh / heightfield / reduction / mesh-mesh export：`newton/_src/geometry/narrow_phase.py:1814-2084`
  - hydroelastic launch：`newton/_src/geometry/narrow_phase.py:2085-2097`
- `ContactData` 定义：`newton/_src/geometry/contact_data.py:18-56`
- gap check：`newton/_src/geometry/contact_data.py:87-117`
- concrete creation examples：
  - primitive multi-contact path：`newton/_src/geometry/narrow_phase.py:520-569`
  - mesh-plane path：`newton/_src/geometry/narrow_phase.py:1172-1186`

exact handoff 的阅读方式建议是：先看 `launch_custom_write()` 顶层 stage comment，再挑一个具体 branch（primitive path 最容易），最后回到 `ContactData` 定义确认统一字段。

### 6. Writer handoff: `ContactData -> Contacts`

- writer 组装点：`newton/_src/sim/collide.py:888-942`
- writer 函数本体：`newton/_src/sim/collide.py:62-145`
- final buffer 定义：`newton/_src/sim/contacts.py:67-189`
- clear 语义：`newton/_src/sim/contacts.py:221-271`
- exact handoff：
  - `CollisionPipeline.collide()` 先把 `state.body_q`、`model.shape_body`、`model.shape_gap` 和 `contacts.rigid_contact_*` 数组装进 `ContactWriterData`
  - `NarrowPhase.launch_custom_write(...)` 直接调用 `write_contact(...)`
  - `write_contact(...)` 先做 pair gap 判断，再把 world-space 接触几何写成：
    - `out_shape0 / out_shape1`
    - `out_point0 / out_point1`（body frame）
    - `out_offset0 / out_offset1`
    - `out_normal`（world frame）
    - `out_margin0 / out_margin1`

这一步之后，chapter 06 的代码主线就已经封口，下一跳自然是 `07` 和 `08`。

### 7. Concept doc cross-check

- 两阶段 pipeline 概览：`docs/concepts/collisions.rst:32-119`
- `margin` / `gap` 语义：`docs/concepts/collisions.rst:1014-1053`
- `Contacts` 数据布局：`docs/concepts/collisions.rst:1229-1255`
- expert API / custom writer：`docs/concepts/collisions.rst:1777-1855`

如果你想验证这章不是“只从源码倒推故事”，这四段文档是最好的 cross-check。

## Optional Branches

### Branch A: broad phase 变体

- first pass 可跳过：`sap` 的 segmented sort / tile sort 细节，见 `newton/_src/geometry/broad_phase_sap.py:594-671`
- first pass 只要守住：`explicit`、`nxn`、`sap` 最后都写进 `candidate_pair` 和 `candidate_pair_count`
- 真要比较三者差别，再回看：
  - `CollisionPipeline.__init__` 的 mode 选择：`newton/_src/sim/collide.py:596-629`
  - `BroadPhaseAllPairs.launch`：`newton/_src/geometry/broad_phase_nxn.py:305-385`
  - `BroadPhaseSAP.launch`：`newton/_src/geometry/broad_phase_sap.py:510-671`

### Branch B: narrow phase 重分支

- first pass 可跳过：mesh-triangle overlap、global contact reduction、mesh-mesh SDF、hydroelastic
- first pass 只要守住：`launch_custom_write()` 先 triage，再把结果统一交给 writer
- 真要深挖，再回看：
  - primitive / GJK split：`newton/_src/geometry/narrow_phase.py:1752-1812`
  - mesh / reduction 主体：`newton/_src/geometry/narrow_phase.py:1814-2084`
  - hydroelastic：`newton/_src/geometry/narrow_phase.py:2085-2097`

### Branch C: `Contacts` 的扩展能力

- first pass 可跳过：`rigid_contact_diff_*`、`force` 等扩展属性
- 真要看再回去：`newton/_src/sim/contacts.py:153-215`
- chapter 06 主线只需要默认 rigid contact arrays，不需要把 differentiable contacts 一起背下来

## Verification Anchors

| 想验证的 claim | 直接打开哪里 |
|----------------|--------------|
| body 不是直接碰撞输入，shape 才是 | `newton/_src/sim/model.py:193-257`, `newton/_src/sim/model.py:977-1003` |
| broad phase 看到的是按 `margin + gap` 膨胀后的 AABB | `newton/_src/sim/collide.py:192-196`, `newton/_src/sim/collide.py:257-262`, `docs/concepts/collisions.rst:1014-1053` |
| broad phase 只写 candidate pairs，不写 contact geometry | `newton/_src/geometry/broad_phase_common.py:117-128`, `newton/_src/geometry/broad_phase_nxn.py:198-214`, `newton/_src/geometry/broad_phase_sap.py:381-392` |
| narrow phase 再复杂也要收束到统一 contact language | `newton/_src/geometry/contact_data.py:18-56`, `newton/_src/geometry/narrow_phase.py:520-569`, `newton/_src/geometry/narrow_phase.py:1172-1186` |
| final `Contacts` 里点在 body frame、法线在 world frame | `newton/_src/sim/collide.py:118-134`, `newton/_src/sim/contacts.py:120-147`, `docs/concepts/collisions.rst:105-119`, `docs/concepts/collisions.rst:1238-1255` |
