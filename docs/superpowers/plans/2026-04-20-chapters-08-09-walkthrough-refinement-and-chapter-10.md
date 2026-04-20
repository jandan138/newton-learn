# Chapters 08 09 Walkthrough Refinement And Chapter 10 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Refine the main walkthroughs for chapters 08 and 09 to match the stronger beginner standard, then create the first full teaching-first mainline package for chapter 10.

**Architecture:** Keep the repo-wide beginner/deep walkthrough split intact. Only rewrite the two main walkthroughs that are still pedagogically fast, and build chapter 10 around a representation-first storyline: cloth, softbody, and cable are three different internal object families, not one bigger solver catalog.

**Tech Stack:** Markdown docs, git worktree, excerpt-backed main walkthroughs, upstream Newton source anchors, multi-round review, `git diff --check`.

---

## File Map

- Modify:
  - `chapters/08_rigid_solvers/source-walkthrough.md`
  - `chapters/10_softbody_cloth_cable/README.md`
  - `chapters/09_variational_solvers/source-walkthrough.md`
- Create:
  - `chapters/10_softbody_cloth_cable/principle.md`
  - `chapters/10_softbody_cloth_cable/examples.md`
  - `chapters/10_softbody_cloth_cable/source-walkthrough.md`
  - `docs/superpowers/specs/2026-04-20-chapters-08-09-walkthrough-refinement-and-chapter-10-design.md`
  - `docs/superpowers/plans/2026-04-20-chapters-08-09-walkthrough-refinement-and-chapter-10.md`

## Task 1: Refine Chapter 08 Main Walkthrough

**Files:**
- Modify: `chapters/08_rigid_solvers/source-walkthrough.md`

- [ ] **Step 1: Replace the abstract opening with a stronger beginner question**

The top of the file should answer this user-side question before solver taxonomy:

```text
我只改了一行 self.solver = ...，为什么同一个 step(...) 后面就像换了一套世界？
```

- [ ] **Step 2: Reframe the chapter around “who gets treated as the first major internal object”**

Make the single story spine feel like:

```text
same public step(...)
-> SemiImplicit first treats forces as the main object
-> Featherstone first treats joint-space articulation quantities as the main object
-> MuJoCo first treats backend bridge state as the main object
-> Kamino first treats contact / constraint solve objects as the main object
```

- [ ] **Step 3: Add beginner-safe first-use definitions and stronger checkpoints**

Especially define or soften:

```md
- public contract
- force route
- joint-space route
- backend bridge
- chapter 07 recap: Contacts -> rows -> Jacobians -> Delassus
```

Each family stage should end with a short checkpoint sentence that tells the reader what they should now be able to say.

## Task 2: Refine Chapter 09 Main Walkthrough

**Files:**
- Modify: `chapters/09_variational_solvers/source-walkthrough.md`

- [ ] **Step 1: Strengthen the opening around the shared cloth correction problem**

The file should answer this question early:

```text
同一块 hanging cloth、同一个 collide -> step -> swap 外层 loop，solver 到底先修哪一层：一条约束、一个局部块，还是整张布？
```

- [ ] **Step 2: Make the walkthrough track the first repeatedly updated container in each solver**

The main ladder should feel like:

```text
same cloth loop
-> XPBD repeatedly updates per-constraint deltas / lambdas
-> VBD repeatedly updates local force + Hessian containers and local displacements
-> Style3D repeatedly updates global rhs / dx around a fixed PD matrix
```

- [ ] **Step 3: Add earlier plain-language definitions and sharper scope control**

Especially define or soften:

```md
- variational / implicit correction
- predicted state
- inertia target
- lambda
- Hessian
- PCG
```

Also make the cloth mainline and the chapter-10 bridge explicit: `softbody_hanging` appears here only as the VBD extension, not as a second main walkthrough axis.

## Task 3: Build Chapter 10 Mainline Package

**Files:**
- Modify: `chapters/10_softbody_cloth_cable/README.md`
- Create: `chapters/10_softbody_cloth_cable/principle.md`
- Create: `chapters/10_softbody_cloth_cable/examples.md`
- Create: `chapters/10_softbody_cloth_cable/source-walkthrough.md`

- [ ] **Step 1: Rewrite the README around a representation-first chapter promise**

The README should clearly answer:

```text
cloth、softbody、cable 看起来都在“会弯会挂会掉”，但在 Newton 里它们真的是同一种对象吗？
```

It should make clear that chapter 10 is about internal object families and builder helpers, not a repeat of chapter 09 solver math.

- [ ] **Step 2: Write `principle.md` around cloth -> softbody -> cable object representation**

The conceptual spine should feel like:

```text
cloth = particles + triangles + edges / optional springs
softbody = particles + tetrahedra + surface collision mesh
cable = capsule rigid bodies + cable joints (+ articulation)
```

Explain what changes in state representation and why that matters.

- [ ] **Step 3: Write `examples.md` around three teaching anchors with distinct jobs**

Use:

```text
example_cloth_hanging.py
example_softbody_hanging.py
example_cable_twist.py
```

Each example should have a clear teaching role, not a mixed bag of observations.

- [ ] **Step 4: Write `source-walkthrough.md` as a builder-first main walkthrough**

Track this necessity chain:

```text
choose builder helper
-> helper expands into a different internal object graph
-> that choice decides which state fields and solver paths now matter
```

The main source anchors should stay focused on:

```text
builder.add_cloth_grid(...)
builder.add_soft_grid(...)
builder.add_rod(...)
```

## Task 4: Final Review And Verification

**Files:**
- Verify all modified and created chapter docs plus the supporting spec/plan files.

- [ ] **Step 1: Run file-level diff checks**

Run:

`git diff --check -- chapters/08_rigid_solvers/source-walkthrough.md chapters/09_variational_solvers/source-walkthrough.md chapters/10_softbody_cloth_cable/README.md chapters/10_softbody_cloth_cable/principle.md chapters/10_softbody_cloth_cable/examples.md chapters/10_softbody_cloth_cable/source-walkthrough.md docs/superpowers/specs/2026-04-20-chapters-08-09-walkthrough-refinement-and-chapter-10-design.md docs/superpowers/plans/2026-04-20-chapters-08-09-walkthrough-refinement-and-chapter-10.md`

Expected: no output.

- [ ] **Step 2: Run placeholder / template residue scan on chapter 10**

Check that the old scaffold placeholders are gone from the new chapter-10 docs.

- [ ] **Step 3: Re-read for teaching continuity across 08 -> 09 -> 10**

Confirm that:

```text
08 explains different rigid solver families
09 explains different variational correction levels
10 explains different deformable object representations
```

- [ ] **Step 4: Review the final diff**

Run:

`git diff -- chapters/08_rigid_solvers/source-walkthrough.md chapters/09_variational_solvers/source-walkthrough.md chapters/10_softbody_cloth_cable/README.md chapters/10_softbody_cloth_cable/principle.md chapters/10_softbody_cloth_cable/examples.md chapters/10_softbody_cloth_cable/source-walkthrough.md docs/superpowers/specs/2026-04-20-chapters-08-09-walkthrough-refinement-and-chapter-10-design.md docs/superpowers/plans/2026-04-20-chapters-08-09-walkthrough-refinement-and-chapter-10.md`

Expected: the diff stays tightly focused on beginner clarity, mainline chapter content, and chapter-to-chapter continuity.
