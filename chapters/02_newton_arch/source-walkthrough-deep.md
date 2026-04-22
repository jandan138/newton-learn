---
chapter: 02
title: Newton 总体架构
last_updated: 2026-04-22
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

# 02 Newton 总体架构 深读锚点版

这份 deep walkthrough 把 chapter 02 的精确锚点集中在一起。第一次读本章时，建议先读完 `source-walkthrough.md`，把 `example entry -> runtime objects -> simulate loop -> solver` 这条主线读顺，再回来用这一页追具体文件和行号。

如果你现在不是想追精确行号，而是卡在某个具体问题上，比如双 `__main__`、`joint` 连接关系、`p / q / axis` 的角色区别，先去 `question-notes.md` 看图解版，再回这里追精确锚点会更顺。

## Fast Deep Index

| ID | repo@commit path | symbol | why it matters |
|----|------------------|--------|----------------|
| D1 | `newton@1a230702 newton/__init__.py`, `newton/_src/sim/__init__.py` | public exports | `Model / State / Control / CollisionPipeline / eval_fk` 怎样进入 public surface |
| D2 | `newton@1a230702 newton/examples/__main__.py`, `newton/examples/__init__.py` | `main`, `get_examples`, `run`, `init` | examples launcher 和外层 viewer loop |
| D3 | `newton@1a230702 newton/examples/basic/example_basic_pendulum.py` | `Example.__init__`, `simulate`, `step` | 具体例子怎样组装和推进 runtime 对象 |
| D4 | `newton@1a230702 newton/_src/sim/model.py` | `Model.state`, `Model.control`, `Model.contacts`, `Model.collide` | `Model` 与 runtime 对象的边界 |
| D5 | `newton@1a230702 newton/_src/sim/state.py` | `State` | 动态状态字段目录 |
| D6 | `newton@1a230702 newton/_src/solvers/solver.py` | `SolverBase.step`, `integrate_bodies`, `integrate_particles` | solver 统一 contract |
| D7 | `newton@1a230702 newton/_src/sim/builder.py` | `ModelBuilder.finalize` | `builder -> Model` 的冻结入口 |

## Exact Handoff Trace

### 1. 顶层 API 与 examples 路由

- public sim exports：`newton/__init__.py:49-98`, `newton/_src/sim/__init__.py:4-33`
- CLI 薄入口：`newton/examples/__main__.py:4-9`
- example short-name 路由：`newton/examples/__init__.py:395-407`, `newton/examples/__init__.py:702-733`
- exact handoff：
  - `import newton` 直接拿到 `ModelBuilder / Model / State / Control / CollisionPipeline / eval_fk`
  - `python -m newton.examples basic_pendulum` 先过 `__main__.py`
  - 真正的短名解析、模块跳转和 `runpy.run_module()` 发生在 `examples.__init__.main()`

### 2. `basic_pendulum` 怎样组装 runtime stack

- constructor：`newton/examples/basic/example_basic_pendulum.py:20-85`
- `__main__` 分支：`newton/examples/basic/example_basic_pendulum.py:148-155`
- exact handoff：
  - `builder.add_link()`、`add_shape_box()`、`add_joint_revolute()`、`add_articulation()`、`add_ground_plane()` 先建场景
  - `builder.finalize()` 产出 `Model`
  - `SolverXPBD(self.model)`、两份 `state()`、一份 `control()`、一份 `contacts()` 在同一个 constructor 里建齐
  - `eval_fk()` 在进入主循环前把 generalized state 展开到 body state

### Optional Deep Note: 怎样读一个 revolute joint

下面这段只是 second pass 抓手，不属于 chapter 02 的完成门槛；它的作用只是把你在 `basic_pendulum` 里已经看到的 `parent_xform`、`child_xform`、`axis` 往前讲半步，免得你以后回头看时完全没有抓手。

第一遍先只带走四件事：

- `parent=-1` 表示这个 joint 一侧接世界，`child=link_0` 表示另一侧接第一根杆。
- `parent_xform` / `child_xform` 都是在说“同一个关节点，分别在 parent / child 那一侧怎样表达”。
- `j0` 额外在 `parent_xform.q` 上用了 `rot`，把整条 pendulum 链转到更侧对 viewer 的朝向；`j1` 则两侧都还是 `quat_identity()`。
- 这里的 `transform(..., q=...)` 仍然是在写姿态四元数；它和 runtime state 里的 `joint_q` 不是同一种 `q`。

