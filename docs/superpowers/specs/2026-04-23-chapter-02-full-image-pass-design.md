# Chapter 02 Full Image Pass Design

## Goal

Turn `chapters/02_newton_arch` into a section-level visual teaching system: every `##` and `###` heading across all chapter-02 markdown files gets a visual anchor, while the chapter still preserves its current beginner-first reading spine and source-accuracy discipline.

## Scope

Files in scope:

- `chapters/02_newton_arch/README.md`
- `chapters/02_newton_arch/principle.md`
- `chapters/02_newton_arch/examples.md`
- `chapters/02_newton_arch/source-walkthrough.md`
- `chapters/02_newton_arch/source-walkthrough-deep.md`
- `chapters/02_newton_arch/question-notes.md`

Current heading-level scope is approximately 62 `##/###` sections.

## Core Design Principle

This is **not** a plan to generate 62 equally heavy posters.

Instead, every heading gets a visual anchor through one of three tiers:

1. **Fresh image**
   - A new section-specific visual generated for that heading.
2. **Canonical reuse**
   - A heading explicitly reuses a chapter-canonical image that already teaches that concept best.
3. **Lightweight structural visual**
   - A compact navigation / checklist / routing graphic used for low-diagram-value headings.

This is the only scalable way to satisfy “every `##/###` has a picture” without turning chapter 02 into decorative noise.

Two explicit rules close loopholes here:

- **Container headings still count.** Parent headings like `## Main Walkthrough`, `## Exact Handoff Trace`, or `## Optional Branches` must also receive a visual anchor, even if they mainly group child sections.
- **A lightweight visual is not a generic placeholder.** It must encode that section's own routing, grouping, checklist, or scope boundary; a repeated generic card does not count as satisfying the requirement.

## Visual System

### Shared Style

- White background
- Chinese teaching-poster language
- Rounded cards and low-clutter flow arrows
- Large title, sparse text, 3-6 core blocks per image
- Consistent palette with existing chapter-02 assets:
  - blue for main flow / runtime objects
  - green for geometry / structure / positive cues
  - purple for routing / comparison / optional branches
  - orange for solver / warning / downstream handoff
- Prefer landscape compositions for section-level diagrams

### Accuracy Classes

Every image must be labeled internally in the production inventory as one of:

- `source-exact`
  - Must match the pinned source semantics closely.
  - Default for `source-walkthrough.md` and most of `source-walkthrough-deep.md`.
- `teaching-compressed`
  - May compress or simplify relationships for explanation.
  - Must be paired with explicit markdown wording when exact source semantics differ.

### Image Types

Every heading is assigned exactly one primary image type:

- `navigation-map`
  - file responsibilities, chapter routing, reading order, branch maps
- `concept-poster`
  - one concept distilled into a small set of labeled blocks
- `runtime-bridge`
  - control handoff, object assembly, substep flow, exact contract boundaries
- `question-poster`
  - a confusion-driven image answering one likely reader mistake

## Fresh vs Reuse Policy

### Fresh Images Required

Fresh images should be concentrated where visual explanation adds the most instructional value:

- walkthrough stage handoffs
- object assembly and runtime contract diagrams
- example-chain comparison maps
- chapter navigation maps that do not already exist
- any section whose current prose describes a sequence, layered contract, or comparison that would be substantially clearer as a diagram

### Canonical Reuse Required

Existing strong images should be reused wherever they remain the best teaching artifact. Current chapter-canonical reuse targets include:

- `assets/02_model_state_control_solver.svg`
- `assets/02_solver_family_map.svg`
- `assets/02_launcher_handoff.svg`
- `assets/02_pendulum_joint_chain.svg`
- `assets/02_rot_view_comparison.png`
- `assets/02_cli_flow_summary.png`
- `assets/02_joint_steps.png`
- `assets/02_joint_frame_summary.png`
- `assets/02_joint_frame_equivalence.png`

Reuse is mandatory where generating a new image would merely restate the same concept with different cosmetics.

### Lightweight Structural Visuals

Low-value headings still need a visual anchor, but these visuals should stay compact and low-cost. Typical cases:

- `README.md` section navigation
- reading-order sections
- completion/checklist sections
- deep-index / verification-index sections
- “how this file fits with the others” sections

These should use small, reusable structural graphics rather than full posters.

Minimum bar for a lightweight structural visual:

- It must represent a section-specific structure, such as:
  - a chapter routing map
  - a grouped heading tree
  - a read/skip boundary
  - a checklist cluster
  - a branch map
- It may be visually simple, but it may not be content-empty or interchangeable across unrelated sections.

## Asset Organization

All final assets stay in:

- `chapters/02_newton_arch/assets/`

Use stable ASCII filenames with source-file prefix and ordered section identifiers:

