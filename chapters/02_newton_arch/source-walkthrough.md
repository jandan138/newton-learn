---
chapter: 02
title: Newton 总体架构
last_updated: 2026-04-19
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

如果你是第一次读这一章，最好先配合 `principle.md` 一起读；如果你是直接跳进源码走读也没关系，但一旦发现术语开始变快，就先回到 `principle.md` 把对象关系补齐，再回来追源码。

这份主 walkthrough 只追 chapter 02 的最小架构主线：`python -m newton.examples basic_pendulum` 这个入口，怎样一步步接到 public API、`Model / State / Control / Contacts / Solver`，再接到一次真正的 `step()`。目标不是把所有 solver family 全部展开，而是先把“谁在 orchestrate、谁在持有状态、谁在推进系统”讲顺。

## What This Walkthrough Follows

这一页只追下面这条 handoff：

```text
example short name
-> examples launcher
-> concrete Example module
-> Model / State / Control / Contacts / Solver
-> simulate loop
-> solver.step(...)
```

这一页刻意不展开三类东西：

- `builder.finalize()` 的全部内部细节
- XPBD 或其它 solver 的完整数学推导
- 8 个 solver family 的横向比较

第一遍先守住一句话：chapter 02 真正讲的是 **Newton 用什么对象和入口把一轮仿真接起来**。

## One-Screen Chapter Map

```text
python -m newton.examples basic_pendulum
                    |
                    v
      examples/__main__.py -> examples.__init__.main()
                    |
                    v
          example name -> target example module
                    |
                    v
                Example.__init__
                    |
                    v
   ModelBuilder -> Model -> State / Control / Contacts / Solver
                    |
                    v
         run() outer loop -> Example.simulate()
                    |
                    v
   clear_forces -> collide -> solver.step -> swap state
```

## Beginner Path

1. 先看 Stage 1。
   - 想验证什么：`newton` 的 public surface 和 `newton.examples` 入口到底薄到什么程度。
   - 看完后应该能说：CLI 和顶层 API 主要负责把你送到真正的模块，不负责做全部仿真工作。
2. 再看 Stage 2。
   - 想验证什么：`basic_pendulum` 这个具体例子到底组装了哪些 runtime 对象。
   - 看完后应该能说：`Example.__init__` 里已经把 `Model / State / Control / Contacts / Solver` 全部接好了。
3. 再看 Stage 3。
   - 想验证什么：`Model` 怎样分出静态描述、runtime state、control 和 contacts。
   - 看完后应该能说：`Model` 不等于当前时刻的状态，它更像一份静态描述和工厂。
4. 再看 Stage 4。
   - 想验证什么：到底是谁在驱动每帧循环。
   - 看完后应该能说：`examples.run()` 管外层 viewer loop，`Example.simulate()` 管一次真正的 substep 链。
5. 最后看 Stage 5。
   - 想验证什么：solver 在这条链上扮演什么边界角色。
   - 看完后应该能说：solver 统一消费 `state_in / state_out / control / contacts` 这组对象，然后用自己的 kernels 推进一步。

## Main Walkthrough

### Stage 1: public surface 和 example CLI 都是故意做薄的入口

**Claim**

`import newton` 和 `python -m newton.examples ...` 都是目录和分发入口，不是把所有 orchestration 都塞在顶层。

**Why it matters**

如果一开始把 `__main__.py` 或顶层包当成“主控制器”，后面就会一直在错误位置找真正的仿真逻辑。

**Source excerpt**

`newton` 顶层包先把最常用的 sim API 暴露出来：

```python
from ._src.sim import (
    BodyFlags,
    CollisionPipeline,
    Contacts,
    Control,
    JointType,
    Model,
    ModelBuilder,
    State,
    eval_fk,
)

from . import geometry, ik, math, selection, sensors, solvers, usd, utils, viewer
```

而 `python -m newton.examples basic_pendulum` 的 CLI 入口本身只有一个简单转发：

```python
from . import main

if __name__ == "__main__":
    main()
```

真正的路由发生在 `examples.__init__`：

```python
def get_examples() -> dict[str, str]:
    example_map = {}
    ...
    if filename.startswith("example_") and filename.endswith(".py"):
        example_name = filename[8:-3]
        example_map[example_name] = f"newton.examples.{module}.{filename[:-3]}"

def main():
    examples = get_examples()
    example_name = sys.argv[1]
    target_module = examples[example_name]
    sys.argv = [target_module, *sys.argv[2:]]
    runpy.run_module(target_module, run_name="__main__")
```

**Verification cues**

- `newton/__init__.py` 做的是 re-export，不是把实现全堆在顶层。
- `examples/__main__.py` 只是把控制权转给 `main()`。
- `get_examples()` 和 `main()` 的工作是“短名 -> 真实模块”的分发，不是推进仿真。

