---
chapter: 07
title: 约束与接触数学总论 源码走读
last_updated: 2026-04-19
source_paths:
  - docs/concepts/collisions.rst
  - newton/_src/sim/contacts.py
  - newton/_src/solvers/kamino/_src/geometry/contacts.py
  - newton/_src/solvers/kamino/_src/kinematics/constraints.py
  - newton/_src/solvers/kamino/_src/kinematics/jacobians.py
  - newton/_src/solvers/kamino/_src/core/math.py
  - newton/_src/solvers/kamino/_src/dynamics/delassus.py
  - newton/_src/solvers/kamino/solver_kamino.py
paper_keys: []
newton_commit: 1a230702
---

# 07 约束与接触数学总论 源码走读

如果你是第一次读这一章，最好先配合 `principle.md` 一起读；如果你是直接跳进源码走读也没关系，但一旦发现术语开始变快，就先回到 `principle.md` 把对象关系补齐，再回来追源码。

这份主 walkthrough 是给第一次追 chapter 07 源码的人准备的。目标不是把 Kamino 的每个 solver 细节都铺开，而是让你不离开这页，也能把 chapter 06 交出来的 `Contacts` 继续读成一条连续 handoff：`Contacts -> solver-facing contact -> rows -> Jacobians -> Delassus`。

## What This Walkthrough Follows

只追这一条主线：

```text
chapter 06 的 Contacts
-> convert_contacts_newton_to_kamino(...)
-> ContactsKamino.position_A / position_B / gapfunc / frame
-> 1 contact -> 3 rows
-> contact Jacobian blocks
-> Delassus matrix D
```

这一页刻意不展开三类东西：

- `06_collision` 之前的 narrow phase 细节；这里默认 `Contacts` 已经准备好了。
- PADMM、NCP、warm start 这类真正的求解算法；那是 `08_rigid_solvers` 继续接手的内容。
- limit constraints、sparse Jacobian 变体、unified collision pipeline 这类深读分支；它们都放到 `source-walkthrough-deep.md`。

第一遍先守住一句话：chapter 07 真正讲的是 **接触几何怎样被翻译成 contact-space 数学对象**，不是一上来就背大公式。

## One-Screen Chapter Map

```text
Contacts.rigid_contact_point0 / point1 / normal / margin
                        |
                        v
    convert_contacts_newton_to_kamino(...)
                        |
                        v
 ContactsKamino.position_A / position_B / gapfunc / frame / material
                        |
                        v
    make_unilateral_constraints_info(...)
    update_constraints_info(...)
                        |
                        v
            1 contact -> 3 rows
                        |
                        v
          build_contact_jacobians(...)
                        |
                        v
                Delassus = J M^-1 J^T
```

## Beginner Path

1. 先看 Stage 1。
   - 想验证什么：chapter 06 到底交给 chapter 07 哪些字段。
   - 看完后应该能说：`Contacts` 里保存的是接触几何，不是已经求好的接触反力。
2. 再看 Stage 2。
   - 想验证什么：为什么 Kamino 不直接吃 `rigid_contact_point0/1` 这些数组。
   - 看完后应该能说：solver 会先把 runtime `Contacts` 重排成更 solver-friendly 的 `ContactsKamino`。
3. 再看 Stage 3 和 Stage 4。
   - 想验证什么：一条接触为什么会长成三条 rows，以及 Jacobian 怎样把接触点和接触方向写成速度映射。
   - 看完后应该能说：Jacobian 不是抽象矩阵名词，而是接触 frame 加 lever arm 的代码化表达。
4. 最后看 Stage 5。
   - 想验证什么：`J` 为什么会继续长成 Delassus。
   - 看完后应该能说：Delassus 的人话含义就是“沿这些接触方向，系统有多难被推着动”。

## Main Walkthrough

### Stage 1: `Contacts` 先把接触几何稳定保存下来

**Claim**

