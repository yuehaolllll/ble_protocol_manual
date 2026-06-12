# 手机 APP BLE 接入协议与操作手册 V2

适用固件基线：当前工作区 `EEG2channel`，截至 2026-06-12 的 BLE 协议实现。

目标读者：
- 手机 APP 开发
- 联调测试
- 产品与实施

当前版本的核心原则：
- 手机 APP 必须订阅 `FFF1`、`FFF3`、`FFF4`，并通过 `FFF2` 发送控制命令。
- 手机 APP 唯一不需要做的事情，是不需要展示 EEG 波形数据流。
- 当前固件中的专注/疲劳指标不在 `FFF1` 中发送，而是封装在 `FFF4` 的完整帧内。
- 因此手机 APP 即使不展示 EEG 波形，也必须接收并重组 `FFF4`，从中提取专注/疲劳。
- 开始采集后的前 5 秒属于预热阶段，EEG 数据已经开始发送，但专注/疲劳结果暂不生效。

---

## 1. 总体工作模式

设备运行模式：

| 模式 | 数值 | 含义 |
|---|---:|---|
| `READY` | 0 | 设备已上电，但尚未建立 BLE 连接 |
| `CHECK` | 1 | BLE 已连接，正在进行佩戴/阻抗检查 |
| `RUNNING` | 2 | 已开始采集，正在输出 EEG 主数据 |

推荐的手机 APP 用户流程：

1. 扫描并连接设备。
2. 发现主服务 `FFF0`。
3. 订阅 `FFF1`，进入检查页。
4. 订阅 `FFF3`，接收 IMU 姿态与动作事件。
5. 等待 `FFF1` 中佩戴状态和阻抗状态满足要求。
6. 向 `FFF2` 发送开始采集命令。
7. 订阅 `FFF4`。
8. 前 5 秒显示“预热中”，不展示正式专注/疲劳结果。
9. 5 秒后从 `FFF4` 提取有效的专注/疲劳指标。
10. 停止采集时，向 `FFF2` 发送停止命令，界面回到检查页。

---

## 2. GATT 服务与特征定义

主服务 UUID：

| 类型 | UUID |
|---|---|
| Primary Service | `0xFFF0` |

特征定义：

| 特征 | UUID | 属性 | 用途 |
|---|---|---|---|
| 状态流 | `0xFFF1` | Read, Notify | 电池、佩戴、阻抗、事件、设备模式 |
| 控制流 | `0xFFF2` | Read, Write, Write Without Response | 开始采集、停止采集 |
| IMU 流 | `0xFFF3` | Read, Notify | 姿态、加速度、角速度、IMU 事件 |
| EEG 主流 | `0xFFF4` | Read, Notify | EEG 分片帧，内含同帧 `focus/fatigue` |

当前版本的重要说明：
- `FFF1` 不再发送专注/疲劳。
- `FFF3` 是手机 APP 的必订阅通道，不建议省略。
- `FFF4` 是手机 APP 获取专注/疲劳的唯一正式来源。
- 手机 APP 不需要展示 EEG 波形，但仍需接收和重组 `FFF4`。

---

## 3. 控制协议

`FFF2` 写入 1 字节控制命令：

| 命令 | 数值 | 说明 |
|---|---:|---|
| `NONE` | `0x00` | 无操作 |
| `START_ACQUISITION` | `0x01` | 开始采集 |
| `STOP_ACQUISITION` | `0x02` | 停止采集 |

推荐写入方式：
- 优先使用 `Write Without Response`
- 若手机蓝牙栈兼容性一般，可退回普通 `Write`

APP 侧建议：
- 点击“开始采集”后，不要立即把 UI 当作“指标已有效”。
- 等待 `FFF1.device_mode == RUNNING` 后，再进入 `FFF4` 预热计时。

---

## 4. FFF1 状态流协议

### 4.1 发送节奏

| 模式 | 发送周期 |
|---|---|
| `READY` | 1000 ms |
| `CHECK` | 200 ms |
| `RUNNING` | 1000 ms |

