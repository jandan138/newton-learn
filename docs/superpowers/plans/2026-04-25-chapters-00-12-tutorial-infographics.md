# Chapters 00-12 Tutorial Infographics Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add chapter 03 / 04 style PNG tutorial infographics to every already-written chapter that is still missing image coverage: 00, 01, and 08-12.

**Architecture:** Keep Markdown as the source of truth and add generated PNGs as visual anchors only. Generate native-imagegen raster PNGs into chapter-local `assets/` directories, then insert stable relative Markdown references near matching headings. Do not touch chapters 02-07 or README-only outline chapters 13-16.

**Tech Stack:** Markdown, Codex native image generation / built-in imagegen, shell verification, Pillow for PNG dimension inspection.

---

## File Structure

Create:

- `chapters/00_prerequisites/assets/`
- `chapters/01_warp_basics/assets/`
- `chapters/08_rigid_solvers/assets/`
- `chapters/09_variational_solvers/assets/`
- `chapters/10_softbody_cloth_cable/assets/`
- `chapters/11_mpm/assets/`
- `chapters/12_sensors_ik/assets/`

Modify:

- `chapters/00_prerequisites/README.md`
- `chapters/00_prerequisites/principle.md`
- `chapters/01_warp_basics/README.md`
- `chapters/01_warp_basics/principle.md`
- `chapters/01_warp_basics/source-walkthrough.md`
- `chapters/08_rigid_solvers/README.md`
- `chapters/08_rigid_solvers/principle.md`
- `chapters/08_rigid_solvers/source-walkthrough.md`
- `chapters/08_rigid_solvers/examples.md`
- `chapters/09_variational_solvers/README.md`
- `chapters/09_variational_solvers/principle.md`
- `chapters/09_variational_solvers/source-walkthrough.md`
- `chapters/09_variational_solvers/examples.md`
- `chapters/10_softbody_cloth_cable/README.md`
- `chapters/10_softbody_cloth_cable/principle.md`
- `chapters/10_softbody_cloth_cable/source-walkthrough.md`
- `chapters/10_softbody_cloth_cable/examples.md`
- `chapters/11_mpm/README.md`
- `chapters/11_mpm/principle.md`
- `chapters/11_mpm/source-walkthrough.md`
- `chapters/11_mpm/examples.md`
- `chapters/12_sensors_ik/README.md`
- `chapters/12_sensors_ik/principle.md`
- `chapters/12_sensors_ik/source-walkthrough.md`
- `chapters/12_sensors_ik/examples.md`

Do not modify:

- `chapters/02_newton_arch/**`
- `chapters/03_math_geometry/**`
- `chapters/04_scene_usd/**`
- `chapters/05_rigid_articulation/**`
- `chapters/06_collision/**`
- `chapters/07_constraints_contacts_math/**`
- `chapters/13_diffsim/**`
- `chapters/14_viewer_integration/**`
- `chapters/15_multiphysics_pipeline/**`
- `chapters/16_self_experiments/**`
- any `source-walkthrough-deep.md`

## Image Inventory

Chapter 00 assets:

- `00_readme_prereq_spine.png`
- `00_readme_games103_bridge.png`
- `00_readme_reading_order_outputs.png`
- `00_principle_why_ramp_before_newton.png`
- `00_principle_sim_step_loop.png`
- `00_principle_rigidbody_vocab_map.png`
- `00_principle_gpu_warp_mental_model.png`
- `00_principle_jump_to_01_or_02.png`

Chapter 01 assets:

- `01_readme_warp_spine.png`
- `01_readme_file_roles_reading_order.png`
- `01_readme_completion_scope_prereq.png`
- `01_principle_why_warp_before_newton.png`
- `01_principle_cpu_loop_to_kernel.png`
- `01_principle_wp_array_mental_model.png`
- `01_principle_launch_tid_batch_execution.png`
- `01_principle_atomic_add_shared_write.png`
- `01_principle_tile_local_reuse.png`
- `01_principle_graph_execution_pack.png`
- `01_principle_newton_entry_map.png`
- `01_walkthrough_pipeline_overview.png`
- `01_walkthrough_beginner_path.png`
- `01_walkthrough_stage1_kernel_rule.png`
- `01_walkthrough_stage2_launch_tid_batch.png`
- `01_walkthrough_stage3_model_state_control_arrays.png`
- `01_walkthrough_stage4_atomic_add_shared_write.png`
- `01_walkthrough_stage5_graph_tile_execution.png`
- `01_walkthrough_object_ledger_stop_here.png`

