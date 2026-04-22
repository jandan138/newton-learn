---
chapter: 02
title: Newton 总体架构
last_updated: 2026-04-22
source_paths:
  - newton/examples/__main__.py
  - newton/examples/__init__.py
  - newton/examples/basic/example_basic_pendulum.py
  - newton/__init__.py
  - newton/_src/sim/model.py
  - newton/_src/sim/state.py
  - newton/_src/solvers/solver.py
paper_keys:
  - mujoco-warp-paper
newton_commit: 1a230702
---

# 02 Newton 总体架构 源码走读

这份主 walkthrough 先回答一个真正的新手问题：当我运行 `python -m newton.examples basic_pendulum` 时，到底是谁把我带到 `Model / State / Control / Solver` 这条四层主干，以及这一拍的 `Contacts` 结果缓冲区？

`01_warp_basics` 已经先回答了“Newton 里为什么会出现 Warp”。到了这一章，问题换成另一种：这些对象到底是谁接起来的，为什么一个 example 不是一个脚本写到底，而是会先经过 short name、launcher、example module、`Example` 对象，再落到真正的 runtime objects。

这份文档故意只跟一条很窄的 core pass：

```text
basic_pendulum short name
-> launcher
-> concrete module
-> Example.__init__()
-> Model / State / Control / Contacts / Solver
-> simulate loop
```

也就是说，这一页不会要求你现在去啃：

- `ModelBuilder.finalize()` 的内部冻结细节
- `capture()` / CUDA graph replay
- `newton/_src/solvers/xpbd/solver_xpbd.py` 里的 XPBD solver internals
- 其它 examples 的横向源码对读

先补一句路径说明：这个文档仓主要存的是学习材料，不是上游 Newton 源码本体。下面引用的 `newton/...` 路径，指的是上游 Newton 源码 checkout 里的对应文件。

如果你想先看更慢一点的概念版，可以再配合 `principle.md`；但只读这一份，也应该能把 chapter 02 的 first-pass 主线讲顺。

## What This Walkthrough Follows

这一页只追下面这条 first-pass handoff：

```text
example short name
-> examples launcher
-> concrete example module
-> Example object
-> public API names
-> Model / State / Control / Solver + Contacts result buffer
-> simulate loop
-> solver.step(...)
```

这一页刻意不展开四类东西：

- `examples/__init__.py` 里 viewer / parser / device plumbing
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
     newton.__init__.py public exports
                    |
                    v
              Example.__init__
                    |
                    v
  builder.finalize() -> Model
 model.state() x2   -> state_0, state_1
 model.control()    -> Control
 eval_fk(...)       -> fill initial body state
 model.contacts()   -> Contacts buffer
  SolverXPBD(model)  -> Solver
                    |
                    v
         examples.run() outer loop
                    |
                    v
            Example.step() frame wrapper
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
    - 想验证什么：Newton 先想让你认哪些公共对象名，以及具体 example 到底在哪一步第一次把运行时栈组出来。
    - 看完后应该能说：`Example.__init__` 是 chapter 02 最关键的组装点，而 `newton/__init__.py` 只是给你一个应该先认得的公共名字目录。
4. 再看 Stage 4。
    - 想验证什么：`Model / State / Control / Solver` 这条四层主干、`Contacts` 结果缓冲区，以及 `q / qd / body_q / joint_q` 这些变量名，各自负责什么。
    - 看完后应该能说：四层主干和 `Contacts` 不是一团近义词，而 `eval_fk(...)` 解释了为什么 solver 前要先把 body state 准备好。
5. 最后看 Stage 5。
    - 想验证什么：这些对象真正怎样被重复消费成一次次 substep，以及三层 `step` 到底是谁管谁。
    - 看完后应该能说：`run()` 管外层时机，`Example.step()` 是 frame wrapper，`simulate()` 定义 `clear_forces -> apply_forces -> collide -> solver.step -> swap` 这条内层推进链。

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

