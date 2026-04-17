# Newton 学习仓库文档结构 —— 设计文档

- **日期**: 2026-04-17
- **作者**: addison.fields@kus.ac.th
- **目的**: 为系统从零学习 NVIDIA Newton 物理引擎（已安装于 `/shared/smartbot/zhuzihou/dev/newton`）规划一套完整的学习文档目录结构
- **仓库**: `/shared/smartbot/zhuzihou/dev/newton-learn`
- **定位**: 学习仓，不是二次开发仓；首重底层原理、次重使用
- **版本**: v2（经 4 份专家视角审阅迭代，见 §0.4）

---

## 0. 背景与约束

### 0.1 学习者画像

- 粗略学过 GAMES103（王华民《基于物理的计算机动画入门》）
- 已掌握：刚体动力学 + 四元数基础、弹簧质点布料、隐式/半隐式积分、碰撞冲量/penalty、FEM 软体基础、PBD/XPBD、SPH/PIC/FLIP 流体基础
- 未掌握或浅涉：
  1. 机器人动力学（Featherstone 递归 / articulation / 广义坐标）
  2. MuJoCo 风格的约束求解（convex contact、soft constraint、CG/Newton）
  3. GPU 并行范式（Warp kernel / 内核融合 / SIMT）
  4. 可微分物理（adjoint / reverse-mode AD）
  5. MPM 工程实现（APIC/MLS-MPM、多本构）
  6. OpenUSD 数据模型
  7. SDF / BVH / GJK-EPA / MPR / hydroelastic 等工业级碰撞中间件
  8. 位形流形上的积分（Moreau-Jean、SO(3) symplectic）

### 0.2 Newton 引擎事实（已核查）

- 基于 NVIDIA Warp（GPU 加速），以 MuJoCo Warp 为主求解器后端之一
- **Newton 实际含 8 个求解器**（v2 审阅发现，原 spec 低估）：`Featherstone / MuJoCo / SemiImplicit / Kamino / XPBD / VBD / Style3D / ImplicitMPM`
- 源码路径：`/shared/smartbot/zhuzihou/dev/newton/newton/_src/`，子模块：`core / math / geometry / sim / solvers / sensors / usd / viewer / utils`
- `solvers/` 下含多个子目录：`mujoco / featherstone / semi_implicit / kamino / xpbd / vbd / style3d / implicit_mpm` 等
- 官方文档：`docs/concepts/`、`docs/guide/`、`docs/api/`、`docs/tutorials/`、`docs/integrations/`
- 官方示例：约 60+ 个，分布于 12 个目录（basic / robot / cable / cloth / ik / mpm / sensor / selection / diffsim / multiphysics / contacts / softbody）
- 关键特性：GPU 并行（大量 `wp.kernel`/`wp.atomic_add`/`wp.tile`/`wp.Graph`）+ OpenUSD 支持 + 可微分仿真 + 用户可扩展
- 许可：Apache-2.0（代码）、CC-BY-4.0（文档）

### 0.3 通过 8 轮问询已固化的决策

| 决策项 | 选择 |
|--------|------|
| Q2 范围 | **B**：Newton 主体 + 必要前置补课（不覆盖 Newton 不用的其他算法分支） |
| Q3 形式 | **D + 少量 C**：理论 + 源码走读 + 官方例子注释 + 少量自制小实验 |
| Q4 组织 | **C**：双视图（物理层次主骨架 + 源码模块反向索引） |
| Q5 章节模板 | **D 降级版**：六件套改为"二必四选"（见 §3.2 v2 修订） |
| Q6 辅助资产 | 保留 a/b/c/d/f/g/h + 新增 gpu-conventions 和 diffsim-validation |
| Q7 例子覆盖度 | **A**：每个示例目录挑 1~2 个代表性例子详注，共 18 个 |
| Q8 语言 | 中文为主，术语 & 源码/API 名中英并列 |

### 0.4 v2 迭代摘要（经 4 位专家视角审阅）

v1 经 4 份并行审阅（物理学术 / GPU-Warp 工程 / 可微分仿真 / 学习路径教育学），一致指出三大问题：

1. **章节粒度过粗**：01 Warp 被严重低估、06 求解器族需拆三段、10 diffsim 需拆子章
2. **源码映射遗漏**：v1 只列 4 个 solver，实际有 8 个（漏 SolverKamino / ImplicitMPM / VBD / Style3D，会造成错误心智模型）
3. **执行压力过大**：六件套刚性 + "双向对应"强约束 + 首章即断崖，独自学习者完成率预估 30%

v2 采纳全部 18 项修订（A-F 六大类），核心变化：
- 顶层章节 14 → **17**
- 代表性例子 14 → **18**
- 里程碑 6 → **10**（含子里程碑）
- 六件套降为"二必四选"
- 首周冲刺改靶（快速胜利优先）
- 新增 `gpu-conventions.md`、`diffsim-validation.md`

