# 短信唤醒报文设计

> 本文档为 `MlinkPro-Communication-Protocol-Specification.md` 的扩展章节，拟作为 **第 13 章：短信唤醒** 插入主规范。涉及的新增 MessageType、Topic、术语等需同步更新至主规范对应章节。

---

## 13. 短信唤醒

### 13.1 背景

当车辆终端（TBOX）处于离线状态（PowerMode = SLEEP_POLL / STANDBY / OFF），MQTT 连接断开，TSP 无法通过常规下行 Topic 下发指令。此时 TSP 通过蜂窝网络 SMS 通道向 TBOX 发送唤醒短信，TBOX 收到后唤醒、建立 MQTT 连接，TSP 随后通过常规 MQTT 链路下发实际指令。

```
  TSP                    SMS Gateway                 TBOX(离线)
   │                         │                          │
   │── SMS WakeUp ─────────▶│                          │
   │                         │── SMS 投递 ────────────▶│
   │                         │                          │
   │                         │          [验证 HMAC]      │
   │                         │          [唤醒 + 连接MQTT]│
   │                         │                          │
   │           EMQX          │                          │
   │            │             │                          │
   │            │◀──── MQTT Connect ────────────────────│
   │            │             │                          │
   │◀── MpmEvent(ONLINE) + SmsWakeupEvent ──────────────│
   │            │             │                          │
   │── CmdRequest(taskId) ─▶│── down/.../request ─────▶│
   │            │             │                          │
   │◀── CmdResponse(ack) ───│◀── up/.../response ──────│
   │            │             │                          │
   │◀── CmdResponse(sack) ──│◀── up/.../response ──────│
```

### 13.2 设计原则

| 原则         | 说明                                                                 |
| ------------ | ------------------------------------------------------------------- |
| 通道独立     | SMS 走蜂窝短信通道，不经过 EMQX/MQTT，与协议报文链路完全隔离           |
| 安全认证     | SMS 携带 HMAC 签名，使用与 MQTT 认证相同的 deviceSecret，防止伪造唤醒    |
| 最小载荷     | SMS 内容压缩在 80 字符以内，确保单条 SMS（7-bit 编码 160 字符上限）完成投递 |
| 唤醒可追溯   | TBOX 唤醒上线后上报 SmsWakeupEvent，TSP 据此关联 SMS 投递与车辆上线      |
| 超时兜底     | TSP 发送 SMS 后设置唤醒等待窗口，超时未上线则标记唤醒失败，可选重试       |

### 13.3 术语补充

在主规范第 3 章「术语与缩略语」中新增：

| 术语 | 说明                                                           |
| ---- | -------------------------------------------------------------- |
| SWU  | SMS Wake-Up，短信唤醒，TSP 通过 SMS 通道唤醒离线 TBOX 的机制   |
| SMS Gateway | 短信网关，TSP 调用的外部短信发送服务，将唤醒短信投递至 TBOX SIM 卡 |

### 13.4 SMS 报文格式

#### 13.4.1 报文结构

SMS 内容为纯 ASCII 文本，使用管道符 `|` 分隔字段：

```
MWU|{task_id}|{timestamp}|{wake_type}|{hmac}
```

| 字段位置 | 字段名     | 类型   | 长度    | 说明                                                         |
| -------- | ---------- | ------ | ------- | ----------------------------------------------------------- |
| 0        | magic      | 固定串 | 3       | 协议标识，固定为 `MWU`（MlinkPro Wake-Up）                  |
| 1        | task_id    | string | 36      | 待执行指令的 taskId（UUID v4），与后续 CmdRequest 的 task_id 一致 |
| 2        | timestamp  | 数字串 | 10      | SMS 生成时间戳，Unix 秒级（非毫秒，节省空间）                 |
| 3        | wake_type  | 数字串 | 1       | 唤醒类型：`0`=控车指令，`1`=配置下发，`2`=诊断                |
| 4        | hmac       | hex串  | 16      | HMAC-SHA256 签名前 8 字节的十六进制表示（16 个 hex 字符）     |

#### 13.4.2 报文示例

```
MWU|550e8400-e29b-41d4-a716-446655440000|1724000000|0|a1b2c3d4e5f6a7b8
```