chapter 06 交给 chapter 07 的不是 contact force，而是一份 solver 之后还会继续翻译的接触几何记录：shape id、body-frame 接触点、world-frame 法线、接触厚度。

**Why it matters**

这一步最能纠正新手的第一层误会：`Contacts` 是碰撞章节的终点，但还不是 solver 内部最终格式。chapter 07 的所有数学对象，都是从这块几何缓冲里再长出来的。

**Source excerpt**

先看 `newton/_src/sim/contacts.py` 里 chapter 06 留下来的核心字段：

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
self.rigid_contact_force = wp.zeros(rigid_contact_max, dtype=wp.vec3)
```

**Verification cues**

- `rigid_contact_point0 / point1` 是 body-frame 点，不是 world-frame 点；这意味着 solver 如果想做 contact frame 数学，往往还要先把点变回 world-space。
- `rigid_contact_normal` 仍然保留 A-to-B 的 world-frame 法线；这就是后面 contact frame 的法向起点。
- `rigid_contact_force` 只是预留输出槽，不是 collision pipeline 已经填好的答案。

**Output passed to next stage**

一份统一的 runtime 接触记录：它已经够 chapter 06 封口，但还需要被桥接成 solver-facing contact 表示。

### Stage 2: Kamino 先把 runtime `Contacts` 重排成 solver-facing contact block

**Claim**

Kamino 不直接拿 `rigid_contact_point0 / point1 / normal` 做 rows，而是先把它们变成 `position_A / position_B / gapfunc / frame / material` 这套 solver-facing 语言。

**Why it matters**

这一跳就是 chapter 07 最重要的工程事实：接触数学并不是直接长在 chapter 06 的数组名字上，而是先经过一次“为 solver 重排”的桥接。

**Source excerpt**

先看 `newton/_src/solvers/kamino/_src/geometry/contacts.py` 里 `ContactsKaminoData` 真正关心什么：

```python
position_A: wp.array | None = None
position_B: wp.array | None = None
gapfunc: wp.array | None = None
frame: wp.array | None = None
material: wp.array | None = None
```

再看 `convert_contacts_newton_to_kamino(...)` 怎样把 Newton 的 runtime contact 变成这套格式：

```python
p0_world = wp.transform_point(X0, newton_point0[tid])
p1_world = wp.transform_point(X1, newton_point1[tid])

d_newton = wp.dot(p1_world - p0_world, n_newton) - (
    newton_thickness0[tid] + newton_thickness1[tid]
)

distance = d_newton
gapfunc = vec4f(normal[0], normal[1], normal[2], float32(distance))
q_frame = wp.quat_from_matrix(make_contact_frame_znorm(normal))

kamino_position_A[mcid] = pos_A
kamino_position_B[mcid] = pos_B
kamino_gapfunc[mcid] = gapfunc
kamino_frame[mcid] = q_frame
kamino_material[mcid] = vec2f(mu, rest)
```

**Verification cues**

- solver-facing contact 重新把 body-frame 点变回了 world-space `position_A / position_B`；这是因为后面 Jacobian 和 lever arm 都要站在世界系里做。
- `gapfunc.xyz` 保存法线，`gapfunc.w` 保存 signed distance；也就是说 solver 直接看到的是“方向 + 当前间隙/穿透量”。
- `make_contact_frame_znorm(...)` 说明 contact frame 不是外加概念，而是桥接时就被正式构造出来的对象。

**Output passed to next stage**

每条 active contact 现在都长成了一个 solver-facing block：world-space 作用点、gap function、局部 contact frame、材料参数。

### Stage 3: 一条 contact 在 solver 里会变成三条 rows

**Claim**

Kamino 把一条 3D 接触看成一个三维局部 block：一条法向 row，加两条切向 row，所以 active contact 数量会被扩成 `3 * nc`。

**Why it matters**

只有把这里读成“三维局部 block”，你后面看到 `cio_k = 3 * cid_k`、contact reaction 的 `vec3f`、以及 Delassus 的 contact-contact coupling 才不会觉得它们是突然出现的魔法数字。

**Source excerpt**

局部三方向先由 `make_contact_frame_znorm(...)` 决定：

```python
@wp.func
def make_contact_frame_znorm(n: vec3f) -> mat33f:
    n = wp.normalize(n)
    if wp.abs(wp.dot(n, UNIT_X)) < COS_PI_6:
        e = UNIT_X
    else:
        e = UNIT_Y
    o = wp.normalize(wp.cross(n, e))
    t = wp.normalize(wp.cross(o, n))
    return mat33f(t.x, o.x, n.x, t.y, o.y, n.y, t.z, o.z, n.z)
