## MQTT 协议概述

MQTT（Message Queuing Telemetry Transport）最早由 IBM 的 Andy Stanford-Clark 和 Arcom（现 Cirrus Link）的 Arlen Nipper 于 1999 年联合发明，最初用于石油管道远程监控——一个对带宽和功耗极度受限的场景。2013 年，MQTT 被提交至 OASIS 标准化组织，经过社区讨论和修订后，3.1.1 版本于 2014 年成为 OASIS 标准，同年也被 ISO 采纳为 ISO/IEC 20922 标准。MQTT 5.0 于 2019 年发布，是十年来最重要的协议版本更新。

MQTT 本质上是一种 **轻量级发布/订阅消息协议**，运行在 TCP/IP 协议栈之上。它的设计哲学极度节俭：协议固定头最小仅 2 字节，能够以极低的网络开销完成消息的发布与订阅。这一特性使 MQTT 在物联网（IoT）、车联网、移动消息推送和即时通讯等低带宽、高延迟或不稳定网络环境中成为事实上的标准选型。

MQTT 的核心特点可以归纳为以下几点：

- **轻量高效**：最小报文头部仅 2 字节，对网络往返次数和消息体开销均做了极致压缩。
- **发布/订阅模型**：消息发送者（Publisher）与消息接收者（Subscriber）通过 Broker 解耦，无需知晓对方的存在。
- **一对多分发**：一条消息可以被多个订阅方同时接收，天然支持广播场景。
- **低带宽低功耗**：协议设计考虑了受限设备的处理能力和电池续航，保活机制（PING）极低能耗。
- **可靠投递**：提供三级服务质量（QoS），覆盖从"即发即弃"到"恰好一次"的可靠度需求。

### MQTT 3.1.1 与 MQTT 5.0 的主要差异

MQTT 5.0 不是对 3.1.1 的革命性重构，而是一次重大的渐进增强。二者的兼容性良好，但 5.0 在以下方面做了显著扩展：

| 维度 | MQTT 3.1.1 | MQTT 5.0 |
|------|-------------|----------|
| 原因码 | 仅有固定的 ACK 语义 | 丰富的 Reason Code，支持错误原因描述 |
| 用户属性 | 不支持 | 支持 User Properties 键值对扩展 |
| 消息过期间隔 | 无 | 支持设置消息过期时间 |
| 主题别名 | 无 | 支持 Topic Alias 压缩长主题 |
| 订阅选项 | 仅 QoS 设置 | 新增 No Local、Retain As Published、Retain Handling |
| 请求/响应 | 无 | 支持 Correlation Data 与 Response Topic |
| 流控 | 无 | 支持 Receive Maximum 流控 |
| 会话到期 | Clean Session 开关 | Session Expiry Interval 精确控制 |
| 遗留属性 | Will Message 无额外属性 | Will 支持 User Properties 和 Delay Interval |

对于新项目，建议直接采用 MQTT 5.0；而 3.1.1 在存量设备生态中仍占有巨大份额，理解两者的差异有助于兼容性设计。

---

## 发布/订阅模型

MQTT 的发布/订阅模型是整个协议的基石。与传统的点对点消息队列不同，MQTT 中的消息生产者（Publisher）和消息消费者（Subscriber）完全解耦，二者唯一的交集是通过 Broker 中转。

### Broker 的角色

Broker 是 MQTT 架构中的消息中枢，承担以下职责：

1. **连接管理**：接受客户端的 CONNECT 请求，维护会话状态和心跳保活。
2. **主题路由**：根据消息的 Topic 将其分发给所有匹配的订阅者。
3. **会话持久化**：对持久会话存储离线消息，待客户端上线后重投。
4. **遗嘱与保留消息**：触发 Will Message 机制并维护最后的保留消息。

常见的开源 Broker 实现包括 EMQX、Mosquitto、VerneMQ 和 HiveMQ。

### Topic 主题层级结构

MQTT 的 Topic 使用斜杠（`/`）作为层级分隔符，形成类似文件目录的树状结构：

```
sensor/temperature/room1
sensor/humidity/room2
device/123/status
```

