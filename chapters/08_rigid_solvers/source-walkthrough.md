---
chapter: 08
title: 刚体求解器家族 源码走读
last_updated: 2026-04-21
source_paths:
  - newton/solvers.py
  - newton/_src/solvers/solver.py
  - newton/_src/solvers/semi_implicit/solver_semi_implicit.py
  - newton/_src/solvers/featherstone/solver_featherstone.py
  - newton/_src/solvers/mujoco/solver_mujoco.py
  - newton/_src/solvers/kamino/solver_kamino.py
  - newton/_src/solvers/kamino/_src/solver_kamino_impl.py
  - newton/examples/robot/example_robot_cartpole.py
  - newton/_src/solvers/kamino/examples/sim/example_sim_basics_box_on_plane.py
paper_keys:
  - mujoco-warp-paper
  - kamino-report
  - featherstone-book
newton_commit: 1a230702
---

# 08 刚体求解器家族 源码走读

你很可能会先卡在一个非常具体的问题上：我明明只改了 `self.solver = ...`，外面调用的还是同一个 `solver.step(state_in, state_out, control, contacts, dt)`，为什么源码里却像突然换了一个世界？

这份主 walkthrough 就是回答这个问题。它不会把 chapter 08 写成“四种 solver 名字目录”，而是只追一条给新手最稳的主线：**同一个 public `step(...)` 外壳不变，但不同 solver family 会把不同的内部对象当成第一主角。**

如果你想要更慢一点的概念版，可以再配合 `principle.md`；但第一遍只读这一页也应该能跟上，因为这里会尽量把第一次出现的词都落回具体对象和 handoff。

## What This Walkthrough Follows

这份主 walkthrough 只追这一条连续必要链：`同一个 public step -> 不同 family 改变主对象 -> Kamino 接住 chapter 07`。

只追这一条主线：

```text
同一个 public solver.step(...)
-> 问：这一家 solver 在里面先把谁当主对象？
-> 走 force route / joint-space route / backend bridge
-> Kamino 接着续写 chapter 07
```

第一遍先把这几个词压成最朴素的意思：

- `public contract`：调用者能依赖的统一外层约定。先把它读成“外面都还是这样调 `step(...)`”。
- `force route`：先把各种影响累计成力，再积分。
- `joint-space route`：先回到关节坐标、质量矩阵、加速度这些对象上求解。
- `backend bridge`：先把 Newton 里的对象翻译给外部 backend，再把结果翻回来。

这一页刻意不展开三类东西：

- 每个 solver 的完整理论课或调参说明。
- chapter 07 已经讲过的 contact Jacobian / Delassus 推导细节。
- MuJoCo backend、Kamino PADMM、Featherstone CRBA/ABA 的所有深分支；这些都放到 `source-walkthrough-deep.md`。

第一遍先守住一句话：chapter 08 真正讲的是 **同一个 public `step(...)` 入口，后面可以接完全不同的“先处理哪个对象”的内部路线**。

## One-Screen Chapter Map

```text
Model / State / Control / Contacts / dt
                |
                v
     同一个 SolverBase.step(state_in, state_out, control, contacts, dt)
                |
     +-----------+-------------+--------------+
     |           |             |              |
     v           v             v              v
SemiImplicit  Featherstone  MuJoCo        Kamino
force         joint-space   backend       constraint
buffers       system        state/data    system
主对象        主对象        主对象        主对象
```

## Beginner Path

1. 先看 Stage 1。
   - 想验证什么：所有 solver 共享的 public `step(...)` 外壳到底是什么。
   - 看完后应该能说：统一的是调用者看到的 contract，不是内部工作对象。
2. 再看 Stage 2 到 Stage 4。
   - 想验证什么：同一个 `step(...)` 为什么会分叉成三条完全不同的内部路线。
   - 看完后应该能说：区别不只是 solver 名字不同，而是每家先围着不同对象工作。