`FFF1` 是低频状态流，不承担专注/疲劳输出。

### 4.2 数据结构

字节序：小端序 `little-endian`

总长度：29 字节

| 偏移 | 类型 | 字段 | 说明 |
|---:|---|---|---|
| 0 | `uint32` | `sequence` | 状态包序号 |
| 4 | `uint32` | `timestamp_ms` | 状态快照时间戳，毫秒 |
| 8 | `uint8` | `battery_soc_percent` | 电量百分比 |
| 9 | `uint8` | `wear_state` | 佩戴状态 |
| 10 | `uint16` | `proximity_counts` | 佩戴接近值 |
| 12 | `uint16` | `battery_voltage_mv` | 电池电压，mV |
| 14 | `int16` | `battery_current_ma` | 电池电流，mA |
| 16 | `uint16` | `event_flags` | 事件标志位 |
| 18 | `uint8` | `electrode_flags` | 电极状态标志位 |
| 19 | `uint8` | `electrode_ch1_state` | 通道 1 接触状态 |
| 20 | `uint8` | `electrode_ch2_state` | 通道 2 接触状态 |
| 21 | `uint16` | `electrode_ref_impedance_kohm` | 参考端阻抗，kΩ |
| 23 | `uint16` | `electrode_ch1_impedance_kohm` | 通道 1 阻抗，kΩ |
| 25 | `uint16` | `electrode_ch2_impedance_kohm` | 通道 2 阻抗，kΩ |
| 27 | `uint8` | `flags` | 数据有效位 |
| 28 | `uint8` | `device_mode` | 设备模式 |

### 4.3 `wear_state` 枚举

| 数值 | 含义 |
|---:|---|
| 0 | `HELMET_WEAR_UNKNOWN` |
| 1 | `HELMET_WEAR_NOT_WORN` |
| 2 | `HELMET_WEAR_WORN` |

### 4.4 `device_mode` 枚举

| 数值 | 含义 |
|---:|---|
| 0 | `READY` |
| 1 | `CHECK` |
| 2 | `RUNNING` |

### 4.5 `event_flags` 位定义

| Bit | 掩码 | 含义 |
|---:|---:|---|
| 0 | `0x0001` | `IMPACT` |
| 1 | `0x0002` | `FALL` |
| 2 | `0x0004` | `UNRESPONSIVE` |
| 3 | `0x0008` | `HELMET_WORN` |
| 4 | `0x0010` | `BATTERY_LOW` |

### 4.6 `flags` 位定义

| Bit | 掩码 | 含义 |
|---:|---:|---|
| 1 | `0x02` | 电池数据有效 |
| 3 | `0x08` | 佩戴数据有效 |
| 4 | `0x10` | 电极数据有效 |

### 4.7 `electrode_flags` 位定义

| Bit | 掩码 | 含义 |
|---:|---:|---|
| 0 | `0x01` | 电极状态数据有效 |
| 1 | `0x02` | 参考端 / RLD 脱落 |
| 2 | `0x04` | CH1 正端脱落 |
| 3 | `0x08` | CH1 负端脱落 |
| 4 | `0x10` | CH2 正端脱落 |
| 5 | `0x20` | CH2 负端脱落 |
| 6 | `0x40` | 全部电极准备完成 |
| 7 | `0x80` | 阻抗检测正在进行 |

### 4.8 `electrode_ch1_state / electrode_ch2_state` 枚举

| 数值 | 含义 |
|---:|---|
| 0 | `UNKNOWN` |
| 1 | `OFF` |
| 2 | `POOR` |
| 3 | `GOOD` |

### 4.9 `FFF1` 的解析建议

APP 侧解析顺序建议：

1. 先校验整包长度是否为 29 字节。
2. 读取 `device_mode` 判断当前阶段。
3. 读取 `flags` 判断电池、佩戴、电极字段是否有效。
4. 读取 `wear_state`、`electrode_flags.bit6` 判断是否可以开始采集。
5. 读取 `event_flags` 进行告警展示。

