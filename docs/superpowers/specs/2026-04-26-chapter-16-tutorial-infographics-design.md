# Chapter 16 Tutorial Infographics Design

## Goal

Generate and integrate a complete set of Chapter 16 tutorial-style PNG images for the reader-facing chapter pages, using the dense Chinese learning-handout visual direction established by Chapters 03, 04, 13, 14, and 15 and governed by `docs/superpowers/specs/chapter-visual-style-guide.md`.

Chapter 16 should teach self-experiments as source-backed mini loops: one question, one variable, fixed conditions, source-of-truth evidence, written findings. It should not look like a lab photo, a benchmark report, a terminal transcript, or a new Newton demo catalog.

## Scope

In scope:

- `chapters/16_self_experiments/README.md`
- `chapters/16_self_experiments/principle.md`
- `chapters/16_self_experiments/source-walkthrough.md`
- `chapters/16_self_experiments/examples.md`
- `chapters/16_self_experiments/pitfalls.md`
- `chapters/16_self_experiments/exercises.md`
- `chapters/16_self_experiments/assets/`

Out of scope:

- SVG, HTML fallback, screenshots, fake code, fake terminal UI, fake benchmark values.
- Decorative 3D simulation scenes that do not teach protocol boundaries.
- Images that imply viewer output proves correctness.
- Images that imply every experiment needs a new code repository.

## Teaching Spine

Every image should reinforce:

```text
question
-> official baseline
-> fixed conditions
-> one changed knob
-> state/scalar/buffer/FD evidence
-> verdict
-> findings feedback
```

The one-line memory hook is:

```text
自制实验不是另起主线；它只验证一个可复现的问题。
```

## Visual Language

Follow `docs/superpowers/specs/chapter-visual-style-guide.md`.

Use:

- white background with thin blue rounded card borders
- dark-blue title and section headings
- numbered blue badges for protocol stages, file roles, evidence objects, and mistakes
- Chinese-first labels and explanations
- short exact English/code labels only when necessary: `newton/examples/`, `State`, `Contacts`, `why.md`, `findings.md`, `FD`, `seed`, `wp.Tape`, `wp.ScopedTimer`, `CUDA graph`, `solver.step()`
- compact cards, arrows, evidence ledgers, protocol checklists, verdict gates, and bottom recap bars
- concrete icons: question card, official example folder, slider knob, lock/freeze condition, state buffer cylinder, checkmark predicate, chart/scalar, FD ruler, findings notebook

Avoid:

- Chapter 02-style 3D diorama rendering
- photorealistic screenshots or fake simulation result charts
- fake source code, fake commands, fake terminal snippets, or invented filenames/API names
- long paragraphs inside images
- tiny code-heavy tables
- diagrams implying a single screenshot proves correctness
- diagrams implying any unrun benchmark or metric value

## Coverage Strategy

Use 24 PNGs. Each image should have one concrete teaching job.

### README

Create three entry-map images:

1. `16_readme_experiment_loop_spine.png`: calibration image. Shows the Chapter 16 closed loop from source-backed question to findings feedback.
2. `16_readme_file_roles_reading_order.png`: file responsibilities and recommended reading order, including optional `why.md` / `findings.md` experiment notes.
3. `16_readme_completion_scope_prereq.png`: completion gate, scope/non-scope, prerequisites, and the boundary that experiments do not replace the main chapters.

### principle.md

Create eight conceptual images:

1. `16_principle_not_new_mainline_boundary.png`: Chapter 16 as practice layer, not a new physics mainline.
2. `16_principle_question_to_protocol_ladder.png`: claim to hypothesis to fixed conditions to evidence.
3. `16_principle_official_example_baseline.png`: borrow official example first, then make minimal change.
4. `16_principle_one_variable_contract.png`: one-knob experiment contract across builder/model/state/control/solver/viewer layers.
5. `16_principle_evidence_ledger.png`: evidence ledger with commit, device, seed, baseline, knob, metric, verdict, feedback location.
6. `16_principle_metric_family_map.png`: evidence families: state predicate, particle predicate, scalar diagnostic, FD check, bridge buffer.
7. `16_principle_findings_feedback_loop.png`: how findings flow back into pitfalls, exercises, README gates, or later experiments.
8. `16_principle_experiment_cluster_map.png`: solver tolerance, warm-start, cloth substeps, DiffSim FD, MPM density, GPU profiling clusters.

