---
chapter: 03
title: 03 数学与几何基础
last_updated: 2026-04-22
source_paths: []
paper_keys: []
newton_commit: 1a230702
---

# 03 数学与几何基础

`02_newton_arch` 已经回答了 Newton 是怎样组织起来的：examples 入口怎样接到 `Model / State / Control / Solver`。这一章接着回答另一个更窄、但马上会卡住你的问题：当你在 `Model` 里看到 body / joint / shape 的局部关系、`wp.transform`、四元数、spatial quantities 和惯量时，这些几何与空间量词到底在说什么。

如果 `02_newton_arch` 让你第一次接受了 `Model / State / Control / Solver` 这条架构链，那么 chapter 03 的工作就是把这条链背后的 geometry / spatial vocabulary 补稳。它不是完整数学课，而是把 `Model` 先变得可读，再把你送去 `04_scene_usd`、`05_rigid_articulation` 和 `06_collision`。

## 文件分工

- `README.md`：只负责本章边界、完成门槛和阅读入口。
- `principle.md`：先把 frame、transform、quaternion、spatial quantity、shape、inertia 这些词翻译成人话。
- `source-walkthrough.md`：新手 / 主 walkthrough。第一次追 chapter 03 源码先看这一份；它把 `frame -> transform -> spatial -> shape -> inertia` 主线直接讲顺。
- `source-walkthrough-deep.md`：深读锚点版。已经跟上主线后，如果你想精确追上游文件、symbol 和行号，再看这一份。

## 完成门槛

```text
[ ] 我能把 `wp.transform` 先翻译成“平移 + 旋转”，并解释 parent / child frame 为什么会让 builder 代码写成层层组合
[ ] 我能说明 `body`、`joint`、`shape` 为什么都要有各自的局部 frame，而不是一开始全写成 world 坐标
[ ] 我能区分 `primitive / mesh / SDF / heightfield` 在第一遍阅读里的不同角色
[ ] 我能把 twist / wrench / `velocity_at_point` 先理解成“挂在某个 frame 上的运动或受力量”，知道它们会在 `05` 的 articulation helper 里再次出现
[ ] 我能用一句话说明质量、质心、惯量为什么是 geometry 到 articulation 的桥梁
```

## 本章目标

- 把 `Model` 里最常见、也最容易让初学者卡住的几何词汇先读顺：frame、transform、rotation、shape、spatial quantity、inertia。
- 给 `04_scene_usd`、`05_rigid_articulation`、`06_collision` 提前铺一层共同词汇，让后续章节不必第一次见这些名字。
- 保持第一遍阅读深度：先追“这些对象在解决什么问题”，不追完整推导，也不提前吞掉后续章节。

## 本章范围

- frame / transform / quaternion 的最低限度用法。
- spatial quantities（twist / wrench / `velocity_at_point`）的第一遍读法。
- primitive / mesh / SDF / heightfield 这些 shape representation 的第一层区别。
- 质量、质心、惯量为什么是 geometry 到 articulation 的桥梁。

## 本章明确不做什么

- 不展开 ABA / CRBA / articulation 动力学细节；这些留给 `05_rigid_articulation`。
- 不展开 GJK / EPA / hydroelastic 等碰撞算法和接触生成细节；这些留给 `06_collision`。
- 不展开 USD / URDF / MJCF 解析链和 scene parsing；这些留给 `04_scene_usd`。

## 前置依赖

- 建议先读完 `02_newton_arch`；如果你还没有把 `Model / State / Control / Solver` 这条架构链读顺，本章会不知道这些词到底在服务谁。
- 如果你在 `02_newton_arch` 里正卡在 `parent / child`、`p / q / axis`、joint frame 这些图像化困惑上，先回 `02_newton_arch/question-notes.md` 对照补充图解，再进入本章会更顺；chapter 03 会把这些问题重新提升成统一的 frame / transform 词汇。
- `01_warp_basics` 的最低限度心智模型已经够用；这里只会把 `wp.transform`、quaternion 等词当成读码对象，不会展开完整 Warp 技巧。
- 不要求你先学完整 Lie group、四元数课程或 Featherstone 推导；本章只补 Newton 第一遍阅读需要的那一层。

## GAMES103 已有 vs 本章新增

| 维度 | GAMES103 已有 | 本章新增 |
|------|----------------|----------|
| 物理 / 数学视角 | 知道刚体有位姿、旋转、质量和惯量，碰撞也依赖几何表示。 | 把这些直觉压缩成 Newton 读码真正会遇到的词：frame / transform / quaternion / spatial quantity / shape / inertia。 |
| Newton 工程视角 | 不要求你把这些概念挂到 `Model`、builder 和 articulation helper 的真实对象上。 | 解释为什么 `Model` 要保存 body / joint / shape 的局部关系、shape transform 和 mass property。 |
| 章节衔接视角 | 更偏通用物理直觉，不负责把 scene、articulation、collision 拆成工程阅读路线。 | 明确本章只做桥接：先把词汇读顺，再分别送往 `04`、`05`、`06`。 |

## 阅读顺序

1. 先读 `principle.md`，把 frame -> transform -> quaternion -> spatial quantity -> shape -> inertia 这条概念链读顺。
2. 如果你是从 `02_newton_arch/question-notes.md` 的 `p / q / axis 为什么这么容易混` 或 `为什么换一套 joint frame，物理还能不变` 跳过来的，就把这一章当成那两节的系统版：这里会把图里的直觉重新收束成统一的 frame / transform 语言。
3. 第一次追源码时，先看 `source-walkthrough.md`，把这些词对到 `_src/math/`、`_src/geometry/`、builder 和 articulation helper 的源码锚点。
4. 想精确追到上游文件、symbol 和行号，再看 `source-walkthrough-deep.md`。
5. 然后进入 `04_scene_usd`，看场景输入怎样变成这些对象和局部关系。
6. 再进入 `05_rigid_articulation`，看 frame、spatial quantity 和 inertia 怎样真正参与 articulation 动力学。
7. 最后进入 `06_collision`，看 shape representation 怎样被碰撞系统消费，而不是在本章提前深挖算法。

## 预期产出

- `principle.md`：用一条 beginner-safe 主线解释 frame / transform / quaternion / spatial quantity / shape / inertia 为什么会一起出现。
- `source-walkthrough.md`：把这些词钉到 `_src/math/`、`_src/geometry/`、builder 和 articulation helper 的高价值源码锚点上。
- 一套稳定的后续阅读入口：去 `04` 时追“这些对象从哪来”，去 `05` 时追“这些 frame 和 inertia 怎样进入 articulation”，去 `06` 时追“这些 shape representation 怎样进入 collision”。
