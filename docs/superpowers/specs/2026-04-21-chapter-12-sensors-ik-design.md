# Chapter 12 Sensors And IK Design

- **日期**: 2026-04-21
- **仓库**: `/home/zhuzihou/dev/newton-learn`
- **范围**:
  - `chapters/12_sensors_ik/README.md`
  - `chapters/12_sensors_ik/principle.md`
  - `chapters/12_sensors_ik/examples.md`
  - `chapters/12_sensors_ik/source-walkthrough.md`
- **目标**: 把 chapter 12 建成 teaching-first 的 `observe and steer` 章节，让 sensors 和 IK 不再像两个硬拼的主题，而是共享同一条 FK / state backbone。

---

## 1. 这一章真正要回答什么

chapter 10 和 chapter 11 已经把前两层问题讲顺了：

- chapter 10：Newton 到底在模拟什么对象家族。
- chapter 11：这些对象怎样在一次 solver step 里流经数据通路。

chapter 12 最自然的下一问是：

```text
状态既然已经存在，
那我怎样读它？又怎样朝着目标去改它？
```

这就是 sensors 和 IK 在本章里的统一角色：

- sensors = **read-side adapters**，把 state / contacts / rendered scene 变成观测量
- IK = **write-side adapters**，把 task-space 目标变成 joint-space 解

所以这章不该写成 `sensors chapter + IK chapter`，而应该写成：

```text
observe and steer
```

---

## 2. 推荐的 teaching-first spine

### 2.1 共享中轴

这章最重要的共享中轴是：

```text
joint_q
-> eval_fk
-> state.body_q / state.body_qd / state.body_qdd / contacts
```

然后两边分叉：

- sensors 从这条中轴往外读
- IK 从目标往回写到 `joint_q`

更准确地说：

```text
Sensors read the world
IK steers the body toward a goal
Both depend on the same articulated state backbone
```

### 2.2 章节主线

推荐主线是：

```text
simulated worlds need eyes and hands
-> sensors read from state / contacts / sensor frames
-> IK writes from task-space target back to joint-space variables
-> both depend on FK-derived body state and update order
```

这条线最重要的价值是：

1. 避免 sensors 变成 API list。
2. 避免 IK 变成 optimizer 参数介绍。
3. 让读者记住它们共享的 prerequisite 是 `eval_fk` 和 body state，而不是把两者当成两个孤立系统。

---

## 3. 推荐例子分工

### 3.1 主例子

- `newton/examples/ik/example_ik_franka.py`
  - chapter 12 的 main walkthrough anchor
  - 最适合讲清：`IKObjectivePosition + IKObjectiveRotation + IKObjectiveJointLimit -> IKSolver.step -> eval_fk`
- `newton/examples/sensors/example_sensor_imu.py`
  - 最适合讲清：state-based sensor 怎样从 FK 后的 body state 读取观测

### 3.2 第二层分支

- `newton/examples/sensors/example_sensor_contact.py`
  - 它的 job 是讲清：有些 sensor 不是只读 body state，而是读 `Contacts` 这种 side ledger，所以 update order 很重要
- `newton/examples/ik/example_ik_h1.py`
  - 它的 job 是讲清：同一套 IK API 可以扩展到 multi-end-effector，而不是只会解单 TCP

### 3.3 高级分支

- `newton/examples/sensors/example_sensor_tiled_camera.py`
  - job: rendered perception / high-bandwidth sensor branch
  - 不进 first-pass mainline
- `newton/examples/ik/example_ik_custom.py`
  - job: 自定义 objective，说明 IKObjective 是可扩展 residual block
  - 不进 first-pass mainline
- `newton/examples/ik/example_ik_cube_stacking.py`
  - job: IK feeding `control.joint_target_pos` inside a larger task loop
  - 只做 systems branch，不进 first-pass mainline

---

## 4. 推荐文件分工

- `README.md`
  - 建立本章范围、桥接句、完成门槛、阅读顺序
- `principle.md`
  - 先讲 `read world / write pose` 的对称关系
  - 再讲 shared FK / state backbone 和 update order
- `examples.md`
  - 用 one-job-per-example 的方式分配各例子的教学任务
- `source-walkthrough.md`
  - 以 `ik_franka + sensor_imu` 为主线
  - 用 `sensor_contact` 作为必要的 side-branch，讲清 `contacts` 的 update timing

这轮不建议一开始就写 `source-walkthrough-deep.md`。

---

## 5. 推荐主源码锚点

- public API
  - `newton/sensors.py`
  - `newton/ik.py`
- sensors
  - `newton/_src/sensors/sensor_imu.py`
  - `newton/_src/sensors/sensor_contact.py`
  - `newton/_src/sensors/sensor_tiled_camera.py` 只作 advanced branch
- IK
  - `newton/_src/sim/ik/ik_solver.py`
  - `newton/_src/sim/ik/ik_objectives.py`

---

## 6. 最容易写坏的方式

### 6.1 Catalog trap

- 把 sensor 类型平铺成 `Contact / IMU / Camera / Raycast / FrameTransform` 目录
- 把 IK 例子平铺成 `franka / h1 / custom / cube_stacking` 目录

### 6.2 API trap

- 一上来就介绍 constructor 参数，而不解释 prerequisite
- 不先讲清 `eval_fk`、`body_q`、`body_qdd`、`Contacts` 分别是谁

### 6.3 Math trap

- 把 IK 写成 LM / LBFGS / Jacobian mode 理论课
- 把 sensors 写成 measurement equation 课

### 6.4 False separation trap

- 把 sensors 和 IK 当两个互不相关的章节
- 不强调两者共享的 articulated state backbone

---

## 7. 成功标准

如果这轮做对了，chapter 12 应该让读者稳定带走下面这句话：

```text
sensors 负责从已有状态里读出观测，
IK 负责把任务空间目标写回 joint-space 解；
两者都依赖同一条 FK / body-state backbone。
```

而整条章节连续性应该变成：

```text
10: 我在模拟什么对象
-> 11: 一步求解里数据怎样流动
-> 12: 状态既然存在，我怎样读它，又怎样朝目标去改它
```
