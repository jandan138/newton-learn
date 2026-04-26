# Chapter 15 Multiphysics Pipeline Refresh Design

## Goal

Refresh Chapter 15 from the current skeleton into a usable Newton Learn chapter, then add native-imagegen tutorial PNGs in the Chapter 03 / 04 / 14 visual style.

The chapter should answer one first-pass question:

```text
When a Newton scene contains multiple physical systems,
where exactly does coupling happen:
inside one Model/Solver,
through explicit bridge buffers across Models/Solvers,
or only in roadmap/ecosystem language?
```

## User Direction

- Proceed autonomously without stopping for further approval.
- Start from latest `main` in an isolated worktree.
- First confirm whether a previous Chapter 15 branch or body text exists.
- If old正文 exists, migrate it safely by path. If not, record that the check found no old正文 and write from source-backed anchors.
- Follow the same Chapter 03 / 04 tutorial infographic style used by Chapters 13 and 14.
- Use native imagegen PNG assets, not deterministic SVG, HTML, screenshots, or decorative-only rendering.
- Review, commit, and push when verified.

## Safe-Migration Finding

The preflight check found:

- `chapters/15_multiphysics_pipeline/README.md` existed on `main`, but it was only the original skeleton.
- No local worktree for Chapter 15 existed.
- No local or remote branch matching Chapter 15 / multiphysics / pipeline was present.
- Therefore there is no old Chapter 15正文 to migrate in this pass.

The refresh should preserve the skeleton's intent and replace starter prompts with source-backed teaching content.

## Source Anchors

Primary Newton source tree used for this pass:

```text
/home/zhuzihou/dev/newton
upstream commit: 0f583176
```

Primary anchors:

- `newton/examples/multiphysics/example_softbody_dropping_to_cloth.py`: minimal soft body + cloth single-model VBD example.
- `newton/examples/multiphysics/example_softbody_gift.py`: custom soft blocks + cloth straps single-model VBD example.
- `newton/examples/mpm/example_mpm_twoway_coupling.py`: two-model / two-solver rigid + MPM sand bridge.
- `newton/examples/cloth/example_cloth_franka.py`: external rigid solver + cloth VBD bridge.
- `newton/_src/sim/builder.py`: `add_cloth_grid`, `add_cloth_mesh`, `add_soft_grid`, `add_soft_mesh`, `finalize`.
- `newton/_src/solvers/vbd/solver_vbd.py`: VBD / AVBD particle-rigid coupling contract and step phases.
- `newton/_src/solvers/implicit_mpm/solver_implicit_mpm.py`: MPM step, collider setup, impulse collection, collider body mapping.
- `docs/faq.rst`: current roadmap boundary for one-way / two-way / implicit coupling and PhysX distinction.

Current local source has `newton/examples/multiphysics/`, not the old structure-design path `newton/_src/examples/multiphysics/`. The chapter must state the current path and avoid inventing old paths.

## Chapter Teaching Spine

Every reader-facing file should reinforce this spine:

```text
choose the coupling category
-> build material systems into Model(s)
-> allocate State / Control / Contacts
-> configure solver(s)
-> decide the data bridge between systems
-> run step order explicitly
-> verify source-of-truth buffers
-> render/log only after physics update
```

The one-line memory hook is:

```text
多物理不是黑盒；它是 Model / State / Contacts / Solver 之间的耦合边界。
```

## Core Learner Questions

The chapter should let the reader answer:

- Does this example use one `Model` or multiple `Model`s?
- Does this example use one solver or multiple solvers?
- Is coupling internal to `SolverVBD`, or implemented by explicit bridge buffers?
- Which builder APIs create material systems, and why are they not runtime coupling by themselves?
- What role do `Contacts`, `CollisionPipeline`, and bridge buffers play?
- How does MPM two-way coupling move collider impulses back to rigid bodies?
- How does cloth-Franka sequence robot solver, collision pipeline, and cloth solver?
- Which FAQ statements are current source facts, and which are roadmap boundaries?

## Content Scope

In scope:

- Rewrite `chapters/15_multiphysics_pipeline/README.md`.
- Create `principle.md`, `source-walkthrough.md`, `examples.md`, `pitfalls.md`, and `exercises.md`.
- Create `chapters/15_multiphysics_pipeline/assets/`.
- Add native-imagegen PNG visual anchors to all six reader-facing files.
- Use source snippets and line references sparingly, keeping exact semantics in Markdown.

Out of scope:

- Full VBD mathematical derivation.
- Full MPM solver derivation.
- General implicit co-simulation API claims not present in current source.
- Treating viewer output as coupling correctness evidence.
- Rewriting examples or adding runnable experiment code.

## File Roles

- `README.md`: chapter entry, main question, coupling categories, scope, completion gate.
- `principle.md`: conceptual model: builder vs runtime, contacts as coupling surface, single-model VBD, two-model bridge, external-solver bridge, roadmap boundary.
- `source-walkthrough.md`: source-backed path through current inventory, softbody/cloth VBD, gift example, MPM bridge, cloth-Franka bridge.
- `examples.md`: observation notes for the four representative examples.
- `pitfalls.md`: common mistakes and correction patterns.
- `exercises.md`: small self-checks for coupling category and source-of-truth buffers.

