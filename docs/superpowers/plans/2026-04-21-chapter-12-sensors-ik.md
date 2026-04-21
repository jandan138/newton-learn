# Chapter 12 Sensors And IK Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the first full teaching-first chapter-12 docs package around a single `observe and steer` spine that unifies sensors and inverse kinematics.

**Architecture:** Keep chapter 12 centered on one shared FK/state backbone. Use `SensorIMU` and `IK Franka` as the cleanest read-side and write-side anchors, and keep `SensorContact`, `IK H1`, `IK custom`, and `tiled camera` as scoped branches rather than equal-weight catalogs.

**Tech Stack:** Markdown docs, git worktree, upstream Newton source anchors, multi-agent review, `git diff --check`.

---

## File Map

- Modify:
  - `chapters/12_sensors_ik/README.md`
- Create:
  - `chapters/12_sensors_ik/principle.md`
  - `chapters/12_sensors_ik/examples.md`
  - `chapters/12_sensors_ik/source-walkthrough.md`
  - `docs/superpowers/specs/2026-04-21-chapter-12-sensors-ik-design.md`
  - `docs/superpowers/plans/2026-04-21-chapter-12-sensors-ik.md`

## Task 1: Rewrite Chapter 12 README Around `Observe And Steer`

**Files:**
- Modify: `chapters/12_sensors_ik/README.md`

- [ ] **Step 1: Replace the skeleton with a chapter-level teaching promise**

The README should establish that chapter 12 is not two separate mini-chapters. It should answer:

```text
状态既然已经存在，我怎样读它？又怎样朝目标去改它？
```

- [ ] **Step 2: Establish example roles and reading order**

The README should clearly separate core anchors from advanced branches, with `example_sensor_imu.py` / `example_ik_franka.py` as the cleanest main anchors.

## Task 2: Write Chapter 12 Principle Doc

**Files:**
- Create: `chapters/12_sensors_ik/principle.md`

- [ ] **Step 1: Explain the read-side vs write-side symmetry**

The doc should present:

```text
sensors = read-side adapters
IK = write-side adapters
```

- [ ] **Step 2: Make the shared FK/state backbone explicit**

The principle should clearly explain the shared middle:

```text
joint_q -> eval_fk -> state.body_q / state.body_qd / state.body_qdd / contacts
```

- [ ] **Step 3: Make update-order rules explicit**

Especially cover why some sensors need `solver.update_contacts(...)` before `sensor.update(...)`, and why `SensorIMU` needs `body_qdd` allocated.

## Task 3: Write Chapter 12 Examples Doc

**Files:**
- Create: `chapters/12_sensors_ik/examples.md`

- [ ] **Step 1: Give each example one explicit teaching job**

Do not let `sensors` or `ik` examples flatten into a feature catalog. Assign unique jobs to:

```text
sensor_imu
sensor_contact
sensor_tiled_camera
ik_franka
ik_h1
ik_custom
ik_cube_stacking
```

- [ ] **Step 2: Keep advanced examples out of the first-pass path**

`sensor_tiled_camera`, `ik_custom`, and `ik_cube_stacking` should be clearly marked as second-pass / advanced branches.

## Task 4: Write Chapter 12 Main Walkthrough

**Files:**
- Create: `chapters/12_sensors_ik/source-walkthrough.md`

- [ ] **Step 1: Use the repo’s main walkthrough structure**

The file should contain:

```text
What This Walkthrough Follows
One-Screen Chapter Map
Beginner Path
Main Walkthrough
Object Ledger
Stop Here
Go Deeper
```

- [ ] **Step 2: Keep one chapter spine, not two disconnected halves**

The main walkthrough should feel like:

```text
FK-derived state exists
-> sensors read from it
-> IK writes targets back toward it
-> both depend on the same articulated-state backbone
```

- [ ] **Step 3: Anchor on the cleanest core examples**

Use `example_sensor_imu.py` and `example_ik_franka.py` as the mainline, with `example_sensor_contact.py` as the side branch that teaches update timing and `Contacts` dependence.

## Task 5: Final Review And Verification

**Files:**
- Verify all modified and created chapter docs plus the supporting spec/plan files.

- [ ] **Step 1: Run file-level diff checks**

Run:

`git diff --check -- chapters/12_sensors_ik/README.md chapters/12_sensors_ik/principle.md chapters/12_sensors_ik/examples.md chapters/12_sensors_ik/source-walkthrough.md docs/superpowers/specs/2026-04-21-chapter-12-sensors-ik-design.md docs/superpowers/plans/2026-04-21-chapter-12-sensors-ik.md`

Expected: no output.

- [ ] **Step 2: Run placeholder scan on chapter 12**

Check that the old skeleton placeholders are gone.

- [ ] **Step 3: Re-read for continuity across 11 -> 12**

Confirm the handoff now reads like:

```text
11: how state flows through a solver step
12: once state exists, how it is observed and steered
```

- [ ] **Step 4: Review the final diff**

Run:

`git diff -- chapters/12_sensors_ik/README.md chapters/12_sensors_ik/principle.md chapters/12_sensors_ik/examples.md chapters/12_sensors_ik/source-walkthrough.md docs/superpowers/specs/2026-04-21-chapter-12-sensors-ik-design.md docs/superpowers/plans/2026-04-21-chapter-12-sensors-ik.md`

Expected: the diff stays focused on a teaching-first `observe and steer` chapter rather than splitting into a sensor catalog and an IK catalog.
