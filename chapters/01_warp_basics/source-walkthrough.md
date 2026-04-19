---
chapter: 01
title: Warp 编程模型
last_updated: 2026-04-19
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

如果你是第一次读这一章，最好先配合 `principle.md` 一起读；如果你是直接跳进源码走读也没关系，但一旦发现术语开始变快，就先回到 `principle.md` 把对象关系补齐，再回来追源码。

这份主 walkthrough 只追 chapter 01 最值钱的一条线：在 Newton 里应该怎样读 Warp 代码。目标不是让你先背完整 Warp API，而是让你第一次看到 `wp.array`、`wp.launch`、`wp.tid()`、`wp.atomic_add`、`Graph`、`tile` 时，能立刻把它们放回一个稳定的执行模型里。

## What This Walkthrough Follows

这一页只追下面这条执行主线：

```text
long-lived buffers
-> launch site decides the batch
-> kernel body interprets wp.tid()
-> atomic ops handle shared writeback
-> graph / tile reorganize execution
```

这一页刻意不展开三类东西：

- Warp 全 API 手册
- XPBD、MPM、DiffSim 的完整物理细节
- 更重的 GPU 优化话题

第一遍先守住一句话：Warp 代码先读“谁在批量处理谁”，再读“这一批线程到底怎么算”。

## One-Screen Chapter Map

```text
Model / State / Control 里的 wp.array buffers
                    |
                    v
         wp.launch(..., dim=...)
                    |
                    v
      kernel 里的 wp.tid() 解释线程身份
                    |
                    v
    读写自己的槽位 or 原子累加共享槽位
                    |
                    v
   Graph / tile 改的是执行组织，不是物理含义
```

## Beginner Path

1. 先看 Stage 1。
   - 想验证什么：`wp.array` 在 Newton 里到底是临时变量，还是长期存在的数据底座。
   - 看完后应该能说：`Model / State / Control` 这些对象，本质上都在持有批量缓冲区。
2. 再看 Stage 2。
   - 想验证什么：为什么读 kernel 不能只盯函数体，必须先回到 launch site。
   - 看完后应该能说：`wp.tid()` 的意义是由 `dim` 决定的。
3. 再看 Stage 3。
   - 想验证什么：同样的读法怎样放进真实 solver，而不是只停留在玩具例子。
   - 看完后应该能说：XPBD 那次 launch 是按 contact slot 批量跑，不是按 body 跑。
4. 再看 Stage 4。
   - 想验证什么：`wp.atomic_add` 到底在解决什么问题。
   - 看完后应该能说：它是 many-to-one 写回时的安全累加，不是新的物理词。
5. 最后看 Stage 5。
   - 想验证什么：`Graph` 和 `tile` 为什么都属于“执行组织方式”。
   - 看完后应该能说：它们改变的是调度和局部复用，不是在改仿真语义。

## Main Walkthrough

### Stage 1: `wp.array` 先读成长期存在的批量缓冲区

**Claim**

在 Newton 里，`wp.array` 最先出现的位置通常不是零散的临时量，而是 `Model / State / Control` 背后的长期数据底座。

**Why it matters**

如果你一开始分不清 buffer 归谁、会活多久，后面看到 `wp.launch()` 就不知道输入输出到底在接哪一层对象。

**Source excerpt**

`Model` 先把静态和初始状态数据声明成大量 `wp.array` 字段：

```python
self.body_q: wp.array[wp.transform] | None = None
self.body_qd: wp.array[wp.spatial_vector] | None = None
self.body_com: wp.array[wp.vec3] | None = None

self.joint_q: wp.array[wp.float32] | None = None
self.joint_qd: wp.array[wp.float32] | None = None
self.joint_X_p: wp.array[wp.transform] | None = None
self.joint_X_c: wp.array[wp.transform] | None = None
```

等真正进入 runtime，再从 `Model` 里 clone 出 `State` 和 `Control`：

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

