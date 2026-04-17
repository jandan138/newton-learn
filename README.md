# Newton Learn

这个仓库用于围绕 `/shared/smartbot/zhuzihou/dev/newton` 系统学习 Newton。目标不是做二次开发，而是优先建立原理、源码数据流和代表性示例三条主线的稳定心智模型。

## 使用方式

1. 先读本文件，确认整体路径、章节入口和代表性示例。
2. 再看 `PROGRESS.md`，用统一表格记录章节推进、里程碑证据和每周节奏。
3. 开始填内容前，先对照 `conventions/` 与 `templates/`，保证符号、结构和记录方式一致。
4. 引用 Newton 源码时统一遵循 `conventions/source-refs.md` 的 source-ref 格式，避免模糊指代。

## 仓库布局

| 路径 | 作用 |
|------|------|
| `conventions/` | 跨章节统一约定，如符号、术语、source-ref、GPU 和 diffsim 记录规范。 |
| `templates/` | 章节模板，保证 README / principle / source-walkthrough 等文件结构统一。 |
| `references/` | 外部资料导航、BibTeX 条目、按章节整理的论文摘要与链接。 |
| `chapters/` | 17 章主学习骨架，按物理层次和 Newton 心智模型推进。 |
| `assets/` | 仓库级公共图和跨章节复用资产。 |
| `PROGRESS.md` | 章节进度、里程碑证据、冲刺清单与复盘模板。 |
| `INDEX_by_module.md` | 反向索引：上游模块路径对应到学习章节。 |

## 推荐学习路径

主干路径：`00 -> 01 -> 02 -> 03 -> 04 -> 05 -> 06 -> 07 -> 08 -> 09 -> 10 -> 11 -> 12 -> 13 -> 14 -> 15 -> 16`

## 分支路径

先走主干：`00 -> 01 -> 02 -> 03 -> 04 -> 05 -> 06 -> 07`

中段按兴趣分支：`08` / `09 -> 10` / `11` / `13`

最后合流：`12 -> 14 -> 15 -> 16`

## 四大重心

- `06_collision`：碰撞系统、接触生成、hydroelastic 与可微接触伏笔。
- `08_rigid_solvers`：Featherstone、MuJoCo、SemiImplicit、Kamino 四条刚体主路径。
- `11_mpm`：显式 APIC 与隐式 MPM 的工程与数学双路径。
- `13_diffsim`：Warp tape、梯度可用性、FD 验证与优化闭环。

## 章节索引

