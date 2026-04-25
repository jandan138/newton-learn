# Chapters 05-07 Native Imagegen Redo Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace chapters 05, 06, and 07's current deterministic tutorial PNGs with native-imagegen raster infographics that follow the documented chapter 03 / 04 Chinese tutorial-handout style.

**Architecture:** Preserve all existing Markdown image filenames and replace PNG content in place. Start with one calibration image per chapter, review against `chapter-visual-style-guide.md` plus chapter 03 / 04 specs, then regenerate the remaining assets in chapter batches and run path, PNG, visual, and source-truth checks.

**Tech Stack:** Markdown, PNG assets generated through Codex native image generation / built-in imagegen, shell verification with `rg`, `file`, Python image-reference checks, and `git diff --check`.

---

## Files

- Create: `docs/superpowers/specs/2026-04-25-chapters-05-07-native-imagegen-redo-design.md`
- Create/modify: `docs/superpowers/plans/2026-04-25-chapters-05-07-native-imagegen-redo.md`
- Replace in place: `chapters/05_rigid_articulation/assets/05_*.png`
- Replace in place: `chapters/06_collision/assets/06_*.png`
- Replace in place: `chapters/07_constraints_contacts_math/assets/07_*.png`
- Verify only: `chapters/05_rigid_articulation/README.md`
- Verify only: `chapters/05_rigid_articulation/principle.md`
- Verify only: `chapters/05_rigid_articulation/source-walkthrough.md`
- Verify only: `chapters/05_rigid_articulation/examples.md`
- Verify only: `chapters/06_collision/README.md`
- Verify only: `chapters/06_collision/principle.md`
- Verify only: `chapters/06_collision/source-walkthrough.md`
- Verify only: `chapters/06_collision/examples.md`
- Verify only: `chapters/07_constraints_contacts_math/README.md`
- Verify only: `chapters/07_constraints_contacts_math/principle.md`
- Verify only: `chapters/07_constraints_contacts_math/source-walkthrough.md`
- Verify only: `chapters/07_constraints_contacts_math/examples.md`

## Native Imagegen Prompt Contract

Every prompt must include this shared contract:

```text
Use the imagegen skill and the built-in image generation tool only.
Do not use CLI fallback, do not ask for OPENAI_API_KEY, and do not create SVG/HTML.

Create a clean Chinese tutorial infographic for Newton Learn, matching chapters 03 and 04.
Style: white background, thin blue rounded card borders, dark-blue title, numbered blue badges, compact multi-panel teacher review-sheet layout, small friendly schematic icons, arrows, flow boxes, role ledgers, comparison strips, and bottom recap strip. High information density but readable.
Important: include concrete topic sketches inside the handout cards, not only abstract text cards.
Language: Chinese-first. English/code terms only for exact names from the chapter. Keep labels short and readable.
Avoid: Chapter 02 diorama style, photorealistic 3D rendering, fake code, fake terminal UI, invented API names, long paragraphs, tiny labels, SVG, HTML, and local deterministic rendering.
Generate a raster PNG and save it to the requested workspace path.
```

## Task 1: Confirm Style Contract And Native Imagegen Availability

**Files:**
- Verify: `docs/superpowers/specs/chapter-visual-style-guide.md`
- Verify: `docs/superpowers/specs/2026-04-24-chapter-03-tutorial-infographics-design.md`
- Verify: `docs/superpowers/specs/2026-04-25-chapter-04-tutorial-infographics-design.md`
- Verify: `docs/superpowers/specs/2026-04-25-chapters-05-07-native-imagegen-redo-design.md`

- [ ] **Step 1: Verify native image generation is available**

Run:

```bash
codex features list | rg '^image_generation\\s+stable\\s+true'
```

Expected: one matching line for `image_generation stable true`.

- [ ] **Step 2: Confirm the style source excludes chapter 02 native-imagegen design**

Run:

```bash
rg -n 'Chapter 02 does not need revision|not the earlier 3D-diorama|Reuse the chapter 03 visual system|chapter-02-native-imagegen-design.md' \
  docs/superpowers/specs/chapter-visual-style-guide.md \
  docs/superpowers/specs/2026-04-24-chapter-03-tutorial-infographics-design.md \
  docs/superpowers/specs/2026-04-25-chapter-04-tutorial-infographics-design.md \
  docs/superpowers/specs/2026-04-25-chapters-05-07-native-imagegen-redo-design.md
```

