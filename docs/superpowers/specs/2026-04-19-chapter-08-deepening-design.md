# Chapter 08 Deepening Design

- **日期**: 2026-04-19
- **仓库**: `/home/zhuzihou/dev/newton-learn`
- **范围**: `chapters/08_rigid_solvers`
- **目标**: 把 `08_rigid_solvers` 写成一章 teaching-first 的“不同 rigid solver 怎样消费 chapter 07 留下的 contact math / articulation state”桥梁章，而不是一篇平铺 solver 名字的目录。

---

## 1. 背景

主线已经走到：

- `05_rigid_articulation` 讲清了 `joint_q / joint_qd`、`body_q / body_qd` 和 articulation 结构
- `06_collision` 讲清了 `body/world state + shape data -> Contacts`
- `07_constraints_contacts_math` 讲清了 `Contacts -> rows -> Jacobian -> Delassus`

所以 chapter 08 的自然任务不是重讲 contact math，也不是把所有 solver 一字排开做名词表，而是回答：

**当这些状态和约束对象都已经准备好以后，不同 rigid solver 到底怎样消费它们？为什么看起来都叫 `solver.step(...)`，内部走法却完全不同？**

---

## 2. 设计判断

### 2.1 不能写成 flat solver catalog

如果 chapter 08 平铺成：

- `SemiImplicit`
- `Featherstone`
- `MuJoCo`
- `Kamino`

然后每个 solver 同等篇幅介绍一遍，教学风险很高：

1. 读者只会记住“又有四个名字”
2. 看不出它们的数学分野
3. chapter 07 建好的 rows / Jacobians / Delassus 主线会突然失焦

所以 chapter 08 不能是 catalog，它必须先建立比较轴。

### 2.2 最稳的写法是“shared contract first, family split second”

推荐主结构：

1. 先用 `newton.solvers` 和 `SolverBase.step(state_in, state_out, control, contacts, dt)` 建立 shared contract
2. 再告诉读者：同样这个 contract，内部至少分成四种不同消费方式
3. 再按数学家族和状态表示把 solver 分层

这比直接背 solver 名字更容易让初学者建立稳定坐标。

### 2.3 chapter 07 的直接延续必须是 Kamino，但章节入口不能只讲 Kamino

Kamino 是 chapter 07 的最自然下一跳，因为：

- rows / Jacobians / Delassus 在它那里最显式
- `Contacts` 会继续桥接成 solver-facing contact 表示
- `constraints`、`dynamics`、`padmm` 路径最能回答“solver 到底在吃什么”

但 chapter 08 不能一开头就只讲 Kamino。否则会把“solver family”误写成“Kamino 专章”。

更稳的顺序是：

1. 先告诉读者 solver 们共有哪层 public contract
2. 再按家族区分内部数学风格
3. 再让 Kamino 成为 contact-math 主线的 canonical path
4. 其他 solver 作为重要对照

### 2.4 最重要的比较轴不是“功能表”，而是“状态表示 + 约束在哪一层被处理”

最适合 beginner 的比较轴不是 feature matrix，因为 `newton/solvers.py` 里已经有支持矩阵。

chapter 08 更应该优先比较：

1. 用的是 maximal coordinates 还是 generalized coordinates
2. contact / joint constraints 是被当成 force、joint-space solve、外部引擎黑箱，还是显式 constrained dynamics 问题
3. `Contacts` 是继续直接消费、桥接后消费，还是被外部 backend 接管

### 2.5 例子要分成“跨 solver 共用例子”和“contact-math 连续例子”两类

推荐不要只用一个例子解决所有任务。

推荐两类例子：

1. **跨 solver 共用例子**：`newton/examples/robot/example_robot_cartpole.py`
   - 优点：文件里已经并列展示 `SolverMuJoCo`、`SolverSemiImplicit`、`SolverFeatherstone`
   - 很适合解释 solver contract 一样，但状态表示和内部推进不同

2. **chapter 07 连续例子**：`newton/_src/solvers/kamino/examples/sim/example_sim_basics_box_on_plane.py`
   - 优点：能把 `Contacts -> rows -> Jacobians -> Delassus -> solver step` 这条线真正闭环到 Kamino
   - 很适合保持 `07 -> 08` 连续性

---

## 3. chapter 08 应该解决什么

读完 chapter 08 后，读者至少应更容易解释：

1. 为什么四个 solver 看起来都接受 `model/state/control/contacts/dt`，但内部数学完全不同。
2. `SemiImplicit` 为什么更像 force accumulation + integration baseline，而不是 row solver。
3. `Featherstone` 为什么自然落在 generalized coordinates / articulation 路线。
4. `MuJoCo` 为什么更像 external backend bridge，而不是 Newton 内部完整 solver 数学展开。
5. `Kamino` 为什么是 chapter 07 contact math 最直接的 consumer。

---

## 4. chapter 08 不该解决什么

1. 不做完整 CRBA / ABA / PADMM / MuJoCo solver 理论课。
2. 不重写 `newton/solvers.py` 里的功能支持矩阵。
3. 不给“哪个 solver 最好”的场景调参指南。
4. 不把 MuJoCo 内部求解器细节反向讲成 Newton 章节。
5. 不展开所有 softbody / particle / cloth solver；本章只聚焦 rigid-relevant solver 家族。

---

## 5. 推荐 approaches 与取舍

### Approach A: Flat solver catalog

