# Chapter 13 DiffSim Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a full teaching-first chapter 13 package that teaches DiffSim as a `verify before optimize` loop rather than a demo catalog or autodiff theory dump.

**Architecture:** Center chapter 13 on one narrow first-pass spine: `forward sim -> loss -> backward -> FD validation -> update loop`. Use `example_diffsim_ball.py` as the smallest end-to-end anchor, then extend the same logic to `example_diffsim_spring_cage.py` and `example_diffsim_soft_body.py` through the comparison axis of where the optimized parameter lives: `State`, `Model`, or an external parameter buffer. Keep `drone / cloth / bear` explicitly outside the first-pass mainline.

**Tech Stack:** Markdown docs, existing chapter package conventions, upstream Newton diffsim examples, `conventions/diffsim-validation.md`, multi-agent review, `git diff --check`.

---

## File Map

- Modify:
  - `chapters/13_diffsim/README.md`
- Create:
  - `chapters/13_diffsim/principle.md`
  - `chapters/13_diffsim/examples.md`
  - `chapters/13_diffsim/source-walkthrough.md`
  - `chapters/13_diffsim/pitfalls.md`
  - `chapters/13_diffsim/exercises.md`
  - `docs/superpowers/specs/2026-04-22-chapter-13-diffsim-design.md`
  - `docs/superpowers/plans/2026-04-22-chapter-13-diffsim.md`

## Task 1: Rewrite Chapter 13 README Around `Verify Before Optimize`

**Files:**
- Modify: `chapters/13_diffsim/README.md`

- [ ] **Step 1: Replace the skeleton with the chapter’s main question**

The README should open by answering:

```text
一次 rollout 没打中目标时，
我到底该改哪个上游参数，梯度会出现在哪里，
又怎么知道这个梯度可信？
```

It should explicitly position chapter 13 as a meta-capability chapter, not a new solver-family chapter.

- [ ] **Step 2: Establish the first-pass spine and comparison axis**

The README should make these two chapter-level structures explicit:

```text
forward sim -> scalar loss -> wp.Tape backward -> gradient -> FD validation -> update loop
```

and:

```text
state parameter
-> model parameter
-> external parameter buffer
```

- [ ] **Step 3: Assign example roles and defer advanced branches**

The README should make these roles explicit:

```text
diffsim_ball -> first-pass main anchor + mandatory FD
diffsim_spring_cage -> model-parameter branch
diffsim_soft_body -> external-parameter-buffer extension
diffsim_drone / cloth / bear -> advanced branches
```

- [ ] **Step 4: Put FD validation into the completion gate**

The completion section must explicitly require:

```text
I can explain why `tape.backward()` is not enough by itself.
I can explain the FD step-size sweep and trust window.
I can point to one example where the gradient is verified before optimization.
```

## Task 2: Write Chapter 13 Principle Doc

**Files:**
- Create: `chapters/13_diffsim/principle.md`

- [ ] **Step 1: Explain DiffSim as a backward-capability layer on top of forward simulation**

The doc should first answer what changes when a normal forward sim becomes differentiable:

```text
requires_grad=True
-> tracked model/state/buffers
-> wp.Tape records forward kernels
-> backward sends gradients back to upstream parameters
```

- [ ] **Step 2: Make `verify before optimize` the chapter’s governing rule**

The principle doc must elevate `conventions/diffsim-validation.md` into a chapter-level rule, not an appendix. It should explicitly explain:

```text
backward gives a gradient candidate
FD decides whether that gradient is trustworthy enough to optimize with
```

- [ ] **Step 3: Explain the three parameter-placement patterns**

The principle should define in beginner-safe language:

```text
state parameter
model parameter
external parameter buffer
```

and map them to `ball`, `spring_cage`, and `soft_body` respectively.

## Task 3: Write Chapter 13 Examples Doc

**Files:**
- Create: `chapters/13_diffsim/examples.md`

- [ ] **Step 1: Give each upstream diffsim example one teaching job**

Assign unique roles to:

```text
example_diffsim_ball.py
example_diffsim_spring_cage.py
example_diffsim_soft_body.py
example_diffsim_drone.py
example_diffsim_cloth.py
example_diffsim_bear.py
```

Do not let this flatten into a demo list.

- [ ] **Step 2: Separate first-pass, second-pass, and advanced examples**

The page should clearly mark:

```text
first pass: ball
second pass: spring_cage, soft_body
advanced: drone, cloth, bear
```

- [ ] **Step 3: Convert example use into observation tasks**

At minimum, `ball`, `spring_cage`, and `soft_body` should each answer:

```text
what parameter is being optimized?
where does the loss live?
where does the gradient land?
how is the update applied?
what must be validated before trusting the optimization?
```

## Task 4: Write Chapter 13 Main Walkthrough

**Files:**
- Create: `chapters/13_diffsim/source-walkthrough.md`

