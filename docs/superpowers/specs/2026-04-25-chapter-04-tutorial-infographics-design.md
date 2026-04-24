# Chapter 04 Tutorial Infographics Design

## Goal

Generate and integrate a complete set of chapter 04 tutorial-style PNG images for the reader-facing chapter pages, using chapter 03 as the first complete visual style sample and following `docs/superpowers/specs/chapter-visual-style-guide.md`.

Chapter 04 should become the second application of the dense Chinese learning-handout direction. The images should teach how external scene / asset input becomes Newton `Model` static arrays, without turning the chapter into an OpenUSD authoring tutorial or a decorative USD scene gallery.

## User Direction

- Reuse the chapter 03 visual system as the template for chapter 04.
- Start with a calibration infographic before generating the full batch.
- After calibration review, batch-generate PNG tutorial images.
- Integrate accepted PNGs into `README.md`, `principle.md`, and `source-walkthrough.md`.
- Keep `source-walkthrough-deep.md` out of this image pass unless a later task explicitly asks for deep-walkthrough visuals.

## Scope

In scope:

- `chapters/04_scene_usd/README.md`
- `chapters/04_scene_usd/principle.md`
- `chapters/04_scene_usd/source-walkthrough.md`
- `chapters/04_scene_usd/assets/`

Out of scope:

- `chapters/04_scene_usd/source-walkthrough-deep.md`
- complete OpenUSD / MJCF / URDF authoring diagrams
- importer edge-case diagrams for mesh approximation, remeshing, layers, payloads, variants, or every schema branch
- articulation dynamics and collision algorithm visuals, which belong to later chapters

## Chapter 04 Teaching Spine

The image set should reinforce the same spine that chapter 04 already teaches:

```text
scene / asset input
-> add_usd(...)
-> importer identifies body / joint / shape / local pose
-> schema resolver normalizes authored attrs
-> builder accumulates Newton-side rows
-> finalize()
-> Model static arrays
-> onward routes to 05 articulation and 06 collision
```

The reader should leave with one stable bridge model: `Model` is not hand-written and is not the raw scene graph. It is the frozen result of a bounded translation pipeline.

## Core Learner Questions

The images should answer these questions visually:

- If chapter 03 taught `shape_transform`, `joint_X_p / joint_X_c`, `body_com`, and `body_inertia`, where do those fields come from?
- Which scene graph objects matter on the first pass: body, joint, shape, site, local pose, authored attrs, and mass properties?
- Why is `add_usd()` only the public entry, while importer / `parse_usd()` owns the main read-and-translate path?
- What is `schema resolver` responsible for, and what is it explicitly not responsible for?
- How does builder compress scene-side objects into Newton body / joint / shape rows?
- How do authored mass properties and geometry-derived fallback relate before `body_mass / body_com / body_inertia` reach `Model`?
- Why is `finalize()` the point where scene-side language becomes runtime arrays?

## Visual Language

Follow `docs/superpowers/specs/chapter-visual-style-guide.md`.

Use:

- white background with thin blue rounded card borders
- dark-blue title and section headings
- numbered blue badges for stages, object roles, checkpoints, and misconceptions
- Chinese-first labels and explanations
- short exact English/code labels only when necessary, such as `add_usd()`, `parse_usd()`, `schema resolver`, `SchemaResolverManager`, `ShapeConfig`, `MassAPI`, `finalize()`, `Model`, `shape_transform`, `joint_X_p/X_c`, `body_mass`, `body_com`, `body_inertia`
- compact cards, arrows, role ledgers, before/after strips, priority ladders, and bottom recap bars
- small schematic icons for scene graph, importer, resolver, builder, frozen arrays, articulation, and collision

Avoid:

- cinematic USD scenes or photorealistic 3D renderings
- fake source code, fake terminal snippets, or invented USD syntax
- detailed line numbers or file paths inside images
- long paragraphs inside images
- tiny resolver mapping tables that are unreadable in Markdown
- diagrams that imply schema resolver performs parsing, topology construction, solver work, or collision work
- diagrams that imply raw scene graph structure is preserved directly inside `Model`

## Coverage Strategy

Use a full reader-facing coverage pass, while allowing one strong infographic to anchor adjacent short sections.

Expected image count is about 19 PNGs: 3 for `README.md`, 8 for `principle.md`, and 8 for `source-walkthrough.md`. The implementation plan may merge adjacent anchors where the Markdown sections are short and the merged image is clearer.

