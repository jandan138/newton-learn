# Chapter 10 Deep And Chapter 11 MPM Design

- **日期**: 2026-04-21
- **仓库**: `/home/zhuzihou/dev/newton-learn`
- **范围**:
  - `chapters/10_softbody_cloth_cable/source-walkthrough-deep.md`
  - `chapters/10_softbody_cloth_cable/README.md`
  - `chapters/10_softbody_cloth_cable/source-walkthrough.md`
  - `chapters/11_mpm/README.md`
  - `chapters/11_mpm/principle.md`
  - `chapters/11_mpm/examples.md`
  - `chapters/11_mpm/source-walkthrough.md`
- **目标**: 为 chapter 10 补出深读锚点版，并把 chapter 11 建成一套 teaching-first 的 MPM 主线，保持 `09 -> 10 -> 11` 的阅读连续性。

---

## 1. 这轮要解决什么问题

上一轮已经把 chapter 10 的 mainline 建出来了，但它还缺一个重要配套：

- `source-walkthrough.md` 已经能让新手 first pass 读懂 cloth / softbody / cable 是不同对象家族。
- 但读者如果想继续精确追 `builder.py` 里的真实 handoff，现在还没有 deep-track 可以接住。

与此同时，chapter 11 还是一个空 skeleton。读者读完 chapter 10 后，会自然期待下一个问题：

```text
如果 cloth / softbody / cable 讲的是对象家族，
那 MPM 这条路线又是什么对象？它为什么不是前面那套粒子网格或刚体链？
```

所以这轮要同时补两件事：

1. 给 chapter 10 补出 deep walkthrough。
2. 给 chapter 11 补出完整 mainline，且不能写成 generic MPM 教材或 solver 选项目录。

---

## 2. Chapter 10 Deep 应该怎么写

### 2.1 角色定位

chapter 10 的 deep walkthrough 不应该重复 main walkthrough 的 beginner 故事线，也不应该退化成 solver 内部细节。它的角色应该和 chapter 09 deep 一样：

- 保留精确 cross-repo anchor
- 把 builder-side handoff 追实
- 给 optional branches 和 verification 留落点

更准确地说，chapter 10 deep 要回答的是：

```text
main walkthrough 已经告诉我这三种对象家族不同。
那它们到底在 builder 里怎样一步步长出来？
```

### 2.2 推荐结构

沿用已经稳定的 deep 结构：

- `Fast Deep Index`
- `Exact Handoff Trace`
- `Optional Branches`
- `Verification Anchors`

### 2.3 推荐主线

`Exact Handoff Trace` 建议分成五段：

1. `Builder already splits the families`
2. `Cloth: add_cloth_grid -> add_cloth_mesh`
3. `Softbody: add_soft_grid -> particles -> tetrahedra -> generated surface mesh`
4. `Cable: geometry path -> add_rod -> add_joint_cable -> add_articulation`
5. `Cross-family state ledger`

### 2.4 推荐锚点

- cloth
  - `example_cloth_hanging.py`
  - `builder.add_cloth_grid(...)`
  - `builder.add_cloth_mesh(...)`
  - `MeshAdjacency(...)`
  - optional `add_springs`
- softbody
  - `example_softbody_hanging.py`
  - `builder.add_soft_grid(...)`
  - `add_tetrahedron(...)`
  - boundary-face bookkeeping
  - generated `add_triangle(...)` / `add_edge(...)`
- cable
  - `example_cable_twist.py`
  - `create_cable_geometry_with_turns(...)`
  - `builder.add_rod(...)`
  - `add_rod_graph(...)`
  - `add_joint_cable(...)`
  - `add_articulation(...)`

### 2.5 风险控制

- 不重复 chapter 09 solver internals。
- 不把 deep walkthrough 写成 `builder.py` 源码抄录。
- 所有深分支都必须继续服务“对象家族 handoff”这条主线。

---

## 3. Chapter 11 应该怎么立主线

### 3.1 最重要的 framing

reviewer 的一致意见是：chapter 11 绝不能写成一个假二分：

```text
explicit APIC vs implicit MPM
```

更稳的讲法应该是：

```text
同一条 MPM 管线里，
particles 先把信息送到 grid，
grid 上做求解，
再把结果送回 particles。

APIC 讲的是 transfer 这一层怎样更丰富；
implicit 讲的是 grid-side solve 怎样更稳。
```

所以 chapter 11 的主线必须是 **dataflow-first**，而不是 solver-name-first。

