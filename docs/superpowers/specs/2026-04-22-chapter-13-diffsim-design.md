# Chapter 13 DiffSim Design

- **日期**: 2026-04-22
- **仓库**: `/home/zhuzihou/dev/newton-learn`
- **范围**:
  - `chapters/13_diffsim/README.md`
  - `chapters/13_diffsim/principle.md`
  - `chapters/13_diffsim/examples.md`
  - `chapters/13_diffsim/source-walkthrough.md`
  - `chapters/13_diffsim/pitfalls.md`
  - `chapters/13_diffsim/exercises.md`
- **目标**: 把 chapter 13 建成一章真正 teaching-first 的 `verify before optimize` 章节，让读者先建立 “forward sim -> loss -> backward -> FD validation -> update loop” 这条最小可微信任闭环，再分清 state parameter、model parameter 和 external parameter buffer 三类优化对象。

---

## 1. 这章真正要回答什么

chapter 12 刚把一个重要问题讲顺：

```text
状态既然已经存在，
我怎样读它？又怎样朝目标去改它？
```

chapter 13 的下一问不是“再学一种 solver”，而是：

```text
一次 rollout 没打中目标时，
我到底该改哪个上游参数，梯度会出现在哪里，
又怎么知道这个梯度可信？
```

这就是 chapter 13 的主问题。

也就是说，本章的 teaching spine 不应是：

- diffsim 例子目录
- optimizer 目录
- autodiff 理论目录

而应该是：

```text
forward sim
-> scalar loss
-> wp.Tape backward
-> gradient lands on an upstream parameter
-> FD validation checks trust
-> update loop uses that gradient
```

---

## 2. 推荐主线：`verify before optimize`

这章最重要的 framing 不是 “Newton 也能反传”，而是：

```text
在 Newton / Warp 里，
可微分仿真首先是一条要先验证、再优化的闭环。
```

所以 first pass 必须把 `conventions/diffsim-validation.md` 当成本章主线的一部分，而不是附录。

核心结论应该收敛成：

1. `requires_grad=True` 先决定哪些模型 / 状态 / buffer 会参加梯度追踪。
2. `wp.Tape()` 记录 forward 里的 kernel 执行，并在 backward 时回传 adjoint。
3. 梯度最后会落在某个你真正想改的 upstream parameter 上。
4. `FD` 不是可选 debug 技巧，而是这章第一条硬验证门槛。
5. 只有通过可信区间验证，update loop 才有意义。

---

## 3. 推荐比较轴：按“参数放在哪里”来讲，不按 solver family 来讲

chapter 13 最稳的教学轴不是 solver family，而是：

```text
我要优化的参数到底放在哪里？
```

推荐三类：

1. `state parameter`
   - 例：初始速度、初始位置、某一拍状态量
2. `model parameter`
   - 例：弹簧静长、结构参数、几何/物理配置中的持久量
3. `external parameter buffer`
   - 例：材料参数先存在外部 buffer，每次 forward 时再映射进 model

这个比较轴的价值很高，因为它能把 diffsim 读成：

```text
梯度最终会落到哪一层对象上。
```

而不是把 chapter 13 写成“ball / cage / softbody / drone 四个 demo 平铺”。

---

## 4. 推荐例子分工

### 4.1 主锚点：`example_diffsim_ball.py`

- 角色：first-pass main anchor
- job：讲清最小闭环

```text
initial velocity
-> rollout
-> final-state loss
-> tape.backward
-> gradient on initial velocity
-> FD check
-> gradient descent step
```

它最适合作为第一例，不是因为它代表全部 diffsim，而是因为它把整条最小链条都放在一屏里，而且源码里已经自带 `check_grad()`。

### 4.2 第二主锚点：`example_diffsim_spring_cage.py`

- 角色：model-parameter branch
- job：纠正“diffsim 只是在改初始状态”的误解

这里最值得教的是：

- 优化对象不一定挂在 `State` 上
- 也可以直接挂在 `Model` 持久参数上
- 同样能走 `forward -> loss -> backward -> update`

### 4.3 Mainline extension：`example_diffsim_soft_body.py`

- 角色：external-parameter-buffer branch
- job：说明参数还可以先放在外部 buffer，再在每次 forward 时灌进 model