字符统计：3 + 1 + 36 + 1 + 10 + 1 + 1 + 1 + 16 = **70 字符**，远低于单条 SMS 7-bit 编码的 160 字符上限。

#### 13.4.3 HMAC 签名计算

```
hmac_full = HMAC-SHA256(deviceSecret, magic + "|" + task_id + "|" + timestamp + "|" + wake_type)
hmac      = lowercase(hex(hmac_full[0:8]))   // 取前 8 字节，转 16 个小写 hex 字符
```

**待签字符串示例**（不含 hmac 字段本身）：

```
MWU|550e8400-e29b-41d4-a716-446655440000|1724000000|0
```

**计算过程**：

1. 拼接前 4 个字段（`magic` 到 `wake_type`），用 `|` 分隔，得到待签字符串。
2. 使用设备的 `deviceSecret` 作为密钥，计算 HMAC-SHA256。
3. 截取结果前 8 字节，转 16 个小写十六进制字符，作为 SMS 中的 `hmac` 字段。

> **安全说明**：HMAC 截断至 8 字节（64 bit），在 SMS 场景下可接受——攻击者需要平均 2^63 次尝试才能伪造，且 deviceSecret 每车独立，单台被破解不影响其他车辆。若对安全等级有更高要求，可扩展至完整 32 字节 hex（64 字符），仍可放入单条 SMS。

#### 13.4.4 wake_type 定义

| 值  | 枚举名              | 说明                               |
| --- | ------------------- | ---------------------------------- |
| 0   | WAKE_TYPE_COMMAND   | 控车指令唤醒（RDU/RDL/RES/RHL/RCE 等） |
| 1   | WAKE_TYPE_CONFIG    | 配置下发唤醒                        |
| 2   | WAKE_TYPE_DIAGNOSIS | 诊断任务唤醒                        |

### 13.5 TBOX 侧 SMS 处理流程

TBOX 在离线状态（SLEEP_POLL / STANDBY）下仍保持对 SMS 接收的监听。收到 SMS 后执行以下流程：

```
TBOX 收到 SMS
  │
  ├── 1. 解析字段：检查 magic == "MWU"，格式是否合法
  │       ├── 格式不合法 → 忽略，记录日志
  │       └── 格式合法 → 继续
  │
  ├── 2. 校验 timestamp：|当前时间 - timestamp| ≤ 300 秒（5 分钟窗口）
  │       ├── 超时 → 忽略，记录日志（防重放）
  │       └── 未超时 → 继续
  │
  ├── 3. 验证 HMAC：使用 SE 中的 deviceSecret 计算并比对
  │       ├── 不匹配 → 忽略，记录安全日志
  │       └── 匹配 → 继续
  │
  ├── 4. 唤醒动作：
  │       ├── 记录 task_id、wake_type、sms_timestamp
  │       ├── 唤醒 TBOX 主处理器
  │       ├── 建立 MQTT 连接（TLS 8883，流程同 5.1 节）
  │       └── 连接成功后发送 MpmEvent(ONLINE) + SmsWakeupEvent
  │
  └── 5. 保持在线窗口：
          ├── 默认保持 300 秒（5 分钟）等待 TSP 下发指令
          ├── 收到指令后正常执行三段式应答
          └── 窗口内无指令 → 主动下线（发送 MpmEvent(OFFLINE, NORMAL_DISCONNECT)）
```

### 13.6 上行事件 — 短信唤醒上报（SmsWakeupEvent）

TBOX 唤醒并成功建立 MQTT 连接后，在发送 MpmEvent(ONLINE) 之后立即发送 SmsWakeupEvent，用于告知 TSP 本机被 SMS 唤醒的详情。

#### 13.6.1 Topic

```
up/{productKey}/{clientId}/event/sms-wakeup
```

| 属性   | 值                                  |
| ------ | ----------------------------------- |
| QoS    | 1                                   |
| Retain | false                               |
| 方向   | 上行（TBOX → TSP）                  |

#### 13.6.2 MessageType

在主规范第 7.4 节 MessageType 枚举的「上行上报」段（0-99）新增：

```protobuf
MSG_SMS_WAKEUP_EVENT = 6;   // 短信唤醒事件上报
```

#### 13.6.3 Envelope

