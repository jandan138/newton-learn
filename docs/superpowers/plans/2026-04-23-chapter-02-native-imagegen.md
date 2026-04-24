# Chapter 02 Native Imagegen Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace chapter 02's SVG-heavy teaching anchors with coherent native-imagegen raster images, starting with the highest-value runtime and example sections while keeping exact source details in Markdown.

**Architecture:** Work in batches. First freeze a chapter-wide prompt kit and replace the core runtime spine in `source-walkthrough.md`, then expand to `principle.md` and `examples.md`, and only then migrate the navigation-heavy support pages. Each batch uses the same loop: generate, visually inspect, integrate, review, and regenerate if needed.

**Tech Stack:** Markdown, PNG assets, Codex built-in image generation, multi-agent review, git diff verification

---

## File Map

- Modify: `chapters/02_newton_arch/README.md`
- Modify: `chapters/02_newton_arch/principle.md`
- Modify: `chapters/02_newton_arch/examples.md`
- Modify: `chapters/02_newton_arch/source-walkthrough.md`
- Modify: `chapters/02_newton_arch/source-walkthrough-deep.md`
- Modify: `chapters/02_newton_arch/question-notes.md`
- Create/Modify: `chapters/02_newton_arch/assets/02_*.png`
- Create: `docs/superpowers/specs/2026-04-23-chapter-02-native-imagegen-design.md`
- Create: `docs/superpowers/plans/2026-04-23-chapter-02-native-imagegen.md`

### Task 1: Freeze the prompt kit and first-wave target list

**Files:**
- Create: `docs/superpowers/specs/2026-04-23-chapter-02-native-imagegen-design.md`
- Create: `docs/superpowers/plans/2026-04-23-chapter-02-native-imagegen.md`

- [x] **Step 1: Verify native image generation is available**

Run:

```bash
codex features list
```

Expected: `image_generation stable true`

- [x] **Step 2: Lock the first-wave sections**

Use this exact first-wave list:

```text
source-walkthrough.md
- One-Screen Chapter Map
- Stage 3
- Stage 4
- Stage 5
- Object Ledger

principle.md
- 从 basic_pendulum 走到一次 step
- 四层关系图
- 8 个 solver 的全景图

examples.md
- 主例子：basic_pendulum
- basic_pendulum / 它覆盖的架构链
- basic_pendulum / 改这里会怎样
- robot_cartpole / 它覆盖的架构链
- cloth_hanging / 它覆盖的架构链
```

- [x] **Step 3: Keep the text boundary explicit**

Before generating each image, remove any requirement for:

```text
full file paths
full call chains
line numbers
exact symbolic values
long paragraphs inside the image
```

Put those details in Markdown instead.

### Task 2: Generate the core runtime spine batch

**Files:**
- Create: `chapters/02_newton_arch/assets/02_walkthrough_*.png`
- Modify: `chapters/02_newton_arch/source-walkthrough.md`

- [x] **Step 1: Generate the One-Screen Chapter Map image**

Use `codex exec --enable image_generation` through the native-imagegen workflow to create a chapter-wide runtime relay scene for the one-screen map section.

Expected output file pattern:

```text
chapters/02_newton_arch/assets/02_walkthrough_one_screen_map.png
```

- [x] **Step 2: Generate Stage 3, Stage 4, Stage 5, and Object Ledger images**

Create four additional raster images for:

```text
02_walkthrough_stage3_runtime_stack_bridge.png
02_walkthrough_stage4_object_roles_poster.png
02_walkthrough_stage5_simulate_loop_bridge.png
02_walkthrough_object_ledger_poster.png
```

Each prompt should keep labels short and emphasize scene structure over text.

- [x] **Step 3: Inspect the generated files**

Run:

```bash
file chapters/02_newton_arch/assets/02_walkthrough_*.png
ls -lh chapters/02_newton_arch/assets/02_walkthrough_*.png
```

Expected: PNG files exist and have non-trivial sizes. Prefer individual assets to stay around `1.2 MB` or below when practical; if a scene is much heavier, simplify or compress before scaling the same prompt family across the chapter.

- [x] **Step 4: Integrate the new PNG paths into `source-walkthrough.md`**

Replace the existing section-image references for the first-wave runtime sections so Markdown points to the new `.png` files instead of the old `.svg` files.

