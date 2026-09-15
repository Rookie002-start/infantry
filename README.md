# infantry

RoboMaster 双板（上板 / 下板）固件工程，基于 Zephyr + Dust 架构（`zephyr_user/framework`）。

> **仓库根就在 `~/zephyrproject/projects/`**，下面两个目录**就是日常编译的工作目录**（不是副本）。
> `.gitignore` 为白名单模式：只跟踪 `infantry_head/`、`infantry_body/` 与根目录 README/.gitignore；
> 同级的 `temp/`、`cmd/`、`my_app` 等一律忽略。

## 目录

| 目录 | 板卡 | 职责 |
|---|---|---|
| `infantry_head/` | 上板 | 指令 / 决策来源，周期向下板发送状态帧 |
| `infantry_body/` | 下板 | 底盘 / 云台 / 发射执行，周期向上板回传实测状态 |

## 编译

```bash
source ~/zephyrproject/setup_env.sh
cd infantry_head          # 或 infantry_body
west build -b stm32h723vgt6 -- -DBOARD_CFG=board_dm_mc02
```

## 上下板通信（对应标签 `canfd-baseline`）

- 总线：`user-can2`（dm_mc02 的 FDCAN2），**仲裁段 1 Mbps / 数据段 2 Mbps（FDF + BRS）**
- 契约：两端 `inter_cmd/inter_cmd.hpp` **必须逐字一致**（布局表驱动，帧长自动推导 + 编译期校验）。
  改完协议请确认：
  `diff infantry_head/inter_cmd/inter_cmd.hpp infantry_body/inter_cmd/inter_cmd.hpp`
- 帧：

| 方向 | CAN ID | 帧长 | 分片（偏移 / 长度） |
|---|---|---|---|
| 上板 → 下板 | `0x100` | 61 B | CommState(0/21) · AutoAimState(21/24) · ImuState(45/16) |
| 下板 → 上板 | `0x200` | 12 B | GimbalState(0/8, float) · ShooterState(8/4, float) |

- 收发分工：**发送线程持有总线**（`Can::Init` + `CAN_MODE_FD` + 设置收帧分发入口）；
  接收线程只注册 `CAN_RX_HANDLER`，按表解析 + 30 ms 超时判定 → 发布 zbus（含 `online` / `age_ms`）。
- 本版本没有命令帧；开关类数据未纳入协议。

## 提交

```bash
cd ~/zephyrproject/projects
git add -A && git commit -m "..." && git push
```