Topic 可以包含任意层级的路径段，每段可以是字母、数字或特定符号的组合。Topic 没有预先注册的概念——发布者可以向任意 Topic 发送消息，订阅者也可以订阅任意 Topic。但在生产实践中，应当建立统一的 Topic 命名规范，避免混乱。

### 通配符订阅

MQTT 的一个关键优势是支持通配符订阅，允许客户端通过一个订阅表达式匹配多个 Topic。

**单级通配符 `+`**：匹配一个完整层级，但不跨级。

```
sensor/+/room1
```

上述订阅可以匹配 `sensor/temperature/room1` 和 `sensor/humidity/room1`，但不会匹配 `sensor/temperature/room1/extra`。

**多级通配符 `#`**：匹配当前层级及之后的所有层级，必须出现在 Topic 末尾。

```
sensor/#
```

上述订阅可以匹配 `sensor/temperature/room1`、`sensor/humidity`、`sensor/status/error` 等所有以 `sensor/` 开头的 Topic。

通配符的存在使 MQTT 可以非常灵活地组织消息的路由规则，但也带来了性能层面的挑战——当通配符订阅的数量非常大时，Broker 的主题匹配算法需要做高效的树状索引来保证吞吐。

### 解耦特性

发布/订阅模型为系统架构带来的核心好处是解耦：

- **空间解耦**：发布者与订阅者无需知道彼此的地址。
- **时间解耦**：订阅者离线时，持久化会话可以暂存消息，上线后重投。
- **同步解耦**：消息的生产和消费在时间上可以不对称，发布者无需等待消费者处理完毕。

这些特性使 MQTT 非常适合设备网络不稳定、拓扑频繁变化的场景。

---

## MQTT 报文结构

MQTT 报文的整体结构分为三层：固定头（Fixed Header）、可变头（Variable Header）和有效载荷（Payload）。这种分层设计兼顾了解析效率和扩展能力。

### 固定头

固定头是所有 MQTT 报文必须具备的部分，最小长度 2 字节：

```
  Bit     7   6   5   4   3   2   1   0
Byte 1   报文类型    Flags 标志位
Byte 2   剩余长度（Remaining Length）
Byte 3+  剩余长度继续字节（编码后）
```

**报文类型**占据第 1 字节的高 4 位，可表示 14 种报文类型（0 和 15 为保留）。**标志位**占据低 4 位，其含义因报文类型而异，对 PUBLISH 报文来说这 4 位分别为 DUP、QoS 高、QoS 低、RETAIN。

**剩余长度**采用变长编码，使用 1~4 个字节表示，每字节的最高位标识是否还有后续字节，剩余 7 位存储实际数值。这种编码方式支持的最大报文长度为 256 MB，对于 IoT 场景来说绰绰有余。

```
剩余长度编码示例：
值 0~127      → 1 字节 (0b0xxxxxxx)
值 128~16383  → 2 字节
值 16384~2097151 → 3 字节
最大 268435455   → 4 字节
```

### 可变头

可变头的内容因报文类型而异。以 PUBLISH 报文为例，可变头包含：

- **Topic Name**：UTF-8 编码的字符串，以 2 字节长度前缀开头。
- **Packet Identifier**（报文标识符）：2 字节整型，用于 QoS 1 和 QoS 2 的报文确认匹配。QoS 0 的 PUBLISH 无此字段。

对于 PUBACK、SUBSCRIBE、UNSUBSCRIBE 等交互报文，Packet Identifier 是识别请求-响应配对的关键机制。

### 有效载荷

有效载荷是消息的实际内容。PUBLISH 报文的 Payload 即为用户发送的应用消息体。SUBSCRIBE 报文的 Payload 则为订阅过滤器（Topic Filter）+ QoS 的列表。MQTT 不限制 Payload 的格式，二进制数据可以直接传输，由应用层自行解释。

### 14 种报文类型分类

MQTT 协议定义了 14 种报文类型，按功能可归为四类：

**连接类**：
- CONNECT（1）：客户端发起连接请求，包含协议版本、Clean Session、Will 信息等。
- CONNACK（2）：Broker 对 CONNECT 的确认，包含连接结果（成功或失败原因）。

