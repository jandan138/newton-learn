---
chapter: 06
title: 碰撞系统
last_updated: 2026-04-23
source_paths:
  - docs/concepts/collisions.rst
  - newton/_src/sim/model.py
  - newton/_src/sim/collide.py
  - newton/_src/geometry/broad_phase_common.py
  - newton/_src/geometry/broad_phase_nxn.py
  - newton/_src/geometry/broad_phase_sap.py
  - newton/_src/geometry/narrow_phase.py
  - newton/_src/geometry/collision_primitive.py
  - newton/_src/geometry/contact_data.py
  - newton/_src/sim/contacts.py
paper_keys: []
newton_commit: 1a230702
---

# 06 碰撞系统

## 0. 先看一个球落到地面上

先看一个更直观的画面：一个球挂在离地面几毫米的地方，下一步仿真开始时，系统已经知道球对应 body 的世界位姿，也知道地面还在那里。现在真正要回答的问题其实很朴素：这个球壳现在在世界里哪儿，它可能先靠近谁，最后又要把“靠近”写成什么样的数据，才能交给后面的接触数学和求解器。

如果这时你脑子里先冒出来 broad phase、narrow phase，甚至 GJK、MPR 这些名字，也先别急着往术语上跳。chapter 06 第一遍更值得保住的，是这幅球快要落地的具体画面。

chapter `05_rigid_articulation` 已经把 `joint_q / joint_qd` 怎样长成 `body_q / body_qd` 讲顺了。chapter 06 要补的，就是从这份 body/world state 再往前走半步，把它接到真正的碰撞输入上。

如果把这一章先压成一条最小主线，其实就是：

```text
state.body_q + model.shape_transform / shape_type
-> shape 在 world 里的当前位置
-> 一张“也许会撞”的 shape pair 名单
-> 具体 contact 几何
-> Contacts
```

![从 `body_q` 到 `Contacts` 的碰撞桥接图](assets/06_collision_bridge_map.png)

这张图只服务 first pass：先把 `body_q -> shape -> candidate pairs -> narrow phase -> Contacts` 这条 runtime bridge 看成同一条链，而不是一上来就掉进 broad phase / narrow phase 的具体算法细节。你现在先守住“哪一层在提供位姿，哪一层在提供几何，哪一层在写统一 contact 缓冲区”就够了。

第一遍只要把下面这张表记住，后面的名字就不容易糊成一团：

| 人话里的意思 | 球落地时你脑中可以看到什么 | 第一次代码落点 |
|---|---|---|
| 先把 shape 摆到世界里 | 球的局部 collider 跟着 body 一起移动，落到离地面很近的位置 | `newton/_src/sim/collide.py` 里的 `compute_shape_aabbs(...)` |
| 先做一张便宜的 maybe-list | 大部分离很远的 shape 对先被排掉，只留下“也许会碰”的 pair | `broad_phase_common.py`、`broad_phase_nxn.py`、`broad_phase_sap.py` |
| 再把 maybe 变成真实接触几何 | 球和地面到底有没有接触点、法线朝哪、距离是正还是负 | `narrow_phase.py`、`contact_data.py`、`sim/contacts.py` |

## 1. 先纠正一个最容易带错的想法：不是 body 在直接碰撞

第一次读 collision path，最常见的误会是把它想成“body 和 body 在碰撞”。更准确的人话是：**body 提供运动，shape 提供几何，真正直接参加碰撞的是 shape。**

这件事在球落地的例子里很好想。你眼里看到的是“球碰地面”，但运行时真正更像下面这样：

- `state.body_q` 先告诉系统：球那个 body 现在在 world 里什么位置。
- `model.shape_transform` 再告诉系统：球的 collider 相对 body 自己偏在哪。
- `model.shape_type`、`shape_scale`、`shape_source_ptr` 这类静态信息继续说明：这到底是 sphere、box、mesh，还是别的几何。

所以碰撞入口真正先做的，不是问“哪两个 body 打架了”，而是先把“挂在 body 上的 shape 现在在哪儿”算出来。

这也是为什么一个 body 上可以挂多个 shape，而一个 body pair 也可能长出多个 shape pair。比如一个机器人脚掌 body 上既有脚底 box，又有脚尖小球，那么“脚和地面”这个人话，在碰撞器眼里其实已经是好几组 shape 关系了。

如果你是从 chapter 05 带着 `body_qd` 走过来的，这里还有一个小提醒：刚进入 rigid collision 时，最先决定“谁可能靠近谁”的主要是当前位置，也就是 `body_q` 经过 shape 局部位姿后的结果。`body_qd` 当然还重要，但它的价值更明显地体现在后面的接触响应、相对速度和 solver 消费里，而不是这一章最前面的候选筛选里。