---

## 1. 仓库顶层布局

```
newton-learn/
├── README.md                     # 仓库入口、学习路线建议、章节索引、时间投入反推表
├── PROGRESS.md                   # 每章进度 checkbox + 日期 + 笔记列
├── INDEX_by_module.md            # newton/_src/ 子模块 → 章节反向索引
│
├── conventions/                  # 跨章节通用约定
│   ├── notation.md               # 数学符号约定
│   ├── source-refs.md            # 源码引用格式规范
│   ├── glossary.md               # 术语表（中英对照）
│   ├── gpu-conventions.md        # [v2 新增] GPU/Warp 工程约定
│   └── diffsim-validation.md     # [v2 新增] 可微分验证协议（FD 规程）
│
├── templates/                    # 六份空白模板（二必四选版，见 §3.2）
│   ├── README.md.tmpl
│   ├── principle.md.tmpl
│   ├── source-walkthrough.md.tmpl   # [v2 修订] 强制含 "GPU 并行切片" 小节
│   ├── examples.md.tmpl
│   ├── pitfalls.md.tmpl             # [v2 修订] 预置 7 个 GPU anchor
│   └── exercises.md.tmpl
│
├── references/                   # 外部资料库（不存 PDF，只存条目 + 摘要）
│   ├── README.md                 # 资料分类导航
│   ├── papers.bib                # BibTeX 条目集
│   ├── papers/                   # 按章节分类的论文 1-paragraph 摘要
│   │   ├── 01_warp_gpu.md        # [v2 新增]
│   │   ├── 05_rigid_articulation.md
│   │   ├── 07_constraints_contacts_math.md
│   │   ├── 08_rigid_solvers.md
│   │   ├── 09_variational_solvers.md
│   │   ├── 11_mpm.md
│   │   └── 13_diffsim.md         # [v2 修订] 强制 [MUST] 清单
│   └── external-links.md         # 博客 / 视频 / slides 链接
│
├── assets/                       # 仓库级公共图（架构图、流水线图）
│
├── docs/superpowers/specs/       # 本设计文档存放处
│
└── chapters/                     # 17 章主骨架（见 §2）
    ├── 00_prerequisites/              # 轻量入口 + 速查（v2 重定位）
    ├── 01_warp_basics/                # [v2 新拆]
    ├── 02_newton_arch/                # [v2 新拆]
    ├── 03_math_geometry/
    ├── 04_scene_usd/
    ├── 05_rigid_articulation/
    ├── 06_collision/                  # [v2 扩容：MPR/hydroelastic/diff_contacts]
    ├── 07_constraints_contacts_math/  # [v2 新增]
    ├── 08_rigid_solvers/              # [v2 扩容：MuJoCo/Featherstone/SemiImplicit/Kamino]
    ├── 09_variational_solvers/        # [v2 新增：VBD/Style3D/XPBD]
    ├── 10_softbody_cloth_cable/       # [v2 扩容：cable 独立小节]
    ├── 11_mpm/                        # [v2 扩容：explicit APIC vs ImplicitMPM]
    ├── 12_sensors_ik/
    ├── 13_diffsim/                    # [v2 内部拆 13.1–13.6]
    ├── 14_viewer_integration/
    ├── 15_multiphysics_pipeline/
    └── 16_self_experiments/
```

**设计要点**：
- 所有主章节收拢到 `chapters/` 下，仓库根只留元文件和辅助资产
- `references/papers/` 按章节切分而非主题切分，便于查某章时单文件打开
- `assets/` 仅放跨章节公共图；章节专属图放在对应章节目录下的 `assets/`

---

## 2. 17 章主骨架

每章标注四要素：**主题 / 对应源码 / GAMES103 gap / v2 审阅要点**。

### 00_prerequisites — 前置速查与深入指针（v2 重定位为"轻量入口"）

- **主题**：概念速览 + 速查表 + 每个 gap 话题的深入指针（指向首次使用章的"前置小节"）
- **不独立讲完任何一个话题**，避免首章断崖
- **速查内容**：机器人动力学入门词表、LCP/NCP 速览、Warp 编程模型 1 页、adjoint AD 1 页、OpenUSD 数据模型 1 页、**刚体位形流形上的积分（Moreau-Jean、SO(3) symplectic）1 页**
- **对应源码**：无
- **v2 审阅采纳**：教育学建议"打散" + 物理学术建议"加 SO(3) 积分" → 折中为"轻量入口 + 分散深入"

### 01_warp_basics — Warp 编程模型（v2 新拆独立章）

