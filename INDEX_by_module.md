# INDEX by Module

这个索引用于把上游 Newton 的核心模块快速映射到学习章节，优先服务于原理定位、源码走读和代表性示例回查。

## Module Map

| 上游模块 | 对应章节 |
|----------|----------|
| `newton/__init__.py` | `02_newton_arch` |
| `newton/_src/core/` | `02_newton_arch`, `05_rigid_articulation` |
| `newton/_src/math/` | `03_math_geometry` |
| `newton/_src/geometry/` | `03_math_geometry`, `06_collision` |
| `newton/_src/sim/` | `02_newton_arch`, `04_scene_usd`, `05_rigid_articulation`, `06_collision` |
| `newton/_src/usd/` | `04_scene_usd` |
| `newton/_src/solvers/featherstone/` | `05_rigid_articulation`, `08_rigid_solvers` |
| `newton/_src/solvers/mujoco/` | `08_rigid_solvers` |
| `newton/_src/solvers/semi_implicit/` | `08_rigid_solvers`, `10_softbody_cloth_cable` |
| `newton/_src/solvers/kamino/` | `08_rigid_solvers` |
| `newton/_src/solvers/xpbd/` | `09_variational_solvers`, `10_softbody_cloth_cable` |
| `newton/_src/solvers/vbd/` | `09_variational_solvers`, `10_softbody_cloth_cable` |
| `newton/_src/solvers/style3d/` | `09_variational_solvers`, `10_softbody_cloth_cable` |
| `newton/_src/solvers/implicit_mpm/` | `11_mpm` |
| `newton/_src/sensors/` | `12_sensors_ik` |
| `newton/ik.py` | `12_sensors_ik` |
| `newton/examples/diffsim/` | `13_diffsim` |
| `newton/_src/viewer/` | `14_viewer_integration` |
| `newton/viewer.py` | `14_viewer_integration` |
| `newton/examples/multiphysics/` | `15_multiphysics_pipeline` |
| `newton/examples/` | `02_newton_arch`, `16_self_experiments` |

## Maintenance Rules

1. 每次上游 Newton commit 更新后，先在 `PROGRESS.md` 更新 commit hash，再回查本索引是否需要新增或拆分模块项。
2. 章节名变更时，同步修改本索引、根 README 的章节索引表，以及相关章节 README 的反向引用，避免路径漂移。
3. 涉及源码行号的细粒度引用不要写在这里；本文件只维护模块级映射，行号统一记录在章节笔记中并遵循 `conventions/source-refs.md`。
