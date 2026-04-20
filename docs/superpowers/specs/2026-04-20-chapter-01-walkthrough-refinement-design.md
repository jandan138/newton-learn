# Chapter 01 Walkthrough Refinement Design

- **日期**: 2026-04-20
- **仓库**: `/home/zhuzihou/dev/newton-learn`
- **范围**: `chapters/01_warp_basics/source-walkthrough.md`
- **目标**: 把 chapter 01 的主 walkthrough 从“已经知道 Warp 一点点的人可以顺着读”提升成“真正的新手单读也能把来龙去脉讲顺”的版本。

---

## 1. 背景

chapter 01 已经完成过一次 walkthrough redesign，结构上已经对齐了仓库新规范：

- `What This Walkthrough Follows`
- `One-Screen Chapter Map`
- `Beginner Path`
- `Main Walkthrough`
- `Object Ledger`
- `Stop Here`
- `Go Deeper`

但用户的具体反馈说明：

- 现在这版 `source-walkthrough.md` 虽然更像“新手主读版”，但仍然**离真正的新手有点距离**
- 尤其是“来龙去脉”没有完全讲透

换句话说，这不是 walkthrough 模式错了，而是 chapter 01 这份主 walkthrough **还没有把 Warp 的前置动机、执行图景和概念依赖顺序铺够**。

---

## 2. 当前版本的具体问题

综合用户反馈和双 reviewer 结论，当前文件的主要问题有五类。

### 2.1 没先交代“为什么会有 Warp”

现在文件一上来就进入：

- `wp.array`
- `wp.launch`
- `wp.tid()`
- `atomic`

但没有先解释：

- 为什么 Newton 这种仿真系统会需要 Warp
- Warp 到底是在替代什么样的普通写法
- GPU 批量执行解决的是哪类问题

这会导致新手读到概念时，知道“它存在”，但不知道“它为什么存在”。

### 2.2 概念出现顺序仍然偏“源码导向”，不够“学习顺序导向”

当前结构虽然已经有 stages，但一开始还是比较快地落到：

- `Model / State / Control`
- `wp.array`
- `wp.tid()`

这对第一次接触 Warp 的读者来说太快。

更自然的学习顺序应该是：

1. 先看一个普通 Python `for` 循环在干什么
2. 再看为什么这类循环在仿真里要批量并行化
3. 再看 Warp 如何把“单元素规则”和“整批 dispatch”拆开
4. 最后才把这套执行图放回 Newton 的 `Model / State / Control`

### 2.3 没有一个贯穿全文的“单一故事”

当前每个 stage 都有例子，但例子来源较散：

- `Model.state/control`
- `selection_materials`
- `XPBD`
- `mpm_twoway_coupling`
- `diffsim_bear`

这些例子各自都成立，但新手会感觉像：

- 每节都在看一个新东西
- 不是在顺着同一条“故事线”往前走

需要增加一条贯穿全文的统一叙事骨架。

### 2.4 关键术语首次出现时，缺少“定义框”

比如：

- `kernel`
- `wp.tid()`
- `atomic`
- `graph capture`
- `tile`

现在虽然都解释了，但还不够像真正的新手教材。对于新手，第一次见到这些词时，更适合：

- 先给一句非常短的定义
- 再给它解决的问题
- 然后再看代码

### 2.5 需要更多“理解检查点”

当前有 `Stop Here` 总结，但在中间 stages 之间，新手还缺少：

- 我现在是不是已经跟上了？
- 如果没跟上，我应该回退到哪个概念？

所以 chapter 01 主 walkthrough 需要加更明确的 checkpoint 语气。

---

## 3. 设计目标

这次 refinement 的目标不是重写整个 chapter 01，而是把 `source-walkthrough.md` 拉到下面这个标准：

1. 新手单读这份 walkthrough，也能讲清 chapter 01 的 80-90% 主线
2. 读者能回答“为什么会有 Warp、为什么不是普通 `for` 循环”
3. 每个 Warp 关键词第一次出现时，都能被翻译成人话
4. 整个文件有一条贯穿的“普通 loop -> Warp -> Newton”故事线
5. 仍然保留现有 rollout 后的 main/deep 分工，不把 deep 锚点重新搬回主文件