### 4.10 手机 APP 推荐用途

`FFF1` 推荐用于：
- 顶部连接状态栏
- 电池、电压
- 佩戴检测
- 阻抗检查页
- 设备模式切换判断
- 风险 / 告警提示

不推荐用于：
- 专注 / 疲劳显示
- 高频运动曲线

---

## 5. FFF3 IMU 流协议

### 5.1 发送节奏

- IMU 采样周期：20 ms
- 设计目标：约 50 Hz
- BLE 侧每轮最多补发 5 包 IMU

### 5.2 数据结构

字节序：小端序

总长度：30 字节

| 偏移 | 类型 | 字段 | 说明 |
|---:|---|---|---|
| 0 | `uint32` | `sample_count` | IMU 样本计数 |
| 4 | `uint32` | `timestamp_ms` | IMU 时间戳，毫秒 |
| 8 | `int16` | `roll_cdeg` | Roll，单位 0.01° |
| 10 | `int16` | `pitch_cdeg` | Pitch，单位 0.01° |
| 12 | `int16` | `yaw_cdeg` | Yaw，单位 0.01° |
| 14 | `int16` | `accel_x_mg` | X 加速度，mg |
| 16 | `int16` | `accel_y_mg` | Y 加速度，mg |
| 18 | `int16` | `accel_z_mg` | Z 加速度，mg |
| 20 | `int16` | `gyro_x_cdps` | X 角速度，0.01 dps |
| 22 | `int16` | `gyro_y_cdps` | Y 角速度，0.01 dps |
| 24 | `int16` | `gyro_z_cdps` | Z 角速度，0.01 dps |
| 26 | `int16` | `temperature_cdeg` | 温度，0.01 ℃ |
| 28 | `uint8` | `flags` | IMU 数据有效位 |
| 29 | `uint8` | `last_event` | 最后一次 IMU 事件 |

### 5.3 `flags` 位定义

| Bit | 掩码 | 含义 |
|---:|---:|---|
| 0 | `0x01` | IMU 已初始化 |
| 1 | `0x02` | IMU 数据有效 |
| 2 | `0x04` | 当前处于撞击事件保持期 |
| 3 | `0x08` | 当前处于跌倒事件保持期 |
| 4 | `0x10` | 当前处于静止异常事件保持期 |

### 5.4 `last_event` 枚举

| 数值 | 含义 |
|---:|---|
| 0 | `NONE` |
| 1 | `IMPACT` |
| 2 | `FALL` |
| 3 | `UNRESPONSIVE` |

### 5.5 `FFF3` 的解析建议

APP 侧解析顺序建议：

1. 校验长度是否为 30 字节。
2. 读取 `flags`，确认 IMU 是否已初始化、数据是否有效。
3. 读取 `roll/pitch/yaw` 做姿态显示。
4. 读取 `last_event` 和 `flags` 中的事件位，做撞击 / 跌倒 / 静止异常提示。

### 5.6 手机 APP 推荐用途

`FFF3` 在当前手机 APP 方案中视为必订阅通道。

推荐用途：
- 实时姿态球
- 头部朝向
- 跌倒 / 撞击提示
- 运动状态页
- 与 `FFF1` 状态页联动展示设备运行状态

---

## 6. FFF4 EEG 主流协议

### 6.1 关键提醒

当前手机 APP 虽然不需要展示 EEG 波形，但如果需要专注 / 疲劳，仍然必须订阅 `FFF4`。

原因：
- 当前固件的专注 / 疲劳不再通过 `FFF1` 下发。
- 专注 / 疲劳与 EEG 同帧封装在 `FFF4` 的完整帧负载中。

因此手机 APP 的推荐做法是：
- 接收并重组 `FFF4`
- 仅提取 `focus/fatigue`
- 不在界面上展示后面的两个 EEG 数组

### 6.2 分片头结构

字节序：小端序

头长度：18 字节