```

约束计数再明确把每条 contact 展开成三条 rows：

```python
world_maxncc: list[int] = [3 * maxnc for maxnc in world_maxnc]

nlc = nl
ncc = 3 * nc
ncts = njc + nlc + ncc
```

**Verification cues**

- `frame` 里本来就有三条局部轴，所以 contact row 不是“多出来的约定”，而是 contact frame 的直接展开。
- `3 * nc` 出现在最大容量和当前活跃计数两处，说明这不是临时 kernel 习惯，而是 solver 的正式 contract。
- joint limits 仍然只算一条 row，所以 `ncc = 3 * nc` 更能突出 contact block 的特殊性。

**Output passed to next stage**

每条 contact 现在都有了一个三行的局部约束 block，下一步要把它写成“哪些 body velocity 会影响这三行”的 Jacobian。

### Stage 4: Jacobian 把 contact frame 和 lever arm 写成速度映射

**Claim**

contact Jacobian 的核心不是抽象地生成一个矩阵，而是把 `contact frame + contact point 相对质心的位置` 写成对 body twist 的映射。

**Why it matters**

这是 chapter 07 最容易从概念掉进代码的地方。读懂这一步之后，你就能把“偏心接触为什么天然带角向项”直接对到真实实现。

**Source excerpt**

先看 `contact_wrench_matrix_from_points(...)` 怎样把接触点和质心之间的 lever arm 写进 screw/wrench 变换：

```python
def contact_wrench_matrix_from_points(r_k: vec3f, r_i: vec3f) -> mat63f:
    W_ki = W_C_I
    S_ki = wp.skew(r_k - r_i)
    for i in range(3):
        for j in range(3):
            W_ki[3 + i, j] = S_ki[i, j]
    return W_ki
```

再看 `newton/_src/solvers/kamino/_src/kinematics/jacobians.py` 里 contact Jacobian block 的真正组装：

```python
R_k = wp.quat_to_matrix(q_k)
cio_k = 3 * cid_k

W_B_k = contact_wrench_matrix_from_points(r_Bc_k, r_B_k)
JT_c_B_k = W_B_k @ R_k

if bid_A_k > -1:
    W_A_k = contact_wrench_matrix_from_points(r_Ac_k, r_A_k)
    JT_c_A_k = -W_A_k @ R_k
