# 02 Image Inventory

Current `##` and `###` heading coverage for the chapter 02 full image pass.

Inventory conventions:

- `status` is tracked per row, not per underlying asset file.
- `planned` means the heading still needs its visual anchor integrated in that section, even if a reused canonical asset already exists elsewhere in the chapter.
- `integrated` means that heading already has its visual anchor in markdown.
- The `file` column repeats intentionally so the table can still be exported, sorted, or grepped as row-oriented data later.

## README.md

| file | heading | level | image_type | accuracy | mode | asset | source_anchors | status |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| README.md | Source Note | ## | navigation-map | teaching-compressed | lightweight | chapters/02_newton_arch/assets/02_readme_source_note_scope_map.svg | - | integrated |
| README.md | 文件分工 | ## | navigation-map | teaching-compressed | fresh | chapters/02_newton_arch/assets/02_readme_file_roles_map.svg | - | integrated |
| README.md | Core Pass | ## | runtime-bridge | teaching-compressed | fresh | chapters/02_newton_arch/assets/02_readme_core_pass_bridge.svg | - | integrated |
| README.md | Defer For Now | ## | navigation-map | teaching-compressed | lightweight | chapters/02_newton_arch/assets/02_readme_defer_now_map.svg | - | integrated |
| README.md | 完成门槛 | ## | concept-poster | teaching-compressed | lightweight | chapters/02_newton_arch/assets/02_readme_completion_gate_poster.svg | - | integrated |
| README.md | 本章目标 | ## | concept-poster | teaching-compressed | fresh | chapters/02_newton_arch/assets/02_readme_chapter_goals_poster.svg | - | integrated |
| README.md | 前置依赖 | ## | navigation-map | teaching-compressed | fresh | chapters/02_newton_arch/assets/02_readme_prereq_bridge.svg | - | integrated |
| README.md | GAMES103 已有 vs 本章新增 | ## | concept-poster | teaching-compressed | fresh | chapters/02_newton_arch/assets/02_readme_games103_delta_poster.svg | - | integrated |
| README.md | 阅读顺序 | ## | navigation-map | teaching-compressed | fresh | chapters/02_newton_arch/assets/02_readme_reading_order_map.svg | - | integrated |
| README.md | 预期产出 | ## | navigation-map | teaching-compressed | lightweight | chapters/02_newton_arch/assets/02_readme_expected_outputs_map.svg | - | integrated |

## principle.md

| file | heading | level | image_type | accuracy | mode | asset | source_anchors | status |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| principle.md | 0. 从 `00` 和 `01` 走到 Newton 本体 | ## | runtime-bridge | teaching-compressed | fresh | chapters/02_newton_arch/assets/02_principle_00_01_to_newton_bridge.svg | - | integrated |
| principle.md | 1. 从 `basic_pendulum` 走到一次 step | ## | runtime-bridge | source-exact | fresh | chapters/02_newton_arch/assets/02_principle_basic_pendulum_step_bridge.svg | examples launcher -> `example_basic_pendulum.py:Example.__init__()` -> `simulate()` -> `solver.step()` | integrated |
| principle.md | 2. 四层关系图 | ## | concept-poster | teaching-compressed | reuse | chapters/02_newton_arch/assets/02_model_state_control_solver.svg | - | integrated |
| principle.md | 3. 8 个 solver 的全景图 | ## | navigation-map | teaching-compressed | reuse | chapters/02_newton_arch/assets/02_solver_family_map.svg | - | integrated |
| principle.md | 4. 快速胜利示例 | ## | runtime-bridge | teaching-compressed | fresh | chapters/02_newton_arch/assets/02_principle_quick_win_examples_bridge.svg | - | integrated |
| principle.md | 5. 与后续章节的接口 | ## | navigation-map | teaching-compressed | fresh | chapters/02_newton_arch/assets/02_principle_next_chapters_map.svg | - | integrated |
| principle.md | 6. 与 Newton 实现的映射 | ## | runtime-bridge | source-exact | fresh | chapters/02_newton_arch/assets/02_principle_newton_impl_handoff_bridge.svg | `newton/examples/__main__.py`; `newton/__init__.py`; `newton/_src/core/` public/core boundary | integrated |

## examples.md

