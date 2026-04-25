# Chapter 05 Tutorial Infographics Design

## Goal

Generate and integrate a complete set of chapter 05 tutorial-style PNG images for the reader-facing chapter pages, using chapters 03 and 04 as the established visual style samples and following `docs/superpowers/specs/chapter-visual-style-guide.md`.

Chapter 05 should extend the same dense Chinese learning-handout direction from "geometry words" and "scene-to-Model bridge" into the articulation layer: how flat joint layouts, generalized coordinates, FK, motion subspace, and mass / inertia buffers start the rigid-body dynamics path.

## User Direction

- Continue autonomously without waiting for further approval.
- Reuse the chapter 03 / 04 visual system as the template for chapter 05.
- Start with a calibration infographic before generating the full batch.
- After calibration review, batch-generate PNG tutorial images.
- Integrate accepted PNGs into `README.md`, `principle.md`, `source-walkthrough.md`, and `examples.md`.
- Keep `source-walkthrough-deep.md` out of this image pass unless a later task explicitly asks for deep-walkthrough visuals.
- If generation or review finds a concrete issue, iterate and use agents for review rather than stopping.

## Scope

In scope:

- `chapters/05_rigid_articulation/README.md`
- `chapters/05_rigid_articulation/principle.md`
- `chapters/05_rigid_articulation/source-walkthrough.md`
- `chapters/05_rigid_articulation/examples.md`
- `chapters/05_rigid_articulation/assets/`

Out of scope:

- `chapters/05_rigid_articulation/source-walkthrough-deep.md`
- full Featherstone / ABA / CRBA derivations
- Jacobian, Delassus, contact math, and solver-family comparison details
- collision pair generation or contact-candidate visuals, which belong to chapter 06 and later
- robot control, gain tuning, reward shaping, or large-robot workflow diagrams

## Chapter 05 Teaching Spine

The image set should reinforce the same spine that chapter 05 already teaches:

```text
03 geometry words + 04 scene-to-Model fields
-> flat articulation layout
-> joint tree and slice starts
-> joint-space state: joint_q / joint_qd
-> FK and velocity propagation
-> body-space state: body_q / body_qd
-> motion subspace
-> mass / inertia becomes spatial buffers
-> Featherstone continues the handoff
-> onward routes to 06 collision, 07 contact math, and 08 solvers
```

The reader should leave with one stable bridge model: articulation is not a nested object tree in the reader-facing mental model. It is a flat layout plus transforms and state buffers that let joint-space values become body-space motion and solver-side spatial buffers.

## Core Learner Questions

The images should answer these questions visually:

- What gap remains after chapter 04 has already produced `Model` fields?
- Why is an articulation first a joint range / body tree encoded in flat arrays?
- What do `articulation_start`, `joint_q_start`, and `joint_qd_start` tell the reader?
- Why are `joint_q / joint_qd` and `body_q / body_qd` two different state layers?
- How does FK turn `joint_q` plus `joint_X_p / joint_X_c` into `body_q`?
- Why does `joint_qd` need motion subspace before it becomes body velocity?
- How do `body_mass / body_com / body_inertia` become Featherstone-side spatial buffers?
- What does `SolverFeatherstone.step()` continue from the public state handoff, and what remains solver-internal?
- Which examples best expose `joint_q -> body_q`, joint frame changes, and inertia changes?

## Visual Language

Follow `docs/superpowers/specs/chapter-visual-style-guide.md`.

Use:

- white background with thin blue rounded card borders
- dark-blue title and section headings
- numbered blue badges for stages, object roles, checkpoints, and misconceptions
- Chinese-first labels and explanations
- short exact English/code labels only when necessary, such as `articulation_start`, `joint_parent`, `joint_child`, `joint_X_p/X_c`, `joint_q`, `joint_qd`, `joint_q_start`, `joint_qd_start`, `body_q`, `body_qd`, `eval_fk()`, `motion subspace`, `joint_S_s`, `body_I_m`, `body_X_com`, `body_v_s`, `body_f_s`, `SolverFeatherstone.step()`
- compact cards, arrows, role ledgers, before/after strips, state-layer comparisons, slice diagrams, tree-to-array maps, and bottom recap bars
- small schematic icons for joint tree, flat array slices, body/world pose, velocity twist, inertia block, solver workspace, examples

Avoid:

- cinematic robot renders or decorative robot scenes
- fake source code, fake terminal snippets, or invented API calls
- detailed line numbers or file paths inside images
- full ABA / CRBA math derivations inside images
- diagrams that imply `joint_qd` directly equals `body_qd`
- diagrams that imply articulation is preserved as a nested Python object tree
- diagrams that imply Featherstone recreates unrelated external state rather than consuming the same `State` / layout handoff
- tiny matrix / code tables that are unreadable in Markdown

## Coverage Strategy

Use a full reader-facing coverage pass, while allowing one strong infographic to anchor adjacent short sections.