第一遍打开 `newton/examples/__init__.py` 时，不需要从头读到尾；这一章只盯 `get_examples()`、`main()`、`run()` 三段就够了。

以下摘录为教学注释版，注释非原源码。

```python
def get_examples() -> dict[str, str]:
    example_map = {}
    ...
    if filename.startswith("example_") and filename.endswith(".py"):
        example_name = filename[8:-3]  # 从文件名裁出 CLI short name
        example_map[example_name] = f"newton.examples.{module}.{filename[:-3]}"  # short name 指向真实 example module
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

以下摘录为教学注释版，注释非原源码。

```python
from . import main  # __main__ 只做一次薄转发

if __name__ == "__main__":
    main()  # 真正的路由逻辑在 examples.__init__.main()
```

真正的路由发生在 `examples.__init__.main()`：

```python
def main():
    examples = get_examples()  # 先拿到 short name -> module path 映射
    example_name = sys.argv[1]  # CLI 里输入的 short name
    target_module = examples[example_name]  # 把 short name 解析成真实 module
    sys.argv = [target_module, *sys.argv[2:]]  # 剩余参数原样转交给目标模块
    runpy.run_module(target_module, run_name="__main__")  # 正式跳进具体 example
```

如果你总在这里把 `__main__.py`、`__init__.py` 和真正的 example script 混成一层，可以先对照这张图：

![`newton.examples` launcher handoff timeline](assets/02_launcher_handoff.svg)

这里最容易漏掉的是：`__main__` 这层身份会出现两次。

1. 第一次：`python -m newton.examples basic_pendulum` 让 `newton/examples/__main__.py` 以入口模块身份执行。
2. 第二次：`runpy.run_module(target_module, run_name="__main__")` 又把目标 example module 当成新的主程序执行，所以 `example_basic_pendulum.py` 里的 `if __name__ == "__main__":` 也会触发。

所以 `examples.__init__.py` 更像一个路由器：它负责查 short name、改写 `sys.argv`、再把执行权交给真正的 example module；真正创建 `Example(...)` 并进入 `newton.examples.run(...)` 的，还是目标 example 自己。

**Verification cues**

- `__main__.py` 本身没有 builder、solver、state 这些组装逻辑。
- `main()` 最关键的一行是 `target_module = examples[example_name]`。
- `runpy.run_module(...)` 说明 launcher 的工作是“把你送进真正的 example module”，不是自己继续做完整仿真。

**Checkpoint**

如果你现在还觉得“真正的主程序应该就在 `examples/__main__.py`”，先不要继续。这里最重要的动作只是一次跳转。

**Output passed to next stage**

控制权已经落到具体 example module；接下来真正要找的是 module 里的 `Example` 对象。

### Stage 3: concrete `Example` 对象第一次把四层主干和 `Contacts` 结果缓冲区组出来

**Definition**

- `Example`：某个具体场景的一次可运行实例。第一遍先把它读成“这个例子的总装配器”。
- `public API surface`：Newton 刻意先暴露给用户的公共名字目录。第一遍先用它确认“哪些对象值得先认”。
- `Model`：场景的静态描述和 runtime 对象工厂。第一遍先把它读成“世界蓝图 + 生产 state/control/contacts 的地方”。
- `State`：当前时刻会变化的快照，比如位姿、速度、力。
- `Control`：这一拍想施加的输入或目标，比如 joint target、actuation。
- `Contacts`：这一拍碰撞检测写出来的接触结果缓冲区。
- `Solver`：读入 `Model`、当前 `State`、`Control`、`Contacts`，把系统推进到下一拍的执行器。

**Claim**

一旦控制权进入具体 module，`Example.__init__` 就是 chapter 02 最关键的组装点：scene builder、model、solver、states、control、contacts 都在这里第一次接起来。

**Why it matters**

这一步最直接回答了开头那个问题。你真正被“带到四层主干和 `Contacts` 结果缓冲区”的地方，不在顶层 CLI，而在具体 example 的构造阶段。

**Backstory**

Newton 把 scene-specific 组装留在每个 example 自己的 `Example` 里。这样公共 launcher 可以保持很薄，而每个例子仍然能自由决定自己建什么场景、用什么 solver、要不要 viewer graph capture。

**Source excerpt A: 先看 public surface，确认 chapter 02 应该先认哪些名字**

打开 `newton/__init__.py`，你会先看到 Newton 故意 re-export 的一组公共对象：

```python
from ._src.sim import (
    CollisionPipeline,
    Contacts,
    Control,
    Model,
    ModelBuilder,
    State,
    eval_fk,
)

