# Chapter 04 Tutorial Infographics Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Generate and integrate PNG tutorial infographics across chapter 04 reader-facing pages.

**Architecture:** Keep exact source truth in Markdown and add model-generated PNGs as teaching anchors. Reuse chapter 03's dense Chinese learning-handout visual system, but make chapter 04's scene-to-`Model` translation pipeline the central visual spine. Final assets live under `chapters/04_scene_usd/assets/`.

**Tech Stack:** Markdown, PNG assets generated through Codex native image generation, shell verification with `rg`, `file`, Python image-reference checks, and `git diff --check`.

**Implementation note:** Adjacent short administrative sections may share one strong nearby image rather than repeating large PNGs immediately after every heading. `source-walkthrough-deep.md` stays intentionally out of scope.

---

## Files

- Modify: `chapters/04_scene_usd/README.md`
- Modify: `chapters/04_scene_usd/principle.md`
- Modify: `chapters/04_scene_usd/source-walkthrough.md`
- Create: `chapters/04_scene_usd/assets/04_readme_scene_to_model_spine.png`
- Create: `chapters/04_scene_usd/assets/04_readme_file_roles_reading_order.png`
- Create: `chapters/04_scene_usd/assets/04_readme_completion_scope_prereq.png`
- Create: `chapters/04_scene_usd/assets/04_principle_after_ch03_bridge.png`
- Create: `chapters/04_scene_usd/assets/04_principle_min_scene_five_step_flow.png`
- Create: `chapters/04_scene_usd/assets/04_principle_scene_object_field_ledger.png`
- Create: `chapters/04_scene_usd/assets/04_principle_entry_importer_boundary.png`
- Create: `chapters/04_scene_usd/assets/04_principle_schema_resolver_boundary.png`
- Create: `chapters/04_scene_usd/assets/04_principle_builder_accumulation_map.png`
- Create: `chapters/04_scene_usd/assets/04_principle_mass_property_priority.png`
- Create: `chapters/04_scene_usd/assets/04_principle_finalize_next_chapters.png`
- Create: `chapters/04_scene_usd/assets/04_walkthrough_pipeline_overview.png`
- Create: `chapters/04_scene_usd/assets/04_walkthrough_beginner_path.png`
- Create: `chapters/04_scene_usd/assets/04_walkthrough_stage1_add_usd_parse_usd.png`
- Create: `chapters/04_scene_usd/assets/04_walkthrough_stage2_body_joint_local_pose.png`
- Create: `chapters/04_scene_usd/assets/04_walkthrough_stage3_schema_resolver_keys.png`
- Create: `chapters/04_scene_usd/assets/04_walkthrough_stage4_shape_material_mass_builder.png`
- Create: `chapters/04_scene_usd/assets/04_walkthrough_stage5_finalize_model_arrays.png`
- Create: `chapters/04_scene_usd/assets/04_walkthrough_object_ledger_stop_here.png`
- Modify: `docs/superpowers/plans/2026-04-25-chapter-04-tutorial-infographics.md`

## Task 1: Calibration Image

- [ ] **Step 1: Create chapter 04 assets directory**

Run:

```bash
mkdir -p chapters/04_scene_usd/assets
```

Expected: `chapters/04_scene_usd/assets/` exists.

- [ ] **Step 2: Generate the calibration PNG**

Run Codex native image generation for:

`chapters/04_scene_usd/assets/04_principle_min_scene_five_step_flow.png`

Prompt content:

```text
Create a clean Chinese tutorial infographic for Newton Learn Chapter 04: 场景描述与 USD 解析.
Style: white background, thin blue rounded card borders, dark-blue title, numbered blue badges, compact multi-panel teacher review-sheet layout, small friendly icons, arrows, flow boxes, role cards, high information density but readable.
Content: show a minimal teaching scene on the left: ground + box + revolute joint + authored attrs. On the right show the five-step bridge:
1 importer identifies body / joint / shape / local pose
2 schema resolver translates authored attrs
3 builder accumulates body / joint / shape rows
4 finalize() freezes the builder
5 Model static arrays are ready for 05 articulation and 06 collision
Use Chinese-first labels. English/code terms allowed only for exact names: add_usd(), parse_usd(), schema resolver, builder, finalize(), Model, shape_transform, joint_X_p/X_c, body_mass.
Bottom recap: Model 不是原始 scene graph，而是翻译和冻结后的静态物理结构.
Do not invent source code or USD syntax. Do not include long paragraphs. Do not use SVG, HTML, screenshot, terminal UI, or photorealistic 3D rendering. Generate raster PNG.
```

- [ ] **Step 3: Inspect calibration**

Run:

```bash
file chapters/04_scene_usd/assets/04_principle_min_scene_five_step_flow.png
```

Expected: `PNG image data`.

Open the image visually and check it matches `docs/superpowers/specs/chapter-visual-style-guide.md`: white handout, blue cards, numbered badges, readable Chinese labels, not a 3D scene, and clear importer / resolver / builder / finalize boundaries.

## Task 2: README Image Batch

- [ ] **Step 1: Generate README support images**

Generate:

- `04_readme_scene_to_model_spine.png`: chapter identity; chapter 03 fields need a source; chapter 04 explains `scene input -> Model`.
- `04_readme_file_roles_reading_order.png`: file roles and reading order: `README.md -> principle.md -> source-walkthrough.md -> source-walkthrough-deep.md`.
- `04_readme_completion_scope_prereq.png`: completion gate, scope, non-scope, prerequisites, and expected output.

- [ ] **Step 2: Integrate README images**

Insert:

- after the opening chapter overview: `04_readme_scene_to_model_spine.png`
- after `## 文件分工`: `04_readme_file_roles_reading_order.png`
- after `## 完成门槛`: `04_readme_completion_scope_prereq.png`

Coverage note:

- `## 本章目标`, `## 本章范围`, `## 本章明确不做什么`, `## 前置依赖`, `## 先带着这张表读`, `## 阅读顺序`, and `## 预期产出` are covered by the nearby README images without repeated immediate insertion.

## Task 3: Principle Image Batch

- [ ] **Step 1: Generate principle concept images**

Generate:

- `04_principle_after_ch03_bridge.png`
- `04_principle_scene_object_field_ledger.png`
- `04_principle_entry_importer_boundary.png`
- `04_principle_schema_resolver_boundary.png`
- `04_principle_builder_accumulation_map.png`
- `04_principle_mass_property_priority.png`
- `04_principle_finalize_next_chapters.png`

`04_principle_min_scene_five_step_flow.png` already exists from Task 1.

- [ ] **Step 2: Integrate principle images**

Insert:

- after `## 0`: `04_principle_after_ch03_bridge.png`
- after `## 1`: `04_principle_min_scene_five_step_flow.png`
- after `## 2`: `04_principle_scene_object_field_ledger.png`
- after `## 3`: `04_principle_entry_importer_boundary.png`
- after `## 4`: `04_principle_schema_resolver_boundary.png`
- after `## 5`: `04_principle_builder_accumulation_map.png`
- after `## 6`: `04_principle_mass_property_priority.png`
- after `## 7`: `04_principle_finalize_next_chapters.png`
- `## 8` is covered by the immediately preceding finalize / next-chapters image.

## Task 4: Source Walkthrough Image Batch

- [ ] **Step 1: Generate source-walkthrough images**

Generate:

- `04_walkthrough_pipeline_overview.png`
- `04_walkthrough_beginner_path.png`
- `04_walkthrough_stage1_add_usd_parse_usd.png`
- `04_walkthrough_stage2_body_joint_local_pose.png`
- `04_walkthrough_stage3_schema_resolver_keys.png`
- `04_walkthrough_stage4_shape_material_mass_builder.png`
- `04_walkthrough_stage5_finalize_model_arrays.png`
- `04_walkthrough_object_ledger_stop_here.png`

- [ ] **Step 2: Integrate source-walkthrough images**

Insert:

- after `## What This Walkthrough Follows`: `04_walkthrough_pipeline_overview.png`
- `## One-Screen Chapter Map`: covered by the adjacent pipeline overview and existing text map.
- after `## Beginner Path`: `04_walkthrough_beginner_path.png`
- after `### Stage 1`: `04_walkthrough_stage1_add_usd_parse_usd.png`
- after `### Stage 2`: `04_walkthrough_stage2_body_joint_local_pose.png`
- after `### Stage 3`: `04_walkthrough_stage3_schema_resolver_keys.png`
- after `### Stage 4`: `04_walkthrough_stage4_shape_material_mass_builder.png`
- after `### Stage 5`: `04_walkthrough_stage5_finalize_model_arrays.png`
- after `## Object Ledger`: `04_walkthrough_object_ledger_stop_here.png`
- `## Stop Here` and `## Go Deeper`: covered by the adjacent object-ledger / stop-here image and text navigation without extra insertion.

## Task 5: Verification, Review, Commit, Push

- [ ] **Step 1: Verify Markdown image references**

Run:

```bash
python3 - <<'PY'
import pathlib, re, sys
files = [
    pathlib.Path('chapters/04_scene_usd/README.md'),
    pathlib.Path('chapters/04_scene_usd/principle.md'),
    pathlib.Path('chapters/04_scene_usd/source-walkthrough.md'),
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

- [ ] **Step 2: Verify PNG assets**

Run:

```bash
file chapters/04_scene_usd/assets/04_*.png
```

Expected: every line says `PNG image data`.

- [ ] **Step 3: Verify Markdown diff cleanliness**

Run:

```bash
git diff --check -- chapters/04_scene_usd docs/superpowers/specs/2026-04-25-chapter-04-tutorial-infographics-design.md docs/superpowers/plans/2026-04-25-chapter-04-tutorial-infographics.md
```

Expected: no output.

- [ ] **Step 4: Coverage scan**

Run:

```bash
rg -n '^##|^###|!\[' chapters/04_scene_usd/README.md chapters/04_scene_usd/principle.md chapters/04_scene_usd/source-walkthrough.md
```

Expected: each reader-facing section has a nearby image anchor, with documented reuse for short/admin sections.

- [ ] **Step 5: Visual style review**

Create or inspect a contact sheet for all `04_*.png` files and compare against `docs/superpowers/specs/chapter-visual-style-guide.md` and chapter 03 assets.

Expected:

- white Chinese tutorial-handout style
- blue card boundaries and numbered badges
- no blank images
- no decorative 3D scene drift
- no fake code / fake terminal UI / invented USD syntax
- source-truth shortcuts remain bounded by nearby Markdown

- [ ] **Step 6: Commit**

Run:

```bash
git add chapters/04_scene_usd docs/superpowers/specs/2026-04-25-chapter-04-tutorial-infographics-design.md docs/superpowers/plans/2026-04-25-chapter-04-tutorial-infographics.md
git commit -m "docs: add chapter 04 tutorial infographics"
```

- [ ] **Step 7: Merge to main and push**

Run:

```bash
git fetch origin main
git checkout main
git merge --ff-only ch04-tutorial-infographics-spec
git push origin main
```

Expected: `main` fast-forwards and push succeeds.
