---
chapter: 02
title: Newton 总体架构
last_updated: 2026-04-23
source_paths:
  - newton/examples/basic/example_basic_pendulum.py
  - newton/examples/robot/example_robot_cartpole.py
  - newton/examples/cloth/example_cloth_hanging.py
paper_keys:
  - mujoco-warp-paper
newton_commit: 1a230702
---

# 02 Newton 总体架构 例子观察单

`principle.md` 负责讲“这条架构链是什么”，这一页只负责把 quick-win 命令改成观察任务。先把默认命令跑一遍，再带着下面的观察点回看现象与源码；这里写的是观察提示，不是已经执行过的日志。每次只改一个量，再回到 `principle.md` 的四层主干 + `Contacts` 结果缓冲区标签判断你刚才动到的是哪一层。

## 主例子：`basic_pendulum`

![`basic_pendulum` 观察范围图](assets/02_examples_basic_pendulum_scope_bridge.svg)

这张图只把 `basic_pendulum` 限定成 chapter 02 的最小观察入口：先跑默认命令，再把注意力收在最小关节链、runtime objects 和单个旋钮上。只要你还在问“这次改动主要动到哪一层”，就还没有偏离这页的用途。

### 最适合拿来验证什么

![`basic_pendulum` 验证重点海报](assets/02_examples_basic_pendulum_validation_poster.svg)

先把这三项当成 `basic_pendulum` 的检查表，而不是把它当成“已经代表全部 Newton”的例子。它的教学价值在于最小而完整，不在于覆盖面最大。

- 你是否已经把“examples 入口 -> 一个具体 example -> `Model / State / Control / Solver` 四层主干 + `Contacts` 结果缓冲区 -> 一次 `step`”这条最小链读顺。
- 你是否能把 `Example.__init__()` 和 `simulate()` 分开看：前者负责把世界搭出来，后者负责每个 substep 的接力。
- 你是否能在不引入 USD、批量 world、solver 切换之前，先看懂一个最小刚体 articulation 为什么能跑起来。

### 它覆盖的架构链

![`basic_pendulum` 架构链桥接图](assets/02_examples_basic_pendulum_chain_bridge.svg)

先把 handoff 次序读顺就够了，不用急着把这张图继续展开成 `finalize()` 或 solver internals 教程。对这一页来说，最值钱的是你能把“入口、对象、substep 顺序”连成一口气。

`python -m newton.examples basic_pendulum`
-> `newton.examples` 的统一 launcher
-> `example_basic_pendulum.py` 的 `Example.__init__()`
-> `newton.ModelBuilder()` / `builder.finalize()`
-> `self.model.state()` / `control()` / `contacts()`
-> `simulate()` 里的 `clear_forces -> apply_forces -> collide -> solver.step -> swap state`

这条链已经覆盖了 chapter 02 要你先抓住的薄入口、公共 API、运行时对象和一次 step，但还没有引入 USD 导入、多 world 复制或 solver family 切换。

### 跑完后先盯这几个位置

![`basic_pendulum` run-after checkpoints map](assets/02_examples_basic_pendulum_checkpoints_map.svg)

先把这一节当成一个三点检查流程：先看 constructor 组装，再看 `capture()` 有没有被误读成另一条物理支线，最后回到 `simulate()` 把 substep 顺序钉住。

先原样跑一次 `basic_pendulum`，确认你看到的是基线行为，再回到下面三个代码点对照。

- `Example.__init__()`：看场景是怎样被 `builder.add_link()`、`add_joint_revolute()`、`add_ground_plane()` 组起来的。
- 源码里的 `capture()`：把它当成运行后回看的一处代码锚点，确认为什么同一段 `simulate()` 逻辑既能直接跑，也能在 CUDA 上被 graph 重放；它不是命令行参数。
- `simulate()`：这是本章最值得背下来的最小 substep 顺序。

如果你第一次盯 `Example.__init__()` 里的 `j0` / `j1`，先把它看成这条最小关节链：

![`basic_pendulum` two-joint chain intuition](assets/02_pendulum_joint_chain.svg)

- `j0` 用 `parent=-1` 把世界锚点接到 `link_0`。
- `j1` 用 `parent=link_0` 把 `link_1` 接到 `link_0` 末端。
- chapter 02 先把它读成 `world -> link_0 -> link_1` 的两级关节链；`parent_xform`、`child_xform` 和 joint frame 的完整细节，留到后面的 articulation 章节再展开。

顺手记一个符号提醒：`wp.transform(..., q=...)` 里的 `q` 是姿态四元数；`joint_q` 里的 `q` 是关节广义坐标。名字一样，但不是同一种量。

### 改这里会怎样

![`basic_pendulum` knob grouping map](assets/02_examples_basic_pendulum_knobs_map.svg)

这张图先帮你把四种改动分层：前两种更偏 `Model` 布置，第三种是时间推进设置，第四种才是真实转轴方向。先分层，再看下面表格，会比一上来逐格读更快。

