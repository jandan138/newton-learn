---
chapter: 04
title: 场景描述与 USD 解析
last_updated: 2026-04-19
source_paths:
  - newton/usd.py
  - newton/_src/utils/import_usd.py
  - newton/_src/usd/utils.py
  - newton/_src/usd/schema_resolver.py
  - newton/_src/usd/schemas.py
  - newton/_src/sim/builder.py
  - newton/_src/sim/model.py
paper_keys: []
newton_commit: 1a230702
---

# 04 场景描述与 USD 解析

## 0. `03` 之后还缺的那一段桥

`03_math_geometry` 已经先把 `shape_transform`、`joint_X_p / joint_X_c`、`body_com`、`body_inertia` 这些词翻译成人话了。你现在再去看 `Model` 或 builder，不会再完全看不懂字段本身。

但新的卡点会立刻出现：这些字段不是手写进去的，它们最初从哪里来？场景里的 prim、joint、collider、material-like attrs，到底怎样被整理成 Newton 的 `Model` 结构？

这章只补这段桥。它不是完整 OpenUSD 教程，也不是 importer internals 大巡游，更不会提前把 articulation 或 collision 细节讲完。你现在真正需要的，是先把“场景输入怎样长成 `Model`”这条链读顺。

如果你这时想找真实入口函数和源码落点，直接去 `source-walkthrough.md`；这页只保留概念桥，不展开源码锚点目录。

## 1. 先看一个最小 scene，再把主线压成五步

先想一个最小 scene：一块地面、一个 box、外加一个把 box 挂到父节点上的 revolute joint。第一遍读 chapter 04，最先该问的不是某个 schema resolver 具体叫什么，而是：这些 authored prim 和局部 pose，最后怎样进到 `Model` 的 body / joint / shape 数组里？

把这个 toy scene 带在脑子里，再去看下面这条链，会更容易知道每一步到底在搬什么：

1. scene input 先 author 一棵 scene graph，里面有 rigid body、joint、shape、site、material 和一堆局部 pose / scale / attrs。
2. importer 读出 Newton 真正关心的那部分信息：body 是谁，joint 连谁，shape 挂在哪，局部 frame 怎样写，质量和接触相关属性有没有 author。
3. schema resolver 把不同生态里的属性写法先归一到同一层物理语义，比如同样在说 self-collision、contact margin、joint limit gain，但名字可能分别来自 `newton:*`、`physx*` 或 `mjc:*`。
4. builder 用 `add_link()`、`add_joint_*()`、`add_shape_*()` 把这些信息累积成 Newton 自己的 body / joint / shape 结构，并顺手累积质量属性。
5. `finalize()` 再把这份还在施工中的 builder 冻成 `Model`：做验证、定型 geometry、把 Python 侧列表搬成真正用于仿真的静态数组。

所以 chapter 04 的最小心智模型可以先非常朴素：scene graph 不是直接等于 `Model`，中间隔着一层“读出来”，一层“翻译统一”，一层“累积成 Newton 结构”，最后才“冻结成 `Model`”。

## 2. scene graph 里哪些东西会变成 `Model` 字段

第一遍看 importer，最好一直把 scene graph 里的对象往未来的 `Model` 字段上对。

- rigid body prim：最后会落成一个 body 槽位。它的位姿先进入 builder 里的 body pose，后面落成 `body_q`；它的质量属性最后落成 `body_mass / body_com / body_inertia`。
- articulation root：它通常不是“又多一个刚体”，而是在告诉 importer 哪些 body / joint 属于同一条 articulation，以及像 self-collision 这样的组级默认值应该怎样取。
- joint prim：`body0 / body1` 这类 parent-child 关系，外加两侧的 local pose，会变成 `joint_parent / joint_child / joint_X_p / joint_X_c`。limit、drive、初始 position / velocity 也会沿这条线继续写进 joint 相关数组。
- collider / visual shape：几何类型、局部位姿、scale、可见性、material-like props，最后会变成 `shape_type / shape_transform / shape_scale / shape_material_* / shape_flags` 这一组 shape 字段。
- site：在 Newton 里更像“非碰撞、零密度的参考点 shape”。它保留局部位置关系，但不应该参与正常质量累计。

