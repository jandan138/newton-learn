---
chapter: 13
title: 13 可微分仿真实例观察单
last_updated: 2026-04-22
source_paths:
  - newton/examples/diffsim/example_diffsim_ball.py
  - newton/examples/diffsim/example_diffsim_spring_cage.py
  - newton/examples/diffsim/example_diffsim_soft_body.py
  - newton/examples/diffsim/example_diffsim_drone.py
  - newton/examples/diffsim/example_diffsim_cloth.py
  - newton/examples/diffsim/example_diffsim_bear.py
  - conventions/diffsim-validation.md
paper_keys: []
newton_commit: 1a230702
---

# 13 可微分仿真实例观察单

这页不是 `diffsim demos catalog`。它只做一件事: **给 chapter 13 的六个 upstream diffsim examples 各分配一个唯一 teaching job，并把阅读顺序压成 `verify before optimize`。**

所以第一遍不要把这些例子混着跑。先用 `ball` 把最小可信闭环立住，再用 `spring_cage` 和 `soft_body` 区分参数到底放在哪里，最后才碰 `drone / cloth / bear`。

## 总表

| 例子 | 这页给它的唯一 job | 分层 | 参数放置主轴 | 为什么保留 |
|------|---------------------|------|--------------|------------|
| `newton/examples/diffsim/example_diffsim_ball.py` | 建立最小 `forward -> loss -> backward -> FD -> update` 信任闭环 | first pass | `state parameter` | 最小链条最完整，而且自带 `check_grad()` |
| `newton/examples/diffsim/example_diffsim_spring_cage.py` | 纠正“DiffSim 只能改初始状态”的误会 | second pass | `model parameter` | 让你看到梯度也可以直接落在持久模型参数上 |
| `newton/examples/diffsim/example_diffsim_soft_body.py` | 说明参数还可以先放在外部 buffer，再映射进模型 | second pass | `external parameter buffer` | 更接近材料辨识或 material optimization 的真实写法 |
| `newton/examples/diffsim/example_diffsim_drone.py` | 把同一闭环扩展到 control / trajectory optimization | advanced | `defer: beyond first-pass placement axis` | 说明 DiffSim 还能服务 MPC，但第一遍太容易被 rollout complexity 淹没 |
| `newton/examples/diffsim/example_diffsim_cloth.py` | 说明同一闭环也会遇到更大系统规模与更重状态空间 | advanced | `defer: beyond first-pass placement axis` | 它增加的是规模压力，不是 first-pass 必需的新结构 |
| `newton/examples/diffsim/example_diffsim_bear.py` | 说明梯度还能继续喂给 learned controller / actuation loop | advanced | `defer: beyond first-pass placement axis` | 让 chapter 13 接到训练循环，但复杂度太高，不适合拿来建立信任 |

前 3 行才是 chapter 13 第一遍真正比较的参数放置轴；后 3 行只是 advanced branches，故意不并入同一 first-pass taxonomy。

## First Pass Main Anchor: `example_diffsim_ball.py`

**唯一 job**

先把 chapter 13 最窄、但最重要的一条线立住:

```text
initial velocity
-> rollout
-> final-state loss
-> wp.Tape backward
-> gradient on initial velocity
-> FD check
-> gradient descent step
```

**先盯哪几处**

- `requires_grad=True` 的 tracked object 是怎样在 model / state finalize 后被打开的。
- `wp.Tape()` 怎样包住 forward rollout。
- loss 怎样落在 final state，而不是中途任意打印的量上。
- `tape.backward(loss)` 之后，梯度具体落在哪个上游参数上。
- `check_grad()` 怎样拿解析梯度和数值梯度对比。
- update step 怎样真的回写到初始速度这类 upstream state parameter。

**这次只回答五个问题**

- 优化参数是什么: 初始速度这类 `state parameter`。
- loss 在哪里: rollout 结束后的标量目标。
- 梯度落在哪里: 同一个初始速度参数槽位上。
- update 怎样应用: 用梯度更新这个初始参数，再重跑 rollout。
- 真正信它之前要验证什么: `check_grad()` 只能算 example-local sanity check，还要回到 `conventions/diffsim-validation.md` 的 fixed-seed + step-size sweep 规程里确认 trust window。

**不要拿它做什么**

- 不要把它读成“只要 `backward()` 跑通，梯度就一定可信”。
- 不要把它里头一次 contact 成功，读成“所有 contact gradient 都一样稳”。
- 不要第一遍就跳去调 optimizer 超参；这例子的第一任务是先过 FD 门槛。

## Second Pass Branch 1: `example_diffsim_spring_cage.py`

**唯一 job**

证明 chapter 13 不是“只会改初始条件”。同一条闭环也能改 **挂在 `Model` 上的持久参数**。

**这次要观察什么**