- **主题**：Warp 的 kernel / array / tile / launch / atomic / CUDA graph；与 CUDA 的抽象差异
- **产出 3~4 个最小可跑手写 kernel**：vec add → reduction → atomic scatter → tile matmul
- **对应源码**：外部 `warp` 包 + Newton 对 Warp 的使用模式（做不到深讲所有 Warp，锚定 Newton 实际用到的那部分）
- **GAMES103 gap**：GPU 并行范式完全不熟
- **v2 审阅采纳**：GPU agent 指出 Warp 学习量被低估 2~3 倍；拆独立章

### 02_newton_arch — Newton 总体架构（v2 从原 01 分出）

- **主题**：Newton 分层（Model / State / Control / Solver）+ 为什么选 MuJoCo Warp 作主后端 + 8 大求解器全景概览
- **对应源码**：`newton/_src/__init__.py`、`newton/_src/core/`
- **本章含"快速胜利 demo"**：首周冲刺主场（见 §5.3）

### 03_math_geometry — 数学与几何基础

- **主题**：旋转表示、刚体位姿、变换代数、几何原语（mesh / SDF / heightfield / primitive）的数据结构
- **对应源码**：`newton/_src/math/`、`newton/_src/geometry/`（不含碰撞查询，那部分在 06）
- **GAMES103 gap**：SDF 构造与 query、heightfield 采样、spatial vector 代数

### 04_scene_usd — 场景描述与 USD 解析

- **主题**：从 URDF / MJCF / USD 加载模型 → 生成 Model；articulation / site / world 概念
- **对应源码**：`newton/_src/usd/`、`newton/_src/sim/` 中 Model 构建相关
- **GAMES103 gap**：OpenUSD 数据模型（Stage / Prim / Attribute）、Newton custom / extended attributes

### 05_rigid_articulation — 刚体与关节动力学

- **主题**：广义坐标 q / q̇、质量矩阵 M、Coriolis C、重力 g 构建；Featherstone ABA / CRBA；关节雅可比 J
- **对应源码**：`newton/_src/sim/` articulation 部分、`newton/_src/solvers/featherstone/` 中 M / J 装配
- **GAMES103 gap**：**GAMES103 完全未涉及** —— 机器人仿真脊骨
- **v2 可微性伏笔**：principle 末尾加"§本章算子的可微性"小节（ABA 反向）

### 06_collision — 碰撞系统（v2 扩容）

- **主题**：broadphase（BVH）+ narrowphase（GJK / EPA / **MPR**）+ **hydroelastic contact** + SDF query + **contact_reduction** + contact manifold 生成
- **对应源码**：`newton/_src/geometry/` 碰撞查询部分（含 `differentiable_contacts.py`）、`newton/_src/sim/` contact 数据结构
- **GAMES103 gap**：GAMES103 以 penalty 为主，这里补 GJK / EPA / MPR、SDF-based narrowphase、GPU 并行 broadphase、hydroelastic 的连续接触模型
- **v2 审阅采纳**：物理学术 agent 指出 v1 缺 MPR/hydroelastic/contact_reduction/differentiable_contacts；末节作为 §13 diffsim 的伏笔
- **v2 可微性伏笔**：principle 末尾加"§本章算子的可微性"小节（接触 Jacobian 松弛 / 次梯度）

### 07_constraints_contacts_math — 约束与接触数学总论（v2 新增）

- **主题**：LCP / NCP / soft-constraint / **Delassus 算子** / **Proximal-ADMM** / CG-Newton / PGS —— 先讲数学骨架再进具体求解器
- **对应源码**：`newton/_src/dynamics/delassus.py`（若存在）等通用约束数学代码
- **GAMES103 gap**：GAMES103 未涉及凸优化与互补约束理论
- **v2 审阅采纳**：物理学术 agent 建议把原 06 solvers 的数学头拆出做前置

### 08_rigid_solvers — 刚体求解器家族（v2 扩容，重心章）

- **主题**：**MuJoCo Warp**（convex contact + CG/Newton + soft constraint）+ **Featherstone**（Articulated Body Algorithm 积分求解）+ **SemiImplicit** + **Kamino**（Disney Research 的 Proximal-ADMM）
- **对应源码**：`newton/_src/solvers/mujoco/`、`solvers/featherstone/`、`solvers/semi_implicit/`、`solvers/kamino/`、`_src/dynamics/delassus.py`、`solvers/padmm/`
- **核心难点**：对比 4 条求解路径的数学与 GPU 实现差异
- **v2 审阅采纳**：v1 完全遗漏 Kamino；必须补上否则心智模型错误
- **v2 可微性伏笔**：principle 末尾加"§本章算子的可微性"（隐式求解器 IFT / 定点反向）

### 09_variational_solvers — 变分求解器族（v2 新增）

