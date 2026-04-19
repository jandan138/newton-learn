---
chapter: 09
title: 变分求解器族 例子观察单
last_updated: 2026-04-19
source_paths:
  - newton/examples/cloth/example_cloth_hanging.py
  - newton/examples/softbody/example_softbody_hanging.py
  - newton/_src/solvers/xpbd/solver_xpbd.py
  - newton/_src/solvers/vbd/solver_vbd.py
  - newton/_src/solvers/style3d/solver_style3d.py
paper_keys:
  - xpbd-paper
  - vbd-paper
  - style3d-paper
newton_commit: 1a230702
---

# 09 变分求解器族 例子观察单

`principle.md` 已经把本章主线讲清了: chapter 09 不是 catalog，而是同一个 implicit correction problem 怎样被三种更新层级处理。这里不再加新理论，只把这件事变成可观察现象。

下面提到的源码行号都以 `newton_commit: 1a230702` 为准。

这页故意只留两个 anchor:

- `newton/examples/cloth/example_cloth_hanging.py`: 本章主例子，负责跨 solver 对照。
- `newton/examples/softbody/example_softbody_hanging.py`: 本章辅例子，负责说明 VBD 不只做 cloth。

它们分工不同，最好不要拿一个例子同时承担两件教学任务。

## 主例子: `newton/examples/cloth/example_cloth_hanging.py`

建议入口:

```bash
python -m newton.examples cloth_hanging --solver xpbd
python -m newton.examples cloth_hanging --solver vbd
python -m newton.examples cloth_hanging --solver style3d
python -m newton.examples cloth_hanging --solver semi_implicit
```

这个例子最值钱的地方，不是它场景有多复杂，而是它把“同一张 hanging cloth 图，可切不同 solver 路线”这件事写得非常直白。

先盯五处代码就够了:

- `L40-L45`: 四种 solver 的 `sim_substeps` 不是一样的。
- `L52-L57`: `Style3D` 需要先注册 custom attributes。
- `L82-L109`: solver-specific cloth 参数在这里分叉。
- `L111-L117`: `Style3D` 和 `VBD` 都对 builder 有额外要求。
- `L165-L175`: 真正的 simulation loop 仍然保持一致。

### 它最适合验证什么

- 同一个 `cloth_hanging` scene 背后，三家面对的是同一类稳定更新问题。
- `XPBD / VBD / Style3D` 的第一层差别不是“谁支持 cloth”，而是“每步修正先落在哪层”。
- `SemiImplicit` 放在这里很有价值，但它的角色只是 baseline 对照，不是 chapter 09 的 main trio。

### 改哪里最值钱

| 改哪里 | 先看什么 | 这能帮你理解什么 |
|--------|----------|------------------|
| 在 `--solver` 间切换 | 外层 `simulate()` 主循环几乎不变 | shared example 和 shared `step(...)` surface 是真的共享的 |
| 盯 `L40-L45` 的 `sim_substeps` | `SemiImplicit`、`Style3D`、`XPBD / VBD` 对 timestep 组织方式不同 | 同一 scene 下，solver 对“这一步怎么稳”会有不同工程假设 |
| 盯 `L52-L57`、`L111-L113` | `Style3D` 不是直接走通用 cloth builder | `Style3D` 真的是 cloth-specialized 路线 |
| 盯 `L116-L117` | 只有 `VBD` 需要 `builder.color(include_bending=True)` | `VBD` 的局部 block / color update 不是装饰，而是求解骨架的一部分 |
| 盯 `L124-L144` | 四个 solver 构造器并列出现 | `SemiImplicit` 只是对照，chapter 09 真正比较的是后面三家 |

### 第一遍最值得观察什么

- 如果你只看 scene 图像，三家好像都在做“布往下垂”。
- 如果你看脚本 setup，三家已经开始暴露不同求解哲学。
- 如果你再回源码，会发现这些 setup 差别正对应 `constraint -> block -> global solve` 这条梯子。

### 如果结果和预期不一样, 你大概率误解了什么