| 改这里 | 更像在动哪一层 | 预期现象 | 最值得观察 |
|--------|----------------|----------|------------|
| `hx / hy / hz`（连杆盒体尺寸） | `Model` | 连杆长度、厚度和转动惯量一起变，摆动节奏和离地距离也会跟着变。 | 第一帧的几何外形是否和关节位置仍然对得上；如果更长、更低，`viewer.log_contacts()` 里的接触是否更早出现。 |
| `parent_xform` 里 `p=wp.vec3(0.0, 0.0, 5.0)` | `Model` | 整个 pendulum 锚点高度变化，最直接影响是离地余量和何时碰到 ground plane。 | 先别急着看 solver，先看接触有没有从“根本不发生”变成“很快发生”。 |
| `self.sim_substeps`（因此也改了 `self.sim_dt`） | `Solver` 的时间推进设置 | substep 更少时每步更粗，画面和状态更新更容易显得生硬；substep 更多时通常更平滑但每帧工作更多。 | 对照 `simulate()` 的循环次数理解“一帧”和“一次 solver step”不是同一件事。 |
| `axis=wp.vec3(0.0, 1.0, 0.0)` | `Model` | 关节转轴一改，摆动平面就会变；而 `rot` 是在构造时把 pendulum 链绕 `z` 轴转到更侧对 viewer 的朝向，它不是摄像机参数，也不是和 `axis` 同一种改动。 | 先问“真实绕哪根轴摆、在哪个平面里摆”有没有变，再问“从这个视角看起来像不像侧视图”。 |

如果你总把 `rot` 和 `axis` 混成同一种旋钮，先记住三句：

- 改 `axis`：改的是 joint 允许旋转的真实方向，所以摆动平面会变。
- 改 `rot`：改的是 `parent_xform.q` 给整条 pendulum 链的初始朝向；它会改变这条链在世界里的摆放，也会改变 viewer 看到的正侧关系，但它不是“只是把摄像机转一下”。
- 所以 `rot` 和 `axis` 都可能改变你看到的画面；差别在于，一个是在改关节轴向表达，一个是在改这条链被放进世界时的朝向。

![`rot` vs `axis` visual comparison](assets/02_rot_view_comparison.png)

这张图只用来帮你建立视角直觉：源码实际写的是 `wp.quat_from_axis_angle(..., -wp.pi * 0.5)`；图里把它画成 `90°` 的直观示意，只看“整条链被绕 `z` 轴转到另一侧”这层意思就够了，精确符号仍以源码为准。

### 这一例子最容易看错的地方

![`basic_pendulum` 常见误判海报](assets/02_examples_basic_pendulum_pitfalls_poster.svg)

如果你已经开始把 `capture()`、`apply_forces()` 或 `contacts` 看成另一套平行系统，先停下来用这张图纠偏。把这些误判排掉后，再回头看每个符号落在哪个对象上会更稳。

- 源码里的 `capture()` 不是另一套物理逻辑，也不是 CLI 选项；它只是把 `simulate()` 的同一段操作在 CUDA 上录成 graph。
- `viewer.apply_forces()` 是这一拍外部输入入口；不要把它和 `builder` 里的静态建模混成一类改动。
- `contacts` 在这里是显式对象，不是 XPBD 私有黑箱；它不是在推翻四层主干，而是在 runtime loop 里作为并列结果缓冲区补上碰撞这条支线。

## 对照例子：`robot_cartpole --world-count 100`

![`robot_cartpole` 观察范围图](assets/02_examples_robot_cartpole_scope_bridge.svg)

这张图把 `robot_cartpole` 的作用压缩成一句话：它不是来替代主例子，而是拿来补“USD 场景模板 + 多 world 复制 + MuJoCo 路线”这组观察。看它时，先强迫自己把“复制多少 world”与“单个 world 内部发生什么”分开。

### 它最适合补哪一块观察

![`robot_cartpole` 观察重点海报](assets/02_examples_robot_cartpole_observation_poster.svg)

如果你觉得 `basic_pendulum` 已经把主链讲顺了，但对 batch world 还没有图像感，就直接来这一节。它补的不是新的原理页，而是新的观察角度。

- 看“同一个场景模板怎样被复制成很多 world”，把单例思维切换成批量 world 思维。
- 看 `basic_pendulum` 没覆盖到的两件事：USD 场景导入，以及 `contacts = None` 的刚体无接触路径。
- 看 chapter 02 的架构链在 `SolverMuJoCo` 路线下依然成立，只是场景来源和 solver 选择变了。

### 它覆盖的架构链

![`robot_cartpole` 架构链桥接图](assets/02_examples_robot_cartpole_chain_bridge.svg)

读这张图时，最该圈出来的新站点只有三处：`add_usd(...)`、`replicate(...)` 和“没有 `model.collide(...)`”。只要这三点站稳，`robot_cartpole` 对 chapter 02 的补充就足够了。