- `body_q / body_qd / joint_q / joint_qd` 先挂在 `Model` 上，说明它们是长期存在的模型与初始状态数据。
- `Model.state()` 不是“发起求解”，而是在复制初值并补 runtime force buffers。
- `Model.control()` 做的是同类事情，只不过对象换成控制输入。

**Output passed to next stage**

一组有稳定归属关系的批量 buffers。后面看到 launch 时，你就知道它们是在读 `Model`、`State` 还是 `Control`。

### Stage 2: 读 kernel 时先回 launch site，`dim` 决定线程身份

**Claim**

`wp.tid()` 不是自己带语义的；它到底表示 world、body、contact，还是别的索引，必须先回 launch site 看 `dim` 和输入 shape。

**Why it matters**

这是读 Warp 源码最重要的阅读顺序。先看 launch，你才知道这批线程在按什么东西批量跑；再看 kernel，本体才不会像黑箱。

**Source excerpt**

`selection_materials` 里有一个很干净的例子：

```python
@wp.kernel
def compute_middle_kernel(lower: wp.array3d[float], upper: wp.array3d[float], middle: wp.array3d[float]):
    world, arti, dof = wp.tid()
    middle[world, arti, dof] = 0.5 * (lower[world, arti, dof] + upper[world, arti, dof])
```

而决定这三个索引含义的，其实是 launch：

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

**Verification cues**

- `dim=self.default_ant_dof_positions.shape` 明确说明这是一个 3D batch。
- kernel 里把 `wp.tid()` 解成 `world, arti, dof`，本质上是在解释这 3 个 batch 维度。
- 这段代码不是“普通函数算一个中间值”，而是在给一整个 3D buffer 逐元素写值。

**Output passed to next stage**

一个稳定读法：先看 launch 的 `dim`，再回头解释 kernel 里的 `wp.tid()` 和数组索引。

### Stage 3: 真正的 solver 代码也要按同样顺序读

**Claim**

同样的 launch-first 读法，放进真实 solver 一样成立；`wp.launch()` 先告诉你“这一批线程是谁”，kernel 再把 `tid` 翻成具体的 contact、particle 或 body 槽位。

**Why it matters**

如果 chapter 01 只停留在轻量 example，上手真实 Newton 代码时还是会慌。真正关键的是把同一读法迁移到 solver 上。

**Source excerpt**

XPBD 先在 `step()` 里发起一次按 soft-contact slot 批量执行的 launch：

```python
wp.launch(
    kernel=solve_particle_shape_contacts,
    dim=contacts.soft_contact_max,
    inputs=[
        particle_q,
        particle_qd,
        ...
        contacts.soft_contact_particle,
        contacts.soft_contact_shape,
        contacts.soft_contact_body_pos,
        contacts.soft_contact_normal,
        contacts.soft_contact_max,
        dt,
    ],
    outputs=[particle_deltas, body_deltas],
)
```

真正的 kernel 一上来就把 `tid` 翻回 contact slot：

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

- `dim=contacts.soft_contact_max` 说明这次 launch 不是按 body 或 particle 数量发起的。
- kernel 里先做 `tid >= count` 检查，说明 `tid` 代表 contact 缓冲区的槽位。
- `shape_index`、`body_index`、`particle_index` 都是从这一格 contact 继续追出来的。

**Output passed to next stage**

一个清晰判断：这次 solver launch 的并行单位是 contact slot，后面的读写也要按这个粒度理解。

### Stage 4: `wp.atomic_add` 说明有共享写回，不说明新物理

**Claim**

`wp.atomic_add` 和 `wp.atomic_sub` 的第一层含义是“很多线程可能同时往同一个槽位写”，所以需要安全累加。

**Why it matters**

新手很容易把原子操作误读成“某种高级物理技巧”。其实它解决的是并行写回冲突，不是物理模型本身。

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

XPBD contact kernel 里也是一样：很多 contact 线程会回写同一个 particle 或 body：

