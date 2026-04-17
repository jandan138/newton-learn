---
chapter: 02
title: Newton 总体架构
last_updated: 2026-04-17
source_paths:
  - newton/__init__.py
  - newton/_src/core/
  - newton/examples/
paper_keys:
  - mujoco-warp-paper
newton_commit: 1a230702
---

# 02 Newton 总体架构

## 完成门槛

```text
[ ] 我能解释 Model / State / Control / Solver 四层分别负责什么
[ ] 我能跑通并口述 basic_pendulum 的最小执行链
[ ] 我能指出一个具体例子如何从 examples 入口落到四层结构
```

## 本章目标

- 用 `basic_pendulum` 把 Newton 的最小执行链先拉通，而不是一开始就陷进某个 solver 的细节。
- 建立 `Model / State / Control / Solver` 四层心智模型，知道每一层大致放什么、改什么、看什么。
- 对 8 个 solver 先有全景感，后续再按章节拆开深入。

## 前置依赖

- `00_prerequisites`：补齐刚体、约束、优化和 GPU/Warp 的最低限度词汇表。
- `01_warp_basics`：目标上应先理解 kernel、array、launch 这些 Newton 示例会直接碰到的 Warp 基础，但当前仍是骨架，现阶段先用 `00_prerequisites` 里的 Warp 1 页速查作临时桥接。
- `[MUST] mujoco-warp-paper`：本章把它作为 MuJoCo Warp 路线的背景锚点，不要求现在吃透全部推导，但要知道它为何存在。

## GAMES103 已有 vs 本章新增

| 维度 | GAMES103 已有 | 本章新增 |
|------|----------------|----------|
| 物理 / 数学视角 | 知道仿真会分模型、状态、积分与约束求解。 | 把这些抽象对象对应到 Newton 的真实 API 和例子入口。 |
| Newton 工程视角 | 对具体代码库组织通常没有要求。 | 先看 `newton/__init__.py` 暴露了哪些公共对象，再看 `newton/examples/` 怎样把它们串起来。 |
| GPU / Warp 视角 | 对 GPU 数据布局和 kernel 调度着墨较少。 | 知道 Newton 不是单一 solver 教具，而是一组基于 Warp 的 solver 家族入口。 |

## 阅读顺序

1. 先读 `principle.md`，把四层关系和 8 个 solver 的全景图记住。
2. 再实际运行 `basic_pendulum`，用例子把图里的四层对象对上号。
3. 如果中途发现对 Warp、空间代数或约束术语不稳，立刻跳回 `00_prerequisites` 补洞；其中 `01_warp_basics` 当前仍是骨架，所以先把 00 章的 Warp 速查当临时桥接，再回到本章。

## 预期产出

- 一张 `Model / State / Control / Solver` 四层关系图。
- 一张 8-solver 全景图，标出 Featherstone、MuJoCo、SemiImplicit、Kamino、XPBD、VBD、Style3D、ImplicitMPM。
- 能在不看稿的情况下，口述 `basic_pendulum` 从 examples 入口到 solver step 的最小讲解链。
