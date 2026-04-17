# DiffSim Validation

## FD 最小规程

- 扰动步长扫描：`eps in {1e-3, 1e-4, 1e-5, 1e-6, 1e-7}`，避免与时间步 `h` 混用。
- 误差指标：相对误差 `< 1e-3` 视为通过，`1e-3 ~ 1e-2` 视为可疑，`> 1e-2` 视为失败。
- 随机种子必须固定，记录到实验文件头部。

## 执行步骤

1. 固定输入、初值、时间窗口和随机种子。
2. 先跑一次解析梯度，记录梯度向量与标量损失。
3. 对同一输入做多步长中心差分，比较方向导数。
4. 记录最佳步长区间，而不是只报一个数字。

## 记录格式

- `seed`：整数。
- `window_length`：仿真反传窗口长度。
- `metric`：损失定义。
- `fd_error_table`：每个步长对应的相对误差。
- `verdict`：`pass` / `suspicious` / `fail`。
- `vectorized_check`：记录一次 Jacobian-vector product 或多维同测结果，避免只验证单标量方向。

## 何时必须做 FD

- diffsim 章节的第一个例子。
- self experiments 中专门用于 FD validation 的实验。
- 任何新写的梯度相关实验首次落地时。
