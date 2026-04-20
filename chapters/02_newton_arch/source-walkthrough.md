---
chapter: 02
title: Newton 总体架构
last_updated: 2026-04-20
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

这份主 walkthrough 先回答一个真正的新手问题：当我运行 `python -m newton.examples basic_pendulum` 时，到底是谁把我带到 `Model / State / Control / Solver` 这条四层主干，以及与它并列的 `Contacts` handoff object？

`01_warp_basics` 已经先回答了“Newton 里为什么会出现 Warp”。到了这一章，问题换成了另一种：那些对象到底是谁接起来的，为什么一个 example 不是一个脚本写到底，而是会先经过 short name、launcher、example module、`Example` 对象，再落到真正的 runtime objects。

Newton 故意把这几层拆开：用户只输入一个 example short name；launcher 负责把它送到真实 module；具体 module 里的 `Example` 对象负责组四层主干和 `Contacts` handoff；`simulate()` 再把这些对象反复串成一次次 substep。后面整份文件都会沿这条单线往下走。

如果你想先看更慢一点的概念版，可以再配合 `principle.md`；但只读这一份，也应该能把 chapter 02 的主线讲顺。

## What This Walkthrough Follows

这一页只追下面这条 handoff：

```text
example short name
-> examples launcher
-> concrete example module
-> Example object
-> Model / State / Control / Solver + Contacts handoff
-> simulate loop
-> solver.step(...)
```

这一页刻意不展开三类东西：

- `builder.finalize()` 的全部内部细节
- XPBD 或其它 solver 的完整数学推导
- 8 个 solver family 的横向比较

第一遍先守住一句话：chapter 02 真正讲的不是“有哪些对象名”，而是 **一个 example 怎么被送到这些对象，再被这些对象推进起来**。

## One-Screen Chapter Map

```text
python -m newton.examples basic_pendulum
                    |
                    v
        example short name: basic_pendulum
                    |
                    v
examples/__main__.py -> examples.__init__.main()
                    |
                    v
target module: newton.examples.basic.example_basic_pendulum
                    |
                    v
              Example.__init__
                    |
                    v
 builder.finalize() -> Model
 model.state()      -> State
 model.control()    -> Control
 model.contacts()   -> Contacts
 SolverXPBD(model)  -> Solver
                    |
                    v
         examples.run() outer loop
                    |
                    v
          Example.simulate() inner loop
                    |
                    v
 clear_forces -> apply_forces -> collide -> solver.step -> swap state
```

## Beginner Path

1. 先看 Stage 1。
   - 想验证什么：`basic_pendulum` 这种名字一开始到底是什么。
   - 看完后应该能说：它先只是一个 CLI short name，不是已经开始跑的 `Example` 对象。
2. 再看 Stage 2。
   - 想验证什么：launcher 到底做了什么，哪里才是具体 example module。
   - 看完后应该能说：`examples/__main__.py` 很薄，真正的 handoff 是把 short name 解析成真实模块路径并跳过去。
3. 再看 Stage 3。
   - 想验证什么：具体 example 到底在哪一步第一次把运行时栈组出来。
   - 看完后应该能说：`Example.__init__` 是 chapter 02 最关键的组装点。
4. 再看 Stage 4。
   - 想验证什么：`Model / State / Control / Solver` 这条四层主干，以及 `Contacts` 这个并列 handoff object，各自负责什么。
   - 看完后应该能说：四层主干和 `Contacts` 不是一团近义词，而是不同的 job。
5. 最后看 Stage 5。
   - 想验证什么：这些对象真正怎样被重复消费成一次次 substep。
   - 看完后应该能说：`run()` 管外层时机，`simulate()` 定义 `clear_forces -> apply_forces -> collide -> solver.step -> swap` 这条内层推进链。

## Main Walkthrough

### Stage 1: `basic_pendulum` 先只是一个 example short name

**Definition**

`example short name`：你在命令行里输入的简写标签。它的工作只是“选中哪个例子”，还不是最终执行的 Python 对象。

**Claim**

`basic_pendulum` 这种名字首先只是路由 key，不是 `Example` 类本身，也不是 solver 本身。

**Why it matters**

