# Chapter 06 Tutorial Infographics Design

## Goal

Generate and integrate a complete set of chapter 06 tutorial-style PNG images for the reader-facing chapter pages, using chapters 03, 04, and 05 as the established visual samples and following `docs/superpowers/specs/chapter-visual-style-guide.md`.

Chapter 06 should extend the dense Chinese learning-handout direction into the collision layer: how `body_q` and static shape metadata become shape world poses, broad-phase candidate pairs, narrow-phase contact geometry, and the unified `Contacts` handoff.

## User Direction

- Continue autonomously without waiting for further approval.
- Reuse the chapter 05 visual system as the strongest current template for chapter 06.
- Start with a calibration infographic before expanding to the full batch.
- After calibration review, batch-generate PNG tutorial images.
- Integrate accepted PNGs into `README.md`, `principle.md`, `source-walkthrough.md`, and `examples.md`.
- Keep `source-walkthrough-deep.md` out of this image pass unless a later task explicitly asks for deep-walkthrough visuals.
- If generation or review finds a concrete issue, iterate and use reviewer agents rather than stopping.

## Scope

In scope:

- `chapters/06_collision/README.md`
- `chapters/06_collision/principle.md`
- `chapters/06_collision/source-walkthrough.md`
- `chapters/06_collision/examples.md`
- `chapters/06_collision/assets/`

Out of scope:

- `chapters/06_collision/source-walkthrough-deep.md`
- full GJK / MPR / EPA / SDF / hydroelastic derivations
- full broad-phase complexity comparisons
- Jacobian, Delassus, complementarity, friction-cone, or solver-specific contact math
- rigid solver family comparison, XPBD / Featherstone / variational solver internals
- mesh preprocessing, remeshing, SDF build, or low-level geometry asset construction

## Chapter 06 Teaching Spine

The image set should reinforce the same spine that chapter 06 already teaches:

```text
05 body/world state + 04 shape metadata
-> state.body_q + model.shape_transform / shape_type / gap / margin
-> shape world pose and expanded AABB
-> broad phase candidate shape pairs
-> narrow phase routing by shape_type
-> ContactData as unified contact geometry record
-> write_contact(...)
-> Contacts
-> onward routes to 07 contact math and 08 solver consumption
```

The reader should leave with one stable bridge model: body does not directly collide with body. Body provides runtime motion; shape provides geometry; broad phase only builds a conservative maybe-list; narrow phase turns candidate shape pairs into contact geometry; `Contacts` is the common runtime buffer consumed by later chapters.

## Core Learner Questions

The images should answer these questions visually:

- What gap remains after chapter 05 has produced `body_q / body_qd`?
- Why does collision attach body motion to shape geometry instead of testing body pairs directly?
- Which `Model.shape_*` fields matter in the first collision pass?
- How does `body_q + shape_transform` become a shape world pose?
- Why do `margin` and `gap` appear before narrow phase?
- Why is broad phase a conservative candidate-pair generator, not a contact generator?
- Why does narrow phase branch by `shape_type`, and why do those branches still converge?
- What is the difference between `ContactData` and final `Contacts` arrays?
- Which contact fields matter for the chapter 07 / 08 handoff?
- What can `basic_shapes` and `pyramid` reveal without turning the examples into solver tuning tasks?

## Visual Language

Follow `docs/superpowers/specs/chapter-visual-style-guide.md`.

Use:

- white background with thin blue rounded card borders
- dark-blue title and section headings
- numbered blue badges for stages, object roles, checkpoints, and misconceptions
- Chinese-first labels and explanations
- short exact English/code labels only when necessary, such as `state.body_q`, `model.shape_transform`, `shape_body`, `shape_type`, `shape_gap`, `shape_margin`, `compute_shape_aabbs(...)`, `geom_transform`, `broad phase`, `candidate pairs`, `narrow phase`, `ContactData`, `write_contact(...)`, `Contacts`, `rigid_contact_count`, `rigid_contact_shape0/1`, `rigid_contact_point0/1`, and `rigid_contact_normal`
- compact cards, arrows, role ledgers, before/after strips, maybe-list diagrams, branch-and-converge maps, field watchlists, and bottom recap bars
- small schematic icons for world pose, sphere / box / capsule / plane shapes, AABB, pair list, narrow-phase routing, contact normal, contact point, and runtime buffer

Avoid:

- photorealistic collision scenes or decorative robot renders
- fake source code, fake terminal snippets, or invented API calls
- detailed line numbers or source file paths inside images
- full geometry algorithm derivations or tiny math tables
- diagrams that imply broad phase creates final contacts
- diagrams that imply `body_qd` is the primary broad-phase input
- diagrams that imply every body pair maps to exactly one shape pair
- diagrams that imply every shape pair creates exactly one contact
- diagrams that imply solver consumes raw narrow-phase branch internals instead of `Contacts`

## Coverage Strategy

Use a full reader-facing coverage pass, while allowing one strong infographic to anchor adjacent short sections.

Expected image count is about 22 PNGs: 3 for `README.md`, 7 for `principle.md`, 8 for `source-walkthrough.md`, and 4 for `examples.md`. The implementation plan may merge adjacent anchors where the Markdown sections are short and the merged image is clearer.

