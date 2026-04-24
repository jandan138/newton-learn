# Chapter 02 Native Imagegen Redesign

## Goal

Replace chapter 02's SVG-heavy section visuals with a coherent set of raster teaching images generated through the `codex-native-imagegen` workflow, so the chapter feels vivid, beginner-friendly, and visually memorable without moving source-truth out of Markdown.

## Why This Redesign Exists

The current chapter-02 image system is structurally complete, but most of it is expressed as flat SVG maps and posters. That preserves precision, but it does not satisfy the user's request for model-generated images, and it leaves too much of the chapter feeling like diagram inventory instead of visual teaching.

The redesign target is not "replace diagrams with decorative art." The target is a better teaching surface:

- images should make the architecture feel physical and intuitive
- images should help a beginner remember runtime handoffs and object roles
- images should remain low-risk by keeping exact code details in prose, tables, and code blocks

## Scope

In scope:

- `chapters/02_newton_arch/README.md`
- `chapters/02_newton_arch/principle.md`
- `chapters/02_newton_arch/examples.md`
- `chapters/02_newton_arch/source-walkthrough.md`
- `chapters/02_newton_arch/source-walkthrough-deep.md`
- `chapters/02_newton_arch/question-notes.md`
- `chapters/02_newton_arch/assets/`

The current section inventory still applies: every reader-facing `##` and `###` in those files needs a visual anchor.

## Core Decision

Use a family-based native-imagegen system instead of trying to create 62 unrelated posters.

That means:

1. One chapter-wide visual language.
2. A small prompt kit reused across batches.
3. Fresh images concentrated in the sections where visual intuition matters most.
4. Reuse of generated raster assets where a section only needs the same teaching scene from a nearby angle.
5. Markdown remains the carrier of exact source semantics.

## Visual Language

### Shared Style

- white or very light neutral background
- clean 3D teaching-diorama / tabletop lab style
- soft isometric or 3/4 view, not flat infographic view
- vivid but controlled color palette
- minimal in-image text
- large, readable shapes and arrows
- obvious physical metaphors: relay stations, knobs, trays, rails, teaching blocks

### Semantic Mapping

Keep the existing chapter semantics, but express them visually instead of as flat cards.

Preferred role-color targets for the prompt kit:

- `Model`: green or green-blue structural block
- `State`: blue snapshot capsule / tile
- `Control`: orange or warm input panel
- `Contacts`: purple collision tray / result bin
- `Solver`: neutral gray machine with warm core accents

These are prompt targets, not a brittle hard rule. For native image generation, stable object metaphors and short labels matter more than hue-perfect lockstep. Across a batch, the goal is recognizable role continuity, not pixel-identical palette reuse.

### Image Families

Every section image should fall into one of these families:

1. Runtime relay scene
   - best for `runtime-bridge` sections
   - shows a left-to-right handoff across 4-6 stations
2. Teaching tabletop scene
   - best for `concept-poster` sections
   - shows a central object with a few role callouts
3. Path / room / signpost scene
   - best for `navigation-map` sections
   - feels like a guided museum floorplan rather than a flowchart
4. Misconception split scene
   - best for `question-poster` sections
   - left side wrong intuition, right side corrected intuition

## Exactness Discipline

This redesign is only acceptable if the source-truth boundary stays strict.

### Images Should Not Carry

- long function names or full call chains
- line numbers
- file paths
- dense Chinese paragraphs
- exact symbolic values like `-wp.pi * 0.5`
- proof-heavy distinctions that depend on exact tokens

### Markdown Must Carry

- source anchors
- exact API names
- exact call order when the section claims source fidelity
- correction notes for `teaching-compressed` images
- distinctions such as `transform.q` vs `joint_q`, `rot` vs `axis`, and constructor-time vs runtime-time operations

### Accuracy Classes Still Apply

- `source-exact`: the surrounding Markdown makes exact claims and the image must not contradict them
- `teaching-compressed`: the image is a teaching aid and may compress detail, but the Markdown must say so when needed

For `source-walkthrough-deep.md`, tables and prose remain authoritative even after new raster images are introduced.

## Prompt Discipline

Every generation prompt should include these constraints:

- use built-in image generation only
- generate raster output only
- no SVG or HTML fallback
- minimal text, at most a few short labels
- educational, not decorative
- no fake code text
- no screenshots of UIs or terminals
- clear spatial separation between the major objects in the scene

Prompt templates should be standardized by image family so the chapter style does not drift from section to section. Each batch should freeze one small prompt kit before expansion.

## Batch Strategy

The rollout should not start with all sections equally. It should start with the most instructional pages, freeze the style, then expand.

### Batch 1: Core Runtime Spine

- `source-walkthrough.md`
- highest-value sections: one-screen map, stage 3, stage 4, stage 5, object ledger

### Batch 2: Principle Bridges

- `principle.md`
- highest-value sections: `basic_pendulum -> one step`, four-layer relation scene, solver panorama, downstream chapter interface

### Batch 3: Example Scenes

- `examples.md`
- highest-value sections: `basic_pendulum`, `robot_cartpole`, `cloth_hanging`, knobs and misconception visuals

### Batch 4: Navigation and Deep Support

- `README.md`
- `source-walkthrough-deep.md`
- `question-notes.md`

The intention is not to stop after batch 1. The intention is to use batch 1 to calibrate style and guardrails before wider replacement.

## Asset Rules

- final assets stay in `chapters/02_newton_arch/assets/`
- new raster assets use stable ASCII filenames and `.png`
- current SVG assets may remain temporarily during migration, but the target state is that reader-facing chapter-02 anchors point to native-imagegen raster images wherever replacement has been completed
- if a section reuses a generated raster image, reuse the same file path instead of creating cosmetic duplicates
- once a section has fully migrated and no markdown references its old SVG, the legacy SVG becomes cleanup debt for the final sweep rather than a permanent parallel asset
- prefer generated PNG assets to stay around `1.2 MB` or below when practical; if repeated scenes are much larger, simplify prompts or compress before scaling the rollout

## Review Loop

Each batch must go through:

1. prompt writing
2. native-image generation
3. visual review
4. markdown integration
5. multi-agent critique
6. regeneration or prompt tightening if needed
7. path and rendering verification

## Acceptance Criteria

The redesign is successful when:

- chapter 02 visibly reads as a coherent teaching-image system rather than an SVG system with a few PNG exceptions
- the new images feel vivid and beginner-friendly
- Markdown still carries exact source truth
- no integrated image contains visibly broken technical text that a learner could mistake for real code or notation
- the highest-value runtime and example sections have been replaced first and reviewed before expanding outward

## Initial Recommended Replacement Order

1. `source-walkthrough.md` -> `## One-Screen Chapter Map`
2. `source-walkthrough.md` -> `### Stage 3`
3. `source-walkthrough.md` -> `### Stage 4`
4. `source-walkthrough.md` -> `### Stage 5`
5. `source-walkthrough.md` -> `## Object Ledger`
6. `principle.md` -> `## 1. 从 basic_pendulum 走到一次 step`
7. `principle.md` -> `## 2. 四层关系图`
8. `principle.md` -> `## 3. 8 个 solver 的全景图`
9. `examples.md` -> `## 主例子：basic_pendulum` and its key `###` sections
10. `examples.md` -> the two comparison-example chain sections