3. 最后看 Stage 5。
   - 想验证什么：chapter 07 的 contact math 到底在哪条 solver 路线上继续出现。
   - 看完后应该能说：Kamino 是 `Contacts -> rows -> Jacobians -> Delassus` 这条线在 `step(...)` 里的直接下一跳。
4. 如果还想把例子和主线连起来，再回看 `example_robot_cartpole.py` 和 `example_sim_basics_box_on_plane.py`。

## Main Walkthrough

### Stage 1: 所有 solver 先共享同一个 public contract

**Definition**

`public contract`：先把它读成“调用者能依赖的统一外层约定”。也就是不管内部是哪一家 solver，外面都还是把 `state / control / contacts / dt` 交给 `step(...)`。

**Claim**

从 `newton.solvers` 到 `SolverBase.step(...)`，Newton 先把所有 solver 压成同一个外层接口：`Model / State / Control / Contacts / dt -> step(...) -> State`。

**Why it matters**

这一步最能纠正新手的第一层误会：你改 `self.solver = ...` 时，用户侧看到的调用方式几乎没变，但这不意味着内部也没变。chapter 08 的比较必须先在同一个 public API 平面上开始。

**Source excerpt**

`newton/solvers.py` 一开头就把 solver 的统一工作流写出来了：

```python
"""
Solvers are used to integrate the dynamics of a Newton model.
The typical workflow is to construct a Model and a State object,
then use a solver to advance the state forward in time via step.
"""
```

`SolverBase` 再把这个外层入口固定成统一签名：

```python
def step(
    self,
    state_in: State,
    state_out: State,
    control: Control | None,
    contacts: Contacts | None,
    dt: float,
) -> None:
    raise NotImplementedError()
```

而 `example_robot_cartpole.py` 正好给了一个最省力的对照入口：

```python
self.solver = newton.solvers.SolverMuJoCo(self.model)
# self.solver = newton.solvers.SolverSemiImplicit(self.model, ...)
# self.solver = newton.solvers.SolverFeatherstone(self.model)

self.contacts = None

newton.eval_fk(self.model, self.model.joint_q, self.model.joint_qd, self.state_0)
```

**Verification cues**

- `solver.step(...)` 的签名统一了调用者视角，但没有规定内部要不要用 rows、joint-space mass matrix，或者外部 backend。
- `cartpole` 例子里同一个外壳可以切 solver，这正是 chapter 08 的比较起点。
- 在这个例子里，`eval_fk(...)` 对 maximal-coordinate 路线的意义最直观，这已经暗示“共享 contract”之后马上就会发生 family split。

**Checkpoint**

你现在应该能先讲清一句最基础的话：**Newton 先把 solver 统一成同一个 `step(...)` 调用外壳，但它还没有承诺内部到底先围着谁工作。**

**Output passed to next stage**

同一个 public `step(...)` 入口。接下来要看的不是“谁也有 step”，而是“谁在 step 里面先把什么对象摆成第一主角”。

### Stage 2: `SemiImplicit` 先走 `force route`，把 contacts 翻成力再做积分

**Definition**

`force route`：先把各种影响累计到 `particle_f / body_f` 这种力缓冲区里，再统一做时间积分。

**Claim**

`SolverSemiImplicit` 里最先被当成第一主角的，不是 constraint rows，而是 `particle_f / body_f` 这类力缓冲区。`contacts` 进来后，先被翻成力贡献，再进入 semi-implicit integration。

**Why it matters**

这一步最适合建立 chapter 08 的第一条对照轴：不是所有 solver 都会把 contacts 摆成显式 rows。有的 solver 会先把它们读成力源。

**Source excerpt**

`newton/_src/solvers/semi_implicit/solver_semi_implicit.py` 的 `step()` 基本就是一条直线往下读：