| file | heading | level | image_type | accuracy | mode | asset | source_anchors | status |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| examples.md | 主例子：`basic_pendulum` | ## | runtime-bridge | source-exact | fresh | chapters/02_newton_arch/assets/02_examples_basic_pendulum_scope_bridge.svg | `newton/examples/basic/example_basic_pendulum.py:Example.__init__(), simulate()` | integrated |
| examples.md | 最适合拿来验证什么 | ### | concept-poster | teaching-compressed | lightweight | chapters/02_newton_arch/assets/02_examples_basic_pendulum_validation_poster.svg | - | integrated |
| examples.md | 它覆盖的架构链 | ### | runtime-bridge | source-exact | fresh | chapters/02_newton_arch/assets/02_examples_basic_pendulum_chain_bridge.svg | launcher -> `example_basic_pendulum.py` -> `ModelBuilder.finalize()` -> `state()/control()/contacts()` -> `simulate()` | integrated |
| examples.md | 跑完后先盯这几个位置 | ### | question-poster | teaching-compressed | fresh | chapters/02_newton_arch/assets/02_examples_basic_pendulum_checkpoints_map.svg | - | integrated |
| examples.md | 改这里会怎样 | ### | question-poster | teaching-compressed | fresh | chapters/02_newton_arch/assets/02_examples_basic_pendulum_knobs_map.svg | - | integrated |
| examples.md | 这一例子最容易看错的地方 | ### | question-poster | teaching-compressed | lightweight | chapters/02_newton_arch/assets/02_examples_basic_pendulum_pitfalls_poster.svg | - | integrated |
| examples.md | 对照例子：`robot_cartpole --world-count 100` | ## | runtime-bridge | source-exact | fresh | chapters/02_newton_arch/assets/02_examples_robot_cartpole_scope_bridge.svg | `example_robot_cartpole.py:add_usd()`; `builder.replicate(..., world_count=100)`; `SolverMuJoCo(self.model)` | integrated |
| examples.md | 它最适合补哪一块观察 | ### | concept-poster | teaching-compressed | lightweight | chapters/02_newton_arch/assets/02_examples_robot_cartpole_observation_poster.svg | - | integrated |
| examples.md | 它覆盖的架构链 | ### | runtime-bridge | source-exact | fresh | chapters/02_newton_arch/assets/02_examples_robot_cartpole_chain_bridge.svg | `add_usd(...)` -> `builder.replicate(...)` -> `finalize()` -> `SolverMuJoCo.step(...)` | integrated |
| examples.md | 改这里会怎样 | ### | question-poster | teaching-compressed | lightweight | chapters/02_newton_arch/assets/02_examples_robot_cartpole_knobs_poster.svg | - | integrated |
| examples.md | 对照例子：`cloth_hanging --solver xpbd` | ## | runtime-bridge | source-exact | fresh | chapters/02_newton_arch/assets/02_examples_cloth_hanging_scope_bridge.svg | `example_cloth_hanging.py:add_cloth_grid()`; `model.contacts()`; `model.collide()` | integrated |
| examples.md | 它最适合补哪一块观察 | ### | concept-poster | teaching-compressed | lightweight | chapters/02_newton_arch/assets/02_examples_cloth_hanging_observation_poster.svg | - | integrated |
| examples.md | 它覆盖的架构链 | ### | runtime-bridge | source-exact | fresh | chapters/02_newton_arch/assets/02_examples_cloth_hanging_chain_bridge.svg | CLI `--solver` -> `add_ground_plane()/add_cloth_grid()` -> `contacts()` -> `collide()` -> `SolverXPBD.step(...)` | integrated |
| examples.md | 改这里会怎样 | ### | question-poster | teaching-compressed | lightweight | chapters/02_newton_arch/assets/02_examples_cloth_hanging_knobs_poster.svg | - | integrated |
| examples.md | 这页怎么配合其他文件 | ## | navigation-map | teaching-compressed | fresh | chapters/02_newton_arch/assets/02_examples_page_relationships_map.svg | - | integrated |

## source-walkthrough.md

