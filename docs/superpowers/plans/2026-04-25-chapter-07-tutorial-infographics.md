# Chapter 07 Tutorial Infographics Implementation Plan

> **Status:** Superseded for future execution by `docs/superpowers/plans/2026-04-25-chapters-05-07-native-imagegen-redo.md`.

This file is retained only as historical inventory for the first Chapter 07 image integration pass. Do not use it as an active implementation plan. The original local-renderer workflow has been retired because Chapters 05-07 were later regenerated with native imagegen PNGs in the Chapter 03 / 04 dense Chinese tutorial-handout style.

For any future Chapter 07 image repair:

1. Use `docs/superpowers/specs/chapter-visual-style-guide.md`.
2. Use the Chapter 03 / 04 tutorial-handout style baseline.
3. Use native imagegen raster PNGs.
4. Keep the existing stable `chapters/07_constraints_contacts_math/assets/07_*.png` filenames unless a filename itself is wrong.
5. Do not use local Pillow/canvas/SVG/HTML rendering as final tutorial assets.

## Historical Scope

The completed Chapter 07 reader-facing files were:

- `chapters/07_constraints_contacts_math/README.md`
- `chapters/07_constraints_contacts_math/principle.md`
- `chapters/07_constraints_contacts_math/source-walkthrough.md`
- `chapters/07_constraints_contacts_math/examples.md`

`source-walkthrough-deep.md` stayed intentionally out of scope.

## Historical Asset Inventory

The final Chapter 07 inventory is 22 chapter-local PNG assets:

```text
07_readme_contact_math_spine.png
07_readme_file_roles_reading_order.png
07_readme_completion_scope_prereq.png
07_contact_math_bridge_map.png
07_principle_contact_to_three_rows.png
07_principle_jacobian_motion_map.png
07_principle_lever_arm_angular_coupling.png
07_principle_delassus_effective_mass.png
07_principle_box_ground_multi_contacts.png
07_principle_solver_facing_next_chapter.png
07_walkthrough_pipeline_overview.png
07_walkthrough_beginner_path.png
07_walkthrough_stage1_contacts_geometry_handoff.png
07_walkthrough_stage2_solver_facing_contact_block.png
07_walkthrough_stage3_contact_rows_three_directions.png
07_walkthrough_stage4_jacobian_frame_lever_arm.png
07_walkthrough_stage5_delassus_row_space_response.png
07_walkthrough_object_ledger_stop_here.png
07_examples_overview_observation_tasks.png
07_examples_sphere_ground_one_contact_three_rows.png
07_examples_box_ground_multiple_contacts_lever_arm.png
07_examples_self_check_handoff.png
```

## Historical Teaching Spine

```text
chapter 06 Contacts
-> solver-facing contact block
-> contact frame
-> normal + tangent rows
-> Jacobian velocity map
-> lever-arm angular coupling
-> Delassus / effective mass
-> chapter 08 solver consumption
```

## Current Verification Commands

Use these checks if Chapter 07 is touched again:

```bash
python3 - <<'PY'
from pathlib import Path
import re, sys
chapter = Path('chapters/07_constraints_contacts_math')
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
assets = {p.resolve() for p in (chapter / 'assets').glob('07_*.png')}
unreferenced = sorted(assets - refs)
print(f'refs={len(refs)} assets={len(assets)} missing={len(missing)} unreferenced={len(unreferenced)}')
for item in missing:
    print('missing', item)
for path in unreferenced:
    print('unreferenced', path)
if missing or unreferenced:
    sys.exit(1)
PY

find chapters/07_constraints_contacts_math/assets -maxdepth 1 -type f -name '07_*.png' -exec file {} + | rg -v 'PNG image data' || true
git diff --check -- chapters/07_constraints_contacts_math docs/superpowers/specs/2026-04-25-chapter-07-tutorial-infographics-design.md
```

Expected:

- `refs=22 assets=22 missing=0 unreferenced=0`
- PNG type check has no output.
- Diff check has no output.