## 2. broad phase 先做的，只是一张便宜的“也许会撞”名单

另一种很常见的误会是：包围盒重叠了，就等于已经碰撞了。不是。

更好的第一遍读法是：在真正精算几何之前，系统先想办法**便宜地排掉绝大多数明显不可能接触的 pair**。这一层做完之后，留下来的不是“已经撞上的 pair”，而是“值得继续看一眼的 pair”。

这就是 broad phase 的第一层意义。你完全可以先把它记成人话版的 maybe-list。

为什么要这样做，也很直观。场景里如果有很多 shape，直接让每一对都做精细几何查询会太贵。于是系统先给每个 shape 套一个保守的外盒，也就是 AABB，然后只看这些便宜盒子有没有重叠，再顺手应用几类简单过滤：

- 不在同一个 world 的，一开始就不用比。
- collision group 明确不该碰的，一开始就排掉。
- 显式禁碰的 shape pair，也不用进后面的几何查询。

这一步最值得先守住的一句话是：**broad phase 宁可多报，也不能轻易漏报。**

也就是说，两个 shape 的外盒重叠，只说明“它们也许值得进下一轮”；外盒不重叠，才说明“这次大概率不用再看”。

Newton 里这张 maybe-list 可以有不同做法，但第一次读不用把算法名当主角。只要先记住两种味道就够了：

- 小一些的场景，可以直接做 all-pairs 式的保守筛选。
- 大一些的场景，可以用 sweep-and-prune 这类办法，先排序再找可能重叠的区间。

名字后面再记都来得及。真正重要的是：broad phase 的输出是 **candidate shape pairs**，不是最终 contact。

## 3. narrow phase 才把 maybe-pair 变成具体的接触几何

到了这一步，系统手里已经有了一批“也许会撞”的 shape pair。narrow phase 做的事，才是真正把其中的一部分变成“后面章节能继续用”的接触几何。

还是回到球落地。假设 broad phase 已经把“球 shape vs 地面 shape”留了下来，narrow phase 现在要回答的就会具体很多：

- 球和地面最近的是哪一块位置。
- 如果把这次接触写成一个或几个 contact，点应该落在哪。
- 法线该朝哪边。
- 两个形状沿法线方向到底还隔着一点缝，还是已经压进去了。

这里有三个词，最好一开始就用可想象的画面去记：

- `contact normal`：先把它想成“把两个形状推开的方向”。球压在地面上时，你脑中看到的它大致就是朝上的。
- `signed distance`：先把它想成“沿法线量出来，还隔着多少，或者已经压进去多少”。正值表示还有缝，负值表示已经发生穿入。
- `gap / margin`：先把它想成“多早开始把接近当回事，以及表面算多厚”。它们让系统不必等到几何真的完全交错之后才开始记 contact。

第一次读到这几个词时，不要急着把它们全变成公式。先记住球的画面就够了：球刚离地一点点时，距离还是正的；球再往下压，距离会过零变负；法线则提供了“应该沿哪边把它们分开”的方向。

还有一个很值钱的提醒：**一个 shape pair 不一定只产出一个 contact。**

球压地面时，脑子里常常只会想到一个点，所以这点容易被忽略。但如果换成一个 box 落到平面上，narrow phase 完全可能写出多个 contact，用来更稳定地表示“这个面正在贴着地”。所以“pair 数量”和“contact 数量”不是一回事。

## 4. 为什么 narrow phase 里面还要按 shape type 分路

如果 broad phase 只是保守筛选，那 narrow phase 之所以会显得更复杂，一个关键原因就是：不同几何对，真正要解的问题并不一样。

第一次可以先别把它读成“一堆算法名称”，而是读成一句非常工程的话：**不是每对 shape 都值得走同一条几何路径。**

下面这张表，先帮你把这种分路感建立起来：

| 看到的 shape pair | 第一遍先怎么想 | 第一次代码落点 |
|---|---|---|
| plane-sphere、sphere-sphere、capsule-capsule 这类简单 pair | 直接用比较轻的解析几何公式就能给出 contact | `newton/_src/geometry/collision_primitive.py` |
| box-box、convex-convex 这类更一般的凸体 pair | 进入统一的凸体接触查询，不再为每一对手写一套公式 | `newton/_src/geometry/narrow_phase.py` |
| mesh、heightfield、SDF、hydroelastic 相关 pair | 进入更重的 mesh / SDF 分支，必要时再做接触整理或 reduction | `newton/_src/geometry/narrow_phase.py` |

