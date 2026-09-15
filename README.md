# infantry

RoboMaster 双板（上板 / 下板）固件工程，基于 Zephyr + Dust 架构（`zephyr_user/framework`）。

## 目录

| 目录 | 板卡 | 职责 |
|---|---|---|
| `infantry_head/` | 上板 | 指令/决策来源，周期向下板发送状态帧 |
| `infantry_body/` | 下板 | 底盘 / 云台 / 发射执行，周期向上板回传实测状态 |

## 上下板通信（对应标签 `canfd-baseline`）

- 总线：`user-can2`（dm_mc02 的 FDCAN2），**仲裁段 1 Mbps / 数据段 2 Mbps（FDF + BRS）**
- 契约：两端 `inter_cmd/inter_cmd.hpp` **必须逐字一致**（布局表驱动，帧长自动推导 + 编译期校验）
- 帧：

| 方向 | CAN ID | 帧长 | 分片（偏移 / 长度） |
|---|---|---|---|
| 上板 → 下板 | `0x100` | 61 B | CommState(0/21) · AutoAimState(21/24) · ImuState(45/16) |
| 下板 → 上板 | `0x200` | 12 B | GimbalState(0/8, float) · ShooterState(8/4, float) |

- 收发分工：**发送线程持有总线**（`Can::Init` + `CAN_MODE_FD` + 设置收帧分发入口）；
  接收线程只注册 `CAN_RX_HANDLER`，按表解析 + 30 ms 超时判定 → 发布 zbus（含 `online` / `age_ms`）。
- 本版本没有命令帧；开关类数据未纳入协议。

## 注意

- 本仓库是 `projects/infantry_head`、`projects/infantry_body` 两个工作目录的**快照镜像**；
  日常编译仍在原目录进行（`west build -b stm32h723vgt6 -- -DBOARD_CFG=board_dm_mc02`）。
- 若要把本仓库提升为唯一工作目录，需要把两个 `CMakeLists.txt` 里的
  `${CMAKE_CURRENT_SOURCE_DIR}/../../zephyr_user` 改为 `../../../zephyr_user`
  （目录层级多一层），并重新验证编译。