Expected: matches proving the redo follows 03/04 and rejects the 02 diorama direction.

## Task 2: Generate And Review Calibration Images

**Files:**
- Replace: `chapters/05_rigid_articulation/assets/05_principle_flat_articulation_layout.png`
- Replace: `chapters/06_collision/assets/06_collision_bridge_map.png`
- Replace: `chapters/07_constraints_contacts_math/assets/07_contact_math_bridge_map.png`

- [ ] **Step 1: Generate chapter 05 calibration**

Generate `chapters/05_rigid_articulation/assets/05_principle_flat_articulation_layout.png`.

Prompt-specific content:

```text
Chapter 05 topic: flat articulation layout.
Concrete sketch: a simple two-link articulated arm with parent/child joint connection on the left; on the right, flat array slices for articulation_start, joint_parent, joint_child, joint_q_start, joint_qd_start; arrows from tree sketch to array slices; small state-layer cards for joint_q/joint_qd -> body_q/body_qd.
Bottom recap: 关节树先被压平成数组，FK 再把 joint state 推到 body state.
Allowed exact terms: articulation_start, joint_parent, joint_child, joint_q_start, joint_qd_start, joint_q, joint_qd, body_q, body_qd, eval_fk().
```

- [ ] **Step 2: Generate chapter 06 calibration**

Generate `chapters/06_collision/assets/06_collision_bridge_map.png`.

Prompt-specific content:

```text
Chapter 06 topic: collision bridge from body/world state and shape metadata to Contacts.
Concrete sketch: moving body pose and local shape on the left, shape world pose and AABB in the middle, broad-phase maybe pair cards, narrow-phase contact point/normal sketch, ContactData packet, and final Contacts buffer on the right.
Bottom recap: body_q 给运动位置，shape_* 给几何输入，collision 最后只交出 Contacts.
Allowed exact terms: body_q, shape_transform, shape_type, AABB, broad phase, narrow phase, ContactData, write_contact(), Contacts.
```

- [ ] **Step 3: Generate chapter 07 calibration**

Generate `chapters/07_constraints_contacts_math/assets/07_contact_math_bridge_map.png`.

Prompt-specific content:

```text
Chapter 07 topic: contact math bridge.
Concrete sketch: sphere resting on ground with one contact point, normal arrow and two tangent arrows; expand into one normal row plus two tangent rows; show arrows into J, D/effective mass, and chapter 08 solver handoff.
Bottom recap: Contacts 给几何，07 把它翻译成 solver 能吃的 rows / J / D.
Allowed exact terms: Contacts, contact frame, normal, tangent, rows, J, D, effective mass, 08 solver.
```

- [ ] **Step 4: Verify calibration PNG files**

Run:

```bash
file \
  chapters/05_rigid_articulation/assets/05_principle_flat_articulation_layout.png \
  chapters/06_collision/assets/06_collision_bridge_map.png \
  chapters/07_constraints_contacts_math/assets/07_contact_math_bridge_map.png
```

Expected: every line says `PNG image data`.

- [ ] **Step 5: Visual calibration review**

Open the three calibration images and reject any image that is text-only, decorative, unreadable, or drifting toward Chapter 02 diorama / photorealistic 3D style.

Expected:

- white Chinese tutorial-handout style
- blue cards and numbered badges
- concrete topic sketch in each image
- readable short Chinese labels
- no fake code or terminal text

## Task 3: Replace Chapter 05 Assets

**Files:**
- Replace in place: `chapters/05_rigid_articulation/assets/05_*.png`

- [ ] **Step 1: Regenerate README assets**

Generate native-imagegen replacements for:

```text
05_readme_articulation_spine.png
05_readme_file_roles_reading_order.png
05_readme_completion_scope_prereq.png
```

Prompt focus:

```text
Chapter 05 README navigation. Use concrete small sketches: two-link arm, flat arrays, file-role icons, completion checklist, and handoff from 04 scene fields to 05 articulation.
```

- [ ] **Step 2: Regenerate principle assets**

Generate native-imagegen replacements for:

```text
05_principle_after_04_fields_start_moving.png
05_principle_joint_vs_body_state.png
05_principle_fk_chain_map.png
05_principle_motion_subspace_bridge.png
05_principle_inertia_spatial_buffers.png
05_principle_next_chapters_map.png
```

`05_principle_flat_articulation_layout.png` is already regenerated by Task 2.

Prompt focus:

```text
Chapter 05 principle concepts. Use concrete sketches for two-link arm, joint-space vs body-space cards, FK transform arrows, motion subspace velocity arrow, spatial inertia buffer blocks, and next-chapter handoff.
```

- [ ] **Step 3: Regenerate source-walkthrough assets**

Generate native-imagegen replacements for:

```text
05_walkthrough_pipeline_overview.png
05_walkthrough_beginner_path.png
05_walkthrough_stage1_flat_slice_layout.png
05_walkthrough_stage2_state_layers.png
05_walkthrough_stage3_fk_velocity_bridge.png
05_walkthrough_stage4_featherstone_spatial_buffers.png
05_walkthrough_stage5_solver_step_handoff.png
05_walkthrough_object_ledger_stop_here.png
```

Prompt focus:

```text
Chapter 05 walkthrough path. Use concrete sketches for each stage: flat slice ledger, state layers, FK bridge, Featherstone spatial buffers, solver handoff, and stop-here recap.
```

- [ ] **Step 4: Regenerate example assets**

Generate native-imagegen replacements for:

```text
05_examples_overview_observation_tasks.png
05_examples_basic_pendulum_joint_frame.png
05_examples_basic_joints_type_compare.png
05_examples_robot_cartpole_imported_layout.png
```

Prompt focus:

```text
Chapter 05 examples. Use concrete teaching sketches for basic_pendulum, joint frame, joint type comparison, and robot_cartpole imported articulation layout.
```

- [ ] **Step 5: Verify chapter 05**

Run:

```bash
python3 - <<'PY'
import pathlib, re, sys
files = [pathlib.Path(p) for p in [
    'chapters/05_rigid_articulation/README.md',
    'chapters/05_rigid_articulation/principle.md',
    'chapters/05_rigid_articulation/source-walkthrough.md',
    'chapters/05_rigid_articulation/examples.md',
]]
missing = []
for path in files:
    for match in re.finditer(r'!\\[[^\\]]*\\]\\(([^)]+)\\)', path.read_text()):
        target = path.parent / match.group(1)
        if not target.exists():
            missing.append((str(path), match.group(1)))
if missing:
    print(missing)
    sys.exit(1)
print('chapter 05 image refs exist')
PY
file chapters/05_rigid_articulation/assets/05_*.png | rg -v 'PNG image data' || true
```

Expected: `chapter 05 image refs exist`; the `file | rg -v` command prints no PNG mismatches.

## Task 4: Replace Chapter 06 Assets

**Files:**
- Replace in place: `chapters/06_collision/assets/06_*.png`

- [ ] **Step 1: Regenerate README assets**

Generate native-imagegen replacements for:

```text
06_readme_collision_spine.png
06_readme_file_roles_reading_order.png
06_readme_completion_scope_prereq.png
```

Prompt focus:

```text
Chapter 06 README navigation. Use concrete sketches for body pose, shape metadata, collision pipeline, Contacts handoff, file roles, and completion checklist.
```

- [ ] **Step 2: Regenerate principle assets**

Generate native-imagegen replacements for:

```text
06_principle_shape_not_body_boundary.png
06_principle_broad_phase_maybe_list.png
06_principle_narrow_phase_contact_geometry.png
06_principle_shape_type_routing.png
06_principle_contacts_handoff_buffer.png
06_principle_next_chapters_map.png
```

`06_collision_bridge_map.png` is already regenerated by Task 2.

Prompt focus:

```text
Chapter 06 principle concepts. Use concrete sketches for body-vs-shape boundary, AABB and maybe pairs, contact point/normal, shape_type routing, ContactData to Contacts buffer, and 07/08 handoff.
```

- [ ] **Step 3: Regenerate source-walkthrough assets**

Generate native-imagegen replacements for:

```text
06_walkthrough_pipeline_overview.png
06_walkthrough_beginner_path.png
06_walkthrough_stage1_model_state_shape_data.png
06_walkthrough_stage2_compute_shape_aabbs.png
06_walkthrough_stage3_broad_phase_candidate_pairs.png
06_walkthrough_stage4_narrow_phase_contactdata.png
06_walkthrough_stage5_write_contact_contacts.png
06_walkthrough_object_ledger_stop_here.png
```

Prompt focus:

```text
Chapter 06 walkthrough path. Use concrete sketches for model/state/shape handoff, AABB generation, candidate pairs, narrow-phase ContactData, write_contact(), Contacts, and object ledger.
```

- [ ] **Step 4: Regenerate example assets**

Generate native-imagegen replacements for:

```text
06_examples_overview_observation_tasks.png
06_examples_basic_shapes_sphere_ground.png
06_examples_pyramid_contact_set_growth.png
06_examples_contact_fields_watchlist.png
```

Prompt focus:

```text
Chapter 06 examples. Use concrete teaching sketches for sphere-ground contact, pyramid contact-set growth, and field watchlist for Contacts.
```

- [ ] **Step 5: Verify chapter 06**

Run:

```bash
python3 - <<'PY'
import pathlib, re, sys
files = [pathlib.Path(p) for p in [
    'chapters/06_collision/README.md',
    'chapters/06_collision/principle.md',
    'chapters/06_collision/source-walkthrough.md',
    'chapters/06_collision/examples.md',
]]
missing = []
for path in files:
    for match in re.finditer(r'!\\[[^\\]]*\\]\\(([^)]+)\\)', path.read_text()):
        target = path.parent / match.group(1)
        if not target.exists():
            missing.append((str(path), match.group(1)))
if missing:
    print(missing)
    sys.exit(1)
print('chapter 06 image refs exist')
PY
file chapters/06_collision/assets/06_*.png | rg -v 'PNG image data' || true
```

Expected: `chapter 06 image refs exist`; the `file | rg -v` command prints no PNG mismatches.

## Task 5: Replace Chapter 07 Assets

**Files:**
- Replace in place: `chapters/07_constraints_contacts_math/assets/07_*.png`

- [ ] **Step 1: Regenerate README assets**

Generate native-imagegen replacements for:

```text
07_readme_contact_math_spine.png
07_readme_file_roles_reading_order.png
07_readme_completion_scope_prereq.png
```

Prompt focus:

```text
Chapter 07 README navigation. Use concrete sketches for Contacts, contact frame, rows, J, D, chapter 08 handoff, file roles, and completion checklist.
```

- [ ] **Step 2: Regenerate principle assets**

Generate native-imagegen replacements for:

```text
07_principle_contact_to_three_rows.png
07_principle_jacobian_motion_map.png
07_principle_lever_arm_angular_coupling.png
07_principle_delassus_effective_mass.png
07_principle_box_ground_multi_contacts.png
07_principle_solver_facing_next_chapter.png
```

`07_contact_math_bridge_map.png` is already regenerated by Task 2.

Prompt focus:

```text
Chapter 07 principle concepts. Use concrete sketches for sphere-ground contact frame, normal/tangent row expansion, J velocity map, lever-arm torque arrow, Delassus/effective-mass response, multi-contact box support, and chapter 08 handoff.
```

- [ ] **Step 3: Regenerate source-walkthrough assets**

Generate native-imagegen replacements for:

```text
07_walkthrough_pipeline_overview.png
07_walkthrough_beginner_path.png
07_walkthrough_stage1_contacts_geometry_handoff.png
07_walkthrough_stage2_solver_facing_contact_block.png
07_walkthrough_stage3_contact_rows_three_directions.png
07_walkthrough_stage4_jacobian_frame_lever_arm.png
07_walkthrough_stage5_delassus_row_space_response.png
07_walkthrough_object_ledger_stop_here.png
```

Prompt focus:

```text
Chapter 07 walkthrough path. Use concrete sketches for Contacts geometry handoff, solver-facing contact block, contact rows, Jacobian frame and lever arm, Delassus row-space response, and object ledger.
```

- [ ] **Step 4: Regenerate example assets**

Generate native-imagegen replacements for:

```text
07_examples_overview_observation_tasks.png
07_examples_sphere_ground_one_contact_three_rows.png
07_examples_box_ground_multiple_contacts_lever_arm.png
07_examples_self_check_handoff.png
```

Prompt focus:

```text
Chapter 07 examples. Use concrete teaching sketches for sphere-ground one contact expanding to rows, box-ground multiple contacts with lever arms, and self-check handoff.
```

- [ ] **Step 5: Verify chapter 07**

Run:

```bash
python3 - <<'PY'
import pathlib, re, sys
files = [pathlib.Path(p) for p in [
    'chapters/07_constraints_contacts_math/README.md',
    'chapters/07_constraints_contacts_math/principle.md',
    'chapters/07_constraints_contacts_math/source-walkthrough.md',
    'chapters/07_constraints_contacts_math/examples.md',
]]
missing = []
for path in files:
    for match in re.finditer(r'!\\[[^\\]]*\\]\\(([^)]+)\\)', path.read_text()):
        target = path.parent / match.group(1)
        if not target.exists():
            missing.append((str(path), match.group(1)))
if missing:
    print(missing)
    sys.exit(1)
print('chapter 07 image refs exist')
PY
file chapters/07_constraints_contacts_math/assets/07_*.png | rg -v 'PNG image data' || true
```

