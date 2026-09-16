# 车联网 TBOX-TSP 通信协议规范

| 项目     | 内容          |
| -------- | ------------- |
| 文档编号 | TVCP-SPEC-001 |
| 版本     | V1.0.0        |
| 作者     | 徐峰          |
| 创建日期 | 2026-08-17    |
| 状态     | 初稿          |

---

## 版本历史

| 版本   | 日期       | 作者 | 变更说明     |
| ------ | ---------- | ---- | ------------ |
| V1.0.0 | 2026-08-17 | 徐峰 | 初始版本发布 |

---

## 目录

1. [概述](#1-概述)
2. [参考标准](#2-参考标准)
3. [术语与缩略语](#3-术语与缩略语)
4. [系统架构](#4-系统架构)
5. [连接管理](#5-连接管理)
6. [Topic 设计](#6-topic-设计)
7. [消息编码](#7-消息编码)
8. [车辆状态上报](#8-车辆状态上报)
9. [远程控车指令](#9-远程控车指令)
10. [信号采集配置](#10-信号采集配置)
11. [安全设计](#11-安全设计)
12. [错误码定义](#12-错误码定义)
13. [附录 A — Protobuf 定义文件](#附录 A — Protobuf 定义文件)
14. [附录 B — serviceKey 映射表](#附录 B — serviceKey 映射表)

---

## 1. 概述

### 1.1 目的

本文档定义车辆 TBOX（Telematics BOX）与 TSP（Telematics Service Platform）平台之间的 MQTT 通信协议，覆盖连接管理、消息编码、Topic 规范、全部上行/下行消息的 Protobuf 定义以及指令应答机制。

### 1.2 适用范围

本协议适用于TBOX 设备与 TSP 云端平台之间的双向数据通信，包括车辆状态上报、远程控车指令下发与执行结果回复、事件上报等场景。TSP 侧基于 EMQX Broker 进行消息路由，使用 Protocol Buffers v3 作为消息序列化格式。productKey 固定为 `tvcp`。

### 1.3 设计原则

本协议的设计参考了物联网平台的接入规范，遵循以下核心原则：

**上下行分离**：上报（up）与指令（down）使用独立 Topic 命名空间，以 `up/` 和 `down/` 前缀区分，从根本上避免消息环路。

**最小权限**：TBOX 仅可发布自身 clientId（TBOX 设备 ID）下的上行 Topic，仅可订阅自身 clientId 下的下行 Topic，由 EMQX ACL 规则强制约束。

**幂等设计** ：所有下行指令携带全局唯一 `taskId`，TBOX 对同一 `taskId` 的重复指令仅执行一次，但每次都回复 ack，确保平台侧重试安全。

**三段式应答**： TSP 下发指令 → TBOX 回复 ack（已接收）→ TBOX 回复 sack（执行结果），全链路可追踪。

**异常可感知**：TBOX 连接时设置遗嘱消息（LWT），异常断连时 EMQX 自动发布离线事件，平台可即时感知设备掉线。

**信封路由**：所有消息采用 Envelope 信封封装，`bytes payload` 支持独立版本演进，`MessageType type` 用于接收方高效分发和反序列化。

**轻量高效**：采用 Protobuf 二进制编码，相比 JSON 体积减少 60%-80%，适合车载弱网环境。

---

## 2. 参考标准

| 标准/方案               | 说明                             |
| ----------------------- | -------------------------------- |
| MQTT 3.1.1 / 5.0        | OASIS MQTT 协议规范              |
| EMQX 5.x                | 分布式 MQTT Broker，用于消息路由 |
| Protocol Buffers v3     | Google 序列化框架，用于消息编码   |

---

## 3. 术语与缩略语

| 术语     | 说明                                                             |
| -------- | ---------------------------------------------------------------- |
| TSP      | Telematics Service Provider，车联网服务平台                       |
| TBOX     | Telematics BOX，车载远程信息处理终端                             |
| clientId | TBOX 设备 ID，TBOX 在 MQTT 连接中使用的唯一标识符               |
| VIN      | Vehicle Identification Number，17 位车架号，用于标识车辆         |
| EMQX     | 开源分布式 MQTT Broker                                           |
| Protobuf | Protocol Buffers，Google 的二进制序列化协议                      |
| LWT      | Last Will and Testament，MQTT 遗嘱消息                           |
| taskId   | 任务唯一标识，由 TSP 生成（UUID v4），用于指令幂等与追踪         |
| ack      | Acknowledgment，TBOX 对指令的接收确认                            |
| sack     | Status Acknowledgment，TBOX 对指令的执行结果回复                 |
| Envelope | 消息信封，所有上下行消息的公共外层结构                           |
| RVS      | Real-time Vehicle Status，实时车况周期上报事件                   |
| TRS      | Triggered Report Status，触发式车况上报事件                      |
| MPM      | Module Power Mode，模块电源模式（上线/下线）事件                 |
| RDU      | Remote Door Unlock，远程解锁全部车门                             |
| RDL      | Remote Door Lock，远程关闭全部车门                               |
| RTU      | Remote Trunk Unlock，远程解锁后备箱                              |
| RES      | Remote Engine Start/Stop，远程启动/关闭发动机                    |
| RHL      | Remote Horn & Lights，远程闪灯鸣笛                              |
| RCE      | Remote Climate Enable，远程开启/关闭空调及温度设置               |

---

## 4. 系统架构

### 4.1 整体架构

```
┌──────────┐         MQTT (TLS)         ┌──────────┐         ┌──────────┐
│          │  ──── 上行 (up/) ────▶     │          │ ──────▶ │          │
│   TBOX   │                            │   EMQX   │         │   TSP    │
│ (车载终端)│  ◀─── 下行 (down/) ────    │  Broker  │ ◀────── │ (Java)   │
│          │         (Protobuf)         │          │         │          │
└──────────┘                            └──────────┘         └──────────┘
```

### 4.2 数据流向

**上行链路（TBOX → TSP）**：TBOX 通过 `up/` 前缀的 Topic 上报车况数据、事件回复。TSP 侧消费者订阅所有 `up/tvcp/#` Topic 进行业务处理。

**下行链路（TSP → TBOX）**：TSP 通过 `down/` 前缀的 Topic 下发控车指令和配置更新。TBOX 仅订阅自身 clientId 对应的 `down/tvcp/{clientId}/#` Topic。

### 4.3 技术选型

| 组件     | 选型           | 说明                           |
| -------- | -------------- | ------------------------------ |
| 传输协议 | MQTT 3.1.1     | 轻量级、低带宽、支持 QoS       |
| Broker   | EMQX 5.x       | 高并发、规则引擎、ACL 鉴权     |
| 消息编码 | Protobuf v3    | 高效二进制编码，跨语言支持     |
| TSP 开发 | Java           | 指令下发、消息消费、业务处理   |
| TLS      | TLS 1.2+       | 传输层加密                     |

---

## 5. 连接管理

### 5.1 MQTT 连接参数

| 参数          | 值 / 策略                                                    |
| ------------- | ------------------------------------------------------------ |
| Broker 地址   | 由 TSP 分配，TBOX 出厂预置                                   |
| 端口          | 8883（TLS）                                                  |
| MQTT 版本     | 3.1.1（推荐）或 5.0                                          |
| ClientId      | TBOX 设备 ID（TBOX 出厂预置，唯一）                          |
| Username      | {productKey}:{clientId}:{timestamp}，一机一密                |
| Password      | HMAC-SHA256(deviceSecret, clientId + ":" + timestamp)，一机一密 |
| Keep Alive    | 120 秒                                                       |
| Clean Session | false（MQTT 3.1.1）/ Clean Start = false（5.0）              |

#### 为什么要"一车一密"？

本协议要求每台 TBOX 使用独立的 deviceSecret 生成连接凭证（即"一车一密"），从根本上解决共享凭证带来的安全风险：

- **风险集中**：如果所有车辆共用一套用户名和密码，一旦该凭证泄露，所有车辆都将面临被非法控制的风险。一车一密将风险隔离在单台设备范围内，即使某台 TBOX 的凭证泄露，攻击者也无法借此控制其他车辆。
- **无法追踪**：使用共享凭证时，平台无法区分具体是哪辆车辆在执行操作。当出现异常或攻击时，难以进行有效的安全审计和问题溯源。一车一密使每台设备的连接行为都可独立追踪和审计。
- **难以管理**：当某辆车需要被撤销接入权限时（如车辆报废、TBOX 更换），共享凭证方案无法单独操作，只能更换所有车辆的凭证，管理成本极高。一车一密支持对单台设备进行精确的权限授予和撤销，不影响其他车辆的正常运行。

### 5.2 ClientId 规范

TBOX 使用设备 ID（deviceId）作为 MQTT ClientId，在设备注册时分配，全局唯一，满足终端唯一标识需求。TBOX 出厂时预置该 ID。

```
ClientId = TBOX 设备 ID（如 "TBOX-A1B2C3D4"）
```

> clientId 与 VIN（车架号）为独立概念。一辆车对应一个 VIN，但 TBOX 设备可能更换，因此 clientId 标识的是终端设备而非车辆。MpmEvent 中会上报 VIN 用于业务关联。

### 5.3 遗嘱消息（LWT）

TBOX 在建立 MQTT 连接时必须设置遗嘱消息，确保异常断连时 TSP 可即时感知。

| LWT 参数 | 值                                                                              |
| -------- | ------------------------------------------------------------------------------- |
| Topic    | `up/tvcp/{clientId}/event/mpm`                                                       |
| Payload  | Protobuf 编码的 Envelope，内嵌 `MpmEvent`（type=OFFLINE, reason=ABNORMAL_DISCONNECT）|
| QoS      | 1                                                                               |
| Retain   | false                                                                           |

**正常下线流程**：TBOX 主动断开连接前，先发布一条 `type=OFFLINE, reason=NORMAL_DISCONNECT` 的 MPM 消息到 `up/tvcp/{clientId}/event/mpm`，然后断开连接。此场景下遗嘱消息不会被触发。

**异常断连场景**：网络中断、电源异常等情况下，EMQX 在 Keep Alive 超时后自动发布遗嘱消息，TSP 据此标记设备离线。

### 5.4 重连策略

TBOX 断线后采用指数退避重连策略：

| 参数         | 值              |
| ------------ | --------------- |
| 初始重连间隔 | 1 秒            |
| 最大重连间隔 | 120 秒          |
| 退避因子     | 2               |
| 抖动         | ±500ms 随机抖动 |
| 重连上限     | 无限重连        |

重连序列示例：1s → 2s → 4s → 8s → 16s → 32s → 64s → 120s → 120s → ...

### 5.5 会话恢复

TBOX 使用 `Clean Session = false` 连接，EMQX 侧维护离线会话。TBOX 重连后自动恢复离线期间的 QoS 1/2 下行消息。TSP 侧对离线期间的指令设置超时机制，超时未收到 sack 则标记为执行超时。

---

## 6. Topic 设计

### 6.1 命名规范

Topic 采用 UTF-8 编码，使用正斜杠 `/` 分隔层级。

**通用格式**：

```
{direction}/{productKey}/{clientId}/{business}/{sub}
```

| 层级        | 说明                                                       |
| ----------- | ---------------------------------------------------------- |
| direction   | 消息方向：`up`（上行）或 `down`（下行）                   |
| productKey  | 产品标识，固定为 `tvcp`                                    |
| clientId    | TBOX 设备 ID                                                 |
| business    | 业务域：`telemetry` / `event` / `command` / `config` / `location` / `diagnosis` |
| sub         | 业务子分类                                                 |

**约束**：Topic 总层级不超过 6 层，单级长度不超过 64 字符。

### 6.2 Topic 全景表

#### 6.2.1 上行 Topic（TBOX → TSP）

| Topic                              | QoS | Retain | 说明                               |
| :--------------------------------- | :-- | :----- | :--------------------------------- |
| `up/tvcp/{clientId}/telemetry/realtime` | 0   | false  | 实时车况数据（RVS，高频，默认10s） |
| `up/tvcp/{clientId}/telemetry/trigger`  | 1   | false  | 触发式车况数据（TRS，信号变动触发）|
| `up/tvcp/{clientId}/event/mpm`          | 1   | false  | 设备上线/下线事件                  |
| `up/tvcp/{clientId}/event/alarm`        | 1   | false  | 告警事件（碰撞/涉水/防盗等）       |
| `up/tvcp/{clientId}/location/report`    | 1   | false  | 定位数据上报                       |
| `up/tvcp/{clientId}/diagnosis/dtc`      | 1   | false  | DTC 故障码上报                     |
| `up/tvcp/{clientId}/command/response`   | 1   | false  | 控车指令执行结果回复（ack + sack） |

#### 6.2.2 下行 Topic（TSP → TBOX）

| Topic                             | QoS | Retain | 说明             |
| :-------------------------------- | :-- | :----- | :--------------- |
| `down/tvcp/{clientId}/command/request` | 1   | false  | 控车指令下发     |
| `down/tvcp/{clientId}/config/update`   | 1   | false  | 信号采集配置下发 |

### 6.3 EMQX ACL 规则

基于 EMQX 内置 ACL 或外部数据库鉴权，配置最小权限规则：

| 角色 | 操作 | Topic 模式                     | 说明                             |
| ---- | ---- | ------------------------------ | -------------------------------- |
| TBOX | 发布 | `up/tvcp/${clientId}/#`        | 可发布自身 clientId 下所有上行   |
| TBOX | 订阅 | `down/tvcp/${clientId}/#`      | 可订阅自身 clientId 下所有下行   |
| TSP  | 发布 | `down/tvcp/+/command/#`        | 可向任意 clientId 下发指令       |
| TSP  | 发布 | `down/tvcp/+/config/#`         | 可向任意 clientId 下发配置       |
| TSP  | 订阅 | `up/tvcp/#`                    | 可订阅所有上行数据               |

> `${clientId}` 为 EMQX ACL 中的动态变量，取自 MQTT ClientId（即 TBOX 设备 ID）。

---

## 7. 消息编码

### 7.1 编码格式

所有 Topic 的 Payload 均使用 **Protocol Buffers v3** 二进制编码。相比 JSON，Protobuf 在典型车况消息场景下可减少 60%-80% 的传输体积，降低车载流量消耗和弱网传输延迟。

### 7.2 信封模式（Envelope）

所有上行和下行消息共享统一的消息信封结构。信封负责携带路由和元数据信息，业务数据通过 `payload` 字段承载。

```protobuf
// 消息信封 - 所有消息的顶层结构
message Envelope {
    uint32      version   = 1;   // 协议版本号
    MessageType type      = 2;   // 消息类型标识
    uint64      timestamp = 3;   // 消息生成时间戳（Unix 毫秒）
    bytes       payload   = 4;   // 业务消息 Protobuf 编码体
    string      task_id  = 5;   // 全局唯一事务ID（UUID v4）
    string client_id      = 6;   // 设备ID
    string device_type = 7;   // 设备类型
    map<string, string> ext = 8; // 扩展字段，用于灰度、调试等场景
}
```

采用 `bytes payload` + `MessageType type` 组合模式而非 Protobuf `oneof` 的原因：

- `bytes` 载荷支持独立版本演进，业务消息结构变更不影响信封层
- TSP 侧可根据 `type` 字段做高效路由分发，无需反序列化全部载荷
- 便于在 Kafka 桥接层做基于 type 的消息分区
- 可根据 `version` 字段选择不同版本的 Protobuf 定义进行反序列化

### 7.3 协议版本号命名规则

Envelope 中的 `version` 字段使用 **递增整数** 命名规则：

| 规则     | 说明                                                                         |
| -------- | ---------------------------------------------------------------------------- |
| 初始值   | `version = 1`                                                                |
| 递增规则 | 每次协议发生不兼容变更（Envelope 结构变更、核心枚举重编号等）时递增 1        |
| 兼容性   | TSP 和 TBOX 应向前兼容：收到未知 version 值时，记录日志但不丢弃消息，按当前最高支持版本解析 |
| 载荷版本 | payload 内部 message 不单独携带版本号，其结构由 Envelope.version 决定         |

> 当前协议版本为 **1**。后续若出现不兼容变更，版本升至 2，同时提供 v1 和 v2 两套 proto 定义供解析选择。

### 7.4 MessageType 枚举

```protobuf
enum MessageType {
    // === 车辆状态上报（上行） ===
    MSG_RVS_REPORT      = 0;    // 实时车况上报
    MSG_TRS_REPORT      = 1;    // 触发式车况上报
    MSG_MPM_EVENT       = 2;    // 上线/下线事件
    MSG_ALARM_EVENT     = 3;    // 告警事件
    MSG_LOCATION_REPORT = 4;    // 定位数据上报
    MSG_DTC_REPORT      = 5;    // DTC 故障码上报

    // === 远程控车（下行指令） ===
    MSG_CMD_REQUEST     = 100;  // 控车指令下发

    // === 远程控车（上行回复） ===
    MSG_CMD_RESPONSE    = 200;  // 控车执行结果回复（ack / sack）

    // === 配置管理（下行） ===
    MSG_CONFIG_UPDATE   = 300;  // 信号采集配置下发
}
```

分段设计：0-99 为上行上报，100-199 为下行指令，200-299 为上行回复，300-399 为下行配置。新增类型在对应段内递增。

### 7.5 时间戳

所有时间戳字段使用 `uint64` 类型，值为 Unix 毫秒时间戳（自 1970-01-01T00:00:00Z 起的毫秒数）。TBOX 应通过 GNSS 或 NTP 保持时间同步，时间偏差不超过 ±5 秒。

### 7.6 编解码流程

```
发送方:
  1. 构造业务消息 Protobuf 对象（如 CmdRequest）
  2. 将业务消息序列化为 bytes
  3. 填充 Envelope（version + type + timestamp + payload bytes）
  4. 将 Envelope 序列化为二进制，通过 MQTT 发布

接收方:
  1. 收到 MQTT 消息，反序列化为 Envelope
  2. 检查 Envelope.version，选择对应版本的 proto 定义
  3. 根据 Envelope.type 判断业务类型
  4. 将 Envelope.payload 反序列化为对应的业务消息对象
  5. 执行业务逻辑
```

---

## 8. 车辆状态上报

### 8.1 实时车况上报（RVS 事件）

**Topic**：`up/tvcp/{clientId}/telemetry/realtime`
**QoS**：0（允许少量丢失，追求低延迟）
**频率**：默认 10 秒，可通过配置动态调整

TBOX 按配置的采集周期，周期性上报车辆实时运行数据。实时数据体为 `VehicleStatus`，包含车辆全部实时信号字段。

**Envelope**：`version=1, type=MSG_RVS_REPORT`
**Payload**（`VehicleStatus`）：

```protobuf
message VehicleStatus {
    // ===== 基础行驶信息 =====
    double      speed                           = 1;    // 速度 (km/h)
    int32       direction                       = 2;    // 方向 (度, 0-359)
    EngineStatus engineStatus                   = 3;    // 发动机状态
    bool        speedValidity                   = 4;    // 车速有效位
    double      distanceToEmpty                 = 5;    // 可续航里程 (km)
    double      altitude                        = 6;    // 高度 (米)
    double      latitude                        = 7;    // 纬度 (WGS84)
    double      longitude                       = 8;    // 经度 (WGS84)
    string      marsCoordinates                 = 9;    // 显示坐标系
    bool        posCanBeTrusted                 = 10;   // 是否可信任位置
    bool        carLocatorStatUploadEn          = 11;   // 汽车定位器统计是否允许上传
    double      odometer                        = 12;   // 总里程 (km)
    BrakeStatus handBrakeStatus                 = 13;   // 手刹状态
    BrakeStatus electricParkBrakeStatus         = 14;   // 电子手刹状态
    int32       engineSpeed                     = 15;   // 引擎转速 (RPM)
    bool        brakePedalDepressed             = 16;   // 刹车踏板 (false:未踩下, true:踩下)
    UsageMode   usageMode                       = 17;   // 使用模式
    double      aveFuelConsumption              = 18;   // 平均油耗 (L/100km)
    double      avgSpeed                        = 19;   // 平均速度 (km/h)
    double      aveTraFuelConsumption           = 20;   // 50/100公里平均油耗 (L/100km)
    double      averPowerConsumption            = 21;   // 平均电耗 (kWh/100km)
    double      averTraPowerConsumption         = 22;   // 50/100公里平均电耗 (kWh/100km)
    double      odometerOnFuelOnly              = 23;   // 总里程(燃油), 单位km
    double      odometerOnBatteryOnly           = 24;   // 总里程(纯电), 单位km
    bool        engineForbidStatus              = 25;   // 发动机禁止状态 (false:未禁止, true:禁止)

    // ===== 天窗与车窗 =====
    OpenStatus  sunroofOpenStatus               = 26;   // 天窗状态
    bool        ventilateStatus                 = 27;   // 四窗透气
    OpenStatus  winStatusLeftFront              = 28;   // 左前侧车窗位置状态
    OpenStatus  winStatusLeftRear               = 29;   // 左后侧车窗位置状态
    OpenStatus  winStatusRightFront             = 30;   // 右前侧车窗位置状态
    OpenStatus  winStatusRightRear              = 31;   // 右后侧车窗位置状态
    int32       winPosLeftFront                 = 32;   // 左前车窗位置信息 (0-100%)
    int32       winPosLeftRear                  = 33;   // 左后车窗位置信息 (0-100%)
    int32       winPosRightFront                = 34;   // 右前车窗位置信息 (0-100%)
    int32       winPosRightRear                 = 35;   // 右后车窗位置信息 (0-100%)

    // ===== 车门与门锁 =====
    LockStatus  doorLockStatusLeftFront         = 37;   // 左前门锁状态
    LockStatus  doorLockStatusLeftRear          = 38;   // 左后门锁状态
    LockStatus  doorLockStatusRightFront        = 39;   // 右前门锁状态
    LockStatus  doorLockStatusRightRear         = 40;   // 右后门锁状态
    OpenStatus  doorOpenStatusLeftFront         = 41;   // 左前门状态
    OpenStatus  doorOpenStatusLeftRear          = 42;   // 左后门状态
    OpenStatus  doorOpenStatusRightFront        = 43;   // 右前门状态
    OpenStatus  doorOpenStatusRightRear         = 44;   // 右后门状态
    CentralLockStatus centralLockingStatus      = 45;   // 中控锁状态
    OpenStatus  engineHoodOpenStatus            = 46;   // 引擎盖开启/关闭

    // ===== 后备箱 =====
    LockStatus  trunkLockStatus                 = 47;   // 后备箱锁状态
    OpenStatus  trunkOpenStatus                 = 48;   // 后备箱状态

    // ===== 轮胎气压与温度 =====
    WarningStatus tyrePreWarningLeftFront       = 49;   // 左前轮胎警报状态
    WarningStatus tyrePreWarningLeftRear        = 50;   // 左后轮胎警报状态
    WarningStatus tyrePreWarningRightFront      = 51;   // 右前轮胎警报状态
    WarningStatus tyrePreWarningRightRear       = 52;   // 右后轮胎警报状态
    double      tyreStatusLeftFront             = 53;   // 左前轮胎压力值 (bar/kPa)
    double      tyreStatusLeftRear              = 54;   // 左后轮胎压力值 (bar/kPa)
    double      tyreStatusRightFront            = 55;   // 右前轮胎压力值 (bar/kPa)
    double      tyreStatusRightRear             = 56;   // 右后轮胎压力值 (bar/kPa)
    double      tyreTempLeftFront               = 57;   // 左前轮胎温度 (°C)
    double      tyreTempLeftRear                = 58;   // 左后轮胎温度 (°C)
    double      tyreTempRightFront              = 59;   // 右前轮胎温度 (°C)
    double      tyreTempRightRear               = 60;   // 右后轮胎温度 (°C)
    bool        tyreTempWarningLeftFront        = 61;   // 左前轮胎温度报警 (false:正常, true:告警)
    bool        tyreTempWarningLeftRear         = 62;   // 左后轮胎温度报警 (false:正常, true:告警)
    bool        tyreTempWarningRightFront       = 63;   // 右前轮胎温度报警 (false:正常, true:告警)
    bool        tyreTempWarningRightRear        = 64;   // 右后轮胎温度报警 (false:正常, true:告警)

    // ===== 油量与电池 =====
    double      fuelLevel                       = 65;   // 剩余油量 (%)
    double      powerBatteryLevel               = 66;   // 动力电池电量 (%)
    double      chargeLevel                     = 67;   // 蓄电池电量 (%)
    int32       storageBattery12v               = 68;   // 12V电池状态
    double      batteryState                    = 69;   // 动力电池健康状态（百分比）
    double      distanceToEmptyOnBatteryOnly    = 70;   // 纯电续航里程 (km)
    double      distanceToEmptyOnBattery100Soc  = 71;   // 百分之百电量续航里程 (km)
    double      rangeToTenPercent               = 72;   // 到10%电量可行驶里程 (km)

    // ===== 温度与空调 =====
    double      exteriorTemp                    = 73;   // 车外温度 (°C)
    double      interiorTemp                    = 74;   // 内部温度 (°C)
    int32       exteriorPM25                    = 75;   // 外部PM2.5 (μg/m³)
    int32       interiorPM25                    = 76;   // 内部PM2.5 (μg/m³)
    int32       interiorSecondPM25              = 77;   // 车内二排PM2.5浓度 (μg/m³)
    bool        preClimateActive                = 78;   // 空调是否在运行 (false:关, true:开)
    double      airConditionTemperature         = 79;   // 空调设定温度 (°C)
    int32       airConditionTime                = 80;   // 空调设定时间 (分钟)
    bool        airBlowerActive                 = 81;   // 鼓风机状态/座舱通风/座舱净化 (true:on, false:off)
    bool        defrostActive                   = 82;   // 除霜状态 (true:on, false:off)

    // ===== 灯光与鸣笛 =====
    LightStatus hazardLight                     = 83;   // 车辆双闪
    LightStatus leftHeadlights                  = 84;   // 左前大灯
    LightStatus rightHeadlights                 = 85;   // 右前大灯
    SoundStatus honking                         = 86;   // 车辆鸣笛

    // ===== 充电状态 =====
    ChargerState chargerState                   = 87;   // 充电状态
    ChargeScheduleStatus chargeSts              = 88;   // 预约充电状态
    ChargeScheduleStatus bookChargeSts          = 89;   // 充电状态(预约充电)
    HvStatus    chargeHvSts                     = 90;   // 上高压状态
    bool        isPluggedIn                     = 91;   // 是否插入充电桩
    string      chargingStartTime               = 92;   // 充电开始时间
    int32       timeToFullyCharged              = 93;   // 充电剩余时间 (分钟)
    double      chargCapacity                   = 94;   // 开始充电电量 (%)
    int32       totalChargeTime                 = 95;   // 最近一次充电所花时间, 单位:分钟
    double      totalChargeEnergy               = 96;   // 最近一次已充电电量, 单位:KWH(度)
    double      chargingDistanceToEmptyOnStandard = 97;  // 最近一次充电续航里程-标准, 单位:千米
    double      chargingDistanceToEmptyOnDynamic  = 98;  // 最近一次充电续航里程-动态, 单位:千米

    // ===== 交流充电 =====
    double      chargeIAct                      = 99;   // 交流充电电流 (A)
    double      chargeUAct                      = 100;  // 交流充电电压 (V)
    ConnectorStatus statusOfChargerConnection   = 101;  // 交流充电枪连接状态
    double      acChargePower                   = 102;  // 交流充电功率, 单位:W
    OpenStatus  acChargeLidStatus               = 103;  // AC充电盖状态

    // ===== 直流充电 =====
    double      dcChargeIAct                    = 104;  // 直流充电电流 (A)
    double      dcChargeUAct                    = 105;  // 实际直流高压电池充电电压 (V)
    ConnectorStatus dcDcConnectStatus           = 106;  // 直流充电枪连接状态
    double      dcChargePower                   = 107;  // 直流充电功率, 单位:KW
    OpenStatus  dcOrAcDcChargeLidStatus         = 108;  // DC或DCAC充电盖状态

    // ===== 无线充电 =====
    WptStatus   wptChargeSts                    = 109;  // 无线充电状态
    double      wptChargeUAct                   = 110;  // 无线充电电压 (V)
    double      wptChargeIAct                   = 111;  // 无线充电电流 (A)

    // ===== 放电 =====
    DischargeStatus disChargeSts                = 112;  // 放电状态
    DischargeConnectorStatus disChargeConnectStatus = 113;  // 放电枪插电状态
    double      disChargeIAct                   = 114;  // 放电电流 (A)
    double      disChargeUAct                   = 115;  // 放电电压 (V)
    int32       timeToTargetDisCharged          = 116;  // 放电剩余时间 (分钟)

    // ===== 预约出行与预约充电 =====
    ScheduleResultStatus btPreChargeStatus           = 117;  // 预约出行-预约充电的预约结果
    ScheduleResultStatus btPreBatteryPackHeatStatus  = 118;  // 预约出行-电池包预热的预约结果
    bool        btHvFaultActive                 = 119;  // 预约出行-高压故障 (boolean)
    bool        btActive                        = 120;  // 预约出行-总开关状态/周期出行开关 (true:on, false:off)
    bool        btTempActive                    = 121;  // 预约出行-临时出行开关状态 (true:on, false:off)
    bool        bcCycleActive                   = 122;  // 预约充电-周期开关 (true:on, false:off)
    bool        bcTempActive                    = 123;  // 预约充电-临时开关 (true:on, false:off)

    // ===== 遮阳帘 =====
    OpenStatus  curtainOpenStatus               = 124;  // 遮阳帘位置信息
    int32       curtainPos                      = 125;  // 遮阳帘位置状态百分比 (0-100%)
    OpenStatus  sunCurtainRearOpenStatus        = 126;  // 后遮阳帘位置状态
    int32       sunCurtainRearPos               = 127;  // 后遮阳帘位置状态百分比 (0-100%)

    // ===== 座椅通风 =====
    VentilationStatus drvVentSts                = 128;  // 主驾座椅通风状态
    VentilationStatus passVentSts               = 129;  // 副驾座椅通风状态
    VentilationLevel drvVentDetail              = 130;  // 主驾座椅通风等级信息
    VentilationLevel passVentDetail             = 131;  // 副驾座椅通风等级信息

    // ===== 座椅加热 =====
    HeatingStatus drvHeatSts                    = 132;  // 主驾座椅加热状态
    HeatingStatus passHeatingSts                = 133;  // 副驾座椅加热状态
    HeatingStatus rlHeatingSts                  = 134;  // 二排左侧座椅加热状态
    HeatingStatus rrHeatingSts                  = 135;  // 二排右侧座椅加热状态
    HeatingStatus lrdHeatingSts                 = 136;  // 第三排左侧座椅加热状态
    HeatingStatus rrdHeatingSts                 = 137;  // 第三排右侧座椅加热状态
    HeatingLevel drvHeatLv                      = 138;  // 主驾座椅加热等级信息
    HeatingLevel passHeatLv                     = 139;  // 副驾座椅加热等级信息
    HeatingLevel rlHeatLv                       = 140;  // 二排左侧座椅加热等级信息
    HeatingLevel rrHeatLv                       = 141;  // 二排右侧座椅加热等级信息
    HeatingLevel lrdHeatLv                      = 142;  // 第三排左侧座椅加热等级
    HeatingLevel rrdHeatLv                      = 143;  // 第三排右侧座椅加热等级

    // ===== 安全带 =====
    SeatbeltStatus seatBeltStatusLeftFront      = 144;  // 左前安全带锁扣状态
    SeatbeltStatus seatBeltStatusRightFront     = 145;  // 右前安全带锁扣状态
    SeatbeltStatus seatBeltStatusLeftRear       = 146;  // 左后安全带锁扣状态
    SeatbeltStatus seatBeltStatusRightRear      = 147;  // 右后安全带锁扣状态
    SeatbeltStatus seatBeltStatusMiddleRear     = 148;  // 中后安全带锁扣状态

    // ===== 其他 =====
    bool        rvsEnable                       = 149;  // RVS开关 (true:开启, false:关闭)
}

// ===== 枚举定义 =====

enum EngineStatus {
    ENGINE_RUNNING = 0;    // 运行
    ENGINE_STOPPED = 1;    // 关闭
}

enum BrakeStatus {
    BRAKE_RELEASED = 0;    // 释放
    BRAKE_APPLIED = 1;     // 拉起/激活
    BRAKE_FAULT = 2;       // 故障
}

enum UsageMode {
    MODE_ABANDON = 0;      // Abandon
    MODE_INACTIVE = 1;     // Inactive (锁门时)
    MODE_CONVENIENCE = 2;  // Convenience
    MODE_ACTIVE = 11;      // Active
    MODE_DRIVING = 13;     // Driving (发动机启动时)
}

enum OpenStatus {
    OPEN_STATUS_UNKNOWN = 0; // 未知
    OPEN_STATUS_OPEN = 1;    // 开
    OPEN_STATUS_CLOSE = 2;   // 关
    OPEN_STATUS_HALF = 3;    // 半开
}

enum LockStatus {
    LOCK_UNKNOWN = 0;      // 未知
    LOCK_UNLOCKED = 1;     // 开锁
    LOCK_LOCKED = 2;       // 闭锁
    LOCK_SAFE = 3;         // 安全锁
}

enum CentralLockStatus {
    CENTRAL_LOCK_UNKNOWN = 0;      // 未知
    CENTRAL_LOCK_UNLOCKED = 1;     // 解锁
    CENTRAL_LOCK_TRUNK_UNLOCKED = 2; // 后备箱解锁
    CENTRAL_LOCK_LOCKED = 3;       // 闭锁
}

enum WarningStatus {
    WARNING_NORMAL = 0;     // 正常
    WARNING_LOW_PRESSURE = 1; // 低压告警
    WARNING_HIGH_PRESSURE = 2; // 高压告警
}

enum LightStatus {
    LIGHT_UNKNOWN = 0;     // 未知
    LIGHT_ON = 1;          // 开启
    LIGHT_OFF = 2;         // 关闭
}

enum SoundStatus {
    SOUND_UNKNOWN = 0;     // 未知
    SOUND_ON = 1;          // 开启
    SOUND_OFF = 2;         // 关闭
}

enum ChargerState {
    CHARGER_DEFAULT = 0;           // 默认值
    CHARGER_NOT_CHARGING = 1;      // 未充电
    CHARGER_AC_CHARGING = 2;       // AC充电中
    CHARGER_AC_CHARGING_PAUSED = 3; // AC充电停止/充电暂停
    CHARGER_AC_CHARGING_COMPLETED = 4; // AC充电完成
    CHARGER_AC_HEATING = 5;        // AC加热中
    CHARGER_AC_SCHEDULED = 6;      // AC预约中(预约充电和预约出行共用这个信号显示）
    CHARGER_RESERVED_7 = 7;        // 预留
    CHARGER_AC_DISCHARGING = 8;    // AC放电中
    CHARGER_DISCHARGE_PAUSED = 9;  // 放电结束停止/放电暂停
    CHARGER_DISCHARGE_COMPLETED = 10; // 放电完成
    CHARGER_RESERVED_11 = 11;      // 预留
    CHARGER_DISCHARGE_FAULT = 12;  // 放电故障
    CHARGER_AC_PILE_FAULT = 14;    // AC充电故障（桩端）
    CHARGER_DC_CHARGING = 15;      // DC充电中
    CHARGER_RESERVED_16 = 16;      // 预留
    CHARGER_RESERVED_17 = 17;      // 预留
    CHARGER_DC_CAR_FAULT = 18;     // DC车端故障
    CHARGER_DC_PILE_TEMP_FAULT = 19; // DC桩端温度故障
    CHARGER_DC_PILE_CONNECT_FAULT = 20; // DC桩端链接故障
    CHARGER_DC_PILE_OTHER_FAULT = 21; // DC桩端其他故障
    CHARGER_DC_PILE_EMERGENCY_FAULT = 22; // DC桩端急停故障
    CHARGER_DC_PILE_COMM_FAULT = 23; // DC桩端通信故障
    CHARGER_EXTREME_CHARGING = 24; // 极充中
    CHARGER_AC_CHARGING_ENDED_USER = 25; // AC充电结束（由于用户按下充电停止导致的充电结束，用于复位充电停止开关，开关变为开始充电状态）
    CHARGER_DC_CHARGING_ENDED = 26; // DC充电结束
    CHARGER_AC_CAR_FAULT = 27;     // AC充电故障（车端）
    CHARGER_BOOST_CHARGING = 28;   // 升压充电中
    CHARGER_BOOST_CHARGING_FAULT = 29; // 升压充电故障
    CHARGER_WIRELESS_CHARGING = 30; // 无线充电中
}

enum ChargeScheduleStatus {
    SCHEDULE_NONE = 0;           // 无预约/默认
    SCHEDULE_PROCESSING = 1;     // 预约中/开始充电成功
    SCHEDULE_FAILED = 2;         // 开始充电失败
    SCHEDULE_COMPLETED = 3;      // 充电完成/预约充电功能关闭
    SCHEDULE_ACTIVE = 4;         // 有预约充电功能，且正在生效
}

enum HvStatus {
    HV_UNDEFINED = 0;        // Undefined
    HV_FAILED = 1;           // Failed
    HV_SUCCESSFUL = 2;       // Successful
    HV_RESERVED = 3;         // Reserved
}

enum ConnectorStatus {
    CONNECTOR_NOT_CONNECTED = 0;      // 未连接
    CONNECTOR_CHECK_STATUS_FAILED = 1; // 查看状态失败
    CONNECTOR_CHARGING_NOT_POWERED = 2; // 充电未通电
    CONNECTOR_CONNECTED_POWERED = 3;  // 连接并通电
    CONNECTOR_UNKNOWN = 7;            // 未知
}

enum WptStatus {
    WPT_DEFAULT = 0;        // 默认值
    WPT_NOT_CHARGING = 1;   // 未充电
    WPT_CHARGING = 2;       // 充电中
    WPT_CHARGING_PAUSED = 3; // 充电停止/充电暂停
    WPT_CHARGING_COMPLETED = 4; // 充电完成
    WPT_HEATING = 5;        // 加热中
    WPT_SCHEDULED = 6;      // 预约中(预约充电和预约出行共用这个信号显示）
    WPT_RESERVED = 7;       // 预留
    WPT_DISCHARGING = 8;    // 放电中
    WPT_DISCHARGE_ENDED = 9; // 放电结束
    WPT_RESERVED_10 = 10;   // 预留
    WPT_CHARGE_FAULT = 11;  // 充电故障
    WPT_DISCHARGE_FAULT = 12; // 放电故障
}

enum DischargeStatus {
    DISCHARGE_DEFAULT = 0;         // 默认值
    DISCHARGE_NOT_CHARGING = 1;    // 未充电
    DISCHARGE_CHARGING = 2;        // 充电中
    DISCHARGE_PAUSED = 3;          // 充电停止/充电暂停
    DISCHARGE_COMPLETED = 4;       // 充电完成
    DISCHARGE_HEATING = 5;         // 加热中
    DISCHARGE_SCHEDULED = 6;       // 预约中(预约充电和预约出行共用这个信号显示）
    DISCHARGE_RESERVED = 7;        // 预留
    DISCHARGE_DISCHARGING = 8;     // 放电中
    DISCHARGE_DISCHARGE_ENDED = 9; // 放电结束
    DISCHARGE_RESERVED_10 = 10;    // 预留
    DISCHARGE_CHARGE_FAULT = 11;   // 充电故障
    DISCHARGE_DISCHARGE_FAULT = 12; // 放电故障
}

enum DischargeConnectorStatus {
    DISCHARGE_CONN_NOT_CONNECTED = 0;     // 充/放电枪未连接
    DISCHARGE_CONN_PLUGGED_NO_CARD = 1;   // 插上了充电枪但没有刷卡
    DISCHARGE_CONN_PLUGGED_READY = 2;     // 插上了充电枪且刷了卡，但交流电还没有输入
    DISCHARGE_CONN_PLUGGED_POWERED = 3;   // 检测到充电枪连接，且有交流电输入
    DISCHARGE_CONN_RESERVED_INCAR = 4;    // 预留，车内放电
    DISCHARGE_CONN_DETECTED_NO_OUTPUT = 5; // 检测到放电枪连接，但没有电压输出
    DISCHARGE_CONN_RESERVED_INCAR2 = 6;   // 预留，车内放电
    DISCHARGE_CONN_DETECTED_OUTPUT = 7;   // 检测到放电枪连接，且有电压输出
    DISCHARGE_CONN_INIT = 8;              // Init（初始化）
    DISCHARGE_CONN_FAULT = 9;             // Fault 检测到枪线故障
    DISCHARGE_CONN_HALF_CONNECTED = 10;   // NotCompleteConnnected 检测到充电或放电枪处于半连接状态
}

enum ScheduleResultStatus {
    SCHEDULE_RESULT_DEFAULT = 0;      // Default / BookChargeSetResponse_Default
    SCHEDULE_RESULT_SUCCESS = 1;      // BookChargeSetResponse_Success
    SCHEDULE_RESULT_CANCELLED = 2;    // BookChargeSetResponse_Cancelled
    SCHEDULE_RESULT_FAILED = 3;       // BookChargeSetResponse_Fail
}

enum VentilationStatus {
    VENT_OFF = 1;        // 关
    VENT_ON = 2;         // 开
    VENT_ERROR = 3;      // 错误
    VENT_FUNCTION_LIMITED = 4; // 功能受限
    VENT_ENERGY_LIMITED = 5;   // 能量受限
}

enum VentilationLevel {
    VENT_LEVEL_OFF = 0;      // 关
    VENT_LEVEL_1 = 1;        // level1
    VENT_LEVEL_2 = 2;        // level2
    VENT_LEVEL_3 = 3;        // level3
}

enum HeatingStatus {
    HEAT_OFF = 1;            // 关
    HEAT_ON = 2;             // 开
    HEAT_ERROR = 3;          // 错误
    HEAT_FUNCTION_LIMITED = 4; // 功能限制
    HEAT_ENERGY_LIMITED = 5;   // 能量限制
}

enum HeatingLevel {
    HEAT_LEVEL_OFF = 0;      // 关
    HEAT_LEVEL_1 = 1;        // level1
    HEAT_LEVEL_2 = 2;        // level2
    HEAT_LEVEL_3 = 3;        // level3
}

enum SeatbeltStatus {
    SEATBELT_LOCKED = 0;     // 带扣锁
    SEATBELT_UNLOCKED = 1;   // 带扣解锁
    SEATBELT_FAULT = 2;      // 故障
}
```

> **字段说明**：`VehicleStatus` 中信号字段采用优化的数据类型：测量值（如速度、温度、电量等）使用 `double` 类型，计数/状态值使用 `int32` 类型，开关量使用 `bool` 类型，状态枚举使用专用枚举类型（如 EngineStatus、OpenStatus、LockStatus 等），其余文本信息使用 `string` 类型。TSP 侧按相应数据类型进行业务解析。

### 8.2 触发式车况上报（TRS 事件）

**Topic**：`up/tvcp/{clientId}/telemetry/trigger`
**QoS**：1（确保送达）
**触发条件**：下方列出的监控信号值发生变动时立即上报

TBOX 监听下表中的信号项，当任一信号值发生变化时，立即以当前完整的 `VehicleStatus` 作为 Payload 上报。采用完整 VehicleStatus 而非仅上报变动字段，使 TSP 无需合并历史数据即可获得完整车况快照。

**Envelope**：`version=1, type=MSG_TRS_REPORT`
**Payload**（`VehicleStatus`）：与 8.1 RVS 上报使用相同的 VehicleStatus 结构，包含车辆全部实时信号字段。

#### 触发信号清单

| 功能域       | 触发信号                               | 字段编号 | 类型           | 说明             |
| ------------ | -------------------------------------- | -------- | -------------- | ---------------- |
| 车门开关     | doorOpenStatusLeftFront                | 41       | OpenStatus     | 左前门开/关      |
|              | doorOpenStatusLeftRear                 | 42       | OpenStatus     | 左后门开/关      |
|              | doorOpenStatusRightFront               | 43       | OpenStatus     | 右前门开/关      |
|              | doorOpenStatusRightRear                | 44       | OpenStatus     | 右后门开/关      |
| 车锁         | doorLockStatusLeftFront                | 37       | LockStatus     | 左前门锁状态     |
|              | doorLockStatusLeftRear                 | 38       | LockStatus     | 左后门锁状态     |
|              | doorLockStatusRightFront               | 39       | LockStatus     | 右前门锁状态     |
|              | doorLockStatusRightRear                | 40       | LockStatus     | 右后门锁状态     |
|              | centralLockingStatus                   | 45       | CentralLockStatus | 中控锁状态     |
| 车窗开关     | winStatusLeftFront                     | 28       | OpenStatus     | 左前车窗开/关    |
|              | winStatusLeftRear                      | 29       | OpenStatus     | 左后车窗开/关    |
|              | winStatusRightFront                    | 30       | OpenStatus     | 右前车窗开/关    |
|              | winStatusRightRear                     | 31       | OpenStatus     | 右后车窗开/关    |
| 天窗开关     | sunroofOpenStatus                      | 26       | OpenStatus     | 天窗开/关        |
| 后备箱       | trunkOpenStatus                        | 48       | OpenStatus     | 后备箱开/关      |
| 后备箱锁     | trunkLockStatus                        | 47       | LockStatus     | 后备箱锁状态     |
| 发动机       | engineStatus                           | 3        | EngineStatus   | 发动机运行/关闭  |
| 空调         | preClimateActive                       | 78       | bool           | 空调运行/关闭    |

> **说明**：上表仅列出触发上报的监控信号，即这些信号值发生变化时 TBOX 应立即发起一次 TRS 上报。上报的 Payload 为完整的 VehicleStatus，包含车辆全部信号字段，不限于上表所列。

### 8.3 设备上下线事件（MPM 事件）

**Topic**：`up/tvcp/{clientId}/event/mpm`
**QoS**：1

TBOX 上线和下线均通过此 Topic 上报。上线时 TBOX 在 MQTT 连接建立成功后立即发送；下线分两种情况：正常下线由 TBOX 主动发送，异常下线通过 LWT 遗嘱消息自动触发。

**Envelope**：`version=1, type=MSG_MPM_EVENT`
**Payload**（`MpmEvent`）：

```protobuf
message MpmEvent {
    MpmType          type         = 1;   // ONLINE 或 OFFLINE
    PowerMode        power_mode   = 2;   // 电源模式
    DisconnectReason reason       = 3;   // 下线原因（仅 OFFLINE 时有效）
    string           vin          = 4;   // 车架号
    string           iccid        = 5;   // SIM 卡 ICCID
    string           firmware_ver = 6;   // TBOX 固件版本号
    string         device_type = 7;   // 设备类型
}

enum MpmType {
    ONLINE  = 0;
    OFFLINE = 1;
}

enum PowerMode {
    NORMAL  = 0;
    STANDBY = 1;
    SLEEP_POLL = 2;
    OFF = 3;
}

enum DisconnectReason {
    NORMAL_DISCONNECT   = 0;   // 正常下线
    ABNORMAL_DISCONNECT = 1;   // 异常断连
    NETWORK_TIMEOUT     = 2;   // 网络超时
    POWER_OFF_EVENT     = 3;   // 断电
}
```

### 8.4 告警事件

**Topic**：`up/tvcp/{clientId}/event/alarm`
**QoS**：1

车辆发生安全相关事件时立即上报。

**Envelope**：`version=1, type=MSG_ALARM_EVENT`
**Payload**（`AlarmEvent`）：

```protobuf
message AlarmEvent {
    AlarmType            alarm_type    = 1;   // 告警类型
    Severity             severity      = 2;   // 严重程度
    VehicleStatus        vehicle_status = 3;  // 告警发生时的完整车况快照
    Location             location      = 4;   // 告警发生时的位置信息
}

enum AlarmType {
    ALARM_COLLISION        = 0;   // 碰撞
    ALARM_WATER_INGRESSION = 1;   // 涉水
    ALARM_THEFT            = 2;   // 防盗触发
    ALARM_GEOFENCE         = 3;   // 地理围栏
    ALARM_OVERSPEED        = 4;   // 超速
    ALARM_TOWING           = 5;   // 拖车
}

enum Severity {
    SEVERITY_INFO     = 0;
    SEVERITY_WARNING  = 1;
    SEVERITY_CRITICAL = 2;
}
```

### 8.5 定位数据上报

**Topic**：`up/tvcp/{clientId}/location/report`
**QoS**：1

**Envelope**：`version=1, type=MSG_LOCATION_REPORT`
**Payload**（`LocationReport`）：

```protobuf
message LocationReport {
    Location       location        = 1;
    double         speed_kmh       = 2;   // 车速 (km/h)
    double         heading_deg     = 3;   // 航向角 (度, 0-360)
    double         altitude_m      = 4;   // 海拔 (米)
    uint32         satellite_count = 5;   // 可见卫星数
    LocationSource source          = 6;   // 定位源
}

message Location {
    double latitude  = 1;   // 纬度 (WGS84)
    double longitude = 2;   // 经度 (WGS84)
    uint64 timestamp = 3;   // GPS 时间戳（毫秒）
}

enum LocationSource {
    LOC_GPS     = 0;
    LOC_BEIDOU  = 1;
    LOC_GLONASS = 2;
    LOC_DR      = 3;   // 航位推算
}
```

### 8.6 DTC 故障码上报

**Topic**：`up/tvcp/{clientId}/diagnosis/dtc`
**QoS**：1

**Envelope**：`version=1, type=MSG_DTC_REPORT`
**Payload**（`DtcReport`）：

```protobuf
message DtcReport {
    repeated DtcInfo dtcs        = 1;
    uint32           total_count = 2;   // 当前 DTC 总数
}

message DtcInfo {
    string   dtc_code         = 1;   // DTC 编码（如 "P0300"）
    string   description      = 2;   // 故障描述
    Severity severity         = 3;
    uint32   occurrence_count = 4;   // 发生次数
    uint64   first_seen       = 5;   // 首次出现时间戳
    uint64   last_seen        = 6;   // 最近出现时间戳
}
```

---

## 9. 远程控车指令

### 9.1 三段式应答机制

所有远程控车指令采用三段式应答机制，确保指令全链路可追踪：

```
  TSP                          EMQX                         TBOX
   │                             │                            │
   │─── CmdRequest (taskId) ───▶ │ ─── down/.../request ───▶ │
   │                             │                            │
   │                             │ ◀── up/.../response ───── │
   │◀── CmdResponse (ack) ──────│      phase=ACK             │
   │                             │                            │
   │        [TBOX 执行指令]       │                            │
   │                             │                            │
   │                             │ ◀── up/.../response ───── │
   │◀── CmdResponse (sack) ─────│      phase=SACK            │
   │                             │                            │
```

**Phase 1 — 指令下发**：TSP 生成全局唯一 `taskId`（UUID v4），构造 `CmdRequest` 消息，发布到 `down/tvcp/{clientId}/command/request`。

**Phase 2 — 接收确认（ack）**：TBOX 收到指令后，校验 taskId 去重，立即回复 `CmdResponse`（phase=ACK）。

**Phase 3 — 执行结果（sack）**：TBOX 完成指令执行后，回复 `CmdResponse`（phase=SACK），携带执行结果和指令特定的结果数据。

**超时处理**：TSP 对每条指令设置超时计时器（默认 30 秒）。若超时未收到 ack，标记为"投递超时"；若收到 ack 但未收到 sack，标记为"执行超时"。

### 9.2 指令下发（CmdRequest）

**Topic**：`down/tvcp/{clientId}/command/request`
**QoS**：1

**Envelope**：`version=1, type=MSG_CMD_REQUEST`
**Payload**（`CmdRequest`）：

| 字段         | 类型        | 说明                                          |
| ------------ | ----------- | --------------------------------------------- |
| service_key  | string      | 指令标识（如 `"RDU-ON"`），TBOX 据此分发执行  |
| params       | bytes       | 指令参数（Protobuf 编码）                     |
| timeout_sec  | uint32      | 指令执行超时时间（秒），默认 30               |
| created_at   | uint64      | 指令创建时间戳（毫秒）                        |

```protobuf
message CmdRequest {
    string      service_key = 1;   // 指令标识，如 "RDU-ON"
    bytes       params      = 2;
    uint32      timeout_sec = 3;
    uint64      created_at  = 4;
}
```

### 9.3 执行结果回复（CmdResponse）

**Topic**：`up/tvcp/{clientId}/command/response`
**QoS**：1

**Envelope**：`version=1, type=MSG_CMD_RESPONSE`
**Payload**（`CmdResponse`）：

| 字段            | 类型           | 说明                                          |
| --------------- | -------------- | --------------------------------------------- |
| service_key     | string         | 对应指令的 service_key（与 CmdRequest 一致）  |
| operation_trigger| string         | 操作触发源（如 `"ihu"`） |
| vehicle_status  | VehicleStatus  | 执行后车况快照（仅 sack 携带，结构同 8.1）    |
| phase           | ResponsePhase  | `ACK`（接收确认）或 `SACK`（执行结果）        |
| result_code     | ResultCode     | 结果码                                        |
| message         | string         | 可读的结果描述                                |

```protobuf
message CmdResponse {
    string        service_key        = 1;   // 对应指令的 service_key
    string        operation_trigger  = 2;   // 操作触发源
    VehicleStatus vehicle_status     = 3;   // 执行后车况快照（仅 sack 携带）
    ResponsePhase phase              = 4;
    ResultCode    result_code        = 5;
    string        message            = 6;
}

enum ResponsePhase {
    ACK  = 0;   // 接收确认
    SACK = 1;   // 执行结果
}

enum ResultCode {
    SUCCESS              = 0;   // 成功
    ERR_UNSUPPORTED_CMD  = 1;   // 不支持的指令
    ERR_INVALID_PARAMS   = 2;   // 参数错误
    ERR_EXECUTION_FAILED = 3;   // 执行失败
    ERR_VEHICLE_STATE    = 4;   // 车辆状态不允许
    ERR_TIMEOUT          = 5;   // 执行超时
    ERR_DUPLICATE_TASK   = 6;   // 重复任务（幂等去重命中）
}
```

### 9.4 serviceKey 值定义

指令通过 `service_key` 字符串标识，不使用枚举类型。TBOX 根据 `service_key` 字符串值分发执行对应操作。

| serviceKey         | 说明               |
| ------------------ | ------------------ |
| `RDU-ON`           | 远程解锁全部车门   |
| `RDL-OFF`          | 远程关闭全部车门   |
| `RDL-TargetOFF`    | 远程解锁后备箱     |
| `RES-Engine-ON`    | 远程启动发动机     |
| `RES-Engine-OFF`   | 远程关闭发动机     |
| `RHL-HornAndLight` | 远程闪灯鸣笛       |
| `RCE-ON`           | 远程开启空调       |
| `RCE-OFF`          | 远程关闭空调       |
| `RCE-Temperature`  | 空调温度设置       |

> TBOX 收到未识别的 `service_key` 时，应回复 `result_code = ERR_UNSUPPORTED_CMD`，不执行任何操作。

### 9.5 各指令参数与结果

#### 9.5.1 RDU — 远程解锁全部车门

**serviceKey**：`RDU-ON`

**CmdRequest.params → `RduParams`**：无额外参数。

**CmdResponse.vehicle_status（sack）**：返回 `VehicleStatus`（同 8.1 定义），必须包含下面字段：

| 字段 | 类型 | 说明 | 期望值（解锁后） |
|------|------|------|------------------|
| `doorLockStatusLeftFront` | LockStatus | 左前门锁状态 | LOCK_UNLOCKED (1) |
| `doorLockStatusLeftRear` | LockStatus | 左后门锁状态 | LOCK_UNLOCKED (1) |
| `doorLockStatusRightFront` | LockStatus | 右前门锁状态 | LOCK_UNLOCKED (1) |
| `doorLockStatusRightRear` | LockStatus | 右后门锁状态 | LOCK_UNLOCKED (1) |
| `centralLockingStatus` | CentralLockStatus | 中控锁状态 | CENTRAL_LOCK_UNLOCKED (1) |
```protobuf
message RduParams {}
```

#### 9.5.2 RDL — 远程关闭全部车门

**serviceKey**：`RDL-OFF`

**CmdRequest.params → `RdlParams`**：无额外参数。

**CmdResponse.vehicle_status（sack）**：返回 `VehicleStatus`（同 8.1 定义），必须包含下面字段：

| 字段 | 类型 | 说明 | 期望值（闭锁后） |
|------|------|------|------------------|
| `doorLockStatusLeftFront` | LockStatus | 左前门锁状态 | LOCK_LOCKED (2) |
| `doorLockStatusLeftRear` | LockStatus | 左后门锁状态 | LOCK_LOCKED (2) |
| `doorLockStatusRightFront` | LockStatus | 右前门锁状态 | LOCK_LOCKED (2) |
| `doorLockStatusRightRear` | LockStatus | 右后门锁状态 | LOCK_LOCKED (2) |
| `centralLockingStatus` | CentralLockStatus | 中控锁状态 | CENTRAL_LOCK_LOCKED (3) |

```protobuf
message RdlParams {}
```

#### 9.5.3 RTU — 远程解锁后备箱

**serviceKey**：`RDL-TargetOFF`

**CmdRequest.params → `RtuParams`**：无额外参数。

**CmdResponse.vehicle_status（sack）**：返回 `VehicleStatus`（同 8.1 定义），必须包含下面字段：

| 字段 | 类型 | 说明 | 期望值（解锁后） |
|------|------|------|------------------|
| `trunkLockStatus` | LockStatus | 后备箱锁状态 | LOCK_UNLOCKED (1) |
| `trunkOpenStatus` | OpenStatus | 后备箱开启状态 | |

```protobuf
message RtuParams {}
```

#### 9.5.4 RES — 远程启动/关闭发动机

**serviceKey**：启动=`RES-Engine-ON`，关闭=`RES-Engine-OFF`

**CmdRequest.params → `ResParams`**：

```protobuf
message ResParams {
    uint32 duration_min = 1;   // 运行时长（分钟），0=使用默认值（15分钟）
}
```

**CmdResponse.vehicle_status（sack）**：返回 `VehicleStatus`（同 8.1 定义），必须包含下面字段：

| 字段 | 类型 | 说明 | 期望值（启动后） |
|------|------|------|------------------|
| `engineStatus` | EngineStatus | 发动机状态 | ENGINE_RUNNING (0) |
| `engineSpeed` | int32 | 发动机转速 (RPM) | > 0 |
| `engineForbidStatus` | bool | 发动机禁止状态 | false |

#### 9.5.5 RHL — 远程闪灯鸣笛

**serviceKey**：`RHL-HornAndLight`

**CmdRequest.params → `RhlParams`**：

```protobuf
message RhlParams {
    uint32 repeat_count = 1;   // 重复次数，默认 3
    uint32 duration_ms  = 2;   // 单次持续时间（毫秒），默认 500
}
```

#### 9.5.6 RCE — 远程空调控制

**开启空调 serviceKey**：`RCE-ON`

```protobuf
message RceOnParams {
    double   target_temp = 1;   // 目标温度 (°C)，默认 24.0
    FanSpeed fan_speed   = 2;   // 风速，默认 AUTO
    bool     auto_mode   = 3;   // 是否自动模式，默认 true
}

enum FanSpeed {
    FAN_AUTO   = 0;
    FAN_LOW    = 1;
    FAN_MEDIUM = 2;
    FAN_HIGH   = 3;
}
```

**关闭空调 serviceKey**：`RCE-OFF`

```protobuf
message RceOffParams {}
```

**设置温度 serviceKey**：`RCE-Temperature`

```protobuf
message RceTempParams {
    double target_temp = 1;   // 目标温度 (°C)
}
```

**CmdResponse.vehicle_status（sack）**：返回 `VehicleStatus`（同 8.1 定义），必须包含下面字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `preClimateActive` | bool | 空调运行状态 |
| `interiorTemp` | double | 车内当前温度 (°C) |
| `airConditionTemperature` | double | 空调设定温度 (°C) |
| `airBlowerActive` | bool | 风机状态 |
| `engineStatus` | EngineStatus | 引擎状态 |

### 9.6 幂等与去重

TBOX 侧维护一个 **taskId 去重窗口**，确保同一 taskId 的指令不被重复执行。

| 参数         | 值       |
| ------------ | -------- |
| 去重窗口大小 | 30 分钟  |
| 存储方式     | 内存 LRU |
| 淘汰策略     | 超过窗口自动淘汰 |

**处理逻辑**：

1. TBOX 收到 `CmdRequest`，提取 `task_id`。
2. 查询去重窗口内是否已存在该 `task_id`。
3. 若已存在：回复 `CmdResponse`（phase=ACK, result_code=ERR_DUPLICATE_TASK），不执行指令。
4. 若不存在：记录 `task_id`，回复 ack（phase=ACK, result_code=SUCCESS），执行指令，完成后回复 sack（phase=SACK）。

---

## 10. 信号采集配置

### 10.1 配置下发

**Topic**：`down/tvcp/{clientId}/config/update`
**QoS**：1

TSP 可通过此 Topic 动态下发信号采集配置（YAML 格式），TBOX 收到后更新本地采集策略。

**Envelope**：`version=1, type=MSG_CONFIG_UPDATE`
**Payload**（`ConfigUpdate`）：

```protobuf
message ConfigUpdate {
    string     config_id      = 1;   // 配置版本 ID
    ConfigType config_type    = 2;   // 配置类型
    bytes      config_data    = 3;   // 配置数据（YAML 格式）
    uint64     effective_time = 4;   // 生效时间戳（毫秒），0=立即生效
}

enum ConfigType {
    CONFIG_RVS = 0;   // RVS 实时上报配置
    CONFIG_TRS = 1;   // TRS 触发上报配置
}
```

### 10.2 RVS 配置格式（YAML）

`config_data` 字段使用 YAML 格式编码 RVS 采集配置：

```yaml
# RVS 实时车况采集配置
rvs_config:
  version: "1.0"
  period_sec: 10              # 采集周期（秒）
  signals:
    - name: VehicleSpeed
      can_id: "0x100"
      start_bit: 0
      length: 16
      scale: 0.1
      offset: 0
      unit: "km/h"
    - name: EngineRPM
      can_id: "0x100"
      start_bit: 16
      length: 16
      scale: 0.25
      offset: 0
      unit: "RPM"
    - name: BatteryVoltage
      can_id: "0x200"
      start_bit: 0
      length: 16
      scale: 0.01
      offset: 0
      unit: "V"
```

### 10.3 TRS 配置格式（YAML）

```yaml
# TRS 触发式车况采集配置
trs_config:
  version: "1.0"
  signals:
    - name: BCM_CentralLockSWSts
      can_id: "0x300"
      start_bit: 0
      length: 4
      trigger: ON_CHANGE      # 值变化时触发
    - name: DoorOpenStatus
      can_id: "0x301"
      start_bit: 0
      length: 8
      trigger: ON_CHANGE
  min_interval_sec: 1         # 最小上报间隔（防抖）
```

### 10.4 配置确认

TBOX 成功应用配置后，通过 `up/tvcp/{clientId}/command/response` 回复确认消息：

```protobuf
message ConfigAck {
    string     config_id   = 1;
    ResultCode result_code = 2;
    string     message     = 3;
}
```

---

## 11. 安全设计

### 11.1 传输安全

| 项目     | 要求                                                       |
| -------- | ---------------------------------------------------------- |
| TLS 版本 | TLS 1.2 及以上                                             |
| 加密套件 | TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 及以上               |
| 证书     | TSP 侧服务端证书，TBOX 侧可选 mTLS                        |
| CA       | TSP 签发设备证书或使用预共享密钥                           |

### 11.2 认证鉴权

| 项目      | 说明                                                  |
| --------- | ----------------------------------------------------- |
| 设备认证  | 一机一密， Username/Password                          |
| EMQX 鉴权 | 基于 MySQL/Redis 的设备认证插件                       |
| ACL 鉴权  | 基于 clientId 的 Topic 发布/订阅权限控制（见 6.3 节） |

### 11.3 指令签名（可选）

对安全等级较高的控车指令（如 RDU、RES），TSP 可在指令中附加签名，TBOX 校验后执行。

```protobuf
message CmdSignature {
    string algorithm = 1;   // 签名算法，如 "HMAC-SHA256"
    bytes  signature = 2;   // 签名值
    uint64 nonce     = 3;   // 防重放随机数
    uint64 expire_at = 4;   // 签名过期时间戳（毫秒）
}
```

签名计算方式：`HMAC-SHA256(secret_key, task_id + service_key + params + nonce)`

TBOX 校验流程：

1. 检查 `expire_at` 未过期（5 分钟窗口）。
2. 检查 `nonce` 未使用过。
3. 计算 HMAC 并与 `signature` 比对。

---

## 12. 错误码定义

### 12.1 指令结果码（ResultCode）

| 值 | 枚举名              | 说明                     |
| -- | ------------------- | ------------------------ |
| 0  | SUCCESS             | 成功                     |
| 1  | ERR_UNSUPPORTED_CMD | 不支持的指令类型         |
| 2  | ERR_INVALID_PARAMS  | 指令参数错误             |
| 3  | ERR_EXECUTION_FAILED| 指令执行失败             |
| 4  | ERR_VEHICLE_STATE   | 当前车辆状态不允许执行   |
| 5  | ERR_TIMEOUT         | 执行超时                 |
| 6  | ERR_DUPLICATE_TASK  | 重复任务（幂等去重命中） |

### 12.2 MQTT 连接错误

| 值 | 说明               |
| -- | ------------------ |
| 1  | 协议版本不支持     |
| 2  | ClientId 被拒绝    |
| 3  | 服务不可用         |
| 4  | 用户名/密码错误    |
| 5  | 未授权             |

---

## 附录 A — Protobuf 定义文件

完整的 Protobuf 定义见同目录下的 `tvcp-protocol-v2.proto` 文件。

---

## 附录 B — serviceKey 映射表

TSP 下发控车指令时使用的 `service_key` 值及其对应操作如下：

| serviceKey         | 说明               |
| ------------------ | ------------------ |
| `RDU-ON`           | 远程解锁全部车门   |
| `RDL-OFF`          | 远程关闭全部车门   |
| `RDL-TargetOFF`    | 远程解锁后备箱     |
| `RES-Engine-ON`    | 远程启动发动机     |
| `RES-Engine-OFF`   | 远程关闭发动机     |
| `RHL-HornAndLight` | 远程闪灯鸣笛       |
| `RCE-ON`           | 远程开启空调       |
| `RCE-OFF`          | 远程关闭空调       |
| `RCE-Temperature`  | 空调温度设置       |

> TSP 侧在构造 `CmdRequest` 时，将 `service_key` 设置为对应的 serviceKey 字符串值。TBOX 侧根据 `service_key` 字符串分发处理，无需枚举映射。
