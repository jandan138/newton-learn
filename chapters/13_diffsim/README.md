---
chapter: 13
title: 13 可微分仿真：verify before optimize
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

# 13 可微分仿真：verify before optimize

`12_sensors_ik` 刚把一个重要问题讲顺:

```text
状态既然已经存在，
我怎样读它？又怎样朝目标去改它？
```

chapter 13 的自然下一问不是“再学一种 solver”，而是更窄也更关键的一句:

```text
一次 rollout 没打中目标时，
我到底该改哪个上游参数，梯度会出现在哪里，
又怎么知道这个梯度可信？
```

这就是本章的统一标题:

- DiffSim 不是又一种 solver family。
- DiffSim 是加在已有 forward simulation 上的一层 backward capability。
- 但这层 capability 只有先通过验证，后面的优化才值得相信。

第一遍只守住这条 trust-first spine:

```text
forward sim
-> scalar loss
-> wp.Tape() capture
-> backward()
-> gradient lands on an upstream parameter
-> FD validation
-> update loop
```

这里最好把 chapter 13 读成一句更短的话:

```text
verify before optimize
```

## 文件分工

- `README.md`: 立住本章主问题、比较轴、例子分工、完成门槛。
- `principle.md`: 讲清 `requires_grad=True -> wp.Tape() -> backward() -> grad` 这条最小概念链，并把 `FD` 提升为 hard gate。
- `source-walkthrough.md`: beginner-safe main walkthrough。第一遍先沿 `ball -> trust gate -> spring_cage / soft_body` 走完最小闭环。
- `examples.md`: 给所有 upstream diffsim examples 分配唯一 teaching job，并明确 first-pass / advanced 分层。
- `pitfalls.md`: 记录 chapter 13 最容易出现的验证误判。
- `exercises.md`: 留下最小验证练习，而不是开放式理论题。

这一整套文件都应该继续服务同一条主线，而不是把 chapter 13 写成 diffsim demo catalog。

## 例子分工

| 例子 | 这章给它的唯一 job | 角色 | 何时使用 |
|------|---------------------|------|----------|
| `newton/examples/diffsim/example_diffsim_ball.py` | first-pass main anchor。把 `forward -> loss -> backward -> FD -> update` 这条最小闭环放在一屏里 | 第一主例子 | 第一遍必看 |
| `newton/examples/diffsim/example_diffsim_spring_cage.py` | model-parameter branch。纠正“diffsim 只是在改初始状态”的误解 | 必要扩展 | `ball` 稳后立刻看 |
| `newton/examples/diffsim/example_diffsim_soft_body.py` | external-parameter-buffer branch。说明参数也可以先放在外部 buffer，再映射进 model | 第二遍扩展 | `spring_cage` 稳后再看 |
| `newton/examples/diffsim/example_diffsim_drone.py` | control / MPC branch。说明 rollout optimization 会把主线放宽很多 | 高级分支 | 最后再看 |
| `newton/examples/diffsim/example_diffsim_cloth.py` | scaling branch。说明系统更大不代表 first-pass 更适合它 | 高级分支 | 最后再看 |
| `newton/examples/diffsim/example_diffsim_bear.py` | learned-controller / actuation branch。说明训练循环会进一步抬高复杂度 | 高级分支 | 最后再看 |

chapter 13 的比较轴也不要按 solver family 记，而要按“参数到底放在哪里”来记:

```text
state parameter
-> model parameter
-> external parameter buffer
```

也就是:

- `ball`: gradient 落在上游 state-side parameter。
- `spring_cage`: gradient 落在 model-side persistent parameter。
- `soft_body`: gradient 先落在外部参数 buffer，再由 forward 映射进 model。

## 本章目标