- **主题**：**VBD（Variational Backward Euler）** + **Style3D** + **XPBD（position-based extended）** —— 这三个是统一的"投影/变分"视角
- **对应源码**：`newton/_src/solvers/vbd/`、`solvers/style3d/`、`solvers/xpbd/`
- **GAMES103 gap**：XPBD 你学过，但 VBD / Style3D 全新
- **v2 审阅采纳**：v1 完全遗漏 VBD 和 Style3D
- **v2 可微性伏笔**：principle 末尾加"§本章算子的可微性"

### 10_softbody_cloth_cable — 软体 + 布料 + Cable（v2 扩容）

- **主题**：FEM 软体（本构 + Hessian）+ XPBD/VBD 布料（约束投影）+ **§3 Cable（AVBD，非 Cosserat rod）**
- **对应源码**：`solvers/vbd/`（cable 的实际后端）、`solvers/xpbd/`（cloth 部分）、`solvers/style3d/`
- **GAMES103 gap**：对比工业实现与 GAMES103 简版的差异：约束投影顺序、bending 能量、self-collision、AVBD 的 cable 模型
- **v2 审阅采纳**：物理学术 agent 指出 cable 被误标为 Cosserat，实际是 AVBD/VBD
- **v2 可微性伏笔**：principle 末尾加"§本章算子的可微性"（FEM 能量 Hessian 对梯度条件数的影响）

### 11_mpm — MPM：显式 APIC + 隐式 MPM（v2 扩容）

- **主题**：
  - §1–§3 **显式路径**：网格 / 粒子数据结构、P2G / G2P 核、APIC / MLS-MPM 变体
  - §4 **GPU 数据结构专节**（atomic scatter、稀疏网格、race 处理）
  - §5–§6 **隐式路径**：`SolverImplicitMPM` 的 rheology / variational 数学 + 粘塑性 / 颗粒 / 雪 / 粘弹多本构
- **对应源码**：`newton/_src/solvers/implicit_mpm/`（`implicit_mpm_solver_kernels.py` 等）+ 显式 MPM 相关目录
- **GAMES103 gap**：GAMES103 只浅讲，这里补 weak form、APIC 角动量守恒、多本构、隐式 MPM 的 variational 形式
- **v2 审阅采纳**：GPU agent 指出 MPM 的 GPU 特异性被低估；物理学术指出 ImplicitMPM 与显式 APIC 是两套数学
- **v2 可微性伏笔**：principle 末尾加"§本章算子的可微性"（P2G/G2P 的 adjoint）

### 12_sensors_ik — 传感器与逆运动学

- **主题**：接触传感器、IMU、tiled camera 数据流；IK 求解器（Levenberg-Marquardt / nullspace）
- **对应源码**：`newton/_src/sensors/`、`newton/_src/ik.py`

### 13_diffsim — 可微分仿真（v2 拆子章，重心章）

- **13.1 AD 数学**：reverse-mode AD、adjoint method、IFT 推导
- **13.2 Warp tape 机制**：源码走读 + GPU 内存代价 + kernel 重放与 adjoint buffer
- **13.3 接触不可微性**：smoothing / randomized smoothing / sub-gradient + **"Newton 各求解器后端的可微性矩阵表"**（必产出，覆盖 8 个求解器各自的梯度可用性）
- **13.4 刚体 vs FEM vs MPM 梯度对比**：三种形态的梯度特性 case study
- **13.5 梯度病态诊断**：gradient explosion/vanishing、窗口长度、**FD 验证协议**（指向 `conventions/diffsim-validation.md`）
- **13.6 延伸阅读**：policy gradient 融合、sim2real、system ID（仅给入口，不深讲）
- **对应源码**：横跨多模块，聚焦 `solvers/` adjoint 路径 + `examples/diffsim/`
- **v2 审阅采纳**：DiffSim agent 的全部 6 条建议

### 14_viewer_integration — Viewer 与生态集成

- **主题**：GL / USD / ReRun viewer 架构；与 Isaac Sim / MJX / Gymnasium 的互通
- **对应源码**：`newton/_src/viewer/`、顶层 `newton/viewer.py`

### 15_multiphysics_pipeline — 多物理耦合与端到端流水线

- **主题**：刚体 + 软体 + MPM 两两耦合；URDF → sim loop → policy rollout 的端到端走读
- **对应源码**：`newton/_src/examples/multiphysics/`、跨求解器耦合代码

### 16_self_experiments — 自制小实验

