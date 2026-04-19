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

这份 walkthrough 只追 `articulation layout -> runtime state -> FK/world-space spatial quantities -> Featherstone consumption` 这条线。它不推完整 ABA / CRBA，也不把 `solver_featherstone.py` 读成一整本刚体动力学教材。第一遍最重要的是看清：Newton 里的 articulation 不是嵌套对象树，而是几组扁平数组切片；`joint_q / joint_qd` 先怎样被解释成 `body_q / body_qd`；这些 buffers 又怎样继续喂给 Featherstone kernels。公开语义可以先回看 `docs/concepts/articulations.rst:L18-L46`, `docs/concepts/articulations.rst:L315-L425`, `docs/concepts/articulations.rst:L663-L670`，这页只把那套语义对到真实源码。

## 涉及路径

| 路径 | 角色 | 先读原因 |
|------|------|----------|
| `newton/_src/sim/builder.py` | articulation 布局的起点。`add_articulation()` 验证 joint 连续、单调、树结构，然后登记 `articulation_start` 和 `joint_articulation`，见 `newton/_src/sim/builder.py:L1965-L2062`；`add_joint()` 继续把 `joint_parent / joint_child / joint_X_p / joint_X_c / joint_dof_dim / joint_q_start / joint_qd_start` 写进 builder，见 `newton/_src/sim/builder.py:L3590-L3668`；`finalize()` 再补齐 sentinel start arrays 并冻结成 `Model`，见 `newton/_src/sim/builder.py:L10234-L10258`。 | 先把 articulation 看成“joint 范围和切片约定”，后面的 FK 和 solver 输入才不会像凭空读数组。 |
| `newton/_src/sim/model.py` | articulation 的静态字段目录。`body_mass / body_com / body_inertia`、`joint_articulation / joint_axis / joint_dof_dim / joint_q_start / joint_qd_start`、`articulation_start` 都在这里定名，见 `newton/_src/sim/model.py:L390-L492`, `newton/_src/sim/model.py:L566-L572`；`Model.state()` 则把这些默认值克隆成 runtime `State` 快照，见 `newton/_src/sim/model.py:L808-L859`。 | 先认清静态布局和 runtime 初始化的分界线。 |
| `newton/_src/sim/state.py` | articulation 在 runtime 的两层快照：`joint_q / joint_qd` 是 generalized state，`body_q / body_qd` 是 body/world state，见 `newton/_src/sim/state.py:L72-L116`。 | 这一页的主线正是这两层状态之间怎样互相桥接。 |
| `newton/_src/sim/articulation.py` | public articulation helper 主干。`eval_single_articulation_fk()` 按 `joint_q_start / joint_qd_start` 切 joint 坐标、生成 `X_j / v_j`，再沿 parent-child 树写出 `body_q / body_qd`，见 `newton/_src/sim/articulation.py:L191-L374`；`eval_fk()` 再按 articulation 启动 kernel，见 `newton/_src/sim/articulation.py:L377-L516`。 | 这里是 `joint_q / joint_qd -> body_q / body_qd` 第一次真正跑起来的地方。 |
| `newton/_src/solvers/featherstone/kernels.py` | Featherstone 对 articulation buffers 的第一层消费。`compute_spatial_inertia()` 和 `compute_com_transforms()` 把 body 质量属性改写成 `body_I_m / body_X_com`，见 `newton/_src/solvers/featherstone/kernels.py:L20-L49`；`jcalc_transform()`、`jcalc_motion()`、`eval_rigid_fk()`、`compute_link_velocity()` 则把 joint layout 继续变成 solver 内部的 `body_q_com / joint_S_s / body_I_s / body_v_s / body_f_s`，见 `newton/_src/solvers/featherstone/kernels.py:L139-L333`, `newton/_src/solvers/featherstone/kernels.py:L641-L801`。 | 这是 articulation 结构开始变成 Featherstone 递推输入的地方。 |
| `newton/_src/solvers/featherstone/solver_featherstone.py` | orchestration 收口。`_compute_articulation_indices()` 把每条 articulation 转成 batched matrix offsets，见 `newton/_src/solvers/featherstone/solver_featherstone.py:L229-L298`；`_allocate_model_aux_vars()` 预分配 `body_I_m / body_X_com`，见 `newton/_src/solvers/featherstone/solver_featherstone.py:L299-L331`；`step()` 再把 `eval_rigid_fk -> eval_rigid_id -> eval_rigid_tau -> eval_rigid_mass -> integrate_generalized_joints -> eval_fk_with_velocity_conversion` 串起来，见 `newton/_src/solvers/featherstone/solver_featherstone.py:L379-L931`。 | 先看这里，才能知道 solver 不是重新发明一套状态，而是在继续消费 chapter 05 这批 articulation buffers。 |

