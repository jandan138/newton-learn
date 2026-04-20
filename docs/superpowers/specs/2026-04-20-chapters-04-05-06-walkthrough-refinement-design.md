# Chapters 04 05 06 Walkthrough Refinement Design

- **日期**: 2026-04-20
- **仓库**: `/home/zhuzihou/dev/newton-learn`
- **范围**:
  - `chapters/04_scene_usd/source-walkthrough.md`
  - `chapters/05_rigid_articulation/source-walkthrough.md`
  - `chapters/06_collision/source-walkthrough.md`
- **目标**: 在 chapter 01 refined 标准下，把 `04/05/06` 的主 walkthrough 再降一层门槛，补齐它们各自还不够清楚的来龙去脉。

---

## 1. 为什么还要补这三章

上一轮全仓审查里，这三章没有 `03` / `07` 那么高风险，但都还保留了不同程度的“对新手仍偏快”的问题：

- `04`：主线本身清楚，但一开头就进入 importer / resolver / authored attrs 术语。
- `05`：布局、slice、FK、Featherstone 都对，但对第一次接触 articulation 的读者仍然偏 source-first。
- `06`：shape-centric 主线已经不错，但 broad phase / narrow phase / margin / gap 的前置动机还可以更白话。

既然用户明确要求把 `02-07` 这段主 walkthrough **整体** 修到更像新手主读版，这三章也要一起补齐。

---

## 2. 每章还剩什么问题

### 2.1 Chapter 04

当前主要问题：

1. 开头仍然轻度依赖 `principle.md`
2. `schema resolver`、`authored attrs`、`MassAPI` 这些词出现得偏快
3. 缺一个更朴素的入口问题：

```text
我在 USD 里写了一个 body / joint / shape，Newton 到底是怎样把它变成 Model 里的数组？
```

### 2.2 Chapter 05

当前主要问题：

1. articulation 一上来还是更像 slice 结构讲义，而不是从“最小两连杆”出发
2. `joint_q_start / articulation_start` 比“这条树到底在表达什么”更先出现
3. FK / body-space state 的物理故事仍然可以更先于数组名出现

### 2.3 Chapter 06

当前主要问题：

1. `broad phase`、`narrow phase`、`candidate pairs`、`ContactData` 这些词出现得还偏快
2. `margin / gap` 的动机更像在后文推出来，而不是在前文提前交代
3. 对第一次读碰撞的人来说，还可以更明确回答：

```text
为什么不是 body 直接互相碰撞？为什么还要先把 shape 放到 world，再先找 candidate pair？
```

---

## 3. 统一标准

三章都继续保持 walkthrough rollout 后的主结构不变：

- `What This Walkthrough Follows`
- `One-Screen Chapter Map`
- `Beginner Path`
- `Main Walkthrough`
- `Object Ledger`
- `Stop Here`
- `Go Deeper`

本轮重点依旧不是结构，而是：

1. 主 walkthrough 更自包含
2. 一开头先回答一个新手自然会问的问题
3. 先讲“为什么会有这条链”，再讲源码

---

## 4. 每章推荐重写策略

### 4.1 Chapter 04

推荐故事线：

```text
我在 USD / 场景文件里写了东西
-> importer 先要认出 body / joint / shape / mass 这些逻辑对象
-> schema resolver 负责把不同 authoring 命名归一
-> builder 累积这些逻辑对象
-> finalize() 才把它冻结成 Model arrays
```

关键不是删术语，而是让术语回答问题。

### 4.2 Chapter 05

推荐故事线：

```text
先想两根连杆和一个关节
-> 这条树怎样被记成 flat layout
-> joint-space 状态怎样经由 FK 变成 body-space 状态
-> Featherstone 只是继续消费这条 handoff
```

关键不是少讲 slice，而是把 slice 放在“为什么需要它”之后。

### 4.3 Chapter 06

推荐故事线：

```text
body 只提供运动
-> 真正碰撞的是挂在 body 上的 shape
-> shape 先变成 world-space query object
-> broad phase 只负责便宜地筛可能相撞
-> narrow phase 才把它们变成 contact geometry
-> writer 再写进 Contacts
```

关键不是多讲算法，而是把 broad / narrow / writer 三段职责先讲透。

---

## 5. 成功标准

如果这轮做对了，应该出现下面结果：

1. `04` 读者会先理解 importer / resolver / builder 为何存在，而不是只记住名字。
2. `05` 读者会先从“最小 articulation 在表达什么”入手，再接受 flat slices 和 FK。
3. `06` 读者会先理解“shape 为什么是碰撞的真正主体”，再接受 broad / narrow / writer 分工。
