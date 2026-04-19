---
chapter: 02
title: Newton 总体架构
last_updated: 2026-04-19
source_paths:
  - newton/__init__.py
  - newton/_src/core/
  - newton/examples/
paper_keys:
  - mujoco-warp-paper
newton_commit: 1a230702
---

# 02 Newton 总体架构

`00_prerequisites` 先回答了“这些前置词到底是什么意思”，`01_warp_basics` 再回答了“Warp 的批量执行模型到底怎样工作”。到了这一章，才正式开始进入 Newton 本体：先从 `basic_pendulum` 和 `newton.examples` 接住读者，再把公共 API、`Model / State / Control / Solver` 四层和 8 个 solver 的地形图串起来。

## 文件分工

- `README.md`：只负责本章边界、完成门槛和阅读入口。
- `principle.md`：先把 `basic_pendulum -> examples 入口 -> 公共 API -> Model / State / Control / Solver` 这条概念链讲顺。
- `examples.md`：把 `basic_pendulum`、`robot_cartpole`、`cloth_hanging` 先变成观察任务。
- `source-walkthrough.md`：新手 / 主 walkthrough。第一次追 chapter 02 源码先看这一份；它把 `example entry -> runtime objects -> simulate loop -> solver` 主线直接讲顺。
- `source-walkthrough-deep.md`：深读锚点版。已经跟上主线后，如果你想精确追上游文件、symbol 和行号，再看这一份。

## 完成门槛

```text
[ ] 我能解释 Model / State / Control / Solver 四层分别负责什么
[ ] 我能跑通并口述 basic_pendulum 的最小执行链
[ ] 我能指出一个具体例子如何从 examples 入口落到四层结构
```

## 本章目标

- 用 `basic_pendulum` 把“examples 入口 -> 公共 API -> `Model / State / Control / Solver` -> 一次 step”这条最小执行链先拉通，而不是一开始就陷进某个 solver 的细节。
- 建立 `Model / State / Control / Solver` 四层心智模型，知道每一层大致放什么、改什么、看什么。
- 对 8 个 solver 先有全景感，先知道它们各在什么路线里，后续再按章节拆开深入。

## 前置依赖

- `00_prerequisites`：已经先解决“这些前置词到底是什么意思”，包括仿真一步框架、`articulation`、`M`、`J` 以及最小 Warp 词汇。
- `01_warp_basics`：已经先解决“Warp 的批量执行模型是怎么工作的”，包括 `kernel`、`wp.array`、`wp.launch`、`wp.tid()`、`wp.atomic_add` 的第一层心智模型。
- `[MUST] mujoco-warp-paper`：本章把它作为 MuJoCo Warp 路线的背景锚点，不要求现在吃透全部推导，但要知道它为何存在。

## GAMES103 已有 vs 本章新增

| 维度 | GAMES103 已有 | 本章新增 |
|------|----------------|----------|
| 物理 / 数学视角 | 知道仿真会分模型、状态、积分与约束求解。 | 把这些抽象对象对应到 Newton 的真实 API 和例子入口。 |
| Newton 工程视角 | 对具体代码库组织通常没有要求。 | 先看 `newton/__init__.py` 暴露了哪些公共对象，再看 `newton/examples/` 怎样把它们串起来。 |
| GPU / Warp 视角 | 对 GPU 数据布局和 kernel 调度着墨较少。 | 知道 Newton 不是单一 solver 教具，而是一组基于 Warp 的 solver 家族入口。 |

## 阅读顺序

1. 先带着 `00` 和 `01` 的答案进入本章：这里不再重讲前置词和 Warp 执行模型，而是开始问“Newton 自己怎样把这些东西组织起来”。
2. 先读 `principle.md`，顺着“`basic_pendulum` -> examples 入口 -> 公共 API -> `Model / State / Control / Solver` -> 一次 step -> solver family 地图”这条概念链往下走。
3. 再读 `examples.md`，把 `basic_pendulum`、`robot_cartpole --world-count 100`、`cloth_hanging --solver xpbd` 先变成观察任务，明确每次运行要改什么、看什么。
4. 第一次追源码时，先看 `source-walkthrough.md`，把这条概念链对到真实源码路径，尤其先纠正“真正的 orchestrator 在 `examples/__main__.py`”这个常见误解。
5. 想精确追到上游文件、symbol 和行号，再看 `source-walkthrough-deep.md`。
6. 再实际运行 `basic_pendulum`，先做 `examples.md` 里的最小改动观察，再把 walkthrough 里的源码锚点和图里的四层对象、一轮 step 过程对上号。
7. 如果中途发现某个动力学词或 Warp 机制又松了，就分别回 `00_prerequisites` 或 `01_warp_basics` 补洞，再回到本章。

## 预期产出

- 一张 `Model / State / Control / Solver` 四层关系图。
- 一张 8 个 solver 的全景图，标出 Featherstone、MuJoCo、SemiImplicit、Kamino、XPBD、VBD、Style3D、ImplicitMPM。
- 一份 `examples.md` 观察单，把三个 quick-win 命令改写成“改哪里、看哪里”的实验入口。
- 一份 `basic_pendulum` 的源码锚点清单，能指出 examples 薄入口、真实 launcher、runtime objects 分层以及一次 `step` 落在哪些文件上。
- 能在不看稿的情况下，口述 `basic_pendulum` 从 examples 入口到 solver step 的最小讲解链。

读完这一章后，如果你想先把 `Model` 背后的数学和场景构建补稳，就继续到 `03_math_geometry` 和 `04_scene_usd`；如果你已经确定自己要沿刚体主线往下走，就直接去 `05_rigid_articulation`，再到 `08_rigid_solvers` 看刚体 solver 家族怎样真正展开。
