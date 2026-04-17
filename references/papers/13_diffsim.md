# 13 DiffSim

- `difftaichi-paper` `[MUST]`
  - 对应章节：`13_diffsim`
  - 摘要：用来建立端到端可微物理的经典直觉。
  - 获取方式：优先查 DiffTaichi 论文的 arXiv / official paper page；再看项目仓库 README 补运行背景。

- `brax-paper` `[MUST]`
  - 对应章节：`13_diffsim`
  - 摘要：对比可微刚体系统如何处理可并行与可优化。
  - 获取方式：优先查 Brax 论文的 arXiv / official paper page；再看官方 docs 或仓库 README 补工程语境。

- `warp-adjoint-paper` `[MUST]`
  - 对应章节：`13_diffsim`
  - 摘要：Warp tape 与 kernel replay 的主参考。
  - 获取方式：优先查 Warp adjoint 官方论文页；再对照 NVIDIA Warp 官方 docs 或仓库文档。

- `ift-survey` `[MUST]`
  - 对应章节：`13_diffsim`
  - 摘要：隐式求解器反向传播时需要的隐函数定理背景。
  - 获取方式：优先查相关 IFT 综述的 arXiv / publisher page；若入口分散，先从被引用版本整理正式题名。

- `contact-randomized-smoothing` `[MUST]`
  - 对应章节：`13_diffsim`
  - 摘要：补不可微接触的平滑与估计策略。
  - 获取方式：优先查论文 arXiv / publisher page；再看作者项目页或补充材料确认实验设定。

- `shac-paper` `[MUST]`
  - 对应章节：`13_diffsim`
  - 摘要：连接 policy gradient 与 differentiable simulation。
  - 获取方式：优先查 SHAC 论文的 arXiv / official paper page；再看项目仓库 README 或实验页面补背景。

- `plasticinelab-paper` `[RECOMMENDED]`
  - 对应章节：`13_diffsim`, `11_mpm`
  - 摘要：补可微 MPM 和 system ID 的案例感。
  - 获取方式：优先查 PlasticineLab 论文的 arXiv / official paper page；再看项目仓库 README 理解可复现实验入口。

- `system-id-via-diffsim` `[RECOMMENDED]`
  - 对应章节：`13_diffsim`
  - 摘要：给 system identification 一个明确入口，避免只停留在 toy gradient demo。
  - 获取方式：优先查 system identification 相关论文的 arXiv / publisher page；再看项目页或仓库 README 找复现实验入口。
