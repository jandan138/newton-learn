# Chapter Visual Style Guide

## Purpose

Use this guide for future Newton Learn chapter image generation. Chapter 02 does not need to be revised, but later chapters should not repeat the same 3D-diorama PNG direction unless explicitly requested.

The preferred target is a dense, clear Chinese teaching infographic: more like a teacher's one-page review sheet than a decorative illustration.

## Reference Style

Primary reference: the user-provided `basic_pendulum / Chapter 02 六个核心问题` infographic shared on 2026-04-24.

If the reference image is later saved into the repo, put it under:

`docs/superpowers/assets/reference/basic-pendulum-ch02-infographic-style.png`

Future specs and plans should cite this style guide instead of relying on chat history.

## Visual Language

- white background with thin blue card borders
- strong dark-blue chapter title and section headings
- numbered circular badges for question blocks or teaching steps
- multiple compact cards in one image, each focused on one learner question
- Chinese-first explanatory text, with English only for exact code names, filenames, API names, or short labels
- small icons, arrows, boxes, callouts, and flow strips to make relationships explicit
- high information density, but with clear grouping and whitespace
- friendly classroom / study-sheet feeling, not cinematic concept art

## Content Pattern

Prefer images that answer a small set of concrete learner questions.

Good patterns:

- "six core questions" overview sheet
- one-page execution-chain summary
- object-role comparison table with icons
- misconception card: wrong intuition -> corrected intuition
- compact source-to-runtime handoff map
- bottom recap strip with the one-sentence memory hook

Avoid:

- standalone decorative scenes
- vague 3D objects with unclear teaching purpose
- long paragraphs embedded in the image
- fake code blocks or fake terminal text
- tiny labels that become unreadable in Markdown
- visual claims that are more precise than the surrounding source-backed text

## Source-Truth Boundary

Markdown remains authoritative. Images are learning aids.

Images may include short exact terms such as `Model`, `State`, `Control`, `simulate()`, `solver.step()`, or filenames when useful, but exact source semantics should stay in prose, tables, and code blocks.

When an image compresses a call chain or object relationship, the surrounding Markdown must say so. Do not let a generated infographic silently become the source of truth.

## Prompt Template

Use this as the starting point for future native-imagegen prompts:

```text
Create a clean Chinese learning infographic for Newton Learn, in the style of a one-page teacher review sheet.

Visual style: white background, thin blue rounded card borders, dark-blue title typography, numbered blue circular badges, compact multi-panel layout, small friendly icons, arrows, flow boxes, table-like comparison cards, clear whitespace, high information density but readable.

Language: Chinese-first. Use English only for exact code/API names such as `Model`, `State`, `simulate()`, `solver.step()`. Keep labels short and readable.

Content goal: answer these learner questions: <questions>. Show the relationships visually with cards, arrows, and a bottom recap strip. Do not invent code. Do not include long paragraphs. Do not use SVG, HTML, screenshot, terminal UI, or photorealistic 3D rendering. Generate a raster PNG.
```

## Acceptance Criteria

A generated image is acceptable when:

- it looks like a structured Chinese learning handout, not an abstract illustration
- a beginner can identify the main question and answer within a few seconds
- section/card boundaries are clear at normal Markdown reading size
- text is mostly readable without zooming
- code/API names that appear are real and intentionally chosen
- the image does not contradict source-backed Markdown
- compressed teaching shortcuts are fenced by nearby prose

## Chapter Workflow Rule

For every future chapter image-generation spec or plan:

- cite this guide in the visual-language section
- define the chapter's core learner questions before generating images
- generate the first overview infographic as a style calibration image
- review that image before expanding to the rest of the chapter
- keep exact source details in Markdown, not inside generated image text