如果第一步就把 short name 当成“例子已经开始跑了”，后面就很容易把 launcher、example module 和 `Example` 对象混成一层。

**Backstory**

Newton 有很多 examples。让用户直接输入一长串模块路径当然也能工作，但学习入口会非常重。short name 的意义，就是先把“我要看哪个例子”说清楚，再把真正的 Python 路径留给内部去解析。

**Source excerpt**

`examples.__init__` 先把 `example_*.py` 文件整理成 `short name -> module path` 的映射：

```python
def get_examples() -> dict[str, str]:
    example_map = {}
    ...
    if filename.startswith("example_") and filename.endswith(".py"):
        example_name = filename[8:-3]
        example_map[example_name] = f"newton.examples.{module}.{filename[:-3]}"
```

**Verification cues**

- `example_name = filename[8:-3]` 说明 `basic_pendulum` 是从文件名裁出来的简写。
- `example_map[...]` 的 value 是完整模块路径，不是 `Example` 实例。
- 所以你在 CLI 里输入的第一层信息，其实只是“我要去哪个 example module”。

**Checkpoint**

如果你现在还会把 `basic_pendulum` 想成一个已经创建好的对象，先不要继续。第一遍只把它记成“选例子的短名”就够了。

**Output passed to next stage**

一个真实目标：`basic_pendulum` 这个 short name 对应到某个具体 example module。

### Stage 2: launcher 把 short name 送到真实 example module

**Definition**

- `launcher`：负责解析 short name 并把控制权交出去的薄入口。
- `example module`：真正会被执行的 Python 文件；里面才会定义具体 `Example`。

**Claim**

`examples/__main__.py` 和 `examples.__init__.main()` 的职责是 handoff，不是把全部仿真逻辑塞在顶层。

**Why it matters**

很多新手会本能地把 `python -m ...` 的入口文件当成“总 orchestrator”。chapter 02 最先要纠正的，就是这个误解。

**Backstory**

这层分开以后，Newton 才能复用同一个 launcher 去跑很多不同 examples；而每个 example 自己的 scene、solver 选择和 runtime 组装逻辑，继续留在各自 module 里。

**Source excerpt**

`__main__.py` 自己非常薄：

```python
from . import main

if __name__ == "__main__":
    main()
```

真正的路由发生在 `examples.__init__.main()`：

```python
def main():
    examples = get_examples()
    example_name = sys.argv[1]
    target_module = examples[example_name]
    sys.argv = [target_module, *sys.argv[2:]]
    runpy.run_module(target_module, run_name="__main__")
```

**Verification cues**

- `__main__.py` 本身没有 builder、solver、state 这些组装逻辑。
- `main()` 最关键的一行是 `target_module = examples[example_name]`。
- `runpy.run_module(...)` 说明 launcher 的工作是“把你送进真正的 example module”，不是自己继续做完整仿真。

**Checkpoint**

如果你现在还觉得“真正的主程序应该就在 `examples/__main__.py`”，先不要继续。这里最重要的动作只是一次跳转。

**Output passed to next stage**

控制权已经落到具体 example module；接下来真正要找的是 module 里的 `Example` 对象。

### Stage 3: concrete `Example` 对象第一次把四层主干和 `Contacts` handoff 组出来

**Definition**

- `Example`：某个具体场景的一次可运行实例。第一遍先把它读成“这个例子的总装配器”。
- `Model`：场景的静态描述和 runtime 对象工厂。第一遍先把它读成“世界蓝图 + 生产 state/control/contacts 的地方”。
- `State`：当前时刻会变化的快照，比如位姿、速度、力。
- `Control`：这一拍想施加的输入或目标，比如 joint target、actuation。
- `Contacts`：这一拍碰撞检测写出来的接触结果缓冲区。
- `Solver`：读入 `Model`、当前 `State`、`Control`、`Contacts`，把系统推进到下一拍的执行器。

**Claim**

一旦控制权进入具体 module，`Example.__init__` 就是 chapter 02 最关键的 handoff 点：scene builder、model、solver、states、control、contacts 都在这里第一次接起来。

**Why it matters**

这一步最直接回答了开头那个问题。你真正被“带到四层主干和 `Contacts` handoff”的地方，不在顶层 CLI，而在具体 example 的构造阶段。

