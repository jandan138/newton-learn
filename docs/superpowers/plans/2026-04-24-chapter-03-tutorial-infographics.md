# Chapter 03 Tutorial Infographics Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Generate and integrate PNG tutorial infographics across chapter 03 reader-facing pages.

**Architecture:** Keep exact source truth in Markdown and add model-generated PNGs as teaching anchors. Use a small set of dense, reusable Chinese learning-handout images rather than one weak image per short administrative heading. Final assets live under `chapters/03_math_geometry/assets/`.

**Tech Stack:** Markdown, PNG assets generated through Codex native image generation, shell verification with `rg`, `file`, and `git diff --check`.

**Implementation note:** Adjacent short administrative sections use nearby strong summary images instead of repeating the same large PNG multiple times in immediate succession.

---

## Files

- Modify: `chapters/03_math_geometry/README.md`
- Modify: `chapters/03_math_geometry/principle.md`
- Modify: `chapters/03_math_geometry/source-walkthrough.md`
- Create: `chapters/03_math_geometry/assets/03_readme_chapter_spine_overview.png`
- Create: `chapters/03_math_geometry/assets/03_readme_file_roles_and_reading_order.png`
- Create: `chapters/03_math_geometry/assets/03_readme_completion_scope_prereq.png`
- Create: `chapters/03_math_geometry/assets/03_principle_frame_reference_map.png`
- Create: `chapters/03_math_geometry/assets/03_principle_transform_chain_map.png`
- Create: `chapters/03_math_geometry/assets/03_principle_quaternion_orientation_map.png`
- Create: `chapters/03_math_geometry/assets/03_principle_spatial_quantity_map.png`
- Create: `chapters/03_math_geometry/assets/03_principle_shape_geotype_map.png`
- Create: `chapters/03_math_geometry/assets/03_principle_inertia_bridge_map.png`
- Create: `chapters/03_math_geometry/assets/03_principle_next_chapters_map.png`
- Create: `chapters/03_math_geometry/assets/03_walkthrough_pipeline_overview.png`
- Create: `chapters/03_math_geometry/assets/03_walkthrough_beginner_path.png`
- Create: `chapters/03_math_geometry/assets/03_walkthrough_stage1_local_ledger.png`
- Create: `chapters/03_math_geometry/assets/03_walkthrough_stage2_transform_chain.png`
- Create: `chapters/03_math_geometry/assets/03_walkthrough_stage3_spatial_helpers.png`
- Create: `chapters/03_math_geometry/assets/03_walkthrough_stage4_geotype_dispatch.png`
- Create: `chapters/03_math_geometry/assets/03_walkthrough_stage5_inertia_compression.png`
- Create: `chapters/03_math_geometry/assets/03_walkthrough_object_ledger_stop_here.png`

## Task 1: Calibration Image

- [x] **Step 1: Generate the calibration overview PNG**

Run native image generation for `03_readme_chapter_spine_overview.png`.

Prompt content:

```text
Create a clean Chinese tutorial infographic for Newton Learn Chapter 03: 数学与几何基础.
Style: white background, thin blue rounded card borders, dark-blue title, numbered blue badges, compact multi-panel teacher handout layout, small friendly icons, arrows, flow strips, high information density but readable.
Content: teach why chapter 03 exists after chapter 02. Show the spine:
1 local frame relationships
2 transform chains
3 quaternion / orientation
4 spatial quantities
5 shape representation / GeoType
6 inertia bridge
bottom recap: 03 把 Model 背后的几何词汇垫平，再送去 04 / 05 / 06.
Use Chinese-first text; keep English only for exact terms: frame, wp.transform, body_q, body_qd, GeoType, inertia.
Do not invent code. Do not use SVG, HTML, screenshot, terminal UI, or photorealistic 3D rendering. Generate raster PNG.
```

- [x] **Step 2: Inspect calibration**

Run:

```bash
file chapters/03_math_geometry/assets/03_readme_chapter_spine_overview.png
```

Expected: `PNG image data`.

Open the image visually and check it matches `docs/superpowers/specs/chapter-visual-style-guide.md`: white handout, blue cards, numbered badges, readable Chinese labels, not a 3D scene.

## Task 2: README Image Batch

- [x] **Step 1: Generate README support images**

Generate:

- `03_readme_file_roles_and_reading_order.png`: file roles, README -> principle -> source-walkthrough -> deep walkthrough, and reading order.
- `03_readme_completion_scope_prereq.png`: completion gate, in-scope / out-of-scope, prerequisites, and GAMES103 bridge.

