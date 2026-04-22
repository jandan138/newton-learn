---
chapter: 02
title: Newton 总体架构
last_updated: 2026-04-22
source_paths:
  - newton/__init__.py
  - newton/_src/core/
  - newton/examples/__main__.py
paper_keys:
  - mujoco-warp-paper
newton_commit: 1a230702
---

# 02 Newton 总体架构

## 0. 从 `00` 和 `01` 走到 Newton 本体

`00_prerequisites` 已经先把“这些前置词到底是什么意思”补到够用：你见到 `articulation`、`M`、`J`、`kernel`、`wp.array` 这些词时，不会再完全失去方向。`01_warp_basics` 又把另一块地基铺平了：你已经知道 Warp 里的批量执行模型大致是什么，知道 `kernel + wp.launch + wp.tid()` 是怎样把一条规则批量施加到很多对象上的。

所以到了这一章，问题就不再是“这些词是什么意思”，而是“Newton 自己怎样把这些词组织成一个能跑起来的架构”。这也是为什么 `02` 应该读起来像 `00 -> 01` 之后的下一阶台阶，而不是一份孤立的架构备忘录。

本章只做三件事：先用 `basic_pendulum` 接住读者，再从 `newton.examples` 和公共 API 看清 Newton 暴露了哪些核心对象，最后把这些对象收束成 `Model / State / Control / Solver` 四层主干，并补一句 `Contacts` 为什么会在 runtime loop 里并列出现，然后给 8 个 solver 一张鸟瞰地图。目标不是立刻吃透所有 solver，而是先把“一个例子为什么能跑起来”讲顺。

## 1. 从 `basic_pendulum` 走到一次 step

最省力的读法不是一上来扑向某个 solver 源码，而是先跟着一个能跑起来的例子走一遍。`basic_pendulum` 适合作为第一站，不是因为它代表了全部 Newton，而是因为它足够小，却已经把“例子从哪里进来、核心对象怎样接力、一步更新在哪里发生”这条主链完整摆出来了。

第一站是 examples 入口。`python -m newton.examples basic_pendulum` 这类命令提醒你：Newton 不是一堆互不相干的 demo，例子先从同一个入口进来。对初学者来说，这一步的意义不是研究 CLI 包装，而是先接受“学习入口是 example name，后面再顺着例子找到真正的对象和 step”。

第二站是公共 API。沿着 `basic_pendulum` 往里看时，最值得先抓住的不是内部模块树，而是 Newton 选择把哪些名字暴露给用户。把 `newton/__init__.py` 当成目录来读，会更容易看清哪些对象是你应该先认得的核心表面，再决定哪些实现细节可以晚一点追。

第三站才是把这些对象重新放回你在 `00` 里见过的“一步仿真”框架里。到了这里，`basic_pendulum` 的最小执行链就可以顺成下面四句话，再外加一句 `Contacts` 结果缓冲区：

1. 例子先选定一个场景和一组参数，把“我要模拟什么”说清楚。
2. 它通过公共 API 触达核心对象，其中 `Model` 负责把这个场景写成静态描述，也就是“世界里有什么、怎样连着、哪些量通常不会每步重建”。
3. 然后例子准备会随时间变化的 `State`，以及这一拍可能会施加的 `Control`。把它们分别看成“当前快照”和“当前输入”就够支撑第一遍阅读。
4. 最后 `Solver` 读入 `Model`、当前 `State` 和可能存在的 `Control`，完成一次 step，把系统推进到下一拍。
5. 而 `Contacts` 不是另一层静态骨架，而是这条主干旁边的并列结果缓冲区 / handoff object：它接住“这一拍碰撞后得到了什么”，再送给 solver 消费。

这样读时，`basic_pendulum` 就不再只是一个“能跑的 demo 名字”，而是一个教学入口：它把 examples 入口、公共 API、四层主干对象、`Contacts` 结果缓冲区，以及一次 step 串成了同一条链。只要你能用这条主链复述它，后面看更大的例子就有抓手了。

## 2. 四层关系图

![Model / State / Control / Solver 四层关系](assets/02_model_state_control_solver.svg)

这张图不是在引入新的抽象，而是在把 `00` 章的四格直觉换成 Newton 的真实名字。读图时，重点不是背定义，而是看它们在一次 step 里怎样接力：`Model` 先给出边界，`State` 提供当前快照，`Control` 提供这一拍的外部输入，`Solver` 再把前三者组装成一次推进。与此同时，`Contacts` 会作为并列结果缓冲区 / handoff object 在 runtime loop 里出现，负责把碰撞结果送到 solver 手上。

