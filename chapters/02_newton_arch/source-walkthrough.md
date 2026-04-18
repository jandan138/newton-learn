---
chapter: 02
title: Newton 总体架构
last_updated: 2026-04-18
source_paths:
  - newton/examples/__main__.py
  - newton/examples/__init__.py
  - newton/examples/basic/example_basic_pendulum.py
  - newton/__init__.py
  - newton/_src/sim/__init__.py
  - newton/_src/sim/builder.py
  - newton/_src/sim/model.py
  - newton/_src/sim/state.py
  - newton/_src/sim/collide.py
  - newton/_src/solvers/solver.py
  - newton/_src/solvers/xpbd/solver_xpbd.py
paper_keys:
  - mujoco-warp-paper
newton_commit: 1a230702
---

# 02 Newton 总体架构 源码走读

`principle.md` 负责概念层，这份 walkthrough 只做源码锚定：把 `basic_pendulum` 从 examples launcher 一路落到 `Model / State / Control / Solver` 和一次 `step`。阅读重点是“谁在接力”，不是提前钻进 XPBD 的约束数学。

## 涉及路径

| 路径 | 角色 | 先读原因 |
|------|------|----------|
| `newton/examples/__main__.py` | CLI 薄入口；文件本身只有导入并调用 `main()` 两步，见 `newton/examples/__main__.py:L4-L9`。 | 先确认 examples 命令面板不承担真正 orchestration。 |
| `newton/examples/__init__.py` | 真正的 examples launcher；`run()` 在 viewer loop 里驱动 `example.step()` / `render()`，见 `newton/examples/__init__.py:L265-L337`，`get_examples()` 建立短名到模块名映射，见 `newton/examples/__init__.py:L395-L407`，`init()` 建 viewer，见 `newton/examples/__init__.py:L617-L680`，`main()` 解析例子名并 `runpy.run_module()`，见 `newton/examples/__init__.py:L702-L733`。 | 这是从“例子名字”走到“实际例子模块 + 运行循环”的中枢。 |
| `newton/examples/basic/example_basic_pendulum.py` | 最小 end-to-end 例子；`Example.__init__` 组装 builder、model、solver、state/control/contacts，见 `newton/examples/basic/example_basic_pendulum.py:L20-L85`，`simulate()` / `step()` 定义每帧推进，见 `newton/examples/basic/example_basic_pendulum.py:L87-L113`。 | 最容易把 launcher、公共 API、四层对象和一次 step 串成一条线。 |
| `newton/__init__.py` | 公共 API 目录；把 `Model`、`ModelBuilder`、`State`、`Control`、`CollisionPipeline` 和 `solvers` 暴露在 `import newton` 表面，见 `newton/__init__.py:L49-L98`。 | 先认用户真正碰到的名字，再决定何时下钻 `_src`。 |
| `newton/_src/sim/__init__.py` | sim 层 re-export 接缝；把 `ModelBuilder / Model / State / Control / CollisionPipeline / eval_fk` 聚到同一组 sim API，见 `newton/_src/sim/__init__.py:L4-L33`。 | 解释为什么 example 只写 `newton.ModelBuilder()` 就能接到 sim 核心。 |
| `newton/_src/sim/builder.py` | 场景建模到静态模型的落点；`ModelBuilder.finalize()` 把 builder 累积的 host-side 数据整理并转成 `Model` 的长寿命 buffers，见 `newton/_src/sim/builder.py:L9424-L9464`, `newton/_src/sim/builder.py:L9500-L9599`, `newton/_src/sim/builder.py:L10167-L10338`。 | `basic_pendulum` 里的 links / joints / ground 最后都落在这里。 |
| `newton/_src/sim/model.py` | `Model` 到运行时对象的切口；`state()` / `control()` / `contacts()` / `collide()` 把静态描述接到每步对象和碰撞入口，见 `newton/_src/sim/model.py:L808-L1003`。 | 这里最能看清 `Model` 与 `State / Control / Contacts` 的分工边界。 |
| `newton/_src/sim/state.py` | 时间变化量容器；定义 `State` 持有哪些动态数组，以及 `clear_forces()` 如何在每个 substep 开头清空力缓存，见 `newton/_src/sim/state.py:L9-L129`。 | 它把“当前快照”从概念词变成了真实对象。 |
| `newton/_src/sim/collide.py` | 真正的碰撞流水线；`contacts()` 分配接触缓冲，`collide()` 负责 AABB、broad phase、narrow phase 和 soft contact 生成，见 `newton/_src/sim/collide.py:L730-L1010`。 | 架构上要看清“碰撞”并不直接写进 solver，而是先产出 `Contacts`。 |
| `newton/_src/solvers/solver.py` | solver 通用骨架；给出 `step()` contract，并提供 body / particle integration kernels 与 launch site，见 `newton/_src/solvers/solver.py:L10-L157`, `newton/_src/solvers/solver.py:L177-L316`。 | 先看共通接口，再看具体 solver。 |
| `newton/_src/solvers/xpbd/solver_xpbd.py` | 第一个具体 solver step；`SolverXPBD.step()` 负责把 integration、contact/joint iteration、状态写回接成一次推进，见 `newton/_src/solvers/xpbd/solver_xpbd.py:L245-L723`。 | 用一个真实 solver 把“一次 step 真在做什么”落地，但不提前展开全部 solver 家族。 |

