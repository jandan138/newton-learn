# Chapter 15 Multiphysics Pipeline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Refresh Chapter 15 into a complete source-backed multiphysics coupling chapter with native-imagegen tutorial PNGs.

**Architecture:** Work in `.worktrees/ch15-multiphysics-pipeline-refresh` on branch `ch15-multiphysics-pipeline-refresh`. Because no old Chapter 15正文 branch or worktree exists, replace the skeleton with a new source-backed chapter, write a visual spec, generate one calibration PNG, batch-generate the remaining 23 PNGs, connect Markdown references, run path/image/style checks, request review, then commit and push.

**Tech Stack:** Markdown docs, native Codex image generation, local Newton source at `/home/zhuzihou/dev/newton` commit `0f583176`, git worktree workflow.

---

## File Structure

- Modify: `chapters/15_multiphysics_pipeline/README.md`
- Create: `chapters/15_multiphysics_pipeline/principle.md`
- Create: `chapters/15_multiphysics_pipeline/source-walkthrough.md`
- Create: `chapters/15_multiphysics_pipeline/examples.md`
- Create: `chapters/15_multiphysics_pipeline/pitfalls.md`
- Create: `chapters/15_multiphysics_pipeline/exercises.md`
- Create: `chapters/15_multiphysics_pipeline/assets/*.png`
- Create: `docs/superpowers/specs/2026-04-26-chapter-15-multiphysics-pipeline-design.md`
- Create: `docs/superpowers/specs/2026-04-26-chapter-15-tutorial-infographics-design.md`
- Create: `docs/superpowers/plans/2026-04-26-chapter-15-multiphysics-pipeline.md`

## Asset Checklist

The final asset set is:

- `chapters/15_multiphysics_pipeline/assets/15_readme_multiphysics_boundary_map.png`
- `chapters/15_multiphysics_pipeline/assets/15_readme_file_roles_reading_order.png`
- `chapters/15_multiphysics_pipeline/assets/15_readme_completion_scope_prereq.png`
- `chapters/15_multiphysics_pipeline/assets/15_principle_one_model_vs_two_models.png`
- `chapters/15_multiphysics_pipeline/assets/15_principle_coupling_contract_ladder.png`
- `chapters/15_multiphysics_pipeline/assets/15_principle_builder_material_ledger.png`
- `chapters/15_multiphysics_pipeline/assets/15_principle_contacts_are_coupling_surface.png`
- `chapters/15_multiphysics_pipeline/assets/15_principle_vbd_single_solver_loop.png`
- `chapters/15_multiphysics_pipeline/assets/15_principle_mpm_twomodel_impulse_bridge.png`
- `chapters/15_multiphysics_pipeline/assets/15_principle_external_rigid_solver_bridge.png`
- `chapters/15_multiphysics_pipeline/assets/15_principle_roadmap_scope_table.png`
- `chapters/15_multiphysics_pipeline/assets/15_walkthrough_pipeline_overview.png`
- `chapters/15_multiphysics_pipeline/assets/15_walkthrough_beginner_path.png`
- `chapters/15_multiphysics_pipeline/assets/15_walkthrough_stage1_example_inventory.png`
- `chapters/15_multiphysics_pipeline/assets/15_walkthrough_stage2_softbody_cloth_build.png`
- `chapters/15_multiphysics_pipeline/assets/15_walkthrough_stage3_vbd_step_contract.png`
- `chapters/15_multiphysics_pipeline/assets/15_walkthrough_stage4_gift_geometry_and_contacts.png`
- `chapters/15_multiphysics_pipeline/assets/15_walkthrough_stage5_mpm_twoway_bridge.png`
- `chapters/15_multiphysics_pipeline/assets/15_walkthrough_stage6_cloth_franka_bridge.png`
- `chapters/15_multiphysics_pipeline/assets/15_walkthrough_object_ledger_stop_here.png`
- `chapters/15_multiphysics_pipeline/assets/15_examples_role_table.png`
- `chapters/15_multiphysics_pipeline/assets/15_examples_softbody_gift_anchor.png`
- `chapters/15_multiphysics_pipeline/assets/15_pitfalls_coupling_mistakes.png`
- `chapters/15_multiphysics_pipeline/assets/15_exercises_coupling_boundary_quiz.png`

## Execution Checklist

### Task 1: Preflight and Source Anchors

- [x] **Step 1: Sync main and create worktree**

Run:

```bash
git fetch origin main --prune
git worktree add .worktrees/ch15-multiphysics-pipeline-refresh -b ch15-multiphysics-pipeline-refresh
```

Expected: feature worktree exists and branch is clean.

- [x] **Step 2: Confirm no old Chapter 15正文 exists**

Run:

```bash
git branch --all --list '*15*' '*multi*' '*multiphysics*' '*pipeline*'
find chapters/15_multiphysics_pipeline -maxdepth 2 -type f -print
```

Expected: no old branch output and only skeleton README on `main`.

- [x] **Step 3: Confirm source anchors**

Run:

```bash
git -C /home/zhuzihou/dev/newton rev-parse --short HEAD
find /home/zhuzihou/dev/newton/newton/examples/multiphysics -maxdepth 1 -type f -print
```

Expected: `0f583176` and two multiphysics files.

### Task 2: Chapter Docs and Specs

- [x] **Step 1: Rewrite README**

Create a source-backed Chapter 15 overview with coupling categories and 3 README images.

- [x] **Step 2: Create principle.md**

