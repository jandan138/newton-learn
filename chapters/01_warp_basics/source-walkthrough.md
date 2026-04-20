---
chapter: 01
title: Warp 编程模型
last_updated: 2026-04-20
source_paths:
  - newton/_src/sim/model.py
  - newton/_src/solvers/xpbd/solver_xpbd.py
  - newton/_src/solvers/xpbd/kernels.py
  - newton/examples/selection/example_selection_materials.py
  - newton/examples/mpm/example_mpm_twoway_coupling.py
  - newton/examples/diffsim/example_diffsim_bear.py
paper_keys:
  - warp-paper
  - mujoco-warp-paper
newton_commit: 1a230702
---

# 01 Warp 编程模型 源码走读

这份主 walkthrough 先回答一个真正的新手最容易卡住的问题：为什么 Newton 里会出现 Warp，而不是到处写普通 Python `for` 循环。

Newton 每一步仿真里，都有很多“同一条规则要作用到很多对象”的更新。普通 Python 当然也能表达这件事：

```python
for i in range(n):
    out[i] = f(x[i])
```

但问题在于，仿真里这种循环往往不是 10 次、20 次，而是成百上千个 body、joint、particle、contact 一起更新。只靠普通 Python 循环，很快就会在性能上变得吃力。Warp 在 Newton 里的第一层作用，就是把这类普通 loop 拆成更适合批量执行的两层：

- `kernel`：单个 `i` 该怎么做。
- `wp.launch(..., dim=...)`：这次一共发多少个 `i`。

然后再补上另外几块：

- `wp.tid()`：当前这份执行对应哪一个 `i`。
- `wp.array`：这批 `x`、`out` 实际放在哪些缓冲区里。
- `wp.atomic_add`：如果很多个 `i` 都想往同一个地方写，怎样安全累加。
- `Graph` / `tile`：当这套执行已经成立后，怎样进一步组织重复工作和局部复用。

后面整份文件都会反复回到同一条故事线：`ordinary Python loop -> Warp -> Newton 对象`。如果你想先看更慢一点的概念版，可以再配合 `principle.md`；但只读这一份，也应该能把 chapter 01 的主线讲顺。

## What This Walkthrough Follows

这一页只追下面这条执行主线：

```text
ordinary Python loop
-> kernel defines one logical element
-> wp.launch + wp.tid() turn it into a batch
-> Model / State / Control hold the wp.array buffers
-> atomic ops handle shared writeback
-> Graph / tile reorganize execution
```

这一页刻意不展开三类东西：

- Warp 全 API 手册
- XPBD、MPM、DiffSim 的完整物理细节
- 更重的 GPU 优化话题

第一遍先守住一句话：看到 Warp 代码时，先把它翻回“本来那段普通 loop 在批量处理谁”。

## One-Screen Chapter Map

```text
for i in range(n):
    out[i] = f(x[i])

same idea in Newton / Warp:

kernel = one logical i
        |
        v
wp.launch(..., dim=...) + wp.tid() = whole batch + current i
        |
        v
those x / out buffers live on Model / State / Control as wp.array
        |
        v
if many i write one slot, use atomic writeback
        |
        v
Graph / tile reorganize how that execution is submitted or reused
```

## Beginner Path

1. 先看 Stage 1。
   - 想验证什么：为什么 Newton 要把普通 loop 拆成 Warp kernel。
   - 看完后应该能说：`kernel` 不是神秘函数，而是“单个元素规则”。
2. 再看 Stage 2。
   - 想验证什么：没有显式 `for` 循环时，整批执行和当前索引是谁来决定。
   - 看完后应该能说：`wp.launch` 决定批量范围，`wp.tid()` 负责回答“我是哪一个 `i`”。
3. 再看 Stage 3。
   - 想验证什么：这些 `x`、`out` 在真实 Newton 里到底挂在哪些对象上。
   - 看完后应该能说：`wp.array` 往往是 `Model / State / Control` 背后的长期缓冲区。
4. 再看 Stage 4。
   - 想验证什么：为什么有些 kernel 不能只写自己的槽位，而要用 `wp.atomic_add`。
   - 看完后应该能说：atomic 处理的是 many-to-one 共享写回，不是新的物理词。