## 调用链总览

先纠正一个最容易走偏的点：`newton/examples/__main__.py` 不是真正的 orchestrator。它只有 `from . import main` 和 `main()` 两步，见 `newton/examples/__main__.py:L4-L9`。例子发现、名字解析、viewer 初始化和运行循环都在 `newton/examples/__init__.py`。

1. CLI 入口先落到薄封装：`python -m newton.examples basic_pendulum` 命中的只是 `newton/examples/__main__.py:L4-L9`，它马上把控制权转交给 `main()`。
2. 真正的 example launcher 在 `newton/examples/__init__.py`。`get_examples()` 扫描 `example_*.py` 生成短名到模块名映射，见 `newton/examples/__init__.py:L395-L407`；`main()` 校验例子名、重写 `sys.argv`，再用 `runpy.run_module()` 执行目标模块，见 `newton/examples/__init__.py:L702-L733`。
3. 具体例子的 `__main__` 分支并不自己再造一套 CLI，而是复用 `newton.examples.init()` 和 `newton.examples.run()`：`example_basic_pendulum.py` 在 `newton/examples/basic/example_basic_pendulum.py:L148-L155` 里先建 viewer/args，再实例化 `Example` 并交给统一 run loop；`init()` 的共享 viewer/device 初始化逻辑在 `newton/examples/__init__.py:L617-L680`。
4. `Example.__init__` 才是场景组装的真正起点。它通过公共 API `newton.ModelBuilder()` 开始建模，随后 `builder.finalize()` 生成 `Model`，再构造 `newton.solvers.SolverXPBD(self.model)`、两份 `state()`、一份 `control()` 和一份 `contacts()`，最后用 `eval_fk()` 把初始 joint 状态展开到 body state，见 `newton/examples/basic/example_basic_pendulum.py:L20-L85`, `newton/__init__.py:L49-L98`, `newton/_src/sim/__init__.py:L4-L33`, `newton/_src/sim/builder.py:L9424-L9464`, `newton/_src/sim/model.py:L808-L1003`。
5. `newton.examples.run()` 只负责外层 viewer loop，它每帧调用 `example.step()` 和 `example.render()`，见 `newton/examples/__init__.py:L265-L337`。对 pendulum 来说，真正的一次 substep 在 `simulate()` 里展开成 `state_0.clear_forces()` -> `viewer.apply_forces()` -> `model.collide()` -> `solver.step()` -> 交换 `state_0/state_1`，见 `newton/examples/basic/example_basic_pendulum.py:L95-L113`, `newton/_src/sim/state.py:L118-L129`, `newton/_src/sim/model.py:L977-L1003`, `newton/_src/solvers/xpbd/solver_xpbd.py:L245-L723`。
6. 如果设备是 CUDA，`capture()` 会把整段 `simulate()` capture 成 graph，后续 `step()` 只做 `wp.capture_launch(self.graph)` 重放，见 `newton/examples/basic/example_basic_pendulum.py:L87-L113`。所以 examples 层不只是在“选例子”，还承担了把统一 run loop 和 GPU graph 重放接起来的角色。

