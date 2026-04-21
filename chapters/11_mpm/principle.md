---
chapter: 11
title: 11 MPM 原理主线：粒子携材，网格求解
last_updated: 2026-04-21
source_paths:
  - newton/examples/mpm/example_mpm_granular.py
  - newton/examples/mpm/example_mpm_snow_ball.py
  - newton/examples/mpm/example_mpm_twoway_coupling.py
  - newton/_src/solvers/implicit_mpm/solver_implicit_mpm.py
  - newton/_src/solvers/implicit_mpm/implicit_mpm_model.py
  - newton/_src/solvers/implicit_mpm/implicit_mpm_solver_kernels.py
paper_keys:
  - apic-paper
  - implicit-mpm-paper
newton_commit: 1a230702
---

# 11 MPM 原理主线：粒子携材，网格求解

## 0. 先把假二选一拆掉

chapter 11 最容易被误读成这样一组标题:

```text
APIC solver vs implicit solver
```

这个读法不稳，因为它把同一条 timestep 里的两个不同角色硬拆成了两家 solver。

在 Newton 当前这条主线上，更稳的说法是:

- `APIC` 先回答 `粒子和网格之间怎样搬运速度信息`。
- `implicit` 回答 `网格侧怎样更稳定地解 rheology / collision`。

源码其实把这件事写得很直白:

```python
transfer_scheme: Literal["apic", "pic"] = "apic"

def step(...):
    """Transfers particle data to the grid, solves the implicit rheology
    system, and transfers the result back to update particle positions,
    velocities, and stress.
    """
```

所以 chapter 11 的 first-pass 入口必须是 dataflow:

```text
particles carry material/history
-> P2G / APIC
-> implicit grid solve
-> G2P / APIC
-> updated particles
```

## 1. particles are persistent material carriers

MPM 的粒子第一遍不要读成“某个 mesh 的顶点”。更稳的读法是:

```text
particle = 持久存在的材料载体
```

它们跨 timestep 持续存在，携带的是材料相关信息和历史信息，而不是像 chapter 10 那样先靠一套显式 triangle / tet connectivity 定义对象身份。

`example_mpm_granular.py` 和 `example_mpm_snow_ball.py` 都把这个模式写得很清楚:

```python
builder = newton.ModelBuilder()
SolverImplicitMPM.register_custom_attributes(builder)

self.model = builder.finalize()
self.state_0 = self.model.state()
self.state_1 = self.model.state()
```

随后，材料参数和历史变量会直接挂到粒子相关数组上:

```python
model.mpm.young_modulus.fill_(options.young_modulus)
model.mpm.friction.fill_(options.friction_coeff)

self.state_0.mpm.particle_Jp.fill_(0.975)
```

这里最该稳住的不是参数名本身，而是这个 ownership:

- 粒子不是临时计算点；它们会把材料信息和历史带到下一步。
- grid 不是材料本体；第一遍可以把它读成当前一步围着粒子组织起来的主要工作区。
- `state_0` / `state_1` 交换之后，真正持续代表材料与历史的仍然是粒子状态；实现里也会保留少量 grid-derived fields 去服务 warmstart、coupling、rendering 分支。

## 2. grid is per-step workspace

MPM 的另一半核心不是“再来一种 particle solver”，而是:

```text
同一批持久粒子，每一步都会被送进一块临时网格工作区里做求解。
```

`SolverImplicitMPM.step(...)` 的入口顺序就是这样:

```python
pic = self._particles_to_cells(state_in.particle_q)
scratch = self._rebuild_scratchpad(pic)
self._step_impl(state_in, state_out, dt, pic, scratch)
```

而 `_particles_to_cells(...)` 和 `_rebuild_scratchpad(...)` 的文档字符串已经把 grid 的角色写出来了:

```python
def _particles_to_cells(...):
    """Rebuild the grid and grid partition around particles,
    then assign particles to grid cells."""

def _rebuild_scratchpad(...):
    """(Re)create function spaces and allocate per-step temporaries."""
```

所以 chapter 11 里 grid 的关键词是:

- `per-step`
- `workspace`
- `scratch`

而不是“和粒子对等的第二份对象本体”。

更细一点地说，源码里确实有两个 second-pass 事实：

- `grid_type == "fixed"` 时，`_particles_to_cells(...)` 可能复用已有 grid。
- `state_out` 也会暂存 `velocity_field`、`impulse_field` 这类 grid-derived fields，供 coupling / rendering / warmstart 分支继续使用。

但这些都没有改写 chapter 11 的 first-pass 结论：**材料与历史主要挂在粒子上，grid 主要承担本步求解工作区。**

这也是为什么你第一遍不用先背 basis catalog。你只要先知道: grid 每一步围着粒子重建，然后承接本步求解。

## 3. 三步梯子: `P2G -> grid solve -> G2P`

严格说实现里还有 collider rasterization、warmstart、scratch allocation 等中间环节；但 beginner first pass 最稳的 ladder 仍然只有三步。

### 3.1 `P2G`: 把粒子携带的信息转移到本步 grid

最直接的证据在 `_compute_unconstrained_velocity(...)`:

```python
velocity_int = fem.integrate(
    integrate_velocity,
    quadrature=pic,
    values={
        "velocities": state_in.particle_qd,
        "particle_density": mpm_model.particle_density,
        ...
    },
)

if self.apic:
    fem.integrate(
        integrate_velocity_apic,
        quadrature=pic,
        values={
            "velocity_gradients": state_in.mpm.particle_qd_grad,
            ...
        },
        output=velocity_int,
        add=True,
    )
```

这段代码对应的人话是:

- 粒子把质量、速度送到 grid node。
- 如果 transfer scheme 是 `APIC`（Affine Particle-In-Cell），粒子还会额外带一份局部速度梯度 `particle_qd_grad`。
- 这一步的目标不是直接改粒子位置，而是先在网格上建立“本步要解什么”的离散工作区。

### 3.2 `grid solve`: 在 grid side 做 rheology / collision solve

`_step_impl(...)` 把 grid side 的工作顺序写得很清楚:

```python
self._rasterize_colliders(state_in, dt, last_step_data, scratch, inv_cell_volume)
self._compute_unconstrained_velocity(state_in, dt, pic, scratch, inv_cell_volume)
self._build_elasticity_system(state_in, dt, pic, scratch, inv_cell_volume)
self._build_plasticity_system(state_in, dt, pic, scratch, inv_cell_volume)
self._solve_rheology(pic, scratch, rigidity_operator, last_step_data, inv_cell_volume)
```

这里的关键不是把每个子函数都推导一遍，而是看清 solve 发生在哪里:

- collider 先被 rasterize 到 grid fields。
- elastic / plastic system 在 grid / strain spaces 上组装。
- 真正的 implicit solve 用的是 grid-side `MomentumData`、`RheologyData`、`CollisionData`。

同时，材料参数来自 `model.mpm.*`，而不是神秘地漂在 solver 外面:

```python
self.material_parameters.young_modulus = model.mpm.young_modulus
self.material_parameters.poisson_ratio = model.mpm.poisson_ratio
self.material_parameters.friction = model.mpm.friction
self.material_parameters.viscosity = model.mpm.viscosity
```

所以这一层更稳的记法是:

```text
粒子提供材料与历史，grid 负责本步离散求解。
```

### 3.3 `G2P`: 把 grid 结果送回粒子

`_update_particles(...)` 先更新材料相关历史，再做 advection:

```python
fem.interpolate(
    update_particle_strains,
    at=pic,
    values={
        "elastic_strain": state_out.mpm.particle_elastic_strain,
        "particle_stress": state_out.mpm.particle_stress,
        "particle_Jp": state_out.mpm.particle_Jp,
        ...
    },
    fields={
        "grid_vel": scratch.velocity_field,
        "plastic_strain_delta": scratch.plastic_strain_delta_field,
        "elastic_strain_delta": scratch.elastic_strain_delta_field,
        "stress": scratch.stress_field,
    },
)

fem.interpolate(
    advect_particles,
    at=pic,
    values={
        "pos": state_out.particle_q,
        "vel": state_out.particle_qd,
        "vel_grad": state_out.mpm.particle_qd_grad,
        ...
    },
    fields={"grid_vel": scratch.velocity_field},
)
```

