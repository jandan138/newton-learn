# Chapter 07 Tutorial Infographics Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [x]`) syntax for tracking.

**Goal:** Generate and integrate PNG tutorial infographics across chapter 07 reader-facing pages.

**Architecture:** Keep exact source truth in Markdown and add deterministic PNG teaching anchors as visual summaries. Reuse chapter 05 / 06's dense Chinese learning-handout visual system, but make chapter 07's contact-math handoff the central visual spine: runtime `Contacts`, solver-facing `ContactsKamino`, contact frame, `1 normal + 2 tangents`, row expansion, Jacobian velocity mapping, lever-arm angular coupling, Delassus / effective mass, and chapter 08 handoff. Final assets live under `chapters/07_constraints_contacts_math/assets/`.

**Tech Stack:** Markdown, deterministic raster PNG rendering with Python/Pillow, optional temporary CJK font download into ignored `tmp/`, shell verification with `rg`, `file`, Python image-reference checks, and `git diff --check`.

**Implementation note:** Adjacent short administrative sections may share one strong nearby image rather than repeating large PNGs immediately after every heading. `source-walkthrough-deep.md` stays intentionally out of scope. If review finds source-truth drift in generated text, the affected PNG should be re-rendered deterministically rather than accepted with incorrect labels.

---

## Files

- Modify: `chapters/07_constraints_contacts_math/README.md`
- Modify: `chapters/07_constraints_contacts_math/principle.md`
- Modify: `chapters/07_constraints_contacts_math/source-walkthrough.md`
- Modify: `chapters/07_constraints_contacts_math/examples.md`
- Create: `chapters/07_constraints_contacts_math/assets/07_readme_contact_math_spine.png`
- Create: `chapters/07_constraints_contacts_math/assets/07_readme_file_roles_reading_order.png`
- Create: `chapters/07_constraints_contacts_math/assets/07_readme_completion_scope_prereq.png`
- Create: `chapters/07_constraints_contacts_math/assets/07_contact_math_bridge_map.png`
- Create: `chapters/07_constraints_contacts_math/assets/07_principle_contact_to_three_rows.png`
- Create: `chapters/07_constraints_contacts_math/assets/07_principle_jacobian_motion_map.png`
- Create: `chapters/07_constraints_contacts_math/assets/07_principle_lever_arm_angular_coupling.png`
- Create: `chapters/07_constraints_contacts_math/assets/07_principle_delassus_effective_mass.png`
- Create: `chapters/07_constraints_contacts_math/assets/07_principle_box_ground_multi_contacts.png`
- Create: `chapters/07_constraints_contacts_math/assets/07_principle_solver_facing_next_chapter.png`
- Create: `chapters/07_constraints_contacts_math/assets/07_walkthrough_pipeline_overview.png`
- Create: `chapters/07_constraints_contacts_math/assets/07_walkthrough_beginner_path.png`
- Create: `chapters/07_constraints_contacts_math/assets/07_walkthrough_stage1_contacts_geometry_handoff.png`
- Create: `chapters/07_constraints_contacts_math/assets/07_walkthrough_stage2_solver_facing_contact_block.png`
- Create: `chapters/07_constraints_contacts_math/assets/07_walkthrough_stage3_contact_rows_three_directions.png`
- Create: `chapters/07_constraints_contacts_math/assets/07_walkthrough_stage4_jacobian_frame_lever_arm.png`
- Create: `chapters/07_constraints_contacts_math/assets/07_walkthrough_stage5_delassus_row_space_response.png`
- Create: `chapters/07_constraints_contacts_math/assets/07_walkthrough_object_ledger_stop_here.png`
- Create: `chapters/07_constraints_contacts_math/assets/07_examples_overview_observation_tasks.png`
- Create: `chapters/07_constraints_contacts_math/assets/07_examples_sphere_ground_one_contact_three_rows.png`
- Create: `chapters/07_constraints_contacts_math/assets/07_examples_box_ground_multiple_contacts_lever_arm.png`
- Create: `chapters/07_constraints_contacts_math/assets/07_examples_self_check_handoff.png`
- Modify: `docs/superpowers/plans/2026-04-25-chapter-07-tutorial-infographics.md`

