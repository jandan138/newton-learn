# Chapters 08 09 Walkthrough Refinement And Chapter 10 Design

- **日期**: 2026-04-20
- **仓库**: `/home/zhuzihou/dev/newton-learn`
- **范围**:
  - `chapters/08_rigid_solvers/source-walkthrough.md`
  - `chapters/09_variational_solvers/source-walkthrough.md`
  - `chapters/10_softbody_cloth_cable/README.md`
  - `chapters/10_softbody_cloth_cable/principle.md`
  - `chapters/10_softbody_cloth_cable/examples.md`
  - `chapters/10_softbody_cloth_cable/source-walkthrough.md`
- **目标**: 按 chapter 01 refined 标准把 `08/09` 的主 walkthrough 再降一层门槛，并补出 chapter 10 的第一版 teaching-first 主线，让 `09 -> 10` 形成自然 handoff。

---

## 1. 为什么这轮要一起做 08 09 10

上一轮 walkthrough redesign 和 `01-07` refinement 已经把全仓主读版的结构问题基本解决掉，但 `08/09` 还保留着同一类残余风险：

- 结构已经对了。
- 源码路径也基本对了。
- 但对真正第一次追 Newton solver 的读者来说，还是有点偏快。

与此同时，chapter 10 还只有 skeleton `README`，没有形成主线内容。这会让读者在读完 `09` 以后，直接掉进一个空章节，而不是自然进入“deformable object family”这条下一跳。

所以这轮最自然的组合任务是：

1. 把 `08/09` 的主 walkthrough 修到和 `01-07` 同一条 beginner 标准线上。
2. 立刻补出 chapter 10 的 teaching-first 主线，把 cloth / softbody / cable 接成下一条完整故事。

---

## 2. 08 和 09 还差在哪里

三轮 reviewer 的结论很一致：

- 问题不是源码 coverage 不够。
- 问题是 pedagogy distance 还没有完全降下来。

### 2.1 Chapter 08

当前主要问题：

1. 开头还是从 `shared contract / family split / Kamino continuation` 这样的抽象标签开始。
2. 虽然主线已经不是 flat catalog，但第一次读的人仍然容易把四家看成“四张 solver 卡片”。
3. `joint-space`、`backend bridge`、`DualProblem`、`PADMMSolver` 这些词出现时，还没有先被翻译成 beginner-safe 的问题语言。
4. chapter 07 的 handoff 虽然存在，但 recap 还不够短平快。

更准确地说，chapter 08 现在缺的不是更多知识点，而是一个更朴素的开场问题：

```text
我只改了一行 self.solver = ...，为什么同一个 step(...) 后面就像换了一套世界？
```

### 2.2 Chapter 09

当前主要问题：

1. 开头虽然已经聚焦 shared cloth problem，但 `variational` 这个词出现得还是比 beginner 问题更快。
2. `lambda`、`Hessian`、`PD / PCG` 这些关键术语还没有在第一次出现时做足够白话的落地解释。
3. 三条 solver 路线的真正对照轴其实是“每轮迭代里先反复更新哪个容器”，但这个锚点还没有被抬到更前面。
4. `softbody_hanging` 已经在 chapter 09 里露面，但它的角色需要更明确地被收束成“chapter 10 的前置桥”。

chapter 09 现在最需要的开场问题应该更具体：

```text
同一块 hanging cloth、同一个 collide -> step -> swap 外层 loop，solver 到底先修哪一层：一条约束、一个局部块，还是整张布？
```

---

## 3. 这轮的统一标准

`08/09` 继续保持 walkthrough rollout 后的主结构不变：

- `What This Walkthrough Follows`
- `One-Screen Chapter Map`
- `Beginner Path`
- `Main Walkthrough`
- `Object Ledger`
- `Stop Here`
- `Go Deeper`

但和 `01-07` refinement 一样，本轮重点不是结构，而是下面四点：

1. 一开头先回答一个新手自然会问的问题。
2. 整篇只守住一条单一故事线，不再像隐形 catalog。
3. 第一次出现关键术语时就给 beginner-safe 定义。
4. 每个阶段都有 checkpoint 气质，让读者知道自己现在应该已经会说什么。

---

## 4. 08 和 09 的推荐重写策略

### 4.1 Chapter 08

推荐故事线：

```text
same public step(...)
-> 但不同 solver 会先抓不同的第一主角
-> SemiImplicit 先抓 forces
-> Featherstone 先抓 joint-space articulation quantities
-> MuJoCo 先抓 backend handoff objects
-> Kamino 先抓 contact / constraint solve objects
```

这条线最重要的改动不是“多写哪家 solver”，而是把 family split 改写成：

```text
同一外层动作
-> 不同 solver 先处理不同对象
-> 所以内部数学自然分叉
```

更具体地说：

