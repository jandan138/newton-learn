# Chapter 07 Deepening Design

- **日期**: 2026-04-19
- **仓库**: `/home/zhuzihou/dev/newton-learn`
- **范围**: `chapters/07_constraints_contacts_math`
- **目标**: 把 `07_constraints_contacts_math` 写成一章真正可教的“`Contacts` 怎样继续长成约束行、Jacobian 和 Delassus”桥梁章，而不是一篇抽象术语笔记。

---

## 1. 背景

主线已经走到：

- `05_rigid_articulation` 讲清了 articulation buffers、`joint_q / joint_qd` 和 `body_q / body_qd`
- `06_collision` 讲清了 `body/world state + shape data -> candidate pair -> contact geometry -> Contacts`

所以 chapter 07 的自然任务不是重讲碰撞，也不是直接跳进 solver family 比较，而是回答一个更具体的问题：

**当 chapter 06 已经把球和地面的接触写进 `Contacts` 之后，这些接触点、法线和距离，怎样继续长成约束行、Jacobians、effective mass / Delassus 这批后续 solver 真正会吃的数学对象？**

---

## 2. 设计判断

### 2.1 必须先补 `06 -> 07` 的桥

chapter 06 结尾停在“`Contacts` 是 handoff object”。chapter 07 开头必须立刻补上一句新的阅读视角：

- chapter 06 问的是“现在几何上接触发生在哪、法线朝哪、还隔多少”
- chapter 07 问的是“这些几何信息希望系统阻止哪种相对运动，又怎样写成 solver 可消费的约束方向”

这句桥接话必须在 `README.md` 和 `principle.md` 开头都出现，不然读者会觉得换了题目，而不是继续同一条主线。

### 2.2 起手例子必须锁定为 `sphere-ground`

第一例必须明确选：

- 一个球贴近地面，只有一处最容易想象的接触

原因：

1. 球和地面的法线最容易想象
2. 最容易把“几何接触”翻译成“先阻止法向继续闭合”
3. 最容易接受“一处接触点，不是只长成一个标量，而会长成 1 条法向 row + 2 条切向 row”

### 2.3 第二例必须是 `box-ground`

第二例再引：

- 一个箱子贴地或轻微倾斜贴地

作用不是重复球，而是专门教读者两件更难的事：

1. 一个 shape pair 不一定只有一个 contact
2. 接触点离质心有偏移时，Jacobian 不只管平移，也会带来角运动耦合

### 2.4 这章不能从 `J M^{-1} J^T`、LCP、ADMM 开场

这章依然是数学味最重的一章，但不能让术语决定顺序。推荐顺序必须是：

1. 具体接触画面
2. 这个接触到底想阻止什么相对运动
3. 为什么一个接触会长成 normal / tangent 三条 row
4. Jacobian 在这里怎样把“哪种运动会影响接触”写出来
5. Delassus / effective mass 第一层物理意义
6. 再把后续 solver 留给 `08`

### 2.5 source-walkthrough 必须分层，不是平铺文件清单

第一遍 walkthrough 不能把 8-10 个文件平铺给读者。必须显式分层：

- Tier 1: 先看 contact geometry 怎样写入 `Contacts`
- Tier 2: 再看一个 contact 怎样变成 3 条 row 和对应 Jacobian
- Tier 3: 最后才看 Delassus / diagonal / effective-mass 直觉

### 2.6 `Contacts` 不是最终 solver 内部格式，Kamino bridge 需要点明

chapter 07 不能把 `Contacts` 讲成“solver 直接原封不动拿来解”。更准确的说法是：

- `Contacts` 是 chapter 06 的统一 handoff object
- 进入 Kamino solver 路线时，还会桥接成 `ContactsKamino` 或同等 solver-facing contact 表示
- 但教学主线仍然可以先从 `Contacts` 出发，因为这是前一章读者已经认识的对象

---

## 3. 读者最容易想错的地方

这一章要先纠正下面几类误解，再给定义：

1. `Contacts` 里的每条接触不是“一个已经解完的力”，它只是几何 handoff。
2. 一条 contact 不是一个标量约束；在刚体接触里，通常会拆成 1 条法向 row + 2 条切向 row。
3. Jacobian 不是“力矩阵”，它先是“哪些广义速度会影响这个接触方向”的映射。
4. Delassus 不是另一份质量矩阵课本；第一遍更该把它看成“系统沿这些接触方向到底有多容易被推着动”。
5. `Contacts` 里不同字段不全在同一个 frame：点常以 body-local 形式存，法线则是 world-space 几何方向。

---

## 4. 章节主目标

读完 chapter 07 后，读者至少应能更容易解释：

1. 为什么 chapter 06 的一条接触记录不会直接结束，而会继续长成约束对象。
2. 为什么一个接触点通常要拆成 normal / tangent 三条约束 row。
3. Jacobian 在接触数学里到底做什么，它和“接触点位置 + 接触 frame”有什么关系。
4. effective mass / Delassus 最朴素的一层物理直觉是什么。
5. 为什么 chapter 07 仍然是桥梁章，而不是完整 solver 推导章。

---

## 5. 非目标

