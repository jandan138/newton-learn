# 01 Warp / GPU

- `warp-paper` `[MUST]`
  - 对应章节：`01_warp_basics`
  - 摘要：建立 Warp 的 kernel / array / launch / autodiff 心智模型。
  - 获取方式：官方论文页或官方仓库文档。

- `mujoco-warp-paper` `[MUST]`
  - 对应章节：`01_warp_basics`, `08_rigid_solvers`
  - 摘要：说明为什么 MuJoCo 约束求解能迁移到 Warp/GPU。
  - 获取方式：官方论文页或项目仓库。

- `mls-mpm-gpu` `[RECOMMENDED]`
  - 对应章节：`01_warp_basics`, `11_mpm`
  - 摘要：补 GPU 粒子-网格 scatter/gather 的工程感。
  - 获取方式：论文页或作者项目页。

- `cuda-graphs-nvidia` `[RECOMMENDED]`
  - 对应章节：`01_warp_basics`
  - 摘要：帮助理解图捕获为何能减少 launch overhead。
  - 获取方式：NVIDIA 技术博客。

- `kernel-fusion-nvidia` `[RECOMMENDED]`
  - 对应章节：`01_warp_basics`
  - 摘要：补 kernel fusion 对 launch 开销和内存往返的影响。
  - 获取方式：NVIDIA 技术博客。
