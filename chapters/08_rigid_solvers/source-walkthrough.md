---
chapter: 08
title: 刚体求解器家族 源码走读
last_updated: 2026-04-19
source_paths:
  - newton/solvers.py
  - newton/_src/solvers/__init__.py
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

这份 walkthrough 只追一条线:

```text
public solver contract
-> family split
-> Kamino continuation path
```

chapter 07 已经把 `Contacts -> rows -> Jacobian -> Delassus` 讲顺了。第 08 章的源码走读不再从 contact math 本身开场，而是先看同一个 `solver.step(...)` 怎样在源码里分叉成完全不同的 rigid solver 路线。

如果你只想做一次最小 first pass，先看这四站就够了:

1. `newton/solvers.py:L5-L8` 和 `L43-L90`
2. `newton/_src/solvers/solver.py:L177-L356`
3. `newton/examples/robot/example_robot_cartpole.py:L60-L71`
4. `newton/_src/solvers/kamino/solver_kamino.py:L511-L579`

## 先带着哪四个问题读

1. 所有 solver 共享的 public contract 到底是什么？
2. 哪些 solver 主要围着 body/contact forces 打转，哪些围着 joint-space articulation 打转？
3. `MuJoCo` 为什么更像 bridge，而不是 Newton 内部完整 solver 展开？
4. chapter 07 的 rows / Jacobians / Delassus 在哪条 solver 路线上最直接继续出现？

## Tier 1: 先看 shared contract

这一层只做一件事: 先把所有 solver 拉回同一个 public API 平面。你先不要急着看哪家更复杂。

| 路径 | 角色 | 先读原因 |
|------|------|----------|
| `newton/solvers.py` | solver 全景、feature matrix、家族说明 | `L5-L8` 先把 `Model / State / Control / Contacts / dt -> solver.step() -> State` 的统一外观立住；`L43-L90` 直接给出四个 solver 的大类差异；`L129-L134` 又把 generalized vs maximal coordinates 说得很明确。 |
| `newton/_src/solvers/__init__.py` | public exports | 只有 `L4-L13` 一小段，但它能提醒你: public API 暴露的是 solver family 名字，不是内部细节树。 |
| `newton/_src/solvers/solver.py` | `SolverBase` 和 shared helpers | `L177-L183` 说明所有 solver 都从 `SolverBase` 派生；`L224-L299` 的 `integrate_bodies / integrate_particles` 提供共用积分 helper；`L301-L316` 则给出 shared `step(...)` signature。 |

这一层读完以后，你应该先能用一句话复述:

```text
Solver API 的相似性，只是在守住统一入口；它没有承诺内部数学相似。
```

顺手再补一句工程事实。`SolverBase` 里除了 `step(...)` 之外，还有 `update_contacts(...)`。这不是所有 solver 都会主动暴露的重点，但对 `MuJoCo` 和 `Kamino` 这类内部还有自己 contact container 的 solver，很有帮助。它提醒你: `contacts` 既可能是输入 handoff，也可能需要被回写。

## Tier 2: 三条重要对照路径

这一层先不要碰 Kamino 的深层内部，而是先把三条最重要的对照路径读顺。

| 路径 | 角色 | 先读原因 |
|------|------|----------|
| `newton/_src/solvers/semi_implicit/solver_semi_implicit.py` | maximal-coordinate baseline | 让你先看清 contact 也可以被当成 force kernel，而不是显式 row solve。 |
| `newton/_src/solvers/featherstone/solver_featherstone.py` | generalized-coordinate articulation solver | 让你先看清同样是 Newton 内部 solver，也可以围着 joint-space state 和 mass matrix 打转。 |
| `newton/_src/solvers/mujoco/solver_mujoco.py` | external backend bridge | 让你先看清有些 solver 的关键不是“库内怎么解”，而是“怎样把 Newton 数据正确交给外部 backend”。 |

### 路径 A: `SemiImplicit` 先做力累加, 再做积分

推荐只盯 `newton/_src/solvers/semi_implicit/solver_semi_implicit.py:L121-L185`。

这段 `step()` 很适合 beginner，因为它几乎就是一条线往下读:

- `L141-L175`：各种力依次累加，包括 joint force、body contact force、particle-body contact force。
- `L167-L170`：`eval_body_contact_forces(...)` 是这条路径里 rigid contact 的关键入口。
- `L177-L185`：最后调用 `integrate_particles(...)` 和 `integrate_bodies(...)`。

如果你读到这里还想把它理解成 chapter 07 的 row solver，就说明你读偏了。它的重点根本不是 rows，而是:

```text
contacts
-> force kernels
-> semi-implicit integration
```

### 路径 B: `Featherstone` 先把 articulation state 立住

`newton/_src/solvers/featherstone/solver_featherstone.py` 最值得按四段读:

1. `L405-L428`：`eval_rigid_fk` 先从 `joint_q / joint_qd` 更新 body poses。
2. `L433-L585`：累积外力、particle contacts 和 rigid contacts 对 body 的作用。
3. `L639-L818`：构造 `J`、`M`、`H`，求 `joint_qdd`。
4. `L828-L932`：积分 generalized joints，再重建 public `body_q / body_qd`。

这段代码最值钱的教学作用，是逼你接受下面这句人话:

```text
同样一个 rigid solver，既可以围着 body/contact force 打转，
也可以围着 articulation joint-space 打转。
```

另外还值得盯一眼 `L555-L585`。它说明 `Featherstone` 并没有不处理 contacts，只是 contact 在这条路线上更像 body force source，而不是单独摆在台面上的 row-space 求解对象。

### 路径 C: `MuJoCo` 的关键是 handoff, 不是库内展开

`newton/_src/solvers/mujoco/solver_mujoco.py` 很大，但第一遍只要抓三块:

- `L2763-L2995`：构造器把 Newton model 导出成 MuJoCo / `mujoco_warp` 世界。这里最关键的开关是 `use_mujoco_contacts`。
- `L3003-L3026`：`step()` 先同步控制和状态，再让 backend 前进一步。若 `use_mujoco_contacts=False`，还会先走 `_convert_contacts_to_mjwarp(...)`。
- `L3681-L3730`：`update_contacts(...)` 可以把 backend contacts 转回 Newton `Contacts`。

所以这条路径真正该读出的不是 MuJoCo 内部细节，而是这条数据流:

```text
Newton model/state/(optional contacts)
-> MuJoCo backend
-> write back to Newton state/(optional contacts)
```

读到这里如果你还想把 `MuJoCo` 和 `Featherstone` 混成一类，只因为它们都更接近 generalized coordinates，那也还差最后一步。两者的关键区别在于: `Featherstone` 的主求解数学还在 Newton 里，`MuJoCo` 的主求解数学主要在 backend 里。

## Tier 3: chapter 07 的直接延续

这一层不再只是做对照，而是进入本章最重要的主路径: `Kamino`。也就是说，从这里开始你不再只是比较 solver family，而是在追 chapter 07 那条 `rows / Jacobians / Delassus` 主线怎样真正落进 solver step。

| 路径 | 角色 | 先读原因 |
|------|------|----------|
| `newton/_src/solvers/kamino/solver_kamino.py` | Newton-facing Kamino wrapper | 先看 public `step(...)` 怎样把 Newton objects 交给 Kamino。 |
| `newton/_src/solvers/kamino/_src/solver_kamino_impl.py` | 真正的 solver 主路径 | 再看 chapter 07 的 rows / Jacobians / constrained dynamics quantities 怎样进入实际 solve。 |

### 先看 `solver_kamino.py`: bridge 在哪里发生

第一遍只盯两段:

- `L403-L434`：构造器把 `model` 转成 `ModelKamino`，准备 `ContactsKamino`，然后实例化真正干活的 `SolverKaminoImpl`。
- `L511-L579`：`step()` 把 Newton `State / Control` 变成 Kamino 视图；如果外部给了 Newton `Contacts`，就 `convert_contacts_newton_to_kamino(...)`；如果没给，就让 detector 去生成 contacts；随后做 body-origin 和 CoM frame 转换，再调用内部 solver。