**发布类**：
- PUBLISH（3）：发布消息，包含 Topic、Payload 和 QoS。
- PUBACK（4）：QoS 1 的确认报文，确认收到 PUBLISH。
- PUBREC（5）：QoS 2 第一步，表示已收到 PUBLISH（QoS 2 第一阶段）。
- PUBREL（6）：QoS 2 第二步，表示准备释放消息资源（QoS 2 第二阶段）。
- PUBCOMP（7）：QoS 2 第三步，表示消息完全处理完成（QoS 2 最终确认）。

**订阅类**：
- SUBSCRIBE（8）：订阅请求，包含一组 Topic Filter + QoS。
- SUBACK（9）：订阅确认，返回每个 Topic Filter 的授权 QoS。
- UNSUBSCRIBE（10）：取消订阅请求。
- UNSUBACK（11）：取消订阅确认。

**控制类**：
- PINGREQ（12）：心跳请求，客户端发送以维持连接。
- PINGRESP（13）：心跳响应，Broker 对 PINGREQ 的回复。
- DISCONNECT（14）：正常断开连接，MQTT 5.0 增加原因码和属性。
- AUTH（15，MQTT 5.0 新增）：认证交换报文，用于增强认证流程。

---

## QoS 服务质量

QoS（Quality of Service）是 MQTT 为消息投递定义的可靠性等级。MQTT 定义了三个等级，覆盖从"尽力而为"到"严格不重复"的全场景需求。

### QoS 0：至多一次

QoS 0 是"即发即弃"（Fire-and-Forget）模式。消息发送后，Broker 不会向发布者发送任何确认，也不会尝试重试。消息可能到达一次，也可能根本不到达。

```
Publisher → PUBLISH → Broker → PUBLISH → Subscriber
```

- **开销**：最低，无确认报文，无存储需求。
- **可靠性**：无保证。
- **适用场景**：传感器数据上报、日志采集、心跳状态——丢失一条数据不影响整体结论。

### QoS 1：至少一次

QoS 1 保证消息至少到达一次，但可能存在重复。发布者发送 PUBLISH 后等待 PUBACK，若超时未收到则重发。

```
Publisher → PUBLISH → Broker → PUBLISH → Subscriber
Publisher ← PUBACK ← Broker   Subscriber → PUBACK → Broker
```

- **开销**：一次 PUBACK 往返。
- **可靠性**：消息不丢失，但可能重复。
- **适用场景**：通知推送、指令下发——允许消费者做幂等处理。

### QoS 2：恰好一次

QoS 2 是最严格的等级，通过四步握手协议保证消息不丢失且不重复。

```
Publisher → PUBLISH → Broker
Publisher ← PUBREC ← Broker          (收到，暂存)
Publisher → PUBREL → Broker           (释放请求)
Publisher ← PUBCOMP ← Broker         (彻底完成)

Broker → PUBLISH → Subscriber
Broker ← PUBACK ← Subscriber
```

1. 发布者发送 PUBLISH 并携带 Packet Identifier。
2. Broker 收到后存储消息，回复 PUBREC（Publication Received）。
3. 发布者收到 PUBREC 后，丢弃原始消息，发送 PUBREL（Publication Release）。
4. Broker 收到 PUBREL 后，确认消息已被消费，发送 PUBCOMP（Publication Complete）。双方均可删除该报文标识符的相关状态。

- **开销**：四次报文交互，存储需求较高。
- **可靠性**：消息不丢失、不重复。
- **适用场景**：支付通知、订单状态变更、计费相关——任何重复或丢失都不可接受。

### QoS 协商机制

一个常见的理解误区是"发布者指定了 QoS 2，消费者就能以 QoS 2 接收"。实际并非如此。MQTT 的 QoS 采用**取最小值**协商机制：

```
最终服务质量 = min( 发布者 QoS, 订阅者 QoS, Topic QoS )
```

例如，发布者以 QoS 2 发送，但某个订阅者在 SUBSCRIBE 时申请的 QoS 仅为 0，那么 Broker 以 QoS 0 向该订阅者推送消息。这种机制允许发布者以高可靠投递，而不同能力的消费者各取所需。

### 各场景选型建议

