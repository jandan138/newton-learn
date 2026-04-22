# Chapter 06 Collision Bridge Image Design

## Goal

Validate the `codex-native-imagegen` workflow by generating one real raster teaching asset and integrating it into `chapters/06_collision/principle.md`.

## Target Slice

- Chapter: `06_collision`
- File: `chapters/06_collision/principle.md`
- Placement: immediately after the minimal bridge chain in section `0. 先看一个球落到地面上`

## Image Content

Generate a single Chinese teaching infographic covering this bridge:

```text
state.body_q + model.shape_transform / shape_type
-> shape 在 world 里的当前位置
-> maybe-list candidate pairs
-> narrow phase contact geometry
-> Contacts
```

The image should communicate five beginner-safe ideas:

1. `body_q` gives the body pose in world.
2. `shape_transform` and `shape_type` determine which shape is attached and how it sits on the body.
3. broad phase only produces a cheap maybe-list.
4. narrow phase turns maybe-pairs into concrete contact geometry.
5. final results are written into `Contacts` as a unified runtime buffer.

## Style

- Chinese infographic / teaching poster
- White background
- Clean card-based layout
- Similar visual language to the chapter-02 learner diagrams: large title, numbered or left-to-right stages, colored blocks, concise labels
- Not photorealistic
- Minimal decorative detail
- Text must stay short and readable

## Integration Rules

- Save the generated raster under `chapters/06_collision/assets/` with a stable ASCII filename.
- Add the image to `principle.md` with a short caption paragraph that states it is a first-pass bridge map, not a complete collision algorithm diagram.
- Do not rewrite the whole section; only add the image and a compact bridging sentence if needed.

## Verification

- Verify the generated file exists and is a raster PNG.
- Verify markdown references the new asset path.
- Verify `git diff --check` is clean for the touched files.
