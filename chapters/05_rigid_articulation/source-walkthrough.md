---
chapter: 05
title: 刚体与关节动力学
last_updated: 2026-04-19
source_paths:
  - docs/concepts/articulations.rst
  - newton/_src/sim/builder.py
  - newton/_src/sim/model.py
  - newton/_src/sim/state.py
  - newton/_src/sim/articulation.py
  - newton/_src/solvers/featherstone/kernels.py
  - newton/_src/solvers/featherstone/solver_featherstone.py
paper_keys:
  - featherstone-book
  - aba-crba-notes
newton_commit: 1a230702
---

# 05 刚体与关节动力学 源码走读

如果你是第一次读这一章，最好先配合 `principle.md` 一起读；如果你是直接跳进源码走读也没关系，但一旦发现术语开始变快，就先回到 `principle.md` 把对象关系补齐，再回来追源码。

这份主 walkthrough 只追 chapter 05 的关键桥：articulation 结构怎样被组织成 flat slices，`joint_q / joint_qd` 怎样经由 FK 和速度传播变成 `body_q / body_qd`，以及这些状态又怎样继续被 Featherstone 路线消费。目标不是现在就推完整 ABA / CRBA，而是先把“结构和状态怎样接力”讲顺。

## What This Walkthrough Follows

这一页只追下面这条 handoff：

```text
builder articulation layout
-> Model / State joint and body buffers
-> eval_fk(...)
-> body_q / body_qd
-> Featherstone spatial buffers
-> solver.step(...) continues the chain
```

这一页刻意不展开三类东西：

- 完整 Featherstone 数学推导
- Jacobian / Delassus / contact math
- rigid solver family 的横向比较

第一遍先守住一句话：chapter 05 真正讲的是 **articulation 的结构布局，以及 joint-space 状态怎样被译成 body-space 状态**。

## One-Screen Chapter Map

```text
builder.add_joint / add_articulation
                |
                v
   joint_q_start / joint_qd_start / articulation_start
                |
                v
      Model.state() gives joint_q / joint_qd / body_q / body_qd
                |
                v
        eval_single_articulation_fk(...)
                |
                v
          body_q / body_qd in world / COM convention
                |
                v
  Featherstone: body_I_m / body_X_com / joint_S_s / body_v_s
                |
                v
         solver.step(...) integrates generalized joints
```

## Beginner Path

1. 先看 Stage 1。
   - 想验证什么：Newton 里的 articulation 为什么不是嵌套对象树，而是一组切片约定。
   - 看完后应该能说：`joint_q_start / joint_qd_start / articulation_start` 才是关节布局真正的骨架。
2. 再看 Stage 2。
   - 想验证什么：`Model` 和 `State` 为什么同时保留 joint-space 和 body-space 两层状态。
   - 看完后应该能说：`joint_q / joint_qd` 不是 `body_q / body_qd` 的别名，它们是不同层级的数据。
3. 再看 Stage 3。
   - 想验证什么：FK 和速度传播到底怎样把 joint-space 变成 body-space。
   - 看完后应该能说：`joint_q / joint_qd` 要先过 `X_j / v_j` 和 transform chain，才会变成 `body_q / body_qd`。
4. 再看 Stage 4。
   - 想验证什么：为什么 mass / inertia 会再次出现。
   - 看完后应该能说：Featherstone 先把它们变成 spatial form，继续喂给 solver 内部 buffers。
5. 最后看 Stage 5。
   - 想验证什么：Featherstone solver 到底有没有重新发明一套状态表示。
   - 看完后应该能说：没有，它仍然围绕同一套 articulation 布局和 state handoff 在工作。

## Main Walkthrough

### Stage 1: articulation 先被记成 flat slice layout，而不是对象树

**Claim**

Newton 的 articulation 结构首先表现为一组扁平数组和切片起点：joint 的 parent / child、两侧 frame、每个 joint 在 `joint_q / joint_qd` 里占哪一段，以及每条 articulation 从哪一个 joint 开始。

**Why it matters**

