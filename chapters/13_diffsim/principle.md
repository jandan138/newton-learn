---
chapter: 13
title: 13 可微分仿真原理主线：先验证，再优化
last_updated: 2026-04-22
source_paths:
  - newton/examples/diffsim/example_diffsim_ball.py
  - newton/examples/diffsim/example_diffsim_spring_cage.py
  - newton/examples/diffsim/example_diffsim_soft_body.py
  - newton/examples/diffsim/example_diffsim_drone.py
  - newton/examples/diffsim/example_diffsim_cloth.py
  - newton/examples/diffsim/example_diffsim_bear.py
paper_keys:
  - difftaichi-paper
  - brax-paper
  - warp-adjoint-paper
  - ift-survey
  - contact-randomized-smoothing
  - shac-paper
newton_commit: 1a230702
---

# 13 可微分仿真原理主线：先验证，再优化

## 0. 先把 chapter 13 的标题改对

chapter 13 最容易被写坏成两种样子:

```text
又一种 solver family
+
autodiff 理论大全
```

这两种开场都不稳，因为它们会把真正的 first-pass 问题盖掉。

更稳的入口只有一句:

```text
一次 rollout 没打中目标时，
我到底该改哪个上游参数，梯度会出现在哪里，
又怎么知道这个梯度可信？
```

所以 DiffSim 在 Newton 里第一遍不要读成“新的 forward solver”，而更该读成:

```text
在已有 forward simulation 上，
增加一条把 loss 回传到 upstream parameter 的 backward capability。
```

但 chapter 13 真正要守住的还不只是 `backward capability`，而是这条更完整的主线:

```text
verify before optimize
```

也就是说:

- `backward()` 给出的是 gradient candidate。
- `FD` 决定这个 gradient 能不能被当成可信优化信号。
- 只有先过验证门槛，update loop 才有意义。

## 1. `requires_grad=True` 先决定谁会进入梯度账本

一个普通 forward sim 变成 differentiable，最先变化的不是 loss，也不是 optimizer，而是:

```text
哪些对象会被梯度追踪。
```

在 Warp / Newton 这一层，`requires_grad=True` 的 job 可以先读得很短:

```text
把某个 model / state / buffer 标成“这是上游参数候选”。
```

它真正改变的是 ownership:

- 你没有标记成 `requires_grad=True` 的东西，默认就不该指望它在 backward 后冒出你想要的梯度。
- 你标记了的对象，才有机会在 tape 反传后接到 adjoint。
- chapter 13 的重点不是“所有东西都开梯度”，而是先找准你真正想改的那个 upstream parameter。

所以第一遍看 diffsim，不要先问“optimizer 用什么”，而要先问:

```text
我要优化的参数到底放在哪里？
```

这也是本章后面那条参数放置轴的起点。

## 2. `wp.Tape()` 记录的是 forward kernel 账本，不是正确性证明

当上游参数被标进梯度账本后，下一步才轮到 `wp.Tape()`。

first pass 可以把它读成:

```text
把这次 forward rollout 里发生过的 kernel 执行记下来，
以便后面 backward 时知道梯度该沿哪条路往回传。
```

这句话里最重要的是“记下来”，不是“证明正确”。

- `wp.Tape()` 解决的是记录问题。
- 它让 backward 知道前向做过哪些可微操作。
- 它本身不替你判断这些梯度值是不是可信。

所以 chapter 13 最危险的误读之一就是:

```text
Tape 开了
-> backward 跑了
-> 梯度已经没问题
```

这条推理是不成立的。更稳的说法是:

```text
Tape 让梯度有路可回。
至于这条路回来的数值值不值得信，后面还要过 FD 这道门。
```

## 3. `backward()` 的 job 是把 loss 送回上游参数

有了 tracked parameter，也有了 tape 记录，chapter 13 的第三步才是:

```text
scalar loss
-> backward()
-> gradient lands on an upstream parameter
```

这里第一遍最该盯住两件事。

### 3.1 loss 必须先被写成标量目标

DiffSim 不是“只要跑了 forward 就自然有梯度”。更准确的顺序是:

```text
rollout 先产生状态
-> 你从某个终态或观测里定义 scalar loss
-> backward() 才知道要把什么往回传
```

也就是说，loss 才是 backward 的起点。没有这个起点，梯度就没有“往哪一边回”的方向。

### 3.2 梯度会落在某个上游参数上，不是自动落在所有对象上

chapter 13 的真正问题不是“有没有 gradient”，而是:

```text
gradient 最后落在哪一层对象上？
```

这也是为什么这章最稳的比较轴不是 solver family，而是 parameter placement:

- 有的 gradient 落在 state-side parameter。
- 有的 gradient 落在 model-side persistent parameter。
- 有的 gradient 落在 external parameter buffer。

如果你没先问清这件事，就很容易把一堆 diffsim example 看成“都在做差不多的优化”。实际上，它们最关键的区别恰恰是 upstream parameter 的落点不同。

## 4. chapter 13 的第一比较轴：参数到底放在哪里

本章最实用的一张表，不是 solver 对比表，而是 parameter placement 表。

