# Walkthrough Redesign Design

- **日期**: 2026-04-19
- **仓库**: `/home/zhuzihou/dev/newton-learn`
- **范围**: 全仓 `chapters/**/source-walkthrough*.md` 规范重构
- **目标**: 把当前偏“跨仓 + 行号索引器”的源码走读，升级成一套对新手更友好的双层 walkthrough 体系：新手只读主 walkthrough 也能掌握 80-90%，而深读者仍然保有精确源码锚点。

---

## 1. 背景

当前仓库已有 8 个 `source-walkthrough.md`：

- `chapters/01_warp_basics/source-walkthrough.md`
- `chapters/02_newton_arch/source-walkthrough.md`
- `chapters/03_math_geometry/source-walkthrough.md`
- `chapters/04_scene_usd/source-walkthrough.md`
- `chapters/05_rigid_articulation/source-walkthrough.md`
- `chapters/06_collision/source-walkthrough.md`
- `chapters/07_constraints_contacts_math/source-walkthrough.md`
- `chapters/08_rigid_solvers/source-walkthrough.md`

另有正在推进中的 `09_variational_solvers/source-walkthrough.md`。

这些文件的优点是：

1. 源码锚点很强
2. 对认真深挖的读者很有价值
3. 能把章节主线牢牢钉回上游 Newton 源码

但当前用户反馈也很明确：

- **跨仓 + 行号锚点的阅读路径不友好，容易劝退**

而且这不是 `07/08/09` 的局部问题，而是**整个仓库的 walkthrough 写法问题**。

---

## 2. 现在的问题到底是什么

当前 walkthrough 的主要问题不是单点缺陷，而是几种摩擦叠加：

1. **跨仓切换太频繁**
   - 读者必须同时停留在 `newton-learn` 和上游 `newton` 两个仓库之间
2. **行号锚点太脆**
   - 即使 `newton_commit` 固定，行号仍然天然更像精确索引，而不是教学入口
3. **一次给太多文件和太多路径**
   - 初次阅读时，很难知道先看哪一个、看到哪里可以先停
4. **当前文件更像“索引器”而不是“带你读源码的人”**
   - 它知道证据在哪，但没有足够低阻力地把读者带到 80-90% 的理解

所以这次 redesign 的核心，不是“删掉源码锚点”，而是：

```text
把 walkthrough 的中心从“你去这些文件这些行号看”
改成“你先理解这个 handoff，源码是证据而不是门槛”。
```

---

## 3. 用户要求转成设计约束

用户的要求可以归纳成四条刚性约束：

1. **新手只看给新手看的部分，也要能理解 80-90%**
2. **给新手看的部分要尽可能详细、通俗易懂**
3. **给新手看的部分也要带源码**
4. **新手部分尽量不需要跳去另一个仓库**

这四条约束意味着：

- 单文件内“塞一个轻量 beginner 区块”还不够
- beginner 读物必须真正自包含
- 但深读用户仍然需要保留精确源码锚点

也就是说，repo-wide redesign 必须同时满足：

- **方案 2 的统一规范**
- **方案 3 的双层阅读体验**

---

## 4. 核心设计决策

### 4.1 采用 repo-wide 的两层 walkthrough 体系

每章今后统一采用两个 walkthrough 文件：

1. `source-walkthrough.md`
   - **新手主读版**
   - 目标：单看这一份，就拿到 80-90% 的核心理解
   - 必须自包含、带关键源码片段、尽量不要求跨仓跳转

2. `source-walkthrough-deep.md`
   - **深读锚点版**
   - 目标：给已经进入源码的人提供精确锚点、跨文件路径和更完整证据链

这不是“多写一份附录”，而是把两类阅读任务正式拆开：

- beginner 任务：理解主线
- deep 任务：验证和深挖

### 4.2 `source-walkthrough.md` 从“索引器”升级成“带读稿”

未来的主 walkthrough 不再以大量 `path:Lx-Ly` 为主体，而要以：

1. 一条 chapter map
2. 少量 handoff stages
3. 内嵌源码片段
4. 每段结束后的“你现在应该看懂什么”

来组织内容。

### 4.3 `source-walkthrough-deep.md` 专门承接跨仓、行号和可选分支

深读版保留这些东西：

- `repo@commit + file + symbol`
- 必要时的行号
- 跨文件追踪
- 可选分支
- 更细粒度的数据结构 / kernel 锚点

这样可以把新手体验和证据精度同时保住，而不再彼此干扰。

---

## 5. 文件职责重构

以后每个 chapter 的标准文件结构变成：

- `README.md`
- `principle.md`
- `source-walkthrough.md`
- `source-walkthrough-deep.md`
- `examples.md`

它们各自的职责如下。

### 5.1 `README.md`

职责：

- 说明本章解决什么问题
- 给出完成门槛
- 明确阅读顺序
- 明确：
  - **第一次读，优先 `source-walkthrough.md`**
  - **想精确追源码，再去 `source-walkthrough-deep.md`**

### 5.2 `principle.md`

职责：

- 讲概念主线
- 负责人话解释、误解拆除、对象关系
- 不承担完整源码导航职责

### 5.3 `source-walkthrough.md`

职责：

- 给新手看的主 walkthrough
- 单看这一份，应该就能掌握 80-90% 的章节源码主线
- 使用少量关键源码片段，而不是要求读者自行在另一个仓库里查找大量行号

### 5.4 `source-walkthrough-deep.md`

职责：

