# Chapters 02 03 07 Walkthrough Refinement Design

- **日期**: 2026-04-20
- **仓库**: `/home/zhuzihou/dev/newton-learn`
- **范围**:
  - `chapters/02_newton_arch/source-walkthrough.md`
  - `chapters/03_math_geometry/source-walkthrough.md`
  - `chapters/07_constraints_contacts_math/source-walkthrough.md`
- **允许的配套对齐**:
  - 如果 chapter package 因主 walkthrough 重写而出现轻量概念不一致，允许对对应 chapter 的 `README.md`、`principle.md`、`examples.md` 做最小一致性修正。
- **目标**: 沿 chapter 01 的新手标准，继续修三个最关键的主 walkthrough 断点，让读者单看主 walkthrough 就更容易把来龙去脉讲顺。

---

## 1. 为什么是这三章

在对 `02-09` 的主 walkthrough 做完一轮全仓审查后，三路结论收敛成两个事实：

1. `02_newton_arch` 是 chapter 01 的直接下一跳，如果它不够顺，读者刚建立的 Warp 心智模型会立刻断掉。
2. `03_math_geometry` 和 `07_constraints_contacts_math` 是当前最明显的“对新手不够友好”的高风险章节。

所以这轮 refinement 不做全仓齐改，而是优先抓：

```text
02 -> 03 -> 07
```

这三个章节分别代表三种不同的问题：

- `02`：承接不足，架构词出现太快
- `03`：术语堆叠过重，没有单一故事线
- `07`：数学对象出现太快，缺“为什么 solver 需要这些东西”的物理来龙去脉

---

## 2. 当前问题摘要

### 2.1 `02_newton_arch`

当前问题不是结构坏掉，而是：

- 太快进入 `Model / State / Control / Contacts / Solver`
- 主 walkthrough 仍然轻度依赖 `principle.md` 提供前置架构解释
- 缺一条足够朴素的“从用户运行 example，到 Newton 内部对象真的开始接起来”的主故事线

### 2.2 `03_math_geometry`

这是当前最像“概念清单”而不是“主 walkthrough”的一章。

问题表现为：

- 一开头就叠 `frame / transform / spatial / shape / inertia`
- 概念顺序更像知识点 inventory，而不是学习顺序
- 新手不知道“为什么我在这里 suddenly 要学这些数学词”

### 2.3 `07_constraints_contacts_math`

这是当前最容易劝退新手的数学章。

问题表现为：

- `solver-facing contact / rows / Jacobian / Delassus` 很快一起出现
- 缺一个更朴素的桥：
  - “为什么光有接触点和法线还不够？”
  - “为什么 solver 还要把接触继续变成 rows？”

---

## 3. 这一轮的统一标准

三章都要继续遵守 walkthrough rollout 之后的新结构：

- `What This Walkthrough Follows`
- `One-Screen Chapter Map`
- `Beginner Path`
- `Main Walkthrough`
- `Object Ledger`
- `Stop Here`
- `Go Deeper`

但本轮 refinement 的重点不是结构，而是**让主 walkthrough 真正有“来龙去脉”**。

统一要求：

1. 主 walkthrough 单读应更自包含
2. 不依赖 `principle.md` 才能讲顺主问题
3. 必须先给一个新手自然会问的问题，再展开源码
4. 每章都要有一条贯穿全篇的单一故事线

---

## 4. 每章的推荐重写策略

### 4.1 Chapter 02: 从“用户运行 example”开始，而不是从对象名开始

推荐故事线：

```text
我运行一个 Newton example
-> launcher 到底把我带到哪里
-> Example 怎么拿到 Model / State / Control / Contacts / Solver
-> simulate loop 为什么总长一个样
```

也就是说，chapter 02 要先从“用户视角的入口”出发，再进入对象分层。

不推荐一开始就让读者面对：

- public re-export
- CLI routing
- `Model / State / Control / Contacts / Solver` 名词束

更稳的顺序是：

1. 先讲“为什么 Newton 的例子不是一个脚本到底”
2. 再讲 Example / launcher 的分层
3. 再把 `Model / State / Control / Contacts / Solver` 放回那条入口故事里

### 4.2 Chapter 03: 从“为什么后面章节老出现这些坐标和量”开始

推荐故事线：

```text
为什么 04/05/06 里总在出现 transform、frame、body_q、shape_transform、body_qd、inertia？
-> 因为 Newton 一直在用局部关系描述世界
-> 这些局部关系怎样变成空间速度 / 空间力 / 几何查询 / 惯量
```

也就是说，chapter 03 不该像数学百科，而应该像：

- “后面所有章节反复会用到的图例说明书”

推荐把主线收成：

1. local frame / transform 为何无处不在
2. spatial quantity 为什么是 6D
3. `GeoType` / shape representation 为什么不是多余细节
4. inertia 为什么是几何走向动力学的桥

### 4.3 Chapter 07: 从“为什么 contact 还不够”开始

推荐故事线：

```text
chapter 06 已经给了接触点和法线
-> 为什么 solver 还不能直接停在这里
-> 它还需要把“该阻止什么运动”写成 rows
-> rows 怎样长成 Jacobian
-> Jacobian 为什么又会长成 Delassus
```

也就是说，chapter 07 需要在数学词出现前，先让读者相信：

- 接触几何不是最终求解形式
- solver 需要的是“沿哪些方向限制速度 / 运动”

只有这样，`rows / Jacobian / Delassus` 才不会像凭空冒出来。

---

## 5. 统一不该做什么

这轮 refinement 不做下面这些事：

1. 不重写 deep walkthrough
2. 不改 chapter 的核心技术结论
3. 不把主 walkthrough 重新变回 line-range-heavy 文档
4. 不一次性处理 `04/05/06/08/09`

---

## 6. 成功标准

如果这轮做对了，应该出现下面结果：

1. `02` 读者会先理解“入口和对象为什么这样分层”，而不是只记住五个对象名。
2. `03` 读者会把它看成后续章节的图例说明书，而不是术语词典。
3. `07` 读者会先懂“为什么 contact 还不够”，再接受 rows / Jacobian / Delassus。
4. 三章都更接近 chapter 01 refined 之后的那种：
   - 先交代问题
   - 再给概念
   - 再看源码

---

## 7. 推荐执行顺序

这轮实现建议按下面顺序推进：

1. `02_newton_arch`
2. `03_math_geometry`
3. `07_constraints_contacts_math`

原因：

- `02` 最靠前，先修它能立刻改善主线连续性
- `03` 和 `07` 是当前最重的两个理解断点
