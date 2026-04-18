---
chapter: 02
title: Newton 总体架构
last_updated: 2026-04-18
source_paths:
  - newton/examples/basic/example_basic_pendulum.py
  - newton/examples/robot/example_robot_cartpole.py
  - newton/examples/cloth/example_cloth_hanging.py
paper_keys:
  - mujoco-warp-paper
newton_commit: 1a230702
---

# 02 Newton 总体架构 例子观察单

`principle.md` 负责讲“这条架构链是什么”，这一页只负责把 quick-win 命令改成观察任务。先把默认命令跑一遍，再带着下面的观察点回看现象与源码；这里写的是观察提示，不是已经执行过的日志。每次只改一个量，再回到 `principle.md` 的四层标签判断你刚才动到的是哪一层。

## 主例子：`basic_pendulum`

### 最适合拿来验证什么

- 你是否已经把“examples 入口 -> 一个具体 example -> `Model / State / Control / Solver` -> 一次 `step`”这条最小链读顺。
- 你是否能把 `Example.__init__()` 和 `simulate()` 分开看：前者负责把世界搭出来，后者负责每个 substep 的接力。
- 你是否能在不引入 USD、批量 world、solver 切换之前，先看懂一个最小刚体 articulation 为什么能跑起来。

### 它覆盖的架构链

`python -m newton.examples basic_pendulum`
-> `newton.examples` 的统一 launcher
-> `example_basic_pendulum.py` 的 `Example.__init__()`
-> `newton.ModelBuilder()` / `builder.finalize()`
-> `self.model.state()` / `control()` / `contacts()`
-> `simulate()` 里的 `clear_forces -> apply_forces -> collide -> solver.step -> swap state`

这条链已经覆盖了 chapter 02 要你先抓住的薄入口、公共 API、运行时对象和一次 step，但还没有引入 USD 导入、多 world 复制或 solver family 切换。

### 跑完后先盯这几个位置

先原样跑一次 `basic_pendulum`，确认你看到的是基线行为，再回到下面三个代码点对照。

- `Example.__init__()`：看场景是怎样被 `builder.add_link()`、`add_joint_revolute()`、`add_ground_plane()` 组起来的。
- 源码里的 `capture()`：把它当成运行后回看的一处代码锚点，确认为什么同一段 `simulate()` 逻辑既能直接跑，也能在 CUDA 上被 graph 重放；它不是命令行参数。
- `simulate()`：这是本章最值得背下来的最小 substep 顺序。

### 改这里会怎样

| 改这里 | 更像在动哪一层 | 预期现象 | 最值得观察 |
|--------|----------------|----------|------------|
| `hx / hy / hz`（连杆盒体尺寸） | `Model` | 连杆长度、厚度和转动惯量一起变，摆动节奏和离地距离也会跟着变。 | 第一帧的几何外形是否和关节位置仍然对得上；如果更长、更低，`viewer.log_contacts()` 里的接触是否更早出现。 |
| `parent_xform` 里 `p=wp.vec3(0.0, 0.0, 5.0)` | `Model` | 整个 pendulum 锚点高度变化，最直接影响是离地余量和何时碰到 ground plane。 | 先别急着看 solver，先看接触有没有从“根本不发生”变成“很快发生”。 |
| `self.sim_substeps`（因此也改了 `self.sim_dt`） | `Solver` 的时间推进设置 | substep 更少时每步更粗，画面和状态更新更容易显得生硬；substep 更多时通常更平滑但每帧工作更多。 | 对照 `simulate()` 的循环次数理解“一帧”和“一次 solver step”不是同一件事。 |
| `axis=wp.vec3(0.0, 1.0, 0.0)` | `Model` | 关节转轴一改，摆动平面就会变，`rot` 只是 viewer 朝向友好化，不是动力学本体。 | 这是区分“相机看起来怎样”和“系统真实在哪个平面运动”的最快办法。 |

### 这一例子最容易看错的地方