这也是为什么 importer 代码里总在做 transform 重写。scene graph 可以有很多中间 `Xform` 节点，但 `Model` 不会保留这棵通用 authoring 树。它只保留后面仿真真正需要的局部关系：shape 相对 body，joint 相对 parent / child，body 再属于某条 articulation。

## 3. 公共入口、真实入口、还有中间那层 importer

对读者来说，`newton/usd.py` 更像 chapter 04 的公共目录。它把常用 USD helper 和 `SchemaResolver*` 这些类型暴露出来，让你先看到 Newton 对 USD 提供了什么公共表面。

真正把场景灌进仿真结构的入口，则在 `ModelBuilder.add_usd()`。它把“我要导入一个 USD stage”这件事挂在 builder 上，然后下接 `_src/utils/import_usd.py` 里的 `parse_usd()`。

`parse_usd()` 做的事情并不神秘。它主要是在做四类读取：

- stage 级信息：比如单位、up axis、PhysicsScene 默认值。
- 结构对象：rigid body、articulation root、joint。
- 几何对象：collider、visual shape、site。
- 辅助属性：material、solver-ish attrs、自定义 attribute、路径映射。

其中一批很关键的 bookkeeping 是 `path_body_map`、`path_joint_map`、`path_shape_map`。这类表面上看起来很“parser”的结构，其实正是在把 scene graph 的 prim path 翻译成 builder 里的整数索引。没有这一步，后面 joint 就没法稳定地说“我到底连的是哪个 body”。

## 4. `schema resolver` 先理解到“属性翻译边界”就够了

第一遍完全没必要把 schema resolver 看成另一套大框架。它最实用的一层意义其实很简单：同一个物理意思，在不同 authoring 习惯里名字不一样，importer 不想把这些分支硬编码得到处都是。

几个最典型的例子就够了：

- articulation 的 self-collision，可能来自 `newton:selfCollisionEnabled`，也可能来自 `physxArticulation:enabledSelfCollisions`。
- 接触包络相关的量，Newton 自己可能写成 `newton:contactMargin / newton:contactGap`，PhysX 则更接近 `restOffset / contactOffset`，MuJoCo 又会写成 `mjc:margin / mjc:gap`。
- joint 的 limit gain、初始 position / velocity，也可能用完全不同的命名空间 author。

`SchemaResolverManager` 做的事，就是按优先顺序问这些 resolver：“这个 prim 上，对我们现在想要的那个逻辑属性，有没有 author 的值？”谁先答上来，就先用谁；都没有，再回退到调用方或 builder 默认值。

所以 resolver 的边界要先记清：它负责把 authored schema 名字翻译成 importer 能消费的统一语义键，但它不负责读取 mesh 几何、不负责建立 joint 拓扑，也不负责直接生成 `Model`。

## 5. builder 才是 scene input 真正压成 Newton 结构的地方

一旦 body、joint、shape 的关系被 importer 读清，builder 就开始接手“把这些关系写成 Newton 自己的数据结构”。

这一步里最重要的三个动作是：

- `add_link()`：给 body 开一个槽位，把 label、初始 pose、kinematic / custom attrs 之类先挂上去。
- `add_joint_*()`：把 parent / child 连接关系和 joint 两侧的局部 frame 收进去，真正形成 `joint_parent / joint_child / joint_X_p / joint_X_c` 这条结构线。
- `add_shape_*()`：把 shape 类型、局部 pose、scale、接触参数、可见性、所属 body 写进去，形成 `shape_type / shape_transform / shape_scale / shape_body` 这一组静态结构。