- [x] **Step 5: Review the runtime spine batch**

Run multi-agent review focused on:

```text
teaching clarity
style consistency
whether any image falsely implies exact code semantics
whether Markdown still carries the exact truth
```

### Task 3: Generate the principle batch

**Files:**
- Create: `chapters/02_newton_arch/assets/02_principle_*.png`
- Modify: `chapters/02_newton_arch/principle.md`

- [x] **Step 1: Generate the `basic_pendulum -> one step` bridge image**

Target file:

```text
chapters/02_newton_arch/assets/02_principle_basic_pendulum_step_bridge.png
```

- [x] **Step 2: Generate the four-layer relation scene**

Target file:

```text
chapters/02_newton_arch/assets/02_model_state_control_solver.png
```

- [x] **Step 3: Generate the solver panorama scene**

Target file:

```text
chapters/02_newton_arch/assets/02_solver_family_map.png
```

- [x] **Step 4: Replace the matching `.svg` references in `principle.md`**

Update the three first-wave principle sections to use the new `.png` files.

- [x] **Step 5: Verify the principle batch**

Run:

```bash
git diff --check -- chapters/02_newton_arch/principle.md chapters/02_newton_arch/source-walkthrough.md
```

Expected: no output.

### Task 4: Generate the examples batch

**Files:**
- Create: `chapters/02_newton_arch/assets/02_examples_*.png`
- Modify: `chapters/02_newton_arch/examples.md`

- [x] **Step 1: Generate the `basic_pendulum` scope, chain, and knobs images**

Target files:

```text
chapters/02_newton_arch/assets/02_examples_basic_pendulum_scope_bridge.png
chapters/02_newton_arch/assets/02_examples_basic_pendulum_chain_bridge.png
chapters/02_newton_arch/assets/02_examples_basic_pendulum_knobs_map.png
```

- [x] **Step 2: Generate the `robot_cartpole` and `cloth_hanging` chain images**

Target files:

```text
chapters/02_newton_arch/assets/02_examples_robot_cartpole_chain_bridge.png
chapters/02_newton_arch/assets/02_examples_cloth_hanging_chain_bridge.png
```

- [x] **Step 3: Replace the matching `.svg` references in `examples.md`**

Only replace the sections with completed raster assets in this batch.

- [x] **Step 4: Review the examples batch**

Run multi-agent review focused on whether the example images clearly separate:

```text
scene setup
runtime handoff
which knobs affect model vs runtime vs solver choice
```

### Task 5: Expand to navigation and deep support pages

**Files:**
- Modify: `chapters/02_newton_arch/README.md`
- Modify: `chapters/02_newton_arch/source-walkthrough-deep.md`
- Modify: `chapters/02_newton_arch/question-notes.md`

- [x] **Step 1: Replace only the navigation-heavy sections that still feel flat after the core batches**

Prefer small scene-based navigation images rather than dense poster remakes.

- [x] **Step 2: Keep deep exactness markdown-led**

Do not move exact source semantics into image text for `source-walkthrough-deep.md`.

- [x] **Step 3: Reuse generated raster assets where reuse is honest**

Do not create cosmetic duplicates for nearby sections that can share the same scene.

- [x] **Step 4: Remove obsolete SVG counterparts at the end of the full migration**

After chapter-wide markdown references no longer point at a replaced walkthrough/principle/examples SVG, delete the obsolete counterpart in the final cleanup sweep instead of leaving permanent duplicate assets behind.

### Task 6: Final verification

**Files:**
- Modify: `chapters/02_newton_arch/*.md`
- Create/Modify: `chapters/02_newton_arch/assets/*.png`

- [x] **Step 1: Verify image paths and markdown hygiene**

Run:

```bash
git diff --check -- chapters/02_newton_arch
```

Expected: no output.

- [x] **Step 2: Verify generated files exist**

Run:

```bash
file chapters/02_newton_arch/assets/*.png
```

Expected: all replaced assets report as PNG image data.

- [x] **Step 3: Run a final visual review**

Review the rendered markdown pages and confirm:

```text
the chapter now feels image-generated rather than SVG-driven
the images are visually coherent
the images are teaching aids rather than decorative noise
the exact technical truth still lives in Markdown
```