```python
eval_spring_forces(model, state_in, particle_f)
eval_triangle_forces(model, state_in, control, particle_f)
eval_bending_forces(model, state_in, particle_f)
eval_tetrahedra_forces(model, state_in, control, particle_f)

eval_body_joint_forces(model, state_in, control, body_f_work, ...)
eval_particle_contact_forces(model, state_in, particle_f)

eval_body_contact_forces(
    model, state_in, contacts, friction_smoothing=self.friction_smoothing, body_f_out=body_f_work
)

eval_particle_body_contact_forces(model, state_in, contacts, particle_f, body_f_work, ...)

self.integrate_particles(model, state_in, state_out, dt)
self.integrate_bodies(model, state_in, state_out, dt, self.angular_damping)
```

**Verification cues**

- `contacts` 的关键入口是 `eval_body_contact_forces(...)`，不是 `build_rows(...)` 之类的名字。
- 积分 helper 还是来自 `SolverBase`，说明这条路线非常强调“力累计后统一推进状态”。
- 这条路径最适合把 contacts 读成 force contribution，而不是 contact-space solve 对象。

**Checkpoint**

你现在应该能说：**同一个 `step(...)` 到了 `SemiImplicit` 里面，最先站到台前的是 `body_f / particle_f`；所以它的主线是 `contacts -> forces -> integration`。**

**Output passed to next stage**

`SemiImplicit` 这条路给了 chapter 08 的第一个对照基线：共享 contract 之后，可以先走“contacts -> forces -> integration”。

### Stage 3: `Featherstone` 转到 `joint-space route`

**Definition**

`joint-space route`：先把系统拉回 `joint_q / joint_qd / J / M / joint_qdd` 这些 articulation 对象上，再在 generalized coordinates 里完成 solve 和积分。

**Claim**

`SolverFeatherstone` 虽然也吃同一个 public `step(...)`，但它更关心 `joint_q / joint_qd`、articulation Jacobian、mass matrix 和 `joint_qdd`。也就是说，这一家的第一主角已经换成了 joint-space articulation solve。

**Why it matters**

这一跳建立 chapter 08 的第二条对照轴：同样是刚体 solver，有的路线主要围着 body/contact force 打转，有的路线则围着 articulation 的 reduced coordinates 打转。

**Source excerpt**

`newton/_src/solvers/featherstone/solver_featherstone.py` 的主骨架非常直白：

```python
wp.launch(
    eval_rigid_fk,
    dim=model.articulation_count,
    ...,
    outputs=[state_in.body_q, state_aug.body_q_com],
)

if contacts is not None and contacts.rigid_contact_max:
    wp.launch(kernel=eval_body_contact, dim=contacts.rigid_contact_max, ..., outputs=[body_f])

wp.launch(eval_rigid_jacobian, dim=model.articulation_count, ..., outputs=[self.J])
wp.launch(eval_rigid_mass, dim=model.articulation_count, ..., outputs=[self.M])
...
wp.launch(eval_dense_solve_batched, dim=model.articulation_count, ..., outputs=[state_aug.joint_qdd, ...])
wp.launch(kernel=integrate_generalized_joints, dim=model.joint_count, ..., outputs=[state_out.joint_q, ...])
```

**Verification cues**

- 一上来先 `eval_rigid_fk`，说明这条路线先要把 articulation 当前姿态立起来。
- contacts 仍然会影响 `body_f`，但它们在这里更像 joint-space solve 的输入之一，而不是单独摆在台面上的 row system。
- `J / M / joint_qdd / integrate_generalized_joints` 这组名字说明 solver 的真正核心已经换到 generalized coordinates。

**Checkpoint**

你现在应该能说：**同一个 `step(...)` 到了 `Featherstone` 里面，最值得盯的已经不是力缓冲区，而是 `J / M / joint_qdd` 这一组 joint-space 对象。**

**Output passed to next stage**

同一个 public `step(...)` 现在又分出第二种内部语言：`contacts + actuation -> articulation solve -> joint-space integration`。

### Stage 4: `MuJoCo` 走的是 `backend bridge`

**Definition**

`backend bridge`：先把 Newton 里的 `model / state / contacts` 正确翻译给外部 backend，再把 backend 算出来的状态或 contacts 翻回 Newton。

**Claim**