**Backstory**

Newton 把 scene-specific 组装留在每个 example 自己的 `Example` 里。这样公共 launcher 可以保持很薄，而每个例子仍然能自由决定自己建什么场景、用什么 solver、要不要 viewer graph capture。

**Source excerpt**

`basic_pendulum` 的 `Example.__init__` 基本就是整章最小样板：

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

- `builder.finalize()` 之后，builder 才冻结成真正可运行的 `Model`。
- solver 不是从 CLI 直接冒出来的，而是在 `Example` 里根据 `self.model` 创建。
- 两份 `state()` 明确提示你：Newton 常用双缓冲推进，不是只拿一份状态原地乱改。
- `control()` 和 `contacts()` 也都是在这里先准备好，等后面的 loop 重复消费。

**Checkpoint**

如果你现在还答不上“哪个地方第一次把 `Model / State / Control / Solver` 这条主干，以及 `Contacts` 这个 handoff object 接到了同一条链上”，先停在这里。第一遍必须先认准 `Example.__init__` 这个组装点。

**Output passed to next stage**

一整套 runtime stack：`Model / State / Control / Solver` 这条主干，以及与它并列的一份 `Contacts` handoff object。

### Stage 4: 四层主干 + `Contacts` handoff 是分工链，不是一团近义词

**Claim**

`Model / State / Control / Solver` 构成的是一次 step 的四层主干；`Contacts` 则是这条主干旁边的并列 handoff object。它们不是“都和仿真有关”的一团大名词，而是沿着一次 step 明确分工的不同角色。

**Why it matters**

如果这一步边界没分开，后面看 `03_math_geometry`、`05_rigid_articulation`、`07_constraints_contacts_math` 时，你会不断把“静态描述”“当前状态”“这一拍输入”“碰撞结果”“求解器职责”搅在一起。

**How to read the five names**

- `Model` 回答“世界里有什么、怎样连着、默认量是什么”，并负责分配 runtime 对象。
- `State` 回答“当前这一拍系统长什么样”。
- `Control` 回答“这一拍我要给它什么输入或目标”。
- `Contacts` 回答“按照当前 `State` 做完碰撞后，这一拍接触结果是什么”。
- `Solver` 回答“拿着上面这些输入，怎样算出下一拍”。

**Source excerpt**

`Model` 自己把 runtime 对象显式分出去：

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

碰撞和 contacts 也不是 solver 私有黑盒，而是 `Model` 这边就有明确边界：

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

而 solver 的公共边界也故意写得很稳定：

```python
def step(
    self, state_in: State, state_out: State, control: Control | None, contacts: Contacts | None, dt: float
) -> None:
    raise NotImplementedError()
```

**Verification cues**

- `State` 里是 `body_q / body_qd / joint_q / joint_qd / body_f` 这类每步会变的量。
- `Control` 明确是一拍的输入缓冲区，不等于系统当前状态。
- `Contacts` 由 collision pipeline 填充，所以它也需要作为独立对象存在。
- `Solver.step(...)` 统一消费 `state_in / state_out / control / contacts / dt`，说明 solver 的 job 是“推进”，不是偷偷替你管理全部对象生命周期。

**Checkpoint**

如果你现在还会把 `Model` 说成“当前时刻的状态”，或者把 `Contacts` 说成“solver 内部自己顺手算的东西”，先不要继续。chapter 02 的核心，就是先把四层主干和 `Contacts` handoff 拆开。

**Output passed to next stage**

一条清晰边界：`Example` 先把四层主干和 `Contacts` handoff 准备好，后面的 loop 再重复消费它们。

### Stage 5: `simulate()` 把这些对象真的接成一轮仿真

**Definition**

`simulate loop`：具体 example 里那条会被反复执行的 substep 链。第一遍先把它读成“已经准备好的对象，在这一拍按什么顺序被消费”。

**Claim**

`examples.run()` 管的是外层时机；真正把 `State / Control / Contacts / Solver` 接成一轮 physics step 的，是 example 自己的 `simulate()`。

**Why it matters**

只有走到这里，开头那句“谁把我带到这些对象”才算完整。前面四个 stage 是组装和分工，这个 stage 才是运行时真正发生 handoff 的地方。