如果你刚从 chapter 07 过来，这层最值钱的理解就是:

```text
Newton 的 Contacts 还不是 Kamino 最终内部格式。
它会先桥接成 ContactsKamino，再进入后面的 constraints / Jacobians / dynamics 路线。
```

### 再看 `solver_kamino_impl.py`: chapter 07 的名词重新出现

这一层建议分三站读:

1. `L194-L252`：构造阶段已经把 `LimitsKamino`、`DenseSystemJacobians / SparseSystemJacobians`、`DualProblem`、`PADMMSolver`、integrator 都搭好了。
2. `L526-L593`：`step()` 本身很薄，它主要负责读输入、调用 integrator、更新 metrics / time，再写输出。
3. `L1000-L1119`：真正的 constrained dynamics 主线在 `_solve_forward_dynamics(...)`、`_forward(...)` 和 `unpack_constraint_solutions(...)` 里。

第一遍最该盯的几件事是:

- `L1096-L1107`：contact / limit detection 和 constraint info 更新。
- `L1109-L1113`：system Jacobians 和 actuation wrenches 更新。
- `L1000-L1013`：`PADMMSolver.solve(...)` 之后，把 constraint multipliers 变成 body wrenches。
- `L1015-L1032`：再把 multipliers unpack 回 contacts / limits，并更新 warmstart caches。

如果你想把这层压成最小数据流，可以记成:

```text
ContactsKamino
-> constraints info
-> Jacobians
-> DualProblem / PADMM solve
-> constraint multipliers
-> body wrenches and contact reactions
```

这就是 chapter 07 的 direct continuation。也正因为如此，本章才把 Kamino 放在 Tier 3 的最后一层，而不是一上来就把整章写成 Kamino 专章。

## 两个源码锚点怎么配合读

这一章最好用两个例子把不同层级的理解串起来。

### `cartpole` 负责前两层理解

`newton/examples/robot/example_robot_cartpole.py:L60-L71` 是全章最省力的对照锚点:

- `L60-L62` 直接并列了 `SolverMuJoCo`、`SolverSemiImplicit`、`SolverFeatherstone`
- `L67-L68` 故意让 `contacts = None`
- `L70-L71` 直接提醒你 maximal-coordinate solvers 才需要显式 `eval_fk(...)`

这让你在不被 contacts 分心的前提下，先看清 shared contract 一样，但 solver family 已经分叉。

### `box_on_plane` 负责最后一层

`newton/_src/solvers/kamino/examples/sim/example_sim_basics_box_on_plane.py` 则守住 chapter 07 的连续性:

- `L137-L139`：最小 box-ground 场景。
- `L146-L159`：显式暴露 constrained dynamics / PADMM / warmstart 配置。
- `L61-L68`：控制回调只有在 active contacts 存在时才施加外力。

这就把 chapter 07 那条 contact math 主线，从“纸上的 rows / Jacobians / Delassus”推进成了“solver 每一步真的在消费的对象”。

## 最后压成一张总图

```text
shared public contract
-> SemiImplicit: contacts as force kernels
-> Featherstone: contacts and external forces into joint-space articulation solve
-> MuJoCo: Newton data bridged into external backend solve
-> Kamino: contacts become solver-facing constraints, Jacobians, dual problem, PADMM solve
```

推荐的一遍读法是:

1. `newton/solvers.py`
2. `newton/_src/solvers/solver.py`
3. `newton/examples/robot/example_robot_cartpole.py`
4. `newton/_src/solvers/semi_implicit/solver_semi_implicit.py`
5. `newton/_src/solvers/featherstone/solver_featherstone.py`
6. `newton/_src/solvers/mujoco/solver_mujoco.py`
7. `newton/_src/solvers/kamino/solver_kamino.py`
8. `newton/_src/solvers/kamino/_src/solver_kamino_impl.py`
9. `newton/_src/solvers/kamino/examples/sim/example_sim_basics_box_on_plane.py`

读完这一遍，你就应该不会再把第 08 章读成“又多了四个 solver 名字”，而会开始用 shared contract、状态表示和约束处理层级去比较 solver family。