- 源码里的 `capture()` 不是另一套物理逻辑，也不是 CLI 选项；它只是把 `simulate()` 的同一段操作在 CUDA 上录成 graph。
- `viewer.apply_forces()` 是这一拍外部输入入口；不要把它和 `builder` 里的静态建模混成一类改动。
- `contacts` 在这里是显式对象，不是 XPBD 私有黑箱；这一点和 `principle.md` 的四层图正好能对上。

## 对照例子：`robot_cartpole --world-count 100`

### 它最适合补哪一块观察

- 看“同一个场景模板怎样被复制成很多 world”，把单例思维切换成批量 world 思维。
- 看 `basic_pendulum` 没覆盖到的两件事：USD 场景导入，以及 `contacts = None` 的刚体无接触路径。
- 看 chapter 02 的架构链在 `SolverMuJoCo` 路线下依然成立，只是场景来源和 solver 选择变了。

### 它覆盖的架构链

`robot_cartpole`
-> `add_usd(...)` 先把单个 cartpole 搭出来
-> `builder.replicate(..., world_count=100)` 把同一模型复制到多 world
-> `builder.finalize()`
-> `SolverMuJoCo(self.model)`
-> `solver.step(...)`（这里没有 `model.collide(...)`）

### 改这里会怎样

| 改这里 | 预期现象 | 最值得观察 |
|--------|----------|------------|
| `--world-count 1 / 10 / 100` | 例子的物理结构不变，但你会越来越明显地把它看成“同一套模型的批量复制”，而不是一个更大的单一场景。 | `builder.replicate(...)` 和 `test_final()` 里按 world 切片的写法是否开始更好理解。 |
| `spacing=(1.0, 2.0, 0.0)` | world 之间的视觉布局变化，但 solver 路线没变。 | 这能帮你把“world 的排布”与“单个 cartpole 的动力学”分开看。 |
| `cartpole.joint_q[-3:] = [0.0, 0.3, 0.0]` | 初始姿态改了，但架构链不变。 | 这是从 `basic_pendulum` 过渡到“同一模型被很多 world 共享同类初值”的最快观察点。 |

## 对照例子：`cloth_hanging --solver xpbd`

### 它最适合补哪一块观察

- 看“同一类场景怎样在 solver family 之间切换”，这比直接背 8 个 solver 名字更容易形成直觉。
- 看 chapter 02 在 soft/cloth 路线下仍然保留同一条主链：建场景、拿 runtime objects、碰撞、solver step。
- 看 `basic_pendulum` 没强调的部分：cloth builder 参数和 solver 参数是两层不同旋钮，不要混改。

### 它覆盖的架构链

`cloth_hanging --solver xpbd`
-> CLI 先选 solver family
-> `add_ground_plane()` + `add_cloth_grid(...)`
-> `builder.finalize()`
-> `self.model.state()` / `control()` / `contacts()`
-> `model.collide(...)`
-> `SolverXPBD(..., iterations=self.iterations).step(...)`

### 改这里会怎样

| 改这里 | 预期现象 | 最值得观察 |
|--------|----------|------------|
| `--solver xpbd` 改成 `vbd` / `semi_implicit` / `style3d` | 场景主题不变，但 solver 路线、substep 设定和部分 builder 参数会跟着切换。 | 先看 `if self.solver_type == ...` 这些分支到底改了哪些地方，再看可视化差异。 |
| `spring_ke`（xpbd 分支） | 布料边约束更软或更硬，但这还是 XPBD 路线内部的参数调节，不是切 solver。 | 这是区分“同一 solver 的参数变化”和“换 solver family”最清楚的一刀。 |
| `--width` / `--height` | cloth 分辨率变了，粒子数和观感复杂度都会跟着变。 | 先从 `add_cloth_grid(...)` 想清楚这是在改场景规模，再去看 `contacts` 和 `step()` 的工作量。 |

## 这页怎么配合其他文件

- `principle.md`：负责回答“为什么 chapter 02 要先讲这条链”。
- `examples.md`：只负责把三个 quick-win 命令改写成“先跑、再改哪里、再看哪里”的观察单。
- `source-walkthrough.md`：负责把你在这里观察到的现象，重新钉回真实源码路径。