Chapter 08 assets:

- `08_readme_solver_family_spine.png`
- `08_readme_file_roles_reading_order.png`
- `08_readme_completion_scope_prereq.png`
- `08_principle_after_07_to_solver_family.png`
- `08_principle_public_contract_routes.png`
- `08_principle_coordinate_family_split.png`
- `08_principle_semiimplicit_force_route.png`
- `08_principle_featherstone_joint_space_route.png`
- `08_principle_mujoco_backend_bridge.png`
- `08_principle_kamino_contact_math_consumer.png`
- `08_principle_solver_family_table.png`
- `08_principle_common_misconceptions.png`
- `08_walkthrough_pipeline_overview.png`
- `08_walkthrough_beginner_path.png`
- `08_walkthrough_stage1_public_contract.png`
- `08_walkthrough_stage2_semiimplicit_force_route.png`
- `08_walkthrough_stage3_featherstone_joint_space.png`
- `08_walkthrough_stage4_mujoco_backend_bridge.png`
- `08_walkthrough_stage5_kamino_contact_continuation.png`
- `08_walkthrough_object_ledger_stop_here.png`
- `08_examples_overview_observation_tasks.png`
- `08_examples_cartpole_solver_swap.png`
- `08_examples_kamino_box_on_plane.png`
- `08_examples_self_check_handoff.png`

Chapter 09 assets:

- `09_readme_variational_solver_spine.png`
- `09_readme_file_roles_reading_order.png`
- `09_readme_completion_scope_prereq.png`
- `09_principle_after_08_contract.png`
- `09_principle_hanging_cloth_shared_problem.png`
- `09_principle_prediction_correction_loop.png`
- `09_principle_semiimplicit_baseline.png`
- `09_principle_xpbd_constraint_projection.png`
- `09_principle_vbd_vertex_block_solve.png`
- `09_principle_style3d_pd_pcg_route.png`
- `09_principle_route_ladder_misconceptions.png`
- `09_walkthrough_pipeline_overview.png`
- `09_walkthrough_beginner_path.png`
- `09_walkthrough_stage1_shared_cloth_loop.png`
- `09_walkthrough_stage2_xpbd_constraint_route.png`
- `09_walkthrough_stage3_vbd_local_problem.png`
- `09_walkthrough_stage4_style3d_global_route.png`
- `09_walkthrough_object_ledger_stop_here.png`
- `09_examples_overview_observation_tasks.png`
- `09_examples_cloth_hanging_xpbd_vbd.png`
- `09_examples_softbody_hanging_vbd_extension.png`
- `09_examples_self_check_handoff.png`

Chapter 10 assets:

- `10_readme_soft_cloth_cable_spine.png`
- `10_readme_file_roles_reading_order.png`
- `10_readme_completion_scope_prereq.png`
- `10_principle_after_09_representation_question.png`
- `10_principle_representation_family_map.png`
- `10_principle_three_object_family_compare.png`
- `10_principle_cloth_surface_grid.png`
- `10_principle_softbody_tetrahedral_volume.png`
- `10_principle_cable_rigidbody_chain.png`
- `10_principle_representation_first_reason.png`
- `10_principle_state_field_changes_misconceptions.png`
- `10_walkthrough_pipeline_overview.png`
- `10_walkthrough_beginner_path.png`
- `10_walkthrough_stage1_family_split.png`
- `10_walkthrough_stage2_cloth_particles_triangles.png`
- `10_walkthrough_stage3_softbody_particles_tetrahedra.png`
- `10_walkthrough_stage4_cable_capsule_chain.png`
- `10_walkthrough_object_ledger_stop_here.png`
- `10_examples_overview_observation_tasks.png`
- `10_examples_cloth_hanging_anchor.png`
- `10_examples_softbody_hanging_anchor.png`
- `10_examples_cable_twist_anchor.png`
- `10_examples_self_check_handoff.png`

Chapter 11 assets:

- `11_readme_mpm_spine.png`
- `11_readme_file_roles_examples_order.png`
- `11_readme_completion_scope_prereq.png`
- `11_principle_false_choice_split.png`
- `11_principle_particle_grid_loop.png`
- `11_principle_persistent_material_particles.png`
- `11_principle_grid_per_step_workspace.png`
- `11_principle_p2g_grid_solve_g2p_ladder.png`
- `11_principle_newton_state_placement.png`
- `11_principle_mpm_vs_mesh_particles.png`
- `11_walkthrough_pipeline_overview.png`
- `11_walkthrough_beginner_path.png`
- `11_walkthrough_stage1_apic_vs_implicit.png`
- `11_walkthrough_stage2_persistent_particle_carrier.png`
- `11_walkthrough_stage3_p2g_apic_grid_workspace.png`
- `11_walkthrough_stage4_implicit_grid_solve.png`
- `11_walkthrough_stage5_g2p_apic_writeback.png`
- `11_walkthrough_object_ledger_stop_here.png`
- `11_examples_overview_observation_tasks.png`
- `11_examples_mpm_granular_anchor.png`
- `11_examples_mpm_snow_ball_anchor.png`
- `11_examples_mpm_twoway_coupling_anchor.png`
- `11_examples_self_check_handoff.png`

Chapter 12 assets:

- `12_readme_sensors_ik_spine.png`
- `12_readme_file_roles_examples_order.png`
- `12_readme_completion_scope_prereq.png`
- `12_principle_title_read_then_write.png`
- `12_principle_read_write_backbone.png`
- `12_principle_fk_state_backbone.png`
- `12_principle_sensor_branches.png`
- `12_principle_ik_write_joint_q_branch.png`
- `12_principle_update_order_gotchas.png`
- `12_principle_one_sentence_handoff.png`
- `12_walkthrough_pipeline_overview.png`
- `12_walkthrough_beginner_path.png`
- `12_walkthrough_stage1_shared_backbone.png`
- `12_walkthrough_stage2_sensor_imu_read_branch.png`
- `12_walkthrough_stage3_sensor_contact_side_ledger.png`
- `12_walkthrough_stage4_ik_franka_objectives.png`
- `12_walkthrough_stage5_iksolver_residual_update.png`
- `12_walkthrough_stage6_eval_fk_rejoin.png`
- `12_walkthrough_object_ledger_stop_here.png`
- `12_examples_overview_observation_tasks.png`
- `12_examples_sensor_imu_anchor.png`
- `12_examples_sensor_contact_anchor.png`
- `12_examples_ik_franka_anchor.png`
- `12_examples_advanced_branches_map.png`
- `12_examples_self_check_handoff.png`

## Task 1: Prepare Workspace And Chapter Directories

**Files:**
- Create directories under `chapters/*/assets/`.
- Modify: none.

- [ ] **Step 1: Confirm branch and cleanliness**

Run:

```bash
git status --short --branch
```

Expected: branch is `ch00-12-tutorial-infographics` and no tracked changes except this plan/spec before committing them.

- [ ] **Step 2: Create missing asset directories**

Run:

```bash
mkdir -p \
  chapters/00_prerequisites/assets \
  chapters/01_warp_basics/assets \
  chapters/08_rigid_solvers/assets \
  chapters/09_variational_solvers/assets \
  chapters/10_softbody_cloth_cable/assets \
  chapters/11_mpm/assets \
  chapters/12_sensors_ik/assets
```

Expected: directories exist.

- [ ] **Step 3: Commit the spec and plan**

Run:

```bash
git add docs/superpowers/specs/2026-04-25-chapters-00-12-tutorial-infographics-design.md docs/superpowers/plans/2026-04-25-chapters-00-12-tutorial-infographics.md
git commit -m "docs: plan chapters 00-12 tutorial infographics"
```

Expected: commit succeeds.

## Task 2: Generate Calibration Images

**Files:**
- Create PNGs listed in the calibration set.

- [ ] **Step 1: Generate the seven calibration PNGs with Codex native imagegen**

Use this prompt frame for each output path:

```text
Use the imagegen skill and the built-in image generation tool only.
Do not use CLI fallback, do not ask for OPENAI_API_KEY, and do not create SVG/HTML.

Generate one raster Chinese tutorial infographic for Newton Learn.
Use case: scientific-educational / infographic-diagram.
Style: chapter 03 / 04 dense Chinese learning handout: white background, thin blue rounded card borders, dark-blue title, numbered blue badges, compact multi-panel layout, arrows, flow strips, comparison cards, small friendly icons, readable short labels.
Avoid: Chapter 02 3D diorama style, photorealistic rendering, fake terminal UI, fake code, long paragraphs, unreadable tiny labels.
Save or move the final generated image to <workspace/output/path.png>.
```