1. 不做完整 LCP / NCP / KKT / ADMM 教材。
2. 不做 MuJoCo / Featherstone / Kamino solver 家族横向比较。
3. 不展开全部稳定化、bias、regularization 和数值调参分支。
4. 不写完整约束求解推导，只把读者送到 `08_rigid_solvers` 前所需的直觉层。

---

## 6. 文件级设计

### 6.1 `README.md`

职责：

- 明确 chapter 07 在 `06_collision` 和 `08_rigid_solvers` 之间的桥接位置
- 直接说清“这章不懂，会在哪卡住”
- 加一个很具体的 prereq self-check，例如：
  - 如果你还不能解释 `rigid_contact_normal`、`rigid_contact_point0 / point1` 在球贴地场景里分别代表什么，先回看 `06_collision/examples.md`
- 明确这章只讲“接触几何怎样长成约束数学”，不讲“solver 怎样解”

### 6.2 `principle.md`

建议按下面顺序组织：

1. `06 -> 07` 桥段：同一个球贴地场景，在这一章开始换一种问题意识
2. 球贴地：接触到底想阻止什么相对运动
3. 为什么一条 contact 会长成 `1 normal + 2 tangent` 三条 row
4. contact frame 和 Jacobian：它们怎样把“哪种运动会改变接触状态”写出来
5. effective mass / Delassus：先讲“沿这个方向推系统，系统有多难动”
6. 箱子贴地：为什么一个 pair 会长出多个 contact，以及离 COM 的偏移为什么会带来角向耦合
7. 明确送往 `08`

推荐的教学辅助表：

| 词 / 对象 | 先怎么想 | 球贴地时在干嘛 | 在源码里第一次看哪里 |
|-----------|-----------|----------------|-------------------------|

### 6.3 `source-walkthrough.md`

必须显式写成三层入口：

#### Tier 1: 先看 geometry handoff

- `docs/concepts/collisions.rst`
- `newton/_src/geometry/contact_data.py`
- `newton/_src/sim/collide.py`
- `newton/_src/sim/contacts.py`

目标：确认 chapter 06 最终交出来的几何到底长什么样。

#### Tier 2: 再看 constraint rows 和 Jacobian

- `newton/_src/solvers/kamino/_src/geometry/contacts.py`
- `newton/_src/solvers/kamino/_src/kinematics/constraints.py`
- `newton/_src/solvers/kamino/_src/kinematics/jacobians.py`
- `newton/_src/solvers/kamino/_src/core/math.py`

目标：确认一个 contact 怎样桥接成 solver-facing contact，再长成 3 条 row 和对应 wrench / Jacobian。

#### Tier 3: 最后看 Delassus

- `newton/_src/solvers/kamino/_src/dynamics/delassus.py`

可选补充：

- `newton/_src/solvers/kamino/_src/geometry/unified.py`
- `newton/_src/solvers/kamino/_src/dynamics/dual.py`

目标：只保住 `D = J M^{-1} J^T` 的第一层物理含义，不展开 solver 细节。

### 6.4 `examples.md`

必须明确写成两个教学例子，而不是泛泛列例子名：

1. `sphere-ground`
   - 用来验证：signed distance、normal、contact frame、`1 contact -> 3 rows`
2. `box-ground`
   - 用来验证：`1 shape pair -> multiple contacts`、接触点偏离 COM 后为什么 Jacobian 会长出角向部分

每个例子都要明确：

- 改哪里最值钱
- 预期哪个量会先变化
- 如果结果和预期不一样，说明你可能误解了什么

---

## 7. 推荐源码锚点顺序

第一遍推荐只按下面顺序读：

1. `docs/concepts/collisions.rst`
2. `newton/_src/geometry/contact_data.py`
3. `newton/_src/sim/collide.py`
4. `newton/_src/sim/contacts.py`
5. `newton/_src/solvers/kamino/_src/geometry/contacts.py`
6. `newton/_src/solvers/kamino/_src/kinematics/constraints.py`
7. `newton/_src/solvers/kamino/_src/kinematics/jacobians.py`
8. `newton/_src/solvers/kamino/_src/dynamics/delassus.py`

说明：

- `newton/_src/sim/articulation.py` 仍然重要，但这章里更像背景，不应占 first-pass walkthrough 主线。
- `newton/_src/solvers/kamino/_src/core/math.py` 应作为 Jacobian / wrench 的关键补充锚点出现。

---

## 8. 成功标准

如果 chapter 07 写完后达成下面状态，就说明它是成功的：

1. 读者第一次读时，不会觉得“我知道 contact 是什么，但完全不知道为什么突然冒出 Jacobian”。
2. 读者能把球贴地这张图，顺着讲到 normal row、tangent rows 和 contact frame。
3. 读者能用人话描述 Delassus / effective mass 的第一层意义，而不是只会复述公式。
4. 读者读 walkthrough 时，知道自己先在找哪条桥，而不是被文件列表压住。

---

## 9. 预期改动文件

- `chapters/07_constraints_contacts_math/README.md`
- `chapters/07_constraints_contacts_math/principle.md`（新增）
- `chapters/07_constraints_contacts_math/source-walkthrough.md`（新增）
- `chapters/07_constraints_contacts_math/examples.md`（新增）
- `docs/superpowers/specs/2026-04-19-chapter-07-deepening-design.md`
