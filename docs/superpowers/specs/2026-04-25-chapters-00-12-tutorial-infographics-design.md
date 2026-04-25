# Chapters 00-12 Tutorial Infographics Design

## Goal

Bring every already-written Newton Learn chapter up to the chapter 03 / 04 tutorial-infographic standard by adding missing reader-facing PNG visual anchors.

This is a catch-up pass. Chapters 02-07 already have image systems; chapters 00, 01, and 08-12 are complete enough as written chapters but have no chapter-local image assets. This pass fills those missing chapters without changing the chapter prose beyond inserting image references.

## Style Authority

Use these documents as the style contract:

- `docs/superpowers/specs/chapter-visual-style-guide.md`
- `docs/superpowers/specs/2026-04-24-chapter-03-tutorial-infographics-design.md`
- `docs/superpowers/specs/2026-04-25-chapter-04-tutorial-infographics-design.md`
- `docs/superpowers/specs/2026-04-25-chapters-05-07-native-imagegen-redo-design.md`

Do not use `docs/superpowers/specs/2026-04-23-chapter-02-native-imagegen-design.md` as the style source. Chapter 02's 3D diorama direction is not the target for this catch-up pass.

## Written-Chapter Inventory

The current repository state separates chapters into three groups:

| Group | Chapters | Status | Action |
| --- | --- | --- | --- |
| Image-complete | 02, 03, 04, 05, 06, 07 | Reader-facing PNG anchors already exist | Leave paths and assets unchanged |
| Written but image-missing | 00, 01, 08, 09, 10, 11, 12 | Chapter prose is complete enough for tutorial images | Generate and integrate PNG anchors |
| Outline only | 13, 14, 15, 16 | README-only skeletons without principle/source/examples coverage | Do not batch-image yet |

This pass defines "already written" as a chapter with substantive instructional prose across its expected reader-facing files. The target set is:

- `chapters/00_prerequisites/README.md`
- `chapters/00_prerequisites/principle.md`
- `chapters/01_warp_basics/README.md`
- `chapters/01_warp_basics/principle.md`
- `chapters/01_warp_basics/source-walkthrough.md`
- `chapters/08_rigid_solvers/README.md`
- `chapters/08_rigid_solvers/principle.md`
- `chapters/08_rigid_solvers/source-walkthrough.md`
- `chapters/08_rigid_solvers/examples.md`
- `chapters/09_variational_solvers/README.md`
- `chapters/09_variational_solvers/principle.md`
- `chapters/09_variational_solvers/source-walkthrough.md`
- `chapters/09_variational_solvers/examples.md`
- `chapters/10_softbody_cloth_cable/README.md`
- `chapters/10_softbody_cloth_cable/principle.md`
- `chapters/10_softbody_cloth_cable/source-walkthrough.md`
- `chapters/10_softbody_cloth_cable/examples.md`
- `chapters/11_mpm/README.md`
- `chapters/11_mpm/principle.md`
- `chapters/11_mpm/source-walkthrough.md`
- `chapters/11_mpm/examples.md`
- `chapters/12_sensors_ik/README.md`
- `chapters/12_sensors_ik/principle.md`
- `chapters/12_sensors_ik/source-walkthrough.md`
- `chapters/12_sensors_ik/examples.md`

## Out of Scope

- Replacing Chapter 02 assets.
- Revising Chapter 03-07 assets.
- Adding images to Chapter 13-16 before those chapters have principle / source / examples coverage.
- Adding images to `source-walkthrough-deep.md` files.
- Rewriting chapter prose except for nearby image insertion.
- Using deterministic SVG, HTML, Pillow, canvas, or local renderer output as final tutorial images.
- Committing prompt logs, temporary generated-image manifests, contact sheets, or native-imagegen debug output.

## Visual Language

Follow the 03 / 04 dense Chinese tutorial-handout direction:

- white background
- thin blue rounded card borders
- dark-blue titles and section headers
- numbered blue badges for stages, roles, checkpoints, learner questions, or misconceptions
- Chinese-first labels
- English/code terms only for exact names taught in the chapter
- compact cards, arrows, flow strips, comparison tables, object ledgers, and bottom recap strips
- concrete subject sketches inside the cards, not text-only boxes
- high information density while remaining readable at normal Markdown size

Every non-admin image must include at least one concrete teaching sketch:

- Chapter 00: simulation step loop, state/control/force vocabulary, rigid-body terms, GPU/Warp mental bridge.
- Chapter 01: CPU loop to Warp kernel, `wp.array`, `wp.launch`, `wp.tid()`, `wp.atomic_add`, `Graph`, tile reuse.
- Chapter 08: shared solver contract, maximal vs generalized coordinates, SemiImplicit / Featherstone / MuJoCo / Kamino route split.
- Chapter 09: cloth prediction-correction loop, XPBD constraint row, VBD vertex/block local solve, Style3D PD/PCG route.
- Chapter 10: cloth surface grid, softbody tetrahedral volume, cable rigid capsule chain, representation-first object families.
- Chapter 11: MPM persistent particles, per-step grid workspace, P2G, grid solve, G2P, particle-vs-mesh distinction.
- Chapter 12: shared FK backbone, sensors as read-side adapters, IK as write-side adapter, contact sensor side ledger, objective solve writing `joint_q`.

