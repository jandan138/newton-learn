---
chapter: 07
title: 约束与接触数学总论
last_updated: 2026-04-19
source_paths:
  - docs/concepts/collisions.rst
  - newton/_src/sim/collide.py
  - newton/_src/sim/contacts.py
  - newton/_src/solvers/kamino/_src/geometry/contacts.py
  - newton/_src/solvers/kamino/_src/geometry/unified.py
  - newton/_src/solvers/kamino/_src/kinematics/constraints.py
  - newton/_src/solvers/kamino/_src/kinematics/jacobians.py
  - newton/_src/solvers/kamino/_src/core/math.py
  - newton/_src/solvers/kamino/_src/dynamics/delassus.py
  - newton/_src/solvers/kamino/solver_kamino.py
  - newton/_src/solvers/kamino/_src/solver_kamino_impl.py
paper_keys: []
newton_commit: 1a230702
---

# 07 约束与接触数学总论 深读锚点版

这份 deep walkthrough 保留精确 cross-repo anchors、可选分支和 verification-heavy trace。第一次读 chapter 07 时，建议先把 `source-walkthrough.md` 读完，再回来用这一页校准细节。

## Fast Deep Index

| ID | repo@commit path | symbol | why it matters |
|----|------------------|--------|----------------|
| D1 | `newton@1a230702 newton/_src/sim/contacts.py` | `Contacts` | chapter 06 留下的 runtime contact layout |
| D2 | `newton@1a230702 newton/_src/solvers/kamino/_src/geometry/contacts.py` | `ContactsKaminoData` | solver-facing contact container 的字段定义 |
| D3 | `newton@1a230702 newton/_src/solvers/kamino/_src/geometry/contacts.py` | `make_contact_frame_znorm` | 法向 + 两条切向的局部 frame 来源 |
| D4 | `newton@1a230702 newton/_src/solvers/kamino/_src/geometry/contacts.py` | `convert_contacts_newton_to_kamino` | `Contacts -> ContactsKamino` 的正式桥接 |
| D5 | `newton@1a230702 newton/_src/solvers/kamino/_src/kinematics/constraints.py` | `make_unilateral_constraints_info`, `_update_constraints_info` | contact rows 为何是 `3 * nc` |
| D6 | `newton@1a230702 newton/_src/solvers/kamino/_src/core/math.py` | `contact_wrench_matrix_from_points` | 接触点到质心的 lever arm 怎样进入数学 |
| D7 | `newton@1a230702 newton/_src/solvers/kamino/_src/kinematics/jacobians.py` | `_build_contact_jacobians_dense`, `_build_contact_jacobians_sparse` | contact Jacobian block 的真实写法 |
| D8 | `newton@1a230702 newton/_src/solvers/kamino/_src/dynamics/delassus.py` | `_build_delassus_elementwise_dense`, `_build_delassus_elementwise_sparse` | `D = J M^-1 J^T` 的源码展开 |
| D9 | `newton@1a230702 newton/_src/solvers/kamino/solver_kamino.py` | `SolverKamino.step` | chapter 08 里外层 wrapper 怎样接住这条主线 |
| D10 | `newton@1a230702 newton/_src/solvers/kamino/_src/solver_kamino_impl.py` | `_solve_forward_dynamics`, `_forward` | Jacobians / Delassus / reaction 真正如何进入 solve |
| D11 | `newton@1a230702 newton/_src/solvers/kamino/_src/geometry/unified.py` | `write_contact_unified_kamino`, `CollisionPipelineKamino.collide` | 可选的 direct-to-Kamino contact 写法 |

## Exact Handoff Trace

### 1. Runtime handoff 还停留在 Newton `Contacts`

- 运行时缓冲定义：`newton/_src/sim/contacts.py:118-151`
- chapter 06 的 writer 来源：`newton/_src/sim/collide.py:62-145`
- 推荐精看：
  - `rigid_contact_point0 / point1`
  - `rigid_contact_offset0 / offset1`
  - `rigid_contact_normal`
  - `rigid_contact_margin0 / margin1`

这一层要校准的边界是：`Contacts` 里已经有 solver 继续需要的几何，但还没有 `gapfunc`、contact frame、local reaction 这些 solver-facing 对象。

### 2. `Contacts -> ContactsKamino`

- container 字段：`newton/_src/solvers/kamino/_src/geometry/contacts.py:198-333`
- frame 构造：`newton/_src/solvers/kamino/_src/geometry/contacts.py:363-371`
- bridge kernel：`newton/_src/solvers/kamino/_src/geometry/contacts.py:760-879`

精确 handoff：

- `newton_point0 / newton_point1` 先经 `body_q` 变回 `p0_world / p1_world`
- `d_newton = dot(p1_world - p0_world, n_newton) - (thickness0 + thickness1)`
- `gapfunc = (normal.xyz, signed_distance)`
- `frame = quat(make_contact_frame_znorm(normal))`
- `material = (mu, restitution)`
- body-static 的一侧会被重排成 Kamino A 侧，保证 `bid_B >= 0` 以及 normal 仍然指向 `A -> B`

如果你想核对 chapter 07 主 walkthrough 里“solver-facing contact 是 position/gapfunc/frame/material”这句话，这一段就是最直接证据。

### 3. Constraint info 把每条 contact 变成三条 rows

- max-count 初始化：`newton/_src/solvers/kamino/_src/kinematics/constraints.py:184-253`
- active-count 更新：`newton/_src/solvers/kamino/_src/kinematics/constraints.py:262-299`

关键阅读点：

- `world_maxncc = [3 * maxnc for maxnc in world_maxnc]`
- `ncc = 3 * nc`
- `data.info.contact_cts_group_offset`