## 调用链总览

1. articulation 的第一层定义仍然发生在 builder。`add_joint()` 给每个 joint 记下 parent-child 拓扑、两侧局部锚点、DOF 维度、轴以及 `joint_q_start / joint_qd_start`；`add_articulation()` 再要求 joint 连续、单调、同 world、无多 parent，最后只留下 `articulation_start` 和 `joint_articulation` 这组切片约定，见 `newton/_src/sim/builder.py:L1965-L2062`, `newton/_src/sim/builder.py:L3590-L3668`。到 `finalize()`，这些 Python lists 会被补成带 sentinel 的 start arrays，冻结到 `Model`，见 `newton/_src/sim/builder.py:L10234-L10258`, `newton/_src/sim/model.py:L445-L492`, `newton/_src/sim/model.py:L566-L572`。
2. `Model` 持有静态布局，`State` 持有运行时快照。`Model.state()` 会把默认 `joint_q / joint_qd / body_q / body_qd` 克隆到新的 `State`，所以 solver 始终同时握着 `generalized coordinates` 和 `maximal coordinates` 两层状态，见 `newton/_src/sim/model.py:L808-L859`, `newton/_src/sim/state.py:L72-L116`。这也正对应了概念文档里的公开语义，见 `docs/concepts/articulations.rst:L18-L46`。
3. public articulation helper 的桥接重点不是求力，而是先把 joint-space 坐标解释出来。`eval_single_articulation_fk()` 用 `joint_q_start / joint_qd_start` 和 `joint_dof_dim` 找到每个 joint 的那一段 `joint_q / joint_qd`，按 joint 类型生成 `X_j / v_j`，再把 `joint_X_p`、parent 的 `body_q / body_qd`、`joint_X_c` 串起来，得到 child 的 `body_q / body_qd`，见 `newton/_src/sim/articulation.py:L227-L374`。`eval_fk()` 再按 `articulation_start[a] : articulation_start[a + 1]` 并行发起整条 FK，见 `newton/_src/sim/articulation.py:L377-L516`。
4. Featherstone 路线并不绕开这些结构，而是把它们换成 solver 更方便消费的 buffers。初始化时，`compute_spatial_inertia()` 和 `compute_com_transforms()` 先把 `body_mass / body_inertia / body_com` 变成 `body_I_m / body_X_com`，见 `newton/_src/solvers/featherstone/kernels.py:L20-L49`, `newton/_src/solvers/featherstone/solver_featherstone.py:L299-L331`。step 开头，solver 先用 `eval_rigid_fk()` 刷新 `body_q / body_q_com`，再把 `joint_qd` 和 joint wrench 转成内部约定，接着由 `eval_rigid_id()`、`eval_rigid_tau()`、`eval_rigid_mass()` 继续消费 `articulation_start`、`joint_q_start / joint_qd_start`、`joint_axis`、`joint_dof_dim`、`body_I_m`、`body_X_com` 这些缓存，见 `newton/_src/solvers/featherstone/solver_featherstone.py:L405-L553`, `newton/_src/solvers/featherstone/solver_featherstone.py:L596-L931`。
5. 所以这页最值得记住的不是某条递推公式，而是 articulation 的四段接力：`builder/model` 先定义布局，`state` 携带两层快照，`articulation.py` 负责把 generalized state 译成 body/world spatial quantities，Featherstone 再继续消费这些 quantities。完整 solver 数学留给后续章节，这里只保住“结构怎样接上去”。

## 数据流切片

