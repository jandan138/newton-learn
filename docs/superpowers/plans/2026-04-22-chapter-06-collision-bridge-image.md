# Chapter 06 Collision Bridge Image Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Generate one real raster infographic for the chapter-06 collision bridge and integrate it into the tutorial.

**Architecture:** Use `codex-native-imagegen` for the raster generation path, save the output under chapter-06 assets, and insert it into `principle.md` right after the section-0 bridge chain so the image strengthens the existing explanation without expanding scope.

**Tech Stack:** Markdown, PNG asset, Codex CLI built-in image generation, git diff verification

---

## File Map

- Create: `chapters/06_collision/assets/06_collision_bridge_map.png`
- Modify: `chapters/06_collision/principle.md`
- Create: `docs/superpowers/specs/2026-04-22-chapter-06-collision-bridge-image-design.md`
- Create: `docs/superpowers/plans/2026-04-22-chapter-06-collision-bridge-image.md`

### Task 1: Generate the raster asset

**Files:**
- Create: `chapters/06_collision/assets/06_collision_bridge_map.png`

- [ ] **Step 1: Verify the chapter-06 assets directory exists**

Run: `ls /home/zhuzihou/dev/newton-learn/chapters/06_collision`
Expected: the chapter directory exists and can hold an `assets/` child directory.

- [ ] **Step 2: Run `codex-native-imagegen` for one infographic**

Generate one Chinese infographic that explains:

```text
body_q + shape_transform / shape_type -> shape world pose -> candidate pairs -> narrow phase -> Contacts
```

Save the final PNG to `chapters/06_collision/assets/06_collision_bridge_map.png`.

### Task 2: Integrate the image into chapter 06

**Files:**
- Modify: `chapters/06_collision/principle.md`

- [ ] **Step 1: Insert the image after the section-0 minimal bridge chain**

Add the new image immediately after the existing five-line bridge chain and follow it with one short paragraph clarifying that the figure is a first-pass runtime bridge map, not a full collision-algorithm taxonomy.

### Task 3: Verify output

**Files:**
- Modify: `chapters/06_collision/principle.md`
- Create: `chapters/06_collision/assets/06_collision_bridge_map.png`

- [ ] **Step 1: Verify the image file**

Run: `file /home/zhuzihou/dev/newton-learn/chapters/06_collision/assets/06_collision_bridge_map.png && ls -lh /home/zhuzihou/dev/newton-learn/chapters/06_collision/assets/06_collision_bridge_map.png`
Expected: PNG image data and a non-zero file size.

- [ ] **Step 2: Verify doc diff hygiene**

Run: `git diff --check -- chapters/06_collision/principle.md docs/superpowers/specs/2026-04-22-chapter-06-collision-bridge-image-design.md docs/superpowers/plans/2026-04-22-chapter-06-collision-bridge-image.md`
Expected: no output.