- 如果你把 `SemiImplicit` 也当成 chapter 09 的主角，你会把整章重新写回“solver 名字目录”。
- 如果你觉得 `XPBD` 和 `VBD` 都只是“粒子 solver”，你就漏掉了局部子问题已经换了。
- 如果你觉得 `Style3D` 只是 cloth 参数更多的版本，你就没有注意到 custom attributes、`_precompute(builder)` 和 PCG 这条主线。

## 对照例子: `newton/examples/softbody/example_softbody_hanging.py`

这个例子承担的是另一件事: **给 `VBD` 补一条 beyond-cloth 的延伸线。**

建议入口:

```bash
python -m newton.examples softbody_hanging
```

它不是三家横向比较例子，所以不需要拿它去问“为什么不用 `XPBD / Style3D`”。脚本本身已经很坦白:

- `L32-L33`: 只支持 `vbd`。
- `L48-L65`: 场景是四组不同阻尼的 tetrahedral soft grids。
- `L68-L80`: 仍然需要 coloring，solver 仍然是 `SolverVBD(...)`。
- `L100-L111`: 外层 loop 和 cloth 例子一样，还是 `collide -> solver.step -> swap states`。

### 它最适合验证什么

- `VBD` 不是“更复杂的 cloth XPBD”，而是一条更一般的局部 force + Hessian 路线。
- 同样的局部 block solve 视角，可以继续处理 volumetric softbody。
- damping、tet elasticity 和 coloring 在这里仍然是活的 solver 要素，而不是附属参数。

### 改哪里最值钱

| 改哪里 | 先看什么 | 这能帮你理解什么 |
|--------|----------|------------------|
| 盯 `L48-L65` 的四组 `k_damp` | 同一 VBD 路线下，材料阻尼会直接改变软体表现 | `VBD` 的局部 solve 不只在 cloth 边和三角形上工作 |
| 盯 `L68-L80` | `builder.color()` 和 `SolverVBD(...)` 仍然是必须条件 | VBD 的 block / color 结构不是 cloth 特例 |
| 盯 `L32-L33` 的 guard | 例子根本不打算做 cross-solver 切换 | 这个例子的任务是“延伸 VBD”，不是“比较三家” |

### 第一遍最值得观察什么

- 你会看到 chapter 09 的主例子和辅例子保持同一个外层 loop，但承担不同教学任务。
- `cloth_hanging` 负责比较三条路怎样处理同一块布。
- `softbody_hanging` 负责补一句: `VBD` 的局部 block / Hessian 思路还能继续走到体单元软体。

### 如果结果和预期不一样, 你大概率误解了什么

- 如果你拿这个例子去问“为什么没有 `Style3D`”，你把它的教学任务想错了。
- 如果你把它理解成“VBD 只是 cloth solver 加了 tet 支持”，你还是低估了局部 block solve 这条主线的通用性。

## 两个例子怎么配合读最顺

推荐顺序不要反过来。

1. 先看 `cloth_hanging`。
原因: 它最适合把 shared problem 和三条更新梯子看顺。

2. 再看 `softbody_hanging`。
原因: 当你已经知道 `VBD` 的局部子问题长什么样，再来看 tet softbody 延伸就不会把它误读成“另一个 demo”。

你也可以把两者压成一句更短的记忆法:

```text
cloth_hanging 负责看“三家怎样解同一张图”
softbody_hanging 负责看“VBD 为什么不只是一条布料路线”
```

## 误解自检

如果你还会这样想，就该回到 `principle.md` 重读那张梯子表:

- “`XPBD`、`VBD`、`Style3D` 只是三种 cloth solver 名字。”
- “`VBD` 只是更复杂的 `XPBD`。”
- “`Style3D` 只是 cloth 特化版 `VBD`。”

更稳的说法应该是:

- `XPBD` 是 per-constraint projection 路线。
- `VBD` 是 per-vertex / per-block local force + Hessian solve 路线。
- `Style3D` 是 cloth-specialized global PD / PCG solve 路线。

## 自检

- 现在只看 `cloth_hanging`，你能不能解释为什么“外层 loop 一样”不代表“内部修正层级一样”？
- 现在只看 `softbody_hanging`，你能不能解释为什么这个例子是在扩展 `VBD`，而不是在替代 chapter 09 的主入口？
