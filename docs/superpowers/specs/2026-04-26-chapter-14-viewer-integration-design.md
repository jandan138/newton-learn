# Chapter 14 Viewer Integration Refresh Design

## Goal

Refresh Chapter 14 from the current skeleton into a usable Newton Learn chapter, then add native-imagegen tutorial PNGs in the Chapter 03 / 04 visual style.

The chapter should answer one first-pass question:

```text
solver 已经把 state 推到下一拍之后，
viewer 到底读了什么、记录了什么，
哪些用户输入又会在下一拍之前写回 state？
```

## User Direction

- Proceed autonomously without stopping for further approval.
- Start from latest `main` in an isolated worktree.
- First confirm whether a previous Chapter 14 branch or body text exists.
- If old正文 exists, migrate it safely by path. If not, record that the check found no old正文 and write the chapter from source-backed anchors.
- Follow the same Chapter 03 / 04 tutorial infographic style used by the recent chapter image passes.
- Use native imagegen PNG assets, not deterministic SVG, HTML, screenshots, or decorative-only rendering.
- Review, commit, and push when the refresh is verified.

## Safe-Migration Finding

The preflight check found:

- `chapters/14_viewer_integration/README.md` existed on `main`, but it was only the original skeleton.
- No local worktree for Chapter 14 existed.
- No local or remote branch matching Chapter 14 / viewer / integration was present.
- Therefore there is no old Chapter 14正文 to migrate in this pass.

The refresh should preserve the intent of the skeleton and replace its starter prompts with source-backed teaching content.

## Source Anchors

Primary Newton source tree used for this pass:

```text
/home/zhuzihou/dev/newton
upstream commit: 0f583176
```

Primary anchors:

- `newton/viewer.py`: public viewer exports.
- `newton/_src/viewer/__init__.py`: high-level viewer module contract.
- `newton/_src/viewer/viewer.py`: `ViewerBase`, `set_model`, `begin_frame`, `log_state`, `log_contacts`, `apply_forces`.
- `newton/_src/viewer/viewer_gl.py`: interactive GL lifecycle, pause state, picking/wind write-back.
- `newton/_src/viewer/viewer_usd.py`: time-sampled USD output backend.
- `newton/_src/viewer/viewer_rerun.py`: Rerun backend and timeline behavior.
- `newton/_src/viewer/viewer_file.py`: file recording backend.
- `newton/_src/viewer/viewer_null.py`: headless/test/benchmark backend.
- `newton/_src/viewer/viewer_viser.py`: browser/Jupyter viewer backend and `.viser` recording path.
- `newton/examples/__init__.py`: CLI viewer choice and outer example run loop.
- `newton/examples/basic/example_basic_pendulum.py`: minimal example loop with `apply_forces`, solver step, `log_state`, and `log_contacts`.
- `newton/examples/ik/example_ik_franka.py`: state freshness before `log_gizmo` and `log_state`.
- `newton/examples/basic/example_basic_viewer.py`: direct viewer overlay/logging demo.
- `newton/examples/basic/example_recording.py`: `ViewerFile` recording demo.
- `newton/examples/robot/example_robot_policy.py`: IsaacLab-trained policy usage example.
- `docs/guide/visualization.rst`: upstream guide for common viewer interface and backend choice.
- `docs/integrations/isaac-lab.rst` and `docs/faq.rst`: Isaac Lab / Isaac Sim integration boundary.

The current local source does not expose a concrete MJX or Gymnasium adapter path. Chapter 14 may mention those as ecosystem-boundary vocabulary from the original structure design, but must not invent source APIs or walkthrough stages for them.

## Chapter Teaching Spine

Every reader-facing file should reinforce this spine:

```text
Model / State / Contacts already exist
-> examples init selects a viewer backend
-> example runner owns the outer lifecycle
-> each frame can receive viewer input before solver.step()
-> solver/collision updates State and Contacts
-> render phase begins a frame
-> viewer logs State, Contacts, gizmos, overlays, arrays, or scalars
-> backend presents, records, exports, or streams the same viewer contract
-> close/save/disconnect finishes the backend
```

The one-line memory hook is:

```text
viewer 是读/记录/展示边界，不是第二套 physics。
```

## Core Learner Questions

The chapter should let the reader answer:

- Why is viewer code outside the solver inner loop?
- Which part of the example runner decides whether the loop continues?
- What does `viewer.is_paused()` pause, and what still renders?
- Why does `apply_forces(state)` run before simulation stepping in interactive examples?
- What does `log_state(state)` read from `State`, and why must state be fresh?
- Why are `log_contacts(contacts, state)` arrows overlays of an existing `Contacts` buffer, not collision generation?
- How do `ViewerGL`, `ViewerUSD`, `ViewerRerun`, `ViewerFile`, `ViewerNull`, and `ViewerViser` share one interface while producing different outputs?
- What belongs to Chapter 14's ecosystem boundary, and what remains a later integration detail?

## Content Scope

In scope:

- Rewrite `chapters/14_viewer_integration/README.md`.
- Create `principle.md`, `source-walkthrough.md`, `examples.md`, `pitfalls.md`, and `exercises.md`.
- Create `chapters/14_viewer_integration/assets/`.
- Add native-imagegen PNG visual anchors to all six reader-facing files.
- Use source snippets and line references sparingly, keeping exact semantics in Markdown.

Out of scope:

- Full OpenGL renderer internals.
- Full USD authoring tutorial; Chapter 04 already covers scene-to-`Model` import.
- Full Rerun, Viser, Omniverse, Isaac Lab, MJX, or Gymnasium tutorials.
- Performance benchmarking methodology beyond explaining `ViewerNull` / benchmark mode at the boundary level.
- Any source claim for MJX or Gymnasium adapter APIs not present in the current local source.

## File Roles

- `README.md`: chapter entry, main question, file map, backend role table, scope, completion gate.
- `principle.md`: conceptual boundary model: outer loop, read/log side, write-before-step side, backend choice.
- `source-walkthrough.md`: source-backed path through CLI selection, runner loop, example step/render, viewer base methods, backend lifecycles.
- `examples.md`: observation notes for `basic_pendulum`, `basic_viewer`, `recording`, `ik_franka`, `sensor_tiled_camera`, and `robot_policy`.
- `pitfalls.md`: common mistakes and correction patterns.
- `exercises.md`: small self-checks that require annotating read/write boundaries.

## Visual Design

Follow `docs/superpowers/specs/chapter-visual-style-guide.md`.

Use:

- white background with thin blue rounded card borders
- dark-blue title and section headings
- numbered blue badges for loop stages, API roles, backend choices, and pitfalls
- Chinese-first labels and short exact English/code labels such as `ViewerBase`, `set_model()`, `is_running()`, `is_paused()`, `apply_forces()`, `begin_frame()`, `log_state()`, `log_contacts()`, `end_frame()`, `ViewerGL`, `ViewerUSD`, `ViewerRerun`, `ViewerNull`, `State`, `Contacts`
- compact cards, arrows, two-lane read/write diagrams, backend comparison tables, lifecycle strips, and bottom recap bars
- concrete drawings of a simulation state ledger, contact arrows as overlays, viewer window, USD file, Rerun timeline, and headless test counter

Avoid:

- Chapter 02-style 3D diorama rendering
- photorealistic viewer screenshots
- fake source code, fake terminal output, or invented API names
- long paragraphs inside images
- tiny code-heavy tables that become unreadable in Markdown
- diagrams implying that viewer runs collision, solver integration, or gradient logic
- diagrams implying MJX/Gymnasium source adapters exist in the current local source unless future source anchors are added

## Calibration Image

The first generated image should be:

```text
chapters/14_viewer_integration/assets/14_readme_viewer_boundary_spine.png
```

It should show:

```text
Model / State / Contacts
-> optional viewer input before step
-> solver step updates State / Contacts
-> begin_frame()
-> log_state() / log_contacts() / log_gizmo()
-> backend output: GL window | USD file | Rerun timeline | file recording | null test
```

Acceptance check:

- It looks like the Chapter 03 / 04 dense Chinese tutorial handout style.
- It is instructional rather than decorative.
- It clearly separates read/log/render arrows from optional write-before-step arrows.
- It uses short labels and only real API names.
- It does not imply viewer is a solver or collision generator.

## Asset Plan

Use 24 PNGs:

- `14_readme_viewer_boundary_spine.png`
- `14_readme_file_roles_reading_order.png`
- `14_readme_completion_scope_prereq.png`
- `14_principle_not_solver_boundary.png`
- `14_principle_outer_loop_two_lanes.png`
- `14_principle_common_interface_map.png`
- `14_principle_log_state_reads_state.png`
- `14_principle_contacts_are_overlay.png`
- `14_principle_input_writeback_timing.png`
- `14_principle_backend_choice_table.png`
- `14_principle_ecosystem_bridge_scope.png`
- `14_walkthrough_pipeline_overview.png`
- `14_walkthrough_beginner_path.png`
- `14_walkthrough_stage1_cli_backend_selection.png`
- `14_walkthrough_stage2_runner_loop.png`
- `14_walkthrough_stage3_example_step_write_boundary.png`
- `14_walkthrough_stage4_render_log_boundary.png`
- `14_walkthrough_stage5_backend_lifecycle.png`
- `14_walkthrough_stage6_debug_overlays.png`
- `14_walkthrough_object_ledger_stop_here.png`
- `14_examples_role_table.png`
- `14_examples_basic_pendulum_anchor.png`
- `14_pitfalls_boundary_mistakes.png`
- `14_exercises_loop_annotation_quiz.png`

## Source-Truth Discipline

Markdown remains authoritative for:

- exact file paths and line references
- source excerpts
- backend-specific caveats
- the distinction between viewer read/log work and physics state mutation
- the current-source absence of concrete MJX/Gymnasium adapters

Images may compress relationships, but must not contradict Markdown. Every compressed diagram should be introduced by nearby prose that states it is a teaching map rather than exact call-order truth.

## Acceptance Criteria

The pass is complete when:

- Chapter 14 no longer contains skeleton starter text.
- The chapter records that no old Chapter 14正文 branch/worktree was found.
- All six reader-facing Markdown files exist and share the same viewer-boundary teaching spine.
- `README.md`, `principle.md`, `source-walkthrough.md`, `examples.md`, `pitfalls.md`, and `exercises.md` include Chapter 14 PNG visual anchors.
- All image references point to existing PNG files under `chapters/14_viewer_integration/assets/`.
- All assets are native-imagegen raster PNGs with stable `14_` filenames.
- No generated image introduces SVG/HTML fallbacks, fake source, or fake terminal UI.
- The visual style follows the Chapter 03 / 04 dense Chinese learning-handout direction.
- Verification confirms no broken Markdown image paths, no missing assets, and no accidental edits outside Chapter 14 support docs unless intentionally listed.