5. 最后看 Stage 5。
   - 想验证什么：`Graph` 和 `tile` 到底是在改什么层次。
   - 看完后应该能说：它们改的是执行组织方式，不是在改仿真语义。

## Main Walkthrough

### Stage 1: 为什么 Newton 需要 Warp，先把 `kernel` 读成“单个元素规则”

**Definition**

`kernel`：Warp 里的 kernel，是“如果我只负责一个逻辑元素，这一格该怎么算”的规则模板。

**Claim**

Newton 里之所以会出现 Warp，是因为仿真一步里有很多“同一条规则重复作用到很多元素”的更新。Warp 先把这条规则拆出来，写成 kernel。

**Why it matters**

如果一开始不先把 kernel 和普通 loop 对上，后面看到 `wp.launch`、`wp.tid()`、`wp.array` 时，就会像在背新语法，而不是在读同一件事的不同层次。

**Plain loop story**

先把最朴素的版本记住：

```python
for i in range(n):
    out[i] = f(x[i])
```

在真实 Newton 代码里，这个 `i` 不一定是单个整数，也可能是 `(world, arti, dof)` 这样的多维索引。但第一遍先把它都看成“当前这一个逻辑元素”就够了。

**Source excerpt**

`selection_materials` 里的 `compute_middle_kernel`，就是把“逐元素写一个中间值”单独抽成 kernel：

```python
@wp.kernel
def compute_middle_kernel(lower: wp.array3d[float], upper: wp.array3d[float], middle: wp.array3d[float]):
    world, arti, dof = wp.tid()
    middle[world, arti, dof] = 0.5 * (lower[world, arti, dof] + upper[world, arti, dof])
```

**Verification cues**

- 这段 kernel 里真正的业务规则只有一句：把 `middle[...]` 写成上下界的平均值。
- `world, arti, dof` 只是多维版的“当前这个 `i`”。
- 如果不用 Warp，这件事本来就可以写成普通的三重 loop；Warp 只是先把“单格规则”拆出来。

**Checkpoint**

如果你现在还会把 kernel 想成“比普通函数更神秘的东西”，先不要继续。第一遍只把它翻译成“单个 `i` 的规则”就够了。

**Output passed to next stage**

一个稳定起点：看到 `@wp.kernel` 时，先问“它在定义哪一种元素的一次处理规则”。

### Stage 2: `wp.launch` 和 `wp.tid()` 让这条规则真正变成整批执行

**Definition**

- `wp.launch(..., dim=...)`：真正发起这次批量执行的地方，它决定要跑多少个逻辑元素。
- `wp.tid()`：当前这份 kernel 执行拿到的索引，也就是“我是哪一个 `i`”。

**Claim**

在 Warp 里，kernel 只写“单个元素怎么做”；真正决定“这一批是谁、总共多少个”的，是 launch。读源码时必须先看 launch，再回头看 `wp.tid()`。

**Why it matters**

普通 loop 里的 `i` 是你显式写出来的循环变量。Warp 里没有这个显式 `for`，所以新手最容易在这里丢线：明明 kernel 只有几行，却不知道它到底在按 body、joint、contact 还是 particle 批量运行。

**Plain loop story**

还是同一条故事线：

```python
for i in range(n):
    out[i] = f(x[i])
```

这里的 `range(n)`，在 Warp 里就是 launch 的 `dim`；这里的循环变量 `i`，在 Warp 里就是 `wp.tid()`。如果 `dim` 是一个 shape tuple，那么 `i` 也就会变成多维索引。

**Source excerpt**

同一个例子里，真正定义 batch 的是 launch：

```python
wp.launch(
    compute_middle_kernel,
    dim=self.default_ant_dof_positions.shape,
    inputs=[
        dof_limit_lower,
        dof_limit_upper,
        self.default_ant_dof_positions,
    ],
)
```

而 kernel 里的 `wp.tid()` 只是把这次 batch 的索引翻出来：

```python
world, arti, dof = wp.tid()
middle[world, arti, dof] = 0.5 * (lower[world, arti, dof] + upper[world, arti, dof])
```