- `Model` 先定边界：拓扑、质量、关节、碰撞形状这类结构，通常不在每步里重建。
- `State` 是时间轴上的快照：位置、速度以及 solver 运行时会更新的量，都放在这一层。
- `Control` 让“这一拍要施加什么命令或目标”有一个独立入口；不是每个例子都复杂，但概念上值得单独分层。
- `Solver` 不是孤立算法名词，而是把前三层接起来完成一次 step 的执行器。
- 真正深入某个例子前，先问变量各自落在哪一层，阅读成本会立刻下降。

## 3. 8 个 solver 的全景图

![8 个 solver 的全景图](assets/02_solver_family_map.svg)

8 个 solver 可以先粗分成两簇：一簇偏刚体与通用动力学主线，另一簇偏约束投影、布料软体和 MPM。第一遍只要先记住“各自大概在哪条路线上、以后去哪一章再展开”，不急着提前吃数学细节。

| Solver | 这一章先记住什么 | 后续章节 |
|--------|------------------|----------|
| Featherstone | 刚体 articulation 主线里的经典名字，先把它和关节树求解联系起来。 | `05_rigid_articulation`, `08_rigid_solvers` |
| MuJoCo | 刚体接触与约束路线里的另一条主路径，先把它和 paper、刚体主线关联起来。 | `07_constraints_contacts_math`, `08_rigid_solvers` |
| SemiImplicit | 最适合帮助你保持“有系统、做一步更新”这条基本节奏的基础路线。 | `08_rigid_solvers`, `10_softbody_cloth_cable` |
| Kamino | 刚体 solver 家族中的另一条高级路线，先记住它属于 rigid path。 | `08_rigid_solvers` |
| XPBD | 约束投影家族的代表，后面读布料和软约束时会频繁回到它。 | `09_variational_solvers`, `10_softbody_cloth_cable` |
| VBD | 变分思路的一条代表路线，后面适合和 XPBD、布料 solver 放在一起看。 | `09_variational_solvers`, `10_softbody_cloth_cable` |
| Style3D | 布料路线里的专门名字，先把它当成 cloth-oriented solver。 | `09_variational_solvers`, `10_softbody_cloth_cable` |
| ImplicitMPM | 粒子-网格材料系统的入口，先记住它明显不在刚体主线里。 | `11_mpm`, `15_multiphysics_pipeline` |

## 4. 快速胜利示例

先把下面四条命令跑过一遍，不求一次讲透全部内部实现，只求把刚才那条主链再走稳一遍：例子名字怎样落到四层主干对象和 `Contacts` 结果缓冲区，又怎样把你带到不同 solver 家族。

```bash
uv sync --extra examples
uv run -m newton.examples basic_pendulum
uv run -m newton.examples robot_cartpole --world-count 100
uv run -m newton.examples cloth_hanging --solver xpbd
```

建议优先改三个参数，观察你改的是四层中的哪一层：

- 改 `--world-count`：观察同一套 `Model`/solver 被复制到多世界时，吞吐和心智模型怎么变化。
- 改 `cloth_hanging` 的 `--solver`：直接体会“同一类场景，不同 solver 家族”的入口切换。
- 改 `basic_pendulum` 或 `robot_cartpole` 的初始条件/驱动相关参数：区分这是在改 `State`、`Control` 还是 `Model`。

## 5. 与后续章节的接口

- `03_math_geometry`：本章只先分清对象角色，下一章再补这些对象背后的坐标、变换和几何基础。
- `04_scene_usd`：本章先把 `Model` 当静态描述，`04` 再拆它是怎样从场景输入被构建出来的。
- `05_rigid_articulation`：本章只把 Featherstone 放进地图，`05` 才真正进入刚体树、关节和 articulation 本身。
- `08_rigid_solvers`：本章只给刚体 solver 地图，`08` 再正面对比 Featherstone、MuJoCo、SemiImplicit、Kamino 这些 rigid routes。

## 6. 与 Newton 实现的映射

- `newton/examples/__main__.py`：提醒你 examples 的运行入口本身很薄，阅读重点应该落在被调起的 example 和它使用的核心对象上。
- `newton/__init__.py`：这里能直接看到 Newton 暴露给用户的基础对象，包括 `Model`、`State`、`Control`、`CollisionPipeline` 等；它是“先认哪些公共名字”的最好目录。
- `newton/_src/core/`：它帮助你理解这些对象背后的基础类型和共用骨架，但 Newton 的总体架构并不只藏在某个 core 大文件里，而是分散在 public API、sim、solver 与 examples 的协作关系中。

如果你读到这里，已经能把 `basic_pendulum` 讲成一条完整链：examples 入口先接住读者，公共 API 给出核心对象名，`Model / State / Control / Solver` 构成四层主干，`Contacts` 作为并列结果缓冲区接住碰撞结果。接下来有两条自然分叉：想先把 `Model` 背后的数学和场景构建补稳，就继续到 `03_math_geometry` 和 `04_scene_usd`；想沿刚体主线往下走，就去 `05_rigid_articulation`，再到 `08_rigid_solvers` 看刚体 solver 家族怎样真正展开。
