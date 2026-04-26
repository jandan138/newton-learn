# Chapter 15 Tutorial Infographics Design

## Goal

Generate and integrate a complete set of Chapter 15 tutorial-style PNG images for the reader-facing chapter pages, using the dense Chinese learning-handout visual direction established by Chapters 03, 04, 13, and 14 and governed by `docs/superpowers/specs/chapter-visual-style-guide.md`.

Chapter 15 should teach multiphysics as coupling boundaries: one-model solver coupling, two-model bridge coupling, and external-solver bridge coupling. It must not make multiphysics look like a generic black-box co-simulation framework unless a specific current source path backs the claim.

## Scope

In scope:

- `chapters/15_multiphysics_pipeline/README.md`
- `chapters/15_multiphysics_pipeline/principle.md`
- `chapters/15_multiphysics_pipeline/source-walkthrough.md`
- `chapters/15_multiphysics_pipeline/examples.md`
- `chapters/15_multiphysics_pipeline/pitfalls.md`
- `chapters/15_multiphysics_pipeline/exercises.md`
- `chapters/15_multiphysics_pipeline/assets/`

Out of scope:

- SVG, HTML fallback, screenshots, fake code, or fake terminal UI.
- Decorative 3D scenes that do not teach source/data boundaries.
- Diagrams claiming generic implicit co-simulation API exists in current source.
- Diagrams that use viewer output as physics validation.

## Chapter 15 Teaching Spine

Every image should reinforce:

```text
choose coupling category
-> build material systems into Model(s)
-> allocate State / Contacts / bridge buffers
-> choose solver(s)
-> run explicit step order
-> render/log after physics update
```

The one-line memory hook is:

```text
多物理不是黑盒；先找 Model / State / Contacts / Solver 的边界。
```

## Visual Language

Follow `docs/superpowers/specs/chapter-visual-style-guide.md`.

Use:

- white background with thin blue rounded card borders
- dark-blue title and section headings
- numbered blue badges for coupling stages, source objects, bridge buffers, and misconceptions
- Chinese-first labels and explanations
- short exact English/code labels only when necessary, such as `Model`, `State`, `Contacts`, `ModelBuilder`, `SolverVBD`, `SolverImplicitMPM`, `SolverMuJoCo`, `SolverFeatherstone`, `CollisionPipeline`, `add_soft_grid()`, `add_cloth_grid()`, `solver.step()`, `collect_collider_impulses()`
- compact cards, arrows, state ledgers, bridge strips, object-role tables, one-model vs two-model maps, and bottom recap bars
- concrete drawings: soft body cube, cloth sheet/strap, sand particles, rigid blocks, robot arm, contact surface, impulse arrows, force buffer, state swap

Avoid:

- Chapter 02-style 3D diorama rendering
- photorealistic screenshots
- fake source code, fake terminal snippets, or invented filenames/API names
- long paragraphs inside images
- tiny code-heavy tables
- diagrams implying every solver pair is automatically coupled
- diagrams implying FAQ roadmap claims are current source APIs
- diagrams implying viewer output proves coupling correctness

## Coverage Strategy

Use 24 PNGs. Each image should have one concrete teaching job.

### README

Create three entry-map images:

1. `15_readme_multiphysics_boundary_map.png`: calibration image. Shows one-model VBD lane, two-model MPM bridge lane, external robot/cloth bridge lane.
2. `15_readme_file_roles_reading_order.png`: file responsibilities and recommended reading order.
3. `15_readme_completion_scope_prereq.png`: completion gate, scope/non-scope, prerequisites, and category table.

### principle.md

Create eight conceptual images:

1. `15_principle_one_model_vs_two_models.png`: one `Model` vs two `Model`s as the first classification.
2. `15_principle_coupling_contract_ladder.png`: classification ladder: model boundary, state boundary, solver boundary, bridge buffers.
3. `15_principle_builder_material_ledger.png`: builder APIs as material/topology ledger.
4. `15_principle_contacts_are_coupling_surface.png`: `Contacts` as solver input, not viewer decoration.
5. `15_principle_vbd_single_solver_loop.png`: single-model `SolverVBD` soft/cloth loop.
6. `15_principle_mpm_twomodel_impulse_bridge.png`: MPM rigid/sand two-model impulse bridge.
7. `15_principle_external_rigid_solver_bridge.png`: robot solver -> collision pipeline -> cloth VBD.
8. `15_principle_roadmap_scope_table.png`: current source vs FAQ roadmap vs forbidden claims.