Generate these exact content targets:

```text
chapters/00_prerequisites/assets/00_principle_sim_step_loop.png
Title: 00 仿真一步的共同骨架
Show: read state -> compute forces/constraints -> solve/update -> write next state -> repeat. Include small cards for state, control, force, constraint, timestep.

chapters/01_warp_basics/assets/01_principle_cpu_loop_to_kernel.png
Title: 01 从 CPU for-loop 到 Warp kernel
Show: one Python loop on left, many parallel lanes on right, `wp.kernel`, `wp.tid()`, `wp.launch` labels.

chapters/08_rigid_solvers/assets/08_principle_solver_family_split.png
Title: 08 同一个 step 壳，四条 solver 路线
Show: shared `solver.step(model, state_in, state_out, control, contacts)` contract splitting into SemiImplicit force route, Featherstone joint-space route, MuJoCo backend bridge, Kamino contact-row route.

chapters/09_variational_solvers/assets/09_principle_prediction_correction_loop.png
Title: 09 先预测，再修正
Show: hanging cloth predicted positions, correction loop, XPBD per-constraint projection, VBD local vertex/block solve, Style3D PD/PCG route.

chapters/10_softbody_cloth_cable/assets/10_principle_representation_family_map.png
Title: 10 先认表示，再谈 solver
Show: cloth as particles+triangles surface, softbody as particles+tetrahedra volume, cable as rigid capsule chain with joints.

chapters/11_mpm/assets/11_principle_particle_grid_loop.png
Title: 11 MPM: 粒子携材，网格求解
Show: particles carrying material data, P2G arrow to grid workspace, grid solve, G2P arrow back to particles.

chapters/12_sensors_ik/assets/12_principle_read_write_backbone.png
Title: 12 先读，再写回
Show: FK/state backbone in center, sensors as read-side branches, IK as write-side branch that updates `joint_q`.
```

Expected: each path exists and is a PNG.

- [ ] **Step 2: Verify calibration files**

Run:

```bash
file chapters/00_prerequisites/assets/00_principle_sim_step_loop.png \
  chapters/01_warp_basics/assets/01_principle_cpu_loop_to_kernel.png \
  chapters/08_rigid_solvers/assets/08_principle_solver_family_split.png \
  chapters/09_variational_solvers/assets/09_principle_prediction_correction_loop.png \
  chapters/10_softbody_cloth_cable/assets/10_principle_representation_family_map.png \
  chapters/11_mpm/assets/11_principle_particle_grid_loop.png \
  chapters/12_sensors_ik/assets/12_principle_read_write_backbone.png
```

Expected: every line includes `PNG image data`.

- [ ] **Step 3: Review calibration**

Check the seven PNGs against `docs/superpowers/specs/chapter-visual-style-guide.md`.

Expected:

- not 02 diorama style
- concrete sketches present
- Chinese-first labels
- real chapter terms only
- readable enough at Markdown size

Regenerate any failed calibration before proceeding.

- [ ] **Step 4: Commit calibration images**

Run:

```bash
git add chapters/00_prerequisites/assets chapters/01_warp_basics/assets chapters/08_rigid_solvers/assets chapters/09_variational_solvers/assets chapters/10_softbody_cloth_cable/assets chapters/11_mpm/assets chapters/12_sensors_ik/assets
git commit -m "docs: calibrate chapters 00-12 tutorial infographics"
```

Expected: commit succeeds.

## Task 3: Generate Remaining Chapter 00 And 01 Images

**Files:**
- Create remaining `00_*.png` and `01_*.png`.

- [ ] **Step 1: Generate remaining Chapter 00 images**

Use the same imagegen prompt frame from Task 2 and generate:

```text
00_readme_prereq_spine.png: Chapter 00 entry map from math/GPU vocabulary to chapters 01 and 02.
00_readme_games103_bridge.png: GAMES103已有 vs 本章新增 comparison sheet.
00_readme_reading_order_outputs.png: reading order and expected outputs as a compact checklist map.
00_principle_why_ramp_before_newton.png: why a ramp before Newton reduces friction for learners.
00_principle_rigidbody_vocab_map.png: articulation, generalized coordinates, mass matrix M, Jacobian J, ABA, CRBA vocabulary map.
00_principle_gpu_warp_mental_model.png: GPU/Warp mental bridge: many lanes executing one rule over arrays.
00_principle_jump_to_01_or_02.png: decision map for when to jump to 01 vs 02.
```

Save under `chapters/00_prerequisites/assets/`.

- [ ] **Step 2: Generate remaining Chapter 01 images**

Use the same imagegen prompt frame from Task 2 and generate:

```text
01_readme_warp_spine.png: Chapter 01 spine from CPU loop to Warp arrays to Newton source reading.
01_readme_file_roles_reading_order.png: README/principle/source/deep file roles and reading order.
01_readme_completion_scope_prereq.png: completion gate, scope, prerequisites.
01_principle_why_warp_before_newton.png: why Newton source needs Warp mental model first.
01_principle_wp_array_mental_model.png: `wp.array` as typed device-side data buffer, not a Python list.
01_principle_launch_tid_batch_execution.png: `wp.launch` + `wp.tid()` batch execution.
01_principle_atomic_add_shared_write.png: shared writes and why `wp.atomic_add` appears.
01_principle_tile_local_reuse.png: tile as local reuse / staged computation.
01_principle_graph_execution_pack.png: Graph as packed repeated execution.
01_principle_newton_entry_map.png: how these concepts prepare chapter 02 source reading.
01_walkthrough_pipeline_overview.png: source walkthrough pipeline overview.
01_walkthrough_beginner_path.png: beginner path checkpoints.
01_walkthrough_stage1_kernel_rule.png: Stage 1 kernel as single-element rule.
01_walkthrough_stage2_launch_tid_batch.png: Stage 2 launch/tid batch execution.
01_walkthrough_stage3_model_state_control_arrays.png: Stage 3 Model/State/Control arrays.
01_walkthrough_stage4_atomic_add_shared_write.png: Stage 4 shared write / atomic add.
01_walkthrough_stage5_graph_tile_execution.png: Stage 5 graph and tile execution organization.
01_walkthrough_object_ledger_stop_here.png: object ledger and stop-here recap.
```

Save under `chapters/01_warp_basics/assets/`.

- [ ] **Step 3: Verify Chapter 00 and 01 PNG counts**

Run:

```bash
printf '00 assets '; find chapters/00_prerequisites/assets -maxdepth 1 -type f -name '00_*.png' | wc -l
printf '01 assets '; find chapters/01_warp_basics/assets -maxdepth 1 -type f -name '01_*.png' | wc -l
```

Expected:

```text
00 assets 8
01 assets 19
```

## Task 4: Generate Chapter 08 And 09 Images

**Files:**
- Create remaining `08_*.png` and `09_*.png`.

- [ ] **Step 1: Generate remaining Chapter 08 images**

Use the same imagegen prompt frame from Task 2 and generate every Chapter 08 asset listed in Image Inventory except `08_principle_solver_family_split.png`, which was already generated during calibration.

Key content constraints:

- `SemiImplicit` translates contacts into a force route.
- `Featherstone` is articulation-native joint-space solve.
- `MuJoCo` is an external backend bridge.
- `Kamino` is the direct continuation of chapter 07 contact math.
- Do not imply all solvers consume contacts the same way.

- [ ] **Step 2: Generate remaining Chapter 09 images**

Use the same imagegen prompt frame from Task 2 and generate every Chapter 09 asset listed in Image Inventory except `09_principle_prediction_correction_loop.png`, which was already generated during calibration.

Key content constraints:

- The shared problem is hanging cloth prediction and correction.
- `XPBD` is per-constraint projection.
- `VBD` is per-vertex / per-block local solve.
- `Style3D` is fixed PD structure plus nonlinear iteration and PCG.
- `SemiImplicit` remains a baseline in this chapter.

- [ ] **Step 3: Verify Chapter 08 and 09 PNG counts**

Run:

```bash
printf '08 assets '; find chapters/08_rigid_solvers/assets -maxdepth 1 -type f -name '08_*.png' | wc -l
printf '09 assets '; find chapters/09_variational_solvers/assets -maxdepth 1 -type f -name '09_*.png' | wc -l
```