### README

Create tutorial-navigation images for:

1. Chapter identity: chapter 03 fields now need a source, and chapter 04 explains the scene-to-`Model` bridge.
2. File responsibilities and reading order: `README.md`, `principle.md`, `source-walkthrough.md`, and `source-walkthrough-deep.md`.
3. Completion threshold, chapter scope, explicit non-scope, prerequisites, and expected output.

These should be placed across the README so the file reads like a visual entry map, not a wall of administrative text. The "先带着这张表读" section can share the chapter-identity or scope image if a separate table image would be redundant.

### principle.md

Create conceptual tutorial infographics for:

1. `03` after-bridge plus the chapter 04 five-step chain.
2. Minimal toy scene: ground / box / revolute joint as authored scene objects, then the five-stage compression path.
3. Scene object ledger: body, articulation root, joint, collider / visual shape, site, authored attrs, and the `Model` fields they feed.
4. Public entry vs real importer boundary: `newton/usd.py`, `ModelBuilder.add_usd()`, `parse_usd()`, and the path maps.
5. `schema resolver` as authored-attribute translation boundary, not parser / solver / collision system.
6. Builder accumulation: `add_link()`, `add_joint_*()`, `add_shape_*()`, `ShapeConfig`, and body / joint / shape lists.
7. Mass-property priority: authored `MassAPI`, collider-authored mass info, geometry + density fallback, and final `body_mass / body_com / body_inertia`.
8. `finalize()` boundary plus onward split to `05_rigid_articulation` and `06_collision`.

The first or second principle image should also serve as the calibration image, because it exercises the main visual grammar: scene graph objects, arrows, builder rows, and frozen `Model` arrays.

### source-walkthrough.md

Create source-walkthrough tutorial infographics for:

1. What the walkthrough follows: scene / asset input to `Model` static arrays.
2. Beginner path: five stages and the verification question each stage answers.
3. Stage 1: `add_usd()` is the public entry; `parse_usd()` is the importer handoff.
4. Stage 2: importer extracts body, joint, parent-child relation, and local pose maps.
5. Stage 3: schema resolver normalizes authored attrs into logical keys.
6. Stage 4: shape, material, `ShapeConfig`, and mass property information are compressed into builder rows.
7. Stage 5: `finalize()` freezes builder lists into `Model.shape_* / body_* / joint_*` arrays.
8. Object ledger plus stop-here recap: who produces what, who consumes it, and when the beginner target is complete.

Do not make a separate image for `Go Deeper` unless the implementation pass finds that the end of the page needs a small navigation visual. The deep file remains the source-anchor document.

## Recommended Asset Names

Use stable ASCII filenames with prefix `04_` and suffix `.png`.

Recommended names:

- `04_readme_scene_to_model_spine.png`
- `04_readme_file_roles_reading_order.png`
- `04_readme_completion_scope_prereq.png`
- `04_principle_after_ch03_bridge.png`
- `04_principle_min_scene_five_step_flow.png`
- `04_principle_scene_object_field_ledger.png`
- `04_principle_entry_importer_boundary.png`
- `04_principle_schema_resolver_boundary.png`
- `04_principle_builder_accumulation_map.png`
- `04_principle_mass_property_priority.png`
- `04_principle_finalize_next_chapters.png`
- `04_walkthrough_pipeline_overview.png`
- `04_walkthrough_beginner_path.png`
- `04_walkthrough_stage1_add_usd_parse_usd.png`
- `04_walkthrough_stage2_body_joint_local_pose.png`
- `04_walkthrough_stage3_schema_resolver_keys.png`
- `04_walkthrough_stage4_shape_material_mass_builder.png`
- `04_walkthrough_stage5_finalize_model_arrays.png`
- `04_walkthrough_object_ledger_stop_here.png`

The implementation plan may rename files if a clearer grouping emerges, but all names must stay stable, ASCII, and chapter-prefixed.

## Source-Truth Discipline

Markdown remains authoritative for:

- exact source file names and source snippets
- exact resolver class names and branch details
- precise schema mapping behavior
- importer edge cases
- authored mass property fallback caveats
- distinctions between scene graph, builder rows, and frozen `Model` arrays

Images may compress the idea, but must not contradict the Markdown. In particular:

- A toy scene image must be labeled as a teaching toy, not as a full USD authoring recipe.
- Resolver diagrams must show "logical key translation" rather than implying a second importer.
- Builder diagrams must show accumulation into builder-side rows before `finalize()`, not direct writes into `Model`.
- Mass-property diagrams must show priority / fallback at a conceptual level and leave exact branch logic in prose.
- `Model` array diagrams should use representative field names, not exhaustive field lists.

## Asset Rules

- Final images go under `chapters/04_scene_usd/assets/`.
- Generate raster PNG files only.
- Do not introduce SVG or HTML image fallbacks.
- Do not optimize for file size unless image payload becomes an operational blocker.
- Prefer reusing one strong summary image across nearby short sections over generating redundant weak images.
- Keep generated images reader-facing; do not place implementation-plan screenshots or prompt artifacts in chapter assets.

## Calibration Image

The first generated image should be:

`chapters/04_scene_usd/assets/04_principle_min_scene_five_step_flow.png`

It should show a minimal teaching scene on the left and the chapter 04 bridge on the right:

```text
toy scene: ground + box + revolute joint + authored attrs
-> importer identifies body / joint / shape / local pose
-> schema resolver translates authored attrs
-> builder accumulates body / joint / shape rows
-> finalize()
-> Model static arrays
```

Acceptance check for the calibration image:

- It looks like chapter 03's dense Chinese tutorial-handout style.
- It is visibly instructional, not decorative.
- It uses short, readable Chinese labels.
- It includes only real code/API terms that chapter 04 already teaches.
- It does not pretend to show exact USD source code or exact line anchors.
- It makes the resolver / builder / finalize boundaries visually distinct.

Only expand to the remaining images after this calibration image passes review.

## Generation Strategy

Start with calibration:

1. Generate `04_principle_min_scene_five_step_flow.png`.
2. Review it against `chapter-visual-style-guide.md` and chapter 03 assets.
3. If it drifts toward decorative 3D art, vague USD branding, unreadable text, or invented code, tighten the prompt and regenerate.
4. Once accepted, generate the remaining images in batches by target file.
5. Integrate Markdown links after each batch, so broken paths and awkward placement are caught early.

Prompt direction:

```text
Create a clean Chinese tutorial infographic for Newton Learn chapter 04.
Style: white background, thin blue rounded cards, dark-blue title, numbered blue badges, compact multi-panel teacher review-sheet layout, friendly small icons, arrows, flow boxes, role ledgers, priority ladders, and bottom recap strip. High information density but readable.
Content: teach how scene / asset input becomes Newton Model static arrays through add_usd(), importer / parse_usd(), schema resolver, builder accumulation, finalize(), and Model fields. Use short Chinese labels. English/code terms allowed only for exact names such as add_usd(), parse_usd(), ShapeConfig, MassAPI, finalize(), Model, shape_transform, joint_X_p/X_c, body_mass, body_com, body_inertia. Do not invent source code or USD syntax. Do not include long paragraphs. Do not use SVG, HTML, screenshot, terminal UI, or photorealistic 3D rendering. Generate raster PNG.
```

## Review Loop

For each batch:

1. Generate images through native image generation.
2. Inspect generated files and reject blank, decorative, unreadable, or source-contradicting images.
3. Integrate accepted PNGs into Markdown.
4. Run path/reference verification.
5. Review style consistency against chapter 03 and `chapter-visual-style-guide.md`.
6. Review source-truth risk against `README.md`, `principle.md`, and `source-walkthrough.md`.
7. Regenerate or adjust Markdown where review finds concrete issues.

## Acceptance Criteria

The pass is complete when:

- `README.md`, `principle.md`, and `source-walkthrough.md` have tutorial-style PNG visual anchors across their reader-facing sections.
- `source-walkthrough-deep.md` remains intentionally out of scope.
- all new chapter-04 inline image references point to existing PNG files under `chapters/04_scene_usd/assets/`.
- all new assets are raster PNG files with stable `04_` filenames.
- no generated image introduces SVG / HTML fallbacks, fake code, or fake terminal UI.
- the image style matches the information-rich Chinese tutorial-handout direction established by chapter 03.
- images teach the scene-to-`Model` bridge without overclaiming exact source semantics.
- Markdown remains the source of truth for exact code snippets, resolver details, fallback branches, and source anchors.