| 场景 | 推荐 QoS | 理由 |
|------|----------|------|
| 环境温度、湿度采集 | QoS 0 | 数据频率高，偶尔丢失可容忍 |
| 智能家居开关指令 | QoS 1 | 要求送达，但可接受偶尔重复 |
| 金融支付通知 | QoS 2 | 零丢失零重复 |
| 日志批量上报 | QoS 0~1 | 结合本地缓存做断点续传 |
| OTA 固件升级 | QoS 2 | 关键数据，必须完整可靠 |
| 移动端推送 | QoS 1 | 网络不稳定时 PUBACK 保障到达 |

实际项目中，QoS 2 的使用应当谨慎——它的四步握手带来的延迟和吞吐开销在百万级设备规模下不可忽视。很多团队选择 QoS 1 配合应用层幂等去重，在可靠性和性能之间取得更好的平衡。

---

## MQTT 会话与会话持久化

MQTT 的会话机制决定了客户端离线期间消息的处理方式。这一设计是 MQTT 区别于普通 TCP 长连接的核心特性之一。

### Clean Session 与非 Clean Session

MQTT 3.1.1 通过 CONNECT 报文中的 **Clean Session** 标志位控制会话行为：

- **Clean Session = true**：客户端上线后不恢复旧会话，Broker 不保留该客户端过去的订阅关系和离线消息。每次连接都从"白板"状态开始。
- **Clean Session = false**：客户端上线后若存在之前的持久化会话，Broker 恢复该会话的全部订阅关系和未消费的离线消息。

MQTT 5.0 废弃了 Clean Session 标志，改用 **Session Expiry Interval** 属性。将其设为 0 等价于 Clean Session = true；设为一个正整数值（秒）表示会话在该时长内持续有效；设为 `0xFFFFFFFF` 表示永不过期。

### 离线消息存储与重投

当启用了持久会话且客户端断开时，Broker 会为它暂存 QoS 1 和 QoS 2 的消息。待客户端重新上线后，Broker 依次投递这些消息。需要特别注意的是，QoS 0 的消息在离线期间不会被存储——因为 QoS 0 本身就不保证投递可靠性。

离线消息的存储策略因 Broker 实现而异。EMQX 支持将离线消息持久化到磁盘数据库（如 Mnesia 或外部 RDBMS），而轻量级 Broker 如 Mosquitto 通常只做内存缓存。在大量设备频繁上下线的场景下，离线消息堆积可能成为瓶颈，需要为每客户端限制最大离线消息数。

### 会话到期（MQTT 5.0）

MQTT 5.0 的 Session Expiry Interval 提供了更精确的会话生命周期控制。当客户端由于网络原因短暂断开时，一个较长的会话过期时间可以保证设备重连后无需重新订阅 Topic，避免了额外的通信开销。而针对偶尔上线的低功耗设备（如电池供电的传感器），可以设定较短的 Session Expiry 或直接设置为 0，Broker 不必长期为其保留状态。

---

## 遗嘱消息与保留消息

这两个机制是 MQTT 协议中容易被忽视但极具实用价值的功能。

### 遗嘱消息

遗嘱消息（Last Will and Testament, LWT）是客户端在 CONNECT 时注册的一条"遗言"，当 Broker 检测到客户端异常断开（如网络断开、心跳超时、Broker 主动关闭连接）时，由 Broker 代为发布。

遗嘱消息的配置包含以下字段：

- **Will Topic**：遗嘱的发布目标 Topic。
- **Will Payload**：遗嘱消息的内容。
- **Will QoS**：发布该消息时使用的 QoS 等级。
- **Will Retain**：该遗嘱消息是否被标记为保留消息。

MQTT 5.0 还为 Will 消息增加了 Will Delay Interval 和 Will User Properties，允许延迟触发或携带业务上下文。

典型的使用场景是"设备在线状态检测"：设备上线时将遗嘱 Topic 设为 `device/{id}/status`，Payload 设为 `offline`；同时设备正常运行时定期向该 Topic 发送 `online` 的保留消息。当设备异常断网时，Broker 自动发布遗嘱消息 `offline`，系统的其他组件据此判断设备下线。

需要强调的是，正常发送 DISCONNECT 报文不会触发遗嘱消息——即"体面告别"不发布遗言。

### 保留消息