| file | heading | level | image_type | accuracy | mode | asset | source_anchors | status |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| source-walkthrough.md | What This Walkthrough Follows | ## | runtime-bridge | source-exact | fresh | chapters/02_newton_arch/assets/02_walkthrough_scope_bridge.svg | short name -> launcher -> concrete module -> `Example.__init__()` -> runtime objects -> `simulate()` | integrated |
| source-walkthrough.md | One-Screen Chapter Map | ## | runtime-bridge | source-exact | fresh | chapters/02_newton_arch/assets/02_walkthrough_one_screen_map.svg | `python -m newton.examples basic_pendulum` -> `examples.__init__.main()` -> `builder.finalize()` -> `solver.step()` | integrated |
| source-walkthrough.md | Beginner Path | ## | navigation-map | teaching-compressed | fresh | chapters/02_newton_arch/assets/02_walkthrough_beginner_path_map.svg | - | integrated |
| source-walkthrough.md | Main Walkthrough | ## | navigation-map | teaching-compressed | fresh | chapters/02_newton_arch/assets/02_walkthrough_stage_overview_map.svg | - | integrated |
| source-walkthrough.md | Stage 1: `basic_pendulum` 先只是一个 example short name | ### | runtime-bridge | source-exact | fresh | chapters/02_newton_arch/assets/02_walkthrough_stage1_short_name_bridge.svg | `newton/examples/__init__.py:get_examples()`; `filename[8:-3]` -> short name | integrated |
| source-walkthrough.md | Stage 2: launcher 把 short name 送到真实 example module | ### | runtime-bridge | source-exact | reuse | chapters/02_newton_arch/assets/02_launcher_handoff.svg | `newton/examples/__main__.py` -> `examples.__init__.main()` -> `runpy.run_module(target_module)` | integrated |
| source-walkthrough.md | Stage 3: concrete `Example` 对象第一次把四层主干和 `Contacts` 结果缓冲区组出来 | ### | runtime-bridge | source-exact | fresh | chapters/02_newton_arch/assets/02_walkthrough_stage3_runtime_stack_bridge.svg | `newton/__init__.py` public exports; `example_basic_pendulum.py:builder.finalize(), state(), control(), contacts(), eval_fk()` | integrated |
| source-walkthrough.md | Stage 4: 四层主干 + `Contacts` 结果缓冲区是分工链，不是一团近义词 | ### | concept-poster | source-exact | fresh | chapters/02_newton_arch/assets/02_walkthrough_stage4_object_roles_poster.svg | `newton/_src/sim/model.py:state(), control(), contacts(), collide()`; `newton/_src/solvers/solver.py:step(...)` | integrated |
| source-walkthrough.md | Stage 5: `simulate()` 把这些对象真的接成一轮仿真 | ### | runtime-bridge | source-exact | fresh | chapters/02_newton_arch/assets/02_walkthrough_stage5_simulate_loop_bridge.svg | `examples.run()`; `example_basic_pendulum.py:simulate(), step()`; `clear_forces -> apply_forces -> collide -> solver.step -> swap` | integrated |
| source-walkthrough.md | Object Ledger | ## | concept-poster | source-exact | fresh | chapters/02_newton_arch/assets/02_walkthrough_object_ledger_poster.svg | `builder.finalize()`; `Model.state()/control()/contacts()`; `eval_fk()`; `Solver.step(...)` | integrated |
| source-walkthrough.md | Stop Here | ## | navigation-map | teaching-compressed | lightweight | chapters/02_newton_arch/assets/02_walkthrough_stop_here_gate.svg | - | integrated |
| source-walkthrough.md | Go Deeper | ## | navigation-map | teaching-compressed | fresh | chapters/02_newton_arch/assets/02_walkthrough_go_deeper_map.svg | - | integrated |

## source-walkthrough-deep.md

