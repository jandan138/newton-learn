# Chapter 02 Image Integration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Integrate all learner-generated chapter-02 images into the existing tutorial and add the missing chapter-02 supplement needed to hold the image-driven explanations.

**Architecture:** Keep the current chapter-02 teaching spine intact and distribute the images by teaching depth. Put source-accurate visuals in the main walkthrough files, keep `rot` / `axis` guidance in `examples.md`, and add a new `question-notes.md` supplement for learner summary diagrams that resolve real confusions but compress source layers.

**Tech Stack:** Markdown, repository image assets, git diff verification

---

## File Map

- Create: `chapters/02_newton_arch/assets/02_launcher_handoff.svg`
- Create: `chapters/02_newton_arch/assets/02_pendulum_joint_chain.svg`
- Create: `chapters/02_newton_arch/assets/02_rot_view_comparison.png`
- Create: `chapters/02_newton_arch/assets/02_cli_flow_summary.png`
- Create: `chapters/02_newton_arch/assets/02_joint_steps.png`
- Create: `chapters/02_newton_arch/assets/02_joint_frame_summary.png`
- Create: `chapters/02_newton_arch/assets/02_joint_frame_equivalence.png`
- Create: `chapters/02_newton_arch/question-notes.md`
- Modify: `chapters/02_newton_arch/README.md`
- Modify: `chapters/02_newton_arch/source-walkthrough.md`
- Modify: `chapters/02_newton_arch/examples.md`
- Modify: `chapters/02_newton_arch/source-walkthrough-deep.md`
- Create: `docs/superpowers/specs/2026-04-22-chapter-02-image-integration-design.md`
- Create: `docs/superpowers/plans/2026-04-22-chapter-02-image-integration.md`

### Task 1: Copy and stabilize image assets

**Files:**
- Create: `chapters/02_newton_arch/assets/02_launcher_handoff.svg`
- Create: `chapters/02_newton_arch/assets/02_pendulum_joint_chain.svg`
- Create: `chapters/02_newton_arch/assets/02_rot_view_comparison.png`
- Create: `chapters/02_newton_arch/assets/02_cli_flow_summary.png`
- Create: `chapters/02_newton_arch/assets/02_joint_steps.png`
- Create: `chapters/02_newton_arch/assets/02_joint_frame_summary.png`
- Create: `chapters/02_newton_arch/assets/02_joint_frame_equivalence.png`

- [ ] **Step 1: Verify source and destination directories exist**

Run: `ls /home/zhuzihou/dev/newton-learn/tmp/incoming-images && ls /home/zhuzihou/dev/newton-learn/chapters/02_newton_arch/assets`
Expected: the five incoming PNG files are present and the chapter assets directory exists.

- [ ] **Step 2: Keep the source-accurate incoming image and convert the rest into corrected repo-owned assets**

Run:

```bash
cp "/home/zhuzihou/dev/newton-learn/tmp/incoming-images/ChatGPT Image 2026年4月22日 20_42_46.png" "/home/zhuzihou/dev/newton-learn/chapters/02_newton_arch/assets/02_rot_view_comparison.png"
```

Then create corrected `02_launcher_handoff.svg` and `02_pendulum_joint_chain.svg` inside `chapters/02_newton_arch/assets/`, and also copy the remaining learner PNGs into stable ASCII filenames for the supplement page.

Expected: the accurate view-comparison PNG is copied, the two source-accurate SVG assets are created locally, and the remaining learner PNGs are available for the supplement page.

### Task 2: Update the main walkthrough

**Files:**
- Modify: `chapters/02_newton_arch/source-walkthrough.md`

- [ ] **Step 1: Add the launcher handoff image and clarify the two `__main__` moments**

Insert the new image after the Stage 2 `runpy.run_module(...)` code block and add a short explanation that:

```text
第一次 __main__：examples/__main__.py 作为 CLI 入口被执行。
第二次 __main__：runpy.run_module(..., run_name="__main__") 把目标 example module 当作新的主程序执行。
```

Expected result: Stage 2 becomes easier to follow for readers who are confused by module routing.

### Task 3: Update the example observation sheet

**Files:**
- Modify: `chapters/02_newton_arch/examples.md`

- [ ] **Step 1: Add the two-joint pendulum intuition SVG near the constructor-reading guidance**

