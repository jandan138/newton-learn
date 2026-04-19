---
chapter: 05
title: 刚体与关节动力学
last_updated: 2026-04-19
source_paths:
  - docs/concepts/articulations.rst
  - newton/_src/sim/enums.py
  - newton/_src/sim/model.py
  - newton/_src/sim/state.py
  - newton/_src/sim/builder.py
  - newton/_src/sim/articulation.py
  - newton/_src/solvers/featherstone/kernels.py
paper_keys:
  - featherstone-book
  - aba-crba-notes
newton_commit: 1a230702
---

# 05 刚体与关节动力学

## 0. `04` 之后还差的，不是更多字段，而是“字段怎样开始动”

`04_scene_usd` 已经把 scene input 压成了 `Model`。这时你再看 `joint_parent / joint_child`、`joint_X_p / joint_X_c`、`joint_q / joint_qd`、`body_mass / body_inertia`，至少知道它们不是凭空出现的。

但读者通常还是会在这里卡住。因为字段虽然已经在 `Model` 里了，你还不知道它们怎样组成 articulation，怎样从 joint-space 坐标长成 body-space 运动，怎样继续被 Featherstone 这类 dynamics 路线消费。

这章只补这段桥。它不是完整 Featherstone 教材，不展开接触数学，也不提前做 rigid solver 家族比较。你现在真正需要的，是先把“静态 articulation 布局 -> FK -> velocity propagation -> inertia consumption”这条主线读顺。

## 1. 先把 articulation 看成“一段 joint 范围 + 一棵 body tree”

第一遍最容易踩的坑，是把 articulation 想成某个嵌套对象。Newton 里的 articulation 更像一套切片约定：body、joint、joint coordinate 仍然都铺在扁平数组里，但有几组索引把“哪些 joint 属于同一条树”标出来。

builder 里的 `add_articulation()` 先做的也正是这件事。它要求 joint 索引连续、单调、处在同一 world，并且每个 child 只能有一个 parent。换句话说，先把“这是一条可前向传播的 joint tree”守住。

到了 `Model` 里，这条树主要靠下面几组容器站住：

- `articulation_start`：每条 articulation 在 joint 数组里的起点。`articulation_start[a] : articulation_start[a + 1]` 就是一整段 joint 范围。
- `joint_articulation`：每个 joint 属于哪条 articulation；如果是 `-1`，说明它不在任何 articulation 里。
- `joint_parent / joint_child`：joint 连接哪两个 body，决定树边本身。
- `joint_ancestor`：从当前 joint 往上回溯时，该接到哪一个上游 joint。
- `joint_X_p / joint_X_c`：同一个 joint frame 分别怎样落在 parent 和 child 两侧。

所以 articulation 的第一层读法可以非常朴素：它不是一坨新数学，而是“同一条 joint tree 在扁平数组里的打包方式”。这样做的直接好处，是 FK、Jacobian、mass matrix 这些递推都能按 articulation 并行，而不必为每台机器人再造一层对象图。

## 2. `joint_q / joint_qd` 和 `body_q / body_qd` 不是同一层状态

这一章最值得先分清的，就是 generalized coordinates 和 maximal coordinates 不是同一种东西。

- `joint_q`：joint-space 的位置坐标。它回答“每个 joint 现在处在自己的参数化里什么位置”。
- `joint_qd`：joint-space 的速度坐标。它回答“沿每个 joint 允许的运动方向，现在走多快”。
- `body_q`：body-space 的世界位姿。每个 body 都是一个 world-space transform。
- `body_qd`：body-space 的世界速度。每个 body 都是一条 spatial velocity；公开语义下，线速度部分是 body COM 的 world-frame 速度。

这两层状态的长度也不一样。`body_q / body_qd` 总是按 body 来排，每个 body 固定占 7 个 pose 参数和 6 个 velocity 参数。`joint_q / joint_qd` 则按 joint 类型变长，所以必须再靠 `joint_q_start` 和 `joint_qd_start` 才能切出每个 joint 的那一段。

