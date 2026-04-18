---
chapter: 01
title: Warp 编程模型
last_updated: 2026-04-18
source_paths:
  - newton/_src/sim/model.py
  - newton/_src/solvers/xpbd/solver_xpbd.py
  - newton/_src/solvers/xpbd/kernels.py
  - newton/examples/selection/example_selection_materials.py
  - newton/examples/mpm/example_mpm_twoway_coupling.py
  - newton/examples/diffsim/example_diffsim_bear.py
paper_keys:
  - warp-paper
  - mujoco-warp-paper
newton_commit: 1a230702
---

# 01 Warp 编程模型 源码走读

`principle.md` 先给概念，这份 walkthrough 只做一件事：把 chapter 01 里的 Warp 词，锚到 Newton 真会出现的代码角色上。阅读重点是“这些词在 Newton 里各自像什么”，不是提前钻 solver 数学。下面所有 `path:line` 引用都以 front matter 里的 `newton_commit: 1a230702` 为准；如果你对照的是别的 revision，行号可能会漂移。

## 涉及路径

| 路径 | 角色 | 先读原因 |
|------|------|----------|
| `newton/_src/sim/model.py` | `Model / State / Control` 的 `wp.array` 缓冲区落点。 | chapter 01 里最先要把 `wp.array` 从抽象名词变成 Newton 里的长寿命数据底座。 |
| `newton/examples/selection/example_selection_materials.py` | 一个很干净的 example：既有 `wp.launch` + `wp.tid()`，也有 graph capture/replay。 | 可以先用不太重的业务语义，把 Warp 读法练熟。 |
| `newton/_src/solvers/xpbd/solver_xpbd.py` | 真实 solver 的 launch site。 | 用来练“先看 launch，再跳 kernel”的阅读顺序。 |
| `newton/_src/solvers/xpbd/kernels.py` | kernel 本体；这里能看到 `wp.tid()` 和共享累加。 | 对上 launch 之后，才知道单个线程到底在处理谁。 |
| `newton/examples/mpm/example_mpm_twoway_coupling.py` | 最直白的 `wp.atomic_add` 例子。 | 能把 `atomic_add` 读成“安全累加”，而不是“新物理”。 |
| `newton/examples/diffsim/example_diffsim_bear.py` | chapter 01 唯一值得保留的 `tile` 锚点。 | 说明 `tile` 确实存在，但在 Newton 里是少见且偏高级的写法。 |

## 先带着哪五个问题读

1. 这块 `wp.array` 是 `Model`、`State` 还是 `Control` 的缓冲区？它会活多久？
2. 这个 kernel 是按 body、joint、contact、particle，还是别的维度在批量跑？先看 launch site 才知道。
3. `wp.tid()` 拿到的是哪个逻辑索引？一维还是多维？
4. `wp.atomic_add` 是不是只是因为很多线程要往同一块缓冲区回写？
5. `tile` 和 `Graph` 是不是在改执行组织方式，而不是在引入新的物理概念？

## 源码锚点

### 1. `wp.array` 在 Newton 里先读成长寿命缓冲区

`Model` 里直接把大量长期存在的仿真数据声明成 `wp.array` 字段，比如 `body_q`、`body_qd`、`body_mass`、`joint_q`、`joint_qd` 等，见 `newton/_src/sim/model.py:L390-L497`。这一步就已经足够说明：在 Newton 里，`wp.array` 不是零散临时值，而是模型数据真正落地的批量缓冲区。

继续往下看，`Model.state()` 会 clone 初始位置/速度并分配力缓冲，`Model.control()` 会 clone 或复用控制输入，见 `newton/_src/sim/model.py:L808-L902`。所以 chapter 01 里的第一层翻译可以固定成一句话：`wp.array` 在 Newton 里常常就是 `Model / State / Control` 这些对象背后的长期数据底座。

### 2. `kernel + wp.launch + wp.tid()` 要从 launch site 往外读

先看一个轻量入口。`selection_materials` 里，launch site 写的是 `dim=self.default_ant_dof_positions.shape`，见 `newton/examples/selection/example_selection_materials.py:L119-L131`；跳到 kernel 本体，`compute_middle_kernel()` 立刻把 `wp.tid()` 解成 `world, arti, dof`，见 `newton/examples/selection/example_selection_materials.py:L37-L40`。这就告诉你：这里不是“普通函数在算一个中间值”，而是在对一个三维批量缓冲区逐元素写入。

同样的读法放进真实 solver 更重要。`SolverXPBD.step()` 发起 `solve_particle_shape_contacts` 时，`dim=contacts.soft_contact_max`，输出是 `particle_deltas` 和 `body_deltas`，见 `newton/_src/solvers/xpbd/solver_xpbd.py:L364-L397`。再跳到 kernel，本体先拿 `tid = wp.tid()`，然后把它翻成 contact slot，再沿着 `contact_shape[tid]`、`contact_particle[tid]` 找到真正受影响的 shape 和 particle，见 `newton/_src/solvers/xpbd/kernels.py:L143-L152`。第一遍阅读最该抓住的不是接触约束公式，而是：这次 launch 在按 contact 批量跑，不是在按 body 或 particle 直接跑。

### 3. `wp.atomic_add` 是共享累加，不是新的物理词