| file | heading | level | image_type | accuracy | mode | asset | source_anchors | status |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| source-walkthrough-deep.md | Fast Deep Index | ## | navigation-map | source-exact | fresh | chapters/02_newton_arch/assets/02_walkthrough_deep_fast_index_map.svg | D1-D7 across `newton/__init__.py`, examples, `model.py`, `state.py`, `solver.py`, `builder.py` | integrated |
| source-walkthrough-deep.md | Exact Handoff Trace | ## | runtime-bridge | source-exact | fresh | chapters/02_newton_arch/assets/02_walkthrough_deep_exact_handoff_trace.svg | `__main__.py` -> `examples.__init__.py` -> `example_basic_pendulum.py` -> `model.py` -> `solver.py` | integrated |
| source-walkthrough-deep.md | 1. 顶层 API 与 examples 路由 | ### | runtime-bridge | source-exact | fresh | chapters/02_newton_arch/assets/02_walkthrough_deep_top_api_route_bridge.svg | `newton/__init__.py:49-98`; `newton/examples/__main__.py:4-9`; `newton/examples/__init__.py:395-407,702-733` | integrated |
| source-walkthrough-deep.md | 2. `basic_pendulum` 怎样组装 runtime stack | ### | runtime-bridge | source-exact | fresh | chapters/02_newton_arch/assets/02_walkthrough_deep_pendulum_runtime_stack.svg | `newton/examples/basic/example_basic_pendulum.py:20-85,148-155` | integrated |
| source-walkthrough-deep.md | Optional Deep Note: 怎样读一个 revolute joint | ### | question-poster | source-exact | reuse | chapters/02_newton_arch/assets/02_joint_steps.png | `example_basic_pendulum.py:j0 parent=-1`; `j1 parent=link_0`; `axis=(0,1,0)`; `parent_xform/child_xform` | integrated |
| source-walkthrough-deep.md | 3. `Model` 是静态描述和对象工厂 | ### | concept-poster | source-exact | fresh | chapters/02_newton_arch/assets/02_walkthrough_deep_model_factory_poster.svg | `newton/_src/sim/model.py:state(), control(), contacts(), collide()`; `newton/_src/sim/state.py:State` | integrated |
| source-walkthrough-deep.md | 4. 外层 run loop 与内层 substep chain | ### | runtime-bridge | source-exact | fresh | chapters/02_newton_arch/assets/02_walkthrough_deep_outer_inner_loop_bridge.svg | `newton/examples/__init__.py:265-337`; `example_basic_pendulum.py:87-113` | integrated |
| source-walkthrough-deep.md | 5. solver contract 的精确入口 | ### | runtime-bridge | source-exact | fresh | chapters/02_newton_arch/assets/02_walkthrough_deep_solver_contract_bridge.svg | `example_basic_pendulum.py:102-103`; `newton/_src/solvers/solver.py:10-157,224-316,243-299`; `solver_xpbd.py:245-609` | integrated |
| source-walkthrough-deep.md | Optional Branches | ## | navigation-map | teaching-compressed | fresh | chapters/02_newton_arch/assets/02_walkthrough_deep_optional_branches_map.svg | - | integrated |
| source-walkthrough-deep.md | Branch A: `examples.init()` 的 viewer/device 选择 | ### | navigation-map | source-exact | lightweight | chapters/02_newton_arch/assets/02_walkthrough_deep_branch_a_viewer_device_map.svg | `newton/examples/__init__.py:617-680` | integrated |
| source-walkthrough-deep.md | Branch B: `ModelBuilder.finalize()` 的内部冻结细节 | ### | runtime-bridge | source-exact | lightweight | chapters/02_newton_arch/assets/02_walkthrough_deep_branch_b_finalize_map.svg | `newton/_src/sim/builder.py:9424-10449` | integrated |
| source-walkthrough-deep.md | Branch C: 其它 examples | ### | navigation-map | teaching-compressed | lightweight | chapters/02_newton_arch/assets/02_walkthrough_deep_branch_c_other_examples_map.svg | - | integrated |
| source-walkthrough-deep.md | Verification Anchors | ## | navigation-map | source-exact | fresh | chapters/02_newton_arch/assets/02_walkthrough_deep_verification_anchors_map.svg | public API; launcher; constructor/runtime stack; outer vs inner loop; solver contract checks | integrated |

## question-notes.md

| file | heading | level | image_type | accuracy | mode | asset | source_anchors | status |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| question-notes.md | 1. 为什么 `__main__` 会出现两次 | ## | question-poster | teaching-compressed | reuse | chapters/02_newton_arch/assets/02_cli_flow_summary.png | - | integrated |
| question-notes.md | 2. `joint` 到底在干嘛 | ## | question-poster | source-exact | reuse | chapters/02_newton_arch/assets/02_joint_steps.png | `example_basic_pendulum.py:j0 parent=-1`; `j1 parent=link_0`; `axis=(0,1,0)`; `parent_xform/child_xform` | integrated |
| question-notes.md | 3. `p / q / axis` 为什么这么容易混 | ## | question-poster | teaching-compressed | reuse | chapters/02_newton_arch/assets/02_joint_frame_summary.png | - | integrated |
| question-notes.md | 4. 为什么换一套 joint frame，物理还能不变 | ## | question-poster | teaching-compressed | reuse | chapters/02_newton_arch/assets/02_joint_frame_equivalence.png | - | integrated |
| question-notes.md | 5. 这页怎么和主线配合 | ## | navigation-map | teaching-compressed | fresh | chapters/02_newton_arch/assets/02_question_notes_page_relationships_map.svg | - | integrated |