**Output passed to next stage**

一个具体 example module，例如 `newton.examples.basic.example_basic_pendulum`。

### Stage 2: 具体 `Example` 对象一次性组装出运行时栈

**Claim**

真正的架构 handoff 在具体 example 的构造函数里完成：scene builder、model、solver、state、control、contacts 都在这里接起来。

**Why it matters**

这一步最能把 chapter 02 的抽象词落地。否则 `Model / State / Control / Solver` 很容易继续停留在名词表层面。

**Source excerpt**

`basic_pendulum` 的 `Example.__init__` 基本就是整章的最小样板：

```python
builder = newton.ModelBuilder()

link_0 = builder.add_link()
builder.add_shape_box(link_0, hx=hx, hy=hy, hz=hz)

link_1 = builder.add_link()
builder.add_shape_box(link_1, hx=hx, hy=hy, hz=hz)

j0 = builder.add_joint_revolute(...)
j1 = builder.add_joint_revolute(...)
builder.add_articulation([j0, j1], label="pendulum")
builder.add_ground_plane()

self.model = builder.finalize()
self.solver = newton.solvers.SolverXPBD(self.model)

self.state_0 = self.model.state()
self.state_1 = self.model.state()
self.control = self.model.control()
newton.eval_fk(self.model, self.model.joint_q, self.model.joint_qd, self.state_0)
self.contacts = self.model.contacts()
```

**Verification cues**

- `builder.finalize()` 之后才有真正可运行的 `Model`。
- 两份 `state()` 明确告诉你 Newton 常用双缓冲推进，而不是 in-place 乱改一份状态。
- `control()` 和 `contacts()` 不是 solver 私有物，而是 example 在外层先准备好的对象。

**Output passed to next stage**

一整套 runtime 对象：`Model`、`State`、`Control`、`Contacts`、`Solver`。

### Stage 3: `Model` 自己不等于“当前时刻”，它更像静态描述和对象工厂

**Claim**

`Model` 持有的是静态描述和默认值；真正会在每步变化的状态，要靠 `state()`、`control()`、`contacts()`、`collide()` 这些边界方法分出来。

**Why it matters**

这是 chapter 02 最重要的对象分层。如果把 `Model` 和 `State` 混成一层，后面的 articulation、collision、solver 章节都会乱。

**Source excerpt**

`Model.state()` 和 `Model.control()` 负责分配 runtime 对象：

```python
s = State()
...
s.body_q = wp.clone(self.body_q, requires_grad=requires_grad)
s.body_qd = wp.clone(self.body_qd, requires_grad=requires_grad)
s.body_f = wp.zeros_like(self.body_qd, requires_grad=requires_grad)
...
s.joint_q = wp.clone(self.joint_q, requires_grad=requires_grad)
s.joint_qd = wp.clone(self.joint_qd, requires_grad=requires_grad)

c = Control()
...
c.joint_target_pos = wp.clone(self.joint_target_pos, requires_grad=requires_grad)
c.joint_target_vel = wp.clone(self.joint_target_vel, requires_grad=requires_grad)
c.joint_act = wp.clone(self.joint_act, requires_grad=requires_grad)
c.joint_f = wp.clone(self.joint_f, requires_grad=requires_grad)
```

`Contacts` 和碰撞入口也明确挂在 `Model` 这层：

```python
def contacts(self, collision_pipeline: CollisionPipeline | None = None) -> Contacts:
    if self._collision_pipeline is None:
        self._init_collision_pipeline()
    return self._collision_pipeline.contacts()

def collide(self, state: State, contacts: Contacts | None = None, *, collision_pipeline=None) -> Contacts:
    if self._collision_pipeline is None:
        self._init_collision_pipeline()
    if contacts is None:
        contacts = self._collision_pipeline.contacts()
    self._collision_pipeline.collide(state, contacts)
    return contacts
```

**Verification cues**

- `State` 持有的是 `body_q / body_qd / joint_q / joint_qd / body_f` 这类每步会变的量。
- `Control` 持有的是本步控制输入，不跟 `Model` 永久绑死。
- `Contacts` 是碰撞 pipeline 的结果缓冲区，所以也必须被显式分出来。

**Output passed to next stage**

一条清晰边界：`Model` 负责描述和分配，`State / Control / Contacts` 负责被一轮仿真真正消费。

### Stage 4: 外层 loop 和内层 substep 是分开的两层 orchestration

**Claim**

`examples.run()` 只管 viewer/测试这一层的外循环；一次真正的仿真推进，定义在具体 example 的 `simulate()` 里。

**Why it matters**

这一步最能纠正“是不是 `examples/__init__.py` 在直接做全部仿真”的误解。它管框架，但不管每个例子的具体推进链。

**Source excerpt**

外层统一 run loop：

