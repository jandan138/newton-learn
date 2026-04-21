# Chapter 10 Deep And Chapter 11 MPM Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a deep-track walkthrough for chapter 10 and build the first full teaching-first mainline package for chapter 11 MPM.

**Architecture:** Keep chapter 10’s main walkthrough unchanged in role and add a separate deep anchor sheet that traces builder-side family handoffs. Build chapter 11 around one dataflow-first spine: `particle -> grid -> solve -> particle`, using `example_mpm_granular.py` as the main anchor and keeping `snow_ball` and `twoway_coupling` as scoped branches.

**Tech Stack:** Markdown docs, git worktree, upstream Newton source anchors, multi-agent review, `git diff --check`.

---

## File Map

- Modify:
  - `chapters/10_softbody_cloth_cable/README.md`
  - `chapters/10_softbody_cloth_cable/source-walkthrough.md`
  - `chapters/11_mpm/README.md`
- Create:
  - `chapters/10_softbody_cloth_cable/source-walkthrough-deep.md`
  - `chapters/11_mpm/principle.md`
  - `chapters/11_mpm/examples.md`
  - `chapters/11_mpm/source-walkthrough.md`
  - `docs/superpowers/specs/2026-04-21-chapter-10-deep-and-chapter-11-mpm-design.md`
  - `docs/superpowers/plans/2026-04-21-chapter-10-deep-and-chapter-11-mpm.md`

## Task 1: Add Chapter 10 Deep Walkthrough

**Files:**
- Create: `chapters/10_softbody_cloth_cable/source-walkthrough-deep.md`
- Modify: `chapters/10_softbody_cloth_cable/README.md`
- Modify: `chapters/10_softbody_cloth_cable/source-walkthrough.md`

- [ ] **Step 1: Write the deep walkthrough using the existing deep-track structure**

The new file should use:

```text
Fast Deep Index
Exact Handoff Trace
Optional Branches
Verification Anchors
```

- [ ] **Step 2: Keep the trace builder-first, not solver-first**

The `Exact Handoff Trace` should cover:

```text
Builder already splits the families
Cloth: add_cloth_grid -> add_cloth_mesh
Softbody: add_soft_grid -> particles -> tetrahedra -> generated surface mesh
Cable: geometry path -> add_rod -> add_joint_cable -> add_articulation
Cross-family state ledger
```

- [ ] **Step 3: Update chapter 10 entry points to acknowledge the new deep file**

`README.md` should stop saying there is no deep walkthrough, and `source-walkthrough.md` should point readers to the new deep file in `Go Deeper`.

## Task 2: Rewrite Chapter 11 README Around the MPM Dataflow

**Files:**
- Modify: `chapters/11_mpm/README.md`

- [ ] **Step 1: Replace the skeleton with a teaching-first chapter promise**

The README should clearly establish that chapter 11 is about:

```text
particle -> grid -> solve -> particle
```

not a fake binary of `APIC solver vs implicit solver`.

- [ ] **Step 2: Establish example roles and chapter boundaries**

The README should clearly separate:

```text
example_mpm_granular.py as mainline
example_mpm_snow_ball.py as material-history branch
example_mpm_twoway_coupling.py as advanced coupling branch
```

## Task 3: Write Chapter 11 Principle Doc

**Files:**
- Create: `chapters/11_mpm/principle.md`

- [ ] **Step 1: Explain the MPM object model before solver details**

The doc should first explain:

```text
particles are persistent material carriers
grid is per-step workspace
```

- [ ] **Step 2: Build the three-step ladder**

The principle should organize the chapter around:

```text
P2G transfer
grid-side solve
G2P transfer + particle update
```

- [ ] **Step 3: Make Newton-specific state placement explicit**

Explain the role of:

```text
register_custom_attributes(...)
model.mpm.*
state.mpm.*
```

## Task 4: Write Chapter 11 Examples Doc

**Files:**
- Create: `chapters/11_mpm/examples.md`

- [ ] **Step 1: Give each example one clear teaching job**

The examples page should not be a flat MPM demo catalog. Each example should have a single role:

```text
granular = mainline pipeline
snow_ball = particle material/history branch
twoway_coupling = rigid-body coupling branch
```

- [ ] **Step 2: Add observation tasks that reinforce the particle-grid-particle story**

Especially direct the reader to look at `register_custom_attributes`, particle emission, `solver.step(...)`, and any post-step projection / coupling logic.

## Task 5: Write Chapter 11 Main Walkthrough

**Files:**
- Create: `chapters/11_mpm/source-walkthrough.md`

- [ ] **Step 1: Keep the repo’s main walkthrough structure**

The file should use:

```text
What This Walkthrough Follows
One-Screen Chapter Map
Beginner Path
Main Walkthrough
Object Ledger
Stop Here
Go Deeper
```

- [ ] **Step 2: Keep one story spine: particle -> grid -> solve -> particle**

The main walkthrough should track this chain through the actual upstream anchors:

```text
register_custom_attributes
ImplicitMPMModel.setup_particle_material
SolverImplicitMPM.step
P2G / APIC transfer
implicit rheology / collision solve
G2P / particle update
```

- [ ] **Step 3: Keep advanced branches out of the mainline**

Do not let the main walkthrough get buried in:

```text
full constitutive math
basis-function catalog
solver/config option tables
two-way coupling internals
```

## Task 6: Final Review And Verification

**Files:**
- Verify all modified and created chapter docs plus the supporting spec/plan files.

- [ ] **Step 1: Run file-level diff checks**

Run:

`git diff --check -- chapters/10_softbody_cloth_cable/README.md chapters/10_softbody_cloth_cable/source-walkthrough.md chapters/10_softbody_cloth_cable/source-walkthrough-deep.md chapters/11_mpm/README.md chapters/11_mpm/principle.md chapters/11_mpm/examples.md chapters/11_mpm/source-walkthrough.md docs/superpowers/specs/2026-04-21-chapter-10-deep-and-chapter-11-mpm-design.md docs/superpowers/plans/2026-04-21-chapter-10-deep-and-chapter-11-mpm.md`

Expected: no output.

- [ ] **Step 2: Run placeholder scan on chapter 11**

Check that the old chapter-11 skeleton placeholders are gone.

- [ ] **Step 3: Re-read for continuity across 10 -> 11**

Confirm the handoff now reads like:

```text
10: different deformable object families
11: a different object family again, this time with particles as material carriers and grid as workspace
```

- [ ] **Step 4: Review the final diff**

Run:

`git diff -- chapters/10_softbody_cloth_cable/README.md chapters/10_softbody_cloth_cable/source-walkthrough.md chapters/10_softbody_cloth_cable/source-walkthrough-deep.md chapters/11_mpm/README.md chapters/11_mpm/principle.md chapters/11_mpm/examples.md chapters/11_mpm/source-walkthrough.md docs/superpowers/specs/2026-04-21-chapter-10-deep-and-chapter-11-mpm-design.md docs/superpowers/plans/2026-04-21-chapter-10-deep-and-chapter-11-mpm.md`

Expected: the diff stays focused on deep anchors for chapter 10 and a teaching-first MPM mainline for chapter 11.