### 3.2 推荐 beginner 问题

这一章最适合从下面这个问题开始：

```text
前面几章的 particles 多半直接挂在 cloth / softbody 拓扑上。
为什么到了 MPM，这些粒子还要先把信息“泼”到 grid，再从 grid 取回来？
```

### 3.3 推荐主线

chapter 11 的 mainline 建议压成这一条单一管线：

```text
particles carry material/history
-> P2G / APIC transfer to per-step grid workspace
-> implicit grid-side rheology / collision solve
-> G2P / APIC transfer back to particles
-> particle position / velocity / stress / strain update
```

这条线最适合让读者先看懂：

1. MPM 不是 cloth / softbody 那种固定拓扑粒子网格。
2. particle 是 persistent material carriers。
3. grid 是 per-step workspace，不是最终对象本体。
4. APIC 和 implicit solve 分别站在 transfer 和 grid solve 两层。

### 3.4 推荐例子分工

- `example_mpm_granular.py`
  - chapter 11 主例子
  - 负责建立最小 end-to-end 数据流
- `example_mpm_snow_ball.py`
  - 材料历史分支
  - 负责把 `particle_Jp`、`particle_stress`、snow rheology 讲成“粒子携带历史”的证据
- `example_mpm_twoway_coupling.py`
  - advanced branch
  - 负责说明 MPM 怎样和 rigid bodies 互相传递 impulses

### 3.5 推荐源码锚点

- `SolverImplicitMPM.register_custom_attributes(...)`
- `ImplicitMPMModel.setup_particle_material(...)`
- `SolverImplicitMPM.step(...)`
- `_compute_unconstrained_velocity(...)`
- `integrate_velocity(...)`
- `integrate_velocity_apic(...)`
- `_build_elasticity_system(...)`
- `_build_plasticity_system(...)`
- `_solve_rheology(...)`
- `advect_particles(...)`
- `update_particle_strains(...)`
- `_rasterize_colliders(...)`
- `project_outside(...)`

### 3.6 README / principle / examples / walkthrough 的分工

- `README.md`
  - 建立本章边界：这是 `particle -> grid -> particle` 的章节。
  - 明确 APIC 不是另一条平行 solver chapter，而是 transfer 层的关键词。
- `principle.md`
  - 先讲 MPM object model：particle carries state, grid is workspace。
  - 再讲 `P2G -> grid solve -> G2P` 三段。
- `examples.md`
  - 分清 `granular`、`snow_ball`、`twoway_coupling` 三个例子的唯一 job。
- `source-walkthrough.md`
  - 追一条主线：`register_custom_attributes -> model/state.mpm fields -> step() -> P2G -> implicit solve -> G2P`。

---

## 4. 这一章最容易写坏的方式

### 4.1 对 chapter 10 deep

- 把 main walkthrough 再抄一遍，只是多加行号。
- 把 solver 细节重新写成长文，丢掉 builder-side representation 主线。
- 没有明确 cross-family 对照，导致 deep 文档变成三个孤岛。

### 4.2 对 chapter 11

- 一开头就掉进 constitutive law、Hencky strain、Drucker-Prager 公式。
- 写成 demo catalog：`granular / snow / twoway coupling` 平铺并列。
- 写成 solver/config catalog：`grid_type`、`strain_basis`、`solver`、`warmstart_mode` 平铺目录。
- 把 chapter 11 误写成“APIC solver vs implicit solver”的二选一故事。
- 没有把 Newton-specific state placement 讲清：`model.mpm.*`、`state.mpm.*`、`register_custom_attributes(...)`。

---

## 5. 成功标准

如果这轮做对了，应该出现下面结果：

1. chapter 10 deep 能让读者精确追 `builder.py` 里的 family split，而不用退回 main walkthrough 重新找锚点。
2. chapter 11 不会被读成 generic MPM 课程，而会被读成 Newton 里的一条清晰数据流。
3. `09 -> 10 -> 11` 的连续性会变成：

```text
09: 同一问题，不同修正层级
-> 10: 不同对象家族，不同 builder handoff
-> 11: 又一种对象家族：material points + per-step grid workspace
```

4. 读者读完 chapter 11 mainline 后，应该至少能自己讲顺这一句：

```text
MPM 里真正长期存在的是粒子携带的材料和历史状态；
grid 不是对象本体，而是每一步拿来做 transfer、碰撞和隐式求解的工作空间。
```
