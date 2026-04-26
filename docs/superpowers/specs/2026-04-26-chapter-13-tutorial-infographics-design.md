# Chapter 13 Tutorial Infographics Design

## Goal

Generate and integrate a complete set of chapter 13 tutorial-style PNG images for the reader-facing Chapter 13 pages, using the dense Chinese learning-handout visual direction established by chapters 03 and 04 and governed by `docs/superpowers/specs/chapter-visual-style-guide.md`.

Chapter 13 already has a strong textual spine from the restored `ch13-diffsim` branch. This pass should keep that content intact and add native imagegen PNGs that make the trust-first DiffSim loop easier to scan and remember.

## User Direction

- Base the work on latest `main`, not by directly merging the old `ch13-diffsim` branch.
- Safely migrate the old Chapter 13正文 first, then supplement it with native imagegen tutorial graphics.
- Follow the Chapter 03 / 04 infographic style, not the Chapter 02 diorama direction.
- Use concrete generated drawings and teaching diagrams from native imagegen, not deterministic SVG/HTML replacements.
- Proceed autonomously, review, commit, and push.

## Scope

In scope:

- `chapters/13_diffsim/README.md`
- `chapters/13_diffsim/principle.md`
- `chapters/13_diffsim/source-walkthrough.md`
- `chapters/13_diffsim/examples.md`
- `chapters/13_diffsim/pitfalls.md`
- `chapters/13_diffsim/exercises.md`
- `chapters/13_diffsim/assets/`

Out of scope:

- writing a deep walkthrough for Chapter 13
- changing Chapter 13 source semantics beyond image placement prose
- revising existing chapters 00-12
- replacing native imagegen PNGs with SVG, HTML, screenshots, or deterministic renderer output
- proving any new DiffSim claims not already supported by the Markdown

## Chapter 13 Teaching Spine

Every image should reinforce the same spine:

```text
rollout missed target
-> choose upstream parameter
-> requires_grad=True marks tracked objects
-> wp.Tape() records forward rollout
-> scalar loss starts backward()
-> gradient lands on State / Model / external buffer
-> FD validation establishes trust window
-> only then update loop
```

The one-line memory hook is:

```text
verify before optimize
```

## Core Learner Questions

The image set should answer these questions visually:

- Why is DiffSim a backward capability on top of forward simulation, not a new solver family?
- What does `requires_grad=True` decide before `wp.Tape()` appears?
- What does `wp.Tape()` record, and why is it not a correctness proof?
- Why must rollout output become a scalar loss before `backward()`?
- Where does the gradient land in `ball`, `spring_cage`, and `soft_body`?
- Why is finite-difference validation a hard trust gate rather than optional debug?
- Why does a single FD epsilon not prove trust?
- Why are `drone / cloth / bear` advanced branches, not first-pass anchors?

## Visual Language

Follow `docs/superpowers/specs/chapter-visual-style-guide.md`.

Use:

- white background with thin blue rounded card borders
- dark-blue title and section headings
- numbered blue badges for loop stages, trust gates, parameter placements, and pitfalls
- Chinese-first labels and explanations
- short exact English/code labels only when necessary, such as `requires_grad=True`, `wp.Tape()`, `backward()`, `FD`, `State`, `Model`, `material_params`, `check_grad()`, `loss`, `grad`, `update`
- compact cards, arrows, role ledgers, loop strips, trust-gate checkpoints, comparison tables, and bottom recap bars
- concrete drawings of a rollout path, target marker, parameter ledger, FD sweep table, and update loop

Avoid:

- Chapter 02-style 3D diorama rendering
- cinematic physics scenes or decorative-only images
- fake source code, fake terminal snippets, or invented filenames
- long paragraphs inside images
- tiny code-heavy tables that become unreadable in Markdown
- visual claims that imply `backward()` proves correctness
- diagrams that generalize one `ball` contact check into contact-wide trust

## Coverage Strategy

Use 27 PNGs. Each image should have one concrete teaching job, while nearby short sections may share a strong anchor image.

### README

Create three entry-map images:

1. `13_readme_verify_before_optimize_spine.png`: calibration image. Shows rollout miss -> upstream parameter -> Tape -> backward -> FD trust gate -> update loop.
2. `13_readme_file_roles_reading_order.png`: file responsibilities and recommended reading order.
3. `13_readme_example_roles_completion_gate.png`: example roles, parameter-placement axis, scope, prerequisites, and completion checklist.

### principle.md

Create seven conceptual images:

1. `13_principle_trust_first_title.png`: chapter title correction: not solver family, not theory dump, but trust-first backward capability.
2. `13_principle_requires_grad_ledger.png`: `requires_grad=True` selects tracked model/state/buffer candidates.
3. `13_principle_tape_loss_backward_chain.png`: Tape records forward rollout, scalar loss starts backward, gradient returns upstream.
4. `13_principle_parameter_placement_axis.png`: State / Model / external buffer comparison table.
5. `13_principle_ball_state_loop.png`: `ball` as minimal state-parameter trust loop.
6. `13_principle_fd_trust_gate.png`: FD sweep, trust window, and update-after-verdict sequence.
7. `13_principle_advanced_defer_map.png`: why `drone / cloth / bear` are deferred.

### source-walkthrough.md

Create nine walkthrough images:

1. `13_walkthrough_pipeline_overview.png`: one-screen chapter map.
2. `13_walkthrough_beginner_path.png`: six beginner stages and their verification questions.
3. `13_walkthrough_stage1_requires_grad.png`: `ball` chooses upstream initial velocity and tracked objects.
4. `13_walkthrough_stage2_tape_loss.png`: Tape captures forward rollout and loss kernel creates scalar loss.
5. `13_walkthrough_stage3_backward_update.png`: `backward()` deposits gradient and `step_kernel` applies update.
6. `13_walkthrough_stage4_fd_trust_gate.png`: `check_grad()` plus repo-level FD protocol.
7. `13_walkthrough_stage5_spring_cage_model_param.png`: gradient lands on `model.spring_rest_length`.
8. `13_walkthrough_stage6_soft_body_external_buffer.png`: `material_params` -> remap -> `model.tet_materials`.
9. `13_walkthrough_object_ledger_stop_here.png`: object ledger plus stop-here recap.

### examples.md

Create four observation-sheet images:

1. `13_examples_role_table.png`: six examples, unique jobs, first/second/advanced layering.
2. `13_examples_ball_anchor.png`: `ball` five observation questions.
3. `13_examples_parameter_branches.png`: `spring_cage` and `soft_body` branch comparison.
4. `13_examples_advanced_defer.png`: `drone / cloth / bear` defer reasons.

### pitfalls.md

Create two risk-review images:

1. `13_pitfalls_verification_traps.png`: `backward()` ran, one epsilon, parameter location confusion.
2. `13_pitfalls_contact_scope_and_fallback.png`: contact overclaim boundary and lowest fallback order.

### exercises.md

Create two exercise images:

1. `13_exercises_fd_error_table.png`: FD error table worksheet with seed / window / metric.
2. `13_exercises_validation_quiz.png`: parameter placement quiz, invalid validation checks, and drone defer question.

## Asset Rules

- Final images go under `chapters/13_diffsim/assets/`.
- Use stable ASCII filenames with prefix `13_` and suffix `.png`.
- Generate raster PNG files only.
- Keep native-imagegen prompt artifacts outside the tracked chapter assets.
- Do not optimize for file size unless image payload becomes an operational blocker.
- Do not overwrite unrelated assets or change chapters 00-12.

## Calibration Image

The first generated image should be:

`chapters/13_diffsim/assets/13_readme_verify_before_optimize_spine.png`

It should show:

```text
rollout miss
-> choose upstream parameter
-> requires_grad=True
-> wp.Tape() records forward
-> scalar loss
-> backward()
-> gradient on upstream parameter
-> FD sweep / trust window
-> update loop
```

Acceptance check for the calibration image:

- It looks like a Chapter 03 / 04 dense Chinese tutorial handout.
- It is visibly instructional, not decorative.
- It contains concrete diagram elements: target marker, parameter ledger, Tape box, FD gate, update arrow.
- It uses only short readable labels.
- It does not include fake source code or terminal output.
- It does not imply `backward()` proves correctness.

## Prompt Direction

Use native imagegen with prompts shaped like this:

```text
Create a clean Chinese tutorial infographic for Newton Learn chapter 13.
Style: white background, thin blue rounded cards, dark-blue title, numbered blue badges, compact multi-panel teacher review-sheet layout, friendly small icons, arrows, flow boxes, role ledgers, trust-gate checkpoints, and bottom recap strip. High information density but readable.
Content: teach the assigned DiffSim concept from Chapter 13: verify before optimize, upstream parameter placement, wp.Tape(), scalar loss, backward(), finite-difference trust gate, or example role separation. Use short Chinese labels. English/code terms allowed only for exact names such as requires_grad=True, wp.Tape(), backward(), FD, State, Model, material_params, check_grad(), loss, grad, update. Do not invent code. Do not include long paragraphs. Do not use SVG, HTML, screenshot, terminal UI, or photorealistic 3D rendering. Generate raster PNG.
```

## Source-Truth Discipline

Markdown remains authoritative for:

- exact source file paths
- code excerpts and precise call order
- `check_grad()` limitations
- `conventions/diffsim-validation.md` thresholds and verdict wording
- contact caveats
- distinctions among `State`, `Model`, and external buffer parameter placement

Images may compress relationships, but they must not contradict the Markdown. In particular:

- `FD` graphics should show a sweep / trust window, not a single magic epsilon.
- `backward()` graphics should label the result as a gradient candidate before FD.
- Contact graphics must stay local to the current fixed seed / fixed window / current example.
- Advanced branch graphics should label `drone / cloth / bear` as deferred, not inferior.

## Review Loop

For each batch:

1. Generate images through native imagegen.
2. Inspect file type, dimensions, and file size.
3. Reject blank, decorative, unreadable, or source-contradicting images.
4. Integrate accepted PNG links into Markdown.
5. Run path/reference verification.
6. Review style consistency against chapter 03 / 04 and `chapter-visual-style-guide.md`.
7. Review source-truth risk against Chapter 13 Markdown.
8. Regenerate or adjust Markdown where review finds concrete issues.

## Acceptance Criteria

The pass is complete when:

- Chapter 13正文 has been safely migrated onto a branch based on latest `main`.
- `README.md`, `principle.md`, `source-walkthrough.md`, `examples.md`, `pitfalls.md`, and `exercises.md` all contain nearby tutorial-style PNG anchors.
- all new inline image references point to existing PNG files under `chapters/13_diffsim/assets/`.
- all new assets are raster PNG files with stable `13_` filenames.
- no generated image introduces SVG / HTML fallbacks, fake code, or fake terminal UI.
- the image style matches the information-rich Chinese tutorial-handout direction established by chapters 03 and 04.
- Markdown remains the source of truth for exact code and validation protocol details.