- 给需要进一步验证、追锚点、深挖支线的读者
- 保留精确源码锚点、跨文件跳转、深层数据路径

### 5.5 `examples.md`

职责：

- 把 principle 和 walkthrough 变成可观察现象
- 帮读者在“不继续深挖源码”的情况下完成自检

---

## 6. 新手主读版 walkthrough 模板

以后每章的 `source-walkthrough.md` 统一用下面结构：

### 6.1 `What This Walkthrough Follows`

- 只说这页追哪条线
- 明确不追什么

### 6.2 `One-Screen Chapter Map`

- 用一张一屏就能看懂的流程图或 pipeline
- 例子：

```text
body/world state
-> candidate pairs
-> ContactData
-> Contacts
```

### 6.3 `Beginner Path`

- 给 3-5 站最小路径
- 每站说明：
  - 先看哪段内嵌源码
  - 想验证什么
  - 看完后应该能说出什么

### 6.4 `Main Walkthrough`

- 按 3-6 个 handoff stages 组织
- 每个 stage 固定五项：
  - Claim
  - Why it matters
  - Source excerpt
  - Verification cues
  - Output passed to next stage

### 6.5 `Object Ledger`

- 统一表格：
  - `对象 | 谁生产 | 谁消费 | 盯哪些字段`

### 6.6 `Stop Here`

- 明确告诉读者：
  - 读到这里已经够 80-90%
  - 如果还想更深，再去 deep 版

### 6.7 `Go Deeper`

- 指向 `source-walkthrough-deep.md` 对应段落

---

## 7. 深读锚点版模板

以后每章的 `source-walkthrough-deep.md` 统一用下面结构：

### 7.1 `Fast Deep Index`

- 一张简短索引表：
  - `ID | repo@commit path | symbol | why it matters`

### 7.2 `Exact Handoff Trace`

- 按数据流或控制流分段
- 允许跨文件
- 允许更多精确锚点

### 7.3 `Optional Branches`

- 说明哪些路径是 first pass 可跳过的

### 7.4 `Verification Anchors`

- 给深读者验证 claim 的精确入口

---

## 8. 引用规则重构

### 8.1 新手版的主引用单位

新手版正文里，源码锚点优先级改成：

1. `repo@commit + file + symbol`
2. verification cue
3. 行号只做补充，不做主叙事单位

也就是以后正文尽量写成：

- 去 `solver_xpbd.py` 的 `step()`
- 先盯 `particle_deltas`
- 再盯 `solve_springs(...)`

而不是大段写成：

- `solver_xpbd.py:L245-L309`

### 8.2 新手版必须内嵌源码片段

规则：

- 每个关键 stage 至少有一个内嵌源码片段
- 片段长度建议 `8-30` 行；必要时可到 `40` 行
- 片段要配人话解释，不可只贴代码
- 片段要足以让读者理解 claim，而不是只当装饰

### 8.3 deep 版才是行号密集区

`source-walkthrough-deep.md` 可以继续使用：

- `path:Lx-Ly`
- 多路径对照
- 可选深挖分支

但仍建议优先写：

- file + symbol
- 行号作为补充定位

---

## 9. 维护性要求

因为新手版会包含源码片段，所以必须正面处理维护成本。

### 9.1 所有片段都绑定 `newton_commit`

- 每章 frontmatter 已经有 `newton_commit`
- 新手版和 deep 版都必须明确使用同一个 commit

### 9.2 源码片段不要追求“全量”

- 只嵌入关键 20% 片段
- 目标是支撑 80-90% 理解，不是替代整个上游源码树

### 9.3 行号漂移问题集中在 deep 版承接

- 新手版减少行号依赖，就能显著降低“行号一变就劝退”的风险
- deep 版允许更脆，但它本来就是给愿意继续追的人看的

---

## 10. 迁移策略

这次 redesign 不应一次性全仓同时重写，而应分阶段推进。

### Phase 1: 先写仓库级规范

- 也就是本设计文档

### Phase 2: pilot 一章

推荐 pilot：`chapters/06_collision`

原因：

1. pipeline 最清楚
2. 跨文件最典型
3. 也是最容易被“跨仓 + 行号”劝退的一章
4. 改好以后，`07/08/09` 最容易直接继承模板

### Phase 3: 先推广到近主线章节

优先顺序：

1. `06_collision`
2. `07_constraints_contacts_math`
3. `08_rigid_solvers`
4. `09_variational_solvers`

### Phase 4: 再回扫 `01-05`

- 用同一规范逐章收敛

---

## 11. 成功标准

如果 redesign 成功，应该出现下面这些变化：

1. 新手第一次读 `source-walkthrough.md`，不需要频繁跳到上游 repo 才能跟上主线。
2. 新手单读主 walkthrough，也能掌握 80-90% 的章节源码主线。
3. deep reader 仍然可以通过 `source-walkthrough-deep.md` 精确追到源码锚点。
4. walkthrough 从“高精度索引器”升级成“带你读源码的人”。

---

## 12. 非目标

这次 redesign 不做下面这些事：

1. 不删除 deep 源码锚点能力
2. 不要求把所有源码都完整嵌入仓库文档
3. 不在这一步引入自动抽取源码片段脚本
4. 不一次性重写全仓所有章节

---

## 13. 推荐的下一步

最自然的下一步是：

1. 基于本规范写实现计划
2. 先 pilot 重做 `chapters/06_collision/source-walkthrough.md`
3. 新增 `chapters/06_collision/source-walkthrough-deep.md`
4. 验证效果后，再把同一模式推广到 `07/08/09`