如果第一遍还把 articulation 想成“Python 里一层层套起来的 joint 对象树”，后面看到 `joint_q_start`、`articulation_start` 就会很不自然。

**Source excerpt**

builder 在加 joint 时就已经把这些切片信息记下来了：

```python
self.joint_parent.append(parent)
self.joint_child.append(child)
self.joint_X_p.append(parent_xform)
self.joint_X_c.append(child_xform)
...
self.joint_q_start.append(self.joint_coord_count)
self.joint_qd_start.append(self.joint_dof_count)
```

真正声明“这一串 joint 组成一条 articulation”的地方也很直接：

```python
self.articulation_start.append(sorted_joints[0])
self.articulation_label.append(label or f"articulation_{articulation_idx}")
...
for joint_idx in joints:
    self.joint_articulation[joint_idx] = articulation_idx
```

`finalize()` 再把这些起点补上 sentinel：

```python
joint_q_start = copy.copy(self.joint_q_start)
joint_q_start.append(self.joint_coord_count)
joint_qd_start = copy.copy(self.joint_qd_start)
joint_qd_start.append(self.joint_dof_count)
articulation_start = copy.copy(self.articulation_start)
articulation_start.append(self.joint_count)
```

**Verification cues**

- `joint_q_start / joint_qd_start` 明确告诉你每个 joint 在 generalized coordinates 里占哪一段。
- `articulation_start` 则告诉你每条 articulation 的 joint 范围。
- 这套布局天然适合 kernel 按 articulation 或按 joint 范围并行，而不是按对象树递归。

**Output passed to next stage**

一套 solver-friendly 的结构布局：joint ranges、articulation ranges，以及 parent-child 拓扑和局部 frame。

### Stage 2: `Model` 和 `State` 同时保留 joint-space 与 body-space 两层状态

**Claim**

在 chapter 05 里，Newton 明确同时持有两层状态：`joint_q / joint_qd` 是 generalized coordinates，`body_q / body_qd` 是 body-space world state。它们会在同一个 `State` 里并存，但并不属于同一层语义。

**Why it matters**

这是理解 articulation handoff 的前提。后面你之所以需要 FK，就是因为这两层状态不能直接互相替代。

**Source excerpt**

`Model` 先声明 joint 层和 articulation 层字段：

```python
self.joint_q: wp.array[wp.float32] | None = None
self.joint_qd: wp.array[wp.float32] | None = None
...
self.joint_q_start: wp.array[wp.int32] | None = None
self.joint_qd_start: wp.array[wp.int32] | None = None
self.articulation_start: wp.array[wp.int32] | None = None
```

`State` 则明确同时放 body 和 joint 两层快照：

```python
self.body_q: wp.array | None = None
self.body_qd: wp.array | None = None
...
self.joint_q: wp.array | None = None
self.joint_qd: wp.array | None = None
```

而 `Model.state()` 会一次性把这两层都 clone 出来：

```python
s.body_q = wp.clone(self.body_q, requires_grad=requires_grad)
s.body_qd = wp.clone(self.body_qd, requires_grad=requires_grad)
...
s.joint_q = wp.clone(self.joint_q, requires_grad=requires_grad)
s.joint_qd = wp.clone(self.joint_qd, requires_grad=requires_grad)
```

**Verification cues**

- joint-space 和 body-space 字段在 `State` 里同时存在，不是互斥的两种模式。
- `Model.state()` 不会帮你自动“算完 FK”，它只是把两层默认值都分配出来。
- 所以后面必须有一条显式 bridge 把 joint-space 状态解释成 body-space 状态。

**Output passed to next stage**

一份同时含有 `joint_q / joint_qd` 和 `body_q / body_qd` 的 `State`，等待 FK 和速度传播来把两层状态真正对齐。

### Stage 3: FK 和速度传播把 `joint_q / joint_qd` 译成 `body_q / body_qd`

**Claim**

`eval_single_articulation_fk()` 的核心任务不是求力，而是先把每个 joint 在 `joint_q / joint_qd` 中的那一段拿出来，构造 `X_j / v_j`，再沿 parent-child 链传播成 child body 的 `body_q / body_qd`。