- 优化参数是什么: `model.spring_rest_length` 这类 `model parameter`。
- loss 在哪里: 仍然是 rollout 之后可写成标量的目标，不因为参数换到 `Model` 就改变章法。
- 梯度落在哪里: 直接落在 `model.spring_rest_length` 上，而不是落回 `State`。
- update 怎样应用: 更新 rest length，再用新模型参数重跑 forward。
- 真正信它之前要验证什么: 还是要做 FD；只是这次扰动的是 model parameter，而不是初始速度。

**它教会你的不是更多 feature，而是更稳的分类法**

```text
DiffSim 的关键问题不是“这是哪个 solver family”，
而是“我想改的参数到底放在哪一层对象上”。
```

**不要拿它做什么**

- 不要把它误读成“所有 model field 都适合直接做梯度优化”。
- 不要因为 `Model` 看起来更静态，就跳过 FD；静态存储不等于梯度天然可信。

## Second Pass Branch 2: `example_diffsim_soft_body.py`

**唯一 job**

说明还有第三种常见写法: **先把可优化参数放在外部 buffer 里，再在每次 forward 时映射进模型字段。**

**这次要观察什么**

- 优化参数是什么: `material_params` 这类 external parameter buffer。
- loss 在哪里: 仍然是一个 rollout 后的标量目标。
- 梯度落在哪里: 先落在 `material_params` 上，而不是直接落在最终被消费的 `model.tet_materials` 上。
- update 怎样应用: 先更新 buffer，再把它重新映射进 `model.tet_materials` 后继续 forward。
- 真正信它之前要验证什么: FD sweep 也要对准 buffer-level parameter，而不是只盯模型里最后那份已映射数组。

**它真正增加的认识**

- 你不一定要把可优化量永久存进 `State` 或 `Model` 才能做 DiffSim。
- external buffer 写法更接近材料辨识、系统辨识和 batched optimization 的真实工程形态。
- 但章法没变，还是 `forward -> loss -> backward -> FD -> update`。

**不要拿它做什么**

- 不要把“外部 buffer -> model remap”误读成新的 solver family。
- 不要因为 remap 看起来像普通数据搬运，就忽略梯度到底穿过了哪一层对象。

## Advanced Branch: `example_diffsim_drone.py`

**唯一 job**

告诉你 `ball` 那条最小闭环还能扩成 control / trajectory optimization，甚至走到 MPC 风格的多步控制问题。

**为什么 defer**

- 它一下子把单次 rollout 扩成更长时域的问题。
- 你要同时跟踪 control interpolation、trajectory parameterization、可能的多 rollout 比较。
- 如果 first pass 还没稳住，这些额外结构会让你分不清到底是梯度不可信，还是控制问题本身更难。

**只带走一句话**

```text
`drone` 增加的是 control complexity，
不是 chapter 13 第一遍必须新增的信任结构。
```

## Advanced Branch: `example_diffsim_cloth.py`

**唯一 job**

说明同一套可微闭环在更大的 deformable system 里也能成立，但系统规模、状态维度和接触复杂度都会变得更重。

**为什么 defer**

- 它增加的是 system scale，不是更适合 first-pass 的教学透明度。
- 更大的 state / contact 结构更容易把“梯度来源”和“梯度是否可信”搅在一起。
- 如果读者还没在 `ball` 里建立 trust loop，这里很容易退化成“看一个更大的 demo”。

## Advanced Branch: `example_diffsim_bear.py`

**唯一 job**

说明 DiffSim 不只会把梯度送回显式物理参数，也能继续送进 learned controller 或 actuation weights，变成训练循环的一部分。

**为什么 defer**

- 这条线会把 controller weights、actuation、长时程 rollout 和训练 loop 一次性拉进来。
- 你一旦没先区分“physics gradient 是否可信”和“learning loop 是否稳定”，就会很难定位问题。
- chapter 13 第一遍要先学会验证梯度，再去学会消费梯度。

## 推荐顺序

1. 先看 `example_diffsim_ball.py`，把最小可信闭环立住。
2. 再看 `example_diffsim_spring_cage.py`，确认参数也可以放在 `Model`。
3. 然后看 `example_diffsim_soft_body.py`，确认参数还可以先放在外部 buffer。
4. 最后才把 `drone / cloth / bear` 当 advanced branches，看同一闭环怎样扩到 control、规模和 learned-controller 问题。

## 自检

- 现在只看 `ball`，你能不能讲清为什么 `FD` 是进入 update loop 之前的硬门槛？
- 现在只看 `spring_cage`，你能不能一句话说明为什么它不是“又一个改初始状态的例子”？
- 现在只看 `soft_body`，你能不能解释为什么 external buffer 写法最容易暴露“梯度到底落在哪层对象上”这个问题？
- 现在只看 `drone / cloth / bear`，你能不能说清它们为什么都属于 advanced，而不是拿来建立 first-pass trust 的主锚点？