```
version = 1
type    = MSG_SMS_WAKEUP_EVENT
```

#### 13.6.4 Payload

```protobuf
message SmsWakeupEvent {
    string       task_id       = 1;   // 唤醒对应的任务ID（来自SMS中的task_id）
    WakeupType   wake_type     = 2;   // 唤醒类型
    WakeupResult result        = 3;   // 唤醒结果
    uint64       sms_timestamp = 4;   // SMS中的时间戳（秒级，转为uint64）
    uint64       wakeup_time   = 5;   // TBOX实际唤醒时间（Unix毫秒）
    string       vin           = 6;   // 车架号
    string       iccid         = 7;   // SIM卡ICCID
}

enum WakeupType {
    WAKE_TYPE_COMMAND   = 0;   // 控车指令唤醒
    WAKE_TYPE_CONFIG    = 1;   // 配置下发唤醒
    WAKE_TYPE_DIAGNOSIS = 2;   // 诊断任务唤醒
}

enum WakeupResult {
    WAKEUP_SUCCESS              = 0;   // 唤醒成功
    WAKEUP_ERR_HMAC_INVALID     = 1;   // HMAC验证失败（仍上报，供TSP审计）
    WAKEUP_ERR_TIMESTAMP_EXPIRED = 2;  // 时间戳过期
    WAKEUP_ERR_FORMAT_INVALID   = 3;   // SMS格式无法识别
}
```

> **字段说明**：
> - `task_id`：与 SMS 中的 task_id 一致，TSP 据此关联待执行的指令。
> - `result`：即使 HMAC 验证失败或格式错误，TBOX 仍可选择连接上报（以 `WAKEUP_ERR_*` 标记），供 TSP 审计异常唤醒尝试。若 HMAC 失败且 TSP 判定为攻击，可通过密钥轮换或撤销流程处置。
> - `sms_timestamp`：从 SMS 中提取的秒级时间戳，TSP 可据此计算 SMS 投递延迟。
> - `wakeup_time`：TBOX 实际被唤醒的时刻（毫秒），TSP 可据此计算唤醒耗时。

### 13.7 TSP 侧 SMS 唤醒流程

#### 13.7.1 触发条件

TSP 检测到以下条件时触发 SMS 唤醒：

1. 用户/App 发起控车请求（如远程解锁、启动空调等）。
2. TSP 查询设备状态为**离线**（最近一次 MpmEvent 为 OFFLINE，或 Keep Alive 超时）。
3. TSP 生成 taskId，构造 CmdRequest，但发现 MQTT 下行投递失败（或判定设备不在线直接走 SMS）。

#### 13.7.2 SMS 发送策略

| 参数             | 默认值 | 说明                                                         |
| ---------------- | ------ | ------------------------------------------------------------ |
| 最大重试次数     | 2      | 首次发送 + 2 次重试，共 3 条 SMS                              |
| 重试间隔         | 30 秒  | 每次重试间隔                                                 |
| 唤醒等待超时     | 90 秒  | 发送 SMS 后等待车辆上线的超时时间                             |
| 保持在线窗口     | 300 秒 | TBOX 唤醒后保持在线的时间，超时后 TBOX 主动下线               |
| 同一 taskId 去重 | 5 分钟 | 同一 taskId 在 5 分钟内不重复发送 SMS                        |

#### 13.7.3 处理流程

```
TSP 收到控车请求，设备离线
  │
  ├── 1. 生成 taskId（UUID v4）
  │
  ├── 2. 构造 SMS 报文：
  │       ├── 拼接 magic|task_id|timestamp|wake_type
  │       ├── 计算 HMAC（使用 deviceSecret）
  │       └── 通过 SMS Gateway 发送
  │
  ├── 3. 启动唤醒等待计时器（90 秒）
  │       │
  │       ├── 收到 MpmEvent(ONLINE) + SmsWakeupEvent ──┐
  │       │                                             │
  │       ├── 验证 SmsWakeupEvent.task_id == taskId     │
  │       │   ├── 匹配 → 下发 CmdRequest（MQTT 正常链路）│
  │       │   └── 不匹配 → 记录异常日志，继续等待        │
  │       │                                             │
  │       ├── 超时未上线                                │
  │       │   ├── 重试次数 < 2 → 等待 30 秒后重发 SMS   │
  │       │   └── 重试次数 ≥ 2 → 标记「唤醒失败」，通知用户 │
  │       │                                             │
  │       └── 车辆上线但 wake_type 不匹配               │
  │           └── 记录异常，仍可下发指令（容错）          │
  │
  └── 4. 指令下发成功后：
          ├── 清除唤醒等待计时器
          └── 正常进入三段式应答流程
```

