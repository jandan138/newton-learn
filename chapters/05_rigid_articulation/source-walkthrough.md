---
chapter: 05
title: 刚体与关节动力学
last_updated: 2026-04-20
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

如果你刚从 `04_scene_usd` 走到这里，最自然的问题通常不是 `joint_q_start` 是什么，而是：最小的一条 articulation 到底在表达什么？

先想一个最小画面：一个 parent body，通过一个 joint，连着一个 child body。系统至少要同时记住三件事：谁连谁，这个 joint 自己的角度/速度放在哪，以及这些量最后怎样把 child 的世界位姿和速度算出来。

这份主 walkthrough 只追 chapter 05 的关键桥：articulation 结构怎样被组织成 flat layout，`joint_q / joint_qd` 怎样经由 FK 和速度传播变成 `body_q / body_qd`，以及这些状态又怎样继续被 Featherstone 路线消费。目标不是现在就推完整 ABA / CRBA，而是先把“结构和状态怎样接力”讲顺。

如果你想再配一个更慢一点的概念版，可以再看 `principle.md`；但只读这一份，也应该能把 chapter 05 的主线讲顺。

第一次先把下面几个词翻成人话：

- `articulation`：一条由 body 和 joint 连起来、需要一起前向传播的树。
- `generalized coordinates`：每个 joint 自己的参数化坐标，也就是 `joint_q / joint_qd` 这层状态；它们不是 world pose。
- `joint_q_start`：某个 joint 在扁平 `joint_q` 数组里从哪开始切自己的那一段。
- `articulation_start`：某条 articulation 在扁平 joint 数组里从哪一个 joint 开始。
- FK：forward kinematics。第一遍先把它读成“用 joint 坐标和局部 frame 算 body world pose / velocity”。

这里顺手补一句最容易漏掉的前情：不同 joint 占的坐标长度本来就可能不同。revolute 常常只占 1 个角度，自由关节会占更多位置和速度自由度，所以 Newton 才必须用 `joint_q_start / joint_qd_start` 明确记“这一段从哪开始切”。

## What This Walkthrough Follows

这一页只追下面这条 handoff：

```text
minimal joint tree
-> builder articulation layout
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
最小画面：parent body --joint--> child body
                |
                v
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
   - 想验证什么：最小 articulation 需要先记住哪些结构信息，为什么 Newton 不把它存成嵌套对象树。
   - 看完后应该能说：`joint_q_start / joint_qd_start / articulation_start` 这组切片约定，才是 flat articulation layout 的骨架。
2. 再看 Stage 2。
   - 想验证什么：`generalized coordinates` 和 body/world state 为什么要分两层保存。
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

Newton 的 articulation 结构首先表现为一组扁平数组和切片起点：joint 的 parent / child、两侧 frame、每个 joint 在 `joint_q / joint_qd` 里占哪一段，以及每条 articulation 从哪一个 joint 开始。也就是说，你脑中看到的“最小两连杆”会先被压成一条可切片的 joint range。

**Why it matters**

如果第一遍还把 articulation 想成“Python 里一层层套起来的 joint 对象树”，后面看到 `joint_q_start`、`articulation_start` 就会很不自然。这里先把“树的物理图像”和“flat layout 的工程写法”对上，后面就不会只剩数组名。

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

- `joint_q_start / joint_qd_start` 明确告诉你每个 joint 在 generalized coordinates 里占哪一段，也就是“自己的角度/速度参数从大数组哪里开始切”。
- `articulation_start` 则告诉你每条 articulation 的 joint 范围，也就是“这条树从 joint 列表的哪开始”。
- 这套布局天然适合 kernel 按 articulation 或按 joint 范围并行，而不是按对象树递归。

**Output passed to next stage**

一套 solver-friendly 的结构布局：joint ranges、articulation ranges，以及 parent-child 拓扑和局部 frame。

### Stage 2: `Model` 和 `State` 同时保留 joint-space 与 body-space 两层状态

**Claim**

在 chapter 05 里，Newton 明确同时持有两层状态：`joint_q / joint_qd` 是 generalized coordinates，`body_q / body_qd` 是 body-space world state。它们会在同一个 `State` 里并存，但并不属于同一层语义。第一遍可以把这件事读成：一层在记“joint 自己现在走到哪”，另一层在记“body 现在在世界里怎样摆、怎样动”。

**Why it matters**

这是理解 articulation handoff 的前提。后面你之所以需要 FK，就是因为这两层状态不能直接互相替代；`joint_qd` 也不能跳过中间桥，直接拿来当 `body_qd`。

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

`eval_single_articulation_fk()` 的核心任务不是求力，而是先把每个 joint 在 `joint_q / joint_qd` 中的那一段拿出来，构造 `X_j / v_j`，再沿 parent-child 链传播成 child body 的 `body_q / body_qd`。第一遍直接把 FK 读成“joint 坐标怎样长成 body world state”就够了。

**Why it matters**

这一步正是 chapter 05 的主 handoff。读懂这里，你就知道为什么 `joint_qd` 不能直接拿来当 `body_qd`，也知道 `joint_X_p / joint_X_c` 为什么不是静态摆设。

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

Featherstone 路线不会直接拿着原始 `body_mass / body_com / body_inertia` 就开算；它先把这些量变成 spatial inertia 和 COM transforms，再把 articulation layout 继续展开成 solver 内部的 `joint_S_s / body_v_s / body_f_s` 等 buffers。第一遍遇到 `joint_S_s` 时，只要先把它读成“joint 允许运动的方向基”就够了。

**Why it matters**

这一步解释了为什么 chapter 03 的 inertia 词汇会在这里重新出现，也解释了 chapter 05 为什么要把 FK 和 mass property 放在同一章：solver 继续消费的就是同一条 articulation handoff。

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

- `body_I_m` 和 `body_X_com` 说明 solver 先把 body-level mass properties 改写成更方便递推的格式。
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
先想最小两连杆和一个 joint；
builder 再把这条树记成 joint 切片和局部 frame；
State 同时保留 joint-space 和 body-space 两层状态；
FK 把 joint_q / joint_qd 译成 body_q / body_qd；
Featherstone 只是继续沿同一条 handoff 消费这些结果。
```

这时你已经可以稳定进入 `06_collision`、`07_constraints_contacts_math` 和 `08_rigid_solvers`。

## Go Deeper

如果你还想继续精确追源码，再去 `source-walkthrough-deep.md`：

- 想保留所有 file/symbol/line anchors：看 `Fast Deep Index`
- 想逐跳追 articulation layout 到 Featherstone step 的 exact handoff：看 `Exact Handoff Trace`
- 想知道哪些 internal conversion / matrix branches 第一遍可以跳过：看 `Optional Branches`
- 想逐条核对这里的 claim：看 `Verification Anchors`