保留消息（Retain）是 PUBLISH 报文的 RETAIN 标志位控制的特性。当一条 PUBLISH 报文设置了 RETAIN = 1，Broker 会将该消息作为该 Topic 的最新保留消息存储在内存或磁盘中。

保留消息的核心作用是解决"信息发布时订阅者尚未就绪"的问题。新订阅者上线后，Broker 会立即向其推送该 Topic 的最后一条保留消息，使其快速获取当前状态，而不必等待下一次发布。

```
PUBLISH topic=status/power, payload=on, RETAIN=1
  → Broker 保留该消息

SUBSCRIBE topic=status/+
  → Broker 立即推送保留消息 "on"
  → 之后正常接收实时消息
```

保留消息的典型应用包括：设备状态快照（在线/离线、当前温度）、系统配置信息、版本号广播等。要清除某 Topic 的保留消息，只需向该 Topic 发布一条 Payload 为空的保留消息即可。

---

## MQTT 5.0 关键新特性

MQTT 5.0 不是对 3.1.1 的盲目增加功能，而是对实践中痛点的针对性回应。

### 原因码与增强确认

3.1.1 的 ACK 只有成功/失败两种语义，故障排查时信息严重不足。5.0 引入了原因码（Reason Code），每个确认报文（CONNACK、PUBACK、SUBACK 等）都携带一个字节的原因码来描述具体结果。

```
CONNACK Reason Code 示例：
0x00    成功
0x80    未指定的错误
0x81    MQTT 协议版本不被接受
0x82    Client ID 被拒
0x83    服务器不可用
0x85    未授权
0x8A    双名违规（同一个 Client ID 二次连接）
```

### 用户属性

User Properties 是 Payload 之外的键值对扩展机制，可以出现在 CONNECT、PUBLISH、DISCONNECT 等多种报文的可变头中。这使得业务元数据（请求来源、优先级、Trace ID 等）可以附加在协议层而不污染消息体。

### 消息过期间隔

PUBLISH 报文支持 `Message Expiry Interval` 属性。当该属性指定的秒数过期后，Broker 不应继续向订阅者投递此消息。这有效防止了在设备离线时间较长后重新上线时，突然涌出一大堆过期的消息。

### 主题别名

IoT 场景中 Topic 名称可能非常长（如 `home/floor2/kitchen/sensor/temperature`），每次 PUBLISH 都携带完整 Topic 会大量浪费带宽。Topic Alias 机制允许客户端和 Broker 协商一个整型编号来代表完整 Topic，此后 PUBLISH 只需携带别名编号而非完整字符串。

### 订阅选项

5.0 的 SUBSCRIBE 报文为每个 Topic Filter 增加了三个选项：

- **No Local**：发布者不会收到自己发布的消息，适用于网关设备避免消息回环。
- **Retain As Published**：Broker 转发保留消息时保持其 RETAIN 标志的状态。
- **Retain Handling**：控制保留消息的发送时机——发送保留消息、仅在新订阅时发送、或完全不发保留消息。

### 请求/响应模式

5.0 增加了 Response Topic 和 Correlation Data 属性，使 MQTT 原生支持请求/响应模式。发布者可以在 PUBLISH 中设置一条消息期望的响应 Topic 和相关标识，订阅者在处理完请求后将结果发布到该响应 Topic。这一模式使 MQTT 可以覆盖 RPC 类通信场景。

### 流控

3.1.1 没有流控机制，客户端可以不受限制地向 Broker 发送消息，当发送速度远超消费速度时，Broker 会承受巨大的内存压力。5.0 的 `Receive Maximum` 属性允许对端指定当前连接上允许的最大未完成 PUBLISH 报文数量（未确认的 QoS 1/2 消息），超过该数量时发送方必须暂停发送，形成自然背压。

---

## 完整连接交互流程

一个标准的 MQTT 连接生命周期包含以下阶段：