Expected: `chapter 07 image refs exist`; the `file | rg -v` command prints no PNG mismatches.

## Task 6: Review, Diff Hygiene, Commit, Push

**Files:**
- Verify: `chapters/05_rigid_articulation/`
- Verify: `chapters/06_collision/`
- Verify: `chapters/07_constraints_contacts_math/`
- Verify: `docs/superpowers/specs/2026-04-25-chapters-05-07-native-imagegen-redo-design.md`
- Verify: `docs/superpowers/plans/2026-04-25-chapters-05-07-native-imagegen-redo.md`

- [ ] **Step 1: Verify deep walkthrough files are untouched**

Run:

```bash
git diff -- chapters/05_rigid_articulation/source-walkthrough-deep.md chapters/06_collision/source-walkthrough-deep.md chapters/07_constraints_contacts_math/source-walkthrough-deep.md
```

Expected: no output.

- [ ] **Step 2: Verify all image refs in 05/06/07**

Run:

```bash
python3 - <<'PY'
import pathlib, re, sys
files = [pathlib.Path(p) for p in [
    'chapters/05_rigid_articulation/README.md',
    'chapters/05_rigid_articulation/principle.md',
    'chapters/05_rigid_articulation/source-walkthrough.md',
    'chapters/05_rigid_articulation/examples.md',
    'chapters/06_collision/README.md',
    'chapters/06_collision/principle.md',
    'chapters/06_collision/source-walkthrough.md',
    'chapters/06_collision/examples.md',
    'chapters/07_constraints_contacts_math/README.md',
    'chapters/07_constraints_contacts_math/principle.md',
    'chapters/07_constraints_contacts_math/source-walkthrough.md',
    'chapters/07_constraints_contacts_math/examples.md',
]]
missing = []
count = 0
for path in files:
    for match in re.finditer(r'!\\[[^\\]]*\\]\\(([^)]+)\\)', path.read_text()):
        count += 1
        target = path.parent / match.group(1)
        if not target.exists():
            missing.append((str(path), match.group(1)))
if missing:
    print('Missing image refs:')
    for item in missing:
        print(item)
    sys.exit(1)
print(f'all image refs exist: {count}')
PY
```

Expected: all image references exist.

- [ ] **Step 3: Verify PNG formats**

Run:

```bash
file chapters/05_rigid_articulation/assets/05_*.png chapters/06_collision/assets/06_*.png chapters/07_constraints_contacts_math/assets/07_*.png | rg -v 'PNG image data' || true
```

Expected: no PNG mismatch output.

- [ ] **Step 4: Run Markdown diff hygiene check**

Run:

```bash
git diff --check -- \
  chapters/05_rigid_articulation \
  chapters/06_collision \
  chapters/07_constraints_contacts_math \
  docs/superpowers/specs/2026-04-25-chapters-05-07-native-imagegen-redo-design.md \
  docs/superpowers/plans/2026-04-25-chapters-05-07-native-imagegen-redo.md
```

Expected: no output.

- [ ] **Step 5: Request review**

Ask a reviewer agent to inspect:

```text
Review the chapters 05-07 native-imagegen redo. Focus on whether the images follow chapter 03/04 tutorial-handout style, contain concrete topic sketches, preserve source-truth boundaries, avoid fake code/text, and keep Markdown references valid. Report only blocking or concrete issues.
```

Expected: no blocking findings, or concrete findings that can be regenerated/fixed.

- [ ] **Step 6: Commit**

Run:

```bash
git add \
  chapters/05_rigid_articulation/assets \
  chapters/06_collision/assets \
  chapters/07_constraints_contacts_math/assets \
  docs/superpowers/specs/2026-04-25-chapters-05-07-native-imagegen-redo-design.md \
  docs/superpowers/plans/2026-04-25-chapters-05-07-native-imagegen-redo.md
git commit -m "docs: redo chapters 05-07 tutorial infographics"
```

- [ ] **Step 7: Merge and push**

Run from the main worktree:

```bash
git fetch origin main
git merge --ff-only ch05-07-native-imagegen-redo
git push origin main
```

Expected: `main` fast-forwards and push succeeds.