再往前半步，你会遇到“同一个物理 joint，可以换一套 joint-frame 坐标系继续描述”的情况：

有时你会看到 `parent_xform`、`child_xform`、`axis` 的表达一起变了，但物理 joint 仍然等价。这里先只记住一句话：joint parameterization 可能存在等价写法；这属于第二遍问题，不影响 chapter 02 现在这条 architecture handoff 主线。

如果你想先用图把这个困惑看顺，再回源码核对，可以直接跳到 `question-notes.md` 里对应的两节：`p / q / axis 为什么这么容易混` 和 `为什么换一套 joint frame，物理还能不变`。

### 3. `Model` 是静态描述和对象工厂

- `Model.state()` / `control()`：`newton/_src/sim/model.py:808-902`
- `Model.contacts()` / `collide()`：`newton/_src/sim/model.py:951-1003`
- `State` 字段目录：`newton/_src/sim/state.py:9-129`
- exact handoff：
  - `State` 拿到的是 body/joint 当前快照和 force buffers
  - `Control` 拿到的是 joint target / actuation buffers
  - `Contacts` 来自 cached `CollisionPipeline`
  - `collide()` 明确消费当前 `State` 并填充 `Contacts`

### 4. 外层 run loop 与内层 substep chain

- run loop：`newton/examples/__init__.py:265-337`
- viewer init：`newton/examples/__init__.py:617-680`
- pendulum simulate/step：`newton/examples/basic/example_basic_pendulum.py:87-113`
- exact handoff：
  - `run()` 只负责 viewer 生命周期、step 和 render 的时机
  - `Example.simulate()` 决定这章真正关心的链：`clear_forces -> apply_forces -> collide -> solver.step -> swap`
  - `step()` 还可以走 graph replay，所以 outer loop 与 inner simulation chain 是分层的

### 5. solver contract 的精确入口

- common contract：`newton/_src/solvers/solver.py:224-316`
- integrate helpers：`newton/_src/solvers/solver.py:10-157`, `newton/_src/solvers/solver.py:243-299`
- concrete XPBD path：`newton/_src/solvers/xpbd/solver_xpbd.py:245-439`
- exact handoff：
  - 外层统一传 `state_in / state_out / control / contacts / dt`
  - `SolverBase` 先提供通用 particle/body integration helpers
  - concrete solver 再继续进入自己的 kernel launch 序列

## Optional Branches

### Branch A: `examples.init()` 的 viewer/device 选择

- first pass 可跳过：`newton/examples/__init__.py:617-680`
- 关键原因：chapter 02 真正要守住的是 architecture handoff，不是 viewer matrix

### Branch B: `ModelBuilder.finalize()` 的内部冻结细节

- first pass 可跳过：`newton/_src/sim/builder.py:9424-9639`, `newton/_src/sim/builder.py:10167-10258`
- 关键原因：这里更适合和 `04_scene_usd` 一起读，chapter 02 先知道它是 `builder -> Model` 的冻结点就够了

### Branch C: 其它 examples

- first pass 可跳过：比较不同 example module 怎样组织自己的 `Example`
- 关键原因：`basic_pendulum` 已经足够展示 runtime stack 的最小样板

## Verification Anchors

| 想验证的 claim | 直接打开哪里 |
|----------------|--------------|
| 顶层 public API 只是目录与 re-export | `newton/__init__.py:49-98`, `newton/_src/sim/__init__.py:4-33` |
| `examples/__main__.py` 只是转发器 | `newton/examples/__main__.py:4-9` |
| 真正的 example 路由在 `examples.__init__.py` | `newton/examples/__init__.py:395-407`, `newton/examples/__init__.py:702-733` |
| `basic_pendulum` constructor 一次性把 runtime stack 组好 | `newton/examples/basic/example_basic_pendulum.py:20-85` |
| `run()` 和 `simulate()` 分别负责外层 loop 与内层推进链 | `newton/examples/__init__.py:265-337`, `newton/examples/basic/example_basic_pendulum.py:87-113` |
| solver 统一消费 `state / control / contacts` | `newton/_src/solvers/solver.py:301-316`, `newton/_src/solvers/solver.py:243-299` |