### source-walkthrough.md

Create nine walkthrough images:

1. `16_walkthrough_pipeline_overview.png`: one-screen source-to-evidence path.
2. `16_walkthrough_beginner_path.png`: six beginner stages and stop-here rule.
3. `16_walkthrough_stage1_choose_claim.png`: choose one claim from previous chapters and make it testable.
4. `16_walkthrough_stage2_clone_baseline_example.png`: borrow `basic_pendulum` baseline and mark source objects.
5. `16_walkthrough_stage3_freeze_conditions.png`: freeze commit, device, seed, dt, frames, solver, viewer boundary.
6. `16_walkthrough_stage4_change_one_knob.png`: safe one-variable sweep examples.
7. `16_walkthrough_stage5_record_metrics.png`: record table template with evidence source and failure flag.
8. `16_walkthrough_stage6_compare_and_verdict.png`: support/refute/inconclusive/out-of-scope verdict gate.
9. `16_walkthrough_object_ledger_stop_here.png`: object ledger and stop-here recap.

### examples.md

Create two observation-sheet images:

1. `16_examples_experiment_role_table.png`: selected official examples and each one's unique experiment job.
2. `16_examples_fd_validation_anchor.png`: DiffSim finite-difference validation anchor.

### pitfalls.md

Create one risk-review image:

1. `16_pitfalls_experiment_mistakes.png`: common experiment mistakes and corrected mental models.

### exercises.md

Create one worksheet image:

1. `16_exercises_protocol_design_quiz.png`: protocol worksheet for choosing variable, fixed conditions, evidence, and verdict.

## Calibration Image Prompt Direction

Use native imagegen with a prompt shaped like this:

```text
Create a clean Chinese tutorial infographic for Newton Learn chapter 16.
Style: white background, thin blue rounded cards, dark-blue title, numbered blue badges, compact multi-panel teacher review-sheet layout, friendly small icons, arrows, flow boxes, evidence ledger cards, protocol checklist, verdict gate, and bottom recap strip. High information density but readable.
Content: teach self-experiments as a source-backed mini loop. Show seven steps:
1. 章节疑问 / source-backed question
2. 借 official example / newton/examples/
3. 固定条件: commit, device, seed, dt, frames, solver
4. 只改一个 knob
5. 读取 evidence: State, Contacts, scalar, FD
6. verdict: support / refute / inconclusive
7. findings.md 回填章节
Use short Chinese labels. English/code terms allowed only for exact names such as State, Contacts, why.md, findings.md, FD, seed, solver.step(). Do not invent code, commands, metrics, or benchmark numbers. Do not use SVG, HTML, screenshot, terminal UI, or photorealistic 3D rendering. Generate raster PNG.
```

## Review Loop

For each batch:

1. Generate images through native image generation.
2. Inspect generated files and reject blank, decorative, unreadable, or source-contradicting images.
3. Integrate accepted PNGs into Markdown.
4. Run path/reference verification.
5. Review style consistency against Chapters 03 / 04 / 14 / 15 and `chapter-visual-style-guide.md`.
6. Review source-truth risk against Chapter 16 Markdown.
7. Regenerate or adjust Markdown where review finds concrete issues.

## Acceptance Criteria

The pass is complete when:

- all six Chapter 16 Markdown files have tutorial-style PNG visual anchors.
- all 24 image references point to existing PNG files under `chapters/16_self_experiments/assets/`.
- all new assets are raster PNG files with stable `16_` filenames.
- no generated image introduces SVG / HTML fallbacks, fake code, fake terminal UI, fake benchmark numbers, or invented APIs.
- the image style matches the information-rich Chinese tutorial-handout direction established by Chapters 03 / 04 / 14 / 15.
- images do not contradict current source boundaries: state predicates, viewer read/log boundary plus `apply_forces()`, FD validation, and multiphysics bridge ownership.
