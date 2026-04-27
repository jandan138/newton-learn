# Bookwide Chapter QA Polish Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Run a full-book documentation and asset QA pass across Chapters 00-16, then patch any concrete consistency defects before publishing the branch.

**Architecture:** Keep existing chapter content and native imagegen assets stable. Use automated shell/Python checks for path, asset, placeholder, and navigation correctness; only patch docs where the checks or review identify concrete mismatches. Historical plan/spec examples are not treated as live Markdown image references; chapter-facing Markdown and chapter-local assets are the authoritative image-reference scope.

**Tech Stack:** Markdown, chapter-local PNG assets, shell verification, Python standard library, git worktree workflow.

---

## File Structure

Create:

- `docs/superpowers/plans/2026-04-27-bookwide-qa-polish.md`

Modify only if checks find concrete defects:

- `README.md`
- `PROGRESS.md`
- `INDEX_by_module.md`
- `chapters/*/*.md`
- `docs/superpowers/specs/*.md`
- `docs/superpowers/plans/*.md`

Do not modify:

- `chapters/*/assets/*.png`, unless a referenced file is corrupt or missing and must be regenerated in a separate native-imagegen pass.
- Source snapshots or external Newton repository files.

## Task 1: Worktree And Baseline

- [x] Confirm `main` is synchronized with `origin/main`.
- [x] Create isolated worktree `.worktrees/bookwide-qa-polish` on branch `bookwide-qa-polish`.
- [x] Confirm `.worktrees/` is ignored.
- [x] Detect whether the repository has package or test configuration.
- [x] Record that this repository uses custom docs/asset QA commands instead of a project test runner.

Verification command:

```bash
git status --short --branch
rg --files -g 'package.json' -g 'pyproject.toml' -g 'requirements.txt' -g 'Cargo.toml' -g 'go.mod' -g 'Makefile' -g 'justfile' -g 'Taskfile.yml' || true
```

Expected:

- Status shows `bookwide-qa-polish`.
- Config-file search has no output.

## Task 2: Automated Bookwide Checks

- [x] Check every chapter-facing Markdown image reference resolves to an existing file.
- [x] Check every chapter-local PNG asset is referenced by at least one chapter Markdown file, with documented exceptions if any are intentionally unreferenced.
- [x] Check every `.png` file under `chapters/` is valid PNG image data.
- [x] Check all Markdown links to repo-local `.md` files resolve.
- [x] Scan for skeleton markers, placeholder language, and unfinished template fragments.

Verification commands:

```bash
python3 - <<'PY'
from pathlib import Path
import re
root = Path('.')
chapters = sorted([p for p in (root / 'chapters').iterdir() if p.is_dir() and re.match(r'^(0[0-9]|1[0-6])_', p.name)])
missing = []
outside = []
resolved = []
for chapter in chapters:
    for md in sorted(chapter.rglob('*.md')):
        text = md.read_text(encoding='utf-8')
        for raw in re.findall(r'!\[[^\]]*\]\(([^)\s]+)(?:\s+"[^"]*")?\)', text):
            target = re.split(r'[?#]', raw, 1)[0]
            path = (md.parent / target).resolve()
            resolved.append(path)
            if not path.exists():
                missing.append((str(md), raw))
            try:
                path.relative_to((chapter / 'assets').resolve())
            except ValueError:
                outside.append((str(md), raw))
assets = [p.resolve() for chapter in chapters for p in (chapter / 'assets').glob('*.png')]
unreferenced = sorted(set(assets) - {p for p in resolved if p.suffix.lower() == '.png'})
print(f'chapters={len(chapters)} image_refs={len(resolved)} png_assets={len(assets)}')
print(f'missing_refs={len(missing)} unreferenced_png_assets={len(unreferenced)} outside_chapter_local_refs={len(outside)}')
for label, items in [('missing', missing), ('unreferenced', unreferenced), ('outside', outside)]:
    for item in items:
        print(label, item)
if missing or unreferenced or outside:
    raise SystemExit(1)
PY

find chapters -type f -name '*.png' -exec file {} + | rg -v 'PNG image data' || true

python3 - <<'PY'
from pathlib import Path
import re
link_re = re.compile(r'(?<!!)\[[^\]]+\]\(([^)]+)\)')
scope = [Path('README.md'), Path('PROGRESS.md'), Path('INDEX_by_module.md')]
scope += list(Path('chapters').glob('*/*.md'))
scope += list(Path('conventions').glob('*.md'))
scope += list(Path('docs/superpowers/specs').glob('*.md'))
scope += list(Path('docs/superpowers/plans').glob('*.md'))
missing = []
checked = 0
for md in sorted(p for p in scope if p.exists()):
    text = md.read_text(encoding='utf-8')
    in_fence = False
    lines = []
    for line in text.splitlines():
        if line.lstrip().startswith('```'):
            in_fence = not in_fence
            continue
        if not in_fence:
            lines.append(line)
    for raw in link_re.findall('\n'.join(lines)):
        target = raw.strip().split()[0].split('#', 1)[0]
        if not target or '://' in target or raw.strip().startswith('#') or raw.strip().startswith('mailto:') or target.startswith('<'):
            continue
        if target.lower().endswith(('.png', '.jpg', '.jpeg', '.gif', '.svg')):
            continue
        if target.endswith('.md') or target.endswith('/') or '/' in target:
            checked += 1
            path = (md.parent / target).resolve()
            if not path.exists():
                missing.append((str(md), raw))
