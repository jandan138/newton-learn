# Source Walkthrough Annotation Policy Design

- **日期**: 2026-04-21
- **仓库**: `/home/zhuzihou/dev/newton-learn`
- **范围**:
  - `chapters/*/source-walkthrough.md`
  - `templates/source-walkthrough.md.tmpl`
- **目标**: 为所有 mainline `source-walkthrough.md` 建立统一的源码摘录注释规则，让 first-pass 初学者在看代码块时更容易立即理解关键行的作用，同时避免把数学块和结构密集块注释得过于嘈杂。

---

## 1. 为什么要加这条仓库级规则

最近对 chapter 02 的重写说明了一件事：

```text
当代码摘录本身就承载了 chapter 的主故事线时，
中文行内注释能明显降低新手理解门槛。
```

尤其是下面这几类摘录：

- 例子入口到 runtime objects 的装配链
- `simulate()` / `step()` 这类控制流骨架
- 首次出现的 public API / helper 调用
- 跨对象 handoff 的关键代码段

如果这些块保持完全无注释，新手往往需要在代码块和下方 prose 之间来回跳很多次，才能把“哪一行在主线里负责什么”对上号。

但另一边也有明确风险：

- 数学推导块如果逐行加注，会快速变得嘈杂
- Warp kernel / Jacobian / MPM transfer 这类结构密集块，如果每行都注释，会遮住代码形状本身
- 字段表 / 声明表如果硬加注释，通常只会制造重复

所以这条规则不应是“所有摘录都逐行重写成中文解释稿”，而应是：

```text
默认提供中文行内注释，
但允许少量明确例外。
```

---

## 2. 适用范围

### 2.1 目标文件

这条规则适用于：

- 所有 `chapters/*/source-walkthrough.md`
- 对应的 `templates/source-walkthrough.md.tmpl`

不直接约束：

- `source-walkthrough-deep.md`
- `principle.md`
- `examples.md`

原因是 main walkthrough 的任务是 first-pass teaching，而 deep walkthrough 的任务是精确追锚点；两者不需要完全同一种摘录风格。

### 2.2 目标对象

目标对象是 walkthrough 中的 **源码摘录块**，不是所有 fenced code block。

也就是说，这条规则主要管：

- 从上游 Newton 源码摘出来的 Python / Warp 代码段
- 为教学而截短、重排过的源码 excerpt

不重点管：

- 纯 shell 命令
- 纯文本示意图
- 纯公式推导

---

## 3. 规则本体

### 3.1 默认规则

在 `source-walkthrough.md` 中，源码摘录默认提供中文行内注释。

默认意味着：

- 如果一个代码块直接承载本章主线
- 如果读者需要靠这段代码快速看出“这行在干什么”
- 如果摘录经过了教学裁剪，已经不再是完整原文件上下文

那么优先使用中文行内注释版摘录。

### 3.2 明确例外

下面这些类型允许不加行内注释，而改用块前 / 块后中文解释：

1. 数学推导块
   - 例如接触 row、Jacobian、Delassus、变分量、残差构造
2. 字段声明列表
   - 例如 buffer 字段表、dataclass / state namespace 字段目录
3. 很短且语义自明的摘录
   - 例如 2-4 行、函数名和变量名已经足够说明问题
4. 结构密集块
   - 例如 Warp kernel、Jacobian build、MPM transfer、矩阵装配

这里的关键不是“禁止注释”，而是：

```text
如果行内注释会明显破坏代码形状，
那就把解释移到块前或块后。
```

---

## 4. 注释应该怎么写

### 4.1 注释目标

行内注释的目标不是逐字翻译代码语法，而是解释：

```text
这行在章节主线里负责什么。
```

好的注释应优先回答：

- 这行是在创建什么对象
- 这行是在把什么交给什么
- 这行为什么出现在这里
- 这行和上一层概念有什么对应关系

### 4.2 不推荐的写法

不要把注释写成机械翻译：

```python
self.model = builder.finalize()  # 把 builder finalize 一下
```

这类注释几乎不提供额外信息。

也不要把一整段 prose 塞进行内注释，导致代码块本身失去可扫读性。

### 4.3 推荐的写法

更推荐这样的注释：

```python
self.model = builder.finalize()  # builder 冻结成可运行的 Model
self.state_0 = self.model.state()  # 当前状态缓冲区
self.state_1 = self.model.state()  # 下一状态缓冲区
```

这种注释不是在翻译语法，而是在补上 beginner 最需要的角色解释。

---

## 5. 块前提示规则

如果一个摘录块包含教学用中文行内注释，块前统一允许加一句提示：

```text
以下摘录为教学注释版，注释非原源码。
```

这句提示不是每块都强制要写，但在下面两种情况应优先加：

1. 代码块被明显截短、重排或省略了上下文
2. 行内注释较多，读者可能误以为注释来自上游原文件

---

## 6. 不同摘录类型的推荐策略

### 6.1 应优先加行内注释的摘录

- 例子入口与 launcher handoff
- builder / model / state / solver 装配链
- `simulate()` / `step()` 控制流骨架
- 首次出现的 helper 调用，如 `eval_fk(...)`
- chapter 主线里最关键的 5-15 行核心段落

### 6.2 应优先保持干净的摘录

- 数学结构主要靠跨行关系理解的块
- 大段字段表 / buffer 声明
- Warp kernel / FEM / Jacobian / MPM transfer 这类结构密集块
- 代码本身已经极短且语义清晰的摘录

这类代码块更适合：

- 块前先说明“接下来要看什么”
- 块后用 `Verification cues` / `Checkpoint` 解释关键关系

---

## 7. 对现有仓库风格的影响

这条规则不是要把整个 repo 改成“每段代码都写满中文旁白”，而是要把现在已经在 chapter 02 展现出来的成功做法推广出去。

也就是说，repo 的主风格会变成：

1. 先用 prose 建立本章问题和主线
2. 代码摘录默认带中文角色注释
3. 对不适合行内注释的摘录，保留干净代码，但块前 / 块后解释要更明确
4. `Verification cues` 继续保留，用来解释跨行关系和章节级意义

---

## 8. 成功标准

如果这条规则落地得对，读者在 main walkthrough 中应能获得下面这些体验：

1. 扫代码块本身，就能先看出关键行的作用。
2. 不需要频繁在代码块和 prose 之间来回跳，才能知道“这行在干什么”。
3. 数学块和结构密集块不会因为强行逐行注释而失去可读性。
4. 能明显感受到 repo 的 `source-walkthrough.md` 风格更加统一。

换句话说，这条规则追求的不是“注释越多越好”，而是：

```text
让 beginner 在第一遍读源码摘录时更容易建立主线，
同时不牺牲代码块本身的形状和可读性。
```