- 开头先补一句最朴素的 user-side 场景：我只切了一个 solver 构造器。
- 在 Stage 1 前后补齐 `public contract`、`force route`、`joint-space route`、`backend bridge` 的简定义。
- 给 chapter 07 加一个极短 recap：`Contacts -> rows -> Jacobians -> Delassus` 是什么主线，Kamino 为什么刚好接住它。
- 每条 family 末尾补一句 checkpoint：我现在能不能说出“这家 solver 先处理谁”。

### 4.2 Chapter 09

推荐故事线：

```text
same hanging cloth loop
-> same “先预测、再修正”问题
-> XPBD 先反复修单条约束
-> VBD 先反复解一个 vertex / block 的局部子问题
-> Style3D 先反复解整张 cloth 的全局线性子问题
```

这条线真正要抬高的是“每轮先更新哪个容器”这根比较轴：

- XPBD: `particle_deltas` / `lambdas`
- VBD: `particle_forces` / `particle_hessians` / `particle_displacements`
- Style3D: `rhs` / `dx` / fixed PD matrix

更具体地说：

- 开头先把 `variational` 翻译成人话：不是凭感觉改位置，而是在做“先预测、再修正”的稳定更新。
- 把 `predicted state`、`inertia target`、`lambda`、`Hessian`、`PCG` 的第一次解释提前。
- 更明确地交代 chapter 09 的 clean mainline 仍然是 cloth；`softbody_hanging` 是 VBD 的延伸，也正好为 chapter 10 做桥。
- 每一条 solver path 末尾补一句 checkpoint：我现在能不能说出“每轮反复改的是哪个容器”。

---

## 5. Chapter 10 应该怎样起主线

reviewer 的共识是：chapter 10 最稳的写法不是 solver-first，而是 representation-first。

最值得先回答的问题是：

```text
cloth、softbody、cable 看起来都在“会弯会挂会掉”，但在 Newton 里它们真的是同一种对象吗？
```

推荐主线：

```text
同样都像“可变形物”
-> 但 builder 会把它们展开成完全不同的内部对象图
-> cloth = particles + triangles + bending edges / optional springs
-> softbody = particles + tetrahedra + surface collision mesh
-> cable = capsule rigid bodies + cable joints (+ articulation)
```

这条设计的价值是：

1. 它自然接在 chapter 09 后面，但不重复 chapter 09 的 solver math。
2. 它能直接回答“为什么 softbody / cloth / cable 不该写成一个 catalog”。
3. 它把读者真正会碰到的 builder helper 变成主角，而不是把 solver internals 再讲一遍。

### 5.1 推荐分工

- `README.md`
  - 建立章节边界、阅读顺序和完成门槛。
  - 明确 chapter 10 是“对象表示”章节，不是“solver internals 补课”。
- `principle.md`
  - 先讲“同样都像可变形物，为何内部表示不同”。
  - 再讲 cloth / softbody / cable 各自的 builder-side object graph 和 solver consequence。
- `examples.md`
  - 用 `cloth_hanging`、`softbody_hanging`、`cable_twist` 形成三个观察锚点。
  - 强调三个例子的教学任务不同，不做性能 PK。
- `source-walkthrough.md`
  - 追一条 builder-first 主线：`choose helper -> expand into internal objects -> see which state fields and solver route now matter`。

### 5.2 推荐范围

主例子和主源码锚点建议是：

- `newton/examples/cloth/example_cloth_hanging.py`
- `newton/examples/softbody/example_softbody_hanging.py`
- `newton/examples/cable/example_cable_twist.py`
- `newton/_src/sim/builder.py`
  - `add_cloth_grid(...)`
  - `add_soft_grid(...)`
  - `add_rod(...)`

### 5.3 这章明确先不做什么

- 不重讲 chapter 09 的 XPBD / VBD / Style3D 数学细节。
- 不展开 `add_cloth_mesh(...)`、`add_soft_mesh(...)`、Style3D custom attributes 的更深分支。
- 不把 cable 的 twist quaternion 构造或 closed-loop rod graph 讲成主线。
- 不在这一章回答“哪个 solver 最快”“参数怎么调最好”。

---

## 6. 成功标准

如果这轮做对了，应该出现下面结果：

1. `08` 读者会先回答“同一个 `step(...)` 后面为什么像换了一套世界”，再去接 family split。
2. `09` 读者会先回答“每轮先修哪一层对象”，而不是只背 `XPBD / VBD / Style3D` 名字。
3. `10` 读者会先明白 cloth / softbody / cable 在 Newton 里不是同一种内部对象，再去接受各自的 builder helper 和 solver consequence。
4. 整条阅读体验会变成：

```text
07: solver-facing contact math
-> 08: different rigid solvers consume different first-class objects
-> 09: different variational solvers repair the same cloth step at different levels
-> 10: different deformable object families are not even built out of the same internal representation
```