这个例子比 `spring_cage` 更接近现实 material optimization，但第一遍不能让它喧宾夺主。

### 4.4 明确 defer 的高级分支

- `example_diffsim_drone.py`
  - 角色：advanced control / MPC branch
  - defer 原因：多 rollout、trajectory optimization、control interpolation 会淹没 first-pass 主线
- `example_diffsim_cloth.py`
  - 角色：advanced scaling branch
  - defer 原因：系统规模更大，但不提供新的 first-pass 闭环结构
- `example_diffsim_bear.py`
  - 角色：advanced learned-controller / actuation branch
  - defer 原因：训练循环、控制权重、长时程复杂度过高

---

## 5. 推荐文件分工

### `README.md`

- 立住本章主问题：`verify before optimize`
- 明确三类参数放置轴：state / model / external buffer
- 给出例子分工和阅读顺序
- 把 `FD validation` 写进完成门槛

### `principle.md`

- 讲清 `requires_grad=True`、`wp.Tape()`、`backward()`、`grad` 这条概念链
- 解释为什么 `FD` 是第一条 hard gate
- 强调 diffsim 是 meta-capability，不是新 solver family

### `source-walkthrough.md`

- 主 walkthrough 只追一条最窄主线：

```text
ball
-> loss on final state
-> backward to initial velocity
-> FD validation
-> update loop
-> spring_cage / soft_body as parameter-placement extensions
```

- 第一遍不做 deep walkthrough

### `examples.md`

- 给每个 upstream diffsim example 分配唯一 teaching job
- 明确 first pass / second pass / advanced 分层

### `pitfalls.md`

- 记录 diffsim 最容易误判的坑：
  - 以为 `backward()` 自动等于梯度可信
  - 忽略 fixed seed / step-size sweep
  - 把 contact branch 的偶然通过当成一般可微信任结论

### `exercises.md`

- 给出最小验证练习，而不是开放大题
- 第一题就应该围绕 `FD validation table`

---

## 6. 推荐源码锚点

### 主锚点

- `newton/examples/diffsim/example_diffsim_ball.py`
- `newton/examples/diffsim/example_diffsim_spring_cage.py`
- `newton/examples/diffsim/example_diffsim_soft_body.py`

### 高级只点名，不进 first pass 细读

- `newton/examples/diffsim/example_diffsim_drone.py`
- `newton/examples/diffsim/example_diffsim_cloth.py`
- `newton/examples/diffsim/example_diffsim_bear.py`

### 仓库内约定锚点

- `conventions/diffsim-validation.md`

---

## 7. 最容易写坏的方式

### 7.1 Demo catalog trap

- 把 `ball / spring_cage / soft_body / drone / cloth / bear` 平铺成 demo 表
- 不给每个例子分配唯一 teaching role

### 7.2 Theory dump trap

- 一上来讲 IFT、adjoint、implicit differentiation
- 结果读者还没看懂 `Tape` 和 `FD`，就先被理论压垮

### 7.3 Autodiff illusion trap

- 把 `tape.backward()` 写成“梯度已经证明正确”
- 不把 `FD validation` 作为 first-pass 必修步骤

### 7.4 Contact overgeneralization trap

- 让读者把 `diffsim_ball` 的简单 contact 结果误读成“所有 contact gradient 都一样稳”

### 7.5 Drone too early trap

- 过早引入 `drone`
- 让读者在 first pass 就被 MPC / rollout batches / control interpolation 淹没

---

## 8. 成功标准

如果 chapter 13 做对了，读者应能稳定带走下面这段话：

```text
DiffSim 不是又一种 solver。
它是在已有 forward simulation 上，
把 loss 通过 `wp.Tape()` 反传回某个 upstream parameter；
但在真正优化之前，必须先用 FD 验证这个梯度的可信区间。
```

而 chapter package 的完成度至少应达到：

1. `README.md` 不再是 skeleton
2. `principle.md` 讲清最小概念链
3. `source-walkthrough.md` 把 `ball -> spring_cage -> soft_body` 串成一条参数放置主线
4. `examples.md` 明确 first-pass / advanced 分工
5. `pitfalls.md` 和 `exercises.md` 至少给出一轮可复习、可验证的最小闭环