print(f'checked_links={checked} missing_links={len(missing)}')
for md, raw in missing:
    print(f'{md}: {raw}')
if missing:
    raise SystemExit(1)
PY

rg -n "T[B]D|T[O]DO|本章要回答什[么]问题|读完后应该能独立讲清哪些对[象]|本章最终应有哪些文[件]|二选一先开[写]|\\{\\{" \
  README.md PROGRESS.md INDEX_by_module.md chapters conventions docs/superpowers/plans/2026-04-27-bookwide-qa-polish.md || true

rg -n "优先 SVG|Similar visual language to the chapter-02|regenerate or deterministically|deterministically re-render|deterministically render|deterministic raster re-rendering|deterministic PNG teaching anchors|Tech Stack:\\*\\* Markdown, deterministic" \
  docs/superpowers/specs docs/superpowers/plans/2026-04-25-chapter-05-tutorial-infographics.md docs/superpowers/plans/2026-04-25-chapter-06-tutorial-infographics.md docs/superpowers/plans/2026-04-25-chapter-07-tutorial-infographics.md || true
```

Expected:

- `missing_refs=0`, `unreferenced_png_assets=0`, and `outside_chapter_local_refs=0`.
- PNG type check has no output.
- `missing_links=0`.
- Skeleton scan has no true unfinished chapter/template fragments in the active documentation scope.
- Stale visual-rule scan has no output.

## Task 3: Patch Concrete Defects

- [x] Patch broken links, stale navigation, or stale style references reported by checks or review.
- [x] Keep edits minimal and traceable to a concrete QA finding.
- [x] Avoid rewriting chapter substance unless a defect is directly confirmed.
- [x] Re-run the exact check that exposed each defect.

Concrete findings addressed:

- Chapter structure review found stale root example navigation and mixed baseline metadata.
- Visual-style review found old SVG-first, Chapter 02-style, and deterministic-renderer final-asset instructions that conflicted with the later native-imagegen Chapter 03 / 04 baseline.

Verification command:

```bash
git diff --check
git status --short
```

Expected:

- Diff check has no output.
- Status shows only intended docs changes.

## Task 4: Independent Review

- [x] Run independent image/path asset audit.
- [x] Run independent chapter structure and navigation audit.
- [x] Run independent visual-style consistency audit against the Chapter 03/04 baseline.
- [x] Fix Critical and Important findings.
- [x] Document any Minor findings that are intentionally deferred.

Review acceptance:

- No unresolved Critical or Important findings remain.
- No Minor findings were intentionally deferred in this QA pass.

## Task 5: Final Verification, Commit, Push

- [ ] Run full image-reference and asset checks.
- [ ] Run Markdown local-link check.
- [ ] Run PNG file-type check.
- [ ] Run skeleton-marker scan.
- [ ] Run `git diff --check`.
- [ ] Commit with message `docs: polish bookwide chapter QA`.
- [ ] Push branch `bookwide-qa-polish` to origin.

Final verification commands:

```bash
git status --short --branch
python3 - <<'PY'
from pathlib import Path
import re
root = Path('.')
chapters = sorted([p for p in (root / 'chapters').iterdir() if p.is_dir() and re.match(r'^(0[0-9]|1[0-6])_', p.name)])
missing = []
outside = []
resolved = []
for chapter in chapters:
    for md in sorted(chapter.rglob('*.md')):
        text = md.read_text(encoding='utf-8')
        for raw in re.findall(r'!\[[^\]]*\]\(([^)\s]+)(?:\s+"[^"]*")?\)', text):
            target = re.split(r'[?#]', raw, 1)[0]
            path = (md.parent / target).resolve()
            resolved.append(path)
            if not path.exists():
                missing.append((str(md), raw))
            try:
                path.relative_to((chapter / 'assets').resolve())
            except ValueError:
                outside.append((str(md), raw))
