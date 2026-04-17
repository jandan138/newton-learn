---
chapter: 00
title: 前置速查与深入指针
last_updated: 2026-04-17
source_paths: []
paper_keys:
  - warp-paper
  - featherstone-book
newton_commit: 1a230702
---

# 00 前置速查与深入指针

## 0. 使用方式

- 不要试图一口气学完整章；这里的目标是先补当前缺口，再尽快回到真正的主线章节。
- 如果你只是看到了一个陌生词，就先查表、记住它大概回答什么问题，以及第一次该去哪一章深入。
- 如果你已经在读某个例子或某篇 paper，优先解决那个当下的具体困惑，而不是转去补一整套背景课。

## 1. 机器人动力学词表速查

| 词 | 最小理解 | 第一次深入章节 |
|----|----------|----------------|
| `articulation` | 由刚体和关节组成的树状或近树状机械系统，是 Newton 刚体主线里的基本对象。 | `05_rigid_articulation` |
| generalized coordinates | 用最小自由度参数描述系统状态的坐标，比如关节角而不是所有刚体世界坐标。 | `05_rigid_articulation` |
| mass matrix `M` | 把加速度、惯性和广义力联系起来的系统质量矩阵，是后续约束和求解器推导的底板。 | `07_constraints_contacts_math` |
| Jacobian `J` | 把广义速度映射到约束空间或任务空间的线性算子。 | `07_constraints_contacts_math` |
| ABA | Articulated-Body Algorithm，偏向正向动力学，适合从关节树结构理解 Featherstone 路线。 | `05_rigid_articulation` |
| CRBA | Composite Rigid Body Algorithm，常用来高效组装系统质量矩阵。 | `05_rigid_articulation` |
| Delassus operator | 约束空间里的等效算子，可把 `J M^-1 J^T` 这类接触/约束耦合关系看得更直接。 | `07_constraints_contacts_math` |
| soft constraint | 不是硬性满足，而是允许一定误差并通过罚项、顺应性或阻尼来稳定求解的约束。 | `09_variational_solvers` |

## 2. Warp 编程模型 1 页速查

- `wp.kernel`：声明 GPU/CPU 上批量执行的 kernel，本质上是“对很多元素重复同一段逻辑”的入口。
- `wp.array`：Warp 的核心数据容器，学习时先把它看成“能被 kernel 读取/写入的设备数组”。
- `wp.launch`：把某个 kernel 以指定维度发射出去；你主要关心输入、输出和 launch 维度有没有对上。
- `wp.tid()`：当前线程或当前元素的索引，通常是 kernel 里定位“我现在在处理谁”的最小入口。
- `wp.atomic_add`：多个线程都可能写同一位置时，用原子加避免数据竞争；后续 MPM 的 scatter 会频繁见到它。
- `wp.tile`：把一小块数据搬到更适合局部复用的执行片段里理解，先建立“为并行局部性服务”的直觉即可。
- `wp.Graph`：把一串 launch 和依赖关系录成可复用执行图，先知道它解决的是重复调度开销问题。
- `Tape/adjoint`：`Tape` 负责记录前向执行，adjoint 对应反向传播入口；读 `13_diffsim` 时再深入可微分链路细节。

## 3. Gap -> 首次深入章节导航

| 当前缺口 | 先补到什么程度 | 首次深入章节 |
|----------|----------------|--------------|
| 机器人动力学 | 能区分 articulation、广义坐标、`M`、`J`、ABA/CRBA 的角色，不要求先手推完整公式。 | `05_rigid_articulation` |
| Warp/GPU | 能看懂 kernel、array、launch、atomic 的最小执行模型。 | `01_warp_basics` |
| OpenUSD | 知道 USD/URDF/MJCF 是怎样进入 `Model` 构建链路的。 | `04_scene_usd` |
| 接触数学 | 知道约束、接触、Delassus operator、互补或近似优化之间是什么关系。 | `07_constraints_contacts_math` |
| MPM | 知道粒子/网格双表示和 P2G/G2P 为什么需要 scatter/gather。 | `11_mpm` |
| 可微分物理 | 知道前向 step、`Tape` 记录和 adjoint 反传为什么能构成优化闭环。 | `13_diffsim` |