`mpm_twoway_coupling` 里的 `compute_body_forces()` 是很好的第一锚点。这个 kernel 对每个 collider impulse 取一次 `wp.tid()`，算出它对应哪个 `body_index`，最后用 `wp.atomic_add(body_f, body_index, ...)` 把力和力矩累加回刚体缓冲区，见 `newton/examples/mpm/example_mpm_twoway_coupling.py:L25-L56`。对应的 launch site 是 `dim=self.collider_impulse_ids.shape[0]`，见 `newton/examples/mpm/example_mpm_twoway_coupling.py:L188-L205`。

这里最值得带走的不是 sand-mpm 细节，而是读法：很多 collider 节点都可能回写同一个 body，所以需要安全累加。`wp.atomic_add` 解决的是“并行写回同一格”的执行问题，不是在宣告一个新的 Newton 物理概念。XPBD 里的 contact kernels 也有同样模式，比如 `solve_particle_shape_contacts()` 往 `delta` 回写时就用了原子累加，见 `newton/_src/solvers/xpbd/kernels.py:L218-L222`。

### 4. `wp.tile` 在 chapter 01 只保留一个高级锚点

`tile` 在 Newton 代码里不是随处可见的基础语法。chapter 01 只需要记住一个真实落点：`diffsim_bear` 的 `network()` kernel 用 `wp.tile_load`、`wp.tile_matmul`、`wp.tile_map`、`wp.tile_store` 做一小段 tiled GEMM，见 `newton/examples/diffsim/example_diffsim_bear.py:L69-L81`；它的 launch 也不是普通 `wp.launch`，而是 `wp.launch_tiled(...)`，见 `newton/examples/diffsim/example_diffsim_bear.py:L222-L233`。

这在阅读上的含义很简单：作者开始显式关心局部复用和执行效率了。对 chapter 01 来说，到这里就够，不必把 `tile` 误读成“又多了一套 solver 概念”。除了这种少数高级路径，大多数 Newton 入门代码并不需要先靠 `tile` 才能读懂。

### 5. `Graph` 是围住重复 step loop 的 capture/replay

`selection_materials` 的 `capture()` 很直接：如果设备是 CUDA，就用 `wp.ScopedCapture()` 把整段 `self.simulate()` 包起来，见 `newton/examples/selection/example_selection_materials.py:L153-L158`。`simulate()` 里其实就是一段会反复出现的 substep loop：`clear_forces()`、可能的 `collide()`、`solver.step()`、交换 state，见 `newton/examples/selection/example_selection_materials.py:L160-L172`。之后 `step()` 里不再逐条重发同样的 Warp 工作流，而是直接 `wp.capture_launch(self.graph)` 重放，见 `newton/examples/selection/example_selection_materials.py:L174-L180`。

所以 chapter 01 里对 `Graph` 的稳定翻译应该是：把每帧都会重复的一串 launch 关系先 capture 下来，再 replay。它改变的是调度组织方式，不是仿真规律本身。

## 回指原理

| 源码点 | 对应原理 | 第一遍应该怎么翻译 |
|--------|----------|--------------------|
| `newton/_src/sim/model.py:L390-L497`, `newton/_src/sim/model.py:L808-L902` | `wp.array` 是 Newton 里的模型/状态/控制缓冲区。 | 先问“这块 buffer 归谁、活多久、谁会反复读写它”。 |
| `newton/examples/selection/example_selection_materials.py:L37-L40`, `newton/examples/selection/example_selection_materials.py:L119-L131` | `kernel + wp.launch + wp.tid()` 先是批量执行关系，再是函数体细节。 | 先看 launch 的 `dim` 和输入 shape，再回头解释 `tid`。 |
| `newton/_src/solvers/xpbd/solver_xpbd.py:L364-L397`, `newton/_src/solvers/xpbd/kernels.py:L143-L152` | 真实 solver 里也一样，要从 launch site 判断“这批线程是谁”。 | 第一遍先认出这是按 contact 批量跑。 |
| `newton/examples/mpm/example_mpm_twoway_coupling.py:L25-L56`, `newton/examples/mpm/example_mpm_twoway_coupling.py:L188-L205` | `wp.atomic_add` 是 many-to-one 写回时的安全累加。 | 把它翻译成“共享 accumulation”，不要翻译成“新物理项”。 |
| `newton/examples/diffsim/example_diffsim_bear.py:L69-L81`, `newton/examples/diffsim/example_diffsim_bear.py:L222-L233` | `wp.tile` 稀少、偏高级，主要是局部复用信号。 | 知道它存在就够，不要让它抢走 chapter 01 的主线。 |
| `newton/examples/selection/example_selection_materials.py:L153-L180` | `Graph` 是 capture/replay 重复 step loop。 | 先认出它在省重复调度，不是在换算法。 |

## 带着这些锚点进入 Chapter 02

进入 `02_newton_arch` 前，只要把下面五句带走就够了：

1. `wp.array` 往往就是 `Model / State / Control` 真正持有的批量数据。
2. 读 kernel 时先找 launch site，别反过来。
3. `wp.tid()` 只有放回 launch 的 `dim` 里才有意义。
4. `wp.atomic_add` 多半意味着共享写回，不意味着新物理。
5. `tile` 很少见，`Graph` 是重放重复工作流，这两者都先别喧宾夺主。