真实 solver 里也是一样。XPBD 先决定这次 launch 按 soft-contact slot 跑：

```python
wp.launch(
    kernel=solve_particle_shape_contacts,
    dim=contacts.soft_contact_max,
    ...,
    outputs=[particle_deltas, body_deltas],
)
```

然后 kernel 再把 `tid` 翻成具体 contact slot：

```python
tid = wp.tid()

count = min(contact_max, contact_count[0])
if tid >= count:
    return

shape_index = contact_shape[tid]
body_index = shape_body[shape_index]
particle_index = contact_particle[tid]
```

**Verification cues**

- `dim=self.default_ant_dof_positions.shape` 说明 `compute_middle_kernel` 不是在算一个值，而是在写满一整个 3D buffer。
- `dim=contacts.soft_contact_max` 说明 XPBD 那次 launch 的并行单位是 contact slot，不是 body。
- 读 Warp 代码时，`wp.tid()` 的含义永远要回 launch 现场去找。

**Checkpoint**

如果你现在读 kernel 还会下意识先盯函数体，不先找 `wp.launch(..., dim=...)`，就先停一下。能先回答“这一批线程到底在按谁跑”，才算真正跟上这一节。

**Output passed to next stage**

一个稳定读法：先看 launch 决定的 batch，再回头把 `wp.tid()` 翻成具体槽位。

### Stage 3: `wp.array` 再放回 Newton 的 `Model / State / Control`

**Definition**

`wp.array`：给 Warp kernel 稳定读写的批量缓冲区。第一遍先把它读成“这批 `x`、`out` 真正放在哪里”。

**Claim**

前两节里的 `x[i]`、`out[i]` 在真实 Newton 里通常不是零散临时变量，而是挂在 `Model`、`State`、`Control` 上的 `wp.array` 字段。

**Why it matters**

如果你不知道 buffer 归谁、会活多久，后面看到某个 launch 的输入输出，就不知道那是在读模型常量、当前状态，还是控制量。

**Plain loop story**

普通 loop 虽然只写了 `x` 和 `out`：

```python
for i in range(n):
    out[i] = f(x[i])
```

但真正读源码时，你一定会追问两件事：`x` 放在哪？`out` 写回谁？在 Newton 里，这两个问题经常要回到 `Model / State / Control` 才能答出来。

**Source excerpt**

`Model` 先把很多长期数据声明成 `wp.array` 字段：

```python
self.body_q: wp.array[wp.transform] | None = None
self.body_qd: wp.array[wp.spatial_vector] | None = None
self.body_com: wp.array[wp.vec3] | None = None

self.joint_q: wp.array[wp.float32] | None = None
self.joint_qd: wp.array[wp.float32] | None = None
self.joint_X_p: wp.array[wp.transform] | None = None
self.joint_X_c: wp.array[wp.transform] | None = None
```

进入 runtime 后，再由 `Model` 派生出 `State` 和 `Control`：

```python
s = State()
...
s.body_q = wp.clone(self.body_q, requires_grad=requires_grad)
s.body_qd = wp.clone(self.body_qd, requires_grad=requires_grad)
s.body_f = wp.zeros_like(self.body_qd, requires_grad=requires_grad)
...
s.joint_q = wp.clone(self.joint_q, requires_grad=requires_grad)
s.joint_qd = wp.clone(self.joint_qd, requires_grad=requires_grad)

c = Control()
...
c.joint_target_pos = wp.clone(self.joint_target_pos, requires_grad=requires_grad)
c.joint_target_vel = wp.clone(self.joint_target_vel, requires_grad=requires_grad)
c.joint_act = wp.clone(self.joint_act, requires_grad=requires_grad)
c.joint_f = wp.clone(self.joint_f, requires_grad=requires_grad)
```

**Verification cues**

- `body_q / body_qd / joint_q / joint_qd` 先挂在 `Model` 上，说明它们不是随手创建的局部列表，而是长期缓冲区。
- `Model.state()` 不是“开始求解”本身，而是在准备本帧要读写的状态 buffers。
- `Model.control()` 做的是同类事情，只不过对象换成控制输入。

**Checkpoint**