这一步告诉你：Kamino 的 contact rows 不是隐式存在的，它在 constraints info 里被正式计数、正式分配、正式写入 group offset。

### 4. Jacobian block 是 `frame + lever arm` 的直接实现

- lever arm helper：`newton/_src/solvers/kamino/_src/core/math.py:761-793`
- dense Jacobian build：`newton/_src/solvers/kamino/_src/kinematics/jacobians.py:760-810`
- sparse Jacobian build：`newton/_src/solvers/kamino/_src/kinematics/jacobians.py:813-904`

精确 handoff：

- `R_k = quat_to_matrix(q_k)` 把 contact frame 变成三列方向
- `cio_k = 3 * cid_k` 给这条 contact 分配三行 block 起点
- `W_B_k @ R_k` 生成 follower body 的 Jacobian block
- `-W_A_k @ R_k` 生成 base body 的 Jacobian block
- 如果 `bid_A_k == -1`，世界体一侧就没有 A block

这里最值得核对的是：Jacobians 里根本没有“凭空写一个 3x6 模板”，而是每一条 contact 都带着自己的 frame 和自己的 lever arm。

### 5. Delassus 是 row-row 在 inverse mass / inertia 下的再投影

- dense path：`newton/_src/solvers/kamino/_src/dynamics/delassus.py:106-193`
- sparse path：`newton/_src/solvers/kamino/_src/dynamics/delassus.py:196-286`

精确 handoff：

- 从 `jacobians_cts_data` 里逐 body 读 `Jv_i / Jw_i / Jv_j / Jw_j`
- 线性项：`inv_m_k * dot(Jv_i, Jv_j)`
- 角向项：`dot(Jw_i, inv_I_k @ Jw_j)`
- 最后写入 `delassus_D[dmio + ncts * i + j]`

如果你想核对“Delassus 的人话就是 rows 在系统惯性下的耦合强弱”，这一段最值得反复看。

### 6. 这条 contact math 主线怎样继续进入求解器

- outer wrapper：`newton/_src/solvers/kamino/solver_kamino.py:530-579`
- solver init：`newton/_src/solvers/kamino/_src/solver_kamino_impl.py:194-250`
- solve 主段：`newton/_src/solvers/kamino/_src/solver_kamino_impl.py:1000-1119`

关键 handoff：

- `SolverKamino.step()` 如果拿到 Newton `Contacts`，先 `convert_contacts_newton_to_kamino(...)`
- `SolverKaminoImpl.__init__()` 里已经把 `make_unilateral_constraints_info(...)`、`Dense/SparseSystemJacobians`、`DualProblem`、`PADMMSolver` 全搭好了
- `_solve_forward_dynamics()` 再按 `detector/limits -> constraint info -> Jacobians -> actuation -> _forward()` 这条顺序推进
- `_forward()` 里继续走 `_update_dynamics()`、`_update_constraints()`、`_update_wrenches()`

也就是说，chapter 07 不是“只在文档里讲一下 `J` 和 `D`”；这条线在 chapter 08 的 Kamino solver 里会直接继续出现。

## Optional Branches

### Branch A: unified collision pipeline 直接写 `ContactsKamino`

- writer：`newton/_src/solvers/kamino/_src/geometry/unified.py:130-247`
- pipeline 接线：`newton/_src/solvers/kamino/_src/geometry/unified.py:578-588`, `611-649`

这条分支跳过了 `Contacts -> ContactsKamino` 的二次桥接，直接把 `ContactData` 写成 `gapfunc + frame + material`。如果你想确认 solver-facing contact 的最小字段到底是什么，这条分支很有价值。

### Branch B: sparse Jacobian / sparse Delassus

- sparse Jacobian：`newton/_src/solvers/kamino/_src/kinematics/jacobians.py:813-904`
- sparse Delassus：`newton/_src/solvers/kamino/_src/dynamics/delassus.py:196-286`

first pass 可以完全跳过。只要先守住 dense 版，你已经足够理解 chapter 07 的主线。

### Branch C: limits 和 contacts 共享同一套 constraints info

- 初始化：`newton/_src/solvers/kamino/_src/kinematics/constraints.py:116-253`
- active update：`newton/_src/solvers/kamino/_src/kinematics/constraints.py:284-299`

这能帮助你理解为什么 chapter 07 既讲 contacts，也顺手提到 unilateral constraints。Kamino 里 limit rows 和 contact rows 的容器管理是并列的，只是 contact 被扩成 `3 * nc`。

## Verification Anchors

| 想验证的 claim | 直接打开哪里 |
|----------------|--------------|
| chapter 06 留下的是几何缓冲，不是 solver 反力 | `newton/_src/sim/contacts.py:118-151`, `newton/_src/sim/collide.py:62-145` |
| Kamino 会先把 Newton `Contacts` 重排成 `position / gapfunc / frame / material` | `newton/_src/solvers/kamino/_src/geometry/contacts.py:269-299`, `760-879` |
| 每条 contact 被正式扩成三条 rows | `newton/_src/solvers/kamino/_src/kinematics/constraints.py:189-191`, `284-287` |
| Jacobian 是 contact frame 加 lever arm 的产物 | `newton/_src/solvers/kamino/_src/core/math.py:761-793`, `newton/_src/solvers/kamino/_src/kinematics/jacobians.py:785-810` |
| Delassus 确实是在做 `J M^-1 J^T` 的逐 body 累加 | `newton/_src/solvers/kamino/_src/dynamics/delassus.py:169-193` |
| chapter 08 的 Kamino solver 会直接接住这条主线 | `newton/_src/solvers/kamino/solver_kamino.py:542-566`, `newton/_src/solvers/kamino/_src/solver_kamino_impl.py:1096-1119` |