Place `02_pendulum_joint_chain.svg` after the `Example.__init__()` bullet and add a short note that:

```text
j0 把世界锚点接到 link_0；
j1 再把 link_1 接到 link_0 的末端；
这一章先把它读成“二级关节链”，先别展开关节帧公式。
```

- [ ] **Step 2: Expand the `axis` row context and add the `rot` vs `axis` comparison image**

Add `02_rot_view_comparison.png` below the `改这里会怎样` table and add a warning block covering:

```text
- 改 `axis`：改的是关节允许转动的真实方向，所以摆动平面会变。
- 改 `rot`：改的是构造时 link / joint 的初始朝向表达；它会改变你看到的朝向，但不该被误读成“只是改了摄像机”。
- 第一遍最实用的判断标准：你改完以后，系统真实绕哪根轴摆、在哪个平面里摆，是否变了？
```

- [ ] **Step 3: Add a notation warning for `transform.q` vs `joint_q`**

Add a short note near the `basic_pendulum` observation section:

```text
这里会同时出现两种 `q`：
- `transform(..., q=...)` 里的 `q` 是姿态四元数；
- `joint_q` 里的 `q` 是关节广义坐标。
名字一样，但不是同一种量。
```

### Task 4: Update the deep walkthrough

**Files:**
- Modify: `chapters/02_newton_arch/source-walkthrough-deep.md`

- [ ] **Step 1: Add an optional joint-frame note using the single-joint figure**

Insert a short text-only optional note under the runtime-stack section with an explicit label such as `Optional Deep Note: How to read one revolute joint`.

- [ ] **Step 2: Add the equivalence warning as an advanced note, not a completion requirement**

Add a short explanation that equivalent `parent_xform` / `child_xform` / `axis` parameterizations can describe the same physical joint, but that this is second-pass material.

Expected result: chapter 02 gains a place to store advanced learner notes without polluting the core pass.

### Task 5: Add the question-driven supplement page

**Files:**
- Create: `chapters/02_newton_arch/question-notes.md`
- Modify: `chapters/02_newton_arch/README.md`
- Modify: `chapters/02_newton_arch/source-walkthrough-deep.md`

- [ ] **Step 1: Create `question-notes.md` with one section per learner image**

Include sections for:

```text
- 为什么 `__main__` 会出现两次
- `joint` 到底在干嘛
- `p` / `q` / `axis` 各负责什么
- 为什么 joint 参数可以换一套坐标系还保持同一个物理关节
```

Each section must include:

```text
- the learner image
- what confusion it solves
- what it compresses or simplifies
- what the pinned source says exactly
```

- [ ] **Step 2: Link the supplement from `README.md` and `source-walkthrough-deep.md`**

Add `question-notes.md` to the chapter file map and reading order, and point deep-walkthrough readers to it when they want image-driven confusion notes instead of pure anchor lists.

Expected result: every learner image has a documented place inside chapter 02.

### Task 6: Verify doc consistency

**Files:**
- Modify: `chapters/02_newton_arch/README.md`
- Modify: `chapters/02_newton_arch/source-walkthrough.md`
- Modify: `chapters/02_newton_arch/examples.md`
- Modify: `chapters/02_newton_arch/source-walkthrough-deep.md`
- Create: `chapters/02_newton_arch/question-notes.md`

- [ ] **Step 1: Check markdown image paths and diff hygiene**

Run: `git diff --check -- chapters/02_newton_arch/README.md chapters/02_newton_arch/source-walkthrough.md chapters/02_newton_arch/examples.md chapters/02_newton_arch/source-walkthrough-deep.md chapters/02_newton_arch/question-notes.md docs/superpowers/specs/2026-04-22-chapter-02-image-integration-design.md docs/superpowers/plans/2026-04-22-chapter-02-image-integration.md`
Expected: no output.

- [ ] **Step 2: Review the targeted diff**

Run: `git diff -- chapters/02_newton_arch/README.md chapters/02_newton_arch/source-walkthrough.md chapters/02_newton_arch/examples.md chapters/02_newton_arch/source-walkthrough-deep.md chapters/02_newton_arch/question-notes.md docs/superpowers/specs/2026-04-22-chapter-02-image-integration-design.md docs/superpowers/plans/2026-04-22-chapter-02-image-integration.md`
Expected: only chapter-02 image integration and supporting doc changes appear.
