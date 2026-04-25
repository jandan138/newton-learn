# Chapters 05-07 Native Imagegen Redo Design

## Goal

Replace the current chapters 05, 06, and 07 tutorial PNGs with native-imagegen raster teaching infographics that follow the chapter 03 / 04 tutorial-handout system, while preserving the existing Markdown image paths and source-truth boundaries.

This is a corrective pass. The previous deterministic renderer produced clean but overly mechanical diagrams. The redo must use model-generated raster images with concrete teaching sketches inside the structured handout layout.

## Style Authority

Use these documents as the style contract:

- `docs/superpowers/specs/chapter-visual-style-guide.md`
- `docs/superpowers/specs/2026-04-24-chapter-03-tutorial-infographics-design.md`
- `docs/superpowers/specs/2026-04-25-chapter-04-tutorial-infographics-design.md`

Do not use `docs/superpowers/specs/2026-04-23-chapter-02-native-imagegen-design.md` as the style source for this redo. Chapter 02's 3D diorama direction is explicitly not the target for later chapters.

## User Direction

- Do not ask for more approval before proceeding.
- Use the chapter 03 / 04 documented tutorial infographic style.
- Keep PNG assets, not SVG or HTML fallbacks.
- Use image generation, not a deterministic local drawing renderer.
- If quality or correctness problems appear, use review agents and regenerate until the result is usable.
- Commit and push after path checks, visual review, and source-truth review pass.

## Visual Language

Follow the information-rich Chinese tutorial-handout direction:

- white background
- thin blue rounded card borders
- dark-blue title and section headings
- numbered blue badges for stages, roles, checkpoints, or misconceptions
- Chinese-first short labels
- English/code terms only for exact names already taught by the chapter
- compact cards, arrows, flow strips, role ledgers, comparison tables, and bottom recap strips
- concrete topic sketches inside the cards, not text-only boxes

Concrete sketch requirement:

- Chapter 05 images should show topic-specific sketches such as a two-link arm, joint tree, flat array slices, transform arrows, state layers, motion-subspace arrows, or spatial inertia buffers.
- Chapter 06 images should show topic-specific sketches such as body/world pose, shape proxies, AABB boxes, broad-phase candidate pairs, narrow-phase contact points, `ContactData`, or `Contacts` buffers.
- Chapter 07 images should show topic-specific sketches such as sphere-ground contact, contact normal / tangent frame, row expansion, Jacobian velocity mapping, lever arm torque arrow, Delassus / effective-mass block, or multi-contact box support.
- Administrative README images may be more structural, but still need icons or small visual anchors rather than pure text cards.

Avoid:

- cinematic or photorealistic 3D rendering
- Chapter 02 diorama / tabletop-lab style
- plain deterministic card grids with no concrete teaching drawing
- fake source code, fake terminal UI, or invented API names
- long paragraphs inside images
- tiny unreadable labels
- visual claims that are more precise than nearby Markdown

## Generation Path

Use Codex native image generation / built-in imagegen:

- Generate raster PNGs through native image generation.
- Save final project-bound assets into the existing chapter asset paths.
- Do not leave referenced assets only under `$CODEX_HOME/generated_images`.
- Do not use `scripts/image_gen.py` CLI fallback unless the user explicitly requests it.
- Do not use a local Pillow/canvas/SVG/HTML renderer to produce the final tutorial PNGs.

The existing filenames should be preserved so Markdown integration remains stable. Replacing the PNG content in place is expected.

## Scope

In scope:

- `chapters/05_rigid_articulation/assets/05_*.png`
- `chapters/06_collision/assets/06_*.png`
- `chapters/07_constraints_contacts_math/assets/07_*.png`
- `chapters/05_rigid_articulation/README.md`
- `chapters/05_rigid_articulation/principle.md`
- `chapters/05_rigid_articulation/source-walkthrough.md`
- `chapters/05_rigid_articulation/examples.md`
- `chapters/06_collision/README.md`
- `chapters/06_collision/principle.md`
- `chapters/06_collision/source-walkthrough.md`
- `chapters/06_collision/examples.md`
- `chapters/07_constraints_contacts_math/README.md`
- `chapters/07_constraints_contacts_math/principle.md`
- `chapters/07_constraints_contacts_math/source-walkthrough.md`
- `chapters/07_constraints_contacts_math/examples.md`

Out of scope:

- `source-walkthrough-deep.md` for chapters 05, 06, and 07.
- Rewriting the chapter prose unless a generated image requires a small nearby source-truth fence.
- Renaming Markdown image paths unless an existing filename is wrong.
- Reusing the old deterministic rendering scripts or committing prompt/debug artifacts.

## Chapter Teaching Spines

Chapter 05:

```text
04 scene-to-Model fields
-> flat articulation layout
-> joint-space state
-> FK
-> motion subspace
-> body-space state
-> mass / inertia spatial buffers
-> solver handoff
```

Chapter 06:

```text
body/world state + static shape metadata
-> shape world pose
-> expanded AABB
-> broad-phase candidate pairs
-> narrow-phase contact geometry
-> ContactData
-> write_contact(...)
-> Contacts
```

Chapter 07:

```text
Contacts from chapter 06
-> solver-facing contact block
-> contact frame
-> normal + tangent rows
-> Jacobian velocity map
-> lever-arm angular coupling
-> Delassus / effective mass
-> chapter 08 solver consumption
```

## Calibration Images

Regenerate these first and review before expanding:

- `chapters/05_rigid_articulation/assets/05_principle_flat_articulation_layout.png`
- `chapters/06_collision/assets/06_collision_bridge_map.png`
- `chapters/07_constraints_contacts_math/assets/07_contact_math_bridge_map.png`

Each calibration must pass:

- visually matches chapter 03 / 04 dense Chinese tutorial-handout style
- contains a concrete topic sketch, not only cards and text
- uses short, readable Chinese labels
- includes only real code/API terms from the chapter
- avoids fake code, fake terminal UI, SVG, HTML, and photorealistic 3D
- keeps exact source semantics in Markdown

## Review Loop

For each chapter batch:

1. Generate native-imagegen PNGs in existing asset paths.
2. Inspect generated files and reject blank, decorative, unreadable, or source-contradicting images.
3. Compare against chapter 03 / 04 assets and `chapter-visual-style-guide.md`.
4. Run Markdown image-reference checks.
5. Run `file` checks on every replaced PNG.
6. Use agent review for style consistency, source-truth risk, and missing concrete teaching sketches.
7. Regenerate assets with concrete findings rather than accepting weak images.

## Acceptance Criteria

The redo is complete when:

- all existing chapter 05, 06, and 07 Markdown image references still point to existing PNG files
- final referenced assets are native-imagegen raster PNGs saved under the chapter asset directories
- current PNGs visibly follow the chapter 03 / 04 Chinese tutorial-handout direction
- each non-admin image contains a concrete topic sketch or visual metaphor that helps the learner understand the concept
- no image depends on fake code, fake terminal text, SVG, HTML, or local deterministic rendering
- `source-walkthrough-deep.md` files remain untouched
- path checks, PNG checks, visual review, and source-truth review pass
