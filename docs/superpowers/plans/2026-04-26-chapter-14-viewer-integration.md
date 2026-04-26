# Chapter 14 Viewer Integration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Refresh Chapter 14 into a source-backed viewer/ecosystem chapter and integrate native-imagegen tutorial PNGs in the Chapter 03 / 04 visual style.

**Architecture:** Work in `.worktrees/ch14-viewer-integration-refresh` on branch `ch14-viewer-integration-refresh`. Because no old Chapter 14正文 branch or worktree exists, replace the skeleton with a new source-backed chapter, write a visual spec, generate one calibration PNG, batch-generate the remaining 23 PNGs, connect Markdown references, run path/image/style checks, request review, then commit and push.

**Tech Stack:** Git worktrees, Markdown, Newton source anchors from `/home/zhuzihou/dev/newton` at commit `0f583176`, native Codex image generation through `codex exec --enable image_generation`, PNG assets under `chapters/14_viewer_integration/assets/`, shell verification with `rg`, `find`, `file`, `identify`, and `git diff --check`.

---

## File Map

- Modify:
  - `chapters/14_viewer_integration/README.md`
- Create:
  - `chapters/14_viewer_integration/principle.md`
  - `chapters/14_viewer_integration/source-walkthrough.md`
  - `chapters/14_viewer_integration/examples.md`
  - `chapters/14_viewer_integration/pitfalls.md`
  - `chapters/14_viewer_integration/exercises.md`
  - `chapters/14_viewer_integration/assets/14_readme_viewer_boundary_spine.png`
  - `chapters/14_viewer_integration/assets/14_readme_file_roles_reading_order.png`
  - `chapters/14_viewer_integration/assets/14_readme_completion_scope_prereq.png`
  - `chapters/14_viewer_integration/assets/14_principle_not_solver_boundary.png`
  - `chapters/14_viewer_integration/assets/14_principle_outer_loop_two_lanes.png`
  - `chapters/14_viewer_integration/assets/14_principle_common_interface_map.png`
  - `chapters/14_viewer_integration/assets/14_principle_log_state_reads_state.png`
  - `chapters/14_viewer_integration/assets/14_principle_contacts_are_overlay.png`
  - `chapters/14_viewer_integration/assets/14_principle_input_writeback_timing.png`
  - `chapters/14_viewer_integration/assets/14_principle_backend_choice_table.png`
  - `chapters/14_viewer_integration/assets/14_principle_ecosystem_bridge_scope.png`
  - `chapters/14_viewer_integration/assets/14_walkthrough_pipeline_overview.png`
  - `chapters/14_viewer_integration/assets/14_walkthrough_beginner_path.png`
  - `chapters/14_viewer_integration/assets/14_walkthrough_stage1_cli_backend_selection.png`
  - `chapters/14_viewer_integration/assets/14_walkthrough_stage2_runner_loop.png`
  - `chapters/14_viewer_integration/assets/14_walkthrough_stage3_example_step_write_boundary.png`
  - `chapters/14_viewer_integration/assets/14_walkthrough_stage4_render_log_boundary.png`
  - `chapters/14_viewer_integration/assets/14_walkthrough_stage5_backend_lifecycle.png`
  - `chapters/14_viewer_integration/assets/14_walkthrough_stage6_debug_overlays.png`
  - `chapters/14_viewer_integration/assets/14_walkthrough_object_ledger_stop_here.png`
  - `chapters/14_viewer_integration/assets/14_examples_role_table.png`
  - `chapters/14_viewer_integration/assets/14_examples_basic_pendulum_anchor.png`
  - `chapters/14_viewer_integration/assets/14_pitfalls_boundary_mistakes.png`
  - `chapters/14_viewer_integration/assets/14_exercises_loop_annotation_quiz.png`
  - `docs/superpowers/specs/2026-04-26-chapter-14-viewer-integration-design.md`
  - `docs/superpowers/specs/2026-04-26-chapter-14-tutorial-infographics-design.md`
  - `docs/superpowers/plans/2026-04-26-chapter-14-viewer-integration.md`

