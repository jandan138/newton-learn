# Source References

## 统一格式

- 单行：`newton/_src/solvers/mujoco/mujoco.py:L120`
- 行区间：`newton/_src/geometry/collisions.py:L80-L140`
- 函数 + 行区间：`solve_constraints()` at `newton/_src/solvers/mujoco/mujoco.py:L200-L280`

## 禁止项

- 不写“看 mujoco solver 那边”。
- 不写“几何模块里有”。
- 不只写函数名，不给路径。
- 不写“上游最近改过这里”这类没有文件与行号的描述。

## Newton 升级后的维护流程

1. 在 `PROGRESS.md` 记录新的 upstream commit hash。
2. 先检查 `INDEX_by_module.md` 的路径是否仍存在。
3. 再检查四大重心章节里最密集的源码引用是否漂移。
4. 如果函数还在、行号变了，优先保留函数名并回绑行号。
5. 完成 line rebinding 后，更新受影响章节中的 source-ref。

## 月度同步命令

```bash
git -C /shared/smartbot/zhuzihou/dev/newton rev-parse --short HEAD
```

预期：输出一个 7 位以上的 commit hash，例如 `1a230702`。
