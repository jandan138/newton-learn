# Chapter 16 Self Experiments Design

## Goal

Refresh `chapters/16_self_experiments` into a complete closing chapter for Newton Learn. Chapter 16 should teach a practical protocol for turning previous chapter conclusions into small, source-backed experiments.

The chapter must not become a new physics topic or a large code project. It should teach how to borrow a minimal official Newton example, identify source-of-truth buffers, change exactly one variable, run a fixed protocol, and write evidence-backed findings.

## User-Facing Positioning

Recommended title:

```text
16 自制小实验：source-backed mini loop
```

Core memory hook:

```text
自制实验不是另起主线；它只验证一个 source-backed 问题。
```

First-pass spine:

```text
choose one source-backed question
-> borrow the smallest official example
-> mark source-of-truth buffers
-> freeze commit / device / seed / dt / frames / solver
-> change exactly one knob
-> run a fixed protocol
-> evaluate state predicate / scalar / buffer / FD check
-> write findings
-> feed the result back to the relevant chapter
```

## Source Anchors

Current Newton source root: `/home/zhuzihou/dev/newton`, commit `0f583176`.

Primary source anchors:

- `newton/examples/basic/example_basic_pendulum.py`: minimal build, runtime, predicate, and viewer loop.
- `newton/examples/basic/example_basic_shapes.py`: one-knob solver / shape / material variation, limited to XPBD/VBD.
- `newton/examples/basic/example_basic_plotting.py`: scalar diagnostics from solver data.
- `newton/examples/diffsim/example_diffsim_ball.py`: verify-before-optimize through Tape and finite differences.
- `newton/examples/multiphysics/example_softbody_dropping_to_cloth.py`: single-model VBD multiphysics predicate branch.
- `newton/examples/mpm/example_mpm_twoway_coupling.py`: two-model / two-solver bridge ownership branch.
- `newton/examples/__init__.py`: example runner, test mode, NaN guard, `test_body_state()`, `test_particle_state()`.
- `newton/_src/sim/state.py`: time-varying q/qd/f buffers, `clear_forces()`, `assign()`, `requires_grad`.
- `newton/_src/sim/model.py`: `state()`, `control()`, `contacts()`, `collide()`, runtime model update caveats.
- `newton/_src/viewer/viewer.py`: `log_state()`, `log_contacts()`, and `apply_forces()` boundary.

Chapter context anchors:

- `chapters/13_diffsim`: verify before optimize; FD hard gate.
- `chapters/14_viewer_integration`: viewer output is read/log/render, with limited write-before-step interactions.
- `chapters/15_multiphysics_pipeline`: coupling boundary and buffer ownership.

## Scope

In scope:

- Rewrite `chapters/16_self_experiments/README.md`.
- Add `principle.md`, `source-walkthrough.md`, `examples.md`, `pitfalls.md`, `exercises.md`.
- Add `chapters/16_self_experiments/assets/` with native imagegen PNG tutorial assets.
- Add this design spec and a tutorial-infographics spec.
- Add an implementation plan under `docs/superpowers/plans/`.
- Verify links, assets, source claims, and style consistency.

Out of scope:

- Creating a full `experiments/` code repository in this pass.
- Running long physics experiments or publishing fake benchmark results.
- Adding new Newton code.
- Changing chapters 00-15 except for unavoidable cross-reference fixes.
- SVG, HTML, screenshot, or terminal-style fallback diagrams.

## Chapter File Responsibilities

### `README.md`

Responsibilities:

- Establish Chapter 16 as the closing practice protocol.
- Explain what counts as an acceptable self-experiment.
- Define scope, prerequisites, completion gates, and reading order.
- Include README PNG anchors:
  - `16_readme_experiment_loop_spine.png`
  - `16_readme_file_roles_reading_order.png`
  - `16_readme_completion_scope_prereq.png`

### `principle.md`

Responsibilities:

- Explain the protocol model: source-backed question, one variable, fixed conditions, evidence ledger, findings feedback.
- Distinguish builder-time, model-time, state-time, control-time, solver-time, and viewer-time changes.
- Map evidence families to source objects.
- Include principle PNG anchors:
  - `16_principle_not_new_mainline_boundary.png`
  - `16_principle_question_to_protocol_ladder.png`
  - `16_principle_official_example_baseline.png`
  - `16_principle_one_variable_contract.png`
  - `16_principle_evidence_ledger.png`
  - `16_principle_metric_family_map.png`
  - `16_principle_findings_feedback_loop.png`
  - `16_principle_experiment_cluster_map.png`