第一遍记到下面这张表就够用了：

| Joint type | `joint_q` 坐标数 | `joint_qd` 坐标数 | 第一遍先怎么读 |
|------------|------------------|-------------------|----------------|
| `PRISMATIC` | 1 | 1 | 一条平移轴上的位置和速度。 |
| `REVOLUTE` | 1 | 1 | 一条转轴上的角度和角速度。 |
| `BALL` | 4 | 3 | 位置用 quaternion，速度用 3D 角速度。 |
| `FREE` | 7 | 6 | 位置是 `3 + quat`，速度是 `3 + 3` twist。 |
| `DISTANCE` | 7 | 6 | 第一遍先把它看成和 `FREE` 同样的存储外形。 |
| `FIXED` | 0 | 0 | 没有自由度。 |
| `D6` | 看激活轴数 | 看激活轴数 | 不是固定 6，而是由 `joint_dof_dim` 和 `joint_axis` 决定。 |

这里还有两个很值钱的阅读提醒。

第一，`joint_axis` 不是“每个 joint 一根轴”，而是按 DOF 排的轴表。单自由度 joint 当然只会读一根轴，但 `D6` 这类 joint 会顺着 `joint_dof_dim` 拿多根轴。

第二，`joint_qd` 不能直接等于 `body_qd`。尤其对 `FREE / DISTANCE`，公开接口里的 `joint_qd` 约定是“child COM twist 在 joint parent frame 里怎么写”；而 `body_qd` 则是 world-frame 的 body twist。它们服务的坐标系和参考点都不同。

## 3. FK 是 articulation 第一条真正跑起来的链

一旦 `joint_q` 有了值，articulation 最先做的不是求力，而是先把 pose 和 velocity 往 body 侧传播。`newton.eval_fk()` 和 `_src/sim/articulation.py` 里的 `eval_single_articulation_fk()`，就是这条主线第一次真正变成代码的地方。

这条 FK 链第一遍可以压成四步：

1. 先用 `articulation_start` 找到一条 articulation 对应的 joint 范围。
2. 对每个 joint，按 `joint_type`、`joint_q_start`、`joint_qd_start` 取出自己的那一段坐标。
3. 用 joint 类型、轴和坐标构出当前 joint 内部的相对运动，也就是代码里的 `X_j` 和 `v_j`。
4. 再把 `joint_X_p`、parent 的 `body_q`、以及 child 侧的 `joint_X_c` 串起来，得到 child 的世界位姿 `body_q[child]`，并同步更新 `body_qd[child]`。

这也是 `03`、`04` 留下的局部几何第一次真正开始“动”的地方。`joint_X_p / joint_X_c` 不再只是静态 frame 定义，而是变成了 joint motion 要穿过的两侧锚点。

如果你想把 FK 的人话再压缩一点，可以直接记成：

`joint_q -> joint local transform -> parent chain -> body_q`

这条链一旦稳住，`joint_q` 和 `body_q` 的关系就不再像两份平行数组，而会变成“同一条 articulation 在两种状态层里的两种写法”。

## 4. `joint_qd -> body_qd` 中间隔着一层 motion subspace

速度这边比 pose 更容易误读，因为 `joint_qd` 看起来像“已经是速度了”，但它还不是 body 自己的世界速度。

更好的第一遍读法是：`joint_qd` 只是每个 joint 允许运动方向上的系数；真正的 joint spatial motion，要先把这些系数乘到对应的 motion subspace 上。

对单自由度 joint，这件事其实非常直观：

- prismatic joint：沿某根轴平移，所以 `qd` 乘上一条线速度方向。
- revolute joint：绕某根轴转，所以 `qd` 乘上一条角速度方向。

对多自由度 joint，同样的事只是变成“多列一起加”。Featherstone 侧把这些允许方向记成 `joint_S_s`，`jcalc_motion()` 则负责根据 joint 类型、轴和当前 frame，把每个 DOF 对应的 spatial direction 列出来，再和 `joint_qd` 组合成 joint velocity。