### README

Create tutorial-navigation images for:

1. Chapter identity: chapter 05 body state plus chapter 04 shape metadata now enter collision.
2. File responsibilities and reading order: `README.md -> source-walkthrough.md -> principle.md -> examples.md -> source-walkthrough-deep.md -> 07 / 08`, matching the chapter 06 README's first-pass path.
3. Completion threshold, chapter scope, explicit non-scope, prerequisites, GAMES103 bridge, and expected output.

These should be placed across the README so the file reads like a visual entry map. The GAMES103 and reading-order sections can be covered by nearby scope / file-role images if a separate table image would be redundant.

### principle.md

Create conceptual tutorial infographics for:

1. Opening bridge: `body_q + shape metadata -> shape world pose -> candidate pairs -> narrow phase -> Contacts`.
2. Body is not the direct collision object: body gives motion, shape gives geometry, one body can hold multiple shapes.
3. Broad phase: expanded AABB plus world/group/filter rules produce only a conservative maybe-list.
4. Narrow phase: candidate pair becomes contact geometry with normal, point, signed distance, gap, and margin.
5. Shape-type routing: simple primitives, convex pairs, mesh / SDF branches, then convergence.
6. `ContactData` and `Contacts`: geometry result compressed into a unified runtime handoff object.
7. Chapter 06 onward split to chapter 07 contact math and chapter 08 solver consumption.

The first principle image should serve as the calibration image. It already exists as `06_collision_bridge_map.png`, but should be regenerated or deterministically re-rendered to match the chapter 05 sample and the current source-truth wording.

### source-walkthrough.md

Create source-walkthrough tutorial infographics for:

1. What the walkthrough follows: body motion plus `Model.shape_*` to `Contacts`.
2. Beginner path: five stages and the verification question each stage answers.
3. Stage 1: `Model.collide()` connects runtime `State` and static shape data to `CollisionPipeline`.
4. Stage 2: `compute_shape_aabbs(...)` computes shape world transform, expanded AABB, `geom_data`, and `geom_transform`.
5. Stage 3: broad phase writes `candidate_pair` and `candidate_pair_count`, not contact geometry.
6. Stage 4: narrow phase routes by `shape_type` and converges on `ContactData`.
7. Stage 5: `write_contact(...)` writes `ContactData` into solver-readable `Contacts`.
8. Object ledger plus stop-here recap.

Do not make a separate image for `Go Deeper` unless implementation review finds that the end of the page needs a small navigation visual. The deep file remains the source-anchor document.

### examples.md

Create example-observation infographics for:

1. Examples overview: why `basic_shapes` and `pyramid` are observation tasks, not solver tuning tasks.
2. `basic_shapes`: sphere-ground teaches `body_q -> shape world pose -> first contact`.
3. `pyramid --test --pyramid-size 4 --num-pyramids 1`: contact sets grow through many boxes, many candidate pairs, and multiple contacts per pair.
4. Contact field watchlist: `rigid_contact_count`, `rigid_contact_shape0/1`, `rigid_contact_point0/1`, `rigid_contact_normal`, and the handoff back to `principle.md` / `source-walkthrough.md`.

These images should stay lighter than the principle / walkthrough diagrams, because examples.md already uses observation tables.

## Recommended Asset Names

Use stable ASCII filenames with prefix `06_` and suffix `.png`.

Recommended names:

- `06_readme_collision_spine.png`
- `06_readme_file_roles_reading_order.png`
- `06_readme_completion_scope_prereq.png`
- `06_collision_bridge_map.png`
- `06_principle_shape_not_body_boundary.png`
- `06_principle_broad_phase_maybe_list.png`
- `06_principle_narrow_phase_contact_geometry.png`
- `06_principle_shape_type_routing.png`
- `06_principle_contacts_handoff_buffer.png`
- `06_principle_next_chapters_map.png`
- `06_walkthrough_pipeline_overview.png`
- `06_walkthrough_beginner_path.png`
- `06_walkthrough_stage1_model_state_shape_data.png`
- `06_walkthrough_stage2_compute_shape_aabbs.png`
- `06_walkthrough_stage3_broad_phase_candidate_pairs.png`
- `06_walkthrough_stage4_narrow_phase_contactdata.png`
- `06_walkthrough_stage5_write_contact_contacts.png`
- `06_walkthrough_object_ledger_stop_here.png`
- `06_examples_overview_observation_tasks.png`
- `06_examples_basic_shapes_sphere_ground.png`
- `06_examples_pyramid_contact_set_growth.png`
- `06_examples_contact_fields_watchlist.png`

The implementation plan may rename files if a clearer grouping emerges, but all names must stay stable, ASCII, and chapter-prefixed.

## Source-Truth Discipline

Markdown remains authoritative for:

- exact source file names and source snippets
- exact broad-phase implementation variants and kernel details
- exact narrow-phase branch coverage
- `ContactData` field semantics and downstream conversion details
- exact contact-buffer field names and edge cases
- distinctions among collision geometry, contact math, and solver consumption

