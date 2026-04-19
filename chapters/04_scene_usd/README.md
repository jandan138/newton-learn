---
chapter: 04
title: 04 场景描述与 USD 解析
last_updated: 2026-04-19
source_paths: []
paper_keys: []
newton_commit: 1a230702
---

# 04 场景描述与 USD 解析

`03_math_geometry` 已经把 `Model` 里的 geometry / spatial 词汇先垫平了：你现在至少知道 `shape_transform`、`joint_X_p / joint_X_c`、`body_com` 这些名字在说什么。chapter 04 接着回答另一个马上会冒出来的问题：这些字段不是手写进 `Model` 的，它们最初是怎样从 scene / asset 输入，经由 importer、schema resolver 和 builder，长成 Newton 静态模型结构的。

所以这一章站在 `03` 之后、`05_rigid_articulation` 和 `06_collision` 之前。它先把“外部场景描述 -> Newton `Model`”这段桥接链读顺，让你在进入 articulation 和 collision 之前，知道 body / joint / shape / mass property 到底从哪里来。

## 文件分工

- `README.md`：只负责本章范围、完成门槛和阅读入口。
- `principle.md`：负责把 `scene input -> importer -> schema resolver -> builder -> Model` 这条概念桥讲顺。
- `source-walkthrough.md`：负责把 importer / builder / `finalize()` / `Model` 的源码锚点串起来。

## 完成门槛

```text
[ ] 我能解释 `add_usd()` / importer / schema resolver / builder / `finalize()` 各自负责哪一段链路
[ ] 我能把 scene graph 里的 body / joint / shape / site / local pose 对到 `shape_transform`、`joint_X_p/X_c`、拓扑关系和 `Model` 静态字段
[ ] 我能说明 schema resolver 只是属性翻译边界，不是 solver、本体动力学或碰撞系统
[ ] 我能说清 authored mass property 与 geometry-derived fallback 的第一层优先关系，以及它为什么会影响后面的 `body_mass / body_com / body_inertia`
```

## 本章目标

- 把 `scene input -> importer -> schema resolver -> builder -> Model` 这条链先读顺，回答 chapter 03 里那些局部 frame 和 mass-property 字段的来源。
- 解释 scene graph 里的 body / joint / shape / site / authored attrs，怎样一步步变成 builder 里的结构和 `Model` 里的静态数组。
- 只保住第一遍阅读真正需要的边界：先知道 importer 在翻译什么、builder 在累积什么、`finalize()` 在冻结什么，而不是提前掉进 parser internals。

## 本章范围

- `builder.add_usd()` / importer 在做什么：先把 authored scene object、局部位姿和资产元数据读成 Newton 能继续累积的结构。
- schema resolver 为什么存在：把 Newton / PhysX / MuJoCo 风格里名字不同、写法不同的属性先归一到同一层物理含义。
- body / joint / shape / site 怎样从 scene graph 变成 builder 字段，再在 `finalize()` 之后落成 `Model` 静态结构。
- mass / inertia 的第一层优先关系：先看资产里有没有明确 author 的质量属性；缺项时，再退回 geometry + density 这一类推导式 fallback。

## 本章明确不做什么

- 不展开 articulation dynamics、Featherstone 递推或 solver 细节；这些留给 `05_rigid_articulation` 和后续 rigid solver 章节。
- 不展开 collision algorithms、接触生成或 broadphase/ narrowphase 细节；这些留给 `06_collision`。
- 不写完整 OpenUSD / MJCF / URDF 教程，不讲完整 authoring workflow、layers、payloads、variants 等生态内容。
- 不深挖 mesh approximation、remeshing、hydroelastic branches，或 importer 的所有边缘分支。

## 前置依赖

- 建议先读完 `03_math_geometry`；如果 `shape_transform`、`joint_X_p/X_c`、`body_com` 这些词还没读顺，这一章会只剩 parser 名词。
- `02_newton_arch` 的四层心智模型也最好还在：这里默认你已经知道 `Model` 是静态结构容器，只是还不知道它最初怎样被喂出来。
- 不要求你先学完整 OpenUSD 课程；本章只使用 scene graph、authored attrs、schema 这些对 Newton importer 真正有用的那一层词汇。

## 先带着这张表读

| 词 / 对象 | 先怎么想 | 一个最小 scene 里它像什么 | 第一次去哪里看 |
|-----------|-----------|----------------------------|------------------|
| scene input | 还没进 Newton 的原始场景描述 | 地面 prim、box prim、revolute joint prim，以及它们各自 author 的局部 pose / attrs | `principle.md` 第 1 节 |
| importer | 把 scene 里真正有用的结构先读出来 | 认出“哪个 prim 是 body、哪个是 joint、哪个 shape 挂在哪” | `principle.md` 第 3 节 |
| schema resolver | 不同 authoring 名字的翻译边界 | 同样在说 joint limit 或 contact margin，但名字可能来自 `newton:*`、`physx*`、`mjc:*` | `principle.md` 第 4 节 |
| builder -> `Model` | 把读出来的关系压成 Newton 静态结构 | box 变成某个 body 槽位、joint 变成 `joint_parent / joint_child`、shape 变成 `shape_transform` 等数组 | `principle.md` 第 5-7 节 |

## 阅读顺序

1. 先读 `principle.md`，把 scene / asset / schema / builder / `Model` 之间的桥接关系读顺。
2. 再读 `source-walkthrough.md`，把 `newton/usd.py`、importer、schema resolver、`builder.py` 和 `model.py` 的边界对到真实源码锚点。
3. 然后进入 `05_rigid_articulation`，看这些 joint、mass 和 inertia 字段怎样被 articulation 动力学消费。
4. 再进入 `06_collision`，看这些 shape 类型、局部位姿和几何配置怎样被碰撞系统消费。

## 预期产出

- `principle.md`：用一条 beginner-safe 主线解释 scene graph、authored attrs、schema resolver、builder 和 `Model` 的关系。
- `source-walkthrough.md`：把 public USD entry、importer、schema resolver、`add_shape()` / `add_joint()` / `finalize()` 和 `Model` 落点串成稳定源码链。
- 一套稳定的字段映射心智模型：知道 local pose、parent-child 关系、shape 配置和 authored mass property 最终分别落到哪些 builder / `Model` 字段。
- 明确的后续入口：去 `05` 继续追 joint / mass / inertia 怎样被动力学消费，去 `06` 继续追 shape / transform 怎样被碰撞系统消费。