Avoid:

- Chapter 02 diorama / tabletop-lab style
- cinematic or photorealistic 3D rendering
- abstract decorative images without a teaching relationship
- fake code blocks, fake terminal UI, or invented API names
- long paragraphs inside images
- tiny labels that are unreadable in Markdown
- visual claims more precise than nearby Markdown

## Generation Path

Use Codex native image generation / built-in imagegen:

- Generate raster PNGs through native image generation.
- Save final assets under the chapter-local `assets/` directory.
- Use stable ASCII filenames with chapter prefixes: `00_`, `01_`, `08_`, `09_`, `10_`, `11_`, `12_`.
- Do not leave project-referenced assets only under `$CODEX_HOME/generated_images`.
- Do not use `scripts/image_gen.py` CLI fallback unless the user explicitly requests it.
- Do not use a local Pillow/canvas/SVG/HTML renderer to produce final tutorial PNGs.

## Coverage Model

Use the same reader-facing coverage pattern established by chapters 03-07:

- README: three entry images where the chapter has a README.
- principle: one image per major conceptual section or adjacent concept group.
- source-walkthrough: overview, beginner path, stage images, and object-ledger recap.
- examples: overview plus the main example anchors and self-check, where an examples file exists.

Expected target counts:

| Chapter | README | Principle | Source walkthrough | Examples | Total |
| --- | ---: | ---: | ---: | ---: | ---: |
| 00 | 3 | 5 | 0 | 0 | 8 |
| 01 | 3 | 8 | 8 | 0 | 19 |
| 08 | 3 | 10 | 8 | 4 | 25 |
| 09 | 3 | 8 | 7 | 4 | 22 |
| 10 | 3 | 8 | 7 | 5 | 23 |
| 11 | 3 | 7 | 8 | 5 | 23 |
| 12 | 3 | 7 | 9 | 6 | 25 |

Total expected new PNGs: 145.

This count is intentionally close to the existing 03-07 coverage density. It should not grow further unless review finds a major section without a visual anchor.

## Calibration Images

Generate these first and review them before expanding each chapter:

- `chapters/00_prerequisites/assets/00_principle_sim_step_loop.png`
- `chapters/01_warp_basics/assets/01_principle_cpu_loop_to_kernel.png`
- `chapters/08_rigid_solvers/assets/08_principle_solver_family_split.png`
- `chapters/09_variational_solvers/assets/09_principle_prediction_correction_loop.png`
- `chapters/10_softbody_cloth_cable/assets/10_principle_representation_family_map.png`
- `chapters/11_mpm/assets/11_principle_particle_grid_loop.png`
- `chapters/12_sensors_ik/assets/12_principle_read_write_backbone.png`

Each calibration must pass:

- visually matches chapter 03 / 04 dense Chinese tutorial-handout style
- contains concrete topic sketches
- uses short readable Chinese labels
- uses only real chapter terms and API names
- avoids fake code, fake terminal UI, SVG, HTML, and photorealistic 3D
- keeps exact source semantics in Markdown

## Review Loop

For each chapter batch:

1. Generate native-imagegen PNGs into the chapter `assets/` directory.
2. Insert Markdown image references near the relevant heading or stage.
3. Check that every Markdown image reference points to an existing asset.
4. Run `file` checks on every new PNG.
5. Check dimensions are consistent with the chapter 03 / 04 native-imagegen families.
6. Review style consistency against `chapter-visual-style-guide.md` and existing chapter 03 / 04 images.
7. Review source-truth risk: no generated image should contradict nearby Markdown or invent exact implementation detail.
8. Regenerate weak images with targeted prompts rather than accepting a bad batch.

## Acceptance Criteria

The catch-up pass is complete when:

- chapters 00, 01, and 08-12 have local `assets/` directories with the expected PNG count
- README / principle / source-walkthrough / examples pages in scope have reader-facing image references
- all new image references point to existing PNG files
- all new assets are raster PNGs
- new images follow the chapter 03 / 04 Chinese tutorial-handout direction, not the chapter 02 diorama direction
- non-admin images contain concrete topic sketches, not pure text cards
- `source-walkthrough-deep.md` files remain untouched
- chapters 02-07 assets and Markdown paths remain untouched
- chapters 13-16 remain unchanged until their prose is expanded
- verification, visual review, source-truth review, commit, and push are completed
