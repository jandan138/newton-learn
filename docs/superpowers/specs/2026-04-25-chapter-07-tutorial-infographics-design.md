# Chapter 07 Tutorial Infographics Design

## Goal

Generate and integrate a complete set of chapter 07 tutorial-style PNG images for the reader-facing chapter pages, using chapters 03, 04, 05, and 06 as the established visual samples and following `docs/superpowers/specs/chapter-visual-style-guide.md`.

Chapter 07 should extend the dense Chinese learning-handout direction into the contact-math bridge: how chapter 06's runtime `Contacts` become solver-facing contacts, local contact directions, scalar rows, contact Jacobians, and Delassus / effective-mass intuition before chapter 08 starts discussing solver consumption.

## User Direction

- Continue autonomously without waiting for further approval.
- Reuse the chapter 05 / 06 visual system as the strongest current template for chapter 07.
- Start with a calibration infographic before expanding to the full batch.
- After calibration review, batch-generate PNG tutorial images.
- Integrate accepted PNGs into `README.md`, `principle.md`, `source-walkthrough.md`, and `examples.md`.
- Keep `source-walkthrough-deep.md` out of this image pass unless a later task explicitly asks for deep-walkthrough visuals.
- If generation or review finds a concrete issue, iterate and use reviewer agents rather than stopping.

## Scope

In scope:

- `chapters/07_constraints_contacts_math/README.md`
- `chapters/07_constraints_contacts_math/principle.md`
- `chapters/07_constraints_contacts_math/source-walkthrough.md`
- `chapters/07_constraints_contacts_math/examples.md`
- `chapters/07_constraints_contacts_math/assets/`

Out of scope:

- `chapters/07_constraints_contacts_math/source-walkthrough-deep.md`
- broad-phase or narrow-phase collision algorithm visuals already covered by chapter 06
- full rigid solver algorithm derivations covered by chapter 08
- PADMM, LCP, NCP, KKT, ADMM, warm start, stabilization, bias, or regularization deep dives
- solver-family comparisons across Kamino, MuJoCo, XPBD, or other approaches
- exact source line numbers, full source snippets, or fake code blocks inside images

## Chapter 07 Teaching Spine

The image set should reinforce the same spine that chapter 07 already teaches:

```text
chapter 06 Contacts
-> contact point / normal / margin are still geometry handoff
-> Kamino bridge builds solver-facing ContactsKamino
-> position_A / position_B + gapfunc + frame + material
-> one contact frame: normal + two tangents
-> 1 contact -> 3 rows
-> contact Jacobian = frame direction + lever arm velocity mapping
-> Delassus / effective mass = row-space response through inverse mass / inertia
-> chapter 08 solver consumption
```

The reader should leave with one stable bridge model: chapter 06 answers "where did geometry touch"; chapter 07 answers "which relative motions should be watched or limited"; chapter 08 answers "how a solver consumes those rows and quantities."

## Core Learner Questions

The images should answer these questions visually:

- Why are `Contacts` still a geometry handoff rather than a finished solver object?
- Why does chapter 07 introduce `ContactsKamino` / solver-facing contact blocks?
- How do `position_A`, `position_B`, `gapfunc`, `frame`, and `material` prepare contact geometry for rows?
- Why does one 3D contact usually become `1` normal row plus `2` tangent rows?
- Why is a tangent plane two-dimensional rather than one-dimensional?
- What does a Jacobian row mean before it is treated as a matrix?
- How do contact frame directions and lever arms decide which body translation / rotation affects a row?
- Why does off-center contact naturally produce angular coupling?
- What is the first physical intuition for Delassus / effective mass?
- Why can a box-ground shape pair produce multiple contacts and many rows?
- Why is `D` a row-row response object rather than only a single scalar?
- Where should the reader stop before moving to chapter 08?

## Visual Language

Follow `docs/superpowers/specs/chapter-visual-style-guide.md`.

Use:

- white background with thin blue rounded card borders
- dark-blue title and section headings
- numbered blue badges for stages, contact directions, row blocks, and misconception checkpoints
- Chinese-first labels and explanations
- short exact English/code labels only when necessary, such as `Contacts`, `ContactsKamino`, `position_A`, `position_B`, `gapfunc`, `frame`, `material`, `normal`, `tangent`, `row`, `3 * nc`, `qd`, `J_row`, `J`, `M^-1`, `D = J M^-1 J^T`, `Delassus`, and `08_rigid_solvers`
- compact cards, arrows, local-frame diagrams, row expansion strips, two-column comparisons, cause-effect maps, source-to-runtime handoff ledgers, and bottom recap bars
- small schematic icons for contact point, normal arrow, tangent plane, three row slots, Jacobian mapping, lever arm, inverse-mass response, row-row coupling, sphere-ground, and box-ground

Avoid:

- photorealistic simulation renders or decorative physics scenes
- fake source code, fake terminal snippets, or invented API calls
- source paths or line numbers inside images
- equations that become the visual focus before the intuition is established
- diagrams that imply chapter 07 solves the constraints
- diagrams that imply `Contacts` is already Kamino's final internal solver format
- diagrams that imply one contact is one scalar row
- diagrams that imply one shape pair always produces one contact
- diagrams that imply Delassus is always a single number instead of row-space coupling
- visuals that re-teach chapter 06 collision detection instead of using it as input
- visuals that preview chapter 08 solver-family details before the reader knows the row objects

## Coverage Strategy

Use a full reader-facing coverage pass, while allowing one strong infographic to anchor adjacent short sections.

Expected image count is about 22 PNGs: 3 for `README.md`, 7 for `principle.md`, 8 for `source-walkthrough.md`, and 4 for `examples.md`. The implementation plan may merge adjacent anchors where the Markdown sections are short and the merged image is clearer.

### README

Create tutorial-navigation images for:

1. Chapter identity: chapter 06 collision geometry now becomes contact math, but chapter 08 still owns solving.
2. File responsibilities and reading order: `README.md -> source-walkthrough.md -> principle.md -> examples.md -> source-walkthrough-deep.md -> 08`, matching the chapter 07 README's first-pass path.
3. Completion threshold, chapter scope, explicit non-scope, prerequisites, GAMES103 bridge, and expected output.

These should be placed across the README so the file reads like a visual entry map. The GAMES103 and reading-order sections can be covered by nearby scope / file-role images if a separate table image would be redundant.

### principle.md

Create conceptual tutorial infographics for:

1. Opening bridge: `Contacts -> contact directions -> rows -> Jacobian -> Delassus -> 08`.
2. One contact becomes three rows: one normal direction plus two tangent directions.
3. Jacobian first meaning: which body motions change a contact direction's relative velocity.
4. Lever arm and angular coupling: off-center contact makes rotation visible.
5. Delassus / effective mass: row-space response through inverse mass and inertia.
6. Box-ground: one pair can produce multiple contacts, and each contact contributes rows.
7. Engineering caveat plus next-chapter handoff: `Contacts` is not final Kamino solver format, and chapter 08 owns actual solving.

The first principle image should serve as the calibration image.

### source-walkthrough.md

Create source-walkthrough tutorial infographics for:

1. What the walkthrough follows: runtime `Contacts` to row-space response.
2. Beginner path: five stages and the verification question each stage answers.
3. Stage 1: `Contacts` stores geometry fields, not motion constraints.
4. Stage 2: solver-facing contact block adds world-space points, `gapfunc`, contact `frame`, and `material`.
5. Stage 3: contact frame expands one active contact into three rows.
6. Stage 4: contact Jacobian uses frame direction plus lever arm to map body velocities to row velocities.
7. Stage 5: Delassus accumulates linear and angular inverse-mass response into row-row coupling.
8. Object ledger plus stop-here recap.

Do not make a separate image for `Go Deeper` unless implementation review finds that the end of the page needs a small navigation visual. The deep file remains the source-anchor document.

### examples.md

Create example-observation infographics for:

1. Examples overview: why `basic_shapes` is an observation task, not solver tuning.
2. `sphere-ground`: one visible contact point still implies one local frame and three solver rows.
3. `box-ground`: multiple contact points and off-center lever arms make angular coupling visible.
4. Self-check and file handoff: keep `rigid_contact_count` as geometry count, then return to `principle.md` / `source-walkthrough.md`.

These images should stay lighter than the principle / walkthrough diagrams, because examples.md already uses observation tables.

## Recommended Asset Names

Use stable ASCII filenames with prefix `07_` and suffix `.png`.

Recommended names:

- `07_readme_contact_math_spine.png`
- `07_readme_file_roles_reading_order.png`
- `07_readme_completion_scope_prereq.png`
- `07_contact_math_bridge_map.png`
- `07_principle_contact_to_three_rows.png`
- `07_principle_jacobian_motion_map.png`
- `07_principle_lever_arm_angular_coupling.png`
- `07_principle_delassus_effective_mass.png`
- `07_principle_box_ground_multi_contacts.png`
- `07_principle_solver_facing_next_chapter.png`
- `07_walkthrough_pipeline_overview.png`
- `07_walkthrough_beginner_path.png`
- `07_walkthrough_stage1_contacts_geometry_handoff.png`
- `07_walkthrough_stage2_solver_facing_contact_block.png`
- `07_walkthrough_stage3_contact_rows_three_directions.png`
- `07_walkthrough_stage4_jacobian_frame_lever_arm.png`
- `07_walkthrough_stage5_delassus_row_space_response.png`
- `07_walkthrough_object_ledger_stop_here.png`
- `07_examples_overview_observation_tasks.png`
- `07_examples_sphere_ground_one_contact_three_rows.png`
- `07_examples_box_ground_multiple_contacts_lever_arm.png`
- `07_examples_self_check_handoff.png`