**Source excerpt**

外层统一 run loop 只决定什么时候 `step()`：

```python
def run(example, args):
    viewer = example.viewer
    ...
    while viewer.is_running():
        if not viewer.is_paused():
            example.step()
        example.render()
```

但 `basic_pendulum` 里真正的一次 substep 是这样展开的：

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

- `run()` 管的是 viewer 生命周期和调用时机，不是具体 physics 链条。
- `clear_forces()` 操作的是当前 `State` 里的 force buffers。
- `collide()` 读当前 `State`，把这一拍的结果写进 `Contacts`。
- `solver.step(...)` 再统一消费 `state_0 / state_1 / control / contacts / dt`。
- 最后一行 swap 说明双缓冲 state 会在每个 substep 之间交换角色。

**Checkpoint**

如果你现在还会说“我运行 example 以后，CLI 直接把我送进 solver”，先停一下。更准确的说法应该是：short name 把你送进 module，module 里的 `Example` 先组对象，`simulate()` 才把这些对象真的交给 solver。

**Output passed to next stage**

一条完整的 chapter-02 主链：example short name -> launcher -> concrete module -> `Example` -> `Model / State / Control / Solver` 四层主干 + `Contacts` handoff -> `simulate()` -> `clear_forces -> apply_forces -> collide -> solver.step -> swap`。

## Object Ledger

| 对象 | 谁生产 | 谁消费 | 第一遍盯什么 |
|------|--------|--------|--------------|
| example short name | `get_examples()` 的 key | `main()` | `basic_pendulum` 先只是 CLI 路由名 |
| target example module | `get_examples()` 的 value | `runpy.run_module()` | `newton.examples.basic.example_basic_pendulum` 才是被执行的文件 |
| `Example` | concrete example module | `examples.run()` | `model`、`state_0/1`、`control`、`contacts`、`solver` |
| `Model` | `builder.finalize()` | `state()`、`control()`、`contacts()`、`collide()`、solver | 静态 body/joint/shape buffers 与工厂边界 |
| `State` | `Model.state()` | `collide()`、`solver.step()`、viewer | 当前快照：`body_q`、`body_qd`、`joint_q`、`joint_qd`、forces |
| `Control` | `Model.control()` | `solver.step()` | 这一拍输入：joint target / actuation buffers |
| `Contacts` | `Model.contacts()` / collision pipeline | `solver.step()`、viewer | 这一拍接触结果缓冲区 |
| `Solver` | example 构造函数 | `Example.simulate()` | `step(state_in, state_out, control, contacts, dt)` |

## Stop Here

读到这里就已经够 chapter 02 的 80-90% 了。

如果你现在能用自己的话讲顺下面这句话，这一章的 beginner 目标就完成了：

```text
我先输入 example short name；
launcher 把它解析成真实 example module；
module 里的 Example 对象组出 `Model / State / Control / Solver` 四层主干，并补上 `Contacts` handoff；
run() 负责外层时机；
simulate() 负责 clear_forces -> apply_forces -> collide -> solver.step -> swap 这条内层推进链。
```

如果你现在还答不上下面任一题，就先不要急着进入 deep walkthrough：

- `basic_pendulum` 一开始到底是什么，不是什么。
- `examples/__main__.py` 的 job 到底是什么。
- `Example.__init__` 为什么是 chapter 02 最关键的 handoff 点。
- `Model / State / Control / Solver` 四层主干和 `Contacts` handoff 为什么不能混成一团。
- `run()` 和 `simulate()` 分别在管哪一层循环。

这几件事能答顺，你就已经带着稳定主线进入 `03_math_geometry`、`04_scene_usd` 或 `05_rigid_articulation` 了。

## Go Deeper

如果你还想继续精确追源码，再去 `source-walkthrough-deep.md`：

- 想保留所有 file/symbol/line anchors：看 `Fast Deep Index`
- 想逐跳追 `basic_pendulum` 的 exact handoff：看 `Exact Handoff Trace`
- 想知道哪些支线第一遍可以先跳过：看 `Optional Branches`
- 想逐条核对这里的 claim：看 `Verification Anchors`
