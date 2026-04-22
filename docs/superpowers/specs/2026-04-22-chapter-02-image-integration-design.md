# Chapter 02 Image Integration Design

## Goal

Integrate all learner-generated images from `tmp/incoming-images/` into chapter 02 so they reduce concrete beginner confusions without expanding chapter 02 into a full joint-parameterization chapter.

## Teaching Constraints

- Keep chapter 02 centered on the current core pass: example entry -> launcher -> concrete module -> `Example.__init__()` -> runtime objects -> `simulate()`.
- Use images to clarify handoff, observation, and beginner traps.
- If a learner image does not fit the core pass cleanly, create a scoped supplement inside chapter 02 rather than dropping the image.
- Defer full joint-frame algebra and articulation parameterization details to later chapters, but allow a labeled optional deep note and a question-driven supplement page inside chapter 02.

## Image Placement

### Main Walkthrough

- Add a source-accurate launcher / `runpy.run_module(...)` timeline SVG to `chapters/02_newton_arch/source-walkthrough.md` in Stage 2, immediately after the `runpy.run_module(...)` excerpt.
- Add a short note below it explaining why `__name__ == "__main__"` becomes true twice.

### Example Observation Sheet

- Add a source-accurate pendulum two-joint chain SVG to `chapters/02_newton_arch/examples.md` in the `basic_pendulum` section after the `Example.__init__()` observation bullets.
- Add the camera / view-angle comparison image to `chapters/02_newton_arch/examples.md` near the `axis` row in the `改这里会怎样` table, followed by a warning that `rot` changes pose presentation while `axis` changes the modeled swing plane.
- Expand the surrounding text so readers can distinguish:
  - `parent=-1` vs `parent=link_0`
  - world anchor vs link-to-link joint
  - `rot` vs `axis`

### Deep Walkthrough

- Add a short optional deep note in `chapters/02_newton_arch/source-walkthrough-deep.md` explaining how to read `parent_xform`, `child_xform`, and equivalent joint parameterizations at a high level.
- Keep the note explicitly labeled as second-pass material, not chapter-02 completion requirements.

### Problem-Driven Supplement

- Add `chapters/02_newton_arch/question-notes.md` as a chapter-02 supplement for learner-generated summary diagrams that are useful but too compressed for the main walkthrough.
- Put the remaining learner images there with one local rule repeated for each figure:
  - what confusion the figure resolves
  - what the figure intentionally compresses or simplifies
  - what the pinned upstream source says exactly
- Use this supplement to cover:
  - the original CLI / `__main__` / `runpy.run_module(...)` summary image
  - the step-by-step `joint` image
  - the joint-frame role image
  - the synchronous-coordinate-change / equivalent-parameterization image

## Text Additions

- `source-walkthrough.md`
  - Explain the two `__main__` moments more explicitly.
  - Add a small reminder that module routing and example execution are distinct stages.
- `examples.md`
  - Add a compact note on why the pendulum has two revolute joints and what each one attaches.
  - Add a warning that `transform.q` and `joint_q` are different meanings of `q`.
  - Clarify that changing `rot` is not the same kind of change as changing `axis`.
- `source-walkthrough-deep.md`
  - Add a short optional note introducing joint frame interpretation and equivalent parameterizations.
  - Link to `question-notes.md` for the image-driven version of those confusions.
- `question-notes.md`
  - Add one section per learner image.
  - For source-compressed diagrams, add explicit correction notes instead of pretending the figure is a line-by-line source mirror.
- `README.md`
  - Add the new supplement file to file responsibilities and reading order.

## Asset Handling

- Keep only source-accurate visuals inline.
- Keep source-accurate visuals in the main walkthrough files.
- For learner images that collapse layers or disagree with the pinned upstream source, place them in `question-notes.md` with explicit "teaching summary vs exact source" correction notes.
- Reference all inlined assets from `chapters/02_newton_arch/assets/`, not from the temporary `tmp/incoming-images/` directory.

## Out Of Scope

- Full derivation of joint-frame parameterization.
- Full explanation of quaternion equivalence classes.
- Turning chapter 02 into a replacement for `05_rigid_articulation`.
