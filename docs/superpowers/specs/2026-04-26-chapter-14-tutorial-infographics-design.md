# Chapter 14 Tutorial Infographics Design

## Goal

Generate and integrate a complete set of Chapter 14 tutorial-style PNG images for the reader-facing chapter pages, using the dense Chinese learning-handout visual direction established by Chapters 03 and 04 and governed by `docs/superpowers/specs/chapter-visual-style-guide.md`.

Chapter 14 should teach viewer as a boundary layer: it reads, logs, displays, exports, records, or no-ops around an already-running simulation. It must not make viewer look like a solver, contact generator, USD importer, or training framework.

## User Direction

- Reuse the Chapter 03 / 04 visual system, not the Chapter 02 diorama direction.
- Start with a calibration infographic.
- Use native imagegen PNGs with concrete teaching drawings.
- Batch-generate the remaining PNGs after calibration passes review.
- Integrate images into `README.md`, `principle.md`, `source-walkthrough.md`, `examples.md`, `pitfalls.md`, and `exercises.md`.
- Keep exact source truth in Markdown, not inside generated images.

## Scope

In scope:

- `chapters/14_viewer_integration/README.md`
- `chapters/14_viewer_integration/principle.md`
- `chapters/14_viewer_integration/source-walkthrough.md`
- `chapters/14_viewer_integration/examples.md`
- `chapters/14_viewer_integration/pitfalls.md`
- `chapters/14_viewer_integration/exercises.md`
- `chapters/14_viewer_integration/assets/`

Out of scope:

- OpenGL shader or renderer internals.
- USD authoring or import diagrams. Chapter 04 covers scene-to-`Model`.
- Full Rerun, Viser, Isaac Lab, Omniverse, MJX, or Gymnasium tutorials.
- Any diagram that invents missing MJX/Gymnasium source adapters.
- SVG, HTML fallback, screenshots, fake code, or fake terminal UI.

## Chapter 14 Teaching Spine

Every image should reinforce:

```text
Model / State / Contacts already exist
-> examples init selects viewer backend
-> runner owns outer lifecycle
-> optional viewer input writes before solver step
-> solver/collision update State and Contacts
-> render phase reads fresh data
-> backend presents / exports / records / streams / no-ops
```

The one-line memory hook is:

```text
viewer 是读/记录/展示边界，不是第二套 physics。
```

## Core Learner Questions

The image set should answer:

- Where does viewer backend selection happen?
- Why is the outer loop viewer-aware while physics remains solver-owned?
- Which viewer calls are read/log/render and which can write before the next step?
- Why does `log_state()` need a fresh `State`?
- Why are contact arrows just overlays of existing `Contacts`?
- How do GL, USD, Rerun, File, Null, and Viser differ by output?
- Why is `ViewerUSD` output not the same direction as Chapter 04 USD import?
- Why are Isaac Lab / Isaac Sim ecosystem boundaries not the same as a viewer backend?
- Why should MJX/Gymnasium stay as non-walkthrough vocabulary until source anchors exist?

## Visual Language

Follow `docs/superpowers/specs/chapter-visual-style-guide.md`.

Use:

- white background with thin blue rounded card borders
- dark-blue title and section headings
- numbered blue badges for loop stages, API roles, backend choices, and pitfalls
- Chinese-first labels and explanations
- short exact English/code labels only when necessary, such as `ViewerBase`, `State`, `Contacts`, `set_model()`, `is_running()`, `is_paused()`, `apply_forces()`, `solver.step()`, `begin_frame()`, `log_state()`, `log_contacts()`, `log_gizmo()`, `end_frame()`, `ViewerGL`, `ViewerUSD`, `ViewerRerun`, `ViewerFile`, `ViewerNull`, `ViewerViser`
- compact cards, arrows, two-lane read/write strips, source-to-runtime maps, backend role tables, contact overlay mini-diagrams, and bottom recap bars
- concrete drawings: state ledger, contact arrow overlay, GL window, USD file, Rerun timeline, recording file, null frame counter, Isaac Lab external box

Avoid:

- Chapter 02-style 3D diorama rendering
- cinematic physics scenes or decorative-only viewer windows
- photorealistic screenshots
- fake source code, fake terminal snippets, or invented filenames/API names
- long paragraphs inside images
- tiny code-heavy tables
- diagrams implying viewer runs solver, collision, contact generation, or gradient validation
- diagrams implying current-source MJX/Gymnasium adapters exist

## Coverage Strategy

Use 24 PNGs. Each image should have one concrete teaching job, while nearby short sections may share a strong visual anchor.

### README

Create three entry-map images:

1. `14_readme_viewer_boundary_spine.png`: calibration image. Shows physics state already exists, optional input before step, solver update, render/log phase, and backend outputs.
2. `14_readme_file_roles_reading_order.png`: file responsibilities and recommended reading order.
3. `14_readme_completion_scope_prereq.png`: completion gate, scope/non-scope, prerequisites, and backend role table.

### principle.md

Create eight conceptual images:

1. `14_principle_not_solver_boundary.png`: title correction: viewer is boundary, not physics.
2. `14_principle_outer_loop_two_lanes.png`: two-lane loop: lifecycle/render lane plus optional write-before-step lane.
3. `14_principle_common_interface_map.png`: shared interface map across backends.
4. `14_principle_log_state_reads_state.png`: `Model + State -> log_state -> backend records`, with state freshness callout.
5. `14_principle_contacts_are_overlay.png`: contact arrows are overlay of existing `Contacts`.
6. `14_principle_input_writeback_timing.png`: `apply_forces()` timing before `solver.step()`.
7. `14_principle_backend_choice_table.png`: choose backend by output.
8. `14_principle_ecosystem_bridge_scope.png`: USD/Rerun/Viser/Isaac Lab boundary, with MJX/Gymnasium marked as no current source walkthrough.

