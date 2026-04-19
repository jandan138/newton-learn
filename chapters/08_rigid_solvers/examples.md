---
chapter: 08
title: 刚体求解器家族 例子观察单
last_updated: 2026-04-19
source_paths:
  - newton/examples/robot/example_robot_cartpole.py
  - newton/_src/solvers/kamino/examples/sim/example_sim_basics_box_on_plane.py
  - newton/_src/solvers/kamino/solver_kamino.py
  - newton/_src/solvers/kamino/_src/solver_kamino_impl.py
paper_keys:
  - mujoco-warp-paper
  - kamino-report
  - featherstone-book
newton_commit: 1a230702
---

# 08 刚体求解器家族 例子观察单

`principle.md` 已经把本章的核心问题讲清了: 同一个 `solver.step(...)` contract 后面，可以接完全不同的 rigid solver 路线。这里不再加新理论，只用两个例子把这件事变成可观察现象。

这页故意只留两个 anchor:

- `newton/examples/robot/example_robot_cartpole.py`: 跨 solver 对照例子。
- `newton/_src/solvers/kamino/examples/sim/example_sim_basics_box_on_plane.py`: chapter 07 continuation 例子。

它们分工不同，最好不要拿一个例子同时承担两件教学任务。

## 主例子: `newton/examples/robot/example_robot_cartpole.py`

建议入口:

```bash
python -m newton.examples robot_cartpole --world-count 100
```

这个例子最值钱的地方，不是它多复杂，而是它把“同一个 public scene / loop，可切换不同 solver”这件事写得非常直白。

先盯三处代码就够了:

- `example_robot_cartpole.py:L60-L62`: 只改这三行，就能在 `SolverMuJoCo`、`SolverSemiImplicit`、`SolverFeatherstone` 之间切换。
- `L67-L68`: 这个例子明确把 `contacts` 设成了 `None`。
- `L70-L71`: 注释直接告诉你，只有 maximal-coordinate solvers 需要显式 `eval_fk(...)`。

### 它最适合验证什么

- 同一个 `solver.step(state_0, state_1, control, contacts, dt)` 外壳，后面可以接不同 solver family。
- 不是所有 rigid solver 都需要把 contacts 当主线处理。这个例子里故意没有 contacts，就是为了先把状态表示和推进方式看顺。
- `SemiImplicit`、`Featherstone`、`MuJoCo` 的差别，不需要先从复杂 contact scene 才能看出来。

### 改哪里最值钱

| 改哪里 | 先看什么 | 这能帮你理解什么 |
|--------|----------|------------------|
| 在 `L60-L62` 切换 solver 构造器 | 外层 simulation loop 基本不变 | shared contract 是真的共享的 |
| 保持 `contacts = None` 不变 | scene 仍然能正常推进 | solver family 的第一层差别不依赖接触场景才看得见 |
| 注意 `L70-L71` 的 `eval_fk(...)` 注释 | 只有 maximal-coordinate solvers 需要这一步初始化 | `SemiImplicit` 和 `Featherstone / MuJoCo` 的主状态表示不一样 |

### 第一遍最值得观察什么

- 换 solver 时，`simulate()` 里的主循环几乎不变，说明 public API 确实统一。
- 但“哪一组状态在当主角”已经不同了。`SemiImplicit` 更像在推进 body state，`Featherstone` 和 `MuJoCo` 更像在推进 articulation / backend state。
- 这个例子没有 contacts，所以它非常适合提醒你: chapter 08 不是“contact 章节的继续抄写”，而是“solver family 怎样消费共享接口”的章节。

### 如果结果和预期不一样, 你大概率误解了什么

- 如果你一看到 `contacts=None` 就觉得这个例子对第 08 章没用，你低估了 shared contract 的教学价值。
- 如果你把三个 solver 都想成“只是实现细节不同”，你还没有真正注意到 `eval_fk` 注释暗示的 coordinate split。
- 如果你在这个例子里硬找 chapter 07 的 rows / Delassus 主线，你就拿错了例子。它不是为那件事准备的。

## 对照例子: `newton/_src/solvers/kamino/examples/sim/example_sim_basics_box_on_plane.py`

这个例子承担的是另一件事: **把 chapter 07 留下来的 contact math，真正送进 solver。**

如果你用的是 pip 安装版 Newton，而不是源码工作区，这个路径可能并不直接可运行。第一遍把它当成“跟源码一起看的观察锚点”也完全够用；重点是看 Kamino 怎样消费 contact-driven constrained dynamics 数据。