```python
def run(example, args):
    viewer = example.viewer
    ...
    while viewer.is_running():
        if not viewer.is_paused():
            example.step()
        example.render()
```

但 pendulum 里真正的一次 substep 是这样展开的：

```python
def simulate(self):
    for _ in range(self.sim_substeps):
        self.state_0.clear_forces()

        self.viewer.apply_forces(self.state_0)

        self.model.collide(self.state_0, self.contacts)
        self.solver.step(self.state_0, self.state_1, self.control, self.contacts, self.sim_dt)

        self.state_0, self.state_1 = self.state_1, self.state_0

def step(self):
    if self.graph:
        wp.capture_launch(self.graph)
    else:
        self.simulate()
```

**Verification cues**

- `run()` 只做“什么时候 step、什么时候 render”。
- `simulate()` 才决定 `clear_forces -> collide -> solver.step -> swap` 这条主线。
- `step()` 还能插入 graph replay，说明外层 orchestration 和内层推进链是分开的。

**Output passed to next stage**

一次已经准备好输入的 solver 调用：`solver.step(state_0, state_1, control, contacts, dt)`。

### Stage 5: solver 统一消费同一组对象，再各自决定内部 kernels

**Claim**

从架构上看，solver 的公共边界非常稳定：它接收 `state_in / state_out / control / contacts / dt`，然后用自己的 kernels 推进一步。

**Why it matters**

这一步决定了为什么 chapter 02 可以先讲架构，不必立刻讲完所有 solver 家族。因为它们先共享同一条外部 contract。

**Source excerpt**

`SolverBase` 先定义统一的 `step()` 入口：

```python
def step(
    self, state_in: State, state_out: State, control: Control | None, contacts: Contacts | None, dt: float
) -> None:
    raise NotImplementedError()
```

它也提供一些通用 integration helpers：

```python
if model.body_count:
    wp.launch(
        kernel=integrate_bodies,
        dim=model.body_count,
        inputs=[
            state_in.body_q,
            state_in.body_qd,
            state_in.body_f,
            model.body_com,
            model.body_mass,
            model.body_inertia,
            ...
        ],
        outputs=[state_out.body_q, state_out.body_qd],
    )
```

**Verification cues**

- solver 外部接口始终围绕 `State / Control / Contacts` 这组对象展开。
- helper kernel 的输入里同时有 `state_in` 和 `model`，说明 solver 在消费“当前状态 + 静态模型描述”。
- 具体 solver 可以差很多，但外部 handoff 先是一致的。

**Output passed to next stage**

一条完整的 chapter-02 架构链：example 入口把对象组好，run loop 驱动一次次 `solver.step()`，而 solver 再继续进入自己的内部实现。

## Object Ledger

| 对象 | 谁生产 | 谁消费 | 盯哪些字段 |
|------|--------|--------|------------|
| example short name | `get_examples()` | `main()` | `basic_pendulum -> newton.examples.basic.example_basic_pendulum` |
| `Example` | concrete example module | `examples.run()` | `viewer`、`model`、`state_0/1`、`control`、`contacts`、`solver` |
| `Model` | `builder.finalize()` | `state()`、`control()`、`contacts()`、`collide()`、solvers | 静态 body/joint/shape buffers |
| `State` | `Model.state()` | `collide()`、`solver.step()`、viewer | `body_q`、`body_qd`、`joint_q`、`joint_qd`、forces |
| `Control` | `Model.control()` | `solver.step()` | joint target / actuation buffers |
| `Contacts` | `Model.contacts()` / collision pipeline | `solver.step()`、viewer | contact counts 与 contact arrays |
| `Solver` | example 构造函数 | `Example.simulate()` | `step(state_in, state_out, control, contacts, dt)` |

## Stop Here

读到这里就已经够 chapter 02 的 80-90% 了。

如果你现在能用自己的话讲顺下面这句话，这一章的 beginner 目标就完成了：

```text
examples 短名先路由到具体 Example 模块；
Example.__init__ 组装 Model / State / Control / Contacts / Solver；
run() 驱动外层循环；
simulate() 定义 clear_forces -> collide -> solver.step -> swap 的内层推进；
solver 再继续消费同一组 runtime 对象。
```

这时你已经可以稳定进入 `03_math_geometry`、`04_scene_usd` 或 `05_rigid_articulation`，不会再把 Newton 架构读成一团散乱入口。

## Go Deeper

如果你还想继续精确追源码，再去 `source-walkthrough-deep.md`：

- 想保留所有 file/symbol/line anchors：看 `Fast Deep Index`
- 想逐跳追 `basic_pendulum` 的 exact handoff：看 `Exact Handoff Trace`
- 想知道哪些支线第一遍可以先跳过：看 `Optional Branches`
- 想逐条核对这里的 claim：看 `Verification Anchors`