如果你现在还说不清“某个 launch 里的输入输出到底归 `Model`、`State` 还是 `Control`”，先别继续往下读。先回到这一节，把 loop 里的 `x`、`out` 分别对应到真实对象上。

**Output passed to next stage**

一组有稳定归属关系的批量 buffers。后面看到读写冲突时，你会知道冲突发生在哪个共享输出缓冲区上。

### Stage 4: `wp.atomic_add` 说明有共享写回，不说明新物理

**Definition**

`wp.atomic_add`：当很多个执行单元都可能更新同一个槽位时，用来做并行安全累加的操作。

**Claim**

看到 `wp.atomic_add` 或 `wp.atomic_sub` 时，第一反应应该是“这里有 many-to-one 共享写回”，而不是“这里出现了新的高深物理概念”。

**Why it matters**

这是 Warp 新手最常见的误读之一。atomic 真正解决的是并行写冲突，不是物理意义本身。

**Plain loop story**

同样从普通 loop 出发，只是这次目标槽位不再等于 `i`：

```python
for i in range(n):
    body_f[body_of(i)] += impulse(i)
```

这里很多不同的 `i` 都可能满足 `body_of(i) == 3`。也就是说，很多线程都可能想同时更新 `body_f[3]`。

**Conflict picture**

把它想成下面这个冲突就够了：

```text
thread 7  wants: body_f[3] += a
thread 12 wants: body_f[3] += b
```

如果两边都先读旧值、各自加上自己的量、再写回去，其中一份更新就可能被覆盖。`wp.atomic_add` 的作用，就是保证这类“往同一格累加”的写回不会丢失贡献。

**Source excerpt**

在 `mpm_twoway_coupling` 里，很多 collider impulse 都可能回写同一个 body：

```python
i = wp.tid()

cid = collider_ids[i]
...
body_index = body_ids[cid]
...
f_world = collider_impulses[i] / dt
...
wp.atomic_add(body_f, body_index, wp.spatial_vector(f_world, wp.cross(r, f_world)))
```

XPBD contact kernel 里也是同一件事：

```python
delta_total = (delta_f - delta_n) / denom * relaxation

wp.atomic_add(delta, particle_index, w1 * delta_total)

if body_index >= 0:
    delta_t = wp.cross(r, delta_total)
    wp.atomic_sub(body_delta, body_index, wp.spatial_vector(delta_total, delta_t))
```

**Verification cues**

- `body_index`、`particle_index` 都不是 `tid` 本身，所以很多线程完全可能命中同一格输出。
- atomic 周围总会有共享输出缓冲区，比如 `body_f`、`delta`、`body_delta`。
- 第一遍把 atomic 直接翻译成“并行安全累加”最稳。

**Checkpoint**

如果你看到 atomic 还会下意识问“这里是不是又多了一层 solver 数学”，先停一下。先回答“哪些不同的 `i` 可能在写同一个槽位”，通常就已经抓到它的核心了。

**Output passed to next stage**

一个稳定判断：看到原子操作，就先去找 many-to-one 的共享写回关系。

### Stage 5: `Graph` 和 `tile` 改的是执行组织方式

**Definition**

- `Graph`：把一串会重复执行的 Warp 工作流先 capture 下来，后面直接 replay。
- `tile`：在 kernel 内把一小块数据成组加载、复用、写回的执行组织方式。

**Claim**

`Graph` 和 `tile` 都建立在前面那条 loop -> kernel -> launch -> buffer 的主线上。它们没有改“每个元素怎么算”，改的是“这套执行怎样更有组织地提交和复用”。

**Why it matters**

新手很容易把这两类东西误读成“新 solver 概念”或者“新的物理层”。如果这样读，chapter 01 的主线就会散掉。

**Plain loop story**

先守住这一点：

```python
for i in range(n):
    out[i] = f(x[i])
```

到了 `Graph` 和 `tile` 这一层，`f(x[i])` 的物理含义并没有变。变的是：这串工作是否要被重复打包，以及局部数据是否要成块复用。

**Source excerpt**

`selection_materials` 里的 graph capture / replay：

