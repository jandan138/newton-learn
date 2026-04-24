# Chapter 03 Tutorial Infographics Design

## Goal

Generate and integrate a complete set of chapter 03 tutorial-style PNG images for the reader-facing chapter pages, using the information-rich Chinese learning-handout style defined in `docs/superpowers/specs/chapter-visual-style-guide.md`.

Chapter 02 does not need revision. Chapter 03 should become the first full application of the new preferred visual direction: dense, readable, structured tutorial infographics rather than decorative 3D-diorama images.

## User Direction

- Proceed autonomously without stopping for further approval.
- Follow the recommended design unless a hard blocker appears.
- If quality or correctness problems appear, use multiple agents and iterate until there is a usable result.
- Generate PNG tutorial images, not SVG or HTML replacements.
- Use the information-rich tutorial-diagram style, but do not force every image into a fixed "six questions" layout.

## Scope

In scope:

- `chapters/03_math_geometry/README.md`
- `chapters/03_math_geometry/principle.md`
- `chapters/03_math_geometry/source-walkthrough.md`
- `chapters/03_math_geometry/assets/`

Out of scope:

- `chapters/03_math_geometry/source-walkthrough-deep.md`

The deep walkthrough remains a source-anchor document. It may reference already-created chapter images later if useful, but this pass should not create deep-specific visuals.

## Chapter 03 Teaching Spine

The images should reinforce the same spine that chapter 03 already teaches:

```text
local frame relationships
-> transform chains
-> quaternion / orientation reading
-> spatial quantities
-> shape representation and GeoType dispatch
-> inertia as the bridge from geometry to dynamics
-> onward routes to 04 / 05 / 06
```

The image set should make this chain memorable without moving exact source truth out of Markdown.

## Visual Language

Follow `docs/superpowers/specs/chapter-visual-style-guide.md`.

Use:

- white background with thin blue rounded card borders
- dark-blue title and section headings
- numbered blue badges for steps, concepts, stages, or checkpoints
- Chinese-first labels and explanations
- short exact English/code labels only when necessary, such as `frame`, `wp.transform`, `body_q`, `body_qd`, `GeoType`, `shape_transform`, `body_com`, `inertia`
- compact cards, arrows, comparison strips, role tables, and bottom recap bars
- high information density with clear grouping

Avoid:

- cinematic 3D scenes
- standalone decorative images
- fake source code or fake terminal snippets
- long paragraphs inside images
- tiny unreadable labels
- visual claims that are more precise than the nearby Markdown

## Coverage Strategy

Use a full reader-facing coverage pass, while allowing reuse where one infographic naturally anchors adjacent short sections.

### README

Create tutorial-navigation images for:

1. Chapter identity and role after chapter 02.
2. File responsibilities and reading order.
3. Completion threshold / expected output checklist.
4. Scope and explicit non-scope.
5. Prerequisites and GAMES103 bridge.

These can be placed across the README sections so every major reader-facing section has a nearby visual anchor, without creating redundant one-card images for very short administrative sections.

### principle.md

Create one conceptual tutorial infographic for each major concept section:

1. Why chapter 03 exists after chapter 02.
2. `frame` as "who is relative to whom".
3. `wp.transform` as translation plus rotation and local-chain composition.
4. quaternion as an orientation work container.
5. spatial quantities as frame-attached motion and force.
6. shape representation categories.
7. inertia as the geometry-to-articulation bridge.
8. onward split to chapters 04, 05, and 06.

### source-walkthrough.md

Create source-walkthrough tutorial infographics for:

1. What the walkthrough follows.
2. One-screen chapter map.
3. Beginner path.
4. Stage 1: local relation ledger.
5. Stage 2: transform chain to `body_q` and shape world pose.
6. Stage 3: `body_qd` and spatial helper wrappers.
7. Stage 4: `GeoType` as representation / query dispatch key.
8. Stage 5: geometry-to-inertia compression.
9. Object ledger.
10. Stop-here / go-deeper summary.

## Source-Truth Discipline

Markdown remains authoritative for:

- exact source file names and line anchors
- exact code snippets
- precise call order
- implementation caveats
- distinctions that require exact wording, such as body frame vs shape local frame

Images may compress the idea, but must not contradict the Markdown. When an image simplifies a source path, the surrounding prose should keep the exact version clear.

## Asset Rules

- Final images go under `chapters/03_math_geometry/assets/`.
- Use stable ASCII filenames with the prefix `03_` and suffix `.png`.
- Do not introduce SVG or HTML image fallbacks.
- Do not optimize for file size unless image payload becomes an operational blocker.
- Prefer reusing one strong summary image across nearby short sections over generating redundant weak images.

## Generation Strategy

Start with a calibration image before expanding:

1. Generate the chapter-03 spine overview infographic.
2. Review whether it matches the desired information-rich tutorial style.
3. If the result drifts toward decorative art, tighten prompts and regenerate before proceeding.
4. Generate the remaining images in coherent batches by file.
5. Integrate Markdown links after each batch, so broken paths are caught early.

Prompt direction:

```text
Create a clean Chinese tutorial infographic for Newton Learn chapter 03.
Style: white background, thin blue rounded cards, dark-blue title, numbered blue badges, compact multi-panel teaching layout, friendly icons, arrows, comparison tables, flow strips, high information density but readable.
Content: teach the named chapter-03 concept, flow, object relationship, or misconception assigned in the implementation plan. Use short labels only. English/code terms allowed only for exact API or object names. Do not invent code. Do not include long paragraphs. Do not use SVG, HTML, screenshot, terminal UI, or photorealistic 3D rendering. Generate raster PNG.
```

## Review Loop

For each batch:

1. Generate images through native image generation.
2. Inspect generated files and reject obviously broken images.
3. Integrate accepted PNGs into Markdown.
4. Run path/reference verification.
5. Use agent review for style consistency, source-truth risk, and coverage gaps.
6. Regenerate or adjust Markdown where the review finds concrete issues.

## Acceptance Criteria

The pass is complete when:

- `README.md`, `principle.md`, and `source-walkthrough.md` have tutorial-style PNG visual anchors across their reader-facing sections.
- `source-walkthrough-deep.md` remains intentionally out of scope.
- all new chapter-03 inline image references point to existing PNG files under `chapters/03_math_geometry/assets/`.
- no generated image introduces SVG/HTML fallback.
- the image style matches the information-rich Chinese tutorial-handout direction, not the earlier 3D-diorama direction.
- exact source semantics remain in Markdown.