Write one-model vs two-model classification, builder/runtime boundary, contacts as coupling surface, VBD, MPM bridge, cloth-Franka bridge, roadmap boundary.

- [x] **Step 3: Create source-walkthrough.md**

Write current inventory, softbody/cloth build, VBD contract, gift example, MPM bridge, cloth-Franka bridge, object ledger.

- [x] **Step 4: Create examples, pitfalls, and exercises**

Write example role table, common mistakes, and self-check exercises.

- [x] **Step 5: Write design and visual specs**

Create both specs under `docs/superpowers/specs/` and this plan under `docs/superpowers/plans/`.

### Task 3: Calibration Image

- [ ] **Step 1: Generate calibration image through native imagegen**

Create:

```text
chapters/15_multiphysics_pipeline/assets/15_readme_multiphysics_boundary_map.png
```

Prompt goal: three lanes for single-model VBD, two-model MPM bridge, and external robot/cloth bridge.

- [ ] **Step 2: Verify calibration file**

Run:

```bash
file chapters/15_multiphysics_pipeline/assets/15_readme_multiphysics_boundary_map.png
ls -lh chapters/15_multiphysics_pipeline/assets/15_readme_multiphysics_boundary_map.png
```

Expected: PNG image data and nontrivial file size.

- [ ] **Step 3: Style/source check calibration**

Check that it matches `chapter-visual-style-guide.md`, is concrete, and does not imply generic implicit co-simulation.

### Task 4: Batch Generate Remaining PNGs

- [ ] **Step 1: Generate README and principle batch**

Generate the remaining README images and all 8 principle images. Use native image generation, one image per asset, no SVG/HTML fallback.

- [ ] **Step 2: Generate source-walkthrough batch**

Generate all 9 walkthrough images. Use native image generation, one image per asset, no SVG/HTML fallback.

- [ ] **Step 3: Generate examples/pitfalls/exercises batch**

Generate the 4 remaining auxiliary images. Use native image generation, one image per asset, no SVG/HTML fallback.

- [ ] **Step 4: Verify all images exist**

Run:

```bash
find chapters/15_multiphysics_pipeline/assets -maxdepth 1 -type f -name '15_*.png' | sort | wc -l
find chapters/15_multiphysics_pipeline/assets -maxdepth 1 -type f -name '15_*.png' -exec file {} \;
```

Expected: 24 PNG files.

### Task 5: Markdown Reference Verification

- [ ] **Step 1: Verify references and assets**

Run:

```bash
refs=$(rg -o 'assets/15_[^) ]+\.png' chapters/15_multiphysics_pipeline/*.md | sed 's/.*assets/assets/' | sort -u | wc -l)
assets=$(find chapters/15_multiphysics_pipeline/assets -maxdepth 1 -type f -name '15_*.png' | wc -l)
printf 'refs=%s assets=%s\n' "$refs" "$assets"
for f in $(rg -o 'assets/15_[^) ]+\.png' chapters/15_multiphysics_pipeline/*.md | sed 's/.*assets/assets/' | sort -u); do
  test -f "chapters/15_multiphysics_pipeline/$f" || echo "missing $f"
done
```

Expected: `refs=24 assets=24` and no missing output.

- [ ] **Step 2: Scan for skeleton starter text**

Run:

```bash
rg -n 'T[B]D|T[O]DO|place''holder|本章要回答什[么]|预期产[出]|\{\{' chapters/15_multiphysics_pipeline docs/superpowers/specs/2026-04-26-chapter-15-multiphysics-pipeline-design.md docs/superpowers/specs/2026-04-26-chapter-15-tutorial-infographics-design.md docs/superpowers/plans/2026-04-26-chapter-15-multiphysics-pipeline.md
```

Expected: no output.

### Task 6: Review, Commit, Push

- [ ] **Step 1: Run git checks**

Run:

```bash
git diff --check
git status --short --branch
```

Expected: no whitespace errors and only intentional Chapter 15 files changed.

- [ ] **Step 2: Run visual/source-truth review**

Inspect representative images and review Markdown against current source anchors. Reject or regenerate any image that contradicts the source boundary.

- [ ] **Step 3: Request code review**

Ask a reviewer to check source-truth claims, image wiring, starter-text remnants, and spec alignment. Fix concrete findings.

- [ ] **Step 4: Commit**

Run:

```bash
git add chapters/15_multiphysics_pipeline docs/superpowers/specs/2026-04-26-chapter-15-multiphysics-pipeline-design.md docs/superpowers/specs/2026-04-26-chapter-15-tutorial-infographics-design.md docs/superpowers/plans/2026-04-26-chapter-15-multiphysics-pipeline.md
git commit -m "docs: refresh chapter 15 multiphysics pipeline"
```

- [ ] **Step 5: Merge to main and push**

Run:

```bash
git -C /home/zhuzihou/dev/newton-learn pull --ff-only origin main
git -C /home/zhuzihou/dev/newton-learn merge --ff-only ch15-multiphysics-pipeline-refresh
git -C /home/zhuzihou/dev/newton-learn push origin main
```

- [ ] **Step 6: Clean worktree**

Run:

```bash
git -C /home/zhuzihou/dev/newton-learn worktree remove .worktrees/ch15-multiphysics-pipeline-refresh
git -C /home/zhuzihou/dev/newton-learn branch -d ch15-multiphysics-pipeline-refresh
```

Expected: main clean and only main worktree remains.
