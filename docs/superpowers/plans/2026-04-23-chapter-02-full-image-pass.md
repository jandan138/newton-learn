# Chapter 02 Full Image Pass Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give every `##` and `###` section in chapter 02 a visual anchor while preserving the chapter's beginner-first reading spine and source-accuracy rules.

**Architecture:** Execute the work in six file-based batches. Each batch first records section-level image decisions in a shared inventory, then generates or reuses assets, integrates them into markdown, runs multi-agent review, and fixes problems before moving to the next file.

**Tech Stack:** Markdown, PNG/SVG assets, Codex built-in image generation, multi-agent review, git diff verification

---

## File Map

- Create: `chapters/02_newton_arch/image-inventory.md`
- Create/Modify: `chapters/02_newton_arch/assets/02_*.png`
- Create/Modify: `chapters/02_newton_arch/assets/02_*.svg`
- Modify: `chapters/02_newton_arch/README.md`
- Modify: `chapters/02_newton_arch/principle.md`
- Modify: `chapters/02_newton_arch/examples.md`
- Modify: `chapters/02_newton_arch/source-walkthrough.md`
- Modify: `chapters/02_newton_arch/source-walkthrough-deep.md`
- Modify: `chapters/02_newton_arch/question-notes.md`
- Create: `docs/superpowers/specs/2026-04-23-chapter-02-full-image-pass-design.md`
- Create: `docs/superpowers/plans/2026-04-23-chapter-02-full-image-pass.md`

### Task 1: Create the section inventory

**Files:**
- Create: `chapters/02_newton_arch/image-inventory.md`

- [ ] **Step 1: Create the inventory table skeleton**

Create `chapters/02_newton_arch/image-inventory.md` with one row per `##` / `###` heading across these files:

```text
README.md
principle.md
examples.md
source-walkthrough.md
source-walkthrough-deep.md
question-notes.md
```

Use this column schema exactly:

```text
| file | heading | level | image_type | accuracy | mode | asset | source_anchors | status |
```

Where:

```text
image_type   = navigation-map / concept-poster / runtime-bridge / question-poster
accuracy     = source-exact / teaching-compressed
mode         = fresh / reuse / lightweight
status       = planned / generated / integrated / reviewed
```

- [ ] **Step 2: Fill in all current headings from the six in-scope files**

Run a heading scan and populate the inventory. Expected outcome: every `##` and `###` in chapter 02 appears exactly once.

- [ ] **Step 3: Record the existing canonical assets for reuse rows**

Mark current reusable assets in the inventory where applicable:

```text
assets/02_model_state_control_solver.svg
assets/02_solver_family_map.svg
assets/02_launcher_handoff.svg
assets/02_pendulum_joint_chain.svg
assets/02_rot_view_comparison.png
assets/02_cli_flow_summary.png
assets/02_joint_steps.png
assets/02_joint_frame_summary.png
assets/02_joint_frame_equivalence.png
```

### Task 2: Batch A — `README.md`

**Files:**
- Modify: `chapters/02_newton_arch/README.md`
- Modify: `chapters/02_newton_arch/image-inventory.md`
- Create: `chapters/02_newton_arch/assets/02_readme_*.png`

- [ ] **Step 1: Assign image decisions for every README heading**

For each `##` in `README.md`, decide whether it uses:

```text
fresh
reuse
lightweight
```

and record the exact asset path in the inventory.

- [ ] **Step 2: Generate the missing README visuals**

Generate only the README visuals that are not already satisfied by reuse. Keep them lightweight and structural. Prioritize:

```text
file map
chapter-neighbor map
GAMES103 delta map
```

- [ ] **Step 3: Integrate one visual anchor into every README `##` section**

Place the visual near the top of each section. For low-value sections, compact structural visuals are sufficient.

- [ ] **Step 4: Batch-review README visuals**

Run multi-agent review focused on:

```text
section coverage
whether lightweight visuals still carry section-specific value
whether README stayed navigational rather than poster-heavy
```

- [ ] **Step 5: Verify README batch hygiene**

Run:

```bash
git diff --check -- chapters/02_newton_arch/README.md chapters/02_newton_arch/image-inventory.md
```

Expected: no output.

### Task 3: Batch B — `principle.md`

**Files:**
- Modify: `chapters/02_newton_arch/principle.md`
- Modify: `chapters/02_newton_arch/image-inventory.md`
- Create: `chapters/02_newton_arch/assets/02_principle_*.png`

- [ ] **Step 1: Assign every `principle.md` heading an image decision**

Use canonical reuse wherever the current image is already the right teaching artifact.

- [ ] **Step 2: Reuse canonical assets for the already-solved concept sections**

Keep these as reuse rows where they remain best-in-class:

```text
02_model_state_control_solver.svg
02_solver_family_map.svg
```

- [ ] **Step 3: Generate fresh concept posters only for missing principle bridges**

Main fresh-image targets:

```text
00/01 -> 02 bridge
basic_pendulum -> one step bridge if not already satisfied by walkthrough reuse
chapter interfaces / downstream map if a fresh visual is still needed after reuse
```

- [ ] **Step 4: Integrate visuals into every `principle.md` `##` section**

Each section must end the batch with one visual anchor, whether fresh, reused, or lightweight.

- [ ] **Step 5: Review and verify the principle batch**

Run multi-agent review and then:

```bash
git diff --check -- chapters/02_newton_arch/principle.md chapters/02_newton_arch/image-inventory.md
```

Expected: no output.

### Task 4: Batch C — `examples.md`

**Files:**
- Modify: `chapters/02_newton_arch/examples.md`
- Modify: `chapters/02_newton_arch/image-inventory.md`
- Create: `chapters/02_newton_arch/assets/02_examples_*.png`

