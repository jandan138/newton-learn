# Chapter 06 Tutorial Infographics Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [x]`) syntax for tracking.

**Goal:** Generate and integrate PNG tutorial infographics across chapter 06 reader-facing pages.

**Architecture:** Keep exact source truth in Markdown and add deterministic PNG teaching anchors as visual summaries. Reuse chapter 05's dense Chinese learning-handout visual system, but make chapter 06's collision handoff the central visual spine: body/world state, shape world pose, expanded AABB, broad-phase candidate pairs, narrow-phase `ContactData`, `write_contact(...)`, and final `Contacts`. Final assets live under `chapters/06_collision/assets/`.

**Tech Stack:** Markdown, deterministic raster PNG rendering with Python/Pillow, optional temporary CJK font download into ignored `tmp/`, shell verification with `rg`, `file`, Python image-reference checks, and `git diff --check`.

**Implementation note:** Adjacent short administrative sections may share one strong nearby image rather than repeating large PNGs immediately after every heading. `source-walkthrough-deep.md` stays intentionally out of scope. If review finds source-truth drift in generated text, the affected PNG should be re-rendered deterministically rather than accepted with incorrect labels.

---

## Files

- Modify: `chapters/06_collision/README.md`
- Modify: `chapters/06_collision/principle.md`
- Modify: `chapters/06_collision/source-walkthrough.md`
- Modify: `chapters/06_collision/examples.md`
- Modify: `chapters/06_collision/assets/06_collision_bridge_map.png`
- Create: `chapters/06_collision/assets/06_readme_collision_spine.png`
- Create: `chapters/06_collision/assets/06_readme_file_roles_reading_order.png`
- Create: `chapters/06_collision/assets/06_readme_completion_scope_prereq.png`
- Create: `chapters/06_collision/assets/06_principle_shape_not_body_boundary.png`
- Create: `chapters/06_collision/assets/06_principle_broad_phase_maybe_list.png`
- Create: `chapters/06_collision/assets/06_principle_narrow_phase_contact_geometry.png`
- Create: `chapters/06_collision/assets/06_principle_shape_type_routing.png`
- Create: `chapters/06_collision/assets/06_principle_contacts_handoff_buffer.png`
- Create: `chapters/06_collision/assets/06_principle_next_chapters_map.png`
- Create: `chapters/06_collision/assets/06_walkthrough_pipeline_overview.png`
- Create: `chapters/06_collision/assets/06_walkthrough_beginner_path.png`
- Create: `chapters/06_collision/assets/06_walkthrough_stage1_model_state_shape_data.png`
- Create: `chapters/06_collision/assets/06_walkthrough_stage2_compute_shape_aabbs.png`
- Create: `chapters/06_collision/assets/06_walkthrough_stage3_broad_phase_candidate_pairs.png`
- Create: `chapters/06_collision/assets/06_walkthrough_stage4_narrow_phase_contactdata.png`
- Create: `chapters/06_collision/assets/06_walkthrough_stage5_write_contact_contacts.png`
- Create: `chapters/06_collision/assets/06_walkthrough_object_ledger_stop_here.png`
- Create: `chapters/06_collision/assets/06_examples_overview_observation_tasks.png`
- Create: `chapters/06_collision/assets/06_examples_basic_shapes_sphere_ground.png`
- Create: `chapters/06_collision/assets/06_examples_pyramid_contact_set_growth.png`
- Create: `chapters/06_collision/assets/06_examples_contact_fields_watchlist.png`
- Modify: `docs/superpowers/plans/2026-04-25-chapter-06-tutorial-infographics.md`

Temporary ignored implementation files:

- Create/use: `tmp/ch06-imagegen/render_infographics.py`
- Optional create/use: `tmp/ch06-imagegen/fonts/NotoSansCJKsc-Regular.otf`

## Task 1: Calibration Renderer and Image

- [x] **Step 1: Prepare ignored renderer workspace**

Run:

```bash
mkdir -p tmp/ch06-imagegen/fonts chapters/06_collision/assets
```

Expected: both directories exist. `tmp/` is ignored by `.gitignore`, so the renderer does not become part of the final commit.

- [x] **Step 2: Ensure a readable CJK font is available**

Run:

```bash
python3 - <<'PY'
from pathlib import Path
paths = [
    Path('/usr/share/fonts/opentype/noto/NotoSansCJK-Regular.ttc'),
    Path('/usr/share/fonts/opentype/noto/NotoSansCJKsc-Regular.otf'),
    Path('/usr/share/fonts/truetype/noto/NotoSansCJK-Regular.ttc'),
    Path('tmp/ch06-imagegen/fonts/NotoSansCJKsc-Regular.otf'),
]
print(next((str(p) for p in paths if p.exists()), 'missing'))
PY
```

Expected: a font path or `missing`. If it prints `missing`, download `NotoSansCJKsc-Regular.otf` into `tmp/ch06-imagegen/fonts/` and re-run the probe. Do not commit the font.

- [x] **Step 3: Create the deterministic renderer**

Write `tmp/ch06-imagegen/render_infographics.py`.

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

Calibration content for `06_collision_bridge_map.png`:

```text
body/world state + shape metadata
-> shape world pose
-> expanded AABB
-> broad phase candidate pairs
-> narrow phase ContactData
-> write_contact(...)
-> Contacts
```

The image must explicitly say `broad phase` produces `maybe-list / candidate pairs`, not final contacts.

- [x] **Step 4: Render calibration only**

Run:

```bash
python3 tmp/ch06-imagegen/render_infographics.py --only 06_collision_bridge_map.png
```

Expected: `chapters/06_collision/assets/06_collision_bridge_map.png` is created or replaced.

- [x] **Step 5: Inspect calibration file**

Run:

```bash
file chapters/06_collision/assets/06_collision_bridge_map.png
```

Expected: `PNG image data`.

Open the image visually and check it matches `docs/superpowers/specs/chapter-visual-style-guide.md`: white handout, blue cards, numbered badges, readable Chinese labels, not decorative collision art, and clear body / shape / broad phase / narrow phase / `Contacts` boundaries.

## Task 2: README Image Batch

- [x] **Step 1: Render README support images**

Render:

- `06_readme_collision_spine.png`: chapter identity; chapter 05 body state plus chapter 04 shape metadata now enter collision.
- `06_readme_file_roles_reading_order.png`: file roles and reading order: `README.md -> source-walkthrough.md -> principle.md -> examples.md -> source-walkthrough-deep.md -> 07 / 08`.
- `06_readme_completion_scope_prereq.png`: completion gate, scope, non-scope, prerequisites, GAMES103 bridge, and expected output.

Run:

```bash
python3 tmp/ch06-imagegen/render_infographics.py \
  --only 06_readme_collision_spine.png \
  --only 06_readme_file_roles_reading_order.png \
  --only 06_readme_completion_scope_prereq.png
```

- [x] **Step 2: Integrate README images**

Insert:

- after the opening chapter overview: `06_readme_collision_spine.png`
- after `## 文件分工`: `06_readme_file_roles_reading_order.png`
- after `## 完成门槛`: `06_readme_completion_scope_prereq.png`

Coverage note:

- `## 本章目标`, `## 本章范围`, `## 本章明确不做什么`, `## 前置依赖`, `## GAMES103 已有 vs 本章新增`, `## 阅读顺序`, and `## 预期产出` are covered by the nearby README images without repeated immediate insertion.

## Task 3: Principle Image Batch

- [x] **Step 1: Render principle concept images**

Render:

- `06_principle_shape_not_body_boundary.png`
- `06_principle_broad_phase_maybe_list.png`
- `06_principle_narrow_phase_contact_geometry.png`
- `06_principle_shape_type_routing.png`
- `06_principle_contacts_handoff_buffer.png`
- `06_principle_next_chapters_map.png`

`06_collision_bridge_map.png` already exists from Task 1.

Run:

```bash
python3 tmp/ch06-imagegen/render_infographics.py \
  --only 06_principle_shape_not_body_boundary.png \
  --only 06_principle_broad_phase_maybe_list.png \
  --only 06_principle_narrow_phase_contact_geometry.png \
  --only 06_principle_shape_type_routing.png \
  --only 06_principle_contacts_handoff_buffer.png \
  --only 06_principle_next_chapters_map.png
```

- [x] **Step 2: Integrate principle images**

Insert:

- keep the existing bridge slot in `## 0` and use the regenerated `06_collision_bridge_map.png`
- after `## 1`: `06_principle_shape_not_body_boundary.png`
- after `## 2`: `06_principle_broad_phase_maybe_list.png`
- after `## 3`: `06_principle_narrow_phase_contact_geometry.png`
- after `## 4`: `06_principle_shape_type_routing.png`
- after `## 5`: `06_principle_contacts_handoff_buffer.png`
- after `## 6`: `06_principle_next_chapters_map.png`

## Task 4: Source Walkthrough Image Batch

- [x] **Step 1: Render source-walkthrough images**

Render:

- `06_walkthrough_pipeline_overview.png`
- `06_walkthrough_beginner_path.png`
- `06_walkthrough_stage1_model_state_shape_data.png`
- `06_walkthrough_stage2_compute_shape_aabbs.png`
- `06_walkthrough_stage3_broad_phase_candidate_pairs.png`
- `06_walkthrough_stage4_narrow_phase_contactdata.png`
- `06_walkthrough_stage5_write_contact_contacts.png`
- `06_walkthrough_object_ledger_stop_here.png`

Run:

```bash
python3 tmp/ch06-imagegen/render_infographics.py \
  --only 06_walkthrough_pipeline_overview.png \
  --only 06_walkthrough_beginner_path.png \
  --only 06_walkthrough_stage1_model_state_shape_data.png \
  --only 06_walkthrough_stage2_compute_shape_aabbs.png \
  --only 06_walkthrough_stage3_broad_phase_candidate_pairs.png \
  --only 06_walkthrough_stage4_narrow_phase_contactdata.png \
  --only 06_walkthrough_stage5_write_contact_contacts.png \
  --only 06_walkthrough_object_ledger_stop_here.png
```

- [x] **Step 2: Integrate source-walkthrough images**

Insert:

- after `## What This Walkthrough Follows`: `06_walkthrough_pipeline_overview.png`
- `## One-Screen Chapter Map`: covered by the adjacent pipeline overview and existing text map.
- after `## Beginner Path`: `06_walkthrough_beginner_path.png`
- after `### Stage 1`: `06_walkthrough_stage1_model_state_shape_data.png`
- after `### Stage 2`: `06_walkthrough_stage2_compute_shape_aabbs.png`
- after `### Stage 3`: `06_walkthrough_stage3_broad_phase_candidate_pairs.png`
- after `### Stage 4`: `06_walkthrough_stage4_narrow_phase_contactdata.png`
- after `### Stage 5`: `06_walkthrough_stage5_write_contact_contacts.png`
- after `## Object Ledger`: `06_walkthrough_object_ledger_stop_here.png`
- `## Stop Here` and `## Go Deeper`: covered by the adjacent object-ledger / stop-here image and text navigation without extra insertion.

## Task 5: Examples Image Batch

- [x] **Step 1: Render examples images**

Render:

- `06_examples_overview_observation_tasks.png`
- `06_examples_basic_shapes_sphere_ground.png`
- `06_examples_pyramid_contact_set_growth.png`
- `06_examples_contact_fields_watchlist.png`

Run:

```bash
python3 tmp/ch06-imagegen/render_infographics.py \
  --only 06_examples_overview_observation_tasks.png \
  --only 06_examples_basic_shapes_sphere_ground.png \
  --only 06_examples_pyramid_contact_set_growth.png \
  --only 06_examples_contact_fields_watchlist.png
```

- [x] **Step 2: Integrate examples images**

Insert:

- after the opening examples overview: `06_examples_overview_observation_tasks.png`
- after `## 主例子：`basic_shapes` 里的球落地`: `06_examples_basic_shapes_sphere_ground.png`
- after `## 对照例子：`pyramid --test --pyramid-size 4 --num-pyramids 1``: `06_examples_pyramid_contact_set_growth.png`
- after `## 这页怎么配合其他文件`: `06_examples_contact_fields_watchlist.png`

## Task 6: Verification, Review, Commit, Push

- [x] **Step 1: Verify Markdown image references**

Run:

```bash
python3 - <<'PY'
import pathlib, re, sys
files = [
    pathlib.Path('chapters/06_collision/README.md'),
    pathlib.Path('chapters/06_collision/principle.md'),
    pathlib.Path('chapters/06_collision/source-walkthrough.md'),
    pathlib.Path('chapters/06_collision/examples.md'),
]
missing = []
refs = 0
for path in files:
    text = path.read_text()
    for match in re.finditer(r'!\[[^\]]*\]\(([^)]+)\)', text):
        refs += 1
        target = (path.parent / match.group(1)).resolve()
        if not target.exists():
            missing.append((str(path), match.group(1)))
if missing:
    print('Missing image refs:')
    for item in missing:
        print(item)
    sys.exit(1)
print(f'all image refs exist ({refs} refs)')
PY
```

Expected: `all image refs exist (22 refs)`.

- [x] **Step 2: Verify expected asset inventory**

Run:

```bash
python3 - <<'PY'
import pathlib, sys
expected = {
    '06_collision_bridge_map.png',
    '06_readme_collision_spine.png',
    '06_readme_file_roles_reading_order.png',
    '06_readme_completion_scope_prereq.png',
    '06_principle_shape_not_body_boundary.png',
    '06_principle_broad_phase_maybe_list.png',
    '06_principle_narrow_phase_contact_geometry.png',
    '06_principle_shape_type_routing.png',
    '06_principle_contacts_handoff_buffer.png',
    '06_principle_next_chapters_map.png',
    '06_walkthrough_pipeline_overview.png',
    '06_walkthrough_beginner_path.png',
    '06_walkthrough_stage1_model_state_shape_data.png',
    '06_walkthrough_stage2_compute_shape_aabbs.png',
    '06_walkthrough_stage3_broad_phase_candidate_pairs.png',
    '06_walkthrough_stage4_narrow_phase_contactdata.png',
    '06_walkthrough_stage5_write_contact_contacts.png',
    '06_walkthrough_object_ledger_stop_here.png',
    '06_examples_overview_observation_tasks.png',
    '06_examples_basic_shapes_sphere_ground.png',
    '06_examples_pyramid_contact_set_growth.png',
    '06_examples_contact_fields_watchlist.png',
}
asset_dir = pathlib.Path('chapters/06_collision/assets')
actual = {p.name for p in asset_dir.glob('06_*.png')}
missing = sorted(expected - actual)
extra = sorted(actual - expected)
print('expected', len(expected), 'actual', len(actual))
if missing or extra:
    print('missing', missing)
    print('extra', extra)
    sys.exit(1)
PY
```

Expected: `expected 22 actual 22` and no missing / extra output.

- [x] **Step 3: Verify PNG assets**

Run:

```bash
file chapters/06_collision/assets/06_*.png
```

Expected: every line says `PNG image data`.

- [x] **Step 4: Verify Markdown diff cleanliness**

Run:

```bash
git diff --check -- chapters/06_collision docs/superpowers/specs/2026-04-25-chapter-06-tutorial-infographics-design.md docs/superpowers/plans/2026-04-25-chapter-06-tutorial-infographics.md
```

Expected: no output.

- [x] **Step 5: Verify deep walkthrough stayed untouched**

Run:

```bash
git diff -- chapters/06_collision/source-walkthrough-deep.md
git status --short -- chapters/06_collision/source-walkthrough-deep.md
```

Expected: no output.

- [x] **Step 6: Run visual / semantic review**

Review checklist:

- style matches chapter 05 dense Chinese handout direction
- no fake source code, fake terminal UI, SVG, or HTML fallback
- `body_q` and shape metadata are visually separate
- broad phase only produces `candidate pairs`
- narrow phase produces `ContactData`, not solver output
- `write_contact(...)` writes final `Contacts`
- examples remain observation tasks, not solver tuning advice

Fix any concrete issues, re-render affected PNGs, and re-run Steps 1-5.

- [x] **Step 7: Mark this plan complete**

After all checks pass, update every checkbox in this file from `[ ]` to `[x]`.

- [x] **Step 8: Commit and push**

Run:

```bash
git add chapters/06_collision docs/superpowers/specs/2026-04-25-chapter-06-tutorial-infographics-design.md docs/superpowers/plans/2026-04-25-chapter-06-tutorial-infographics.md
git diff --cached --check
git commit -m "docs: add chapter 06 tutorial infographics"
git fetch origin main
git switch main
git merge --ff-only ch06-tutorial-infographics
git push origin main
git worktree remove .worktrees/ch06-tutorial-infographics
git branch -d ch06-tutorial-infographics
```

Expected: merge is fast-forward, push succeeds, feature worktree and branch are cleaned up.