把这条链压成一句话就是：`examples/__main__.py` 只是 CLI 转发器，`examples/__init__.py` 才是 launcher，而真正的仿真推进发生在具体 example 对 `Model / State / Control / Contacts / Solver` 的接力上。

## 数据流切片

| 切片 | 读入 | 中间缓存 / tile | 写出 | 证据 |
|------|------|------------------|------|------|
| builder -> model | `ModelBuilder` 中累计的 links / bodies / shapes / joints / gravity 等 host-side 数据。 | `finalize()` 先做 validation、world starts 构建、逆质量计算、几何 finalize，再把数据转成 `wp.array`。这里看到的是 builder 缓存，不是 per-step tile。 | 长寿命 `Model` 静态 buffers，包括 `body_q/body_qd`、`joint_q/joint_qd`、`shape_*`、`gravity` 等。 | `ModelBuilder.finalize()` at `newton/_src/sim/builder.py:L9424-L9464`, `newton/_src/sim/builder.py:L9500-L9599`, `newton/_src/sim/builder.py:L10167-L10338` |
| model -> runtime objects | `Model` 里的初始状态、控制量和 requested attribute 配置。 | `Model.state()` 克隆初始 `q/qd` 并分配 force buffers，`Model.control()` 克隆或复用 control arrays，`Model.contacts()` 懒创建 collision pipeline 并分配固定容量的 contact buffers。 | `State` / `Control` / `Contacts` 三类运行时对象。 | `Model.state()` at `newton/_src/sim/model.py:L808-L859`, `Model.control()` at `newton/_src/sim/model.py:L861-L902`, `Model.contacts()` at `newton/_src/sim/model.py:L951-L975`, `State` at `newton/_src/sim/state.py:L9-L129` |
| collide path | `Model` 的静态碰撞几何、当前 `State` 的 `body_q` / `particle_q`，以及一个待填充的 `Contacts`。 | `CollisionPipeline.collide()` 清 counters，先算 shape AABB，再跑 broad phase / narrow phase，并在需要时补 differentiable rigid contacts 与 soft contacts。没有 tile 证据，主要是固定容量 contact buffers 和 broad-phase pair buffers。 | 被填满的 `Contacts`，供 solver 下一步消费。 | `Model.collide()` at `newton/_src/sim/model.py:L977-L1003`, `CollisionPipeline.collide()` at `newton/_src/sim/collide.py:L772-L1010` |
| solver substep | `Model`、`state_in`、`state_out`、`control`、`contacts`。 | 通用 integration 之后，XPBD 在 iteration loop 里反复读写 `particle_deltas`、`body_deltas`、`contact_impulse` 等中间缓存；如果有粒子还会先建 `HashGrid`。 | 下一拍写进 `state_out`，并把上一次 step 的 contact impulse 缓存在 solver 内部。 | `SolverBase.integrate_bodies()` / `integrate_particles()` at `newton/_src/solvers/solver.py:L224-L299`, `SolverXPBD.step()` at `newton/_src/solvers/xpbd/solver_xpbd.py:L245-L723` |

## GPU 并行切片

