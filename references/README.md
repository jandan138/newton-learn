# References

## 使用规则

1. 每篇论文先在 `papers/<chapter>.md` 写至少 1 句摘要，再决定是否精读；随着仓库内容成熟，优先扩展到 2-3 句。
2. 只有确认元数据后，才把 BibTeX 条目写进 `papers.bib`。
3. `[MUST]` 必读全文；`[RECOMMENDED]` 至少读摘要；`[OPTIONAL]` 只在卡住时回头查。
4. 章节 frontmatter 里的 `paper_keys` 可以先指向已预留的 key；等后续核实完成后，再把对应 BibTeX 条目补进 `papers.bib`。

## 文件分工

- `papers.bib`: 只存经过核实的 BibTeX 条目。
- `papers/<chapter>.md`: 按章节收资料，不按学科大类混放。
- `external-links.md`: 官方文档、仓库、教程、工具页。

## 当前已建章节

- `01_warp_gpu.md`：服务 `01_warp_basics`，集中放 Warp / GPU / MuJoCo Warp 的入口资料。
- `05_rigid_articulation.md`
- `07_constraints_contacts_math.md`
- `08_rigid_solvers.md`
- `09_variational_solvers.md`
- `11_mpm.md`
- `13_diffsim.md`