Temporary ignored implementation files:

- Create/use: `tmp/ch07-imagegen/render_infographics.py`
- Optional create/use: `tmp/ch07-imagegen/fonts/NotoSansCJKsc-Regular.otf`

## Task 1: Calibration Renderer and Image

- [ ] **Step 1: Prepare ignored renderer workspace**

Run:

```bash
mkdir -p tmp/ch07-imagegen/fonts chapters/07_constraints_contacts_math/assets
```

Expected: both directories exist. `tmp/` is ignored by `.gitignore`, so the renderer does not become part of the final commit.

- [ ] **Step 2: Ensure a readable CJK font is available**

Run:

```bash
python3 - <<'PY'
from pathlib import Path
paths = [
    Path('/usr/share/fonts/opentype/noto/NotoSansCJK-Regular.ttc'),
    Path('/usr/share/fonts/opentype/noto/NotoSansCJKsc-Regular.otf'),
    Path('/usr/share/fonts/truetype/noto/NotoSansCJK-Regular.ttc'),
    Path('tmp/ch07-imagegen/fonts/NotoSansCJKsc-Regular.otf'),
]
print(next((str(p) for p in paths if p.exists()), 'missing'))
PY
```

Expected: a font path or `missing`. If it prints `missing`, download `NotoSansCJKsc-Regular.otf` into `tmp/ch07-imagegen/fonts/` and re-run the probe. Do not commit the font.

- [ ] **Step 3: Create the deterministic renderer**

Write `tmp/ch07-imagegen/render_infographics.py`.

Renderer requirements:

- canvas: `1680x945`
- background: white
- title: dark blue, top centered
- cards: thin blue rounded borders, light blue header strips where useful
- badges: numbered blue circles
- text: Chinese-first, short, readable
- icons: simple schematic shapes only, drawn with Pillow primitives
- recap: bottom strip with one memory hook
- output: exactly the 22 PNG filenames listed in this plan

Calibration content for `07_contact_math_bridge_map.png`:

```text
Contacts geometry handoff
-> solver-facing contact block
-> contact frame: normal + two tangents
-> 1 contact -> 3 rows
-> Jacobian maps qd to row velocity
-> Delassus / effective mass
-> 08 solver consumption
```

The image must explicitly say chapter 07 builds row-space math objects but chapter 08 owns solving.

- [ ] **Step 4: Render calibration only**

Run:

```bash
python3 tmp/ch07-imagegen/render_infographics.py --only 07_contact_math_bridge_map.png
```

Expected: `chapters/07_constraints_contacts_math/assets/07_contact_math_bridge_map.png` is created.

- [ ] **Step 5: Inspect calibration file**

Run:

```bash
file chapters/07_constraints_contacts_math/assets/07_contact_math_bridge_map.png
```

Expected: `PNG image data`.

Open the image visually and check it matches `docs/superpowers/specs/chapter-visual-style-guide.md`: white handout, blue cards, numbered badges, readable Chinese labels, not decorative contact art, and clear `Contacts` / `ContactsKamino` / rows / `J` / `D` / chapter 08 boundaries.

## Task 2: README Image Batch

- [ ] **Step 1: Render README support images**

Render:

- `07_readme_contact_math_spine.png`: chapter identity; chapter 06 geometry becomes contact math, but chapter 08 owns solving.
- `07_readme_file_roles_reading_order.png`: file roles and reading order: `README.md -> source-walkthrough.md -> principle.md -> examples.md -> source-walkthrough-deep.md -> 08`.
- `07_readme_completion_scope_prereq.png`: completion gate, scope, non-scope, prerequisites, GAMES103 bridge, and expected output.

Run:

```bash
python3 tmp/ch07-imagegen/render_infographics.py \
  --only 07_readme_contact_math_spine.png \
  --only 07_readme_file_roles_reading_order.png \
  --only 07_readme_completion_scope_prereq.png
```

- [ ] **Step 2: Integrate README images**

Insert:

- after the opening chapter overview: `07_readme_contact_math_spine.png`
- after `## 文件分工`: `07_readme_file_roles_reading_order.png`
- after `## 完成门槛`: `07_readme_completion_scope_prereq.png`

Coverage note:

- `## 本章目标`, `## 本章范围`, `## 本章明确不做什么`, `## 前置依赖`, `## GAMES103 已有 vs 本章新增`, `## 阅读顺序`, and `## 预期产出` are covered by the nearby README images without repeated immediate insertion.

## Task 3: Principle Image Batch

- [ ] **Step 1: Render principle concept images**

Render:

- `07_principle_contact_to_three_rows.png`
- `07_principle_jacobian_motion_map.png`
- `07_principle_lever_arm_angular_coupling.png`
- `07_principle_delassus_effective_mass.png`
- `07_principle_box_ground_multi_contacts.png`
- `07_principle_solver_facing_next_chapter.png`

`07_contact_math_bridge_map.png` already exists from Task 1.

Run:

```bash
python3 tmp/ch07-imagegen/render_infographics.py \
  --only 07_principle_contact_to_three_rows.png \
  --only 07_principle_jacobian_motion_map.png \
  --only 07_principle_lever_arm_angular_coupling.png \
  --only 07_principle_delassus_effective_mass.png \
  --only 07_principle_box_ground_multi_contacts.png \
  --only 07_principle_solver_facing_next_chapter.png
```

- [ ] **Step 2: Integrate principle images**

Insert:

- after `## 0`: `07_contact_math_bridge_map.png`
- after `## 1`: `07_principle_contact_to_three_rows.png`
- after `## 2`: `07_principle_jacobian_motion_map.png`
- after `## 3`: `07_principle_lever_arm_angular_coupling.png`
- after `## 4`: `07_principle_delassus_effective_mass.png`
- after `## 5`: `07_principle_box_ground_multi_contacts.png`
- after `## 6`: `07_principle_solver_facing_next_chapter.png`

Coverage note:

- `## 7` is covered by the adjacent solver-facing / next-chapter handoff image and the section's own prose.

## Task 4: Source Walkthrough Image Batch

- [ ] **Step 1: Render source-walkthrough images**

Render:

- `07_walkthrough_pipeline_overview.png`
- `07_walkthrough_beginner_path.png`
- `07_walkthrough_stage1_contacts_geometry_handoff.png`
- `07_walkthrough_stage2_solver_facing_contact_block.png`
- `07_walkthrough_stage3_contact_rows_three_directions.png`
- `07_walkthrough_stage4_jacobian_frame_lever_arm.png`
- `07_walkthrough_stage5_delassus_row_space_response.png`
- `07_walkthrough_object_ledger_stop_here.png`

Run:

```bash
python3 tmp/ch07-imagegen/render_infographics.py \
  --only 07_walkthrough_pipeline_overview.png \
  --only 07_walkthrough_beginner_path.png \
  --only 07_walkthrough_stage1_contacts_geometry_handoff.png \
  --only 07_walkthrough_stage2_solver_facing_contact_block.png \
  --only 07_walkthrough_stage3_contact_rows_three_directions.png \
  --only 07_walkthrough_stage4_jacobian_frame_lever_arm.png \
  --only 07_walkthrough_stage5_delassus_row_space_response.png \
  --only 07_walkthrough_object_ledger_stop_here.png
```

- [ ] **Step 2: Integrate source-walkthrough images**

Insert:

- after `## What This Walkthrough Follows`: `07_walkthrough_pipeline_overview.png`
- `## One-Screen Chapter Map`: covered by the adjacent pipeline overview and existing text map
- after `## Beginner Path`: `07_walkthrough_beginner_path.png`
- after `### Stage 1`: `07_walkthrough_stage1_contacts_geometry_handoff.png`
- after `### Stage 2`: `07_walkthrough_stage2_solver_facing_contact_block.png`
- after `### Stage 3`: `07_walkthrough_stage3_contact_rows_three_directions.png`
- after `### Stage 4`: `07_walkthrough_stage4_jacobian_frame_lever_arm.png`
- after `### Stage 5`: `07_walkthrough_stage5_delassus_row_space_response.png`
- after `## Object Ledger`: `07_walkthrough_object_ledger_stop_here.png`
- `## Stop Here` and `## Go Deeper`: covered by the adjacent object-ledger / stop-here image and text navigation without extra insertion.

