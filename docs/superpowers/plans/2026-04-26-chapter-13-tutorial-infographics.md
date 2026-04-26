# Chapter 13 Tutorial Infographics Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Safely refresh Chapter 13 on latest `main`, then generate and connect native-imagegen PNG tutorial infographics that follow the Chapter 03 / 04 visual style.

**Architecture:** Work from a fresh `ch13-diffsim-refresh` worktree based on `main`. Restore only Chapter 13正文 and old Chapter 13 design/plan files from `ch13-diffsim`, then add a new image spec, generate a calibration image, batch-generate 26 remaining images, connect Markdown references, and verify paths/style/source-truth before commit and push.

**Tech Stack:** Git worktrees, Markdown, native Codex image generation through `codex exec --enable image_generation`, PNG assets under `chapters/13_diffsim/assets/`, shell verification with `rg`, `find`, `file`, `identify`, and `git diff --check`.

---

## File Map

- Modify:
  - `chapters/13_diffsim/README.md`
  - `chapters/13_diffsim/principle.md`
  - `chapters/13_diffsim/source-walkthrough.md`
  - `chapters/13_diffsim/examples.md`
  - `chapters/13_diffsim/pitfalls.md`
  - `chapters/13_diffsim/exercises.md`
- Create:
  - `chapters/13_diffsim/assets/13_readme_verify_before_optimize_spine.png`
  - `chapters/13_diffsim/assets/13_readme_file_roles_reading_order.png`
  - `chapters/13_diffsim/assets/13_readme_example_roles_completion_gate.png`
  - `chapters/13_diffsim/assets/13_principle_trust_first_title.png`
  - `chapters/13_diffsim/assets/13_principle_requires_grad_ledger.png`
  - `chapters/13_diffsim/assets/13_principle_tape_loss_backward_chain.png`
  - `chapters/13_diffsim/assets/13_principle_parameter_placement_axis.png`
  - `chapters/13_diffsim/assets/13_principle_ball_state_loop.png`
  - `chapters/13_diffsim/assets/13_principle_fd_trust_gate.png`
  - `chapters/13_diffsim/assets/13_principle_advanced_defer_map.png`
  - `chapters/13_diffsim/assets/13_walkthrough_pipeline_overview.png`
  - `chapters/13_diffsim/assets/13_walkthrough_beginner_path.png`
  - `chapters/13_diffsim/assets/13_walkthrough_stage1_requires_grad.png`
  - `chapters/13_diffsim/assets/13_walkthrough_stage2_tape_loss.png`
  - `chapters/13_diffsim/assets/13_walkthrough_stage3_backward_update.png`
  - `chapters/13_diffsim/assets/13_walkthrough_stage4_fd_trust_gate.png`
  - `chapters/13_diffsim/assets/13_walkthrough_stage5_spring_cage_model_param.png`
  - `chapters/13_diffsim/assets/13_walkthrough_stage6_soft_body_external_buffer.png`
  - `chapters/13_diffsim/assets/13_walkthrough_object_ledger_stop_here.png`
  - `chapters/13_diffsim/assets/13_examples_role_table.png`
  - `chapters/13_diffsim/assets/13_examples_ball_anchor.png`
  - `chapters/13_diffsim/assets/13_examples_parameter_branches.png`
  - `chapters/13_diffsim/assets/13_examples_advanced_defer.png`
  - `chapters/13_diffsim/assets/13_pitfalls_verification_traps.png`
  - `chapters/13_diffsim/assets/13_pitfalls_contact_scope_and_fallback.png`
  - `chapters/13_diffsim/assets/13_exercises_fd_error_table.png`
  - `chapters/13_diffsim/assets/13_exercises_validation_quiz.png`
  - `docs/superpowers/specs/2026-04-26-chapter-13-tutorial-infographics-design.md`
  - `docs/superpowers/plans/2026-04-26-chapter-13-tutorial-infographics.md`
- Preserve:
  - `docs/superpowers/specs/2026-04-22-chapter-13-diffsim-design.md`
  - `docs/superpowers/plans/2026-04-22-chapter-13-diffsim.md`

## Task 1: Establish Safe Refresh Branch

**Files:**
- Restore: `chapters/13_diffsim/*.md`
- Restore: `docs/superpowers/specs/2026-04-22-chapter-13-diffsim-design.md`
- Restore: `docs/superpowers/plans/2026-04-22-chapter-13-diffsim.md`

- [ ] **Step 1: Confirm latest main is clean**

Run:

```bash
git fetch origin main --prune
git status --short --branch
git rev-list --left-right --count main...origin/main
```

Expected: clean `main` with `0 0` ahead/behind.

- [ ] **Step 2: Create isolated worktree**

Run:

```bash
git worktree add .worktrees/ch13-diffsim-refresh -b ch13-diffsim-refresh main
```

Expected: worktree checked out at latest `main`.

- [ ] **Step 3: Restore only Chapter 13 paths**

Run inside `.worktrees/ch13-diffsim-refresh`:

```bash
git restore --source ch13-diffsim -- \
  chapters/13_diffsim \
  docs/superpowers/plans/2026-04-22-chapter-13-diffsim.md \
  docs/superpowers/specs/2026-04-22-chapter-13-diffsim-design.md
```

Expected: no 00-12 asset deletions and no visual guide rollback.

- [ ] **Step 4: Verify migration scope**

Run:

```bash
git status --short
git diff --name-status -- . ':(exclude)chapters/13_diffsim' ':(exclude)docs/superpowers/plans/2026-04-22-chapter-13-diffsim.md' ':(exclude)docs/superpowers/specs/2026-04-22-chapter-13-diffsim-design.md'
```

Expected: the first command only shows Chapter 13正文 and two old Chapter 13 support docs; the second command prints nothing.

## Task 2: Write Image Spec And Plan

**Files:**
- Create: `docs/superpowers/specs/2026-04-26-chapter-13-tutorial-infographics-design.md`
- Create: `docs/superpowers/plans/2026-04-26-chapter-13-tutorial-infographics.md`

- [ ] **Step 1: Write the Chapter 13 image spec**

The spec must cite `docs/superpowers/specs/chapter-visual-style-guide.md`, require Chapter 03 / 04 style, and define the calibration image as:

```text
chapters/13_diffsim/assets/13_readme_verify_before_optimize_spine.png
```

- [ ] **Step 2: Write the execution plan**

The plan must enumerate all 27 assets and the six target Markdown files.

- [ ] **Step 3: Self-review the spec and plan**

Run:

```bash
rg -n "TBD|TODO|placeholder|SVG|HTML fallback|Chapter 02-style" docs/superpowers/specs/2026-04-26-chapter-13-tutorial-infographics-design.md docs/superpowers/plans/2026-04-26-chapter-13-tutorial-infographics.md
```

Expected: no placeholder hits; any `Chapter 02-style` hit must be in an explicit avoid statement.

## Task 3: Generate Calibration Image

**Files:**
- Create: `chapters/13_diffsim/assets/13_readme_verify_before_optimize_spine.png`

- [ ] **Step 1: Generate through native imagegen**

Run nested Codex with built-in image generation only. Prompt content:

```text
Create a clean Chinese tutorial infographic for Newton Learn chapter 13.
Use case: scientific-educational / infographic-diagram.
Asset: README calibration image, saved as chapters/13_diffsim/assets/13_readme_verify_before_optimize_spine.png.
Style: white background, thin blue rounded card borders, dark-blue title, numbered blue badges, compact multi-panel teacher review-sheet layout, small friendly icons, arrows, flow boxes, trust-gate checkpoint, bottom recap strip, high information density but readable.
Content: show the trust-first DiffSim spine: rollout miss -> choose upstream parameter -> requires_grad=True -> wp.Tape() records forward -> scalar loss -> backward() -> gradient candidate on upstream parameter -> FD sweep / trust window -> update loop. Add concrete drawings: target marker, parameter ledger, Tape box, FD gate, update arrow.
Language: Chinese-first. English/code terms allowed only for exact names: requires_grad=True, wp.Tape(), backward(), FD, loss, grad, update.
Avoid: fake code, fake terminal, long paragraphs, SVG, HTML, screenshot, photorealistic 3D, decorative-only scene, implying backward() proves correctness.
Generate raster PNG.
```

- [ ] **Step 2: Verify calibration file**

Run:

```bash
file chapters/13_diffsim/assets/13_readme_verify_before_optimize_spine.png
identify -format '%w %h\n' chapters/13_diffsim/assets/13_readme_verify_before_optimize_spine.png
ls -lh chapters/13_diffsim/assets/13_readme_verify_before_optimize_spine.png
```

Expected: PNG image, nonzero dimensions, reasonable file size.

- [ ] **Step 3: Style check calibration**

Open or inspect the image and compare against:

```text
docs/superpowers/specs/chapter-visual-style-guide.md
docs/superpowers/specs/2026-04-24-chapter-03-tutorial-infographics-design.md
docs/superpowers/specs/2026-04-25-chapter-04-tutorial-infographics-design.md
```

Expected: dense Chinese tutorial handout, not decorative art.

## Task 4: Batch Generate Remaining Images

**Files:**
- Create: remaining 26 PNGs listed in the File Map.

- [ ] **Step 1: Generate README batch**

Generate:

```text
13_readme_file_roles_reading_order.png
13_readme_example_roles_completion_gate.png
```

Use the same style as the calibration image and keep labels short.

- [ ] **Step 2: Generate principle batch**

Generate:

```text
13_principle_trust_first_title.png
13_principle_requires_grad_ledger.png
13_principle_tape_loss_backward_chain.png
13_principle_parameter_placement_axis.png
13_principle_ball_state_loop.png
13_principle_fd_trust_gate.png
13_principle_advanced_defer_map.png
```

Each prompt must focus on one concept and avoid invented code.

- [ ] **Step 3: Generate source-walkthrough batch**

Generate:

```text
13_walkthrough_pipeline_overview.png
13_walkthrough_beginner_path.png
13_walkthrough_stage1_requires_grad.png
13_walkthrough_stage2_tape_loss.png
13_walkthrough_stage3_backward_update.png
13_walkthrough_stage4_fd_trust_gate.png
13_walkthrough_stage5_spring_cage_model_param.png
13_walkthrough_stage6_soft_body_external_buffer.png
13_walkthrough_object_ledger_stop_here.png
```

Each prompt must map directly to the nearby walkthrough stage.

- [ ] **Step 4: Generate examples / pitfalls / exercises batch**

Generate:

```text
13_examples_role_table.png
13_examples_ball_anchor.png
13_examples_parameter_branches.png
13_examples_advanced_defer.png
13_pitfalls_verification_traps.png
13_pitfalls_contact_scope_and_fallback.png
13_exercises_fd_error_table.png
13_exercises_validation_quiz.png
```

Keep the images worksheet-like and review-sheet-like.

- [ ] **Step 5: Verify generated PNG set**

Run:

```bash
find chapters/13_diffsim/assets -maxdepth 1 -type f -name '13_*.png' | sort | wc -l
find chapters/13_diffsim/assets -maxdepth 1 -type f -name '13_*.png' -exec file {} \;
find chapters/13_diffsim/assets -maxdepth 1 -type f -name '13_*.png' -size -10k -print
```

Expected: 27 PNGs, all recognized as PNG, no suspicious tiny files.

## Task 5: Integrate Markdown References

**Files:**
- Modify: `chapters/13_diffsim/README.md`
- Modify: `chapters/13_diffsim/principle.md`
- Modify: `chapters/13_diffsim/source-walkthrough.md`
- Modify: `chapters/13_diffsim/examples.md`
- Modify: `chapters/13_diffsim/pitfalls.md`
- Modify: `chapters/13_diffsim/exercises.md`

- [ ] **Step 1: Insert README images**

Add images after the main intro, file responsibilities, and example/completion sections:

```markdown
![Chapter 13 verify before optimize spine](assets/13_readme_verify_before_optimize_spine.png)
![Chapter 13 file roles and reading order](assets/13_readme_file_roles_reading_order.png)
![Chapter 13 example roles and completion gate](assets/13_readme_example_roles_completion_gate.png)
```

- [ ] **Step 2: Insert principle images**

Place the seven principle images near their matching sections:

```markdown
![Chapter 13 trust-first title correction](assets/13_principle_trust_first_title.png)
![Chapter 13 requires_grad ledger](assets/13_principle_requires_grad_ledger.png)
![Chapter 13 Tape loss backward chain](assets/13_principle_tape_loss_backward_chain.png)
![Chapter 13 parameter placement axis](assets/13_principle_parameter_placement_axis.png)
![Chapter 13 ball state parameter loop](assets/13_principle_ball_state_loop.png)
![Chapter 13 FD trust gate](assets/13_principle_fd_trust_gate.png)
![Chapter 13 advanced branch defer map](assets/13_principle_advanced_defer_map.png)
```

- [ ] **Step 3: Insert walkthrough images**

Place the nine walkthrough images near their matching map, beginner path, stages, and ledger sections.

- [ ] **Step 4: Insert examples / pitfalls / exercises images**

Place the four examples images, two pitfalls images, and two exercise images near their matching sections.

- [ ] **Step 5: Verify Markdown image references**

Run:

```bash
rg -n '!\[.*\]\((assets/13_[^)]+\.png)\)' chapters/13_diffsim
```

Expected: all 27 image references appear in Chapter 13 Markdown.

## Task 6: Verification, Review, Commit, Push

**Files:**
- Verify: all changed files.

- [ ] **Step 1: Path reference verification**

Run a path check that extracts all `assets/13_*.png` references from `chapters/13_diffsim/*.md` and confirms the files exist.

Expected: no missing refs.

- [ ] **Step 2: Scope verification**

Run:

```bash
git status --short
git diff --name-status
git diff --check
```

Expected: changes are limited to Chapter 13 and Chapter 13 support docs; diff check clean.

- [ ] **Step 3: Style and source-truth review**

Review the generated PNG set against:

```text
docs/superpowers/specs/chapter-visual-style-guide.md
docs/superpowers/specs/2026-04-26-chapter-13-tutorial-infographics-design.md
```

Expected: no Chapter 02 diorama drift, no fake code, no unsupported DiffSim claims.

- [ ] **Step 4: Commit safe migration and image pass**

Use focused commits:

```bash
git add chapters/13_diffsim docs/superpowers/plans/2026-04-22-chapter-13-diffsim.md docs/superpowers/specs/2026-04-22-chapter-13-diffsim-design.md docs/superpowers/specs/2026-04-26-chapter-13-tutorial-infographics-design.md docs/superpowers/plans/2026-04-26-chapter-13-tutorial-infographics.md
git commit -m "docs: refresh chapter 13 diffsim content"
git commit -m "docs: add chapter 13 tutorial infographics"
```

If all work is already staged together, split commits by path before committing.

- [ ] **Step 5: Merge to main and push**

After verification and review:

```bash
git checkout main
git merge --ff-only ch13-diffsim-refresh
git push origin main
```

Expected: `main` advances cleanly and remote push succeeds.
