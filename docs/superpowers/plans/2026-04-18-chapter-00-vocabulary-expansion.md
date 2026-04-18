# Chapter 00 Vocabulary Expansion Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Expand the rigid-body and robot-dynamics vocabulary section in chapter `00` from a compact glossary table into a small prerequisite lesson that better prepares readers for chapters `01` and `02`.

**Architecture:** Keep the change tightly scoped to `chapters/00_prerequisites/principle.md`. Replace the current table-heavy vocabulary block with a layered structure: an orienting paragraph, a short chain-level explanation, six expanded term blocks, a relationship summary, and a compressed recap table. Preserve the current chapter tone and keep all detailed mathematics deferred to later chapters.

**Tech Stack:** Markdown, current chapter-00 writing style, repository conventions, existing glossary terminology.

---

### Task 1: Rewrite the chapter-00 vocabulary section into a small prerequisite lesson

**Files:**
- Modify: `chapters/00_prerequisites/principle.md`
- Reference: `conventions/glossary.md`
- Reference: `chapters/02_newton_arch/principle.md`
- Reference: `docs/superpowers/specs/2026-04-18-chapter-00-vocabulary-expansion-design.md`

- [ ] **Step 1: Locate the exact section to replace**

Identify the current block starting at:

```md
## 2. 刚体与机器人动力学最小词汇
```

and ending just before:

```md
## 3. GPU / Warp 最小直觉
```

Only this section should be substantially rewritten.

- [ ] **Step 2: Replace the opening of section 2 with a stronger orienting setup**

Write 2-3 short paragraphs that establish:

```md
- 这些词不是散点词表，而是一条刚体阅读链的最小骨架
- 你现在不需要会推导，只需要知道每个词在链里回答什么问题
- 这节服务的是 `02` 的架构阅读，以及 `05` / `07` 的第一次深入
```

The tone should stay consistent with the rest of `00`: reassuring, plain-language, and navigation-oriented.

- [ ] **Step 3: Add one short chain-level paragraph before the term-by-term blocks**

Write one compact paragraph that introduces this sequence explicitly:

```md
`articulation` -> generalized coordinates -> `M` / `J` -> ABA / CRBA
```

It must explain, in words, why a multi-rigid-body system naturally leads from “what system am I describing?” to “what coordinates describe it?” to “how is inertia organized?” to “how do different algorithm names enter?”.

- [ ] **Step 4: Expand `articulation` into a short teaching block**

Write a 2-4 paragraph block that covers:

```md
- 它在解决什么问题：单个刚体 vs 关节连接的多刚体系统
- 它和后续词的关系：是广义坐标、`M`、`J` 讨论的系统容器
- 你在后面第一次怎么碰到它：`05_rigid_articulation`
- 现在先懂到什么程度就够
```

Avoid formulae. Keep the emphasis on system-level understanding.

- [ ] **Step 5: Expand generalized coordinates into a short teaching block**

Write a 2-4 paragraph block that covers:

```md
- 为什么不能只盯世界坐标
- 为什么需要更省变量的系统描述方式
- 它为什么是 `M` 和 `J` 的共同语言
- 现在先知道“它描述系统姿态/自由度”就够
```

- [ ] **Step 6: Expand mass matrix `M` into a short teaching block**

Write a 2-4 paragraph block that covers:

```md
- 它在解决什么问题：系统惯性如何整体组织
- 直觉重点：哪些方向难推、哪些自由度互相耦合
- 它和 generalized coordinates 的关系
- 先不要进入矩阵推导
```

- [ ] **Step 7: Expand Jacobian `J` into a short teaching block**

Write a 2-4 paragraph block that covers:

```md
- 它在解决什么问题：关节空间与任务/接触空间之间的映射
- 为什么末端、接触点、约束经常和它一起出现
- 它和 `M` 不同：`M` 讲惯性组织，`J` 讲速度/力的映射桥梁
- 先不要进入完整约束数学
```

- [ ] **Step 8: Expand ABA and CRBA into distinct short teaching blocks**

Write separate short blocks for both terms:

```md
ABA:
- 面向正向动力学的高效算法名字
- 属于 Featherstone 路线关键词
- 主要在 `05_rigid_articulation` 真正展开

CRBA:
- 面向质量矩阵构造的高效算法名字
- 和 `M` 的关系最重要
- 常和 ABA 一起出现，但回答的问题不同
```

Do not collapse them into one sentence. The point is to let the reader feel the distinction.

- [ ] **Step 9: Add a relationship-summary paragraph after the term blocks**

Write a short summary paragraph that restitches the whole chain:

```md
- 先有系统对象 (`articulation`)
- 再有系统坐标 (generalized coordinates)
- 再有全局惯性和映射 (`M` / `J`)
- 最后才看到不同算法名字 (ABA / CRBA)
```

The wording should help readers realize why these six terms appear together.

- [ ] **Step 10: Reintroduce a compressed recap table at the end of section 2**

Add a compact table with these columns:

```md
| 词 | 它主要回答什么问题 | 在这条链里扮演什么角色 | 第一次深入章节 |
```

The table should summarize the expanded prose, not replace it.

---

### Task 2: Tighten scope, align wording, and verify the expanded section still fits chapter 00

**Files:**
- Modify: `chapters/00_prerequisites/principle.md`
- Reference: `chapters/01_warp_basics/principle.md`
- Reference: `chapters/02_newton_arch/principle.md`

- [ ] **Step 1: Check the expanded section against the non-goals**

Read the rewritten section and cut any paragraph that drifts into:

```md
- 正式刚体动力学推导
- 质量矩阵具体装配细节
- Jacobian 公式细节
- Featherstone 算法步骤细节
- 约束 / 接触数学细节
```

Expected result: the section feels richer, but still clearly belongs to chapter `00` rather than `05` or `07`.

- [ ] **Step 2: Align terminology with the rest of the first three chapters**

Make sure these names and roles stay consistent with `01` and `02`:

```md
- `articulation`
- generalized coordinates
- mass matrix `M`
- Jacobian `J`
- `Model / State / Control / Solver`
```

If wording drifts, tighten it so the reader does not feel like chapter `00` uses a different language than chapters `01` and `02`.

- [ ] **Step 3: Keep the section readable as a first pass**

Review for density and convert any overpacked paragraph into shorter blocks. Prefer this pattern:

```md
- what problem this term solves
- how it connects to the next term
- how far to understand it now
```

Expected result: a reader can scan the section quickly, then reread it more slowly if needed.

- [ ] **Step 4: Verify the section still ends by sending the reader forward**

Ensure the rewritten section still points readers toward the next real destinations:

```md
- `02_newton_arch` for architecture-level contact with these words
- `05_rigid_articulation` for rigid-body deepening
- `07_constraints_contacts_math` where `J` and related math become more serious
```

These can be in the prose or the recap table, but the navigation must remain explicit.

- [ ] **Step 5: Run targeted verification commands**

Run these commands after editing:

```bash
git diff --check -- "chapters/00_prerequisites/principle.md"
rg -n "本章要回答什么问题？|读完后应该能独立讲清哪些对象、流程和边界？|本章最终应有哪些文件：" "chapters/00_prerequisites/principle.md"
```

Expected result:

```text
- `git diff --check` prints nothing
- `rg` finds no leftover template language in the file
```

- [ ] **Step 6: Write a short handoff summary for reviewers**

Summarize the final state in 3 bullets:

```md
- what got expanded in section 2
- how the section is now more than a glossary but still not a full course
- what remains intentionally deferred to `05` and `07`
```