Expected:

```text
08 assets 24
09 assets 22
```

## Task 5: Generate Chapter 10, 11, And 12 Images

**Files:**
- Create remaining `10_*.png`, `11_*.png`, and `12_*.png`.

- [ ] **Step 1: Generate remaining Chapter 10 images**

Use the same imagegen prompt frame from Task 2 and generate every Chapter 10 asset listed in Image Inventory except `10_principle_representation_family_map.png`, which was already generated during calibration.

Key content constraints:

- Cloth is a surface particle/triangle family.
- Softbody is a volume particle/tetrahedron family.
- Cable is a rigid capsule chain with joints.
- The chapter is representation-first, not solver-first.

- [ ] **Step 2: Generate remaining Chapter 11 images**

Use the same imagegen prompt frame from Task 2 and generate every Chapter 11 asset listed in Image Inventory except `11_principle_particle_grid_loop.png`, which was already generated during calibration.

Key content constraints:

- MPM particles are persistent material carriers.
- Grid is a per-step workspace.
- The central route is `P2G -> grid solve -> G2P`.
- MPM particles are not chapter 10 mesh vertices.

- [ ] **Step 3: Generate remaining Chapter 12 images**

Use the same imagegen prompt frame from Task 2 and generate every Chapter 12 asset listed in Image Inventory except `12_principle_read_write_backbone.png`, which was already generated during calibration.

Key content constraints:

- Sensors are read-side adapters.
- IK is a write-side adapter.
- Both share FK/state backbone.
- IK writes back `joint_q`, not body transforms directly.
- Contact sensor reads a contacts side ledger.

- [ ] **Step 4: Verify Chapter 10, 11, and 12 PNG counts**

Run:

```bash
printf '10 assets '; find chapters/10_softbody_cloth_cable/assets -maxdepth 1 -type f -name '10_*.png' | wc -l
printf '11 assets '; find chapters/11_mpm/assets -maxdepth 1 -type f -name '11_*.png' | wc -l
printf '12 assets '; find chapters/12_sensors_ik/assets -maxdepth 1 -type f -name '12_*.png' | wc -l
```

Expected:

```text
10 assets 23
11 assets 23
12 assets 25
```

## Task 6: Insert Markdown Image References

**Files:**
- Modify the 25 Markdown files listed in File Structure.

- [ ] **Step 1: Insert README images**

For each chapter README, add three image references after the opening orientation paragraph or nearest matching section:

```markdown
![<chapter readable spine alt>](assets/<prefix>_readme_*_spine.png)
![<chapter file roles alt>](assets/<prefix>_readme_*roles*.png)
![<chapter completion scope alt>](assets/<prefix>_readme_*completion*.png)
```

Chapter 00 uses:

```markdown
![00 前置速查主线](assets/00_readme_prereq_spine.png)
![00 GAMES103 与本章桥接](assets/00_readme_games103_bridge.png)
![00 阅读顺序与预期产出](assets/00_readme_reading_order_outputs.png)
```

- [ ] **Step 2: Insert principle images**

Add each `*_principle_*.png` immediately after the matching `##` heading introduction paragraph. Keep nearby prose authoritative and do not add explanatory claims beyond the alt text.

- [ ] **Step 3: Insert source-walkthrough images**

Add:

- overview image after `## One-Screen Chapter Map`
- beginner image after `## Beginner Path`
- one stage image near each matching `### Stage N`
- object ledger recap near `## Object Ledger` or `## Stop Here`

- [ ] **Step 4: Insert examples images**

Add:

- overview image after the examples page introduction or `## 总表`
- one example image under each main teaching anchor
- self-check image before or under `## 自检`

- [ ] **Step 5: Confirm no deep files changed**

Run:

```bash
git diff -- chapters/01_warp_basics/source-walkthrough-deep.md chapters/08_rigid_solvers/source-walkthrough-deep.md chapters/09_variational_solvers/source-walkthrough-deep.md chapters/10_softbody_cloth_cable/source-walkthrough-deep.md
```

Expected: no output.

## Task 7: Verification And Review

**Files:**
- All new assets and modified Markdown files.

- [ ] **Step 1: Check Markdown image references**

Run:

```bash
python3 - <<'PY'
import pathlib, re, sys
chapters = [
    '00_prerequisites',
    '01_warp_basics',
    '08_rigid_solvers',
    '09_variational_solvers',
    '10_softbody_cloth_cable',
    '11_mpm',
    '12_sensors_ik',
]
missing=[]; count=0
for chapter in chapters:
    for path in pathlib.Path('chapters', chapter).glob('*.md'):
        if path.name == 'source-walkthrough-deep.md':
            continue
        for match in re.finditer(r'!\[[^\]]*\]\(([^)]+)\)', path.read_text()):
            count += 1
            target = path.parent / match.group(1)
            if not target.exists():
                missing.append((str(path), match.group(1)))
if missing:
    print('Missing image refs:')
    for item in missing:
        print(item)
    sys.exit(1)
print(f'all new-scope image refs exist: {count}')
PY
```

Expected: no missing refs; count equals 144.

- [ ] **Step 2: Check PNG file types**

Run:

```bash
file chapters/00_prerequisites/assets/00_*.png \
  chapters/01_warp_basics/assets/01_*.png \
  chapters/08_rigid_solvers/assets/08_*.png \
  chapters/09_variational_solvers/assets/09_*.png \
  chapters/10_softbody_cloth_cable/assets/10_*.png \
  chapters/11_mpm/assets/11_*.png \
  chapters/12_sensors_ik/assets/12_*.png | rg -v 'PNG image data' || true
```

Expected: no output.

- [ ] **Step 3: Check dimensions and tiny files**

Run:

```bash
python3 - <<'PY'
from PIL import Image
from pathlib import Path
issues=[]
for chapter in ['00_prerequisites','01_warp_basics','08_rigid_solvers','09_variational_solvers','10_softbody_cloth_cable','11_mpm','12_sensors_ik']:
    prefix=chapter[:2]
    for p in Path('chapters', chapter, 'assets').glob(f'{prefix}_*.png'):
        with Image.open(p) as im:
            w,h=im.size
        if w < 900 or h < 900:
            issues.append(f'{p}: {w}x{h}')
print('dimension issues: none' if not issues else '\n'.join(issues))
PY

find chapters/00_prerequisites/assets chapters/01_warp_basics/assets chapters/08_rigid_solvers/assets chapters/09_variational_solvers/assets chapters/10_softbody_cloth_cable/assets chapters/11_mpm/assets chapters/12_sensors_ik/assets -type f -name '*.png' -size -200k -print
```

Expected:

```text
dimension issues: none
```

and no small-file output.

- [ ] **Step 4: Check untouched chapters**

Run:

```bash
git diff -- chapters/02_newton_arch chapters/03_math_geometry chapters/04_scene_usd chapters/05_rigid_articulation chapters/06_collision chapters/07_constraints_contacts_math chapters/13_diffsim chapters/14_viewer_integration chapters/15_multiphysics_pipeline chapters/16_self_experiments
```

Expected: no output.

- [ ] **Step 5: Visual/source-truth review**

Review generated assets against:

- `docs/superpowers/specs/chapter-visual-style-guide.md`
- `docs/superpowers/specs/2026-04-25-chapters-00-12-tutorial-infographics-design.md`

Expected:

- 03/04 handout style
- no 02 diorama style
- concrete sketches in non-admin images
- readable Chinese labels
- real chapter terms only
- no fake code/terminal UI
- no source contradiction

Regenerate any failed assets before committing.

## Task 8: Commit, Merge, Push, And Cleanup

**Files:**
- All generated assets, modified Markdown, spec, and plan.

- [ ] **Step 1: Commit final implementation**

Run:

```bash
git add chapters/00_prerequisites chapters/01_warp_basics chapters/08_rigid_solvers chapters/09_variational_solvers chapters/10_softbody_cloth_cable chapters/11_mpm chapters/12_sensors_ik
git commit -m "docs: add chapters 00-12 tutorial infographics"
```

Expected: commit succeeds.

- [ ] **Step 2: Merge to main and push**

Run from the main worktree:

```bash
git fetch origin main
git merge --ff-only ch00-12-tutorial-infographics
git push origin main
```

Expected: push updates `origin/main`.

- [ ] **Step 3: Final status**

Run:

```bash
git status --short --branch
git log --oneline --decorate -5
```

Expected: `main` is synchronized with `origin/main` and includes the final implementation commit.