from . import geometry, ik, math, selection, sensors, solvers, usd, utils, viewer
```

这一步的重点不是研究包结构，而是先认出：`ModelBuilder / Model / State / Control / Contacts / eval_fk / solvers` 的确是 Newton 想先暴露给用户的公共名字。它告诉你后面去 `basic_pendulum` 构造函数里，应该优先盯哪些对象，而不是一上来扑向内部目录树。

**Source excerpt B: `basic_pendulum` 的 `Example.__init__()` 是 chapter 02 最小样板**

`basic_pendulum` 的 `Example.__init__()` 基本就是整章最小样板：

下面这段是 **为了 first pass 而截短过的摘录**；你现在不用关心 excerpt 外围的 viewer / capture 支线。

以下摘录为教学注释版，注释非原源码。

```python
builder = newton.ModelBuilder()  # 先创建场景 builder

link_0 = builder.add_link()  # 添加第一个 rigid link
builder.add_shape_box(link_0, hx=hx, hy=hy, hz=hz)  # 给这个 link 绑定盒形几何

link_1 = builder.add_link()  # 添加第二个 rigid link
builder.add_shape_box(link_1, hx=hx, hy=hy, hz=hz)  # 给第二个 link 绑定盒形几何

j0 = builder.add_joint_revolute(...)  # world -> link_0 的转动关节
j1 = builder.add_joint_revolute(...)  # link_0 -> link_1 的转动关节
builder.add_articulation([j0, j1], label="pendulum")  # 把两条关节收成一条 articulated chain
builder.add_ground_plane()  # 补上 ground plane

self.model = builder.finalize()  # builder 冻结成可运行的 Model
self.solver = newton.solvers.SolverXPBD(self.model)  # 给这个 model 选定 solver

