---
chapter: 07
title: 约束与接触数学总论 源码走读
last_updated: 2026-04-19
source_paths:
  - docs/concepts/collisions.rst
  - newton/_src/geometry/contact_data.py
  - newton/_src/sim/collide.py
  - newton/_src/sim/contacts.py
  - newton/_src/solvers/kamino/_src/geometry/contacts.py
  - newton/_src/solvers/kamino/_src/geometry/unified.py
  - newton/_src/solvers/kamino/_src/kinematics/constraints.py
  - newton/_src/solvers/kamino/_src/kinematics/jacobians.py
  - newton/_src/solvers/kamino/_src/core/math.py
  - newton/_src/solvers/kamino/_src/dynamics/delassus.py
  - newton/_src/solvers/kamino/_src/dynamics/dual.py
paper_keys: []
newton_commit: 1a230702
---

# 07 约束与接触数学总论 源码走读

这份 walkthrough 只追一条线:

```text
chapter 06 的 Contacts
-> solver-facing contact
-> 1 contact 对应的 3 条 rows
-> contact Jacobians
-> Delassus
```

第一遍不要把自己困在 solver 算法名里。你要守住的是: chapter 06 交出来的几何，怎样一步一步变成“哪些运动影响接触方向”以及“沿这些方向系统有多难动”。

如果你只想先做一次最小读法，先只看这三站就够了:

1. `newton/_src/sim/contacts.py:L118-L151`，先认清 chapter 06 真正交出来了什么。
2. `newton/_src/solvers/kamino/_src/kinematics/constraints.py:L189-L191` 和 `L284-L299`，先确认为什么 `1 contact -> 3 rows`。
3. `newton/_src/solvers/kamino/_src/dynamics/delassus.py:L105-L193`，先看这些 rows 为什么最后会长成 `D = J M^{-1} J^T`。

## 先带着哪四个问题读

1. chapter 06 写进 `Contacts` 的几何，到了这里到底还剩下哪些最关键的信息？
2. 为什么一个 contact 在 solver 这边会变成 `3` 条 rows，而不是一个标量？
3. Jacobian 在代码里到底是怎样把“接触方向 + 作用点位置”变成对 body 速度的映射？
4. Delassus 在代码里为什么看起来像 `J M^{-1} J^T`，但人话上又可以先理解成“沿这个方向有多难推”？

## Tier 1: 先确认 geometry handoff 里到底有什么

这一层只做一件事: 确认 chapter 06 真正交给后面的几何边界是什么。你先不要急着跳 Kamino。

| 路径 | 角色 | 先读原因 |
|------|------|----------|
| `docs/concepts/collisions.rst` | 接触字段的公开语义 | 先把 `normal / point / distance / margin` 的人话定义读顺。特别是 `L105-L119` 说明法线是 world-frame、接触点是 body-frame，`L1229-L1255` 则把 `Contacts` 明确成 solver 消费的统一容器。 |
| `newton/_src/geometry/contact_data.py` | narrow phase 的统一中间 contact | `ContactData` 在 `L18-L56` 里就把 `contact_point_center`、`contact_normal_a_to_b`、`contact_distance`、`margin_a / margin_b`、`shape_a / shape_b` 摆齐了; `L87-L117` 还能看到 gap check 仍然是沿法线在看距离。 |
| `newton/_src/sim/collide.py` | 把 geometry 真正写进 runtime `Contacts` | `write_contact(...)` 在 `L62-L145` 里把世界接触点换到 body frame，写入 `out_point0 / out_point1`、`out_offset0 / out_offset1`、`out_normal`、`out_margin0 / out_margin1`。这就是 chapter 06 的真正 handoff 边界。 |
| `newton/_src/sim/contacts.py` | runtime `Contacts` 布局 | `L118-L151` 明确了这些数组的含义: `rigid_contact_point0 / point1` 是 body-frame 接触点，`rigid_contact_normal` 是 A-to-B 法线，`offset0 / offset1` 和 `margin0 / margin1` 则是后续摩擦 / 杠杆臂计算要继续用的厚度信息。 |

这一层读完以后，你应该能用一句话复述:

```text
chapter 06 交给 chapter 07 的不是“接触力”，而是一份自包含的接触几何记录。
```