Images may compress the idea, but must not contradict the Markdown. In particular:

- A ball-ground or box-stack image must be labeled as a teaching toy, not as a full collision algorithm.
- Body / shape diagrams must show body providing motion and shape providing geometry.
- Broad-phase diagrams must show maybe-list output only.
- Narrow-phase diagrams must show branch-and-converge behavior without pretending every branch has the same algorithm.
- Contact diagrams must distinguish `ContactData` from final `Contacts`.
- Example diagrams must stay observational; they should not become solver tuning, control, or numerical stability advice.

## Asset Rules

- Final images go under `chapters/06_collision/assets/`.
- Generate raster PNG files only.
- Do not introduce SVG or HTML image fallbacks.
- Do not optimize for file size unless image payload becomes an operational blocker.
- Prefer reusing one strong summary image across nearby short sections over generating redundant weak images.
- Keep generated images reader-facing; do not place implementation-plan screenshots or prompt artifacts in chapter assets.
- `chapters/06_collision/assets/06_collision_bridge_map.png` may be replaced with a style-aligned raster image, because it is the existing principle calibration slot.

## Calibration Image

The first generated image should be:

`chapters/06_collision/assets/06_collision_bridge_map.png`

It should show the minimal chapter 06 collision bridge:

```text
state.body_q + model.shape_transform / shape_type
-> shape world pose
-> expanded AABB / margin / gap
-> broad phase candidate pairs
-> narrow phase contact geometry
-> ContactData
-> write_contact(...)
-> Contacts
```

Acceptance check for the calibration image:

- It looks like chapters 03 / 04 / 05 dense Chinese tutorial-handout style.
- It is visibly instructional, not decorative collision art.
- It uses short, readable Chinese labels.
- It includes only real code/API terms that chapter 06 already teaches.
- It does not pretend to show exact source code or exact line anchors.
- It keeps body motion, shape geometry, broad phase, narrow phase, and `Contacts` boundaries visually distinct.
- It clearly says broad phase produces a maybe-list, not final contacts.

Only expand to the remaining images after this calibration image passes review.

## Generation Strategy

Start with calibration:

1. Generate or deterministically re-render `06_collision_bridge_map.png`.
2. Review it against `chapter-visual-style-guide.md` and chapter 05 assets.
3. If it drifts toward decorative art, unreadable text, fake source, or broad-phase/contact confusion, tighten the prompt or re-render.
4. Once accepted, generate the remaining images in batches by target file.
5. Integrate Markdown links after each batch, so broken paths and awkward placement are caught early.

If review finds source-truth errors in exact labels, stage boundaries, contact-field names, or collision/contact/solver boundaries, regenerate or deterministically re-render those PNGs as raster assets. Exact educational claims are more important than preserving the initial generative output.

Prompt direction:

```text
Create a clean Chinese tutorial infographic for Newton Learn chapter 06.
Style: white background, thin blue rounded cards, dark-blue title, numbered blue badges, compact multi-panel teacher review-sheet layout, friendly small icons, arrows, maybe-list diagrams, branch-and-converge maps, field watchlists, and bottom recap strip. High information density but readable.
Content: teach how state.body_q and Model.shape_* metadata become shape world pose, expanded AABB, broad-phase candidate pairs, narrow-phase ContactData, write_contact(...), and final Contacts. Use short Chinese labels. English/code terms allowed only for exact names such as state.body_q, model.shape_transform, shape_body, shape_type, shape_gap, shape_margin, compute_shape_aabbs(...), broad phase, candidate pairs, narrow phase, ContactData, write_contact(...), Contacts, rigid_contact_count, rigid_contact_shape0/1, rigid_contact_point0/1, rigid_contact_normal. Do not invent source code. Do not include long paragraphs. Do not use SVG, HTML, screenshot, terminal UI, photorealistic rendering, or full GJK/MPR/EPA derivation. Generate raster PNG.
```

## Review Loop

For each batch:

1. Generate or render images.
2. Inspect generated files and reject blank, decorative, unreadable, or source-contradicting images.
3. Integrate accepted PNGs into Markdown.
4. Run path/reference verification.
5. Review style consistency against chapter 05 and `chapter-visual-style-guide.md`.
6. Review source-truth risk against `README.md`, `principle.md`, `source-walkthrough.md`, and `examples.md`.
7. Regenerate or adjust Markdown where review finds concrete issues.

## Acceptance Criteria

The pass is complete when:

- `README.md`, `principle.md`, `source-walkthrough.md`, and `examples.md` have tutorial-style PNG visual anchors across their reader-facing sections.
- `source-walkthrough-deep.md` remains intentionally out of scope.
- all new chapter-06 inline image references point to existing PNG files under `chapters/06_collision/assets/`.
- all new assets are raster PNG files with stable `06_` filenames.
- no generated image introduces SVG / HTML fallbacks, fake code, fake terminal UI, or fake algorithm derivations.
- the image style matches the information-rich Chinese tutorial-handout direction established by chapters 03 / 04 / 05.
- images teach the collision bridge without overclaiming exact source semantics.
- Markdown remains the source of truth for exact code snippets, field semantics, branch details, and source anchors.
