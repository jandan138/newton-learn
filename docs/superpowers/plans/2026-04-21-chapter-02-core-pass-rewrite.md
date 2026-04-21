# Chapter 02 Core Pass Rewrite Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rewrite chapter 02 so beginners can follow a narrower core source path, understand which source to defer, and read the `basic_pendulum` assembly/simulate chain with less guesswork.

**Architecture:** Keep the chapter's existing entry-to-runtime spine, but tighten the scope. `README.md` will become the scope gate for core-vs-deferred reading, and `source-walkthrough.md` will become a more code-first walkthrough with explicit line meanings, variable-name decoding, `eval_fk` explanation, and a three-step-level model.

**Tech Stack:** Markdown docs, upstream Newton source excerpts, teaching-first walkthrough structure, `git diff --check`.

---

## File Map

- Modify:
  - `chapters/02_newton_arch/README.md`
  - `chapters/02_newton_arch/source-walkthrough.md`
- Create:
  - `docs/superpowers/specs/2026-04-21-chapter-02-core-pass-rewrite-design.md`
  - `docs/superpowers/plans/2026-04-21-chapter-02-core-pass-rewrite.md`

## Task 1: Rewrite Chapter 02 README As A Scope Gate

**Files:**
- Modify: `chapters/02_newton_arch/README.md`

- [ ] **Step 1: Add a `Core Pass` section with exact files and symbols**

The README should explicitly list the first-pass reading targets:

```text
newton/examples/__main__.py
newton/examples/__init__.py -> get_examples(), main(), run()
newton/examples/basic/example_basic_pendulum.py -> Example.__init__(), simulate(), step()
newton/__init__.py -> public exports
newton/_src/sim/model.py -> state(), control(), contacts(), collide()
newton/_src/solvers/solver.py -> step(...)
```

- [ ] **Step 2: Add a `Defer For Now` section**

The README should explicitly mark the first-pass skips:

```text
examples/__init__.py viewer/parser plumbing
example_basic_pendulum.py capture()/render()/test_final()
ModelBuilder.finalize() internals
xpbd solver internals
other examples source tracing
```

- [ ] **Step 3: Tighten the reading order and completion checks**

- [ ] **Step 3: Add a `Source Note`**

The README should explicitly state that `newton/...` paths refer to the upstream Newton source checkout, not files inside `newton-learn`.

- [ ] **Step 4: Tighten the reading order and completion checks**

The reading order should become:

```text
principle.md (optional warm-up)
-> source-walkthrough.md (first-pass mainline)
-> run basic_pendulum
-> examples.md
-> source-walkthrough-deep.md
```

The completion gate should explicitly require:

```text
I can point to Example.__init__() as the assembly point.
I can explain why eval_fk(...) appears in the constructor.
I can distinguish run(), Example.step(), and Solver.step(), and place simulate() under Example.step().
I can explain why state_0 and state_1 swap roles.
```

## Task 2: Rewrite Chapter 02 Main Walkthrough Around Concrete Source Reading

**Files:**
- Modify: `chapters/02_newton_arch/source-walkthrough.md`

- [ ] **Step 1: Tighten the upfront scope statement**

In the early sections, make the first-pass scope explicit: this walkthrough follows one example, one runtime stack, and one simulate loop; it does not ask the reader to study `finalize()` internals or XPBD internals.

- [ ] **Step 2: Add a small public-surface excerpt**

Quote the `newton/__init__.py` export block for:

```python
Model
ModelBuilder
State
Control
Contacts
eval_fk
solvers
```

Then explain why this file is being read: not for package trivia, but to identify the names Newton wants users to learn first.

- [ ] **Step 3: Expand the `Example.__init__()` excerpt into a guided read**

The walkthrough should annotate the `basic_pendulum` constructor so the reader can tell what each line is doing:

```python
builder = newton.ModelBuilder()
link_0 = builder.add_link()
builder.add_shape_box(link_0, ...)
j0 = builder.add_joint_revolute(...)
builder.add_articulation([j0, j1], label="pendulum")
self.model = builder.finalize()
self.solver = newton.solvers.SolverXPBD(self.model)
self.state_0 = self.model.state()
self.state_1 = self.model.state()
self.control = self.model.control()
newton.eval_fk(...)
self.contacts = self.model.contacts()
```

The prose must explicitly decode `link`, `shape`, `joint`, `articulation`, `Model`, `State`, `Control`, and `Contacts`.

- [ ] **Step 4: Add a variable-name cheat sheet plus `eval_fk(...)` explanation**

Include a compact table covering:

```text
q
qd
body_q
body_qd
joint_q
joint_qd
body_f
```

Then explain `eval_fk(...)` as the initializer that expands joint-space values into body-space state before stepping.

- [ ] **Step 5: Add a `Three Step Levels` explanation**

The walkthrough should clearly separate:

```text
examples.run() -> outer timing loop
Example.step() -> frame-level wrapper
Solver.step() -> physics solver contract
```

- [ ] **Step 6: Expand the `simulate()` excerpt with double-buffer semantics**

Annotate the five core calls:

```python
state_0.clear_forces()
viewer.apply_forces(state_0)
model.collide(state_0, contacts)
solver.step(state_0, state_1, control, contacts, sim_dt)
state_0, state_1 = state_1, state_0
```

The prose must explain why the solver reads `state_0`, writes `state_1`, and then swaps them.

- [ ] **Step 7: Demote non-core branches**

Mentions of `capture()` and graph replay should be reduced to a clear “skip this for first pass” note instead of a side topic.

## Task 3: Verification And Review

**Files:**
- Verify the modified chapter docs and the supporting spec/plan files.

- [ ] **Step 1: Run file-level diff checks**

Run:

`git diff --check -- chapters/02_newton_arch/README.md chapters/02_newton_arch/source-walkthrough.md docs/superpowers/specs/2026-04-21-chapter-02-core-pass-rewrite-design.md docs/superpowers/plans/2026-04-21-chapter-02-core-pass-rewrite.md`

Expected: no output.

- [ ] **Step 2: Re-read for teaching-order discipline**

Confirm the edited docs now make these boundaries explicit:

```text
core pass: launcher -> Example.__init__() -> runtime objects -> simulate loop
defer: viewer plumbing, finalize() internals, xpbd internals
```

- [ ] **Step 3: Re-read for source-location clarity**

Confirm the edited docs now explicitly answer where the referenced `newton/...` files live and do not assume those files exist inside `newton-learn`.

- [ ] **Step 4: Beginner dry-run check**

Re-read the chapter as if you were a new learner and verify you can answer, from the docs alone:

```text
Which repo do I open for the cited source files?
Which files/functions do I read now?
Which files/functions do I skip for now?
```

- [ ] **Step 5: Review the final diff**

Run:

`git diff -- chapters/02_newton_arch/README.md chapters/02_newton_arch/source-walkthrough.md docs/superpowers/specs/2026-04-21-chapter-02-core-pass-rewrite-design.md docs/superpowers/plans/2026-04-21-chapter-02-core-pass-rewrite.md`

Expected: the diff stays tightly focused on beginner clarity rather than widening chapter 02's scope.

- [ ] **Step 6: No commit unless explicitly requested**

Do not create a git commit unless the user explicitly asks for one.
