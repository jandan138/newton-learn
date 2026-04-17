# 01 Warp / GPU

- `warp-paper` `[MUST]`
  - 对应章节：`01_warp_basics`
  - 摘要：建立 Warp 的 kernel / array / launch / autodiff 心智模型。
  - 获取方式：优先查 NVIDIA Warp 官方论文页；若论文页信息不全，再回到官方仓库 README / docs 交叉确认。

- `mujoco-warp-paper` `[MUST]`
  - 对应章节：`01_warp_basics`, `08_rigid_solvers`
  - 摘要：说明为什么 MuJoCo 约束求解能迁移到 Warp/GPU。
  - 获取方式：优先查 MuJoCo Warp 官方论文页或 arXiv；再用项目仓库 README 确认实现背景。

- `mls-mpm-gpu` `[RECOMMENDED]`
  - 对应章节：`01_warp_basics`, `11_mpm`
  - 摘要：补 GPU 粒子-网格 scatter/gather 的工程感。
  - 获取方式：优先查论文的 arXiv / publisher page；再看作者项目页或项目仓库补实现细节。

- `cuda-graphs-nvidia` `[RECOMMENDED]`
  - 对应章节：`01_warp_basics`
  - 摘要：帮助理解图捕获为何能减少 launch overhead。
  - 获取方式：优先查 NVIDIA 官方 CUDA Graphs 技术博客；必要时回看 CUDA 官方文档。

- `kernel-fusion-nvidia` `[RECOMMENDED]`
  - 对应章节：`01_warp_basics`
  - 摘要：补 kernel fusion 对 launch 开销和内存往返的影响。
  - 获取方式：优先查 NVIDIA 官方技术博客或性能指南；再用 CUDA 文档确认术语和上下文。