## Visual Design

Follow `docs/superpowers/specs/chapter-visual-style-guide.md`.

Use:

- white background with thin blue rounded card borders
- dark-blue title and section headings
- numbered blue badges for coupling stages, source objects, bridge buffers, and pitfalls
- Chinese-first labels and short exact English/code labels such as `Model`, `State`, `Contacts`, `ModelBuilder`, `SolverVBD`, `SolverImplicitMPM`, `SolverMuJoCo`, `SolverFeatherstone`, `CollisionPipeline`, `add_soft_grid()`, `add_cloth_grid()`, `model.collide()`, `solver.step()`, `collect_collider_impulses()`
- compact cards, arrows, two-lane source-of-truth diagrams, one-model vs two-model maps, bridge-buffer strips, and bottom recap bars
- concrete drawings of soft body blocks, cloth sheets/straps, sand particles, rigid bodies, bridge buffers, contact surfaces, and state ledgers

Avoid:

- Chapter 02-style 3D diorama rendering
- decorative multiphysics scenes with no source/data boundary
- fake source code, fake terminal output, or invented APIs
- long paragraphs inside images
- diagrams implying every solver pair is automatically coupled
- diagrams implying FAQ roadmap items are already current APIs
- diagrams implying viewer output is coupling correctness proof

## Calibration Image

The first generated image should be:

```text
chapters/15_multiphysics_pipeline/assets/15_readme_multiphysics_boundary_map.png
```

It should show:

```text
single Model / SolverVBD lane:
  soft body + cloth -> Model/State/Contacts -> SolverVBD -> state update

two Model bridge lane:
  rigid Model + sand Model -> impulses/body forces bridge -> two solvers -> two states

external solver bridge lane:
  robot solver -> CollisionPipeline -> cloth SolverVBD
```

Acceptance check:

- It looks like the Chapter 03 / 04 dense Chinese tutorial handout style.
- It is instructional rather than decorative.
- It clearly separates one-model, two-model, and external-solver coupling categories.
- It uses short labels and only real API names.
- It does not imply generic implicit co-simulation exists in current source.

## Asset Plan

Use 24 PNGs:

- `15_readme_multiphysics_boundary_map.png`
- `15_readme_file_roles_reading_order.png`
- `15_readme_completion_scope_prereq.png`
- `15_principle_one_model_vs_two_models.png`
- `15_principle_coupling_contract_ladder.png`
- `15_principle_builder_material_ledger.png`
- `15_principle_contacts_are_coupling_surface.png`
- `15_principle_vbd_single_solver_loop.png`
- `15_principle_mpm_twomodel_impulse_bridge.png`
- `15_principle_external_rigid_solver_bridge.png`
- `15_principle_roadmap_scope_table.png`
- `15_walkthrough_pipeline_overview.png`
- `15_walkthrough_beginner_path.png`
- `15_walkthrough_stage1_example_inventory.png`
- `15_walkthrough_stage2_softbody_cloth_build.png`
- `15_walkthrough_stage3_vbd_step_contract.png`
- `15_walkthrough_stage4_gift_geometry_and_contacts.png`
- `15_walkthrough_stage5_mpm_twoway_bridge.png`
- `15_walkthrough_stage6_cloth_franka_bridge.png`
- `15_walkthrough_object_ledger_stop_here.png`
- `15_examples_role_table.png`
- `15_examples_softbody_gift_anchor.png`
- `15_pitfalls_coupling_mistakes.png`
- `15_exercises_coupling_boundary_quiz.png`

## Source-Truth Discipline

Markdown remains authoritative for:

- exact file paths and line references
- source excerpts
- whether an example uses one model or multiple models
- whether a bridge is explicit and staggered
- the distinction between current source, FAQ roadmap, and viewer output

Images may compress relationships, but must not contradict Markdown. In particular:

- VBD diagrams must show single-model / single-solver coupling for current multiphysics examples.
- MPM diagrams must show two `Model`/state families and bridge buffers.
- Cloth-Franka diagrams must show robot solver before collision pipeline before cloth solver.
- FAQ diagrams must mark two-way / implicit coupling as roadmap unless backed by the specific source example being discussed.

## Acceptance Criteria

The pass is complete when:

- Chapter 15 no longer contains skeleton starter text.
- The chapter records that no old Chapter 15正文 branch/worktree was found.
- All six reader-facing Markdown files exist and share the same coupling-boundary teaching spine.
- `README.md`, `principle.md`, `source-walkthrough.md`, `examples.md`, `pitfalls.md`, and `exercises.md` include Chapter 15 PNG visual anchors.
- All image references point to existing PNG files under `chapters/15_multiphysics_pipeline/assets/`.
- All assets are native-imagegen raster PNGs with stable `15_` filenames.
- No generated image introduces SVG/HTML fallbacks, fake source, or fake terminal UI.
- The visual style follows the Chapter 03 / 04 / 14 dense Chinese learning-handout direction.
- Verification confirms no broken Markdown image paths, no missing assets, and no accidental edits outside Chapter 15 support docs unless intentionally listed.