```python
def capture(self):
    self.graph = None
    if wp.get_device().is_cuda:
        with wp.ScopedCapture() as capture:
            self.simulate()
        self.graph = capture.graph

def step(self):
    if self.graph:
        wp.capture_launch(self.graph)
    else:
        self.simulate()
```

`diffsim_bear` 里的 tiled kernel / tiled launch：

```python
@wp.kernel
def network(phases: wp.array2d[float], weights: wp.array2d[float], tet_activations: wp.array2d[float]):
    i = wp.tid()
    p = wp.tile_load(phases, shape=(PHASE_COUNT, 1))
    w = wp.tile_load(weights, shape=(TILE_TETS, PHASE_COUNT), offset=(i * TILE_TETS, 0))
    out = wp.tile_matmul(w, p)
    activations = wp.tile_map(tanh, out)
    wp.tile_store(tet_activations, activations, offset=(i * TILE_TETS, 0))

wp.launch_tiled(
    kernel=network,
    dim=self.padded_tet_count // TILE_TETS,
    ...,
)
```

**Verification cues**

- graph 例子里，被重复复用的仍然是同一段 `simulate()` 工作流。
- tiled 例子里，kernel 还是同一个 kernel，只是内部开始显式按 tile 组织局部数据。
- 两者都没有引入新的 Newton 对象边界，也没有改“单个元素规则”本身。

**Checkpoint**

如果你现在看到 `Graph` 或 `tile` 还会觉得“物理问题是不是换了”，就退回到开头那条普通 loop。只要 `f(x[i])` 没变，这里改变的就只是执行组织。

**Output passed to next stage**

一套完整的第一遍读法：先把源码翻回普通 loop，再分清 kernel、launch/tid、buffers、shared writeback，最后才看 graph 和 tile 这些执行组织层的东西。

## Object Ledger

| 对象 | 谁生产 | 谁消费 | 第一遍盯什么 |
|------|--------|--------|--------------|
| `kernel` | example / solver 源码里的 `@wp.kernel` 定义 | 对应 launch | 这一段是不是只在写“单个元素规则” |
| `wp.launch(..., dim=...)` | example / solver 调度层 | 对应 kernel | 这次 batch 到底按谁发起 |
| `wp.tid()` | Warp runtime | kernel 本体 | 当前这份执行对应哪个逻辑槽位 |
| `Model / State / Control` 里的 `wp.array` | `ModelBuilder.finalize()`、`Model.state()`、`Model.control()` | solver launches、example launches | 这批输入输出缓冲区归谁 |
| 共享输出缓冲区 | launch 前分配 | atomic 写回 | 哪些不同线程可能命中同一格 |
| `graph` | `wp.ScopedCapture()` | `wp.capture_launch()` | capture 的是不是一整段重复工作流 |
| `tile` | tiled kernel 本体 | `wp.launch_tiled()` | 是不是在做局部块级复用，而不是改物理含义 |

## Stop Here

读到这里就已经够 chapter 01 的 80-90% 了。

如果你现在能用自己的话讲顺下面这句话，这一章的 beginner 目标就完成了：

```text
Newton 里经常先有一段概念上的普通 loop；
kernel 定义单个 i 的规则；
wp.launch 和 wp.tid() 让整批执行真正发生；
这些输入输出通常挂在 Model / State / Control 的 wp.array 上；
atomic 处理 many-to-one 共享写回；
Graph 和 tile 只是执行组织方式。
```

如果你现在还答不上下面任一题，就先不要急着进入 `02_newton_arch`：

- Warp 在 Newton 里到底是在替代哪类普通 loop。
- 为什么读 kernel 前要先找 launch。
- 为什么 atomic 首先是在提醒你“共享写回”，而不是“新物理”。

这三件事能答顺，你就已经带着稳定读法进入下一章了。

## Go Deeper

如果你还想继续精确追源码，再去 `source-walkthrough-deep.md`：

- 想保留所有 file/symbol/line anchors：看 `Fast Deep Index`
- 想按真实文件顺序补完整证据链：看 `Exact Handoff Trace`
- 想知道哪些分支第一遍可以跳过：看 `Optional Branches`
- 想逐条核对这里的判断：看 `Verification Anchors`