- `02_readme_01_source-note.png`
- `02_readme_02_file-map.png`
- `02_principle_01_pendulum-step-bridge.png`
- `02_examples_04_cartpole-bridge.png`
- `02_walkthrough_03_stage-3-assembly.png`
- `02_deep_05_solver-contract.png`
- `02_qnotes_03_p-q-axis.png`

Do not create nested asset directories in this phase; keep the flat asset layout for simpler markdown references and lower migration risk.

Filename rule for reused assets:

- Existing canonical assets keep their current canonical filenames.
- New section-specific assets use the ordered section-identifier scheme above.
- Reuse does **not** require duplicating or aliasing an asset under a second section-specific filename unless the reused file is materially edited into a new variant.

## Placement Rules

- The image for a heading should appear near the top of that section, usually after the first framing paragraph or code/chain block.
- If a heading is mostly a container label with little or no body text, place its visual immediately after the heading and before the first child heading.
- Every inserted image must be followed by at most one short “how to read this image” paragraph.
- “Short” here means 1 paragraph and roughly 2-4 sentences, not another mini-section.
- Low-value structural sections should not get giant posters.
- Reused images may appear in multiple files if that is the most accurate and economical teaching choice.

## File-by-File Policy

### README.md

- Give each `##` a visual anchor, but most should be lightweight structural visuals.
- Do not turn Source Note, Core Pass, Defer For Now, or completion checklist sections into heavy posters.
- Highest-value fresh visuals here:
  - file map
  - prerequisite / chapter-neighbor map
  - GAMES103 vs chapter-02 delta map

### principle.md

- Prefer concept posters and reuse of chapter-canonical concept maps.
- Reuse existing four-layer and solver-family images.
- Use new visuals only where the prose introduces a bridge not already visualized in walkthrough files.

### examples.md

- Prefer question-posters and runtime-bridge diagrams.
- Reuse current pendulum and `rot` / `axis` images.
- Add fresh visuals mainly for `robot_cartpole` and `cloth_hanging` architecture chains.

### source-walkthrough.md

- This is the highest-value image file.
- Most stage sections should use `runtime-bridge` images.
- Fresh images are required for the currently under-visualized stages, especially:
  - Stage 3 object assembly
  - Stage 5 substep chain
- Reuse Stage 2 launcher image and the existing four-layer map where appropriate.

### source-walkthrough-deep.md

- Keep source-accuracy stricter here.
- Tables remain primary for lookup-heavy sections.
- Add visuals only where they clarify exact handoff structure, not where the table is already the best artifact.
- Many sections here can satisfy the “one image” requirement through small structural visuals or canonical reuse.

### question-notes.md

- This file already carries the strongest confusion-driven poster style.
- Existing learner-image-derived posters remain canonical anchors.
- The remaining heading without a strong image should get a lightweight navigation visual linking the page back to the main chapter files.

## Review Safeguards

To prevent quality collapse over a 60+ section pass:

1. Review by batch, not only at the end.
2. Keep a per-section inventory recording:
   - heading
   - file path
   - heading level (`##` or `###`)
   - image type
   - accuracy class
   - fresh vs reuse vs lightweight
   - asset path
   - source anchor(s) if `source-exact`
   - final asset filename
3. For any `teaching-compressed` image in walkthrough-related files, add an explicit markdown note if the image is not source-exact.
4. Re-review generated text inside images for readability and language drift before integrating into markdown.
5. Run a final coverage audit: heading inventory count must equal the count of visual anchors assigned across the six files.
6. Run a final source audit for `source-exact` images: every one must have a recorded source anchor in the inventory.

## Execution Strategy

Execute in 6 file-based batches:

1. `README.md`
2. `principle.md`
3. `examples.md`
4. `source-walkthrough.md`
5. `source-walkthrough-deep.md`
6. `question-notes.md`

Each batch must complete this loop:

1. Section inventory and image-type assignment
2. First-pass generation
3. Multi-agent review using this rubric:
   - section-specific value
   - visual-system consistency
   - source accuracy or correction-note sufficiency
   - text readability inside image
4. Corrections / regeneration
5. Markdown integration
6. Diff hygiene verification (`git diff --check`, asset existence, markdown path correctness)

Recommended checkpoint cadence:

- After every batch, stop for a controller synthesis and only then move to the next file.
- Do not queue all 62 headings into a single generation wave.

Fallback rule for hard sections:

- If a fresh image fails repeated review for a `source-exact` section, downgrade that section to a section-specific lightweight visual or canonical reuse only if the visual anchor still carries that section's own structure and the inventory records the downgrade explicitly.

## Out of Scope

- Refactoring chapter-02 prose structure unrelated to image integration
- Expanding chapter 02 into a full articulation or solver chapter
- Reorganizing all existing assets into subdirectories in this same pass
- Generating photorealistic illustrations or decorative art without teaching value