| kernel / 函数 | 并行维度 | atomic | 内存访问 | tile | graph |
|---------------|----------|--------|----------|------|-------|
| `Example.capture()` / `Example.step()` at `newton/examples/basic/example_basic_pendulum.py:L87-L113` | `-` | `-` | capture 的是整段 `simulate()` 对既有 model/state/control/contacts buffers 的访问，不是单个 kernel 的局部缓存。 | `-` | CUDA 上用 `wp.ScopedCapture()` 建图，后续 `wp.capture_launch()` 重放。 |
| `integrate_particles()` at `newton/_src/solvers/solver.py:L10-L47`，launch at `newton/_src/solvers/solver.py:L282-L299` | `dim = model.particle_count`；kernel 内每个 `wp.tid()` 对应一个 particle。 | 本 kernel 未见 atomic。 | 读旧 `x/v/f` 和 `Model` 常量，写 `x_new/v_new`。 | `-` | 可被上层 graph capture，但图证据在 pendulum example，不在 kernel 本身。 |
| `integrate_bodies()` at `newton/_src/solvers/solver.py:L98-L157`，launch at `newton/_src/solvers/solver.py:L243-L264` | `dim = model.body_count`；每个 `wp.tid()` 处理一个 body。 | 本 kernel 未见 atomic。 | 读 body state、质量/惯量、gravity，写 `body_q_new/body_qd_new`。 | `-` | 同上，可被上层 graph capture。 |
| `compute_shape_aabbs` launch at `newton/_src/sim/collide.py:L820-L845` | `dim = model.shape_count`。 | 调用点未展示，不在这里下判断。 | 读 `state.body_q` 与 shape 数据，写 narrow-phase 所需的 AABB / geom buffers。 | `-` | 本文件未展示 graph capture。 |
| `create_soft_contacts` launch at `newton/_src/sim/collide.py:L974-L1010` | `dim = particle_count * model.shape_count`。 | 调用点未展示，不在这里下判断。 | 读 particle/body/shape arrays，写 soft-contact 计数与 contact buffers。 | `-` | 本文件未展示 graph capture。 |
| `SolverXPBD.step()` at `newton/_src/solvers/xpbd/solver_xpbd.py:L245-L723` | 调度层在一个 iteration loop 中发起多次 launch，维度在 `particle_count`、`body_count`、`contacts.*_max`、`joint_count` 之间切换。 | 这一层只看调度，不直接对下游 kernel 是否 atomic 做额外结论。 | 围绕 `particle_deltas`、`body_deltas`、`contact_impulse` 和 `state_out` 反复读写。 | `-` | 可被 example 层整体 capture。 |

这一组 chapter-02 anchors 里没有 `wp.tile` 或 `launch_tiled` 证据，所以 `tile` 一列统一留空不是遗漏，而是刻意避免过度解读。

## 回指原理

| 源码点 | 对应原理 | 为什么不是别的原理 | 待补证据 |
|--------|----------|--------------------|----------|
| `newton/examples/__main__.py:L4-L9`, `main()` at `newton/examples/__init__.py:L702-L733` | examples 入口很薄，真正的 launcher 在 `examples/__init__.py`。 | 不是“每个 example 各自维护一套 CLI”，因为短名发现、名字校验与模块跳转都集中在 `examples/__init__.py`。 | 无；这一点的证据已经足够直接。 |
| `newton/__init__.py:L49-L98`, `newton/_src/sim/__init__.py:L4-L33` | 公共 API 是第一张目录图。 | 不是“先钻 `_src` 深处才能知道架构”，因为 `Model / ModelBuilder / State / Control / CollisionPipeline / solvers` 已先被集中暴露在 public surface。 | 若想继续追 API 背后的文件分工，再回本页“涉及路径”表往下读。 |
| `ModelBuilder.finalize()` at `newton/_src/sim/builder.py:L9424-L9464`, `newton/_src/sim/builder.py:L9500-L9599`, `newton/_src/sim/builder.py:L10167-L10338`, `Model.state()` / `control()` at `newton/_src/sim/model.py:L808-L902` | `Model / State / Control` 是运行时角色分层，不只是 glossary 词。 | 不是“builder 直接长期持有仿真状态”，因为 builder 先收束成静态 `Model`，之后每步需要的 `State / Control` 是再从 `Model` 分配出来的。 | 如果要继续问“builder 数据最初从哪来”，下一跳是 `04_scene_usd`。 |
| `run()` at `newton/examples/__init__.py:L265-L337`, `simulate()` at `newton/examples/basic/example_basic_pendulum.py:L95-L113`, `SolverXPBD.step()` at `newton/_src/solvers/xpbd/solver_xpbd.py:L245-L723` | `basic_pendulum` 可以被口述成一条完整执行链。 | 不是“viewer 或 launcher 完成了仿真”，因为外层 loop 只负责调 `example.step()`，真正推进系统的是 `clear_forces -> collide -> solver.step -> swap state`。 | 若要比较别的 solver 家族，再回 `principle.md` 的 solver 全景图。 |
| `newton/_src/sim/model.py:L951-L1003`, `newton/_src/sim/collide.py:L730-L1010` | `Contacts` 是 `Model` 与 `Solver` 之间的显式中间物，而不是 solver 私有黑箱。 | 不是“碰撞已经埋在 XPBD 里”，因为 `model.contacts()` / `model.collide()` 先生成通用 `Contacts`，solver 只是消费它。 | 后续若要展开接触数学，去 `06_collision` 和 `07_constraints_contacts_math`。 |
