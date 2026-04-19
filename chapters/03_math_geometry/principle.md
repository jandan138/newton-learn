---
chapter: 03
title: 数学与几何基础
last_updated: 2026-04-18
source_paths:
  - newton/_src/math/
  - newton/_src/geometry/
  - newton/_src/sim/articulation.py
  - newton/_src/sim/builder.py
paper_keys: []
newton_commit: 1a230702
---

# 03 数学与几何基础

## 0. 为什么 `02` 之后还会卡住

`02_newton_arch` 已经把 `Model / State / Control / Solver` 这条主链接上了，但你真的去翻 `Model`、builder 或 articulation 源码时，马上会撞上另一层词汇：`wp.transform`、parent / child frame、四元数、`shape_transform`、`body_com`、twist、wrench、惯量。

这章只负责把这层词汇翻译成人话，让 `Model` 先变得可读。它不是完整数学课，不会把四元数、刚体动力学、碰撞算法全部讲完。你现在需要的不是一套推导，而是一套足够稳定的阅读坐标。

如果把 `02` 看成“Newton 里有哪些核心对象，step 怎样接起来”，那 `03` 更像“这些对象内部为什么会长成这种几何样子”。读完这一章，你不用立刻会推公式，但应该能看懂这些量各自在服务谁。

## 1. 先把 `frame` 看成“谁在拿谁当参考系”

第一遍读 Newton，不要把 frame 想得太抽象。它就是一个带原点和朝向的参考系。只要有 frame，你就可以说“这个点的位置是相对谁写的”“这根轴的朝向是在哪个坐标系里表达的”。

这也是为什么 `Model` 里很少把一切都直接写成 world 坐标。`body` 有 body frame，joint 有 joint frame，shape 也有自己的局部 frame。这样做不是为了增加术语，而是为了让部件能被组合、复制、挂到不同父节点上，而不是每换一个场景就重写一遍绝对坐标。

第一遍看 builder 时，下面四个名字最值得先认熟：

- `shape_transform`：shape 相对所属 body 的局部位姿。
- `joint_X_p`：joint frame 从 parent body 那一侧看过去的位姿。
- `joint_X_c`：同一个 joint frame 从 child body 那一侧看过去的位姿。
- `body_com`：body 的质心在 body 局部 frame 里的位置。

如果你一时分不清一个量在说什么，先问一句：它是谁相对谁写的？多数几何阅读困难，卡的都不是公式，而是 frame 没对齐。

## 2. `wp.transform` 先只读成“平移 + 旋转”

`wp.transform` 第一遍完全可以翻译成“一个位置 + 一个朝向”。位置告诉你 frame 原点在哪里，朝向告诉你这个 frame 相对参考系转成了什么方向。先把它看成 rigid placement 的容器，不必一上来就脑补完整矩阵代数。

在 chapter 03 这个深度里，`wp.transform` 主要在做三件事：

1. 说明 joint 锚点怎样落在 parent / child 两侧的局部 frame 里。
2. 说明 shape 怎样挂在 body 上，而不是天然长在 body 原点。
3. 说明某个局部几何量怎样被搬到 body frame 里，例如把 shape 的质心和惯量转到 body 上再累加。

所以当你看到 transform 组合时，可以先把它翻译成一句很朴素的人话：局部关系在一层层往外接。builder 代码之所以看起来像不断在拼 transform，本质上是在维护“shape 属于 body，body 通过 joint 接到 parent，最后才落到 world”这条局部链。

这一层想稳后，`Model` 里那些局部位姿就不再像离散数字，而会变成一张结构图。

## 3. 四元数和旋转：先会读，不急着会推

四元数在这里的任务也不神秘。它只是 Newton 选择的一种旋转表示，让 orientation 能稳定地存、乘、逆和插入到 transform 里。你现在不需要把它当成一门独立课程，只要先认出几个最常见动作。

第一遍最有用的四个读法是：

- `wp.quat_identity()`：没有旋转。
- `wp.quat_from_axis_angle(axis, q)`：绕某根轴转一个角度。
- `q2 * q1`：把旋转按顺序接起来。
- `wp.quat_rotate(...)` 或相关 helper：拿这个旋转去转一个方向或局部轴。

这已经足够支撑你读 articulation 里的大部分关节代码。revolute joint 很自然地长成“围绕 joint axis 旋转一个角度”；ball joint 则直接把一个完整方向状态存成 quaternion。`quat_decompose()` 这类 helper 的意义，也先理解成“把相对朝向重新拆回更适合关节坐标的读法”，而不是邀请你现在就跳进 Euler angle 的全部细节。

`_src/math/spatial.py` 里还会出现 `quat_between_vectors_robust()` 这样的名字。它提醒你另一件很实用的事：很多时候四元数不是在做抽象数学，而是在回答一个工程问题，“把这根方向安全地转到另一根方向上去”。

如果你现在能把 quaternion 看成“orientation 的工作容器”，而不是“四个看不懂的数”，这一节就够用了。

## 4. spatial quantities：先当成“挂在 frame 上的运动和受力”

一旦 frame 和 transform 稍微稳一点，twist 和 wrench 就没那么吓人了。它们不是额外一套神秘物理，而是在把“刚体整体怎么动”“刚体整体受了什么”打包成适合刚体计算的 6D 量。

- twist：把线速度和角速度放在一起看。
- wrench：把力和力矩放在一起看。

在 Newton 的第一遍阅读里，你不必先推 spatial algebra。更有用的是养成三个问题：

1. 这个量是在哪个 frame 里表达的？
2. 这个运动是相对 body 原点，还是相对质心？
3. 现在这段代码是在换 frame，还是在把量从一个点移到另一个点？