```python
delta_total = (delta_f - delta_n) / denom * relaxation

wp.atomic_add(delta, particle_index, w1 * delta_total)

if body_index >= 0:
    delta_t = wp.cross(r, delta_total)
    wp.atomic_sub(body_delta, body_index, wp.spatial_vector(delta_total, delta_t))
```

**Verification cues**

- `body_index` 和 `particle_index` 都不是 `tid` 本身，所以很多线程可能命中同一格。
- 原子操作总是围绕共享输出缓冲区出现，比如 `body_f`、`delta`、`body_delta`。
- 你完全可以把它翻译成“并行安全累加”。第一遍不需要赋予它更多含义。

**Output passed to next stage**

一个稳定判断：看到原子操作，就先去找哪些线程在共享写回同一块缓冲区。

### Stage 5: `Graph` 和 `tile` 改的是执行组织方式

**Claim**

`Graph` 把重复 launch 的工作流 capture/replay；`tile` 把 kernel 内部的一段计算改成更显式的块级数据复用。两者都在改执行组织方式，不是在改 Newton 的物理语义。

**Why it matters**

如果把这两类东西读成“新 solver 概念”，chapter 01 的主线就会被带偏。它们应该被放在“执行组织”这一层。

**Source excerpt**

Graph capture/replay 的最小例子：

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

`tile` 的高级例子则长这样：

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

- Graph 例子里，capture 的仍然是同一段 `simulate()` 工作流，只是以后不再逐条重新提交。
- `launch_tiled()` 仍然是在跑 kernel，只不过 kernel 内部开始显式使用 tile-local 数据块。
- 这两段都没有引入新的仿真对象；它们只是换了执行方式。

**Output passed to next stage**

一套更完整的 Warp 阅读顺序：先认 buffer，再认 launch，再认 `tid`，最后才把原子操作、graph、tile 放回执行组织层。

## Object Ledger

| 对象 | 谁生产 | 谁消费 | 盯哪些字段 |
|------|--------|--------|------------|
| `Model` 里的 `wp.array` | `ModelBuilder.finalize()` 和 `Model` 初始化 | `Model.state()`、`Model.control()`、solver launches | `body_q / body_qd / joint_q / joint_qd` 这类长期缓冲区 |
| `State` | `Model.state()` | `clear_forces()`、`model.collide()`、`solver.step()` | `body_q`、`body_qd`、`body_f` |
| `Control` | `Model.control()` | `solver.step()` | `joint_target_pos`、`joint_target_vel`、`joint_act`、`joint_f` |
| `wp.launch(..., dim=...)` | example / solver 调度层 | 对应 kernel | `dim` 到底按什么对象批量跑 |
| `wp.tid()` | Warp runtime | kernel 本体 | 这一格线程对应哪个 logical slot |
| 共享输出缓冲区 | kernel launch 前分配 | 原子操作写回 | `body_f`、`delta`、`body_delta` |
| `graph` | `wp.ScopedCapture()` | `wp.capture_launch()` | capture 的是不是一整段重复工作流 |

## Stop Here

读到这里就已经够 chapter 01 的 80-90% 了。

如果你现在能用自己的话讲顺下面这句话，这一章的 beginner 目标就完成了：

```text
Newton 里的 wp.array 往往是长期缓冲区；
wp.launch 先决定这批线程在按谁批量跑；
kernel 再用 wp.tid() 解释每个线程的槽位；
atomic 处理共享写回；
Graph 和 tile 只是执行组织方式。
```

这时你已经可以带着稳定读法进入 `02_newton_arch`，不会再把 Warp 代码当成一整片黑箱。

## Go Deeper

如果你还想继续精确追源码，再去 `source-walkthrough-deep.md`：

- 想保留所有 file/symbol/line anchors：看 `Fast Deep Index`
- 想按真实文件顺序补完整证据链：看 `Exact Handoff Trace`
- 想知道哪些分支第一遍可以跳过：看 `Optional Branches`
- 想逐条核对这里的判断：看 `Verification Anchors`