assets = [p.resolve() for chapter in chapters for p in (chapter / 'assets').glob('*.png')]
unreferenced = sorted(set(assets) - {p for p in resolved if p.suffix.lower() == '.png'})
print(f'chapters={len(chapters)} image_refs={len(resolved)} png_assets={len(assets)}')
print(f'missing_refs={len(missing)} unreferenced_png_assets={len(unreferenced)} outside_chapter_local_refs={len(outside)}')
for label, items in [('missing', missing), ('unreferenced', unreferenced), ('outside', outside)]:
    for item in items:
        print(label, item)
if missing or unreferenced or outside:
    raise SystemExit(1)
PY

python3 - <<'PY'
from pathlib import Path
import re
link_re = re.compile(r'(?<!!)\[[^\]]+\]\(([^)]+)\)')
scope = [Path('README.md'), Path('PROGRESS.md'), Path('INDEX_by_module.md')]
scope += list(Path('chapters').glob('*/*.md'))
scope += list(Path('conventions').glob('*.md'))
scope += list(Path('docs/superpowers/specs').glob('*.md'))
scope += list(Path('docs/superpowers/plans').glob('*.md'))
missing = []
checked = 0
for md in sorted(p for p in scope if p.exists()):
    text = md.read_text(encoding='utf-8')
    in_fence = False
    lines = []
    for line in text.splitlines():
        if line.lstrip().startswith('```'):
            in_fence = not in_fence
            continue
        if not in_fence:
            lines.append(line)
    for raw in link_re.findall('\n'.join(lines)):
        target = raw.strip().split()[0].split('#', 1)[0]
        if not target or '://' in target or raw.strip().startswith('#') or raw.strip().startswith('mailto:') or target.startswith('<'):
            continue
        if target.lower().endswith(('.png', '.jpg', '.jpeg', '.gif', '.svg')):
            continue
        if target.endswith('.md') or target.endswith('/') or '/' in target:
            checked += 1
            path = (md.parent / target).resolve()
            if not path.exists():
                missing.append((str(md), raw))
print(f'checked_links={checked} missing_links={len(missing)}')
for md, raw in missing:
    print(f'{md}: {raw}')
if missing:
    raise SystemExit(1)
PY

find chapters -type f -name '*.png' -exec file {} + | rg -v 'PNG image data' || true
rg -n "T[B]D|T[O]DO|本章要回答什[么]问题|读完后应该能独立讲清哪些对[象]|本章最终应有哪些文[件]|二选一先开[写]|\\{\\{" \
  README.md PROGRESS.md INDEX_by_module.md chapters conventions docs/superpowers/plans/2026-04-27-bookwide-qa-polish.md || true
rg -n "优先 SVG|Similar visual language to the chapter-02|regenerate or deterministically|deterministically re-render|deterministically render|deterministic raster re-rendering|deterministic PNG teaching anchors|Tech Stack:\\*\\* Markdown, deterministic" \
  docs/superpowers/specs docs/superpowers/plans/2026-04-25-chapter-05-tutorial-infographics.md docs/superpowers/plans/2026-04-25-chapter-06-tutorial-infographics.md docs/superpowers/plans/2026-04-25-chapter-07-tutorial-infographics.md || true
git diff --check
```

Expected:

- Status shows only intended staged/unstaged changes before commit, then clean after commit.
- Image check reports `missing_refs=0`, `unreferenced_png_assets=0`, and `outside_chapter_local_refs=0`.
- Link check reports `missing_links=0`.
- PNG type check has no output.
- Skeleton scan has no true unfinished chapter/template fragments in the active documentation scope.
- Stale visual-rule scan has no output.
- Diff check has no output.