所以 `shape_type` 在这章里最有价值的读法，不是“我要背下所有类型枚举”，而是“它决定候选 pair 接下来该走哪条几何支路”。

你甚至可以把 narrow phase 想成一个分诊台：

- 简单 primitive pair，直接走轻量公式。
- 一般凸体 pair，走统一凸体查询。
- mesh 或 SDF 相关 pair，走更重但更合适的几何路径。

无论前面分成多少支，chapter 06 真正关心的不是推导每一条支路，而是看清这个边界：**shape-type branching 发生在 narrow phase 里，但 downstream 不想继续替每一条支路维护一套完全不同的接口。**

## 5. `ContactData` 和 `Contacts`：把复杂几何压成统一 handoff object

到了这里，碰撞流水线最关键的一次“压缩”就出现了。

窄相位内部先会有一个更接近几何查询的中间表示，比如 `ContactData` 里会带着接触中心、法线、接触距离、两侧 shape id、margin 这类信息。它更像是在说：“这一对 shape 的几何关系，我已经算出来了。”

但真正交给运行时后半段的，不会是“某个 mesh 三角形上的特殊故事”，而是一份统一的 `Contacts` 缓冲区。`Model.contacts()` 负责分配它，`Model.collide(state, contacts)` 负责把这一帧的碰撞结果写进去。

第一遍可以先把 `Contacts` 读成下面这张表：

| 人话里的问题 | 在 `Contacts` 或其直接上游里对应什么 | 为什么后面章节要关心 |
|---|---|---|
| 这是谁和谁的接触？ | `rigid_contact_shape0 / rigid_contact_shape1`，以及上游 `shape_body` 映射 | `07`、`08` 都得先知道接触属于哪两个 shape，再回到 body |
| 两边各自觉得接触点在哪？ | `rigid_contact_point0 / rigid_contact_point1` | 之后要从这些点建立相对运动、约束和力的作用点 |
| 推开的方向朝哪？ | `rigid_contact_normal`，上游则是 `ContactData.contact_normal_a_to_b` | 这是后面法向约束、摩擦方向的几何入口 |
| 现在还隔着多少，还是已经压进去了？ | 上游 `ContactData.contact_distance`，以及可选的 `rigid_contact_diff_distance` | `07` 会把这类几何量继续翻译成接触数学 |
| 表面厚度和接触包络怎么算？ | `rigid_contact_margin0 / rigid_contact_margin1`，以及 pair 级 `gap` | 这决定 contact 何时出现，也影响后续稳定性 |

这里最值得先稳住的感受是：solver 不想知道你这个 contact 原来来自 plane-sphere、box-box，还是 mesh-SDF。它更想拿到一份统一格式的数据，然后继续做后面的数学。

这也解释了 chapter 06 的运行时边界为什么很清楚：

- 章的输入边界，是 `state` 里的当前 body/world pose，加上 `Model` 里的 shape 静态元数据。
- 章的输出边界，是已经写好的 `Contacts`。

到这一步，碰撞器的任务就算完成了。后面如果谁还想继续问“这个 contact 对系统意味着什么”，那已经不是碰撞几何本身，而是接触数学和求解的问题了。

## 6. 带着这条桥，明确交给 `07` 和 `08`

如果你读到这里，chapter 06 的桥接任务就已经完成了。你应该已经能把下面这条话，用自己的语言说顺：

不是 body 直接碰撞，而是 body 带着 shape 进入 world；broad phase 先用便宜、保守的办法做一张 maybe-list；narrow phase 再按 shape type 把 maybe-pair 变成真正的 contact geometry；最后这些几何结果被压进统一的 `Contacts`，交给后面的章节继续消费。

接下来最自然的 handoff 有两条，而且它们负责的问题不一样：

- 去 `07_constraints_contacts_math`：如果你下一步想问“这些 contact 点、法线、距离，怎样继续长成 Jacobian、法向约束、摩擦和更完整的接触数学”。
- 去 `08_rigid_solvers`：如果你下一步想问“不同 rigid solver 怎样消费同一份 `Contacts`，并把它真正变成速度更新、位置修正或接触响应”。

所以这章到这里就停。它不推 GJK / MPR / EPA，也不抢 `07` 的约束数学，更不提前进入 `08` 的 solver 细节。它只负责把 chapter 05 留下的 `body_q / body_qd`，稳稳地接到 shape、candidate pair 和 `Contacts` 这条碰撞主线上。