| 参数放置 | first-pass 怎么读 | 例子角色 | gradient 最后落在哪里 |
|----------|-------------------|----------|------------------------|
| `state parameter` | 你要改的是某个初始状态、初始速度、或某一拍 state-side 量 | `example_diffsim_ball.py` | 某个 state-side upstream parameter |
| `model parameter` | 你要改的是挂在 model 上、跨 rollout 持续存在的配置或物理参数 | `example_diffsim_spring_cage.py` | model-side persistent parameter |
| `external parameter buffer` | 你要改的是先存在外部 buffer、再在 forward 时映射进 model 的参数 | `example_diffsim_soft_body.py` | 外部参数 buffer |

这张表最重要的价值不是记例子名，而是帮你稳定一句判断:

```text
DiffSim 的区别，常常不在于“是不是 optimizer”，
而在于 loss 最后回到了哪一层上游对象。
```

### 4.1 `ball`：先用 state-side parameter 建立最小闭环

`example_diffsim_ball.py` 适合作为第一主例子，不是因为它代表所有 diffsim，而是因为它最适合把下面这条链完整摆出来:

```text
initial state-side parameter
-> rollout
-> final-state loss
-> tape.backward(loss)
-> gradient on that upstream state-side parameter
-> FD check
-> gradient descent step
```

也就是说，`ball` 的教学价值首先是 trust-establishment，而不是 contact 的一般性结论。

### 4.2 `spring_cage`：纠正“diffsim 只改初始状态”的误会

当 `ball` 这条最小闭环稳定后，第二步最该补的是:

```text
gradient 也可以直接落在 model parameter 上。
```

这正是 `example_diffsim_spring_cage.py` 的 teaching job。它提醒你:

- upstream parameter 不一定挂在 `State` 上。
- 有些你真正想调的量，本来就应该挂在 `Model` 上长期存在。
- 但它仍然走同一条主线: `forward -> loss -> backward -> update`。

### 4.3 `soft_body`：参数还可以先放在 model 外面

再往前一步，`example_diffsim_soft_body.py` 又补上一种更接近现实 material optimization 的模式:

```text
我要调的参数先放在外部 buffer，
forward 时再把它映射进 model。
```

这一层的关键词不是“更高级的 optimizer”，而是:

```text
gradient 先落在 external parameter buffer，
再由你决定怎样把它写回下一次 forward。
```

这也是 chapter 13 第一遍为什么要按参数放置讲，而不是按 demo 列表讲。

## 5. `FD` 不是调试附录，而是 hard trust gate

如果说前面三步讲的是“梯度怎么回来”，那 chapter 13 最关键的一刀在这里:

```text
回来，不代表可信。
```

这就是为什么 `conventions/diffsim-validation.md` 不该被放到附录区，而应该被读成 chapter 13 的主规则。

更短的说法是:

```text
backward() gives a gradient candidate.
FD decides whether that gradient is trustworthy enough to optimize with.
```

这条规则至少包含三层意思。

### 5.1 单次 `backward()` 不能替代数值验证

`backward()` 说明的是“解析梯度路径存在，并且系统给了你一个数”。

它没有自动回答:

- 这个数和有限差分是否一致。
- 一致性只在某个偶然步长下成立，还是在一段 trust window 内稳定成立。
- 这条 gradient 在当前时间窗口、当前随机种子、当前 loss 定义下是否值得信。

所以 first pass 必须把下面这句话讲清:

```text
能反传，不等于能相信。
```

### 5.2 FD 要看 sweep 和 trust window，不是只看一个 epsilon

`conventions/diffsim-validation.md` 已经把 chapter 13 最小规程写得很明确:

- 步长扫描: `eps in {1e-3, 1e-4, 1e-5, 1e-6, 1e-7}`。
- 误差门槛: 相对误差 `< 1e-3` 视为通过，`1e-3 ~ 1e-2` 视为可疑，`> 1e-2` 视为失败。
- 随机种子、时间窗口、loss 定义都要固定并记录。

这套规程的含义不是“多做一点表格更严谨”，而是:

```text
gradient 的可信性必须落在一个步长区间判断上，
而不是靠单个 epsilon 的偶然对齐来过关。
```

### 5.3 update loop 只能写在验证之后

chapter 13 的章节标题是 `verify before optimize`，不是 `differentiate and hope`。

顺序必须是:

```text
forward
-> loss
-> backward
-> FD validation
-> verdict pass / suspicious / fail
-> only then update
```

这条顺序一旦倒过来，后面的优化轨迹再好看，第一遍也不该轻易当真。

## 6. 为什么 `drone / cloth / bear` 必须明确放在 first pass 之外

chapter 13 第一遍不是不能提 `drone / cloth / bear`，而是不能让它们抢走主线。

原因都很直接:

- `drone` 更像 control / MPC branch，会带来多 rollout、轨迹优化、控制插值等额外层次。
- `cloth` 更像 scaling branch，系统更大，但没有比最小闭环多出更适合 first-pass 的结构。
- `bear` 会把 learned controller、actuation、训练循环一起带进来，复杂度过高。

所以更稳的教学顺序只能是:

```text
ball
-> spring_cage
-> soft_body
-> only then advanced branches
```

这不是在贬低高级例子，而是在保护 first-pass 的信任建立过程。

## 7. 第一遍只带走一句话也够

```text
DiffSim 不是又一种 solver。
它是在已有 forward simulation 上，
把 scalar loss 通过 `wp.Tape()` 和 `backward()` 送回某个 upstream parameter；
但在真正优化之前，必须先用 FD 验证这个梯度的可信区间。
```

如果这句话已经稳了，chapter 13 的主骨架就已经搭起来了。