## Task 5: Examples Image Batch

- [ ] **Step 1: Render examples images**

Render:

- `07_examples_overview_observation_tasks.png`
- `07_examples_sphere_ground_one_contact_three_rows.png`
- `07_examples_box_ground_multiple_contacts_lever_arm.png`
- `07_examples_self_check_handoff.png`

Run:

```bash
python3 tmp/ch07-imagegen/render_infographics.py \
  --only 07_examples_overview_observation_tasks.png \
  --only 07_examples_sphere_ground_one_contact_three_rows.png \
  --only 07_examples_box_ground_multiple_contacts_lever_arm.png \
  --only 07_examples_self_check_handoff.png
```

- [ ] **Step 2: Integrate examples images**

Insert:

- after the opening examples overview: `07_examples_overview_observation_tasks.png`
- after `## 主例子: `sphere-ground``: `07_examples_sphere_ground_one_contact_three_rows.png`
- after `## 对照例子: `box-ground``: `07_examples_box_ground_multiple_contacts_lever_arm.png`
- after `## 这页怎么配合其他文件`: `07_examples_self_check_handoff.png`

Coverage note:

- `## 自检` is covered by the adjacent self-check / handoff image and its existing question list.

## Task 6: Path, Visual, and Source-Truth Review

- [ ] **Step 1: Verify all expected assets exist**

Run a script that checks every `07_*.png` filename listed above exists under `chapters/07_constraints_contacts_math/assets/`.

- [ ] **Step 2: Verify Markdown image references resolve**

Run a script that parses Markdown image references in chapter 07 and confirms each referenced local asset exists.

- [ ] **Step 3: Verify image files are real PNGs**

Run:

```bash
file chapters/07_constraints_contacts_math/assets/07_*.png | rg -v 'PNG image data'
```

Expected: no output.

- [ ] **Step 4: Review against visual style guide**

Check the generated batch against `docs/superpowers/specs/chapter-visual-style-guide.md`:

- white handout style
- thin blue card borders
- dark-blue section hierarchy
- numbered badges
- Chinese-first short labels
- readable at Markdown size
- no decorative simulation art
- no fake code or terminal text

- [ ] **Step 5: Review against chapter 07 source truth**

Check every image for these source-truth boundaries:

- `Contacts` remains geometry handoff
- `ContactsKamino` / solver-facing contact is a bridge object
- one active contact expands to `1 normal + 2 tangents`
- Jacobian maps body velocities to row velocities through direction and lever arm
- Delassus / effective mass describes row-space response and possible row-row coupling
- chapter 07 does not solve constraints
- one shape pair is not always one contact
- one contact is not one row

- [ ] **Step 6: Request independent review**

Ask at least one reviewer agent to inspect the diff for:

- broken image paths
- source-truth drift
- missing expected coverage
- unreadable or off-style visual assets
- accidental edits to `source-walkthrough-deep.md`

Fix any Critical or Important findings before final commit.

- [ ] **Step 7: Final hygiene**

Run:

```bash
git diff --check -- \
  chapters/07_constraints_contacts_math \
  docs/superpowers/plans/2026-04-25-chapter-07-tutorial-infographics.md \
  docs/superpowers/specs/2026-04-25-chapter-07-tutorial-infographics-design.md
```

Expected: no output.

Then mark every completed checkbox in this plan.

## Task 7: Final Commit and Integration

- [ ] **Step 1: Commit implementation**

Commit the chapter 07 Markdown integrations, PNG assets, and checked-off plan updates.

- [ ] **Step 2: Merge to `main`**

Fast-forward `main` to the feature branch after verification.

- [ ] **Step 3: Push**

Push `main` to `origin`.

- [ ] **Step 4: Cleanup**

Remove the chapter 07 worktree and delete the merged local branch.