- 做法：每个 solver 各写一节，同等篇幅
- 优点：表面上最完整
- 缺点：最容易写散，最不 teaching-first

### Approach B: Shared contract first, one canonical path, others as contrast

- 做法：先讲 shared contract，再讲 Kamino 主路径，其他 solver 做比较
- 优点：chapter 07 延续最顺
- 缺点：如果不加 family framing，容易让其他 solver 沦为“配角附录”

### Approach C: Solver families first by math style

- 做法：先按 maximal/generalized/external/constrained dynamics 分家族，再映射到实现
- 优点：数学分野最清楚
- 缺点：如果一上来就全讲 family taxonomy，会离 chapter 07 的具体 handoff 太远

### 推荐: B with C framing

也就是：

1. shared contract first
2. family split second
3. Kamino 作为 canonical continuation
4. SemiImplicit / Featherstone / MuJoCo 作为对照

这样既保住 chapter 07 的连续性，也不把 solver family 的数学差异写扁。

---

## 6. 文件级设计

### 6.1 `README.md`

职责：

- 明确 chapter 08 在 `07_constraints_contacts_math` 后面的位置
- 强调这章的核心问题不是“还有哪些 solver”，而是“同样的 solver contract，内部到底怎样不同”
- 给出最关键的 prereq self-check：
  - 如果你还不能解释 `Contacts -> rows -> Jacobian -> Delassus`，先回看 chapter 07

### 6.2 `principle.md`

建议结构：

1. `07 -> 08` 桥段：同样的 rows / Jacobians / Delassus，不同 solver 怎样接手
2. shared contract：`solver.step(model/state/control/contacts/dt)` 看起来都一样
3. 第一层大分野：maximal vs generalized coordinates
4. `SemiImplicit`：force-based baseline，contact 更像 force kernel
5. `Featherstone`：generalized coordinates inside Newton，articulation / mass matrix / joint-space solve
6. `MuJoCo`：external backend bridge，solver internals 不在 Newton 内部展开
7. `Kamino`：constrained dynamics 主路径，chapter 07 的直接 consumer
8. 最后给一个教学型对照表

### 6.3 `source-walkthrough.md`

必须分层：

#### Tier 1: shared contract

- `newton/solvers.py`
- `newton/_src/solvers/__init__.py`
- `newton/_src/solvers/solver.py`

#### Tier 2: three contrastive paths

- `newton/_src/solvers/semi_implicit/solver_semi_implicit.py`
- `newton/_src/solvers/featherstone/solver_featherstone.py`
- `newton/_src/solvers/mujoco/solver_mujoco.py`

#### Tier 3: chapter 07 continuation path

- `newton/_src/solvers/kamino/solver_kamino.py`
- `newton/_src/solvers/kamino/_src/solver_kamino_impl.py`

### 6.4 `examples.md`

建议两条主例子：

1. `newton/examples/robot/example_robot_cartpole.py`
   - 用来解释“同一个 public scene / model，solver 可以换，但内部状态表示和推进方式不同”
2. `newton/_src/solvers/kamino/examples/sim/example_sim_basics_box_on_plane.py`
   - 用来解释“chapter 07 那条 contact-math 主线在 solver 里怎样真正落地”

---

## 7. 关键源码锚点

第一遍推荐顺序：

1. `newton/solvers.py`
2. `newton/_src/solvers/solver.py`
3. `newton/examples/robot/example_robot_cartpole.py`
4. `newton/_src/solvers/semi_implicit/solver_semi_implicit.py`
5. `newton/_src/solvers/featherstone/solver_featherstone.py`
6. `newton/_src/solvers/mujoco/solver_mujoco.py`
7. `newton/_src/solvers/kamino/solver_kamino.py`
8. `newton/_src/solvers/kamino/_src/solver_kamino_impl.py`
9. `newton/_src/solvers/kamino/examples/sim/example_sim_basics_box_on_plane.py`

---

## 8. 教学风险

chapter 08 写作时必须主动避免：

1. 把 `SemiImplicit` 误讲成 row / Delassus solver。
2. 把 `Featherstone` 的 articulation mass matrix 路线和 contact Delassus 混成同一种东西。
3. 把 MuJoCo 讲成“Newton 内部完整 solver 数学”，而忽略它的 external backend 属性。
4. 一上来就掉进 PADMM / ABA 推导，让章节失去 teaching-first 入口。
5. 用 feature matrix 取代解释，导致读者知道“支持什么”却不知道“为什么内部不同”。

---

## 9. 成功标准

如果 chapter 08 写完后达成下面状态，就说明设计是成功的：

1. 读者不会再把所有 solver 想成“换个名字而已”。
2. 读者能先用一句话说清每个 solver family 的核心差别。
3. 读者能理解为什么 Kamino 是 chapter 07 的直接延续，而 SemiImplicit / Featherstone / MuJoCo 则各自代表不同消费方式。
4. 读者读 walkthrough 时，知道自己先看 shared contract，再看 family split，最后看 Kamino 主路径。

---

## 10. 预期改动文件

- `chapters/08_rigid_solvers/README.md`
- `chapters/08_rigid_solvers/principle.md`（新增）
- `chapters/08_rigid_solvers/source-walkthrough.md`（新增）
- `chapters/08_rigid_solvers/examples.md`（新增）
- `docs/superpowers/specs/2026-04-19-chapter-08-deepening-design.md`