所以 motion subspace 在这章里先不要读成一整套推导。它更像一句很工程的话：joint velocity 不是直接贴到 body 上，而是先回答“这个 joint 允许你朝哪些 spatial directions 动”，`qd` 只是这些方向的系数。

接下来的传播也就顺了：parent 的速度先沿树传下来，joint 自己再贡献一份相对运动，child 才得到自己的 `body_qd`。在 public FK 里，这个结果会被整理回 body COM 的 world-frame twist；在 Featherstone 路线里，还会保留 solver 更方便消费的内部 velocity 约定，并在需要时做额外转换。

## 5. `body_mass / body_com / body_inertia` 在这里正式进入 `dynamics path`

如果 `03` 和 `04` 给你留下的最强印象还是“geometry 先变成 body 级质量属性”，那 chapter 05 最关键的新增，就是这批质量属性终于开始被 articulation dynamics 真正消费。

第一层桥接非常直接：Featherstone kernels 不会重新回头问 shape 长什么样，它们直接从 `body_mass`、`body_com`、`body_inertia` 出发。

- `compute_spatial_inertia()` 先把 `body_mass + body_inertia` 打包成 solver 更方便递推的 spatial inertia `body_I_m`。
- `compute_com_transforms()` 把 `body_com` 写成 `body_X_com`，明确“body frame 到 COM frame”的那段偏移。
- 真正进入 articulation 递推时，再按当前姿态把这些量变到当前消费的 frame 里，例如 `transform_spatial_inertia()` 会把 inertia 从 mass frame 搬到当前 spatial frame。

到这里，你就可以把 inertia bridge 读成一句很稳定的人话：builder 已经把 shape 级质量信息压成了 body 级质量属性；Featherstone 再把这些 body 级质量属性改写成递推友好的 spatial form。

后面的 `body_I_s`、`joint_S_s`、`tau`、Jacobian、mass matrix，都是在继续消费这条桥，而不是另起炉灶。也正因为如此，这章不需要先把 ABA / CRBA 完整推完，你也已经能明白为什么 `body_mass / body_com / body_inertia` 不会停在 importer 或 builder 那一层。

## 6. 带着这条桥，先经过 `06`，再分向 `07` 和 `08`

如果你读到这里，chapter 05 的桥接任务就已经完成了。你应该能把下面这条主线完整说出来：

`Model` 里的 articulation 布局先用 `articulation_start`、`joint_parent / joint_child`、`joint_q_start / joint_qd_start` 站住；`joint_q / joint_qd` 再经过 FK 和 velocity propagation 变成 `body_q / body_qd`；`body_mass / body_com / body_inertia` 则继续被改写成 Featherstone 侧真正消费的 spatial inertia 与递推量。

接下来最自然的阅读顺序通常是先补 `06` 的碰撞入口，再去 `07` 看约束数学，最后到 `08` 看 solver 家族怎样消费这批 articulation buffers。

如果你想按问题来分，也可以把后续入口压成三条：

- 去 `06_collision`：如果你下一步想问“有了 body/world 状态之后，碰撞几何、pair 和候选接触是怎样开始长出来的”。这一层还是 rigid pipeline 的几何入口，不需要先展开约束矩阵。
- 去 `07_constraints_contacts_math`：如果你下一步想问“有了 articulation 结构、body/world 状态和碰撞候选后，Jacobian、约束和接触数学怎样接上来”。
- 去 `08_rigid_solvers`：如果你下一步想问“不同 rigid solver 尤其是 Featherstone、MuJoCo、SemiImplicit、XPBD，分别把这套 articulation 结构当成什么输入，又怎样各自推进”。

所以这章到这里就停。它只负责把 articulation 的结构、坐标布局、FK、motion subspace 和 inertia bridge 接顺；碰撞入口留给 `06`，完整约束数学留给 `07`，solver 家族比较留给 `08`。