- [ ] **Step 1: Assign image decisions for every `##` and `###` in `examples.md`**

Use existing pendulum and `rot` / `axis` images where appropriate.

- [ ] **Step 2: Generate the missing example-bridge visuals**

Fresh-image priorities:

```text
robot_cartpole coverage chain
cloth_hanging solver-switch chain
any parent container heading that still lacks a section-specific visual anchor
```

- [ ] **Step 3: Integrate visuals into all example sections**

Ensure even framing sections and parent example headers receive either a compact structural visual or a clearly assigned reused anchor.

- [ ] **Step 4: Review and verify the examples batch**

Run multi-agent review and then:

```bash
git diff --check -- chapters/02_newton_arch/examples.md chapters/02_newton_arch/image-inventory.md
```

Expected: no output.

### Task 5: Batch D — `source-walkthrough.md`

**Files:**
- Modify: `chapters/02_newton_arch/source-walkthrough.md`
- Modify: `chapters/02_newton_arch/image-inventory.md`
- Create: `chapters/02_newton_arch/assets/02_walkthrough_*.png`

- [ ] **Step 1: Assign image decisions for every walkthrough heading**

Treat this as the highest-value file for fresh visuals.

- [ ] **Step 2: Reuse the existing launcher image and four-layer image where they are still the most accurate fit**

Do not regenerate them unless review shows they are no longer sufficient.

- [ ] **Step 3: Generate fresh visuals for the under-illustrated stages**

Required fresh targets:

```text
One-Screen Chapter Map if current ASCII map is not enough to satisfy the visual-anchor requirement
Stage 3 object assembly
Stage 5 substep chain
any container section that still lacks a section-specific visual anchor
```

- [ ] **Step 4: Integrate all walkthrough visuals with strict accuracy notes**

For any `teaching-compressed` image in this file, add a short markdown note when exact source semantics differ.

- [ ] **Step 5: Review and verify the walkthrough batch**

Run multi-agent review focused on source accuracy, then:

```bash
git diff --check -- chapters/02_newton_arch/source-walkthrough.md chapters/02_newton_arch/image-inventory.md
```

Expected: no output.

### Task 6: Batch E — `source-walkthrough-deep.md`

**Files:**
- Modify: `chapters/02_newton_arch/source-walkthrough-deep.md`
- Modify: `chapters/02_newton_arch/image-inventory.md`
- Create: `chapters/02_newton_arch/assets/02_deep_*.png`

- [ ] **Step 1: Assign image decisions for every deep heading**

Prefer compact structural visuals or canonical reuse for lookup-heavy sections.

- [ ] **Step 2: Generate only the deep visuals that add exact-handoff value**

Fresh-image priorities:

```text
runtime stack exact handoff
solver contract exact entrance
optional branch tree if needed to satisfy container headings
```

- [ ] **Step 3: Keep deep tables primary**

Do not replace tables that are already the most precise artifact; satisfy the image requirement through small structural visuals where necessary.

- [ ] **Step 4: Review and verify the deep batch**

Run multi-agent review focused on source-exact claims, then:

```bash
git diff --check -- chapters/02_newton_arch/source-walkthrough-deep.md chapters/02_newton_arch/image-inventory.md
```

Expected: no output.

### Task 7: Batch F — `question-notes.md`

**Files:**
- Modify: `chapters/02_newton_arch/question-notes.md`
- Modify: `chapters/02_newton_arch/image-inventory.md`
- Create: `chapters/02_newton_arch/assets/02_qnotes_*.png`

- [ ] **Step 1: Assign image decisions for every question-notes heading**

Reuse the current four learner-derived posters where they are already canonical.

- [ ] **Step 2: Create the remaining navigation visual for the final section**

Main fresh-image target:

```text
`## 5. 这页怎么和主线配合`
```

This should be a lightweight visual linking `question-notes.md` back to the main chapter files.

- [ ] **Step 3: Integrate and review the question-notes batch**

Run multi-agent review focused on whether the supplement remains clearly secondary to the core pass, then:

```bash
git diff --check -- chapters/02_newton_arch/question-notes.md chapters/02_newton_arch/image-inventory.md
```

Expected: no output.

### Task 8: Final chapter-wide audit

**Files:**
- Modify: `chapters/02_newton_arch/image-inventory.md`
- Modify: all six chapter-02 markdown files

- [ ] **Step 1: Run the heading coverage audit**

Verify that the total heading count equals the count of assigned visual anchors in `image-inventory.md`.

- [ ] **Step 2: Run the source-exact audit**

Verify that every `source-exact` row in `image-inventory.md` has at least one source anchor recorded.

- [ ] **Step 3: Run final diff hygiene checks**

Run:

```bash
git diff --check -- chapters/02_newton_arch/README.md chapters/02_newton_arch/principle.md chapters/02_newton_arch/examples.md chapters/02_newton_arch/source-walkthrough.md chapters/02_newton_arch/source-walkthrough-deep.md chapters/02_newton_arch/question-notes.md chapters/02_newton_arch/image-inventory.md docs/superpowers/specs/2026-04-23-chapter-02-full-image-pass-design.md docs/superpowers/plans/2026-04-23-chapter-02-full-image-pass.md
```

Expected: no output.

- [ ] **Step 4: Review the final image inventory and chapter diff**

Run:

```bash
git diff -- chapters/02_newton_arch docs/superpowers/specs/2026-04-23-chapter-02-full-image-pass-design.md docs/superpowers/plans/2026-04-23-chapter-02-full-image-pass.md
```

Expected: the diff shows a coherent chapter-wide visual system rather than unrelated one-off posters.