`SolverMuJoCo` 的重点不是在 Newton 仓内把所有求解细节重写一遍，而是把 Newton 的 model/state/(optional contacts) 正确交给 MuJoCo 或 `mujoco_warp` backend，并在需要时把结果再写回 Newton。这里真正的第一主角，是 backend 那边的 state/data 以及两边对象的翻译边界。

**Why it matters**

这一步把 chapter 08 的第三类 solver 路线立住：有的 solver 的核心价值不是“库内怎么解”，而是“怎样桥接到另一套成熟 backend”。

**Source excerpt**

构造器直接把这条边界说得很清楚：

```python
use_mujoco_contacts: If True, use the MuJoCo contact solver.
If False, use the Newton contact solver
(newton contacts must be passed in through the step function in that case).
```

`step()` 里真正的 handoff 则是：

```python
if self.mjw_model.opt.run_collision_detection:
    self._mujoco_warp_step()
else:
    self._convert_contacts_to_mjwarp(self.model, state_in, contacts)
    self._mujoco_warp_step()

self._update_newton_state(self.model, state_out, self.mjw_data, state_prev=state_in)
```

而 `update_contacts(...)` 又保留了反向写回 Newton `Contacts` 的入口：

```python
wp.launch(
    self._convert_mjw_contacts_to_newton_kernel,
    dim=mj_data.naconmax,
    ...,
    outputs=[
        contacts.rigid_contact_count,
        contacts.rigid_contact_shape0,
        contacts.rigid_contact_shape1,
        contacts.rigid_contact_point0,
        contacts.rigid_contact_point1,
        contacts.rigid_contact_normal,
        contacts.force,
    ],
)
```

**Verification cues**

- `use_mujoco_contacts` 这个开关直接决定 contact 是在 backend 里生成，还是先由 Newton 生成后再桥接过去。
- `step()` 的重点是同步数据和调用 backend，而不是在当前文件里展开一个完整 contact solve。
- `update_contacts(...)` 说明这条路线也可能把 backend contact 再翻译回 Newton 语言。

**Checkpoint**

你现在应该能说：**同一个 `step(...)` 到了 `MuJoCo` 里面，最该盯的不是 rows 也不是 `J / M`，而是 Newton 和 backend 之间的数据 handoff。**

**Output passed to next stage**

`MuJoCo` 给了 chapter 08 的第三种内部路线：`Newton objects -> backend solve -> Newton objects`。

### Stage 5: `Kamino` 是 chapter 07 contact math 的直接 continuation

**Definition**

`solver-facing contact`：先把它读成“已经整理成 solver 真正愿意消费的 contact 版本”，不再只是原始碰撞记录。

**Claim**

`SolverKamino` 不只是“第四个 solver 名字”，它是 chapter 07 那条 `Contacts -> solver-facing contact -> rows -> Jacobians -> Delassus` 主线真正落入求解器的一条直接 continuation。

**Why it matters**

如果这一层没读出来，chapter 07 和 chapter 08 就会像两章互不相干的内容。实际上，Kamino 就是把上一章的数学对象真正接进了 solver step。

**Very short recap of chapter 07**

- `Contacts`：还只是 Newton 手里的接触槽位和几何信息。
- `rows`：把这些接触整理成 solver 真正要满足的约束行。
- `Jacobians`：这些约束行怎样作用到 body 或 joint 的速度方向。
- `Delassus`：把质量/惯量投到约束空间后，每一行“看到”的等效质量结构。

第一遍只要记住这四跳就够了：**原始接触数据 -> solver 可用约束 -> 约束和系统的耦合方向 -> 约束空间里的有效质量。** Kamino 正是把这条链真正塞进 `step(...)` 的那一家。

**Source excerpt**

外层 wrapper 先把 Newton `Contacts` 接成 Kamino 自己的 contact 容器：

```python
if contacts is not None:
    self._kamino.convert_contacts_newton_to_kamino(self.model, state_in, contacts, self._contacts_kamino)
    _detector = None
else:
    _detector = self._collision_detector_kamino

self._solver_kamino.step(
    state_in=state_in_kamino,
    state_out=state_out_kamino,
    control=control_kamino,
    contacts=self._contacts_kamino,
    detector=_detector,
    dt=dt,
)
```

