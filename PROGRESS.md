# PROGRESS

## Upstream Baseline

- 上游 Newton 仓库路径：`/shared/smartbot/zhuzihou/dev/newton`
- 当前追踪的 Newton commit：`1a230702`
- 对应设计 spec：`docs/superpowers/specs/2026-04-17-newton-learn-structure-design.md`
- 最近同步日期：`2026-04-17`
- 行号维护规则：统一遵循 `conventions/source-refs.md`，Newton 升级后保留函数名并重新绑定行号。

## Milestones

| 里程碑 | 目标 | 完成证据 | 备注 |
|--------|------|----------|------|
| `M1` | 架构拉通 | 能画 `Model / State / Control / Solver` 四层关系图；能讲清 Warp 的 `kernel / array / tile`；跑通 `basic_pendulum` | 覆盖 `00`、`01`、`02`。 |
| `M2` | 模型加载 | 从 URDF / USD 加载机器人并跑通最简 step loop | 聚焦 `03`、`04` 的模型构建链路。 |
| `M3` | 刚体闭环 | 能独立讲清 ABA 在 Newton 中的 GPU 实现 | 对应 `05_rigid_articulation`。 |
| `M3.5` | 碰撞闭环 | 能讲清 GJK / EPA / MPR / hydroelastic 在 Newton 的选择策略 | 视作 `M3` 的补丁，对应 `06_collision`。 |
| `M4a` | MuJoCo 主路径 | 读懂 `robot_cartpole`；讲清 MuJoCo Warp 的 convex contact + CG/Newton 迭代 | 对应 `07`、`08` 的 MuJoCo 主线。 |
| `M4b` | 变分求解对比 | 能对比 VBD / Style3D / XPBD 的数学骨架 | 对应 `09_variational_solvers`。 |
| `M4c` | Kamino 攻克 | 能读懂 Proximal-ADMM 的数学与 `solvers/kamino/` 关键 kernel | `08_rigid_solvers` 的重点补强。 |
| `M5a` | 软体 / 布料 | 能讲清 XPBD cloth / FEM softbody / cable (AVBD) 的差异 | 对应 `10_softbody_cloth_cable`。 |
| `M5b` | MPM 双路径 | 能读懂 P2G / G2P atomic scatter；能讲清 explicit APIC vs ImplicitMPM | 对应 `11_mpm`。 |
| `M6a` | Diffsim FD 验证 | 跑通 `diffsim_ball` 并按 `diffsim-validation.md` 做 FD 验证且通过 | `13_diffsim` 的硬门槛。 |
| `M6b` | Diffsim 优化闭环 | 用 `diffsim_drone` 或 `diffsim_spring_cage` 完成完整梯度优化循环，并判断梯度可信区间 | `13_diffsim` 的闭环验证。 |

## Week 1 Sprint

### Day 1-2

- [ ] `uv sync --extra examples`
- [ ] `uv run -m newton.examples basic_pendulum`
- [ ] `uv run -m newton.examples robot_cartpole --world-count 100`
- [ ] `uv run -m newton.examples cloth_hanging --solver xpbd`
- [ ] 记录当前 Newton commit，并搭起 `conventions/`、`templates/`、`references/` 基础骨架。

### Day 3-5

- [ ] 为 `chapters/02_newton_arch/` 建立 README 和 principle 起稿。
- [ ] 在图或笔记里明确 `Model / State / Control / Solver` 四层关系。
- [ ] 梳理 8 个 solver 的全景表，不追求一次写全所有 source-walkthrough。

### Day 6-7

- [ ] 为 `chapters/00_prerequisites/` 建立 README 和前两项速查表。
- [ ] 回看 `basic_pendulum`，写出第一条 pitfalls 记录。
- [ ] 根据首周手感微调模板，但不扩展任务边界。

## Chapter Progress

| 章节 | checkbox | 日期 | 笔记 |
|------|----------|------|------|
| `00_prerequisites` | [ ] | 2026-04-17 | 先作为轻量入口，聚焦速查表和首次使用点的前置回链。 |
| `01_warp_basics` | [ ] |  | 重点先放在 kernel / tile / atomic 的最小心智模型。 |
| `02_newton_arch` | [ ] | 2026-04-17 | 先拉通 `Model / State / Control / Solver` 和 8 solver 全景，作为第一周主战场。 |
| `03_math_geometry` | [ ] |  | 先关注变换、SDF、heightfield 与空间代数。 |
| `04_scene_usd` | [ ] |  | 重点看 USD/URDF/MJCF 到 Model 的载入链路。 |
| `05_rigid_articulation` | [ ] |  | 代表性例子以 `basic_pendulum`、`basic_joints`、`robot_cartpole`、`robot_g1` 为主。 |
| `06_collision` | [ ] |  | 重点跟踪 GJK / EPA / MPR / hydroelastic 与 `contacts_pyramid`。 |
| `07_constraints_contacts_math` | [ ] |  | 先搭 LCP / NCP / Delassus / Prox 数学骨架。 |
| `08_rigid_solvers` | [ ] |  | MuJoCo、Featherstone、SemiImplicit、Kamino 四路并看。 |
| `09_variational_solvers` | [ ] |  | 统一对比 XPBD / VBD / Style3D 的变分视角。 |
| `10_softbody_cloth_cable` | [ ] |  | 聚焦 `cloth_hanging`、`cloth_style3d`、`cable_twist`。 |
| `11_mpm` | [ ] |  | 重点读 `mpm_granular` 与 `mpm_twoway_coupling`。 |
| `12_sensors_ik` | [ ] |  | 以 `sensor_contact` 和 `ik_franka` 为切口。 |
| `13_diffsim` | [ ] |  | 先守住 FD 验证，再进入优化闭环。 |
| `14_viewer_integration` | [ ] |  | 补足 viewer 架构与外部生态接入边界。 |
| `15_multiphysics_pipeline` | [ ] |  | 聚焦 `mpm_twoway_coupling` 与 `softbody_dropping_to_cloth`。 |
| `16_self_experiments` | [ ] |  | 用小实验验证理解，不另起主线。 |

## M1 Hard-Cutoff Retrospective Template

1. 截止当前，我是否已经真正达成 `M1` 的三条证据，还是只完成了其中一部分？
2. 如果未达成，主要阻塞来自时间估计、环境问题、阅读顺序，还是章节粒度过大？
3. 哪个任务最耗时但产出最低，是否应该在下一轮直接缩减或推后？
4. 下一轮冲刺里，什么是必须保留的最小闭环，什么可以延后？
5. 为了在新的截止点前达成 `M1`，我具体要删除、保留和提前哪些动作？
