# Chapter 16 Self Experiments Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Refresh Chapter 16 into a complete source-backed self-experiment protocol chapter with native imagegen tutorial PNGs.

**Architecture:** Treat Chapter 16 as the closing practice layer for Chapters 13-15. The text stays source-backed against Newton commit `0f583176`; images are explanatory PNG learning aids and never the source of truth.

**Tech Stack:** Markdown docs, Newton source references, Codex native image generation, git worktree workflow.

---

## File Structure

- Modify: `chapters/16_self_experiments/README.md`
- Create: `chapters/16_self_experiments/principle.md`
- Create: `chapters/16_self_experiments/source-walkthrough.md`
- Create: `chapters/16_self_experiments/examples.md`
- Create: `chapters/16_self_experiments/pitfalls.md`
- Create: `chapters/16_self_experiments/exercises.md`
- Create: `chapters/16_self_experiments/assets/*.png`
- Create: `docs/superpowers/specs/2026-04-26-chapter-16-self-experiments-design.md`
- Create: `docs/superpowers/specs/2026-04-26-chapter-16-tutorial-infographics-design.md`
- Create: `docs/superpowers/plans/2026-04-26-chapter-16-self-experiments.md`

## Task 1: Preflight And Source Anchors

- [x] Confirm branch/worktree isolation from latest `origin/main`.
- [x] Verify `.worktrees/` is ignored.
- [x] Check for old Chapter 16 branch/worktree/body text.
- [x] Inspect current `chapters/16_self_experiments/README.md`.
- [x] Inspect Newton source root `/home/zhuzihou/dev/newton` and confirm commit `0f583176`.
- [x] Gather source anchors for `basic_pendulum`, `examples.__init__`, `State`, `Model`, `ViewerBase`, `basic_shapes`, `basic_plotting`, `diffsim_ball`, `softbody_dropping_to_cloth`, and `mpm_twoway_coupling`.

## Task 2: Write Chapter 16 Design And Plan

- [x] Write `docs/superpowers/specs/2026-04-26-chapter-16-self-experiments-design.md`.
- [x] Write `docs/superpowers/specs/2026-04-26-chapter-16-tutorial-infographics-design.md`.
- [x] Write this implementation plan.
- [x] Verify there are no design skeleton markers or contradictions.

Verification command:

```bash
rg -n "T[B]D|T[O]DO|placehol[d]er|本章要回答什[么]|预期产[出]|\\{\\{" \
  docs/superpowers/specs/2026-04-26-chapter-16-self-experiments-design.md \
  docs/superpowers/specs/2026-04-26-chapter-16-tutorial-infographics-design.md \
  docs/superpowers/plans/2026-04-26-chapter-16-self-experiments.md || true
```

Expected: no output.

## Task 3: Write Reader-Facing Markdown

- [x] Rewrite `chapters/16_self_experiments/README.md`.
- [x] Create `principle.md`.
- [x] Create `source-walkthrough.md`.
- [x] Create `examples.md`.
- [x] Create `pitfalls.md`.
- [x] Create `exercises.md`.
- [x] Verify all six files include frontmatter, source paths, and image anchors.

Verification command:

```bash
find chapters/16_self_experiments -maxdepth 1 -type f -name '*.md' -printf '%f\n' | sort
rg -n "assets/16_.*\\.png" chapters/16_self_experiments/*.md
```

Expected: six Markdown files and 24 image references.

## Task 4: Generate Native Imagegen PNGs

- [x] Generate calibration image `16_readme_experiment_loop_spine.png`.
- [x] Inspect calibration image against `chapter-visual-style-guide.md`.
- [x] Generate README and principle batch.
- [x] Generate source-walkthrough batch.
- [x] Generate examples / pitfalls / exercises batch.
- [x] Verify all assets are raster PNG files.

Expected assets:

```text
16_readme_experiment_loop_spine.png
16_readme_file_roles_reading_order.png
16_readme_completion_scope_prereq.png
16_principle_not_new_mainline_boundary.png
16_principle_question_to_protocol_ladder.png
16_principle_official_example_baseline.png
16_principle_one_variable_contract.png
16_principle_evidence_ledger.png
16_principle_metric_family_map.png
16_principle_findings_feedback_loop.png
16_principle_experiment_cluster_map.png
16_walkthrough_pipeline_overview.png
16_walkthrough_beginner_path.png
16_walkthrough_stage1_choose_claim.png
16_walkthrough_stage2_clone_baseline_example.png
16_walkthrough_stage3_freeze_conditions.png
16_walkthrough_stage4_change_one_knob.png
16_walkthrough_stage5_record_metrics.png
16_walkthrough_stage6_compare_and_verdict.png
16_walkthrough_object_ledger_stop_here.png
16_examples_experiment_role_table.png
16_examples_fd_validation_anchor.png
16_pitfalls_experiment_mistakes.png
16_exercises_protocol_design_quiz.png
```

Verification command:

```bash
find chapters/16_self_experiments/assets -maxdepth 1 -type f -name '16_*.png' -exec file {} + | rg -v 'PNG image data' || true
find chapters/16_self_experiments/assets -maxdepth 1 -type f -name '16_*.png' | wc -l
```

Expected: first command no output; second command `24`.

## Task 5: Path And Style Review

- [x] Verify every Markdown image reference resolves to an existing PNG.
- [x] Verify no unreferenced `16_*.png` asset remains.
- [x] Inspect representative images: README calibration, a principle diagram, a walkthrough stage, pitfalls, exercises.
- [x] Request independent review for source truth, path correctness, and visual style.
- [x] Fix Critical/Important issues and any easy Minor wording issues.

Verification command:

```bash
refs=$(mktemp)
assets=$(mktemp)
rg -o 'assets/16_[^) ]+\.png' chapters/16_self_experiments/*.md | sed 's#^.*assets/#assets/#' | sort -u > "$refs"
find chapters/16_self_experiments/assets -maxdepth 1 -type f -name '16_*.png' -printf 'assets/%f\n' | sort > "$assets"
echo refs=$(wc -l < "$refs") assets=$(wc -l < "$assets")
echo missing_assets
comm -23 "$refs" "$assets"
echo unreferenced_assets
comm -13 "$refs" "$assets"
rm "$refs" "$assets"
```

Expected: `refs=24 assets=24`, no missing or unreferenced output.

## Task 6: Final Verification, Commit, Push

- [x] Run skeleton-marker scan.
- [x] Run `git diff --check`.
- [x] Run image/reference checks.
- [ ] Commit with message `docs: refresh chapter 16 self experiments`.
- [ ] Fast-forward merge into main.
- [ ] Push `origin main`.
- [ ] Remove temporary worktree and local branch.

Final verification commands:

```bash
git status --short --branch
git diff --check
rg -n "T[B]D|T[O]DO|placehol[d]er|本章要回答什[么]|预期产[出]|\\{\\{" \
  chapters/16_self_experiments \
  docs/superpowers/specs/2026-04-26-chapter-16-self-experiments-design.md \
  docs/superpowers/specs/2026-04-26-chapter-16-tutorial-infographics-design.md \
  docs/superpowers/plans/2026-04-26-chapter-16-self-experiments.md || true
```

Expected:

- status shows only intended Chapter 16 and docs/superpowers files before commit.
- diff check has no output.
- skeleton-marker scan has no output.