**Why it matters**

这一步正是 chapter 05 的主 handoff。读懂这里，你就知道为什么 `joint_qd` 不能直接拿来当 `body_qd`。

**Source excerpt**

FK 先按 `q_start / qd_start` 取出某个 joint 的 generalized coordinates：

```python
q_start = joint_q_start[i]
qd_start = joint_qd_start[i]
...
if type == JointType.REVOLUTE:
    axis = joint_axis[qd_start]

    q = joint_q[q_start]
    qd = joint_qd[qd_start]

    X_j = wp.transform(wp.vec3(), wp.quat_from_axis_angle(axis, q))
    v_j = wp.spatial_vector(wp.vec3(), axis * qd)
```

然后再把 joint motion 沿拓扑链组合到 child body：

```python
X_wpj = X_pj
if parent >= 0:
    X_wp = body_q[parent]
    X_wpj = X_wp * X_wpj

X_wcj = X_wpj * X_j
X_wc = X_wcj * wp.transform_inverse(X_cj)
...
body_q[child] = X_wc
body_qd[child] = origin_twist_to_com_twist(v_wc_origin, X_wc, body_com[child])
```

**Verification cues**

- 同一个 joint 的 position 和 velocity，要靠 `joint_q_start / joint_qd_start` 才能切对位置。
- `X_j` 和 `v_j` 分别表达 joint 的位姿增量和速度增量，不会直接等于 body world pose。
- `body_qd[child]` 最后还经过了 `origin_twist_to_com_twist(...)`，说明 body velocity 还有参考点转换这一步。

**Output passed to next stage**

`body_q / body_qd` 这组 body-space world-state。后面的 collision、Jacobian、solver kernels 都更愿意消费这一层结果。

### Stage 4: Featherstone 先把质量属性和 FK 结果变成 spatial buffers

**Claim**

Featherstone 路线不会直接拿着原始 `body_mass / body_com / body_inertia` 就开算；它先把这些量变成 spatial inertia 和 COM transforms，再把 articulation layout 继续展开成 solver 内部的 `joint_S_s / body_v_s / body_f_s` 等 buffers。

**Why it matters**

这一步解释了为什么 chapter 03 的 inertia 词汇会在这里重新出现，也解释了 chapter 05 为什么要把 FK 和 mass property 放在同一章。

**Source excerpt**

Featherstone 先做两步很干净的预处理：

```python
@wp.kernel
def compute_spatial_inertia(body_inertia, body_mass, body_I_m):
    ...

@wp.kernel
def compute_com_transforms(body_com, body_X_com):
    tid = wp.tid()
    com = body_com[tid]
    body_X_com[tid] = wp.transform(com, wp.quat_identity())
```

之后再把 articulation 的 joint motion 继续变成 solver 内部的 spatial buffers：

```python
v_j_s = jcalc_motion(
    type,
    joint_axis,
    lin_axis_count,
    ang_axis_count,
    X_wpj,
    joint_qd,
    qd_start,
    joint_S_s,
)
...
I_s = transform_spatial_inertia(X_sm, I_m)
f_b_s = I_s * a_s + spatial_cross_dual(v_s, I_s * v_s)

body_v_s[child] = v_s
body_f_s[child] = f_b_s - f_g_s
body_I_s[child] = I_s
```

**Verification cues**

- `body_I_m` 和 `body_X_com` 说明 solver 先把 body-level mass properties改写成更方便递推的格式。
- `joint_S_s` 是 motion subspace，不再只是原始 joint axis 的简单抄写。
- `body_v_s / body_f_s / body_I_s` 说明 Featherstone 在用一套更内部的 spatial buffer 继续工作。

**Output passed to next stage**

一批 solver-side spatial buffers：`body_I_m`、`body_X_com`、`joint_S_s`、`body_v_s`、`body_f_s`、`body_I_s`。

### Stage 5: `SolverFeatherstone.step()` 没有重造状态，只是继续沿同一条链推进

**Claim**