### `source-walkthrough.md`

Responsibilities:

- Use `basic_pendulum` as the first-pass source walkthrough.
- Show how the minimal build/runtime/predicate/viewer loop becomes an experiment protocol.
- Add short branch sections for `basic_shapes`, `basic_plotting`, `diffsim_ball`, `softbody_dropping_to_cloth`, and `mpm_twoway_coupling`.
- Include walkthrough PNG anchors:
  - `16_walkthrough_pipeline_overview.png`
  - `16_walkthrough_beginner_path.png`
  - `16_walkthrough_stage1_choose_claim.png`
  - `16_walkthrough_stage2_clone_baseline_example.png`
  - `16_walkthrough_stage3_freeze_conditions.png`
  - `16_walkthrough_stage4_change_one_knob.png`
  - `16_walkthrough_stage5_record_metrics.png`
  - `16_walkthrough_stage6_compare_and_verdict.png`
  - `16_walkthrough_object_ledger_stop_here.png`

### `examples.md`

Responsibilities:

- Assign each representative example exactly one teaching job.
- Make `basic_pendulum` the first-pass anchor.
- Make `diffsim_ball` the verify-before-optimize anchor.
- Include examples PNG anchors:
  - `16_examples_experiment_role_table.png`
  - `16_examples_fd_validation_anchor.png`

### `pitfalls.md`

Responsibilities:

- Record experiment-specific mistakes that would produce false confidence.
- Include pitfalls PNG anchor:
  - `16_pitfalls_experiment_mistakes.png`

### `exercises.md`

Responsibilities:

- Provide small, checkable protocol design exercises.
- Avoid open-ended project prompts.
- Include exercises PNG anchor:
  - `16_exercises_protocol_design_quiz.png`

## Required Source-Truth Distinctions

The chapter may say:

- `basic_pendulum` is the minimal source-backed experiment anchor.
- `test_body_state()` and `test_particle_state()` are predicate helpers used by examples.
- `examples.run()` runs fixed lifecycle, test hooks, and NaN guard when `--test` is set.
- `State` holds time-varying dynamic quantities.
- `Model.contacts()` allocates a contacts buffer; `Model.collide()` populates it.
- `ViewerBase.log_state()` and `log_contacts()` read/log state and contacts.
- `ViewerBase.apply_forces()` is a write-before-step hook for interaction forces where supported.
- `diffsim_ball` checks analytic gradients against numeric finite differences.
- `mpm_twoway_coupling` uses separate rigid/sand states and explicit bridge buffers.

The chapter must not say:

- Viewer output proves physics correctness.
- Viewer is absolutely read-only in all contexts.
- All examples share identical collision/contact behavior.
- All solvers are interchangeable in all examples.
- Any parameter update automatically propagates to all solvers.
- `tape.backward()` is sufficient without FD validation.
- A single run or a generated chart proves a general solver claim.
- Chapter 16 currently contains a migrated old experiment codebase.

## Visual Direction

Follow `docs/superpowers/specs/chapter-visual-style-guide.md` and the Chapter 13/14/15 raster imagegen style.

Use:

- white background
- dark-blue titles
- thin blue card borders
- numbered blue badges
- compact multi-panel Chinese review sheet composition
- small icons, arrows, flow strips, evidence ledgers, protocol checklists, comparison tables
- short exact English code terms only where useful: `State`, `Contacts`, `why.md`, `findings.md`, `FD`, `seed`, `solver.step()`, `wp.Tape`, `CUDA graph`

Avoid:

- Chapter 02 style 3D diorama images
- decorative screenshots or photorealistic simulation scenes
- fake terminal output, fake code blocks, fake metrics, fake benchmark values
- diagrams that imply viewer output is proof
- diagrams that imply every solver pair is generic-coupled

## Acceptance Criteria

The pass is complete when:

- Chapter 16 has six reader-facing Markdown files.
- All six files include appropriate PNG visual anchors.
- 24 `16_*.png` assets exist under `chapters/16_self_experiments/assets/`.
- Every referenced asset exists and every `16_*.png` asset is referenced.
- Markdown includes no skeleton markers.
- Source claims are consistent with Newton commit `0f583176`.
- Images match the Chapter 03/04/13/14/15 Chinese tutorial infographic style.
- A review pass has checked source truth, paths, and visual consistency.
- Final changes are committed, merged to `main`, pushed, and the temporary worktree is removed.