- [ ] **Step 1: Use the repo’s current walkthrough structure**

The file should include:

```text
What This Walkthrough Follows
One-Screen Chapter Map
Beginner Path
Main Walkthrough
Object Ledger
Stop Here
Go Deeper
```

- [ ] **Step 2: Keep one narrow walkthrough spine**

The main walkthrough should feel like:

```text
ball forward rollout
-> final-state loss
-> wp.Tape backward
-> gradient lands on initial velocity
-> FD checks trust
-> update loop applies gradient
-> spring_cage / soft_body extend the same loop to other parameter placements
```

- [ ] **Step 3: Make `ball` the exact main anchor**

The walkthrough should explicitly cover these anchors in `example_diffsim_ball.py`:

```text
requires_grad=True model finalization
wp.Tape() capture
forward() rollout
loss_kernel on final state
tape.backward(loss)
gradient descent step on initial velocity
check_grad() numeric vs analytic comparison
```

- [ ] **Step 4: Add a local FD-validation section tied to repo convention**

The walkthrough should explicitly connect `ball`’s `check_grad()` to `conventions/diffsim-validation.md`, and explain the difference between:

```text
example-local sanity check
repo-level FD protocol with sweep / verdict / trust window
```

- [ ] **Step 5: Add selective extension stages for `spring_cage` and `soft_body`**

Use these only to clarify parameter placement:

```text
spring_cage -> gradient lands on model.spring_rest_length
soft_body -> gradient lands on external material_params buffer before remapping into model.tet_materials
```

- [ ] **Step 6: Explicitly defer `drone` from first pass**

The walkthrough should say plainly that `drone` is a control/MPC branch and not part of the first-pass trust-establishment path.

## Task 5: Write Pitfalls And Exercises

**Files:**
- Create:
  - `chapters/13_diffsim/pitfalls.md`
  - `chapters/13_diffsim/exercises.md`

- [ ] **Step 1: Write chapter-specific pitfalls**

Include at least these pitfalls:

```text
"backward() ran" != "gradient is trustworthy"
short-horizon contact passing once != contact gradients are generally safe
not separating state/model/external parameter locations
using one FD epsilon and overclaiming correctness
```

- [ ] **Step 2: Write beginner-safe exercises**

At minimum include exercises that ask the reader to:

```text
identify where the optimized parameter lives in ball / spring_cage / soft_body
write an FD error table for one diffsim example
explain why drone is deferred from first pass
```

## Task 6: Review And Verification

**Files:**
- Verify the chapter 13 package and the supporting spec/plan files.

- [ ] **Step 1: Run file-level diff checks**

Run:

`git diff --check -- chapters/13_diffsim/README.md chapters/13_diffsim/principle.md chapters/13_diffsim/examples.md chapters/13_diffsim/source-walkthrough.md chapters/13_diffsim/pitfalls.md chapters/13_diffsim/exercises.md docs/superpowers/specs/2026-04-22-chapter-13-diffsim-design.md docs/superpowers/plans/2026-04-22-chapter-13-diffsim.md`

Expected: no output.

- [ ] **Step 2: Run placeholder scan**

Run:

`rg -n "TODO|TBD" chapters/13_diffsim/README.md chapters/13_diffsim/principle.md chapters/13_diffsim/examples.md chapters/13_diffsim/source-walkthrough.md chapters/13_diffsim/pitfalls.md chapters/13_diffsim/exercises.md docs/superpowers/specs/2026-04-22-chapter-13-diffsim-design.md docs/superpowers/plans/2026-04-22-chapter-13-diffsim.md`

Expected: no output.

- [ ] **Step 3: Re-read for first-pass discipline**

Confirm the chapter package still keeps this shape:

```text
mainline: ball -> FD trust -> update loop
extensions: spring_cage, soft_body
defer: drone, cloth, bear
```

- [ ] **Step 4: Re-read for continuity with 12 -> 13**

Confirm the handoff now reads like:

```text
12: state exists, read and steer it
13: if a rollout misses, trace gradient back to upstream parameters and verify it before optimizing
```

- [ ] **Step 5: Review the final diff**

Run:

`git diff -- chapters/13_diffsim/README.md chapters/13_diffsim/principle.md chapters/13_diffsim/examples.md chapters/13_diffsim/source-walkthrough.md chapters/13_diffsim/pitfalls.md chapters/13_diffsim/exercises.md docs/superpowers/specs/2026-04-22-chapter-13-diffsim-design.md docs/superpowers/plans/2026-04-22-chapter-13-diffsim.md`

Expected: the diff stays focused on a trust-first DiffSim chapter rather than flattening into a diffsim demo catalog or an autodiff theory chapter.

- [ ] **Step 6: No commit unless explicitly requested**

Do not create a git commit unless the user explicitly asks for one.