### source-walkthrough.md

Create nine walkthrough images:

1. `15_walkthrough_pipeline_overview.png`: one-screen source path map.
2. `15_walkthrough_beginner_path.png`: six beginner stages and verification questions.
3. `15_walkthrough_stage1_example_inventory.png`: current `examples/multiphysics/` inventory.
4. `15_walkthrough_stage2_softbody_cloth_build.png`: `add_soft_grid + add_cloth_grid -> Model`.
5. `15_walkthrough_stage3_vbd_step_contract.png`: VBD initialize/iterate/finalize.
6. `15_walkthrough_stage4_gift_geometry_and_contacts.png`: gift custom geometry and contact settings.
7. `15_walkthrough_stage5_mpm_twoway_bridge.png`: rigid/sand bridge with impulses/body forces.
8. `15_walkthrough_stage6_cloth_franka_bridge.png`: robot solver / collision pipeline / cloth VBD step order.
9. `15_walkthrough_object_ledger_stop_here.png`: object ledger and stop-here recap.

### examples.md

Create two observation-sheet images:

1. `15_examples_role_table.png`: selected examples and each one's unique teaching job.
2. `15_examples_softbody_gift_anchor.png`: softbody dropping/gift single-model VBD anchor and contrast to bridge examples.

### pitfalls.md

Create one risk-review image:

1. `15_pitfalls_coupling_mistakes.png`: common coupling mistakes and corrected mental models.

### exercises.md

Create one worksheet image:

1. `15_exercises_coupling_boundary_quiz.png`: annotate coupling category, source-of-truth buffers, and bridge order.

## Asset Rules

- Final images go under `chapters/15_multiphysics_pipeline/assets/`.
- Use stable ASCII filenames with prefix `15_` and suffix `.png`.
- Generate raster PNG files only.
- Keep native-imagegen prompt artifacts outside tracked chapter assets.
- Do not overwrite assets from other chapters.

## Calibration Image Prompt Direction

Use native imagegen with a prompt shaped like this:

```text
Create a clean Chinese tutorial infographic for Newton Learn chapter 15.
Style: white background, thin blue rounded cards, dark-blue title, numbered blue badges, compact multi-panel teacher review-sheet layout, friendly small icons, arrows, flow boxes, source-of-truth state ledgers, bridge buffers, and bottom recap strip. High information density but readable.
Content: teach multiphysics coupling boundaries. Show three lanes:
1. single Model / SolverVBD: soft body + cloth -> Model/State/Contacts -> SolverVBD -> state update.
2. two Models / two solvers: rigid Model + sand Model -> collider impulses/body forces bridge -> SolverMuJoCo + SolverImplicitMPM.
3. external rigid solver bridge: robot solver -> CollisionPipeline -> cloth SolverVBD.
Use short Chinese labels. English/code terms allowed only for exact names such as Model, State, Contacts, SolverVBD, SolverImplicitMPM, SolverMuJoCo, CollisionPipeline, collect_collider_impulses(). Do not invent code. Do not include long paragraphs. Do not use SVG, HTML, screenshot, terminal UI, or photorealistic 3D rendering. Generate raster PNG.
```

## Review Loop

For each batch:

1. Generate images through native image generation.
2. Inspect generated files and reject blank, decorative, unreadable, or source-contradicting images.
3. Integrate accepted PNGs into Markdown.
4. Run path/reference verification.
5. Review style consistency against Chapters 03 / 04 / 14 and `chapter-visual-style-guide.md`.
6. Review source-truth risk against Chapter 15 Markdown.
7. Regenerate or adjust Markdown where review finds concrete issues.

## Acceptance Criteria

The pass is complete when:

- all six Chapter 15 Markdown files have tutorial-style PNG visual anchors.
- all 24 image references point to existing PNG files under `chapters/15_multiphysics_pipeline/assets/`.
- all new assets are raster PNG files with stable `15_` filenames.
- no generated image introduces SVG / HTML fallbacks, fake code, fake terminal UI, or invented APIs.
- the image style matches the information-rich Chinese tutorial-handout direction established by Chapters 03 / 04 / 14.
- images do not contradict current source boundaries: VBD single-model examples, MPM two-model bridge, external robot/cloth bridge, and FAQ roadmap language.