- **主题**：跨章节 C 式小实验代码仓（不是文档章节）
- **结构**：每实验一子目录，含 `why.md` + 代码 + `findings.md`
- **实验清单（v2 扩充）**：
  1. solver tolerance 扫描对收敛步数的影响（对应 §8）
  2. warm-start on/off 对比（对应 §8）
  3. XPBD cloth 子步数与稳定性（对应 §10）
  4. **diffsim 实验簇（对应 §13）**：
     - 4a FD 验证协议执行（步长扫描 + 随机种子固定）
     - 4b 窗口长度 vs 梯度方差
     - 4c 接触 smoothing 系数对梯度偏差/方差的权衡
     - 4d checkpoint 粒度 vs 显存/重算成本
  5. MPM 粒子密度与能量守恒误差（对应 §11）
  6. **GPU 工程实验簇（v2 新增，对应 §01/§08/§11）**：
     - 6a kernel profiling（`wp.ScopedTimer` + Nsight）拆解每 step 耗时
     - 6b batch size / env count 下的吞吐曲线与 occupancy
     - 6c CUDA graph on/off 对比
     - 6d MPM P2G atomic 冲突率（排序 vs 不排序粒子）

**v2 关键变化**：06 (collision)、08 (rigid_solvers)、11 (mpm)、13 (diffsim) 为四大重心，任何路径都需深做。

---

## 3. 每章内部模板（v2 修订：二必四选）

### 3.1 典型章节目录结构

以 `chapters/05_rigid_articulation/` 为样本：

```
chapters/05_rigid_articulation/
├── README.md                    # 本章导航（必，含完成度门槛）
├── principle.md                 # 原理推导（必，含"§本章算子的可微性"末节）
├── source-walkthrough.md        # 源码走读（选，含"GPU 并行切片"小节）
├── examples.md                  # 官方例子注释（选）
├── pitfalls.md                  # 踩坑 / FAQ（选，预置 GPU anchor）
├── exercises.md                 # 自检题（选）
├── assets/                      # 本章专属图，按需创建
└── experiments/                 # C 式自制实验，按需创建
```

### 3.2 v2 修订：二必四选

**v1 的"六件套全必"过重，v2 改为**：

- **必产出**（强制）：
  - `README.md` —— 本章导航 + 完成度门槛
  - `principle.md` **或** `source-walkthrough.md` —— **二选一**，通常先 principle（首重原理）
- **按需产出**（随遇随写，起手允许空）：
  - 另一个（principle / source-walkthrough）—— 强烈鼓励补全，但不阻塞推进
  - `examples.md`
  - `pitfalls.md`
  - `exercises.md`

**完成度门槛**（写在每章 `README.md` 顶部）：
```
本章视为完成当三条命中：
① 能口述本章核心数据流（≤ 2 分钟）
② 能答 3 道概念题（从 exercises.md 抽）
③ 至少读懂 1 个代表性例子
```

三条命中即打勾推进，避免"完美主义陷阱"。

### 3.3 每份文件的内部规范

**`README.md`** —— 本章总控面板
- **完成度门槛**（必，顶部显眼位置）
- 本章目标（一句话）
- 前置依赖（章节 + references 的 [MUST] 条目）
- "GAMES103 已有 vs 本章新增" 表格
- 阅读顺序（principle 与 source-walkthrough 交替推进）
- 预期产出

**`principle.md`** —— 理论笔记
- §0 GAMES103 回顾与 gap（100 字以内）
- §1..§N 理论小节：公式推导 + 算法伪代码 + 直觉解释
- **§N+1 "本章算子的可微性"**（v2 新增，05-11 章必写）：本章涉及的算子在 adjoint 下的表现、稳定性、Newton 源码对应位置；与第 13 章呼应
- §末"与 Newton 实现的映射"：指向 source-walkthrough 对应小节
- 公式用 KaTeX / MathJax
- 图放 `assets/`

**`source-walkthrough.md`** —— 源码走读（v2 模板强化）
- 开头：本章涉及源码路径清单 + 调用链总图
- 按**数据流**切小节
- 代码引用统一格式：`` `newton/_src/sim/articulation.py:L120-L155` ``
- **强制小节 "§N GPU 并行切片"**（v2 新增）：每个关键 kernel 记录
  - 并行维度（per-body / per-contact / per-particle / per-grid-cell）
  - 是否含 atomic
  - 内存访问模式（coalesced / strided / scattered）
  - 是否用 tile
  - 是否进 CUDA graph
- 代码块只贴关键片段（≤20 行）+ "干什么、为什么"
- 每小节末指回 principle.md

**`examples.md`** —— 官方例子注释
- 每个代表性例子一个 `##` 小节
- 结构：目的 → 逐段注释 → "对应原理 §X" → "改这里会怎样"
- 末尾表格列出"其余例子 + diff"

