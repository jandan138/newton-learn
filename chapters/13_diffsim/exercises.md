---
chapter: 13
title: 13 可微分仿真 Exercises
last_updated: 2026-04-22
source_paths:
  - newton/examples/diffsim/example_diffsim_ball.py
  - newton/examples/diffsim/example_diffsim_spring_cage.py
  - newton/examples/diffsim/example_diffsim_soft_body.py
  - newton/examples/diffsim/example_diffsim_drone.py
  - conventions/diffsim-validation.md
paper_keys: []
newton_commit: 1a230702
---

# 13 可微分仿真 Exercises

这些练习都故意做小。目标不是写长篇理论答案，而是留下 **你已经会验证 chapter 13 主线** 的最小证据。

## Exercise 1: FD Error Table

**题目**

围绕一个 diffsim 例子写最小 FD 记录表。第一题优先选 `ball`；如果你已经稳住它，也可以选 `spring_cage` 或 `soft_body`。

| eps | analytic grad | numeric grad | relative error | verdict |
|-----|---------------|--------------|----------------|---------|
| 1e-3 |  |  |  |  |
| 1e-4 |  |  |  |  |
| 1e-5 |  |  |  |  |
| 1e-6 |  |  |  |  |
| 1e-7 |  |  |  |  |

**交付物**

- 一张完整表格。
- 表格上方再补三项 metadata: `seed`、`window_length`、`metric`。

**通过标准**

- 不能只填一个 `eps`。
- 必须给出至少一个区间判断，而不是只说“看起来差不多”。
- `verdict` 必须使用 `pass`、`suspicious` 或 `fail`。

## Exercise 2: 参数放置三分法

**题目**

只用 `ball`、`spring_cage`、`soft_body`，填完下面这张表。

| 例子 | 你要优化的参数 | 它放在哪一层对象上 | 梯度最后落在哪里 |
|------|----------------|--------------------|------------------|
| `ball` |  |  |  |
| `spring_cage` |  |  |  |
| `soft_body` |  |  |  |

**交付物**

- 一张填完的三行表。

**通过标准**

- 三行都能明确写成 `state parameter`、`model parameter`、`external parameter buffer` 之一。
- 不能出现“都差不多”“都是上游参数”这类模糊答案。

## Exercise 3: `ball` 最小闭环复述

**题目**

把 `example_diffsim_ball.py` 的主线压成 6 个箭头，不多不少:

```text
? -> ? -> ? -> ? -> ? -> ?
```

要求必须包含这六类信息:

- forward rollout
- scalar loss
- `wp.Tape`
- gradient lands on initial velocity
- FD validation
- update step

**交付物**

- 一行 6 箭头答案。

**通过标准**

- 不能把 `FD validation` 漏掉。
- 不能把 `optimizer` 写在 `FD` 前面。

## Exercise 4: 抓出伪验证

**题目**

下面四种说法里，哪些不算 chapter 13 的有效验证？每条只回答 `valid` 或 `invalid`，再补一句原因。

1. “`tape.backward(loss)` 没报错，所以梯度可以直接拿来调学习率。”
2. “我固定了 seed，并扫了 5 个 `eps`，误差最小区间在 `1e-4 ~ 1e-5`。”
3. “我只测了一个 `eps=dt`，数值梯度接近解析梯度，所以验证完成。”
4. “我已经知道梯度落在 external buffer 上，所以可以不做 FD。”

**交付物**

- 四行 `valid/invalid + 一句话理由`。

**通过标准**

- 1、3、4 必须判成 `invalid`。
- 你的理由里必须出现 `trust`、`FD sweep`、`parameter placement` 这三类词里至少两类。

## Exercise 5: 为什么 `drone` 不进 first pass

**题目**

用不超过 3 句话解释，为什么 `example_diffsim_drone.py` 被放到 advanced，而不是 first pass。

**交付物**

- 最多 3 句话。

**通过标准**

- 必须明确提到 control / trajectory complexity。
- 必须明确说出“它增加的不是 first-pass 必需的新信任结构”。

## Exercise 6: Contact 结论收缩

**题目**

把下面这句过度结论，改写成 chapter 13 允许的版本:

```text
`diffsim_ball` 通过了，所以 contact gradients 在 Newton 里一般都可信。
```

**交付物**

- 一句改写后的结论。

**通过标准**

- 必须保留“固定 seed / 固定窗口 / 当前例子”这类局部限定词中的至少两个。
- 不能写成 contact-wide generalization。

## 自检

- 你现在能不能不提大段理论，只靠一张参数放置表和一张 FD error table，就证明自己读懂了 chapter 13 的主线？
- 你现在能不能解释为什么 chapter 13 的关键不是“又会一个 demo”，而是“先证明梯度可信，再决定要不要优化”？
