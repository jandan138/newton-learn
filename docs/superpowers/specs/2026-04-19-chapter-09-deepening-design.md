# Chapter 09 Deepening Design

- **日期**: 2026-04-19
- **仓库**: `/home/zhuzihou/dev/newton-learn`
- **范围**: `chapters/09_variational_solvers`
- **目标**: 把 `09_variational_solvers` 写成一章 teaching-first 的“同一个隐式 / 变分问题怎样被 XPBD、VBD、Style3D 这三条路线分别处理”桥梁章，而不是一篇 solver 名字目录。

---

## 1. 背景

主线已经走到：

- `08_rigid_solvers` 讲清了 shared `solver.step(...)` contract 和 solver family split
- 接下来教程自然不该继续围着 rigid solver 转，而应该切到另一类更偏 cloth / softbody 的隐式求解路线

所以 chapter 09 的自然任务不是回答“还有哪些 solver”，而是回答：

**当布料、软体或粒子系统要做稳定的隐式更新时，XPBD、VBD、Style3D 到底在解什么共同问题？它们每一步又是怎样组织修正的？**

---

## 2. 设计判断

### 2.1 不能写成 flat catalog

如果 chapter 09 平铺成：

- `XPBD`
- `VBD`
- `Style3D`

然后每个 solver 各写一节，同等篇幅介绍一遍，教学风险很高：

1. 读者只会记住三个名字
2. 看不出它们其实在面对同一个隐式问题
3. 也看不出它们真正的分野其实是“每次迭代到底在局部求什么”

所以 chapter 09 不能是 catalog，它必须先建立 shared problem。

### 2.2 最稳的写法是“shared variational problem first, local-update split second”

推荐主结构：

1. 先讲 shared `step(...)` surface 和共同的隐式修正目标
2. 再告诉读者：三条 solver 家族的真正差别，在于每次迭代把修正落在哪一层
3. 最后再映射到具体实现

### 2.3 最适合 beginner 的比较轴是“隐式修正到底落在哪一层”

chapter 09 最值钱的比较轴不是 feature matrix，也不是 cloth/softbody/rigid 的支持表。

更该优先比较：

1. `XPBD`：per-constraint projection
2. `VBD`：per-vertex / per-body block solve
3. `Style3D`：global cloth linear solve

也就是说，这章最该让读者建立的梯子是：

```text
constraint projection
-> local block Newton solve
-> global PD/PCG solve
```

### 2.4 主例子必须是 `example_cloth_hanging.py`

原因：

1. 一个例子里已经并列支持 `semi_implicit / style3d / xpbd / vbd`
2. 同一个 hanging cloth 场景最容易保持 shared problem 不变
3. 可以把 `SemiImplicit` 当作 chapter 08 的对照基线，而把 `XPBD / VBD / Style3D` 作为 chapter 09 的主角

### 2.5 辅例子建议用 `example_softbody_hanging.py`

作用不是横向比较三家，而是专门补一句：

- `VBD` 在 Newton 里不只是 cloth solver，它还可以继续走到 volumetric softbody

这能帮助读者理解 `VBD` 不是“更复杂的 XPBD for cloth”，而是一条更一般的局部 Hessian / block descent 路线。

### 2.6 这章必须和 chapter 08 保持相似的外层结构，但不要把 rigid / variational 混成一个 family

chapter 08 已经建立了很好的外层阅读节奏：

- shared contract first
- family split second
- canonical path and contrasts

chapter 09 应延续这种教学节奏，但要明确：

- `SemiImplicit` 在 `cloth_hanging` 里只是 baseline 对照，不属于本章 variational trio 的主角

---

## 3. chapter 09 应该解决什么

读完 chapter 09 后，读者至少应更容易解释：

1. 为什么 `XPBD`、`VBD`、`Style3D` 都可以被看成在处理同一类隐式更新问题。
2. `XPBD` 为什么更像逐约束投影，而不是显式构建 Hessian 块。
3. `VBD` 为什么更像每个 vertex / block 的局部 Newton 步，而不是简单约束投影。
4. `Style3D` 为什么更像 cloth-specialized 的 global PD/PCG 路线。
5. 为什么 `cloth_hanging` 适合作为共享例子，而 `softbody_hanging` 更适合作为 VBD 的补充例子。

---

## 4. chapter 09 不该解决什么

1. 不做完整 projective dynamics、XPBD 合规性推导或 VBD 理论证明。
2. 不展开 AVBD 刚体扩展的完整细节；那会把章节重新拖回 rigid 方向。
3. 不做 diffsim / gradient path 深挖。
4. 不写“哪个 solver 最好”的性能排行。
5. 不把 chapter 08 的 rigid solver family 和 chapter 09 的 variational solver family 混成一个大一统分类。

---

## 5. approaches 与取舍

### Approach A: Flat catalog