| 偏移 | 类型 | 字段 | 说明 |
|---:|---|---|---|
| 0 | `uint32` | `waveform_sequence` | EEG 帧序号 |
| 4 | `uint32` | `frame_timestamp_ms` | 该 EEG 帧时间戳 |
| 8 | `uint32` | `metrics_update_count` | 指标更新计数 |
| 12 | `uint16` | `chunk_index` | 当前块序号，从 0 开始 |
| 14 | `uint16` | `chunk_count` | 总块数 |
| 16 | `uint16` | `payload_len` | 当前块负载长度 |

### 6.3 完整帧负载结构

完整负载类型：`WaveformPacket_t`

字节序：小端序

总长度：2016 字节

| 偏移 | 类型 | 字段 | 说明 |
|---:|---|---|---|
| 0 | `uint32` | `magic` | 固定值 `0xAABBCCDD` |
| 4 | `uint32` | `sequence` | 帧序号，应与 `waveform_sequence` 一致 |
| 8 | `float32` | `focus` | 专注值，范围通常 `0.0 ~ 1.0` |
| 12 | `float32` | `fatigue` | 疲劳值，范围通常 `0.0 ~ 1.0` |
| 16 | `float32[250]` | `ch1_ai_wave` | CH1 滤波后、去噪前对比波形 |
| 1016 | `float32[250]` | `ch2_ai_wave` | CH1 去噪后对比波形 |

### 6.4 手机 APP 的最小解析策略

手机 APP 如果只要专注 / 疲劳：

1. 按 `waveform_sequence` 收集全部 chunk。
2. 当 `chunk_index` 全部齐全后，按顺序拼接负载。
3. 只读取完整帧前 16 字节：
   - `magic`
   - `sequence`
   - `focus`
   - `fatigue`
4. 校验 `magic == 0xAABBCCDD`
5. 不展示后续 2000 字节波形数据

### 6.5 5 秒预热逻辑

采集开始后：
- EEG 会立即开始发送
- 但专注 / 疲劳前 5 秒处于 `warm-up`

当前固件中的表现：
- `focus = 0.0`
- `fatigue = 0.0`
- `metrics_update_count = 0`

这 5 秒等待逻辑的目的：
- 让开始采集后的历史缓冲区、滤波状态、去噪模型输入进入稳定区
- 避免启动瞬态直接被手机 APP 当成正式专注 / 疲劳结果
- 该阶段并不是“没有采集”，而是“已经采集并发送，但指标暂不生效”

手机 APP 必须将以下情况视为“指标未就绪”：
- `metrics_update_count == 0`
- 或 `focus == 0 && fatigue == 0` 且仍处于采集开始后的前几秒

推荐 UI：
- 显示“采集中”
- 显示“预热中 5s”
- 不要把 `0` 当作真实疲劳 / 专注值
- 推荐增加倒计时或进度提示，例如“正在建立稳定指标 5s”

### 6.6 推荐的指标判定策略

| 条件 | 处理 |
|---|---|
| `metrics_update_count == 0` | 视为预热中，不展示正式指标 |
| `metrics_update_count > 0` 且重组成功 | 展示本帧 `focus/fatigue` |
| chunk 丢失或帧重组失败 | 丢弃该帧，不回填旧值为新值 |

### 6.7 FFF4 的解析实现建议

APP 侧建议维护以下缓存：
- 当前 `waveform_sequence`
- `chunk_count`
- 已收到的 chunk 位图
- 拼接缓冲区
- 接收起始时间

超时建议：
- 单帧重组超过 2 秒仍未完成，则丢弃该帧

Android 端建议：
- 尽量请求更大的 MTU，例如 `247`
- 否则 `FFF4` 分片数会增加，重组压力更大

---

## 7. 手机 APP 推荐订阅组合

当前手机 APP 标准订阅组合如下：

- 订阅 `FFF1`
- 订阅 `FFF3`
- 订阅 `FFF4`
- 写入 `FFF2`