## 中间先补一句重要 caveat

runtime `Contacts` 是 chapter 06 的 handoff object，这句话没有错。

这里也先补一句背景，免得 `Kamino` 这个名字像突然冒出来: 你现在可以把它先当成 Newton 里一条具体的刚体 solver 路线。chapter 07 借它不是为了做 solver 家族比较，而是为了让你看到一条最完整、最显眼的 `Contacts -> solver-facing contact -> rows -> Jacobians -> Delassus` 源码桥。

但如果你继续追这条 Kamino solver 路线，就会发现它并不是直接拿 `rigid_contact_point0 / point1` 这些数组原封不动去建 rows。它会先做一次 bridge，把 Newton 的 runtime `Contacts` 重排成 solver-facing 的 `ContactsKamino`。这是 chapter 07 最容易漏掉、但最值得补上的工程事实。

## Tier 2: 再看 solver-facing contacts、rows 和 Jacobian

这一层才真正进入“为什么一个 contact 会长成三条 row”。推荐按下面顺序读。

| 路径 | 角色 | 先读原因 |
|------|------|----------|
| `newton/_src/solvers/kamino/_src/geometry/contacts.py` | Newton `Contacts` 到 `ContactsKamino` 的桥 | `ContactsKaminoData` 在 `L198-L333` 里直接暴露 solver 更爱吃的字段: `position_A / position_B`、`gapfunc=(normal, signed_distance)`、`frame`、`material`。而 `_convert_newton_contacts_to_kamino(...)` 在 `L783-L876` 又把 Newton 的 body-frame 点先变回 world-space，再重建 `distance` 和 contact frame。 |
| `newton/_src/solvers/kamino/_src/kinematics/constraints.py` | contact rows 的计数规则 | `make_unilateral_constraints_info(...)` 在 `L189-L191` 把每个世界的最大 contact constraints 写成 `3 * max_contacts`，`_update_constraints_info(...)` 在 `L284-L299` 也把 active contact constraints 写成 `ncc = 3 * nc`。这就是“一条 contact 长成三条 row”最直接的代码证据。 |
| `newton/_src/solvers/kamino/_src/kinematics/jacobians.py` | 把 contact frame 和 lever arm 变成 Jacobian blocks | `L785-L810` 很值钱: 先把 contact frame quaternion 变成 `R_k`，再令 `cio_k = 3 * cid_k`，最后用 `JT_c_B_k = W_B_k @ R_k`、`JT_c_A_k = -W_A_k @ R_k` 把接触方向和作用点位置一起写进 Jacobian。 |
| `newton/_src/solvers/kamino/_src/core/math.py` | contact wrench / lever-arm 的最小数学核心 | `contact_wrench_matrix_from_points(...)` 在 `L761-L793` 直接写出 `[[I],[skew(r_k - r_i)]]`。这一小段几乎就是“为什么偏心接触天然会带出角向部分”的源码版答案。 |

### Tier 2 应该重点盯哪三个细节

1. `ContactsKamino` 明确把 contact frame 做成了一级公民。
   `make_contact_frame_znorm(...)` 在 `geometry/contacts.py:L362-L371` 用法线先定 `z` 轴，再自动补两条切向轴。这正是 `1 normal + 2 tangents` 的局部几何来源。

2. `3 * cid_k` 在多处反复出现。
   无论是 `constraints.py` 里的 `ncc = 3 * nc`，还是 `jacobians.py` 里的 `cio_k = 3 * cid_k`，都在提醒你: 对 Kamino 来说，“一个 contact”从来不是“一条约束”，而是一个三维局部 contact block。

3. Jacobian 不是先抽象存在，再去套到 contact 上。
   这里的构造顺序恰好相反: 先有接触点位置 `r_Ac_k / r_Bc_k`，再有 contact frame `R_k`，再有 lever-arm 变换 `W_A_k / W_B_k`，最后才得到 Jacobian block。也就是说，Jacobian 本质上就是“哪些 body motion 会改变这组三个接触方向”。

如果你要把这一层压成一张最小数据流图，可以记成:

| 从哪来 | 在 Kamino 里怎么重排 | 为什么对 rows / Jacobian 有用 |
|--------|----------------------|---------------------------------|
| `rigid_contact_point0 / point1` | 先变回 `position_A / position_B` 的 world-space 作用点 | Jacobian 要知道作用点离质心多远，才能把转动带进来 |
| `rigid_contact_normal` | 进 `gapfunc.xyz`，再长成 contact frame 的法向轴 | 法向 row 和两条切向 row 都要从这里出发 |
| `point0 / point1 + margin / thickness` | 在 bridge 里重建成 `gapfunc.w` | 这决定法向 row 现在是接近、刚接触，还是已经穿入 |
| shape/body/material 信息 | 进 `gid_AB / bid_AB / material` | 这决定 rows 作用在哪两个 body 上，以及摩擦 / 恢复系数怎么读 |

## Tier 3: 最后看 Delassus, 只保住第一层物理意义

当 Jacobian 已经建好以后，再去看 Delassus 才不会迷路。

| 路径 | 角色 | 先读原因 |
|------|------|----------|
| `newton/_src/solvers/kamino/_src/dynamics/delassus.py` | 真正组装 `D` | `_build_delassus_elementwise_dense(...)` 在 `L105-L193` 里非常直白: 对每个约束 row 对 `(i, j)`，遍历 body blocks，累加线性项 `inv_m * dot(Jv_i, Jv_j)` 和角向项 `dot(Jw_i, inv_I @ Jw_j)`。这就是 `J M^{-1} J^T` 的源码展开版。 |
| `newton/_src/solvers/kamino/_src/dynamics/dual.py` | 可选补充: contact-space quantities 后面怎样被 solver 用 | `_build_free_velocity_bias_contacts(...)` 在 `L600-L685` 里继续沿用 `3 * cid_k`，并从 `gapfunc.w` 读 penetration / gap，从 `material.x` 读摩擦系数。它能帮助你确认 Delassus 后面确实是在 contact-space 里继续工作。 |
| `newton/_src/solvers/kamino/_src/geometry/unified.py` | 可选补充: 另一条更直接的 bridge | 如果 collision pipeline 一开始就想写 Kamino 格式，可以看 `write_contact_unified_kamino(...)` 的 `L130-L247`: 它直接把 `ContactData` 写成 `gapfunc + frame + material`。这能帮你确认 solver-facing contact 需要的最小字段到底是什么。 |

这里第一遍最值得看的不是大公式，而是 `delassus.py` 里那两项拆分:

- 线性项: 说明 row 对 body 平移惯性的敏感度。
- 角向项: 说明 row 对 body 转动惯性的敏感度。

所以当你脑中放的是 `sphere-ground` 时，Delassus 更像“沿法向推这个球，球有多难动”；而当你脑中换成 `box-ground` 的多接触和偏心接触时，Delassus 又更像“这些 rows 彼此会不会通过同一个箱子的平移 / 转动耦合在一起”。

## 推荐的一遍读法

如果你只想花一次完整 first pass，把最值钱的桥读出来，推荐顺序是:

1. `docs/concepts/collisions.rst:L105-L119` 和 `L1229-L1255`
2. `newton/_src/geometry/contact_data.py:L18-L56`
3. `newton/_src/sim/collide.py:L62-L145`
4. `newton/_src/sim/contacts.py:L118-L151`
5. `newton/_src/solvers/kamino/_src/geometry/contacts.py:L198-L333`
6. `newton/_src/solvers/kamino/_src/geometry/contacts.py:L783-L876`
7. `newton/_src/solvers/kamino/_src/kinematics/constraints.py:L189-L191` 和 `L284-L299`
8. `newton/_src/solvers/kamino/_src/kinematics/jacobians.py:L785-L810`
9. `newton/_src/solvers/kamino/_src/core/math.py:L761-L793`
10. `newton/_src/solvers/kamino/_src/dynamics/delassus.py:L105-L193`

读完这十个锚点，你就已经能把 chapter 07 的主线讲顺了:

```text
Contacts 先保住接触几何
Kamino 再把它桥接成 solver-facing contact
每个 contact 对应 3 条局部方向 rows
Jacobians 负责写出“哪些速度会影响这些方向”
Delassus 负责写出“沿这些方向系统有多难被推着动”
```

这页到这里就停。再往后如果你继续问“这些 rows 最后怎么解”，那就已经自然进入 `08_rigid_solvers` 了。