self.state_0 = self.model.state()  # 当前状态缓冲区
self.state_1 = self.model.state()  # 下一状态缓冲区
self.control = self.model.control()  # 这一拍输入缓冲区
newton.eval_fk(self.model, self.model.joint_q, self.model.joint_qd, self.state_0)  # 先把 joint-space 初值展开到 body-space state
self.contacts = self.model.contacts()  # 这一拍 contacts 结果缓冲区
```

如果你第一次看 builder 相关代码，先把这些词读成下面这样：

- `link`：一个刚体段或者说一个 rigid body 段。
- `shape`：挂在 link 上的几何 / 碰撞形状。
- `joint`：规定 parent 和 child 怎样相对运动的连接件。
- `articulation`：把一组 joints 收成同一个 articulated system，也就是一条运动链。
- `Model`：builder 冻结之后得到的可运行世界描述。
- `State`：这个世界当前或下一拍的动态快照。
- `Control`：这一拍想喂给 solver 的输入缓冲区。
- `Contacts`：碰撞检测写出来、再交给 solver 消费的结果缓冲区。

`Example.__init__()` 后面源码里还会出现 `self.viewer.set_model(...)` 和 `self.capture()`。第一遍先把它们当 viewer / graph replay 支线，跳过即可；chapter 02 真正的核心已经在上面这段里了。

**Why `eval_fk(...)` sits here**

`eval_fk` = `forward kinematics`。第一遍先把它记成一句话：

```text
把 joint-space 的初始条件，展开成 body-space 的位姿 / 速度快照。
```

也就是说，`joint_q`、`joint_qd` 先描述“关节坐标长什么样”；而 `collide()` 和 solver 更常直接消费的是 body-level state。`eval_fk(...)` 出现在 constructor 里，正是在告诉你：进入第一轮碰撞和求解前，先把 `state_0` 这份 body-space 快照准备好。

**Verification cues**

- `newton/__init__.py` 只是公共名字目录，不是主要运行逻辑所在。
- `builder.add_link()` / `add_shape_box()` / `add_joint_revolute()` 说明场景搭建先发生在 `Example.__init__()` 里。
- `builder.finalize()` 之后，builder 才冻结成真正可运行的 `Model`。
- solver 不是从 CLI 直接冒出来的，而是在 `Example` 里根据 `self.model` 创建。
- 两份 `state()` 明确提示你：Newton 常用双缓冲推进，不是只拿一份状态原地乱改。
- `eval_fk(...)` 明确提示你：solver 前先要把 joint-space 初值展开成 body-space 快照。
- `control()` 和 `contacts()` 也都是在这里先准备好，等后面的 loop 重复消费。

**Checkpoint**

如果你现在还答不上“哪个地方第一次把 `Model / State / Control / Solver` 这条主干，以及 `Contacts` 这个结果缓冲区接到了同一条链上”，先停在这里。第一遍必须先认准 `Example.__init__()` 这个组装点。

**Output passed to next stage**

一整套 runtime stack：`Model / State / Control / Solver` 这条主干，以及与它并列的一份 `Contacts` 结果缓冲区。

### Stage 4: 四层主干 + `Contacts` 结果缓冲区是分工链，不是一团近义词

**Claim**

`Model / State / Control / Solver` 构成的是一次 step 的四层主干；`Contacts` 则是这条主干旁边的并列结果缓冲区。它们不是“都和仿真有关”的一团大名词，而是沿着一次 step 明确分工的不同角色。

**Why it matters**

如果这一步边界没分开，后面看 `03_math_geometry`、`05_rigid_articulation`、`07_constraints_contacts_math` 时，你会不断把“静态描述”“当前状态”“这一拍输入”“碰撞结果”“求解器职责”搅在一起。

**Variable name cheat sheet**

第一次看 Newton 里的变量名时，先把这些缩写翻译成下面这样：

| 变量 | 第一遍先怎么读 |
|------|----------------|
| `q` | generalized position coordinate，也就是位置 / 角度坐标 |
| `qd` | `q` 的时间导数，也就是速度坐标 |
| `body_q` | body-space 位姿快照 |
| `body_qd` | body-space 速度快照 |
| `joint_q` | joint-space 位置 / 角度快照 |
| `joint_qd` | joint-space 速度快照 |
| `joint_f` | joint-space 力 / 力矩输入缓冲区，属于 `Control` |
| `body_f` | 这一拍积累到刚体上的外力 / 力矩缓冲区 |

所以 `eval_fk(self.model, self.model.joint_q, self.model.joint_qd, self.state_0)` 可以先读成：

```text
把 model 里的 joint_q / joint_qd，展开写进 state_0 里的 body_q / body_qd。
```

**How to read the five names**

- `Model` 回答“世界里有什么、怎样连着、默认量是什么”，并负责分配 runtime 对象。
- `State` 回答“当前这一拍系统长什么样”。
- `Control` 回答“这一拍我要给它什么输入或目标”。
- `Contacts` 回答“按照当前 `State` 做完碰撞后，这一拍接触结果是什么”。第一遍你可以直接把它当成 contacts result buffer 来读。
- `Solver` 回答“拿着上面这些输入，怎样算出下一拍”。

**Source excerpt**

`Model` 自己把 runtime 对象显式分出去：

这两段更像 runtime object contract，先保持干净代码，重点用块后解释去看 `State / Control / Contacts / Solver` 的边界。

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

这段代码最值得带走的不是每个字段名，而是它在架构上的分工：

- `Model.state()` 会产出一份新的动态快照。
- `Model.control()` 会产出一份新的输入缓冲区。
- 这两者都不是 solver 私有内部变量，而是 example 明确拿在手里的 runtime objects。

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

如果你现在还会把 `Model` 说成“当前时刻的状态”，或者把 `Contacts` 说成“solver 内部自己顺手算的东西”，先不要继续。chapter 02 的核心，就是先把四层主干和 `Contacts` 结果缓冲区拆开。

**Output passed to next stage**

一条清晰边界：`Example` 先把四层主干和 `Contacts` 结果缓冲区准备好，后面的 loop 再重复消费它们。

### Stage 5: `simulate()` 把这些对象真的接成一轮仿真

**Definition**

`simulate loop`：具体 example 里那条会被反复执行的 substep 链。第一遍先把它读成“已经准备好的对象，在这一拍按什么顺序被消费”。

**Claim**

`examples.run()` 管的是外层时机；真正把 `State / Control / Contacts / Solver` 接成一轮 physics step 的，是 example 自己的 `simulate()`。

**Why it matters**

只有走到这里，开头那句“谁把我带到这些对象”才算完整。前面四个 stage 是组装和分工，这个 stage 才是运行时真正发生 handoff 的地方。

**Three Step Levels**

这里最容易混的是：Newton 在 chapter 02 里其实同时出现了三层 `step`。

| 你看到的名字 | 第一遍先怎么读 |
|------|----------------|
| `examples.run()` | outer timing loop，决定何时让 example 前进一步 |
| `Example.step()` | frame-level wrapper，决定这一帧是直接 `simulate()` 还是走 graph replay |
| `Solver.step()` | physics solver contract，真正消费 `state_in / state_out / control / contacts / dt` |

如果你脑子里把这三者压成一个“step”，后面就会很容易觉得架构层次在打架。第一遍先把它们拆开：`run()` 负责调度，`Example.step()` 负责包装，`Solver.step()` 负责物理推进。

而 `simulate()` 不算第四层顶级 step。它更准确的角色是：`Example.step()` 里面那条真正展开 substep 顺序的内层链。

**Source excerpt**

外层统一 run loop 只决定什么时候 `step()`：

以下摘录为教学注释版，注释非原源码。

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
        self.state_0.clear_forces()  # 清空上一拍残留的外力缓冲区

        self.viewer.apply_forces(self.state_0)  # 把这一拍外部作用写进当前 state

        self.model.collide(self.state_0, self.contacts)  # 用当前 state 填充 contacts 结果缓冲区
        self.solver.step(self.state_0, self.state_1, self.control, self.contacts, self.sim_dt)
        # solver 读 state_0，写 state_1

        self.state_0, self.state_1 = self.state_1, self.state_0  # 交换双缓冲角色

def step(self):
    if self.graph:
        wp.capture_launch(self.graph)  # first pass: 把它当优化支线，先跳过
    else:
        self.simulate()
```

