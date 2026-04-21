# Chapter 02 Core Pass Rewrite Design

- **日期**: 2026-04-21
- **仓库**: `/home/zhuzihou/dev/newton-learn`
- **范围**:
  - `chapters/02_newton_arch/README.md`
  - `chapters/02_newton_arch/source-walkthrough.md`
- **参考但不重写**:
  - `chapters/02_newton_arch/principle.md`
  - `chapters/02_newton_arch/examples.md`
  - `chapters/02_newton_arch/source-walkthrough-deep.md`
- **目标**: 把 chapter 02 收窄成一个更稳的 beginner core pass，让读者明确知道这章该看哪几段源码、哪几段先不要看，并把 `basic_pendulum` 的最小执行链讲到能逐行跟住。

---

## 1. 这次重写真正要解决什么

当前 chapter 02 的主线方向其实没有错：

```text
example short name
-> launcher
-> concrete example module
-> Example
-> Model / State / Control / Solver + Contacts
-> simulate loop
```

问题不是“章节主线错了”，而是两件事同时发生：

1. core pass 里真正该学的源码还没有解释到新手能稳稳跟住。
2. first pass 本来就不该展开的源码支线，没有被足够明确地挡在外面。

所以这轮不是扩写成更大的源码 tour，而是反过来做两件更重要的事：

- 先把 `02` 的必学源码范围收窄。
- 再把保留下来的那条 core path 讲得更具体、更逐行。

---

## 2. Chapter 02 的推荐 core pass

这章 first pass 应该只要求读者稳定追完下面这些锚点：

1. `newton/examples/__main__.py`
   - 结论：入口文件极薄，只负责转发。
2. `newton/examples/__init__.py`
   - 只看 `get_examples()`、`main()`、`run()`。
   - 结论：short name 怎样跳到真实 module，outer loop 怎样调用 `example.step()`。
3. `newton/examples/basic/example_basic_pendulum.py`
   - 只看 `Example.__init__()`、`simulate()`、`step()`。
   - 结论：chapter 02 的 runtime stack 在哪里第一次被组装出来。
4. `newton/__init__.py`
   - 只看 public exports。
   - 结论：Newton 公开给用户的核心名字有哪些。
5. `newton/_src/sim/model.py`
   - 只看 `Model.state()`、`Model.control()`、`Model.contacts()`、`Model.collide()`。
   - 结论：`Model` 怎样产出 runtime objects。
6. `newton/_src/solvers/solver.py`
   - 只看 `SolverBase.step(...)` 的统一签名。
   - 结论：solver 在架构上统一吃哪些输入。

---

## 3. 明确要 defer 的内容

这轮必须明确告诉读者：下面这些内容不是 chapter 02 first pass 的必修内容。

- `newton/examples/__init__.py` 里的 viewer/device/parser/plumbing 分支
- `example_basic_pendulum.py` 的 `capture()`、`render()`、`test_final()`
- `newton/_src/sim/builder.py` 里 `ModelBuilder.finalize()` 的内部冻结细节
- `newton/_src/sim/state.py` 的完整字段手册
- `newton/_src/solvers/xpbd/solver_xpbd.py` 的 solver internals
- 其它 examples 的源码对读

也就是说，chapter 02 不是“把 Newton 架构全部吃完”，而是：

```text
先把一个 example 怎样被送进 runtime objects，
再怎样进入一轮 substep，讲顺。
```

---

## 4. README 应该承担什么新职责

`README.md` 这轮不能只做目录导航，还要多承担一个 beginner 保护作用：

1. 把 `Core Pass` 和 `Defer For Now` 明确分栏。
2. 加一条 `Source Note`，明确说明文中的 `newton/...` 路径指向上游 Newton 源码 checkout，而不是 `newton-learn` 仓本身。
3. 把阅读顺序改成更聚焦的版本：

```text
principle.md
-> source-walkthrough.md
-> 运行 basic_pendulum
-> examples.md
-> source-walkthrough-deep.md
```

4. 在完成门槛里加入更具体的自检：
   - 我能指出 `Example.__init__()` 是组装点。
   - 我能区分 `run()`、`Example.step()`、`Solver.step()` 三层 step。
   - 我能解释为什么 `state_0 / state_1` 会交换角色。

---

## 5. Main Walkthrough 应该怎样重写

### 5.1 主目标

主 walkthrough 不该再只是“故事线更顺”，而要做到：

- 每个关键代码块都更像逐行解说，而不是只给 excerpt。
- 每个新手最容易卡住的名词，在第一次出现时就给最够用解释。
- 明确告诉读者“现在先打开哪个文件，看哪几行，别看哪里”。

### 5.2 必须补上的四个解释块

1. **Public surface 小摘录**
   - 从 `newton/__init__.py` 贴出 `ModelBuilder / Model / State / Control / Contacts / eval_fk / solvers`。
   - 作用：把“读 public API”从抽象建议变成看得见的代码证据。

2. **`Example.__init__()` 逐行解说**
   - 明确解释：
     - `add_link()` 是加刚体 link
     - `add_shape_box()` 是给 link 绑定几何/碰撞形状
     - `add_joint_revolute()` 是加转动关节
     - `add_articulation()` 是把关节收成一条运动链
     - `builder.finalize()` 才产出可运行 `Model`

3. **变量名和 `eval_fk(...)` 解释块**
   - 必须给出 `q / qd / body_q / body_qd / joint_q / joint_qd / body_f` 的 cheat sheet。
   - 必须解释 `eval_fk(...)`：
     - forward kinematics
     - 它把 joint-space 初始条件展开到 body-space state
     - 它解释了为什么 solver 前先要把 `state_0` 准备好

4. **三层 step + 双缓冲解释块**
   - `run()`：outer viewer loop
   - `Example.step()`：frame wrapper
   - `Solver.step()`：physics solver contract
   - `state_0 / state_1`：当前状态和下一状态的双缓冲交换

### 5.3 主线上的措辞要求

这轮 main walkthrough 应优先用更直接的 beginner-safe 词：

- `contacts result buffer` 可以先于 `handoff object`
- `which file calls which` 可以先于 `runtime stack`
- 先说明“这行代码把什么接到什么”，再提升到抽象名词

---

## 6. 这轮不做什么

1. 不重写 `principle.md`
2. 不重写 `examples.md`
3. 不展开 `source-walkthrough-deep.md`
4. 不把 `02` 改成 line-range-heavy 的 deep 文档
5. 不进入 XPBD 数学和 `finalize()` internals

---

## 7. 成功标准

如果这轮做对了，读者在 chapter 02 读完后应能稳定说出下面这段话：

```text
我在 CLI 里先输入一个 example short name；
launcher 把它跳转到真实 example module；
`Example.__init__()` 把 Model、两份 State、Control、Contacts、Solver 组出来；
`eval_fk(...)` 先把 joint-space 初值展开到 body-space state；
然后 `simulate()` 用 clear_forces -> apply_forces -> collide -> solver.step -> swap 这条链推进一轮 substep。
```

同时，读者也应明确知道：

```text
现在先不要去啃 finalize() 内部、XPBD 内部和 viewer plumbing。
```

并且不会再卡在一个更基础的问题上：

```text
这些 `newton/...` 路径到底是在这个文档仓里找，还是去上游 Newton 源码仓里找？
```