### source-walkthrough.md

Create nine walkthrough images:

1. `14_walkthrough_pipeline_overview.png`: one-screen source path map.
2. `14_walkthrough_beginner_path.png`: six beginner stages and their verification questions.
3. `14_walkthrough_stage1_cli_backend_selection.png`: `--viewer` choices and `init()` dispatch.
4. `14_walkthrough_stage2_runner_loop.png`: `is_running -> is_paused -> step -> render -> close`.
5. `14_walkthrough_stage3_example_step_write_boundary.png`: `basic_pendulum` step side with `apply_forces`.
6. `14_walkthrough_stage4_render_log_boundary.png`: render side with `begin_frame/log_state/log_contacts/end_frame`.
7. `14_walkthrough_stage5_backend_lifecycle.png`: GL/USD/Rerun/Null lifecycle differences.
8. `14_walkthrough_stage6_debug_overlays.png`: gizmo/lines/points/scalars and ecosystem boundary.
9. `14_walkthrough_object_ledger_stop_here.png`: object ledger and stop-here recap.

### examples.md

Create two observation-sheet images:

1. `14_examples_role_table.png`: selected examples and each one's unique teaching job.
2. `14_examples_basic_pendulum_anchor.png`: step/render boundary in `basic_pendulum`.

### pitfalls.md

Create one risk-review image:

1. `14_pitfalls_boundary_mistakes.png`: viewer-as-solver, contact arrows as collision, stale state, USD direction, fake ecosystem paths.

### exercises.md

Create one worksheet image:

1. `14_exercises_loop_annotation_quiz.png`: annotate lifecycle / physics / render / write-before-step.

## Asset Rules

- Final images go under `chapters/14_viewer_integration/assets/`.
- Use stable ASCII filenames with prefix `14_` and suffix `.png`.
- Generate raster PNG files only.
- Keep native-imagegen prompt artifacts outside tracked chapter assets.
- Do not optimize for file size unless image payload becomes an operational blocker.
- Do not overwrite assets from other chapters.

## Calibration Image

The first generated image should be:

```text
chapters/14_viewer_integration/assets/14_readme_viewer_boundary_spine.png
```

It should show:

```text
Model / State / Contacts already exist
-> optional viewer input apply_forces() writes before solver.step()
-> collision / solver update State and Contacts
-> begin_frame()
-> log_state() / log_contacts() / log_gizmo()
-> backend outputs: GL window | USD file | Rerun timeline | file recording | null counter
```

Acceptance check:

- It looks like a Chapter 03 / 04 dense Chinese tutorial handout.
- It is visibly instructional, not decorative.
- It contains concrete diagram elements: state ledger, contact overlay, input arrow before step, backend output cards.
- It uses only short readable labels.
- It does not include fake source code or terminal output.
- It does not imply viewer is a solver or contact generator.

## Prompt Direction

Use native imagegen with prompts shaped like this:

```text
Create a clean Chinese tutorial infographic for Newton Learn chapter 14.
Style: white background, thin blue rounded cards, dark-blue title, numbered blue badges, compact multi-panel teacher review-sheet layout, friendly small icons, arrows, flow boxes, two-lane read/write boundary, backend output cards, and bottom recap strip. High information density but readable.
Content: teach the assigned Chapter 14 viewer boundary concept: viewer backend selection, outer loop lifecycle, write-before-step input, log_state reading State, log_contacts as overlay, backend output choices, or ecosystem boundary. Use short Chinese labels. English/code terms allowed only for exact names such as ViewerBase, State, Contacts, apply_forces(), solver.step(), begin_frame(), log_state(), log_contacts(), ViewerGL, ViewerUSD, ViewerRerun, ViewerFile, ViewerNull, ViewerViser. Do not invent code. Do not include long paragraphs. Do not use SVG, HTML, screenshot, terminal UI, or photorealistic 3D rendering. Generate raster PNG.
```

## Source-Truth Discipline

Markdown remains authoritative for:

- exact source file paths and line references
- exact call order and source snippets
- which backend methods are no-op vs interactive
- state freshness caveats
- the distinction between `ViewerUSD` output and Chapter 04 USD import
- the current-source absence of MJX/Gymnasium adapter walkthroughs

Images may compress relationships, but must not contradict Markdown. In particular:

- `apply_forces()` graphics must show write-before-step timing, not a generic render effect.
- `log_contacts()` graphics must show arrows as overlay from `Contacts`.
- Backend diagrams must show output differences, not solver differences.
- Ecosystem diagrams may show Isaac Lab as an external integration box, but MJX/Gymnasium must be labeled as no current source walkthrough in this pass.

## Review Loop

For each batch:

1. Generate images through native image generation.
2. Inspect generated files and reject blank, decorative, unreadable, or source-contradicting images.
3. Integrate accepted PNGs into Markdown.
4. Run path/reference verification.
5. Review style consistency against Chapters 03 and 04 and `chapter-visual-style-guide.md`.
6. Review source-truth risk against Chapter 14 Markdown.
7. Regenerate or adjust Markdown where review finds concrete issues.

## Acceptance Criteria

The pass is complete when:

- all six Chapter 14 Markdown files have tutorial-style PNG visual anchors.
- all 24 image references point to existing PNG files under `chapters/14_viewer_integration/assets/`.
- all new assets are raster PNG files with stable `14_` filenames.
- no generated image introduces SVG / HTML fallbacks, fake code, fake terminal UI, or invented APIs.
- the image style matches the information-rich Chinese tutorial-handout direction established by Chapters 03 and 04.
- images teach viewer as a read/log/render boundary with optional write-before-step input.
- Markdown remains the source of truth for exact source anchors and backend caveats.
