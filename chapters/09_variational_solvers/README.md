---
chapter: 09
title: 09 变分求解器族
last_updated: 2026-04-19
source_paths: []
paper_keys:
  - xpbd-paper
  - vbd-paper
  - style3d-paper
newton_commit: 1a230702
---

# 09 变分求解器族

`08_rigid_solvers` 已经把 shared `solver.step(...)` contract 和 solver family split 讲顺了。第 09 章接着回答的不是“又多了哪几个 solver”，而是另一件更关键的事: **当同一块 cloth 或 softbody 需要更稳定的隐式更新时，`XPBD`、`VBD`、`Style3D` 到底在解什么共同问题？它们每一步又把修正落在哪一层？**

所以这章不是 flat solver catalog。它先用同一个 `example_cloth_hanging.py` 把 shared problem 立住，再把三条主路线压成一条 beginner-safe 的梯子:

```text
XPBD: per-constraint projection
-> VBD: per-vertex / per-block local solve
-> Style3D: cloth-specialized global PD / PCG solve
```

顺手先说清一个边界: `example_cloth_hanging.py` 里也会出现 `SemiImplicit`。它在这章里的角色只是 baseline 对照，帮你看清“为什么要走隐式修正”；本章真正的主角是 `XPBD / VBD / Style3D`。

## 文件分工

- `README.md`: 本章边界、完成门槛、阅读入口。
- `principle.md`: 先讲 shared variational / implicit correction problem，再比较三条更新路线各自把修正落在哪一层。
- `source-walkthrough.md`: 新手 / 主 walkthrough。第一次追 chapter 09 源码先看这一份；它内嵌关键源码片段，把 `shared variational problem -> XPBD -> VBD -> Style3D` 主线直接讲顺。
- `source-walkthrough-deep.md`: 深读锚点版。已经跟上主线后，如果你想精确追 symbol、上游路径和可选分支，再看这一份。
- `examples.md`: 用 `example_cloth_hanging.py` 做跨 solver 对照，再用 `example_softbody_hanging.py` 解释 VBD 为什么不只是 cloth solver。

## 完成门槛

```text
[ ] 我能先说清 `08 -> 09` 的桥: 同样是 `solver.step(...)`，但现在关注的是 cloth / softbody 的隐式修正问题
[ ] 我能解释为什么 `XPBD`、`VBD`、`Style3D` 可以被放在同一章里比较，而不是三个孤立名字
[ ] 我能说明三条主路线的局部更新层级差别: `constraint -> vertex/block -> global linear system`
[ ] 我能解释为什么 `example_cloth_hanging.py` 是本章主例子，而 `example_softbody_hanging.py` 只是 VBD 的补充延伸
```

建议读完后直接去 `examples.md` 做自检，不要只停在术语表面。

## 本章目标

- 先把 shared variational / implicit correction problem 讲清，再谈 solver family split，而不是先背 solver 名字。
- 让你用同一个 `cloth_hanging` 场景比较三家，看到它们面对的是同一类稳定更新问题，只是每步修正的组织方式不同。
- 让你建立一条稳定的比较轴: `每次迭代最先修哪层对象`，而不是掉进 feature matrix 或性能排行。

## 本章范围

- 只聚焦 chapter 09 的 main trio: `SolverXPBD`、`SolverVBD`、`SolverStyle3D`。
- `SemiImplicit` 只在共享例子里作为 contrast baseline 出现，不和 main trio 平分章节重心。
- 主例子只有 `newton/examples/cloth/example_cloth_hanging.py`，辅例子只有 `newton/examples/softbody/example_softbody_hanging.py`。
- 重点比较 shared example、solver setup、局部更新层级，以及这些差别怎样映射到 Newton 源码。

## 本章明确不做什么

- 不做完整 XPBD 合规性推导、VBD 证明或 projective dynamics 理论课。
- 不展开 VBD 的 AVBD 刚体细节；chapter 09 只把它当背景，主线仍是 cloth / softbody 粒子求解。
- 不做 diffsim、梯度路径或“哪个 solver 最快”的性能排行。
- 不把 chapter 08 的 rigid solver family 和 chapter 09 的 variational solver family 混成一个更长的 catalog。

## 前置依赖

- 建议先读完 `08_rigid_solvers`。如果你还不能解释同一个 `solver.step(...)` 为什么会分叉成不同 solver family，先回看 chapter 08。
- 默认你已经接受一点最小直觉: 显式积分会先把状态往前推，隐式 / 变分 solver 会多做一轮“把这一步修正稳”的工作。
- 不要求你先会完整 XPBD、VBD 或 PD 推导；这章只要求你愿意先用“修正落在哪一层”来比较 solver。

## 教学型对照表

| Solver / 路线 | 先怎么想 | 每次迭代最先修哪层 | 为什么现在要懂 |
|---------------|-----------|--------------------|----------------|
| `SemiImplicit` | 力累加 + 多 substeps baseline | 不先解隐式修正，主要是先算力再积分 | 帮你看清 chapter 09 为什么需要 main trio |
| `XPBD` | 逐约束投影 | 单条约束 / 单次投影 | 最容易上手，帮你建立“隐式修正不一定先建全局矩阵” |
| `VBD` | local block Newton solve | 单个 vertex / block | 帮你看懂 force + Hessian 为什么能先在局部累计再解 |
| `Style3D` | global cloth PD / PCG solve | 整张 cloth 的线性系统 | 帮你看懂 cloth-specialized 的全局求解路线 |

## 阅读顺序

1. 第一次追源码，先看 `source-walkthrough.md`；这一份就是给 first pass 准备的主 walkthrough。
2. 如果你想先补概念边界，或者读完主 walkthrough 还想把那条更新梯子再翻成人话，再回看 `principle.md`。
3. 然后读 `examples.md`，先用 `cloth_hanging` 观察 shared scene 下的 solver 差异，再用 `softbody_hanging` 看 VBD 的延伸。
4. 想精确追到上游文件、symbol 和行号，再看 `source-walkthrough-deep.md`。
5. 如果你读完还会把三家想成“只是 solver 名字不同”，就回到 `principle.md` 里的梯子表重新看一遍。

## 预期产出

- `principle.md`: 一条 beginner-safe 主线，先讲 shared implicit correction，再讲 `XPBD -> VBD -> Style3D`。
- `source-walkthrough.md`: 给 first pass 的主 walkthrough，用内嵌源码片段把 `shared variational problem -> XPBD -> VBD -> Style3D` 主线直接讲顺。
- `source-walkthrough-deep.md`: 保留精确 symbol、路径、行号和可选分支，给已经跟上主线后还想继续追锚点的读者。
- `examples.md`: 两个观察任务，分别守住“同一块 hanging cloth 的跨 solver 对照”和“VBD 不只处理 cloth”这两层理解。