The implementation plan may rename files if a clearer grouping emerges, but all names must stay stable, ASCII, and chapter-prefixed.

## Source-Truth Discipline

Markdown remains authoritative for:

- exact source file names and source snippets
- exact `Contacts` and `ContactsKamino` field semantics
- exact conversion and bridge behavior
- exact row-count source details and optional branches
- exact Jacobian / Delassus kernels and implementation variants
- solver algorithm details that belong to chapter 08

Images may compress the idea, but must not contradict the Markdown. In particular:

- The `sphere-ground` visual must be labeled as a teaching toy, not as the whole contact solver.
- `Contacts` diagrams must show geometry handoff, not final solver internals.
- `ContactsKamino` diagrams must show a bridge object, not a new collision detector.
- Row diagrams must show `1 normal + 2 tangents`, not one scalar per contact.
- Jacobian diagrams must tie row direction to body motion and lever arm without pretending to be full source code.
- Delassus diagrams must show row-space response and possible row-row coupling.
- Box-ground diagrams must distinguish shape pair count, contact count, and row count.

## Asset Rules

- Final images go under `chapters/07_constraints_contacts_math/assets/`.
- Generate raster PNG files only.
- Do not introduce SVG or HTML image fallbacks.
- Do not optimize for file size unless image payload becomes an operational blocker.
- Prefer reusing one strong summary image across nearby short sections over generating redundant weak images.
- Keep generated images reader-facing; do not place implementation-plan screenshots or prompt artifacts in chapter assets.

## Calibration Image

The first generated image should be:

`chapters/07_constraints_contacts_math/assets/07_contact_math_bridge_map.png`

It should show the minimal chapter 07 contact-math bridge:

```text
Contacts geometry handoff
-> solver-facing contact block
-> contact frame: normal + two tangents
-> 1 contact -> 3 rows
-> Jacobian maps qd to row velocity
-> Delassus / effective mass
-> 08 solver consumption
```

Acceptance check for the calibration image:

- It looks like chapters 03 / 04 / 05 / 06 dense Chinese tutorial-handout style.
- It is visibly instructional, not decorative contact art.
- It uses short, readable Chinese labels.
- It includes only real code/API terms that chapter 07 already teaches.
- It does not pretend to show exact source code or exact line anchors.
- It keeps `Contacts`, solver-facing contact, rows, `J`, `D`, and chapter 08 boundaries visually distinct.
- It clearly says chapter 07 builds the math objects but does not solve them.

Only expand to the remaining images after this calibration image passes review.

## Generation Strategy

Start with calibration:

1. Generate or deterministically render `07_contact_math_bridge_map.png`.
2. Review it against `chapter-visual-style-guide.md` and chapter 06 assets.
3. If it drifts toward decorative art, unreadable text, fake source, or contact/row/solver confusion, tighten the prompt or re-render.
4. Once accepted, generate the remaining images in batches by target file.
5. Integrate Markdown links after each batch, so broken paths and awkward placement are caught early.

If review finds source-truth errors in exact labels, stage boundaries, contact-field names, row counts, or contact-math/solver boundaries, regenerate or deterministically re-render those PNGs as raster assets. Exact educational claims are more important than preserving the initial generative output.

Prompt direction:

```text
Create a clean Chinese tutorial infographic for Newton Learn chapter 07.

Visual style: white background, thin blue rounded card borders, dark-blue title typography, numbered blue circular badges, compact multi-panel layout, small friendly schematic icons, arrows, flow boxes, table-like comparison cards, clear whitespace, high information density but readable.

Content: teach how chapter 06 `Contacts` become contact directions, `1 normal + 2 tangents`, rows, Jacobian velocity mappings, and Delassus / effective-mass intuition. Keep chapter 08 solver solving out of scope. Use only real terms from the chapter. Do not invent code. Do not include long paragraphs. Generate a raster PNG.
```

## Review Checklist

Before final commit:

- Every Markdown image reference resolves to an existing PNG.
- All `chapters/07_constraints_contacts_math/assets/07_*.png` files are real PNG files.
- `source-walkthrough-deep.md` is unchanged unless a concrete review issue requires it.
- The first principle image works as the calibration image.
- README visuals establish the chapter boundary without repeating principle details too early.
- Principle visuals explain contact frame, rows, Jacobian, lever arm, Delassus, multi-contact, and next-chapter handoff.
- Source-walkthrough visuals follow the five-stage path in the Markdown.
- Examples visuals stay observational and avoid solver tuning advice.
- No image claims chapter 07 solves the constraints.
- No image claims `Contacts` is the final solver format.
- No image claims one contact equals one row.
- No image claims one shape pair equals one contact.
- `git diff --check` is clean for the affected files.
