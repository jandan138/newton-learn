---
chapter: 15
title: 15 多物理耦合与端到端流水线练习
last_updated: 2026-04-26
source_paths:
  - newton/examples/multiphysics/example_softbody_dropping_to_cloth.py
  - newton/examples/mpm/example_mpm_twoway_coupling.py
  - newton/examples/cloth/example_cloth_franka.py
paper_keys: []
newton_commit: 0f583176
---

# 15 多物理耦合与端到端流水线练习

这些练习只服务一个目标：你能不能标出 coupling 发生在哪个边界。

![Chapter 15 exercises coupling boundary quiz](assets/15_exercises_coupling_boundary_quiz.png)

## 练习 1：判断 coupling category

把下面例子标成 `single-model solver coupling`、`two-model bridge` 或 `external-solver bridge`：

| 例子 | 分类 |
|------|------|
| `softbody_dropping_to_cloth` |  |
| `softbody_gift` |  |
| `mpm_twoway_coupling` |  |
| `cloth_franka` |  |

参考答案：

- `softbody_dropping_to_cloth`: single-model solver coupling。
- `softbody_gift`: single-model solver coupling。
- `mpm_twoway_coupling`: two-model bridge。
- `cloth_franka`: external-solver bridge。

## 练习 2：找 source-of-truth buffers

读下面变量名，判断它属于 rigid side、sand side、contacts side 还是 viewer side：

```python
self.state_0
self.sand_state_0
self.contacts
self.collider_impulses
self.body_sand_forces
self.viewer.log_points(...)
```

参考答案：

- `self.state_0`: rigid side state。
- `self.sand_state_0`: sand / MPM side state。
- `self.contacts`: rigid collision contacts。
- `self.collider_impulses`: MPM-to-rigid bridge buffer。
- `self.body_sand_forces`: bridge 后写给 rigid bodies 的 forces。
- `viewer.log_points(...)`: viewer side output。

## 练习 3：解释 builder 与 solver 的边界

用两句话回答：

```text
为什么 add_soft_grid() / add_cloth_grid() 不是 runtime coupling 本身？
```

参考答案：

```text
它们在 builder 阶段创建拓扑、材料和初始粒子/元素，把对象写进 Model。
runtime coupling 发生在 collide() 更新 Contacts、solver.step() 消费 Contacts 或 bridge buffer 推进 State 的时候。
```

## 练习 4：给 MPM two-way bridge 排序

给这些步骤排一个合理顺序：

```text
collect collider impulses
compute body forces from previous impulses
rigid solver step
MPM solver step
render rigid state and sand points
```

参考答案：

```text
compute body forces from previous impulses
-> rigid solver step
-> MPM solver step
-> collect collider impulses
-> render rigid state and sand points
```

## 练习 5：判断哪些说法能写入 Chapter 15 first pass

| 说法 | 是否可写 | 原因 |
|------|----------|------|
| 当前 `examples/multiphysics/` 有 soft body + cloth VBD 示例。 | 可以 | 有源码锚点。 |
| `mpm_twoway_coupling` 有 rigid model 和 sand model。 | 可以 | 有源码锚点。 |
| 当前 Newton 有任意 solver pair 的通用 implicit coupling API。 | 不可写 | FAQ 把 implicit coupling 写成 roadmap。 |
| viewer output 可以证明 coupling 正确。 | 不可写 | viewer 只读/记录/展示。 |
| 修改 bridge step order 会改变 source-of-truth timing。 | 可以 | step order 在源码里显式写出。 |