`SolverFeatherstone.step()` 并没有重新发明一套外部状态表示；它还是围绕同一套 articulation layout 和 `State` handoff 工作，只是中间会额外构建 Jacobian、mass matrix、Cholesky 等 solver 内部工作区。

**Why it matters**

这一步能帮你把 chapter 05 和 chapter 08 接起来：solver family 会不同，但它们消费的 articulation 结构和公共 state handoff 是连续的。

**Source excerpt**

step 开头先用当前 `joint_q` 刷新 body poses：

```python
wp.launch(
    eval_rigid_fk,
    dim=model.articulation_count,
    inputs=[
        model.articulation_start,
        model.joint_type,
        model.joint_parent,
        model.joint_child,
        model.joint_q_start,
        model.joint_qd_start,
        state_in.joint_q,
        ...
    ],
    outputs=[state_in.body_q, state_aug.body_q_com],
)
```

真正积分完 generalized joints 之后，又会把结果重新写回公共状态：

```python
wp.launch(
    kernel=integrate_generalized_joints,
    dim=model.joint_count,
    inputs=[
        model.joint_type,
        model.joint_q_start,
        model.joint_qd_start,
        ...
        state_in.joint_q,
        state_aug.joint_qd_internal_in,
        state_aug.joint_qdd,
        dt,
    ],
    outputs=[state_out.joint_q, state_aug.joint_qd_internal_out],
)

eval_fk_with_velocity_conversion(
    model,
    state_out.joint_q,
    state_aug.joint_qd_internal_out,
    state_out,
)
```

**Verification cues**

- step 一开始还是从 `state_in.joint_q` 出发，而不是从某个 solver 私有树对象出发。
- 积分完成后仍然会把结果写回 `state_out.joint_q / joint_qd / body_q / body_qd`。
- 所以 solver 内部虽然很复杂，但 external contract 仍然沿着同一条 chapter-05 handoff 在走。

**Output passed to next stage**

更新后的 generalized state 和 body state。接下来它们就能继续流向 collision、contact math 和更具体的 solver 比较章节。

## Object Ledger

| 对象 | 谁生产 | 谁消费 | 盯哪些字段 |
|------|--------|--------|------------|
| `articulation_start` | `add_articulation()` / `finalize()` | FK、Featherstone kernels | 每条 articulation 的 joint 范围 |
| `joint_q_start / joint_qd_start` | `add_joint()` / `finalize()` | FK、solver integration | 每个 joint 在 generalized arrays 里占哪一段 |
| `State.joint_q / joint_qd` | `Model.state()` | FK、solver integration | joint-space coordinates |
| `State.body_q / body_qd` | `Model.state()`，随后由 FK 重写 | collision、solver、viewer | body-space world pose / twist |
| `body_I_m / body_X_com` | Featherstone 预处理 kernels | Featherstone step | spatial inertia 和 COM transform |
| `joint_S_s / body_v_s / body_f_s` | Featherstone kernels | Featherstone dynamics path | motion subspace、body velocity、body force |

## Stop Here

读到这里就已经够 chapter 05 的 80-90% 了。

如果你现在能用自己的话讲顺下面这句话，这一章的 beginner 目标就完成了：

```text
builder 先把 articulation 记成 joint 切片和局部 frame；
State 同时保留 joint-space 和 body-space 两层状态；
FK 再把 joint_q / joint_qd 译成 body_q / body_qd；
Featherstone 把质量属性和 FK 结果变成 spatial buffers；
solver.step() 继续沿同一条 handoff 推进一步。
```

这时你已经可以稳定进入 `06_collision`、`07_constraints_contacts_math` 和 `08_rigid_solvers`。

## Go Deeper

如果你还想继续精确追源码，再去 `source-walkthrough-deep.md`：

- 想保留所有 file/symbol/line anchors：看 `Fast Deep Index`
- 想逐跳追 articulation layout 到 Featherstone step 的 exact handoff：看 `Exact Handoff Trace`
- 想知道哪些 internal conversion / matrix branches 第一遍可以跳过：看 `Optional Branches`
- 想逐条核对这里的 claim：看 `Verification Anchors`
