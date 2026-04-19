---
chapter: 07
title: 约束与接触数学总论 例子观察单
last_updated: 2026-04-19
source_paths:
  - newton/examples/basic/example_basic_shapes.py
paper_keys: []
newton_commit: 1a230702
---

# 07 约束与接触数学总论 例子观察单

`principle.md` 已经把 chapter 06 那张球贴地的图继续讲成 `Contacts -> rows -> Jacobian -> Delassus`。这一页不再加新术语，只把两组最值钱的观察任务列出来，让你把“接触几何”真地看成“接触方向上的运动限制”。

建议入口仍然是最简单的:

```bash
python -m newton.examples basic_shapes
```

这个例子里本来就同时有 sphere、box 和 ground，所以很适合把 chapter 06 和 chapter 07 连着看。第一遍不必关心所有 shape，只盯 `sphere-ground` 和 `box-ground` 这两组画面。

## 主例子: `sphere-ground`

在 `newton/examples/basic/example_basic_shapes.py:L45-L54`，地面和球的最小配置已经摆好了:

- `builder.add_ground_plane()` 给你一个最简单的 world-static shape。
- `self.sphere_pos = wp.vec3(0.0, -2.0, drop_z)` 和 `builder.add_shape_sphere(...)` 给你 chapter 06 那张最小球贴地图。
- `self.model.collide(self.state_0, self.contacts)` 和 `self.viewer.log_contacts(...)` 则把 contact 变成可见箭头。

### 它最适合验证什么

- 一处最简单的接触为什么不会在数学上只剩一个标量。
- 法向方向和切向方向为什么都来自同一个 contact frame。
- `rigid_contact_count` 看到的通常只是“几何 contact 数”，而不是 solver 里真正会用到的 row 数。

### 改哪里最值钱

| 改哪里 | 预期哪个量会先变化 | 这能帮你理解什么 |
|--------|--------------------|------------------|
| 先只改 `drop_z`，让球离地更近或更远 | `rigid_contact_count` 第一次变成非零的时刻会前后移动 | chapter 06 的 contact 几何是怎么出现的 |
| 把 `radius=0.5` 改大或改小 | 接触出现得更早或更晚，接触点位置也会跟着变 | 法线和距离是几何给定的，不是 solver 先发明的 |
| 在理解垂直下落后，再给球一个初始水平速度 | 画面上的 contact 可能仍只有一处，但你会更容易接受“同一个 contact 还需要切向方向来描述滑动” | 为什么 `1 contact` 不等于 `1 row` |

### 第一遍最值得观察什么

- contact arrow 的朝向是不是稳定地沿着地面法线。
- 球开始接触以后，画面上虽然通常只看到一个接触点，但你已经可以把它脑补成一个局部坐标系: `1` 条法向、`2` 条切向。
- 如果你给球加了水平速度，画面上多出来的不是“第二个接触点”，而是同一个接触点上的切向相对运动变得重要了。
- 如果你愿意多看一眼字段层，可以顺手盯 `rigid_contact_count` 和 `rigid_contact_normal`。它们会比箭头更直接地提醒你: 现在看到的还是几何 contact，不是 solver rows。

### 如果结果和预期不一样, 你大概率误解了什么

- 如果你以为“只看到一个 contact arrow，所以 solver 里就只有一条约束”，那你把几何 contact 和 solver row 混成了一件事。
- 如果你以为 Jacobian 是后来凭空出现的矩阵，那你忽略了这个例子里已经有最关键的输入: 接触方向和接触点位置。

## 对照例子: `box-ground`

同一个文件里，`newton/examples/basic/example_basic_shapes.py:L75-L78` 已经给了最小的 `box-ground` 场景。和球相比，箱子最值钱的地方不是“更复杂”，而是它能把 chapter 07 里两件更难的事同时暴露出来:

- 一个 shape pair 可以长出多个 contact。
- 接触点离质心偏得更明显时，角向耦合会更早变得刺眼。

### 先验证什么

- 平放落地时，`box-ground` 比 `sphere-ground` 更容易出现多个几何 contact。
- 轻微倾斜落地时，最先接触的点通常离箱子质心更远，所以“法向作用也会带出转动”会更容易看见。

### 改哪里最值钱

| 改哪里 | 预期哪个量会先变化 | 这能帮你理解什么 |
|--------|--------------------|------------------|
| 先保持 `hx / hy / hz` 不变，只把 box 初始朝向从 `quat_identity()` 改成轻微倾斜 | 刚接地时更可能先出现角点或边缘接触，箱子会一边被顶住一边回正 | Jacobian 为什么天然会有角向部分 |
| 再把箱子尺寸改得更扁或更宽 | contact 分布和接触点到质心的杠杆臂会变化 | effective mass / Delassus 为什么不只和质量有关，也和几何作用点有关 |
| 等你看懂单箱子后，再让箱子从更低或更高处落下 | contact 数和姿态变化会更明显，但主线仍然是同一个 pair 可能长出多个 contact | 多个 contacts 不是另一个章节才有的东西，而是这章必须接受的现实 |

### 第一遍最值得观察什么

- 平放着地后，box-ground 往往会比 sphere-ground 更容易出现多个接触箭头。这时最该建立的直觉是: 一个 pair 可以贡献多个 contact，而每个 contact 又各自会长成三条 rows。
- 倾斜着地时，如果先只有一个角接触，你应该更容易看到“这个点不在质心上”。哪怕画面里只是一根 arrow，它也已经在暗示后面的角向耦合。
- 当箱子从单角接触逐渐过渡到底面多点接触时，你可以把它脑补成: solver 后面的 rows 数在增长，而且这些 rows 都共享同一个箱子的平移和转动自由度。

### 如果结果和预期不一样, 你大概率误解了什么

- 如果你把“shape pair 数”和“contact 数”直接画上等号，你就会低估 `box-ground` 这类面接触的复杂度。
- 如果你只盯接触点位置，却没注意它离质心多远，你就会很难理解为什么同样一条法向约束也会牵出转动。
- 如果你把 Delassus 只想成一个针对单点的小数字，你会看不见多个 contacts 之间通过同一刚体产生的耦合。

## 这页怎么配合其他文件

- `principle.md`: 负责把这两张图的数学直觉讲顺。
- `source-walkthrough.md`: 负责把这里的观察点钉回 `Contacts`、`ContactsKamino`、Jacobians 和 Delassus 的源码。
- `examples.md`: 只负责帮你在最小例子里看到三件事: `1 contact != 1 row`、`1 pair` 可能有多个 contact、偏心接触会带出角向耦合。

如果这两组观察都能看顺，你再去读 `08_rigid_solvers`，就更容易接受 solver 面前摆着的不是“一个抽象矩阵问题”，而是一组从具体接触图里长出来的 rows 和 constraint-space quantities。

## 自检

- 现在只看 `sphere-ground`，你能不能解释为什么“画面上 1 个接触点”不等于“solver 里只剩 1 条 row”？
- 现在只看 `box-ground`，你能不能解释为什么“接触点离质心更远”会让同一条法向方向也更容易牵出转动？