```

**Verification cues**

- `R_k` 就是上一 stage 的 local contact frame，所以三条 rows 的方向是由法向和两条切向直接决定的。
- `W_A_k / W_B_k` 把接触点到质心的偏移写进了下半块；这就是角向 Jacobian 的来源。
- `cio_k = 3 * cid_k` 说明 Jacobian 也是按 contact block 在排，不是按单一标量接触在排。

**Output passed to next stage**

`J` 现在已经明确了：它告诉 solver 哪些 body 的平移/转动速度会影响每一条 contact row。

### Stage 5: Delassus 把 Jacobian 继续压成 contact-space 的“难推动程度”

**Claim**

Delassus kernel 在代码里就是把 Jacobian row 通过 body 的逆质量和逆惯量再投一次，于是得到 `D = J M^{-1} J^T` 这层 contact-space 量。

**Why it matters**

这一步把 chapter 07 的数学主线真正封口。前面你一直在问“哪些运动会影响接触方向”，现在则变成“沿这些方向施加反力时，系统会如何响应”。

**Source excerpt**

`newton/_src/solvers/kamino/_src/dynamics/delassus.py` 的 dense kernel 基本把这层数学翻成了逐 body 累加：

```python
for k in range(nb):
    for d in range(3):
        Jv_i[d] = jacobians_cts_data[jio_ik + d]
        Jw_i[d] = jacobians_cts_data[jio_ik + d + 3]
        Jv_j[d] = jacobians_cts_data[jio_jk + d]
        Jw_j[d] = jacobians_cts_data[jio_jk + d + 3]

    inv_m_k = model_bodies_inv_m_i[bid_k]
    lin_ij = inv_m_k * wp.dot(Jv_i, Jv_j)

    inv_I_k = data_bodies_inv_I_i[bid_k]
    ang_ij = wp.dot(Jw_i, inv_I_k @ Jw_j)

    D_ij += lin_ij + ang_ij

delassus_D[dmio + ncts * i + j] = 0.5 * (D_ij + D_ji)
```

**Verification cues**

- 线性项和角向项被显式分开累加，所以 Delassus 的物理意义并不神秘：平移惯性和转动惯性都在里面。
- 如果两条 rows 作用在同一刚体上，它们就会通过这次逐 body 累加彼此耦合起来。
- chapter 08 里 Kamino solver 真正做 constrained dynamics 时，就是继续在这层 contact-space 量上工作。

**Output passed to next stage**

一份 solver 可以继续消费的 contact-space 表示：`Contacts -> solver-facing contact -> rows -> J -> D`。这正是 `08_rigid_solvers` 里 Kamino continuation 要接住的东西。

## Object Ledger

| 对象 | 谁生产 | 谁消费 | 盯哪些字段 |
|------|--------|--------|------------|
| `Contacts` | chapter 06 的 `write_contact()` | `convert_contacts_newton_to_kamino(...)` | `shape0/1`、`point0/1`、`normal`、`margin0/1` |
| `ContactsKamino` | `convert_contacts_newton_to_kamino(...)` | constraint info、Jacobians、Kamino solver | `position_A/B`、`gapfunc`、`frame`、`material` |
| contact row block | `make_unilateral_constraints_info(...)`、`update_constraints_info(...)` | Jacobian builder、Delassus builder | `3 * nc`、group offsets、`cio_k` |
| contact Jacobian | `build_contact_jacobians(...)` | Delassus builder、Kamino dynamics solve | row 方向、body A/B block、lever arm |
| Delassus `D` | `build_delassus_*` | chapter 08 的 constrained dynamics solve | row-row coupling、线性项、角向项 |

## Stop Here

读到这里就已经够 chapter 07 的 80-90% 了。

如果你现在能用自己的话讲顺下面这句话，这一章的 beginner 目标就完成了：

```text
chapter 06 的 Contacts 先保住接触几何；
Kamino 再把它桥接成 position / gapfunc / frame 这种 solver-facing contact；
每条 contact 接着长成 3 条局部 rows；
Jacobian 写出哪些速度会影响这些 rows；
Delassus 再写出沿这些 rows 系统有多难被推着动。
```

这时你已经不需要先背完整求解器理论，也能解释 chapter 07 的主干为什么成立。

## Go Deeper

如果你还想继续精确追源码，再去 `source-walkthrough-deep.md`：

- 想保留 cross-repo 精确锚点：看 `Fast Deep Index`
- 想逐跳追 `Contacts -> ContactsKamino -> rows -> J -> D`：看 `Exact Handoff Trace`
- 想分清 unified pipeline、sparse Jacobian 等分支：看 `Optional Branches`
- 想逐条核对这里的 claim：看 `Verification Anchors`