#### 13.7.4 异常场景处理

| 场景                           | 处理方式                                                     |
| ------------------------------ | ------------------------------------------------------------ |
| SMS 发送失败（网关返回错误）    | 立即重试 1 次；仍失败则标记「SMS 投递失败」，通知用户         |
| SMS 发送成功但车辆未上线        | 按重试策略重发 SMS；超过最大重试次数后标记「唤醒超时」         |
| 车辆上线但 SmsWakeupEvent 缺失  | TSP 仍可下发指令（容错），但记录告警日志                       |
| SmsWakeupEvent.result != SUCCESS | TSP 记录安全审计日志；若 result = HMAC_INVALID，触发安全告警  |
| 同一车辆多次 SMS 唤醒           | 通过 taskId 去重窗口（5 分钟）避免重复发送                     |
| 车辆在 SMS 发送期间自行上线     | TSP 检测到 MpmEvent(ONLINE) 后直接下发指令，取消 SMS 重试      |

### 13.8 错误码补充

#### 13.8.1 指令结果码（ResultCode）扩展

在主规范第 12.1 节 ResultCode 枚举中新增：

| 值  | 枚举名               | 说明                                   |
| --- | -------------------- | -------------------------------------- |
| 7   | ERR_DEVICE_OFFLINE   | 设备离线，已发送 SMS 唤醒              |
| 8   | ERR_SMS_WAKEUP_FAILED | SMS 唤醒失败（投递失败或超时未上线）   |
| 9   | ERR_SMS_GATEWAY_ERROR | SMS 网关异常                            |

#### 13.8.2 SmsWakeupEvent 结果码

见 13.6.4 节 `WakeupResult` 枚举定义。

### 13.9 Topic 全景表补充

在主规范第 6.2.1 节上行 Topic 表中新增：

| Topic                                      | QoS | Retain | 说明                   |
| :----------------------------------------- | :-- | :----- | :--------------------- |
| `up/{productKey}/{clientId}/event/sms-wakeup` | 1   | false  | 短信唤醒事件上报       |

### 13.10 ACL 规则补充

SMS 唤醒不涉及额外的 MQTT Topic 权限，SmsWakeupEvent 上报复用现有 TBOX 发布规则：

```text
TBOX 发布: up/{productKey}/${clientId}/event/sms-wakeup  →  已被现有规则 up/{productKey}/${clientId}/# 覆盖
TSP 订阅: up/{productKey}/#                                →  已被现有规则覆盖
```

无需新增 ACL 条目。

### 13.11 SMS 报文安全分析

| 威胁                 | 防护措施                                                     |
| -------------------- | ------------------------------------------------------------ |
| 伪造 SMS 唤醒        | HMAC-SHA256 签名，攻击者无 deviceSecret 无法伪造合法 SMS      |
| 重放攻击             | timestamp 5 分钟窗口校验，超时 SMS 被丢弃                      |
| 短信嗅探             | SMS 内容不含敏感业务数据（无 VIN、无指令参数），仅含 taskId   |
| deviceSecret 泄露    | 一机一密，单台泄露不影响其他车辆；支持密钥轮换（11.4.6 节）     |
| 暴力伪造 HMAC        | 截断至 8 字节（64 bit），暴力破解需 2^63 次尝试；可扩展至 32 字节 |
| SMS 内容被篡改       | HMAC 覆盖 task_id + timestamp + wake_type 全字段，篡改即失效   |

### 13.12 完整 SMS 报文与事件时序示例

以「远程解锁车门（RDU-ON）」为例，车辆离线场景完整流程：