---

## 4. 备选方案

### 方案 A：只增加更多解释段

- 做法：保留现有 stage 顺序，只在每个 stage 前后补更多人话
- 优点：改动小
- 缺点：结构问题仍在，新手还是会感觉“例子切得太快”

### 方案 B：加一个贯穿全文的“普通 Python loop -> Warp”主故事线

- 做法：在保留现有 stages 的同时，加一条统一的对照故事线
- 优点：来龙去脉更清楚
- 缺点：如果不调整 stage 次序，仍然会有一部分“先看到术语，后理解术语”的问题

### 方案 C：按“新手学习顺序”重排 stage，并加贯穿故事线

- 做法：
  1. 先讲为什么会有 Warp
  2. 先从普通 loop 到 kernel/launch/tid 的转换开始
  3. 再讲 `wp.array` 和 Newton 的对象缓冲区
  4. 再讲 `atomic`
  5. 最后才讲 `Graph` / `tile`
- 优点：最符合新手理解顺序
- 缺点：改动最大

### 推荐：方案 C

用户明确要求“把来龙去脉讲清楚”，所以只补字不够，应该按学习顺序重排。

---

## 5. 推荐重写策略

### 5.1 增加一个新的开场段：Why Warp in Newton?

在文件最前面补一段非常朴素的动机：

- 仿真一步里有很多“同样规则作用在很多对象上”的更新
- 普通 Python `for` 循环可以表达这件事
- Warp 则把这类更新拆成：
  - 单元素规则（kernel）
  - 整批发起（launch）

这段是全文件最重要的“来龙去脉入口”。

### 5.2 用一个贯穿全文的普通 loop 对照故事线

建议增加一条统一的对照：

```python
for i in range(n):
    out[i] = f(x[i])
```

然后告诉读者：

- kernel = “单个 `i` 该怎么做”
- launch = “这次一共发多少个 `i`”
- `wp.tid()` = “我现在是哪一个 `i`”
- atomic = “如果很多个 `i` 要往同一个地方写怎么办”

这样每一节都能回到同一条主故事线，不会变成跳例子。

### 5.3 重排主 stage 顺序

建议把主线顺序改成：

1. **Why Warp / 从普通 loop 到 kernel**
2. **`wp.launch` 和 `wp.tid()` 是怎样让“整批执行”真正发生的**
3. **`wp.array` 再放回 Newton 的 `Model / State / Control`**
4. **`wp.atomic_add` 解决什么共享写回问题**
5. **`Graph` / `tile` 作为执行组织优化层**

其中：

- 现在的 `Model / State / Control` 内容要后移
- 先让读者会读 Warp，再把 Warp 放回 Newton

### 5.4 给每个关键术语加定义框

要求：

- `kernel`
- `wp.launch`
- `wp.tid()`
- `wp.array`
- `wp.atomic_add`
- `Graph`
- `tile`

第一次出现时，都要有一句 definition-style 的解释。

### 5.5 在每个 stage 结束时加 checkpoint 语气

不是新增大量结构，而是在每个 stage 的结尾用一句：

- 如果你现在能回答 X，就说明跟上了
- 如果答不上来，先回退到上一节

---

## 6. 文件范围

本次 refinement 只改：

- `chapters/01_warp_basics/source-walkthrough.md`

不改：

- `principle.md`
- `source-walkthrough-deep.md`
- 其它章节

除非在实现过程中发现 README 的阅读顺序提示和新结构明显冲突，才做最小修正。

---

## 7. 成功标准

如果 refinement 做对了，读者第一次读完 chapter 01 主 walkthrough 后，应该更容易回答：

1. Warp 在 Newton 里到底解决了什么类问题？
2. 为什么 kernel 不是“更神秘的函数”，而是“单元素规则”？
3. 为什么读源码时要先看 launch，再看 `wp.tid()`？
4. 为什么 `wp.array` 在 Newton 里经常挂在 `Model / State / Control` 上？
5. 为什么 `atomic`、`Graph`、`tile` 都应该被先放回“执行组织层”理解？

如果用户再去看这份文件，不会再说“还是离新手查的有点距离”，说明这次 refinement 达标。