内部 solver 构造和主 solve 又把 chapter 07 的名词重新排上了台面：

```python
make_unilateral_constraints_info(model=self._model, data=self._data, limits=self._limits, contacts=contacts)
self._jacobians = DenseSystemJacobians(...)
self._problem_fd = DualProblem(...)
self._solver_fd = PADMMSolver(...)

self._update_constraint_info()
self._update_jacobians(contacts=contacts)
self._update_actuation_wrenches()
self._forward(contacts=contacts)
```

而 `_forward()` 真正做的事情就是：

```python
self._solver_fd.solve(problem=self._problem_fd)
compute_constraint_body_wrenches(...)
unpack_constraint_solutions(...)
```

**Verification cues**

- `Contacts` 先被桥接成 `ContactsKamino`，所以这条路线正好接住了 chapter 07 的 solver-facing contact。
- `make_unilateral_constraints_info` 和 `DenseSystemJacobians` 直接把上一章的约束行 / Jacobian 语言重新摆上台面；`DualProblem` 和 `PADMMSolver` 则把这些对象真正送进约束空间 solve。
- `example_sim_basics_box_on_plane.py` 之所以值得配合看，就是因为它把这条 contact continuation 放进了一个最小 box-ground 场景里。

**Checkpoint**

你现在应该能说：**Kamino 的第一主角已经不是一般意义上的“力”或“backend state”，而是从 `Contacts` 整理出来的 constraints/Jacobians/problem 这套对象；所以它正好续上了 chapter 07。**

**Output passed to next stage**

Kamino 把 chapter 07 的 contact math 真正推进成 solver 内部的 constraint reactions 和 body wrenches。这就是 chapter 08 最重要的 continuity。

## Object Ledger

| 对象 | 谁生产 | 谁消费 | 盯哪些字段 |
|------|--------|--------|------------|
| public `step(state_in, state_out, control, contacts, dt)` | `SolverBase` contract | 所有 solver family | 统一输入/输出边界 |
| `body_f_work` / `particle_f` | `SolverSemiImplicit.step()` | `integrate_particles()`、`integrate_bodies()` | contacts 是否先走 force route |
| `J / M / H / joint_qdd` | `SolverFeatherstone.step()` | generalized-coordinate integrate path | articulation Jacobian、mass matrix、加速度解 |
| MuJoCo backend state (`mj_data` / `mjw_data`) | `SolverMuJoCo` | backend step、`update_contacts()` | collision detection 开关、state/contact round-trip |
| `ContactsKamino` / `DualProblem` / `PADMMSolver` | `SolverKamino` 和 `SolverKaminoImpl` | constrained dynamics solve | Newton contacts 桥接、Jacobian/Delassus continuation、reaction unpack |

## Stop Here

读到这里就已经够 chapter 08 的 80-90% 了。

如果你现在能用自己的话讲顺下面这句话，这一章的 beginner 目标就完成了：

```text
所有 solver 共享同一个 step(...) 外壳；
但真正分家的是它们在内部先围着谁工作：SemiImplicit 围着力缓冲区，Featherstone 围着 joint-space articulation，MuJoCo 围着 backend state/data；
Kamino 则直接接住 chapter 07 的 Contacts -> rows -> Jacobians -> Delassus 主线，继续做 constrained dynamics solve。
```

这时你已经不会再把 chapter 08 读成“又多了四个 solver 名字”。

## Go Deeper

如果你还想继续精确追源码，再去 `source-walkthrough-deep.md`：

- 想保留 cross-repo 精确锚点：看 `Fast Deep Index`
- 想逐跳追 `shared step -> main-object split -> Kamino continuation`：看 `Exact Handoff Trace`
- 想分清 MuJoCo CPU / warp backend、Kamino internal detector 等分支：看 `Optional Branches`
- 想逐条核对这里的 claim：看 `Verification Anchors`