- 做法：XPBD / VBD / Style3D 逐个介绍
- 优点：表面直观
- 缺点：最容易让读者只记名字，不记共享问题和分层梯子

### Approach B: Shared problem first, one canonical path, then compare others

- 做法：先讲共同隐式问题，再深挖一个 solver，另外两家做对照
- 优点：最 teaching-first
- 缺点：如果 canonical path 选得不好，容易把其他 solver 写成附录

### Approach C: Local-update family split first

- 做法：直接按 projection / block solve / global solve 三种更新风格讲
- 优点：比较轴非常清楚
- 缺点：如果跳过 shared problem，读者会不知道它们为什么要被放在同一章

### 推荐: B with C framing

也就是：

1. shared variational problem first
2. `XPBD` 作为最容易上手的 canonical entry
3. `VBD` 作为 richer local Newton / block-descent 路线
4. `Style3D` 作为 global cloth PD/PCG 路线

---

## 6. 文件级设计

### 6.1 `README.md`

职责：

- 明确 chapter 09 在 chapter 08 之后的位置
- 强调这章的核心问题不是“又有三个 solver”，而是“同一个隐式问题怎样被三种更新组织方式处理”
- 给出 prereq self-check：
  - 如果你还不能解释 shared `step(...)` contract，先回看 chapter 08

### 6.2 `principle.md`

建议结构：

1. `08 -> 09` 桥段：同样 `step(...)`，但现在关心的是 variational / implicit correction
2. shared problem：从 hanging cloth 这张图开始，先讲“大家都在试图做稳定的隐式更新”
3. `XPBD`：constraint projection
4. `VBD`：per-vertex / per-body block solve
5. `Style3D`：global cloth linear solve / PD matrix / PCG
6. 用一张教学表把三条路线压在同一比较轴上

### 6.3 `source-walkthrough.md`

必须分层：

#### Tier 1: shared surface and shared example

- `newton/solvers.py`
- `newton/examples/cloth/example_cloth_hanging.py`

#### Tier 2: three variational paths

- `newton/_src/solvers/xpbd/solver_xpbd.py`
- `newton/_src/solvers/xpbd/kernels.py`
- `newton/_src/solvers/vbd/solver_vbd.py`
- `newton/_src/solvers/vbd/particle_vbd_kernels.py`
- `newton/_src/solvers/style3d/solver_style3d.py`
- `newton/_src/solvers/style3d/linear_solver.py`

#### Tier 3: VBD beyond cloth

- `newton/examples/softbody/example_softbody_hanging.py`

### 6.4 `examples.md`

建议两条主例子：

1. `newton/examples/cloth/example_cloth_hanging.py`
   - 主例子，负责 cross-solver 对照
2. `newton/examples/softbody/example_softbody_hanging.py`
   - 辅例子，负责解释 VBD 不只是 cloth solver

---

## 7. 关键源码锚点

第一遍推荐顺序：

1. `newton/solvers.py`
2. `newton/examples/cloth/example_cloth_hanging.py`
3. `newton/_src/solvers/xpbd/solver_xpbd.py`
4. `newton/_src/solvers/xpbd/kernels.py`
5. `newton/_src/solvers/vbd/solver_vbd.py`
6. `newton/_src/solvers/vbd/particle_vbd_kernels.py`
7. `newton/_src/solvers/style3d/solver_style3d.py`
8. `newton/_src/solvers/style3d/linear_solver.py`
9. `newton/examples/softbody/example_softbody_hanging.py`

---

## 8. 教学风险

chapter 09 写作时必须主动避免：

1. 把三家写成 solver catalog，而看不见 shared variational problem。
2. 把 `XPBD` 误写成“只是简化版 PBD”，而忽略 compliance / implicit flavor。
3. 把 `VBD` 误写成“更复杂的 XPBD”，而看不见它的 force + Hessian / block solve 逻辑。
4. 把 `Style3D` 误写成“只针对衣服的 VBD”，而看不见 global PD matrix / PCG 这条路。
5. 太早掉进 AVBD、differentiability 或性能排行，打断主线。

---

## 9. 成功标准

如果 chapter 09 写完后达成下面状态，就说明设计是成功的：

1. 读者不会再把 XPBD、VBD、Style3D 只当成三个名字。
2. 读者能先用一句话说清它们各自的局部更新层级。
3. 读者能用 `cloth_hanging` 这一个场景去比较三家，而不是必须换三个完全不同的例子。
4. 读者能理解为什么 `softbody_hanging` 是 VBD 的补充，而不是整个 chapter 09 的主入口。

---

## 10. 预期改动文件

- `chapters/09_variational_solvers/README.md`
- `chapters/09_variational_solvers/principle.md`（新增）
- `chapters/09_variational_solvers/source-walkthrough.md`（新增）
- `chapters/09_variational_solvers/examples.md`（新增）
- `docs/superpowers/specs/2026-04-19-chapter-09-deepening-design.md`