`robot_cartpole`
-> `add_usd(...)` 先把单个 cartpole 搭出来
-> `builder.replicate(..., world_count=100)` 把同一模型复制到多 world
-> `builder.finalize()`
-> `SolverMuJoCo(self.model)`
-> `solver.step(...)`（这里没有 `model.collide(...)`）

### 改这里会怎样

![`robot_cartpole` 旋钮观察海报](assets/02_examples_robot_cartpole_knobs_poster.svg)

这三种改法分别对应“复制数量、world 排布、共享初值”三个观察层次。每次只改一个量，会比一次混改多个参数更容易看出你到底是在动模板外层，还是在动每个 cartpole 自己的状态。

| 改这里 | 预期现象 | 最值得观察 |
|--------|----------|------------|
| `--world-count 1 / 10 / 100` | 例子的物理结构不变，但你会越来越明显地把它看成“同一套模型的批量复制”，而不是一个更大的单一场景。 | `builder.replicate(...)` 和 `test_final()` 里按 world 切片的写法是否开始更好理解。 |
| `spacing=(1.0, 2.0, 0.0)` | world 之间的视觉布局变化，但 solver 路线没变。 | 这能帮你把“world 的排布”与“单个 cartpole 的动力学”分开看。 |
| `cartpole.joint_q[-3:] = [0.0, 0.3, 0.0]` | 初始姿态改了，但架构链不变。 | 这是从 `basic_pendulum` 过渡到“同一模型被很多 world 共享同类初值”的最快观察点。 |

## 对照例子：`cloth_hanging --solver xpbd`

![`cloth_hanging` 观察范围图](assets/02_examples_cloth_hanging_scope_bridge.svg)

`cloth_hanging` 这一节只负责把注意力转到“同一类布料场景怎样换 solver family”。它不是布料原理页，而是 chapter 02 里观察 solver route 差异的最快入口。

### 它最适合补哪一块观察

![`cloth_hanging` 观察重点海报](assets/02_examples_cloth_hanging_observation_poster.svg)

如果你总把 cloth builder 参数和 solver 参数混成一团，先看这张图再读下面的 bullets。把“场景规模”和“求解路线”分层看，是这条例子最值钱的训练。

- 看“同一类场景怎样在 solver family 之间切换”，这比直接背 8 个 solver 名字更容易形成直觉。
- 看 chapter 02 在 soft/cloth 路线下仍然保留同一条主链：建场景、拿 runtime objects、碰撞、solver step。
- 看 `basic_pendulum` 没强调的部分：cloth builder 参数和 solver 参数是两层不同旋钮，不要混改。

### 它覆盖的架构链

![`cloth_hanging` 架构链桥接图](assets/02_examples_cloth_hanging_chain_bridge.svg)

这张图最该帮你记住两件事：solver family 是在 CLI 入口先选的，碰撞是在 `contacts()` 之后显式接上的。只要这两站清楚，后面的 soft/cloth 章节会好读很多。

`cloth_hanging --solver xpbd`
-> CLI 先选 solver family
-> `add_ground_plane()` + `add_cloth_grid(...)`
-> `builder.finalize()`
-> `self.model.state()` / `control()` / `contacts()`
-> `model.collide(...)`
-> `SolverXPBD(..., iterations=self.iterations).step(...)`

### 改这里会怎样

![`cloth_hanging` 旋钮观察海报](assets/02_examples_cloth_hanging_knobs_poster.svg)

最实用的读法是先把这三种改动分成三类：换 solver family、调同一路线内部刚度、改 cloth 场景规模。先分层，再去看画面和代码，判断会快很多。

| 改这里 | 预期现象 | 最值得观察 |
|--------|----------|------------|
| `--solver xpbd` 改成 `vbd` / `semi_implicit` / `style3d` | 场景主题不变，但 solver 路线、substep 设定和部分 builder 参数会跟着切换。 | 先看 `if self.solver_type == ...` 这些分支到底改了哪些地方，再看可视化差异。 |
| `spring_ke`（xpbd 分支） | 布料边约束更软或更硬，但这还是 XPBD 路线内部的参数调节，不是切 solver。 | 这是区分“同一 solver 的参数变化”和“换 solver family”最清楚的一刀。 |
| `--width` / `--height` | cloth 分辨率变了，粒子数和观感复杂度都会跟着变。 | 先从 `add_cloth_grid(...)` 想清楚这是在改场景规模，再去看 `contacts` 和 `step()` 的工作量。 |

## 这页怎么配合其他文件

![`examples.md` 页面关系图](assets/02_examples_page_relationships_map.svg)

把这张图当成分工提醒就够了：这里不负责重复讲原理，也不负责重走源码，只负责把三个 quick-win 命令变成观察任务。每当你觉得自己开始写成第二份原理页或第二份 walkthrough，就该回到这张图纠偏。

- `principle.md`：负责回答“为什么 chapter 02 要先讲这条链”。
- `examples.md`：只负责把三个 quick-win 命令改写成“先跑、再改哪里、再看哪里”的观察单。
- `source-walkthrough.md`：负责把你在这里观察到的现象，重新钉回真实源码路径。