- 把 chapter 13 的主问题立成“rollout miss 以后，梯度回到哪里，以及为什么要先验证它”。
- 让你记住 DiffSim 是 backward capability，不是新 solver family。
- 把第一遍真正要守住的闭环缩到 `forward sim -> loss -> backward -> FD validation -> update loop`。
- 建立 chapter 13 的第一比较轴: `state parameter / model parameter / external parameter buffer`。
- 明确 `drone / cloth / bear` 都不属于 first-pass trust-establishment mainline。

## 本章范围

- 主教学锚点只守住三个 examples: `example_diffsim_ball.py`、`example_diffsim_spring_cage.py`、`example_diffsim_soft_body.py`。
- 本章必须把 `conventions/diffsim-validation.md` 视为主线规则，而不是附录。
- 第一遍只覆盖最小可微信任闭环、参数放置轴、以及高级分支为什么要后置。

## 本章明确不做什么

- 不把 `ball / spring_cage / soft_body / drone / cloth / bear` 平铺成 demo 表。
- 不把本章写成 IFT、adjoint、implicit differentiation 的理论堆栈。
- 不把 `tape.backward()` 误写成“梯度已经自动证明正确”。
- 不把 `drone / cloth / bear` 提前塞进 first pass。

## 前置依赖

- 建议先读完 `12_sensors_ik`。chapter 12 解决的是“状态怎样被读和被写回”；chapter 13 继续追问“loss 怎样回传到上游参数，并在优化前先被验证”。
- 默认你已经接受 chapter 08/09 的最小 contract: 系统会先做 forward rollout，再从某个观测或终态定义 scalar loss。
- `conventions/diffsim-validation.md` 不是可选补充材料，而是本章第一遍就要带着读的验证规程。

## 完成门槛

```text
[ ] 我能用一句话解释为什么 DiffSim 不是又一种 solver family
[ ] 我能顺着说出本章主 spine: forward sim -> loss -> wp.Tape() -> backward() -> upstream grad -> FD -> update loop
[ ] 我能解释为什么 `tape.backward()` 本身只给出 gradient candidate，而不是可信证明
[ ] 我能解释 `conventions/diffsim-validation.md` 里的 FD step-size sweep 和 trust window 为什么是 hard gate
[ ] 我能指出至少一个“先验证梯度、再进入优化”的具体例子
[ ] 我能说出 `ball / spring_cage / soft_body` 分别对应 state、model、external buffer 三种参数放置
[ ] 我知道为什么 `drone / cloth / bear` 被明确放在 first pass 之外
```

## 阅读顺序

1. 先读本文件，把 chapter 13 的标题改写成 `verify before optimize`。
2. 再读 `principle.md`，把 `requires_grad=True -> wp.Tape() -> backward() -> FD` 这条最小链讲顺。
3. 再读 `source-walkthrough.md`，沿 `ball -> trust gate -> spring_cage / soft_body` 真正把源码主线串一遍。
4. 第一遍看例子时，只按 `ball -> spring_cage -> soft_body` 的参数放置顺序推进，不要一开始就把所有 diffsim demo 混着看。
5. 只有前四步稳定后，再把 `drone / cloth / bear` 当成 advanced branches 回看。

## 预期产出

- `principle.md`: 讲清这章为什么是 trust-first chapter，以及 gradient 会落到哪一层对象上。
- `source-walkthrough.md`: 把 `ball` 的最小可微信任闭环讲顺，再用 `spring_cage / soft_body` 解释参数落点。
- `examples.md`: 给 `ball / spring_cage / soft_body / drone / cloth / bear` 六个例子分配唯一 teaching role。
- `pitfalls.md` 和 `exercises.md`: 把 “先验证、再优化” 这条规矩落成可复习、可自检的最小闭环。

读完 chapter 13 后，你最该带走的不是一串 API 名字，而是这句更短的话:

```text
DiffSim 不是又一种 solver。
它是在已有 forward simulation 上，
把 loss 通过 `wp.Tape()` 反传回某个 upstream parameter；
但在真正优化之前，必须先用 FD 验证这个梯度的可信区间。
```
