# GPU Conventions

## 数据布局

- 默认优先 SoA，除非上游 API 已经固定为 AoS。
- 记录 `wp.array` 的 shape 与 dtype，避免只写“一个数组”。
- 章节中的 kernel 说明必须写清并行维度：`per-body`、`per-contact`、`per-particle` 或 `per-grid-cell`。

## 线程与索引

- `wp.tid()` 的语义要写清：它对应 body、contact、particle 还是 grid cell。
- 若一个 kernel 做多维索引映射，明确写出逻辑坐标到线性索引的变换。

## Atomic 与同步

- 只有真的存在并发写冲突时才用 atomic，并在注释里解释冲突来源。
- 文档里统一标记 atomic 风险：非确定性、吞吐瓶颈、调试难度。
- 遇到 host 读取 device 结果时，写清 host/device sync boundary 与数据流向。

## Device / Stream 记号

- `device` 统一写成 `cpu` 或 `cuda:N` 形式。
- `stream` 单独写明是否为默认 stream；若不是，写出创建与传递边界。
- 任何跨 device 的数据搬运都要注明源 device、目标 device 和同步点。

## Graph 与性能

- 记录是否进入 CUDA graph capture。
- 记录 graph capture 对 dynamic shape 的限制与 caveat。
- 性能讨论统一从 launch overhead、内存访问、atomic 冲突三方面描述。