这一步对应的人话是:

- solved grid velocity 回到粒子，更新 `particle_q` 和 `particle_qd`。
- APIC 需要的 `particle_qd_grad` 也在这里继续写回粒子。
- stress / strain / `Jp` 这类 history-aware 量同样回到粒子，供下一步继续使用。

所以 chapter 11 的三步梯子不是抽象 slogan，而是会真正写回 `state_out` 的数据流。

## 4. Newton-specific state placement: 东西到底放哪

这一章最值得单独记的一张表，就是 Newton 对 MPM 数据的放置方式。

| 放置位置 | 放什么 | 为什么放这里 |
|----------|--------|--------------|
| `SolverImplicitMPM.register_custom_attributes(builder)` | 声明 `mpm` namespace 里的 model/state attributes | builder 必须先知道这些自定义数组，后面 `finalize()` 才能把它们挂到 `model` / `state` 上 |
| `model.mpm.*` | 每个粒子的材料参数，如 `young_modulus`、`friction`、`yield_pressure` | 这些量更像“材料配方”，通常跨多个 state 持续有效 |
| `state.mpm.*` | 每个粒子的附加状态，如 `particle_qd_grad`、`particle_elastic_strain`、`particle_Jp`、`particle_stress`、`particle_transform` | 这些量会随着 timestep 演化，属于 particle-carried state/history |
| `state.particle_q` / `state.particle_qd` | 通用粒子位置和速度 | MPM 仍然复用 Newton 的基础粒子运动状态，不把所有东西都塞进 `mpm` namespace |
| solver scratch / grid fields | `velocity_field`、`stress_field`、collider fields 等 | 这些是本步 grid workspace，不是长期对象身份 |

`register_custom_attributes(...)` 里的 assignment 也明确区分了两类东西:

```python
name="young_modulus", assignment=newton.Model.AttributeAssignment.MODEL
name="particle_Jp", assignment=newton.Model.AttributeAssignment.STATE
```

这就是 chapter 11 里最实用的一条工程结论:

```text
材料参数放 model.mpm.*；
历史和附加粒子状态放 state.mpm.*；
本步网格解主要活在 scratch / grid fields 里，必要时也会把部分 grid-derived fields 暂存出去服务后续分支。
```

## 5. MPM particles 不是 chapter 10 的 particle cloth / softbody

chapter 10 里，cloth 和 softbody 虽然都用 particles，但那些 particles 仍然被更稳定地读成 mesh / tet 的 vertex family:

- cloth particle 是 surface mesh vertex。
- softbody particle 是 volumetric tet mesh vertex。

chapter 11 的 MPM particle 不是这一路。

| 对比项 | chapter 10 cloth / softbody particles | chapter 11 MPM particles |
|--------|---------------------------------------|--------------------------|
| 第一身份 | 网格或体网格的顶点 | 材料样本 / 载体 |
| 核心拓扑 | 持久的 triangles / tetrahedra | 每步围着粒子重建的 grid workspace |
| solver 读取方式 | 直接消费表面或体拓扑 | 先 `P2G` 到 grid，再 `G2P` 回粒子 |
| 你最该盯的量 | mesh / tet connectivity | `model.mpm.*`、`state.mpm.*`、grid scratch fields |

这就是为什么 chapter 11 必须单列出来，而不能只算成“另一种 particle softbody”。

## 6. 第一遍只带走一句话也够

```text
MPM 的主线不是 “APIC solver vs implicit solver”。
而是粒子持续携带材料和历史，每一步通过 P2G / G2P 与临时网格工作区交换信息，
由网格完成 implicit rheology / collision solve，再把结果写回粒子。
```

如果这句话已经稳定了，下一步就去读 `source-walkthrough.md`，把这条原理主线真正串进源码。