这里有一个非常值得先记住的点：shape 在 builder 里不只是“几何挂件”。对动态 body 来说，只要 shape 有密度、不是 static，也没有被明确锁死惯量，它就会通过 `compute_inertia_shape(...)` 和 `_update_body_mass(...)` 继续贡献 body 的 `mass / com / inertia`。

也就是说，builder 不只是保存 scene graph；它还在把 scene graph 压缩成 Newton 以后真正消费的物理结构。

## 6. mass / inertia 的优先关系，先记桥接版

chapter 04 不需要把所有 MassAPI 细节讲完，但有一条桥接关系一定要先稳住：`body_mass / body_com / body_inertia` 不是单纯“从几何算出来”，也不是单纯“从文件原样拷过来”。它们是按优先关系 resolve 之后的结果。

第一层可以先这样记：

- 默认路径下，shape 会按自己的 geometry + density 在 builder 里累积出 body 级的 mass / com / inertia。
- 如果 body 上 author 了 `MassAPI` 的质量属性，那么 body 级 `mass`、`diagonalInertia`、`centerOfMass` 会优先覆盖这条默认累计线。
- 如果 body 只 author 了一部分，而不是三件都写全，importer 会先用 collider 信息补一份 baseline mass properties，再把 author 过的那部分覆盖上去。
- 在这条 fallback 里，collider 自己 author 的 mass / inertia 会优先于纯几何推导；collider 没写全时，才退回“按 shape geometry 推导 unit-density mass information”。

这就解释了一个很关键的阅读感受：你在 `Model` 里看到的 `body_mass / body_com / body_inertia`，已经不是“原始 scene attr”，也不是“单步临时变量”，而是 importer 和 builder 协商后的最终静态结果。

第一遍还有一个细节很值得知道，但不用深挖：如果 body author 了 mass，却没 author inertia，importer 会倾向于保住现有 shape 累积出来的惯量结构，再按 author 的 mass 做一致性缩放，而不是假装凭空知道一套新的惯量分布。

## 7. `finalize()` 才是 `Model` 边界真正闭合的地方

如果说 builder 还是“边读边搭的施工现场”，那 `finalize()` 就是把现场封板。

在这一步里，builder 会先做结构检查和必要整理，再把 shape geometry、body、joint、world 分组这些列表真正搬进 `Model`。于是你才会看到那批熟悉的静态字段变成可直接给 solver 用的数组：`shape_transform`、`shape_type`、`shape_scale`、`body_mass`、`body_com`、`body_inertia`、`joint_parent`、`joint_child`、`joint_X_p`、`joint_X_c`。

这也是 `Model` 和 scene graph 的边界差异：scene graph 还能保留 authoring 语义、命名空间和中间层级；`Model` 则已经只剩“仿真真正要消费的静态结构”。到了这里，solver 不再关心某个值原来是 `physx*` 还是 `mjc:*`，也不再关心它原来藏在哪个 `Xform` 下面。

## 8. 带着这条桥，分别走向 `05` 和 `06`

如果你读到这里，chapter 04 的任务就已经完成了：你应该能把 `scene input -> importer -> schema resolver -> builder -> finalize() -> Model` 这条链用自己的话复述出来，也知道 scene graph 里的 body / joint / shape / site / mass attrs 最后大概会落到哪些 `Model` 字段。

接下来最自然的两条分叉是：

- 去 `05_rigid_articulation`：继续追 `joint_parent / joint_child`、`joint_X_p / joint_X_c`、`body_mass / body_com / body_inertia` 这些结构，后面怎样真正进入 articulation 的运动学和动力学。
- 去 `06_collision`：继续追 `shape_type / shape_transform / shape_scale / shape_material_*` 这些结构，后面怎样真正进入碰撞过滤、接触生成和几何查询。

所以这章不负责把 USD 生态讲完。它负责的是在 chapter 03 和后面两章之间，补上一句最关键的人话：`Model` 不是凭空出现的，它是场景输入经过一次有边界的翻译和冻结之后留下来的静态物理结构。