这段代码里最值得你现在就讲顺的是双缓冲语义：

```text
state_0 = 当前状态（solver 读它）
state_1 = 下一状态（solver 写它）
substep 结束后交换名字
下一轮就继续读新的 state_0
```

所以最后那句 swap 不是“随手换个变量名”，而是在表达：刚才算出来的下一拍，现在正式成为当前拍。

而 `step()` 里的 `graph` 分支，这一遍只需要知道它是一个性能 / replay 支线。你现在可以把 `Example.step()` 简化理解成：

```text
这一帧最终会去执行 simulate() 这条 substep 链。
```

**Verification cues**

- `run()` 管的是 viewer 生命周期和调用时机，不是具体 physics 链条。
- `Example.step()` 不是 solver 本体，而是 example 层的 frame wrapper。
- `clear_forces()` 操作的是当前 `State` 里的 force buffers。
- `collide()` 读当前 `State`，把这一拍的结果写进 `Contacts`。
- `solver.step(...)` 再统一消费 `state_0 / state_1 / control / contacts / dt`。
- 最后一行 swap 说明双缓冲 state 会在每个 substep 之间交换角色。

**Checkpoint**

如果你现在还会说“我运行 example 以后，CLI 直接把我送进 solver”，先停一下。更准确的说法应该是：short name 把你送进 module，module 里的 `Example` 先组对象，`Example.step()` 再包装一次这一帧的推进，而 `simulate()` 才真正把这些对象交给 `Solver.step()`。

