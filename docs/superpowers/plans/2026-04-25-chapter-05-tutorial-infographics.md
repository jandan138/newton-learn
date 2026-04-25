# Chapter 05 Tutorial Infographics Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Generate and integrate PNG tutorial infographics across chapter 05 reader-facing pages.

**Architecture:** Keep exact source truth in Markdown and add model-generated PNGs as teaching anchors. Reuse chapters 03 / 04's dense Chinese learning-handout visual system, but make chapter 05's articulation handoff the central visual spine: flat layout, joint-space state, FK, motion subspace, body-space state, and Featherstone spatial buffers. Final assets live under `chapters/05_rigid_articulation/assets/`.

**Tech Stack:** Markdown, PNG assets generated through Codex native image generation, deterministic raster re-rendering for post-review exact-label corrections, shell verification with `rg`, `file`, Python image-reference checks, and `git diff --check`.

**Implementation note:** Adjacent short administrative sections may share one strong nearby image rather than repeating large PNGs immediately after every heading. `source-walkthrough-deep.md` stays intentionally out of scope. If image review finds source-truth drift in generated text, the affected PNG should be replaced by a deterministic raster diagram rather than accepted with incorrect labels.

---

## Files

- Modify: `chapters/05_rigid_articulation/README.md`
- Modify: `chapters/05_rigid_articulation/principle.md`
- Modify: `chapters/05_rigid_articulation/source-walkthrough.md`
- Modify: `chapters/05_rigid_articulation/examples.md`
- Create: `chapters/05_rigid_articulation/assets/05_readme_articulation_spine.png`
- Create: `chapters/05_rigid_articulation/assets/05_readme_file_roles_reading_order.png`
- Create: `chapters/05_rigid_articulation/assets/05_readme_completion_scope_prereq.png`
- Create: `chapters/05_rigid_articulation/assets/05_principle_after_04_fields_start_moving.png`
- Create: `chapters/05_rigid_articulation/assets/05_principle_flat_articulation_layout.png`
- Create: `chapters/05_rigid_articulation/assets/05_principle_joint_vs_body_state.png`
- Create: `chapters/05_rigid_articulation/assets/05_principle_fk_chain_map.png`
- Create: `chapters/05_rigid_articulation/assets/05_principle_motion_subspace_bridge.png`
- Create: `chapters/05_rigid_articulation/assets/05_principle_inertia_spatial_buffers.png`
- Create: `chapters/05_rigid_articulation/assets/05_principle_next_chapters_map.png`
- Create: `chapters/05_rigid_articulation/assets/05_walkthrough_pipeline_overview.png`
- Create: `chapters/05_rigid_articulation/assets/05_walkthrough_beginner_path.png`
- Create: `chapters/05_rigid_articulation/assets/05_walkthrough_stage1_flat_slice_layout.png`
- Create: `chapters/05_rigid_articulation/assets/05_walkthrough_stage2_state_layers.png`
- Create: `chapters/05_rigid_articulation/assets/05_walkthrough_stage3_fk_velocity_bridge.png`
- Create: `chapters/05_rigid_articulation/assets/05_walkthrough_stage4_featherstone_spatial_buffers.png`
- Create: `chapters/05_rigid_articulation/assets/05_walkthrough_stage5_solver_step_handoff.png`
- Create: `chapters/05_rigid_articulation/assets/05_walkthrough_object_ledger_stop_here.png`
- Create: `chapters/05_rigid_articulation/assets/05_examples_overview_observation_tasks.png`
- Create: `chapters/05_rigid_articulation/assets/05_examples_basic_pendulum_joint_frame.png`
- Create: `chapters/05_rigid_articulation/assets/05_examples_basic_joints_type_compare.png`
- Create: `chapters/05_rigid_articulation/assets/05_examples_robot_cartpole_imported_layout.png`
- Modify: `docs/superpowers/plans/2026-04-25-chapter-05-tutorial-infographics.md`

## Task 1: Calibration Image

- [x] **Step 1: Create chapter 05 assets directory**

Run:

```bash
mkdir -p chapters/05_rigid_articulation/assets
```

Expected: `chapters/05_rigid_articulation/assets/` exists.

- [x] **Step 2: Generate the calibration PNG**

Run Codex native image generation for:

`chapters/05_rigid_articulation/assets/05_principle_flat_articulation_layout.png`

Prompt content:

```text
Create a clean Chinese tutorial infographic for Newton Learn Chapter 05: 刚体与关节动力学.
Style: white background, thin blue rounded card borders, dark-blue title, numbered blue badges, compact multi-panel teacher review-sheet layout, small friendly icons, arrows, tree-to-array map, state-layer cards, high information density but readable.
Content: show a minimal two-link articulation on the left: parent body -> joint -> child body. On the right show the flat layout bridge:
1 joint_parent / joint_child are the tree edge
2 joint_X_p / joint_X_c are the two frame anchors
3 articulation_start marks one joint range
4 joint_q_start / joint_qd_start mark slices in generalized arrays
5 joint_q / joint_qd feed FK
6 body_q / body_qd are body-space results
Use Chinese-first labels. English/code terms allowed only for exact names: articulation_start, joint_parent, joint_child, joint_X_p/X_c, joint_q_start, joint_qd_start, joint_q, joint_qd, body_q, body_qd, eval_fk().
Bottom recap: articulation 先是一段可切片的 joint tree，再把 joint-space 翻成 body-space.
Do not invent source code. Do not include long paragraphs. Do not use SVG, HTML, screenshot, terminal UI, photorealistic robot rendering, or full ABA/CRBA derivation. Generate raster PNG.
```

- [x] **Step 3: Inspect calibration**

Run:

```bash
file chapters/05_rigid_articulation/assets/05_principle_flat_articulation_layout.png
```

Expected: `PNG image data`.

Open the image visually and check it matches `docs/superpowers/specs/chapter-visual-style-guide.md`: white handout, blue cards, numbered badges, readable Chinese labels, not decorative robot art, and clear flat-layout / state-layer / FK boundaries.

## Task 2: README Image Batch

- [x] **Step 1: Generate README support images**

Generate:

- `05_readme_articulation_spine.png`: chapter identity; chapter 03 / 04 fields now start moving through articulation.
- `05_readme_file_roles_reading_order.png`: file roles and reading order: `README.md -> principle.md -> source-walkthrough.md -> source-walkthrough-deep.md -> examples.md`.
- `05_readme_completion_scope_prereq.png`: completion gate, scope, non-scope, prerequisites, GAMES103 bridge, and expected output.

- [x] **Step 2: Integrate README images**

Insert:

- after the opening chapter overview: `05_readme_articulation_spine.png`
- after `## 文件分工`: `05_readme_file_roles_reading_order.png`
- after `## 完成门槛`: `05_readme_completion_scope_prereq.png`

Coverage note:

- `## 本章目标`, `## 本章范围`, `## 本章明确不做什么`, `## 前置依赖`, `## GAMES103 已有 vs 本章新增`, `## 阅读顺序`, and `## 预期产出` are covered by the nearby README images without repeated immediate insertion.

## Task 3: Principle Image Batch

- [x] **Step 1: Generate principle concept images**

Generate:

- `05_principle_after_04_fields_start_moving.png`
- `05_principle_joint_vs_body_state.png`
- `05_principle_fk_chain_map.png`
- `05_principle_motion_subspace_bridge.png`
- `05_principle_inertia_spatial_buffers.png`
- `05_principle_next_chapters_map.png`

`05_principle_flat_articulation_layout.png` already exists from Task 1.

- [x] **Step 2: Integrate principle images**

Insert:

- after `## 0`: `05_principle_after_04_fields_start_moving.png`
- after `## 1`: `05_principle_flat_articulation_layout.png`
- after `## 2`: `05_principle_joint_vs_body_state.png`
- after `## 3`: `05_principle_fk_chain_map.png`
- after `## 4`: `05_principle_motion_subspace_bridge.png`
- after `## 5`: `05_principle_inertia_spatial_buffers.png`
- after `## 6`: `05_principle_next_chapters_map.png`

## Task 4: Source Walkthrough Image Batch

- [x] **Step 1: Generate source-walkthrough images**

Generate:

- `05_walkthrough_pipeline_overview.png`
- `05_walkthrough_beginner_path.png`
- `05_walkthrough_stage1_flat_slice_layout.png`
- `05_walkthrough_stage2_state_layers.png`
- `05_walkthrough_stage3_fk_velocity_bridge.png`
- `05_walkthrough_stage4_featherstone_spatial_buffers.png`
- `05_walkthrough_stage5_solver_step_handoff.png`
- `05_walkthrough_object_ledger_stop_here.png`

- [x] **Step 2: Integrate source-walkthrough images**

Insert:

- after `## What This Walkthrough Follows`: `05_walkthrough_pipeline_overview.png`
- `## One-Screen Chapter Map`: covered by the adjacent pipeline overview and existing text map.
- after `## Beginner Path`: `05_walkthrough_beginner_path.png`
- after `### Stage 1`: `05_walkthrough_stage1_flat_slice_layout.png`
- after `### Stage 2`: `05_walkthrough_stage2_state_layers.png`
- after `### Stage 3`: `05_walkthrough_stage3_fk_velocity_bridge.png`
- after `### Stage 4`: `05_walkthrough_stage4_featherstone_spatial_buffers.png`
- after `### Stage 5`: `05_walkthrough_stage5_solver_step_handoff.png`
- after `## Object Ledger`: `05_walkthrough_object_ledger_stop_here.png`
- `## Stop Here` and `## Go Deeper`: covered by the adjacent object-ledger / stop-here image and text navigation without extra insertion.

## Task 5: Examples Image Batch

- [x] **Step 1: Generate examples images**

Generate:

- `05_examples_overview_observation_tasks.png`
- `05_examples_basic_pendulum_joint_frame.png`
- `05_examples_basic_joints_type_compare.png`
- `05_examples_robot_cartpole_imported_layout.png`

- [x] **Step 2: Integrate examples images**

Insert:

- after the opening examples overview: `05_examples_overview_observation_tasks.png`
- after `## 主例子：`basic_pendulum``: `05_examples_basic_pendulum_joint_frame.png`
- after `## 对照例子：`basic_joints``: `05_examples_basic_joints_type_compare.png`
- after `## 对照例子：`robot_cartpole``: `05_examples_robot_cartpole_imported_layout.png`
- `## 这页怎么配合其他文件`: covered by the examples overview and closing prose without extra insertion.

## Task 6: Verification, Review, Commit, Push

- [x] **Step 1: Verify Markdown image references**

Run:

```bash
python3 - <<'PY'
import pathlib, re, sys
files = [
    pathlib.Path('chapters/05_rigid_articulation/README.md'),
    pathlib.Path('chapters/05_rigid_articulation/principle.md'),
    pathlib.Path('chapters/05_rigid_articulation/source-walkthrough.md'),
    pathlib.Path('chapters/05_rigid_articulation/examples.md'),
]
missing = []
for path in files:
    text = path.read_text()
    for match in re.finditer(r'!\[[^\]]*\]\(([^)]+)\)', text):
        target = (path.parent / match.group(1)).resolve()
        if not target.exists():
            missing.append((str(path), match.group(1)))
if missing:
    print('Missing image refs:')
    for item in missing:
        print(item)
    sys.exit(1)
print('all image refs exist')
PY
```

Expected: `all image refs exist`.

- [x] **Step 2: Verify PNG assets**

Run:

```bash
file chapters/05_rigid_articulation/assets/05_*.png
```

Expected: every line says `PNG image data`.

- [x] **Step 3: Verify Markdown diff cleanliness**

Run:

```bash
git diff --check -- chapters/05_rigid_articulation docs/superpowers/specs/2026-04-25-chapter-05-tutorial-infographics-design.md docs/superpowers/plans/2026-04-25-chapter-05-tutorial-infographics.md
```

Expected: no output.

- [x] **Step 4: Coverage scan**

Run:

```bash
rg -n '^##|^###|!\[' chapters/05_rigid_articulation/README.md chapters/05_rigid_articulation/principle.md chapters/05_rigid_articulation/source-walkthrough.md chapters/05_rigid_articulation/examples.md
```

Expected: each reader-facing section has a nearby image anchor, with documented reuse for short/admin sections.

- [x] **Step 5: Visual style review**

Create or inspect a contact sheet for all `05_*.png` files and compare against `docs/superpowers/specs/chapter-visual-style-guide.md` and chapter 04 assets.

Expected:

- white Chinese tutorial-handout style
- blue card boundaries and numbered badges
- no blank images
- no decorative robot render drift
- no fake code / fake terminal UI / invented APIs
- no `joint_qd == body_qd` visual claim
- source-truth shortcuts remain bounded by nearby Markdown

- [x] **Step 6: Commit**

Run:

```bash
git add chapters/05_rigid_articulation docs/superpowers/specs/2026-04-25-chapter-05-tutorial-infographics-design.md docs/superpowers/plans/2026-04-25-chapter-05-tutorial-infographics.md
git commit -m "docs: add chapter 05 tutorial infographics"
```

- [x] **Step 7: Merge to main and push**

Run:

```bash
git fetch origin main
git checkout main
git merge --ff-only ch05-tutorial-infographics
git push origin main
```

Expected: `main` fast-forwards and push succeeds.