| 章节 | 主题 | 对应上游主模块 | 代表性示例 | 备注 |
|------|------|----------------|------------|------|
| `00_prerequisites` | 前置速查 | 无固定源码 | - | 轻量入口，只做速查与深入指针。 |
| `01_warp_basics` | Warp 编程模型 | Warp 用法 + Newton 中的调用模式 | - | 以最小 kernel 心智模型为主。 |
| `02_newton_arch` | Newton 总体架构 | `newton/__init__.py`、`newton/_src/core/`、`newton/examples/` | `basic_pendulum` | 首个快速胜利章。 |
| `03_math_geometry` | 数学与几何基础 | `newton/_src/math/`、`newton/_src/geometry/` | - | 为碰撞、场景和 solver 做几何准备。 |
| `04_scene_usd` | 场景描述与 USD 解析 | `newton/_src/usd/`、`newton/_src/sim/` | `basic_urdf` | 聚焦 Model 构建与载入链路。 |
| `05_rigid_articulation` | 刚体与关节动力学 | `newton/_src/sim/`、`newton/_src/solvers/featherstone/` | `basic_pendulum`, `basic_joints`, `robot_cartpole`, `robot_g1` | 从 articulation 和 Featherstone 切入。 |
| `06_collision` | 碰撞系统 | `newton/_src/geometry/`、`newton/_src/sim/` | `contacts_pyramid` | CLI 名称为 `pyramid`。 |
| `07_constraints_contacts_math` | 约束与接触数学 | 约束数学与接触中间量 | `robot_cartpole`, `contacts_pyramid` | 先搭数学骨架，再进具体 solver。 |
| `08_rigid_solvers` | 刚体求解器家族 | `newton/_src/solvers/featherstone/`、`newton/_src/solvers/mujoco/`、`newton/_src/solvers/semi_implicit/`、`newton/_src/solvers/kamino/` | `robot_cartpole`, `robot_g1`, `contacts_pyramid` | `contacts_pyramid` 的 CLI 名称为 `pyramid`。 |
| `09_variational_solvers` | 变分求解器族 | `newton/_src/solvers/xpbd/`、`newton/_src/solvers/vbd/`、`newton/_src/solvers/style3d/` | `cloth_style3d`, `cable_twist` | 统一看投影/变分视角。 |
| `10_softbody_cloth_cable` | 软体、布料与 Cable | `newton/_src/solvers/semi_implicit/`、`newton/_src/solvers/xpbd/`、`newton/_src/solvers/vbd/`、`newton/_src/solvers/style3d/` | `cloth_hanging`, `cloth_style3d`, `cable_twist` | 对比 FEM、XPBD、AVBD/VBD。 |
| `11_mpm` | MPM 双路径 | `newton/_src/solvers/implicit_mpm/` | `mpm_granular`, `mpm_twoway_coupling` | 同时覆盖显式 APIC 与隐式 MPM。 |
| `12_sensors_ik` | 传感器与 IK | `newton/_src/sensors/`、`newton/ik.py` | `sensor_contact`, `ik_franka` | 聚焦数据流与求解入口。 |
| `13_diffsim` | 可微分仿真 | `newton/examples/diffsim/` + 多 solver adjoint 路径 | `diffsim_ball`, `diffsim_soft_body`, `diffsim_drone`, `diffsim_spring_cage` | 必产出各 solver 可微性矩阵表。 |
| `14_viewer_integration` | Viewer 与生态集成 | `newton/_src/viewer/`、`newton/viewer.py` | - | 看显示链路与外部集成边界。 |
| `15_multiphysics_pipeline` | 多物理耦合流水线 | `newton/examples/multiphysics/` | `mpm_twoway_coupling`, `softbody_dropping_to_cloth` | 关注端到端数据流，而不追求铺全所有耦合。 |
| `16_self_experiments` | 自制小实验 | `newton/examples/` 反向借鉴 + 自建实验 | - | 用实验验证章节结论，不替代主线学习。 |

## 时间投入反推

| 每周投入 | 预估总周期 | 节奏 |
|----------|------------|------|
| 6 小时 | 12 个月 | 休闲节奏；M1 硬截止在第 6 周。 |
| 10 小时 | 8 个月 | 主流节奏；推荐，M1 硬截止在第 3 周。 |
| 15 小时 | 5 个月 | 冲刺节奏；M1 硬截止在第 2 周。 |

## 首周冲刺

首周冲刺只保留方向摘要，具体可执行清单以 `PROGRESS.md` 为准。

- Day 1-2：环境就绪、跑通首批代表性例子、记录上游基线。
- Day 3-5：优先推进 `02_newton_arch`，拉通四层关系与 8 大 solver 全景。
- Day 6-7：回补 `00_prerequisites` 轻量入口，并做首轮 pitfalls / 模板复盘。

## 总纲建议

1. 原理优先不等于先把所有原理一口气看完，`principle` 和 `source-walkthrough` 应交替推进。
2. `pitfalls.md` 不要空着，每次遇到“不对劲”都应及时记下来。
3. `experiments/` 不是必需品，只做能真正验证理解的问题。
4. 论文以条目式阅读为主，只有 `[MUST]` 项才要求读全文。
5. Newton 迭代很快，建议每月同步一次上游并更新行号与差异记录。
6. 每章满足三条完成门槛就推进，不追求一次写到完美。
7. 设定 M1 硬截止，没达成就复盘而不是盲目前进。