**`pitfalls.md`** —— 踩坑 / FAQ（v2 模板预置 GPU anchor）
- Q/A 卡片格式
- **GPU 专项预置 7 个 anchor**（可留空，但骨架必写）：
  1. 非确定性（atomic 顺序）
  2. fp32 精度与 contact 抖动
  3. race condition
  4. `wp.synchronize` 漏写
  5. host / device 内存拷贝
  6. kernel 重编译开销
  7. CUDA graph capture 时的动态 shape 陷阱

**`exercises.md`** —— 自检题
- 概念题（5~8 题）、源码题（3~5 题）、延伸题（2~3 题）
- 不给标准答案
- 允许 `<details>` 折叠记录自答

### 3.4 强约束（v2 放松一项）

1. **frontmatter**：每份文件顶部必有 `章节号 / 标题 / 最后更新日期 / 对应源码路径列表 / 对应论文条目列表`
2. **源码引用格式统一**：`newton/_src/<path>:L<start>-L<end>`，禁止模糊指代；规范见 `conventions/source-refs.md`
3. **principle ↔ source-walkthrough 双向对应 —— v2 降级为软目标**：principle 每节末尾留 `→ source 指针`占位，允许写 `TODO(link)`；滚动完善，不阻塞章节推进

---

## 4. 辅助资产详细规范

### 4.1 `conventions/`

**`notation.md`** —— 数学符号约定
- 坐标系约定（世界 / 局部 / 关节坐标、z-up vs y-up 的 Newton 选择）
- 向量 / 矩阵约定（列向量默认、M / J / λ 记号）
- 旋转表示（四元数顺序、旋转矩阵方向）
- 广义坐标符号（q / q̇ / q̈ / τ / λ）
- 时间步记号（`x_n`、`x_{n+1}`、Δt / h；substep 用上标 `x^{(k)}`）
- 张量下标惯例（MPM / FEM 用）

**`source-refs.md`** —— 源码引用格式规范
- 单行：`` `newton/_src/solvers/mujoco/mujoco.py:L120` ``
- 范围：`` `...:L120-L155` ``
- 函数：函数 \`solve_constraints()\` at `.../mujoco.py:L200-L280`
- 禁止模糊指代
- Newton 升级时的行号维护策略：保留函数名，重新绑定行号；PROGRESS.md 记录 Newton commit hash

**`glossary.md`** —— 术语表（中英对照，字母序，30~50 条起滚动扩展）

**`gpu-conventions.md`**（v2 新增）—— GPU / Warp 工程约定
- SoA vs AoS 选择原则
- `wp.array` 维度排布约定
- `tid()` / 线程索引约定
- atomic 使用规范与审慎原则
- device / stream 记号
- `wp.synchronize` / `wp.launch` / `wp.Graph` 使用约定

**`diffsim-validation.md`**（v2 新增）—— 可微分验证协议
- FD 验证最小规程：步长扫描（`h ∈ {1e-3, 1e-4, ..., 1e-7}`）、相对误差阈值（`< 1e-3`）、随机种子固定协议
- 梯度数值验证的最佳实践
- 向量化验证（多维同测）
- 适用于 `exercises.md` 延伸题和 §16 实验 4a 的强制协议
- 纳入 §7.5 成功判据

### 4.2 `references/`

- **`papers.bib`**：纯 BibTeX，不存 PDF
- **`papers/<chapter>.md`**：按章节切，每篇一段（BibTeX key + 对应章节 + 必读级别 [MUST / RECOMMENDED / OPTIONAL] + 2-3 句摘要 + 获取方式）

**`papers/01_warp_gpu.md`** 强制 [MUST]（v2 新增）：
- Warp 官方论文
- MuJoCo Warp paper
- MLS-MPM GPU 实现
- NVIDIA 技术博客：CUDA graph、kernel fusion

**`papers/13_diffsim.md`** 强制 [MUST]（v2 修订）：
- DiffTaichi
- Brax
- Warp adjoint paper
- Implicit differentiation / IFT 综述
- Randomized smoothing for contact
- Analytic Policy Gradient (SHAC)
- [RECOMMENDED]：PlasticineLab（可微 MPM）、system ID via diffsim

**`external-links.md`**：分类存放官方文档、视频课、博客链接

### 4.3 `templates/`

六份 `.md.tmpl`，含占位符 `{{章节号}}` / `{{章节标题}}` / `{{Newton commit}}`。

新建章节流程：
```bash
cp templates/principle.md.tmpl chapters/NEW_CHAPTER/principle.md
```

### 4.4 `assets/` 放置规则

- 仓库根 `assets/`：跨章节图
- 章节内 `chapters/XX/assets/`：本章专属图
- 格式：优先 SVG；截图 PNG
- 命名：`{章节号}_{内容描述}.svg`
- 相对路径引用

### 4.5 代表性例子清单（v2 扩至 18 个）

| # | 例子 | 所属章节 | 挑选理由 |
|---|------|---------|---------|
| 1 | `basic_pendulum` | 05 | 最简关节体，ABA 数据流最干净 |
| 2 | `basic_urdf` | 04 | URDF 加载完整链路 |
| 3 | `basic_joints` | 05 | 覆盖所有关节类型 |
| 4 | `robot_cartpole` | 05 + 08 | 经典控制基准 |
| 5 | `robot_g1` | 05 + 08 | 真实人形，articulation 复杂度拉满 |
| 6 | `contacts_pyramid` | 06 + 08 | 接触堆叠 + hydroelastic，测 solver 稳定性 |
| 7 | `cloth_hanging` | 10 | XPBD cloth 最小形态 |
| 8 | **`cloth_style3d`** | 10 | **v2 新增** — 唯一 Style3D 入口 |
| 9 | **`cable_twist`** | 10 | **v2 新增** — VBD + cable（AVBD）唯一窗口 |
| 10 | `mpm_granular` | 11 | MPM 最典型应用（颗粒沙） |
| 11 | `mpm_twoway_coupling` | 11 + 15 | MPM 与刚体双向耦合 |
| 12 | `ik_franka` | 12 | IK 典型用例 |
| 13 | `diffsim_ball` | 13 | 可微刚体，梯度最简 |
| 14 | `diffsim_soft_body` | 13 | 可微 FEM，梯度最难 |
| 15 | **`diffsim_drone`** | 13 | **v2 新增** — 控制/轨迹优化，衔接 policy learning |
| 16 | **`diffsim_spring_cage`** | 13 | **v2 新增** — 结构辨识，衔接 sysID |
| 17 | **`softbody_dropping_to_cloth`** | 15 | **v2 新增** — 多求解器耦合真例子 |
| 18 | `sensor_contact` | 12 | 接触传感器数据流 |

**v2 删除**：`selection_materials`（API 演示，理论覆盖度低）。

其余 40+ 例子在相关章节 `examples.md` 末尾用 diff 表格带过。

---

## 5. 学习路径、里程碑、首个冲刺

### 5.1 两条路径

**主干路径（推荐）**
```
00 → 01 → 02 → 03 → 04 → 05 → 06 → 07 → 08 → 09 → 10 → 11 → 12 → 13 → 14 → 15 → 16
```

**分支路径（按兴趣）**
```
必修主干： 00 → 01 → 02 → 03 → 04 → 05 → 06 → 07
                                              ├─→ 08          ← 刚体/机器人支
                                              ├─→ 09 → 10     ← 变分/软体支
                                              ├─→ 11          ← MPM 支
                                              └─→ 13          ← 可微分支