```
客户端                           Broker
  │                                │
  │─── 1. TCP 连接建立 ─────────→  │
  │                                │
  │─── 2. CONNECT ───────────────→ │  协议版本、Client ID、Clean Session、
  │                                │  Will Topic/消息、用户名密码等
  │←── 3. CONNACK ─────────────── │  原因码、Session Present 标志
  │                                │
  │═══ 4. 业务数据交互 ═══════════│
  │─── PUBLISH (QoS 0) ─────────→ │  不需确认
  │─── PUBLISH (QoS 1) ─────────→ │  需 PUBACK 确认
  │←── PUBACK ─────────────────── │
  │─── PUBLISH (QoS 2) ─────────→ │  四步握手
  │←── PUBREC ─────────────────── │
  │─── PUBREL ──────────────────→ │
  │←── PUBCOMP ────────────────── │
  │─── SUBSCRIBE ───────────────→ │  订阅 Topic
  │←── SUBACK ─────────────────── │  确认授权 QoS
  │←── PUBLISH (推送消息) ──────→│  Broker 向订阅者推送
  │─── UNSUBSCRIBE ────────────── │  取消订阅
  │←── UNSUBACK ───────────────── │
  │                                │
  │═══ 5. 保活心跳 ═══════════════│
  │─── PINGREQ ─────────────────→ │  每隔 KeepAlive 间隔的 1.5 倍内至少发一次
  │←── PINGRESP ───────────────── │
  │                                │
  │═══ 6. 断开连接 ═══════════════│
  │─── DISCONNECT ──────────────→ │  体面关闭，不触发 Will
  │─── TCP 连接关闭 ─────────────→ │
```

### 阶段说明

**1~3. 连接建立**：客户端首先建立 TCP 连接（如果是 TLS 则在其上完成握手），然后发送 CONNECT 报文。CONNECT 报文中包含协议版本（3.1.1 或 5.0）、Client ID、Clean Session/Session Expiry Interval、用户名密码、Will 信息等。Broker 处理后回复 CONNACK 报文，携带连接结果和 Session Present 标志。

**4. 业务交互**：连接建立后，客户端可以进行 PUBLISH 和 SUBSCRIBE/UNSUBSCRIBE 交互。多个报文可以在同一个 TCP 连接上交错传输，Broker 通过 Packet Identifier 区分不同的请求-确认配对。

**5. 保活心跳**：TCP 自身的黏连检测不足以应对移动网络下的 NAT 超时问题。MQTT 在应用层设计了心跳机制：客户端在 CONNECT 中声明 `KeepAlive` 间隔（秒），在 1.5 倍的 KeepAlive 时间内至少发送一次 PINGREQ 报文。Broker 如果在 1.5 倍 KeepAlive 时间内未收到任何报文，则判定客户端掉线，触发 Will 消息发布。心跳的间隔通常在 30~300 秒之间，依网络环境而定。

**6. 断开连接**：正常下线时，客户端发送 DISCONNECT 报文后关闭 TCP 连接。MQTT 5.0 的 DISCONNECT 可以携带原因码和 User Properties，用于传递断开原因。正常断开不触发遗嘱消息。

---

## 生产实践建议

基于协议的理解，以下是面向 Java 后端开发者的实践建议：

1. **Client ID 唯一性**：每个设备/服务实例必须有唯一的 Client ID，推荐使用设备序列号或 UUID。同 ID 的二重连接会导致旧连接被断开（Session Takeover）。
2. **QoS 与幂等结合**：首选 QoS 1 配合消息 ID 去重，比 QoS 2 更易横向扩展；确实需要 QoS 2 时，评估 Broker 的处理瓶颈。
3. **持久会话的权衡**：持久会话能减少频繁上下线设备的订阅开销，但 Broker 的离线消息积压会影响集群稳定性。建议为每个客户端设置离线消息的上限（如 EMQX 的 `max_mqueue_len`）。
4. **Topic 命名规范**：采用倒置域名风格的层级结构（如 `com/company/project/sensor/{deviceId}/temp`），避免使用无意义的通配符。
5. **TLS 传输**：MQTT 默认不加密，生产环境必须使用 TLS 加密通信，配合 Client Certificate 做设备认证。
6. **KeepAlive 调优**：移动网络建议 60~120 秒心跳，办公室局域网可延长至 300 秒。过短的心跳会增加 Broker 消息压力，过长则无法及时发现断线。
7. **Java 客户端选型**：Eclipse Paho 是成熟的 MQTT 3.1.1 客户端；EMQX 的 NanoSDK 或 HiveMQ MQTT Client 对 MQTT 5.0 支持更完善，推荐新项目选用。