它最值得先看的不是整份脚本，而是下面几段:

- `example_sim_basics_box_on_plane.py:L137-L139`: 用 `build_box_on_plane` 搭最小的 box-ground 接触场景。
- `L146-L159`: solver config 明确在配 preconditioning、PADMM 容差、warmstart、solver info。
- `L61-L68`: control callback 只有在 `wnc > 0` 时才施加外力。
- `L183-L190`: viewer 打开了 `show_contacts=True`。

### 它最适合验证什么

- chapter 07 的 contact math 不再只是 `Contacts -> rows` 的纸面桥，而是真的进入 solver step。
- `Kamino` 不是把接触当成普通力源，而是把 contacts 当成 constrained dynamics 里的一级对象。
- `box-on-plane` 比 `cartpole` 更适合看 contact-driven solver 行为，因为它直接把接触和摩擦放在舞台中央。

### 改哪里最值钱

| 改哪里 | 先看什么 | 这能帮你理解什么 |
|--------|----------|------------------|
| 先只盯 `L146-L159` 的 solver config | 参数围绕的是 constraints / PADMM / warmstart，而不是普通积分器选项 | `Kamino` 的主问题真的是 constrained dynamics |
| 再看 `L61-L68` 的 `wnc > 0` 条件 | 外力是否生效取决于 active contacts | contacts 在这里是 live solver data |
| 打开 viewer 看 `show_contacts=True` | 接触不是旁观信息，而是这条 solver 路线的主角之一 | chapter 07 的 contact spine 在这里继续活着 |

### 第一遍最值得观察什么

- `box_on_plane` 的价值不在“场景更复杂”，而在它让你把 `contact exists` 和 `solver behavior changes` 直接连起来。
- 你会更容易接受 `Kamino` 这一类 solver 的主线不是“先算一堆力再积分”，而是“先把 constraints / contacts 组织成求解问题，再求反力和更新状态”。
- 这个例子也更接近 chapter 07 的心智模型，因为 box-ground 接触本来就是最适合想象多 contact、摩擦和约束 block 的画面。

### 如果结果和预期不一样, 你大概率误解了什么

- 如果你把这个例子也看成“只是另一个场景 demo”，你就漏掉了 `PADMM`、warmstart 和 active contacts 这些关键词。
- 如果你觉得 `contacts` 只是 viewer 上的箭头，而不是真正参与 solver，你就没有注意到 `wnc > 0` 的控制逻辑。
- 如果你还把 `Kamino` 想成“另一种积分器”，你就把 constrained dynamics 路线写扁了。

## 两个例子怎么配合读最顺

推荐顺序不要反过来。

1. 先看 `cartpole`。
原因: 它最干净，能先把 shared contract 和 coordinate split 看顺，不会被 contacts 分心。

2. 再看 `box_on_plane`。
原因: 当你已经知道 solver family 会分叉，再回来看 `Kamino` 怎样直接消费 contacts / constraints，就更容易把 chapter 07 和 chapter 08 连起来。

你也可以把两者压成一句更短的记忆法:

```text
cartpole 负责看“外层一样”
box_on_plane 负责看“内部为什么不一样”
```

或者更像一张阅读路线图:

```text
cartpole
-> shared contract / family split
-> box_on_plane
-> chapter 07 contact math 真正进入 solver
```

## 误解自检

如果你还会这样想，就该回到 `principle.md` 重读 family split 那一节:

- “四个 solver 只是不同实现的同一套 row solver。”
- “`MuJoCo` 既然也吃 `contacts`，那它应该也算 Newton 内部那条 row math 主线。”
- “`SemiImplicit` 和 `Kamino` 都能处理刚体接触，所以本质上只差调参。”

更稳的说法应该是:

- `SemiImplicit` 是 force-kernel / integration baseline。
- `Featherstone` 是 Newton 内部的 generalized-coordinate articulation 路线。
- `MuJoCo` 是 external backend bridge。
- `Kamino` 是 chapter 07 contact math 的直接 continuation。

## 自检

- 现在只看 `cartpole`，你能不能解释为什么“外层 loop 一样”不代表“内部数学一样”？
- 现在只看 `box_on_plane`，你能不能解释为什么 `Kamino` 比 `SemiImplicit` 更像 chapter 07 的直接下一跳？