该组合下可以达到：
- 电池、佩戴、阻抗、模式显示
- IMU 姿态与事件显示
- 开始 / 停止采集
- 通过 `FFF4` 获取专注 / 疲劳
- 不需要展示 EEG 波形本身

不推荐组合：
- 只订阅 `FFF1` + `FFF2`

原因：
- 当前 `FFF1` 不再提供专注 / 疲劳
- 只能完成连接、检查、控制，不能拿到正式指标

---

## 8. 手机 APP 状态机

### 8.1 连接状态机

1. `Idle`
2. `Scanning`
3. `Connecting`
4. `Discovering`
5. `Subscribing`
6. `Connected-Check`
7. `Connected-Running`
8. `Disconnected`

### 8.2 采集状态机

1. `CheckPending`
   - 刚连接
   - 等待佩戴 / 阻抗稳定

2. `CheckReady`
   - `device_mode == CHECK`
   - `electrode_flags.bit6 == 1`
   - `wear_state == WORN`

3. `AcquireWarmup`
   - 已发送 `START`
   - `device_mode == RUNNING`
   - `FFF4.metrics_update_count == 0`

4. `AcquireActive`
   - `device_mode == RUNNING`
   - `FFF4.metrics_update_count > 0`

5. `Stopping`
   - 已发送 `STOP`
   - 等待 `device_mode` 回到 `CHECK`

### 8.3 UI 对应建议

| 状态 | 手机端显示 |
|---|---|
| `CheckPending` | 正在检查佩戴与电极 |
| `CheckReady` | 可以开始采集 |
| `AcquireWarmup` | 采集中，指标预热中 |
| `AcquireActive` | 采集中，指标有效 |
| `Stopping` | 正在停止采集 |

---

## 9. 推荐的连接与操作顺序

### 9.1 进入检查页

1. 扫描并连接设备
2. 发现 `FFF0`
3. 打开 `FFF1` Notify
4. 打开 `FFF3` Notify
5. 等待状态稳定

检查页推荐展示：
- 设备名
- 电池
- 佩戴状态
- CH1 / CH2 阻抗值
- CH1 / CH2 状态 `GOOD / POOR / OFF`
- 设备模式 `CHECK`

### 9.2 开始采集

1. 向 `FFF2` 写入 `0x01`
2. 观察 `FFF1.device_mode` 变为 `RUNNING`
3. 打开 `FFF4` Notify
4. 开始接收 `FFF4`
5. 前 5 秒进入 `AcquireWarmup`
6. `metrics_update_count > 0` 后进入 `AcquireActive`

### 9.3 停止采集

1. 向 `FFF2` 写入 `0x02`
2. 等待 `FFF1.device_mode` 回到 `CHECK`
3. 保持 `FFF1` 和 `FFF3` 持续接收
4. `FFF4` 可继续保持订阅，或在停止后关闭

---

## 10. 手机 APP 当前可达到的效果

在不展示 EEG 波形的前提下，手机 APP 现在应能实现：

- 稳定连接与断线重连
- 佩戴检查
- 阻抗检查
- 电池状态显示
- IMU 姿态显示
- 跌倒、撞击、静止异常提示
- 设备模式显示
- 开始 / 停止采集
- 采集前 5 秒预热态显示
- 5 秒后实时专注 / 疲劳显示

从产品体验上看，用户应感受到的是：
- 先检查
- 再采集
- 先预热
- 再输出正式认知指标


---

## 11. 手机 APP 最小实现清单

第一版可用 APP 的最小清单如下：

1. 扫描并连接 BLE 设备
2. 发现 `FFF0` 服务
3. 订阅 `FFF1`
4. 订阅 `FFF3`
5. 写 `FFF2` 开始 / 停止采集
6. 订阅 `FFF4`
7. 完成 `FFF4` 分片重组
8. 从完整帧中提取 `focus/fatigue`
9. 做 5 秒预热 UI
10. 不展示 EEG 数组，只展示指标
11. 断开后清空缓存与状态机

---