- [x] **Step 2: Integrate README images**

Insert images after these sections:

- after opening chapter overview: `03_readme_chapter_spine_overview.png`
- after `## 文件分工`: `03_readme_file_roles_and_reading_order.png`
- after `## 完成门槛`: `03_readme_completion_scope_prereq.png`
- `## 前置依赖` and `## 阅读顺序`: covered by the nearby completion/scope and file-role images without immediate duplicate insertion.

## Task 3: Principle Image Batch

- [x] **Step 1: Generate principle concept images**

Generate:

- `03_principle_frame_reference_map.png`
- `03_principle_transform_chain_map.png`
- `03_principle_quaternion_orientation_map.png`
- `03_principle_spatial_quantity_map.png`
- `03_principle_shape_geotype_map.png`
- `03_principle_inertia_bridge_map.png`
- `03_principle_next_chapters_map.png`

- [x] **Step 2: Integrate principle images**

Insert:

- after `## 0`: reuse `03_readme_chapter_spine_overview.png`
- after `## 1`: `03_principle_frame_reference_map.png`
- after `## 2`: `03_principle_transform_chain_map.png`
- after `## 3`: `03_principle_quaternion_orientation_map.png`
- after `## 4`: `03_principle_spatial_quantity_map.png`
- after `## 5`: `03_principle_shape_geotype_map.png`
- after `## 6`: `03_principle_inertia_bridge_map.png`
- after `## 7`: `03_principle_next_chapters_map.png`

## Task 4: Source Walkthrough Image Batch

- [x] **Step 1: Generate source-walkthrough images**

Generate:

- `03_walkthrough_pipeline_overview.png`
- `03_walkthrough_beginner_path.png`
- `03_walkthrough_stage1_local_ledger.png`
- `03_walkthrough_stage2_transform_chain.png`
- `03_walkthrough_stage3_spatial_helpers.png`
- `03_walkthrough_stage4_geotype_dispatch.png`
- `03_walkthrough_stage5_inertia_compression.png`
- `03_walkthrough_object_ledger_stop_here.png`

- [x] **Step 2: Integrate source-walkthrough images**

Insert:

- after `## What This Walkthrough Follows`: `03_walkthrough_pipeline_overview.png`
- `## One-Screen Chapter Map`: covered by the immediately preceding pipeline overview image and the adjacent text map without duplicate insertion.
- after `## Beginner Path`: `03_walkthrough_beginner_path.png`
- after `### Stage 1`: `03_walkthrough_stage1_local_ledger.png`
- after `### Stage 2`: `03_walkthrough_stage2_transform_chain.png`
- after `### Stage 3`: `03_walkthrough_stage3_spatial_helpers.png`
- after `### Stage 4`: `03_walkthrough_stage4_geotype_dispatch.png`
- after `### Stage 5`: `03_walkthrough_stage5_inertia_compression.png`
- after `## Object Ledger`: `03_walkthrough_object_ledger_stop_here.png`
- `## Stop Here`: covered by the adjacent object-ledger/stop-here image without duplicate insertion.

## Task 5: Verification And Commit

- [x] **Step 1: Verify image paths**

Run:

```bash
python3 - <<'PY'
import pathlib, re, sys
files = [
    pathlib.Path('chapters/03_math_geometry/README.md'),
    pathlib.Path('chapters/03_math_geometry/principle.md'),
    pathlib.Path('chapters/03_math_geometry/source-walkthrough.md'),
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
file chapters/03_math_geometry/assets/03_*.png
```

Expected: every line says `PNG image data`.

- [x] **Step 3: Verify Markdown diff cleanliness**

Run:

```bash
git diff --check -- chapters/03_math_geometry docs/superpowers/plans/2026-04-24-chapter-03-tutorial-infographics.md
```

Expected: no output.

- [x] **Step 4: Coverage scan**

Run:

```bash
rg -n '^##|^###|!\[' chapters/03_math_geometry/README.md chapters/03_math_geometry/principle.md chapters/03_math_geometry/source-walkthrough.md
```

Expected: each reader-facing section has a nearby image anchor, with documented reuse for summary/admin sections.

- [x] **Step 5: Commit**

Run:

```bash
git add chapters/03_math_geometry docs/superpowers/plans/2026-04-24-chapter-03-tutorial-infographics.md
git commit -m "docs: add chapter 03 tutorial infographics"
```