**Output passed to next stage**

一条完整的 chapter-02 主链：example short name -> launcher -> concrete module -> `Example` -> `Model / State / Control / Solver` 四层主干 + `Contacts` 结果缓冲区 -> `Example.step()` -> `simulate()` -> `clear_forces -> apply_forces -> collide -> solver.step -> swap`。

## Object Ledger

| 对象 | 谁生产 | 谁消费 | 第一遍盯什么 |
|------|--------|--------|--------------|
| example short name | `get_examples()` 的 key | `main()` | `basic_pendulum` 先只是 CLI 路由名 |
| target example module | `get_examples()` 的 value | `runpy.run_module()` | `newton.examples.basic.example_basic_pendulum` 才是被执行的文件 |
| `Example` | concrete example module | `examples.run()` | `model`、`state_0/1`、`control`、`contacts`、`solver` |
| `Model` | `builder.finalize()` | `state()`、`control()`、`contacts()`、`collide()`、solver | 静态 body/joint/shape buffers 与工厂边界 |
| `State` | `Model.state()` | `collide()`、`solver.step()`、viewer | 当前快照：`body_q`、`body_qd`、`joint_q`、`joint_qd`、forces |
| `Control` | `Model.control()` | `solver.step()` | 这一拍输入：joint target / actuation buffers |
| `eval_fk(...)` | public API helper | `Example.__init__()` | 把 joint-space 初值展开成 body-space state |
| `Contacts` | `Model.contacts()` / collision pipeline | `solver.step()`、viewer | 这一拍接触结果缓冲区 |
| `Solver` | example 构造函数 | `Example.simulate()` | `step(state_in, state_out, control, contacts, dt)` |

## Stop Here

读到这里就已经够 chapter 02 的 80-90% 了。第一遍不需要再去啃 `finalize()` 内部、XPBD 内部和 viewer plumbing。

如果你现在能用自己的话讲顺下面这句话，这一章的 beginner 目标就完成了：

```text
我先输入 example short name；
launcher 把它解析成真实 example module；
module 里的 `Example.__init__()` 组出 `Model / State / Control / Solver` 四层主干，并补上 `Contacts` 结果缓冲区；
`eval_fk(...)` 先把 joint-space 初值展开成 body-space state；
`run()` 负责外层时机，`Example.step()` 负责 frame wrapper；
`simulate()` 是 `Example.step()` 里面那条 `clear_forces -> apply_forces -> collide -> solver.step -> swap` 的内层推进链；
`Solver.step()` 则是真正的 physics contract。
```

如果你现在还答不上下面任一题，就先不要急着进入 deep walkthrough：

- `basic_pendulum` 一开始到底是什么，不是什么。
- `examples/__main__.py` 的 job 到底是什么。
- `Example.__init__()` 为什么是 chapter 02 最关键的 handoff 点。
- `eval_fk(...)` 为什么会在 constructor 里出现。
- `Model / State / Control / Solver` 四层主干和 `Contacts` 结果缓冲区为什么不能混成一团。
- `run()`、`Example.step()`、`simulate()` 和 `Solver.step()` 之间到底是谁包着谁。

这几件事能答顺，你就已经带着稳定主线进入 `03_math_geometry`、`04_scene_usd` 或 `05_rigid_articulation` 了。

## Go Deeper

如果你还想继续精确追源码，再去 `source-walkthrough-deep.md`：

- 想保留所有 file/symbol/line anchors：看 `Fast Deep Index`
- 想逐跳追 `basic_pendulum` 的 exact handoff：看 `Exact Handoff Trace`
- 想知道哪些支线第一遍可以先跳过：看 `Optional Branches`
- 想逐条核对这里的 claim：看 `Verification Anchors`
