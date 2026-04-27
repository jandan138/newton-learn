# Chapter 06 Tutorial Infographics Implementation Plan

> **Status:** Superseded for future execution by `docs/superpowers/plans/2026-04-25-chapters-05-07-native-imagegen-redo.md`.

This file is retained only as historical inventory for the first Chapter 06 image integration pass. Do not use it as an active implementation plan. The original local-renderer workflow has been retired because Chapters 05-07 were later regenerated with native imagegen PNGs in the Chapter 03 / 04 dense Chinese tutorial-handout style.

For any future Chapter 06 image repair:

1. Use `docs/superpowers/specs/chapter-visual-style-guide.md`.
2. Use the Chapter 03 / 04 tutorial-handout style baseline.
3. Use native imagegen raster PNGs.
4. Keep the existing stable `chapters/06_collision/assets/06_*.png` filenames unless a filename itself is wrong.
5. Do not use local Pillow/canvas/SVG/HTML rendering as final tutorial assets.

## Historical Scope

The completed Chapter 06 reader-facing files were:

- `chapters/06_collision/README.md`
- `chapters/06_collision/principle.md`
- `chapters/06_collision/source-walkthrough.md`
- `chapters/06_collision/examples.md`

`source-walkthrough-deep.md` stayed intentionally out of scope.

## Historical Asset Inventory

The final Chapter 06 inventory is 22 chapter-local PNG assets:

```text
06_collision_bridge_map.png
06_readme_collision_spine.png
06_readme_file_roles_reading_order.png
06_readme_completion_scope_prereq.png
06_principle_shape_not_body_boundary.png
06_principle_broad_phase_maybe_list.png
06_principle_narrow_phase_contact_geometry.png
06_principle_shape_type_routing.png
06_principle_contacts_handoff_buffer.png
06_principle_next_chapters_map.png
06_walkthrough_pipeline_overview.png
06_walkthrough_beginner_path.png
06_walkthrough_stage1_model_state_shape_data.png
06_walkthrough_stage2_compute_shape_aabbs.png
06_walkthrough_stage3_broad_phase_candidate_pairs.png
06_walkthrough_stage4_narrow_phase_contactdata.png
06_walkthrough_stage5_write_contact_contacts.png
06_walkthrough_object_ledger_stop_here.png
06_examples_overview_observation_tasks.png
06_examples_basic_shapes_sphere_ground.png
06_examples_pyramid_contact_set_growth.png
06_examples_contact_fields_watchlist.png
```

## Historical Teaching Spine

```text
05 body/world state + 04 shape metadata
-> state.body_q + model.shape_transform / shape_type / gap / margin
-> shape world pose and expanded AABB
-> broad phase candidate shape pairs
-> narrow phase routing by shape_type
-> ContactData as unified contact geometry record
-> write_contact(...)
-> Contacts
-> onward routes to 07 contact math and 08 solver consumption
```

## Current Verification Commands

Use these checks if Chapter 06 is touched again:

```bash
python3 - <<'PY'
from pathlib import Path
import re, sys
chapter = Path('chapters/06_collision')
refs = set()
missing = []
for md in sorted(chapter.glob('*.md')):
    if md.name == 'source-walkthrough-deep.md':
        continue
    for raw in re.findall(r'!\[[^\]]*\]\(([^)]+)\)', md.read_text(encoding='utf-8')):
        target = raw.split('#', 1)[0].strip()
        path = (md.parent / target).resolve()
        refs.add(path)
        if not path.exists():
            missing.append((str(md), raw))
assets = {p.resolve() for p in (chapter / 'assets').glob('06_*.png')}
unreferenced = sorted(assets - refs)
print(f'refs={len(refs)} assets={len(assets)} missing={len(missing)} unreferenced={len(unreferenced)}')
for item in missing:
    print('missing', item)
for path in unreferenced:
    print('unreferenced', path)
if missing or unreferenced:
    sys.exit(1)
PY

find chapters/06_collision/assets -maxdepth 1 -type f -name '06_*.png' -exec file {} + | rg -v 'PNG image data' || true
git diff --check -- chapters/06_collision docs/superpowers/specs/2026-04-25-chapter-06-tutorial-infographics-design.md
```

Expected:

- `refs=22 assets=22 missing=0 unreferenced=0`
- PNG type check has no output.
- Diff check has no output.