最后合流： 12 → 14 → 15 → 16
```

**四大重心**：06（碰撞）、08（刚体求解器 +Kamino）、11（MPM 含隐式）、13（diffsim）—— 任何路径都需深做。

### 5.2 里程碑（v2 拆至 10 个）

| # | 里程碑 | 完成标志 | 对应章节 |
|---|--------|---------|---------|
| **M1** | 架构拉通 | 能画 Model / State / Control / Solver 四层关系图；能讲清 Warp 的 kernel / array / tile；跑通 `basic_pendulum` | 00, 01, 02 |
| **M2** | 模型加载 | 从 URDF / USD 加载机器人并跑通最简 step loop | 03, 04 |
| **M3** | 刚体闭环 | 能独立讲清 ABA 在 Newton 的 GPU 实现 | 05 |
| **M3.5** | 碰撞闭环 | 能讲清 GJK / EPA / MPR / hydroelastic 在 Newton 的选择策略 | 06 |
| **M4a** | MuJoCo 主路径 | 读懂 `robot_cartpole`；讲清 MuJoCo Warp 的 convex contact + CG/Newton 迭代 | 07, 08 (MuJoCo) |
| **M4b** | 变分求解对比 | 能对比 VBD / Style3D / XPBD 的数学骨架 | 09 |
| **M4c** | Kamino 攻克 | 能读懂 Proximal-ADMM 的数学 + `solvers/kamino/` 关键 kernel | 08 (Kamino) |
| **M5a** | 软体 / 布料 | 能讲清 XPBD cloth / FEM softbody / cable (AVBD) 差异 | 10 |
| **M5b** | MPM 双路径 | 能读懂 P2G / G2P 的 atomic scatter；能讲清 explicit APIC vs ImplicitMPM | 11 |
| **M6a** | Diffsim FD 验证 | 跑通 `diffsim_ball` + 遵照 `diffsim-validation.md` 协议做 FD 验证并通过 | 13 |
| **M6b** | Diffsim 优化闭环 | 用 `diffsim_drone` 或 `diffsim_spring_cage` 完成完整梯度优化循环 + 判断梯度可信区间 | 13 |

（实 11 个，M3.5 视为补丁；用户选择约定为"10+1"）

每个里程碑到达时 `PROGRESS.md` 打勾 + 写 100 字"当前理解深度"。

### 5.3 首个冲刺（v2 改靶：快速胜利优先）

**Day 1-2 —— 环境就绪 + 第一个快速胜利**
- 跑通 `python -m newton.examples basic_pendulum`、`robot_cartpole`、`cloth_hanging` 三个例子
- 改 3 个参数看现象（例如 gravity / dt / substep）
- `PROGRESS.md` 记录 Newton commit hash
- 建起 `conventions/` / `templates/` / `references/` 骨架

**Day 3-5 —— 02_newton_arch 实战（v2 改靶）**
- 建 `chapters/02_newton_arch/` 的 README + principle（**不做 `source-walkthrough` 也可以**）
- 在 principle 里画"Model / State / Control / Solver 四层关系图"和"8 大求解器全景"
- 不追求写完，追求走通"写一章"的 workflow + 打勾 M1 的第一条

**Day 6-7 —— 回补 00 的首节 + 复盘**
- 写 `chapters/00_prerequisites/README.md` + 速查表的第 1~2 项
- 跑 `basic_pendulum` 并尝试写第一个 pitfalls 条目
- 根据实际手感微调模板

**冲刺目标**：系统运转起来 + 打勾 M1 的第一条，**不是**学完任何章。
**v2 要点**：第一周不碰最抽象的 00 章完整内容，避免断崖。

### 5.4 时间投入反推表（v2 新增）

| 每周投入 | 预估总周期 | 节奏 |
|---------|-----------|------|
| 6 小时 | 12 个月 | 休闲节奏；M1 硬截止第 6 周 |
| 10 小时 | 8 个月 | 主流节奏；**M1 硬截止第 3 周**（推荐） |
| 15 小时 | 5 个月 | 冲刺节奏；M1 硬截止第 2 周 |

**M1 硬截止**：如果到截止日 M1 仍未达成，停止向前推进、做一次完整复盘（是目标太大？计划太密？时间低估？），宁早失败不晚放弃。

### 5.5 总纲性建议（写入 README.md 末尾）

1. **原理优先 ≠ 先看完所有原理**：principle 与 source-walkthrough 交替推进
2. **pitfalls.md 不要空着**：每次"咦不对劲"记下来
3. **experiments/ 不是必需品**：只做必要的
4. **论文读条目不读全文**：[MUST] 读全文，其他只读摘要
5. **Newton 在快速迭代**：每月同步一次上游、更新行号、在 PROGRESS 记录 diff
6. **三条门槛即推进**：别追求一章完美（v2 新增）
7. **M1 硬截止**：未达成就停下来复盘（v2 新增）

---

## 6. 不在本次设计范围内（YAGNI）

- 自动化工具链
- CI / GitHub Actions
- i18n / 英文版笔记
- 与其他物理引擎的对比章节（Brax / Isaac Gym / PhysX）
- 视频讲解 / podcast
- 官方 examples 的逐一注释（固定为 18 个代表性，其他走 diff 表）
- PDF 论文存档
- 学习日志 `CHANGELOG.md`（并入 `PROGRESS.md`）

---

## 7. 成功判据

本仓库被认为**成功**当：

1. 按主干路径走完 17 章，完成 M1~M6b 十个里程碑（M3.5 视为 M3 的补丁）
2. 每章满足 §3.2 的"三条门槛"即可推进；不要求 principle ↔ source-walkthrough 完美双向对应
3. `INDEX_by_module.md` 反查 Newton 任一源码路径都能到至少一个章节讨论
4. 能独立回答每章 `exercises.md` 全部概念题与源码题
5. 完成 §16 实验 **4a（FD 验证，严格执行 `diffsim-validation.md` 协议）** —— 硬性成功判据
6. 完成 §16 实验 **4b/4c 至少一项** —— diffsim 的"真懂"判据
7. 产出一张 **"Newton 各求解器后端的可微性矩阵表"**（13.3 要求）—— 可微分心智模型完整的判据

---

## 8. 后续步骤

本 spec 通过用户审阅后，转入 `writing-plans` 技能生成详细实施计划，涵盖：

1. 建仓顺序（先建 conventions / templates / 仓库根 → 再建 chapters 骨架 → 最后填首周冲刺的两章）
2. 每个目录 / 文件的初始内容建议
3. 与上游 Newton 的同步机制（commit hash 记录、行号漂移修复节奏）
4. 第一周冲刺的可执行任务分解（Day 1-2 / 3-5 / 6-7）
5. M1 硬截止的复盘清单模板

实施计划由 `writing-plans` 技能产出，不在本设计文档范围内。