## Task 1: Confirm Safe Starting Point

**Files:**
- Inspect: `chapters/14_viewer_integration/README.md`

- [x] **Step 1: Confirm main sync and worktree isolation**

Run:

```bash
git fetch origin main --prune
git status --short --branch
git rev-list --left-right --count main...origin/main
git worktree list
```

Expected: `main` is clean and even with `origin/main`, and the only new feature worktree is `.worktrees/ch14-viewer-integration-refresh`.

- [x] **Step 2: Confirm no old Chapter 14正文 exists**

Run:

```bash
git branch --all --list '*14*' '*viewer*' '*integration*'
find chapters -maxdepth 2 -type f | sort | rg '/14_|chapters/14'
```

Expected: no old branch output and only the skeleton Chapter 14 README on `main`.

- [x] **Step 3: Confirm source anchors**

Run:

```bash
git -C /home/zhuzihou/dev/newton rev-parse --short HEAD
find /home/zhuzihou/dev/newton/newton/_src/viewer -maxdepth 2 -type f | sort
```

Expected: upstream source commit `0f583176` and visible viewer backend files.

## Task 2: Write Chapter Content

**Files:**
- Modify: `chapters/14_viewer_integration/README.md`
- Create: `chapters/14_viewer_integration/principle.md`
- Create: `chapters/14_viewer_integration/source-walkthrough.md`
- Create: `chapters/14_viewer_integration/examples.md`
- Create: `chapters/14_viewer_integration/pitfalls.md`
- Create: `chapters/14_viewer_integration/exercises.md`

- [x] **Step 1: Rewrite README**

The README must establish:

```text
viewer 是读/记录/展示边界，不是第二套 physics。
```

It must include a file map, backend role table, first-pass scope, non-scope, prerequisites, and completion gate.

- [x] **Step 2: Create principle.md**

The principle page must explain:

```text
outer runner loop
-> optional input/write-before-step
-> solver owns physics update
-> render/log phase reads fresh State and Contacts
-> backend decides presentation/export/recording behavior
```

- [x] **Step 3: Create source-walkthrough.md**

The source walkthrough must include exact anchors for:

```text
newton/examples/__init__.py:L424-L428
newton/examples/__init__.py:L660-L678
newton/examples/__init__.py:L278-L299
newton/examples/basic/example_basic_pendulum.py:L95-L145
newton/_src/viewer/viewer.py:L442-L507
newton/_src/viewer/viewer.py:L599-L617
newton/_src/viewer/viewer_gl.py:L1433-L1445
newton/_src/viewer/viewer_usd.py:L67-L190
newton/_src/viewer/viewer_rerun.py:L418-L486
```

- [x] **Step 4: Create examples, pitfalls, and exercises**

The examples page must give each selected example one teaching role. Pitfalls must correct viewer/solver/contact/state-freshness confusion. Exercises must ask the reader to mark read and write boundaries.

- [x] **Step 5: Verify no skeleton prompts remain**

Run:

```bash
rg -n "T[B]D|TO[D]O|本章要回答什.问题|预期产.出|\\{\\{" chapters/14_viewer_integration docs/superpowers/specs/2026-04-26-chapter-14-viewer-integration-design.md docs/superpowers/plans/2026-04-26-chapter-14-viewer-integration.md
```

Expected: no output.

## Task 3: Write Chapter 14 Visual Spec

**Files:**
- Create: `docs/superpowers/specs/2026-04-26-chapter-14-tutorial-infographics-design.md`

- [x] **Step 1: Write the visual spec**

The spec must cite `docs/superpowers/specs/chapter-visual-style-guide.md`, require Chapter 03 / 04 style, and define the calibration image as:

```text
chapters/14_viewer_integration/assets/14_readme_viewer_boundary_spine.png
```

- [x] **Step 2: Enumerate all 24 assets**

The visual spec must enumerate the same 24 image filenames listed in this plan.

- [x] **Step 3: Self-review the visual spec**

Run:

```bash
rg -n "T[B]D|TO[D]O|SVG|HTML fallback|Chapter 02-style" docs/superpowers/specs/2026-04-26-chapter-14-tutorial-infographics-design.md
```

Expected: no starter-prompt hits; any `Chapter 02-style` hit must be in an explicit avoid statement.

## Task 4: Generate Calibration Image

**Files:**
- Create: `chapters/14_viewer_integration/assets/14_readme_viewer_boundary_spine.png`

- [x] **Step 1: Generate through native imagegen**

Run nested Codex with built-in image generation only. Prompt content:

```text
Create a clean Chinese tutorial infographic for Newton Learn chapter 14.
Use case: scientific-educational / infographic-diagram.
Asset: README calibration image, saved as chapters/14_viewer_integration/assets/14_readme_viewer_boundary_spine.png.
Style: white background, thin blue rounded card borders, dark-blue title, numbered blue badges, compact multi-panel teacher review-sheet layout, small friendly icons, arrows, flow boxes, two-lane read/write boundary, backend output cards, bottom recap strip, high information density but readable.
Content: show the viewer boundary spine: Model / State / Contacts already exist; optional viewer input apply_forces() writes before solver.step(); solver/collision updates State and Contacts; begin_frame(); log_state(), log_contacts(), log_gizmo() read/log current data; backend output choices: GL window, USD file, Rerun timeline, file recording, null test counter. Make read/log arrows visually distinct from write-before-step arrows.
Language: Chinese-first. English/code terms allowed only for exact names: ViewerBase, State, Contacts, apply_forces(), solver.step(), begin_frame(), log_state(), log_contacts(), log_gizmo(), ViewerGL, ViewerUSD, ViewerRerun, ViewerNull.
Avoid: fake code, fake terminal, long paragraphs, SVG, HTML, screenshot, photorealistic 3D, decorative-only scene, implying viewer is a solver or contact generator.
Generate raster PNG.
```

- [x] **Step 2: Verify calibration file**

Run:

```bash
file chapters/14_viewer_integration/assets/14_readme_viewer_boundary_spine.png
identify -format '%w %h\n' chapters/14_viewer_integration/assets/14_readme_viewer_boundary_spine.png
ls -lh chapters/14_viewer_integration/assets/14_readme_viewer_boundary_spine.png
```

Expected: PNG image, nonzero dimensions, reasonable file size.

- [x] **Step 3: Style check calibration**

Compare the image against:

```text
docs/superpowers/specs/chapter-visual-style-guide.md
docs/superpowers/specs/2026-04-24-chapter-03-tutorial-infographics-design.md
docs/superpowers/specs/2026-04-25-chapter-04-tutorial-infographics-design.md
```

Expected: dense Chinese tutorial handout, not decorative art.

## Task 5: Batch Generate Remaining Images

**Files:**
- Create: remaining 23 PNGs listed in the File Map.

- [x] **Step 1: Generate README batch**

Generate:

```text
14_readme_file_roles_reading_order.png
14_readme_completion_scope_prereq.png
```

- [x] **Step 2: Generate principle batch**

Generate:

```text
14_principle_not_solver_boundary.png
14_principle_outer_loop_two_lanes.png
14_principle_common_interface_map.png
14_principle_log_state_reads_state.png
14_principle_contacts_are_overlay.png
14_principle_input_writeback_timing.png
14_principle_backend_choice_table.png
14_principle_ecosystem_bridge_scope.png
```

- [x] **Step 3: Generate source-walkthrough batch**

Generate:

```text
14_walkthrough_pipeline_overview.png
14_walkthrough_beginner_path.png
14_walkthrough_stage1_cli_backend_selection.png
14_walkthrough_stage2_runner_loop.png
14_walkthrough_stage3_example_step_write_boundary.png
14_walkthrough_stage4_render_log_boundary.png
14_walkthrough_stage5_backend_lifecycle.png
14_walkthrough_stage6_debug_overlays.png
14_walkthrough_object_ledger_stop_here.png
```

- [x] **Step 4: Generate examples/pitfalls/exercises batch**

Generate:

```text
14_examples_role_table.png
14_examples_basic_pendulum_anchor.png
14_pitfalls_boundary_mistakes.png
14_exercises_loop_annotation_quiz.png
```

- [x] **Step 5: Verify all images exist**

Run:

```bash
find chapters/14_viewer_integration/assets -maxdepth 1 -type f -name '14_*.png' | sort | wc -l
find chapters/14_viewer_integration/assets -maxdepth 1 -type f -name '14_*.png' -exec file {} \;
```

Expected: exactly `24` PNG files, all recognized as PNG images.

## Task 6: Connect Images To Markdown

**Files:**
- Modify: all six Chapter 14 Markdown files.

- [x] **Step 1: Add README image references**

Add image links near chapter spine, file map, and completion/scope sections.

- [x] **Step 2: Add principle image references**

Add image links near each major conceptual section.

- [x] **Step 3: Add source-walkthrough image references**

Add image links near pipeline overview, beginner path, each stage, and object ledger.

- [x] **Step 4: Add examples, pitfalls, and exercises image references**

Add image links near the role table, anchor example, pitfalls list, and exercise worksheet.

- [x] **Step 5: Verify Markdown image references**

Run:

```bash
refs=$(rg -o 'assets/14_[^) ]+\.png' chapters/14_viewer_integration/*.md | sed 's/.*assets/assets/' | sort -u | wc -l); assets=$(find chapters/14_viewer_integration/assets -maxdepth 1 -type f -name '14_*.png' | wc -l); printf 'refs=%s assets=%s\n' "$refs" "$assets"
for f in $(rg -o 'assets/14_[^) ]+\.png' chapters/14_viewer_integration/*.md | sed 's/.*assets/assets/' | sort -u); do test -f "chapters/14_viewer_integration/$f" || echo "missing $f"; done
```

Expected: `refs=24 assets=24` and no `missing` output.

## Task 7: Verify, Review, Commit, Push

**Files:**
- Review all changed files.

- [x] **Step 1: Run repository-local checks**

Run:

```bash
git diff --check
git status --short
rg -n "T[B]D|TO[D]O|\\{\\{|本章要回答什.问题|预期产.出" chapters/14_viewer_integration docs/superpowers/specs/2026-04-26-chapter-14-*.md docs/superpowers/plans/2026-04-26-chapter-14-viewer-integration.md
```

Expected: `git diff --check` has no output, status only contains intended files, starter-prompt scan has no output.

- [x] **Step 2: Run visual/source-truth review**

Review every generated PNG against `chapter-visual-style-guide.md` and check Markdown prose keeps exact source truth. Any image with decorative drift, fake API, or viewer-as-solver implication must be regenerated or fenced by corrected prose.

- [x] **Step 3: Request code review**

Use the requesting-code-review workflow with base `origin/main` and current branch head. Review must check path references, content scope, visual style, and source-truth risks.

- [x] **Step 4: Commit**

Run:

```bash
git add chapters/14_viewer_integration docs/superpowers/specs/2026-04-26-chapter-14-viewer-integration-design.md docs/superpowers/specs/2026-04-26-chapter-14-tutorial-infographics-design.md docs/superpowers/plans/2026-04-26-chapter-14-viewer-integration.md
git commit -m "docs: refresh chapter 14 viewer integration"
```

- [x] **Step 5: Merge to main and push**

Run:

```bash
git -C /home/zhuzihou/dev/newton-learn checkout main
git -C /home/zhuzihou/dev/newton-learn pull --ff-only origin main
git -C /home/zhuzihou/dev/newton-learn merge --ff-only ch14-viewer-integration-refresh
git -C /home/zhuzihou/dev/newton-learn push origin main
```

Expected: `main` receives the Chapter 14 refresh and push succeeds.

- [x] **Step 6: Clean worktree**

Run:

```bash
git -C /home/zhuzihou/dev/newton-learn worktree remove .worktrees/ch14-viewer-integration-refresh
git -C /home/zhuzihou/dev/newton-learn branch -d ch14-viewer-integration-refresh
```

Expected: only the main worktree remains.