Expected image count is about 22 PNGs: 3 for `README.md`, 7 for `principle.md`, 8 for `source-walkthrough.md`, and 4 for `examples.md`. The implementation plan may merge adjacent anchors where the Markdown sections are short and the merged image is clearer.

### README

Create tutorial-navigation images for:

1. Chapter identity: 03 / 04 fields now start moving through articulation.
2. File responsibilities and reading order: `README.md`, `principle.md`, `source-walkthrough.md`, `source-walkthrough-deep.md`, and `examples.md`.
3. Completion threshold, chapter scope, explicit non-scope, prerequisites, GAMES103 bridge, and expected output.

These should be placed across the README so the file reads like a visual entry map. The GAMES103 and reading-order sections can be covered by nearby scope / file-role images if a separate table image would be redundant.

### principle.md

Create conceptual tutorial infographics for:

1. After chapter 04: fields exist, but now need to move.
2. Articulation as joint range plus body tree in flat arrays.
3. Joint-space versus body-space state layers: `joint_q / joint_qd` compared with `body_q / body_qd`.
4. FK chain: `joint_q -> X_j -> parent chain -> body_q`.
5. Motion subspace: `joint_qd` as coefficients over allowed spatial directions before `body_qd`.
6. Inertia bridge: `body_mass / body_com / body_inertia -> body_I_m / body_X_com / body_I_s`.
7. Chapter 05 onward split to chapter 06 collision, chapter 07 contact math, and chapter 08 solver families.

The second principle image should serve as the calibration image, because it exercises the chapter 05 visual grammar: minimal two-link tree, flat slice arrays, transforms, and state layers.

### source-walkthrough.md

Create source-walkthrough tutorial infographics for:

1. What the walkthrough follows: minimal joint tree to solver-side spatial buffers.
2. Beginner path: five stages and the verification question each stage answers.
3. Stage 1: flat articulation slice layout, not nested object tree.
4. Stage 2: `Model` and `State` hold joint-space and body-space layers together.
5. Stage 3: FK and velocity propagation translate joint-space into body-space.
6. Stage 4: Featherstone precomputes spatial inertia, COM transforms, motion subspace, and internal buffers.
7. Stage 5: `SolverFeatherstone.step()` continues the same handoff and writes back public state.
8. Object ledger plus stop-here recap.

Do not make a separate image for `Go Deeper` unless implementation review finds that the end of the page needs a small navigation visual. The deep file remains the source-anchor document.

### examples.md

Create example-observation infographics for:

1. Examples overview: why `basic_pendulum`, `basic_joints`, and `robot_cartpole` are observation tasks, not solver-family tutorials.
2. `basic_pendulum`: two revolute joints expose `joint_q -> eval_fk() -> body_q`, joint frame effects, and link length / inertia intuition.
3. `basic_joints`: revolute / prismatic / ball compare different `joint_q` shapes and FK effects.
4. `robot_cartpole`: imported articulation still follows the same `joint_q -> FK -> body_q` chain; replication changes worlds, not joint-frame semantics.

These images should be lighter than the principle / walkthrough diagrams, because examples.md already uses compact observation tables.

## Recommended Asset Names

Use stable ASCII filenames with prefix `05_` and suffix `.png`.

Recommended names:

- `05_readme_articulation_spine.png`
- `05_readme_file_roles_reading_order.png`
- `05_readme_completion_scope_prereq.png`
- `05_principle_after_04_fields_start_moving.png`
- `05_principle_flat_articulation_layout.png`
- `05_principle_joint_vs_body_state.png`
- `05_principle_fk_chain_map.png`
- `05_principle_motion_subspace_bridge.png`
- `05_principle_inertia_spatial_buffers.png`
- `05_principle_next_chapters_map.png`
- `05_walkthrough_pipeline_overview.png`
- `05_walkthrough_beginner_path.png`
- `05_walkthrough_stage1_flat_slice_layout.png`
- `05_walkthrough_stage2_state_layers.png`
- `05_walkthrough_stage3_fk_velocity_bridge.png`
- `05_walkthrough_stage4_featherstone_spatial_buffers.png`
- `05_walkthrough_stage5_solver_step_handoff.png`
- `05_walkthrough_object_ledger_stop_here.png`
- `05_examples_overview_observation_tasks.png`
- `05_examples_basic_pendulum_joint_frame.png`
- `05_examples_basic_joints_type_compare.png`
- `05_examples_robot_cartpole_imported_layout.png`

The implementation plan may rename files if a clearer grouping emerges, but all names must stay stable, ASCII, and chapter-prefixed.

## Source-Truth Discipline

Markdown remains authoritative for:

- exact source file names and source snippets
- exact line anchors and optional deep branches
- exact joint type storage rules and edge cases
- internal velocity conversion details
- solver-side buffer semantics that require careful wording
- distinctions among FK, Featherstone internals, contact math, and solver-family comparison