这正是 `_src/math/spatial.py` 里 `transform_twist()`、`transform_wrench()`、`velocity_at_point()` 在帮你做的事。它们不是新的物理章节，而是在提醒你：同一个刚体运动，换个参考 frame、换个观察点，读出来的线速度和力矩会变。

`velocity_at_point(qd, r)` 尤其值得先记住，因为它几乎就是一句人话：已知刚体整体 twist，再给我一个相对该 frame 原点的偏移 `r`，这个点的线速度是多少。你在 articulation 或 contact 代码里看到它时，不必先想 6x6 矩阵，只要先翻译成“把 body 整体运动换算到某个具体点上”。

还有一个很值钱的阅读提醒：Newton 在这些 wrapper 里刻意把 twist / wrench 保持成更贴近人类阅读的顺序，twist 读成“线速度 + 角速度”，wrench 读成“力 + 力矩”，再在需要时和 Warp 的底层 helper 对接。第一遍看源码时，先抓这个语义顺序，比死背底层布局更重要。

## 5. shape representation：先分清“这是哪一类几何”

`Model` 里不只有 body 和 joint。shape 也同样重要，因为后面的碰撞、可视化、质量属性，都会从 shape 出发。第一遍真正需要的，不是把所有碰撞算法吃透，而是先分清 Newton 在保存哪一类几何。

第一次看到 `HFIELD`、`SDF` 这类大写词，也不用先脑补另一套新学科。对 chapter 03 来说，它们先只是 `_src/geometry/types.py` 这张目录里的表示名，提醒你：后面的查询路径会因为 shape category 不同而分叉。

| 表示 | 第一遍先怎么理解 | 现在为什么重要 |
|------|------------------|------------------|
| primitive：`SPHERE`、`BOX`、`CAPSULE`、`CYLINDER`、`CONE`、`ELLIPSOID` | 参数化、解析定义的基础形状。 | 它们最容易直接看懂，也最常被 builder 和碰撞代码特殊处理。 |
| `MESH` / `CONVEX_MESH` | 来自顶点和三角形，或其凸包近似。 | 说明几何不再只是几个半径和半长，而是显式数据。 |
| `HFIELD` | `GeoType` 里的 heightfield，高度网格形式的地形表示。 | 它提醒你不是所有几何都值得塞成通用 mesh。 |
| SDF | 在 `GeoType` / `Mesh.build_sdf()` 这条线里，表示 signed distance field 这种“怎样问表面距离和法向”的表示方式。 | 它会在后面的 `06_collision`（包括 hydroelastic 相关路线）里重新出现。 |

`_src/geometry/types.py` 里把这些类型先分成 primitive 和 explicit 两大类，这个切分对初学者非常有帮助。前者更像“我知道这个形状的参数公式”，后者更像“我手里有一份几何数据”。你先把这个区别记稳，后面看到不同 narrow phase 或 SDF helper 时，就不会误以为它们全是同一种东西。

同时也别忽略 `shape_transform`。shape 类型回答的是“它是什么形状”，而 `shape_transform` 回答的是“它在 body 局部 frame 里放在哪、朝哪”。这两件事一起，才构成真正可用的几何描述。

## 6. 惯量是 geometry 通向 articulation 的桥

这一章里最关键的桥，不是某个公式，而是一个转折点：shape 一旦不只是拿来碰撞或显示，它就必须贡献质量属性。也就是质量、质心和惯量。

这正是 `_src/geometry/inertia.py` 和 builder 在接力做的事。`compute_inertia_shape(...)` 先根据 shape 类型、尺寸、密度和是否实心，给出该 shape 自己的 `mass / com / inertia`。这些量先是局部的，先属于 shape 自己。

接着 builder 再用 shape 的 transform 把这些局部质量属性搬到 body frame 里，并通过 `transform_inertia(...)` 和 `_update_body_mass(...)` 把多个 shape 的贡献累到同一个 body 上。于是 `body_mass`、`body_com`、`body_inertia` 才真正出现。

这一步非常值得你先用一句话记住：geometry 到这里不再只是“长什么样”，而是被压缩成“动起来时有多重、重心在哪、绕不同方向有多难转”。这就是 chapter 03 和 chapter 05 之间最重要的接口。

第一遍不需要推平行轴定理，也不需要自己手算惯量矩阵。你现在只需要知道：

- shape 提供局部几何和密度。
- inertia helper 把它变成局部质量属性。
- builder 把局部质量属性搬到 body 上并累加。
- articulation 之后消费的，正是这些已经整理好的 body 级结果。

只要这条桥接线是通的，很多 `Model` 字段就会突然变得合理。

## 7. 带着这套词汇，分别走向 `04`、`05`、`06`

如果你读到这里，chapter 03 的目标其实已经达成了大半：你不一定会完整推导这些量，但应该能把它们翻译成人话。`joint_X_p` / `joint_X_c` 不再只是神秘数组，`shape_transform` 不再只是局部位姿噪声，`velocity_at_point()` 不再像突然冒出来的技巧函数，惯量也不再像和 geometry 毫无关系的另一门课。

接下来最自然的分叉有三条：

- 去 `04_scene_usd`：如果你现在更想问“这些 body / joint / shape / frame 到底从场景输入的哪里来”。
- 去 `05_rigid_articulation`：如果你更想问“这些 frame、twist、质心和惯量怎样真正进入 articulation 的运动学与动力学”。
- 去 `06_collision`：如果你更想问“这些 primitive / mesh / heightfield / SDF 表示，后面到底怎样被碰撞系统消费”。

所以这章不负责把数学讲完，而是负责把后面三章共用的词汇垫平。你能带着“frame -> transform -> quaternion -> spatial quantity -> shape -> inertia”这条链继续往下读，chapter 03 就完成任务了。