| 切片 | 读入 | 中间接力 | 写出 | 证据 |
|------|------|----------|------|------|
| joint layout / starts / dims -> coordinate interpretation | joint 类型、`linear_axes / angular_axes`、两侧 joint frame，以及 articulation 由哪些 joint 组成。 | `add_joint()` 先按 joint 类型累计 `joint_axis`、`joint_dof_dim`、`joint_q_start`、`joint_qd_start` 并为 `joint_q / joint_qd` 预留槽位；`add_articulation()` 再把一段连续 joint 记成 `articulation_start`；`finalize()` 补上 `joint_coord_count / joint_dof_count / joint_count` sentinel，方便内核用 `[start:end]` 切片。概念文档随后用同一套 `joint_q_start / joint_qd_start / joint_dof_dim` 说明怎样解释每个 joint 的 coordinate layout。 | `Model.joint_articulation`、`Model.joint_axis`、`Model.joint_dof_dim`、`Model.joint_q_start`、`Model.joint_qd_start`、`Model.articulation_start`，以及“某个 joint 在 `joint_q` 和 `joint_qd` 里各占哪一段”的统一读法。 | `newton/_src/sim/builder.py:L1965-L2062`, `newton/_src/sim/builder.py:L3590-L3668`, `newton/_src/sim/builder.py:L10234-L10258`, `newton/_src/sim/model.py:L445-L492`, `docs/concepts/articulations.rst:L318-L425` |
| `joint_q / joint_qd` -> FK -> `body_q / body_qd` | `State.joint_q`、`State.joint_qd`，加上 `Model.joint_X_p / joint_X_c / joint_axis / joint_dof_dim / body_com`。 | `eval_single_articulation_fk()` 先按 `q_start / qd_start` 取出该 joint 的坐标，构造 `X_j / v_j`；再沿 `X_wpj -> X_wcj -> X_wc` 做 pose 传播，并在 child body origin 与 COM 之间做速度换读，最后写回 `body_q[child]` 和 `body_qd[child]`。`eval_fk()` 用 `articulation_start` 逐条 articulation 启动这段传播。 | `State.body_q` 的 world transform，和按 Newton 公共语义写出的 `State.body_qd` COM/world twist。 | `newton/_src/sim/state.py:L72-L116`, `newton/_src/sim/articulation.py:L191-L374`, `newton/_src/sim/articulation.py:L377-L516`, `docs/concepts/articulations.rst:L663-L670` |
| body mass/inertia -> spatial inertia | `Model.body_mass`、`Model.body_inertia`、`Model.body_com`。 | Featherstone 先用 `compute_spatial_inertia()` 把 body 质量属性改写成质量框架下的 `body_I_m`，再用 `compute_com_transforms()` 把 `body_com` 写成 `body_X_com`。进入 `compute_link_velocity()` 后，这些量还会结合当前 `body_q_com` 被变换成 solver 递推里的 `body_I_s`，并与 `body_v_s / body_f_s` 同步更新。这里先把它读成“body 级质量属性变成 spatial form”，不展开更深的动力学推导。 | `body_I_m`、`body_X_com`、`body_I_s`，以及后续递推会继续消费的 body-space spatial buffers。 | `newton/_src/sim/model.py:L395-L402`, `newton/_src/solvers/featherstone/kernels.py:L20-L49`, `newton/_src/solvers/featherstone/kernels.py:L717-L801`, `newton/_src/solvers/featherstone/solver_featherstone.py:L299-L331` |
| articulation buffers -> Featherstone step inputs | `Model.articulation_start`、`Model.joint_q_start / joint_qd_start`、`Model.joint_ancestor`、`State.joint_q / joint_qd / body_q`，以及上一步已经准备好的 `body_I_m / body_X_com`。 | `_compute_articulation_indices()` 先把每条 articulation 变成 batched `J / M / H` offsets；`step()` 再依次执行 `eval_rigid_fk()`、public/internal `FREE`/`DISTANCE` 速度转换、`eval_rigid_id()`、`eval_rigid_tau()`、`eval_rigid_jacobian()`、`eval_rigid_mass()`、线性求解与 `integrate_generalized_joints()`，最后用 `eval_fk_with_velocity_conversion()` 把更新后的 generalized state 重建回 public `body_q / body_qd`。 | solver 工作区里的 `joint_S_s`、`body_q_com`、`body_I_s`、`body_v_s`、`body_f_s`、`J / M / H / L`，以及最终更新后的 `state_out.joint_q / joint_qd / body_q / body_qd`。 | `newton/_src/solvers/featherstone/solver_featherstone.py:L229-L331`, `newton/_src/solvers/featherstone/solver_featherstone.py:L405-L553`, `newton/_src/solvers/featherstone/solver_featherstone.py:L596-L931` |

## GPU 并行切片

- public `eval_fk()` 是按 articulation 并行启动的：每个 thread 读一段 `articulation_start[a] : articulation_start[a + 1]`，所以 articulation 在代码里的真正并行单元就是“joint 范围”，不是对象树，见 `newton/_src/sim/articulation.py:L377-L516`。
- Featherstone 也沿用这套切片。`compute_spatial_inertia()` 和 `compute_com_transforms()` 按 body 批量预处理；`eval_rigid_fk()`、`eval_rigid_id()`、`eval_rigid_mass()`、`eval_rigid_jacobian()` 则按 articulation 批量消费 joint layout，见 `newton/_src/solvers/featherstone/kernels.py:L20-L49`, `newton/_src/solvers/featherstone/kernels.py:L641-L801`, `newton/_src/solvers/featherstone/solver_featherstone.py:L405-L931`。

这页到这里就停。第一遍只需要把 `layout -> FK -> spatial buffers -> Featherstone inputs` 这条桥读顺；为什么这些 buffers 能支撑完整 Featherstone 递推、`J / M / H` 各自的数学意义是什么，留给后面的 solver 章节再展开。