Images may compress the idea, but must not contradict the Markdown. In particular:

- A toy articulation image must be labeled as a teaching toy, not as a full robot model.
- Flat-layout diagrams must show slice starts and ranges conceptually rather than inventing exact indexes.
- State-layer diagrams must keep `joint_q / joint_qd` distinct from `body_q / body_qd`.
- Motion-subspace diagrams must show `joint_qd` as coefficients over allowed directions, not as a world-space body velocity.
- Featherstone diagrams must show solver-internal buffers as continuation of the same handoff, not as unrelated external state.
- Example diagrams must stay observational; they should not become control or tuning advice.

## Asset Rules

- Final images go under `chapters/05_rigid_articulation/assets/`.
- Generate raster PNG files only.
- Do not introduce SVG or HTML image fallbacks.
- Do not optimize for file size unless image payload becomes an operational blocker.
- Prefer reusing one strong summary image across nearby short sections over generating redundant weak images.
- Keep generated images reader-facing; do not place implementation-plan screenshots or prompt artifacts in chapter assets.

## Calibration Image

The first generated image should be:

`chapters/05_rigid_articulation/assets/05_principle_flat_articulation_layout.png`

It should show a minimal teaching articulation and the flat layout bridge:

```text
minimal two-link articulation
-> joint_parent / joint_child tree edge
-> joint_X_p / joint_X_c frame anchors
-> articulation_start joint range
-> joint_q_start / joint_qd_start slices
-> joint_q / joint_qd feed FK
-> body_q / body_qd result
```

Acceptance check for the calibration image:

- It looks like chapters 03 / 04 dense Chinese tutorial-handout style.
- It is visibly instructional, not decorative robot art.
- It uses short, readable Chinese labels.
- It includes only real code/API terms that chapter 05 already teaches.
- It does not pretend to show exact source code or exact line anchors.
- It keeps flat articulation layout, state layers, and FK handoff visually distinct.

Only expand to the remaining images after this calibration image passes review.

## Generation Strategy

Start with calibration:

1. Generate `05_principle_flat_articulation_layout.png`.
2. Review it against `chapter-visual-style-guide.md` and chapter 04 assets.
3. If it drifts toward decorative robot art, unreadable math, fake code, or `joint_qd == body_qd` confusion, tighten the prompt and regenerate.
4. Once accepted, generate the remaining images in batches by target file.
5. Integrate Markdown links after each batch, so broken paths and awkward placement are caught early.

Prompt direction:

```text
Create a clean Chinese tutorial infographic for Newton Learn chapter 05.
Style: white background, thin blue rounded cards, dark-blue title, numbered blue badges, compact multi-panel teacher review-sheet layout, friendly small icons, arrows, tree-to-array maps, state-layer comparisons, flow boxes, role ledgers, and bottom recap strip. High information density but readable.
Content: teach how flat articulation layout, joint_q / joint_qd, FK, motion subspace, body_q / body_qd, and mass / inertia spatial buffers start the rigid-body dynamics path. Use short Chinese labels. English/code terms allowed only for exact names such as articulation_start, joint_q_start, joint_qd_start, joint_q, joint_qd, body_q, body_qd, joint_X_p/X_c, eval_fk(), motion subspace, joint_S_s, body_I_m, body_X_com, SolverFeatherstone.step(). Do not invent source code. Do not include long paragraphs. Do not use SVG, HTML, screenshot, terminal UI, photorealistic robot rendering, or full ABA/CRBA derivation. Generate raster PNG.
```

## Review Loop

For each batch:

1. Generate images through native image generation.
2. Inspect generated files and reject blank, decorative, unreadable, or source-contradicting images.
3. Integrate accepted PNGs into Markdown.
4. Run path/reference verification.
5. Review style consistency against chapters 03 / 04 and `chapter-visual-style-guide.md`.
6. Review source-truth risk against `README.md`, `principle.md`, `source-walkthrough.md`, and `examples.md`.
7. Regenerate or adjust Markdown where review finds concrete issues.

## Acceptance Criteria

The pass is complete when:

- `README.md`, `principle.md`, `source-walkthrough.md`, and `examples.md` have tutorial-style PNG visual anchors across their reader-facing sections.
- `source-walkthrough-deep.md` remains intentionally out of scope.
- all new chapter-05 inline image references point to existing PNG files under `chapters/05_rigid_articulation/assets/`.
- all new assets are raster PNG files with stable `05_` filenames.
- no generated image introduces SVG / HTML fallbacks, fake code, fake terminal UI, or decorative robot renderings.
- the image style matches the information-rich Chinese tutorial-handout direction established by chapters 03 and 04.
- images teach the articulation handoff without overclaiming exact source semantics or full dynamics derivations.
- Markdown remains the source of truth for exact code snippets, joint-type edge cases, internal velocity conventions, and source anchors.