```
时间线          TSP                         SMS Gateway              TBOX(离线)
  │
T0   用户发起远程解锁请求
  │
T1   TSP 检测车辆离线
     生成 taskId = 550e8400-e29b-41d4-a716-446655440000
     构造 SMS:
       MWU|550e8400-e29b-41d4-a716-446655440000|1724000000|0|a1b2c3d4e5f6a7b8
  │
T2   调用 SMS Gateway 发送 ──────────▶ 投递 SMS ─────────────▶ TBOX 收到 SMS
  │                                                              │
T3                                                                解析 + 验证 HMAC ✓
                                                                  校验 timestamp ✓
                                                                  唤醒 + 发起 MQTT 连接
  │                                                                │
T4                                                                MQTT 连接成功
                                                                  │
T5   ◀── MpmEvent(ONLINE) ─────────────────────────────────────── 发送上线事件
     ◀── SmsWakeupEvent(task_id=550e8400..., result=SUCCESS) ── 发送唤醒事件
  │
T6   TSP 验证 task_id 匹配
     下发 CmdRequest(RDU-ON, taskId=550e8400...) ──────────────▶ 收到指令
  │                                                                │
T7   ◀── CmdResponse(ACK, SUCCESS) ───────────────────────────── 回复 ack
  │                                                                │
T8                                                                执行解锁
     ◀── CmdResponse(SACK, SUCCESS, vehicle_status) ────────── 回复 sack
  │
T9   指令完成，用户收到解锁成功通知
  │
T10  (300秒后 TBOX 无新指令)
     ◀── MpmEvent(OFFLINE, NORMAL_DISCONNECT) ──────────────── TBOX 主动下线
```

### 13.13 Protobuf 定义汇总

以下为新增的 Protobuf 定义，需追加至附录 A 的 `mlink-pro-protocol-v1.proto` 文件：

```protobuf
// ===== 短信唤醒 =====

message SmsWakeupEvent {
    string       task_id       = 1;   // 唤醒对应的任务ID（来自SMS）
    WakeupType   wake_type     = 2;   // 唤醒类型
    WakeupResult result        = 3;   // 唤醒结果
    uint64       sms_timestamp = 4;   // SMS中的时间戳（秒级）
    uint64       wakeup_time   = 5;   // TBOX实际唤醒时间（Unix毫秒）
    string       vin           = 6;   // 车架号
    string       iccid         = 7;   // SIM卡ICCID
}

enum WakeupType {
    WAKE_TYPE_COMMAND   = 0;   // 控车指令唤醒
    WAKE_TYPE_CONFIG    = 1;   // 配置下发唤醒
    WAKE_TYPE_DIAGNOSIS = 2;   // 诊断任务唤醒
}

enum WakeupResult {
    WAKEUP_SUCCESS              = 0;   // 唤醒成功
    WAKEUP_ERR_HMAC_INVALID     = 1;   // HMAC验证失败
    WAKEUP_ERR_TIMESTAMP_EXPIRED = 2;  // 时间戳过期
    WAKEUP_ERR_FORMAT_INVALID   = 3;   // SMS格式无法识别
}
```

MessageType 枚举新增：

```protobuf
MSG_SMS_WAKEUP_EVENT = 6;   // 短信唤醒事件上报（上行，0-99段）
```

ResultCode 枚举新增：

```protobuf
ERR_DEVICE_OFFLINE    = 7;   // 设备离线，已发送SMS唤醒
ERR_SMS_WAKEUP_FAILED = 8;   // SMS唤醒失败
ERR_SMS_GATEWAY_ERROR = 9;   // SMS网关异常
```

### 13.14 主规范改动清单

| 章节           | 改动内容                                                     |
| -------------- | ------------------------------------------------------------ |
| 第 3 章        | 术语表新增 SWU、SMS Gateway                                   |
| 第 6.2.1 节    | 上行 Topic 表新增 `up/.../event/sms-wakeup`                   |
| 第 7.4 节      | MessageType 枚举新增 `MSG_SMS_WAKEUP_EVENT = 6`              |
| 第 12.1 节     | ResultCode 枚举新增 `ERR_DEVICE_OFFLINE(7)` 等 3 项           |
| 第 13 章       | 新增本文档全部内容                                            |
| 附录 A         | proto 文件追加 SmsWakeupEvent、WakeupType、WakeupResult 定义   |
| 版本历史      | 新增 V1.0.2 版本记录                                         |
