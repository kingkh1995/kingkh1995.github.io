---
title: tcpdump 与 Wireshark 抓包实战
category: dev-tools
date: 2026-05-09
summary: tcpdump 与 Wireshark 抓包实战指南，涵盖过滤表达式、TCP 标志位及问题定位。
---

## tcpdump

### 基本用法

```bash
# 抓取指定网卡的所有包
tcpdump -i eth0

# 列出可用网卡
tcpdump -D

# 限制抓包数量
tcpdump -i eth0 -c 100

# 不解析主机名/端口名（速度快，建议始终加上）
tcpdump -i eth0 -n

# 输出更详细的信息（链路层地址、TTL 等）
tcpdump -i eth0 -v
tcpdump -i eth0 -vv
tcpdump -i eth0 -vvv
```

### 过滤表达式

```bash
# 按主机
tcpdump host 192.168.1.100
tcpdump src host 192.168.1.100
tcpdump dst host 192.168.1.100

# 按端口
tcpdump port 80
tcpdump src port 8080
tcpdump dst port 443
tcpdump portrange 8000-9000

# 按协议 + 端口
tcpdump tcp port 3306
tcpdump udp port 53

# 组合条件 (and / or / not)
tcpdump src host 10.0.0.5 and tcp port 80
tcpdump host 10.0.0.5 and not port 22
tcpdump 'host 10.0.0.5 and (port 80 or port 443)'
```

### TCP 标志位过滤

```bash
# SYN 包
tcpdump 'tcp[tcpflags] & (tcp-syn) != 0'

# SYN + ACK 包
tcpdump 'tcp[tcpflags] & (tcp-syn|tcp-ack) == (tcp-syn|tcp-ack)'

# RST 包
tcpdump 'tcp[tcpflags] & (tcp-rst) != 0'

# FIN 包
tcpdump 'tcp[tcpflags] & (tcp-fin) != 0'

# PSH 包
tcpdump 'tcp[tcpflags] & (tcp-psh) != 0'
```

### 写入文件 / 读取文件

```bash
# 写入 pcap 文件（用 Wireshark 分析）
tcpdump -i eth0 -w capture.pcap

# 写入的同时也在终端查看
tcpdump -i eth0 -w capture.pcap -v

# 限制每个文件大小（MB），轮转写入
tcpdump -i eth0 -w capture.pcap -C 100

# 限制文件数量，达到上限后覆盖最早的
tcpdump -i eth0 -w capture.pcap -C 100 -W 5

# 读取 pcap 文件
tcpdump -r capture.pcap

# 读取时加过滤
tcpdump -r capture.pcap host 10.0.0.5
```

### 常用组合示例

```bash
# 抓取某 IP 的所有 TCP 包，写文件，不做 DNS 解析
tcpdump -i eth0 -nn tcp and host 10.0.0.5 -w /tmp/debug.pcap

# 只抓 TCP 三次握手和四次挥手
tcpdump -i eth0 -nn 'tcp[tcpflags] & (tcp-syn|tcp-fin) != 0'

# 抓取 HTTP 请求（端口 80，带 PSH 的数据包）
tcpdump -i eth0 -nn 'tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)'

# 抓取特定大小的包
tcpdump greater 1000
tcpdump less 500
```

### tcpdump 输出字段速读

```
12:30:45.123456 IP 10.0.0.5.54321 > 10.0.0.6.80: Flags [S], seq 123456789, win 65535, options [mss 1460], length 0
```
- `10.0.0.5.54321` — 源 IP:端口
- `>` — 方向
- `10.0.0.6.80` — 目标 IP:端口
- `Flags [S]` — TCP 标志：S=SYN, F=FIN, R=RST, P=PSH, .=ACK
- `seq` — 序列号
- `win` — 窗口大小
- `length` — TCP 载荷长度

---

## Wireshark

### 常用显示过滤器

```
# IP/端口
ip.src == 10.0.0.5
ip.dst == 10.0.0.6
tcp.port == 80
tcp.srcport == 54321
tcp.dstport == 443

# TCP 标志
tcp.flags.syn == 1
tcp.flags.syn == 1 && tcp.flags.ack == 0     # 仅 SYN（三次握手第一步）
tcp.flags.syn == 1 && tcp.flags.ack == 1     # SYN+ACK
tcp.flags.reset == 1
tcp.flags.fin == 1

# TCP 分析
tcp.analysis.retransmission
tcp.analysis.fast_retransmission
tcp.analysis.duplicate_ack
tcp.analysis.zero_window
tcp.analysis.window_full
tcp.analysis.lost_segment
tcp.analysis.keep_alive

# HTTP
http.request
http.response
http.host contains "example.com"

# 组合
ip.addr == 10.0.0.5 and tcp.port == 80
!(arp or dns or icmp)
```

### 常用列与统计

- **Statistics → Conversations** — 查看所有 TCP 会话的包数、字节数、持续时间
- **Statistics → Flow Graph** — TCP 流的时序图（选 TCP 流后查看更清晰）
- **Statistics → IO Graph** — 吞吐量曲线，可叠加过滤条件对比
- **Analyze → Follow → TCP Stream** — 追踪一个 TCP 连接的完整数据流，直接看到请求/响应原文
- **右键包 → Decode As** — 强制用指定协议解析某端口

### 着色规则与标记

- 黑色背景红字 — RST 包（连接重置）
- 黑色背景黄字 — TCP 重传/乱序/丢包
- `Ctrl+M` — 标记当前包，方便定位

### 常用操作

| 操作 | 快捷键 / 方式 |
|------|------------|
| 开始抓包 | `Ctrl+E` |
| 停止抓包 | `Ctrl+E` |
| 应用显示过滤器 | 输入后按 Enter |
| 追踪 TCP 流 | 右键包 → Follow → TCP Stream |
| 导出特定包 | File → Export Specified Packets |
| 时间格式调整 | View → Time Display Format |

---

## 实战案例：定位 TCP 连接超时问题

### 场景

后端服务 A（10.0.0.5:8080）调用服务 B（10.0.0.6:3306）时偶发超时，日志显示 `dial tcp 10.0.0.6:3306: i/o timeout`。

### Step 1 — 抓包

在服务 A 所在机器抓取到服务 B 的 TCP 包：

```bash
tcpdump -i eth0 -nn host 10.0.0.6 and port 3306 -w /tmp/mysql_timeout.pcap -C 50 -W 3
```

- `-nn` 不做 DNS 和端口名解析
- `-C 50 -W 3` 三个 50MB 文件轮转，避免写满磁盘
- 复现问题后 `Ctrl+C` 停止

### Step 2 — 用 Wireshark 打开 pcap

先用 tcpdump 快速确认：

```bash
tcpdump -r /tmp/mysql_timeout.pcap -nn | head -50
```

将 pcap 拉到本地用 Wireshark 打开。

### Step 3 — 定位问题连接

Wireshark 显示过滤器：

```
tcp.flags.syn == 1 && tcp.flags.ack == 0
```

只显示 SYN 包，找到超时时间点附近发起的连接。

**发现：** 有大量 SYN 包没有对应的 SYN+ACK 回复，发出去后 1s、3s、7s 持续重传 SYN，最终放弃。

### Step 4 — 对比正常连接

选一个超时的连接，右键 **Follow → TCP Stream**，对比正常连接：

| 正常连接 | 异常连接 |
|----------|----------|
| SYN → | SYN → |
| ← SYN+ACK（<1ms） | （无回复） |
| ACK → | SYN 重传（1s 后）→ |
| 数据传输... | SYN 重传（3s 后）→ |
| | ... 最终超时 |

### Step 5 — 根因分析

1. **Statistics → IO Graph** 看 SYN 量：超时发生时 SYN 包数量飙升
2. 过滤 `tcp.analysis.retransmission`，发现重传集中爆发

**结论：** 服务 B 的 SYN 队列满（`net.core.somaxconn` 不足或应用层 accept 太慢），导致新连接 SYN 被丢弃。排查服务 B 的 `ss -s` 或 `netstat -s` 确认 `SYNs to LISTEN sockets dropped` 计数增长。

### 在抓包机器上直接验证

```bash
# 查看重传的 SYN 数量
tcpdump -r /tmp/mysql_timeout.pcap -nn 'tcp[tcpflags] & (tcp-syn) != 0' | wc -l

# 查看被 RST 的连接数
tcpdump -r /tmp/mysql_timeout.pcap -nn 'tcp[tcpflags] & (tcp-rst) != 0' | wc -l
```

---

## 常用 TCP 抓包速查

```bash
# 抓取完整 TCP 连接过程（含握手、挥手、数据），写文件用 Wireshark 分析
tcpdump -i any -nn -s0 host <目标IP> and port <目标端口> -w out.pcap

# 只看握手/挥手
tcpdump -i any -nn 'tcp[tcpflags] & (tcp-syn|tcp-fin|tcp-rst) != 0' and host <目标IP>

# 抓取 HTTP 请求响应（不加密场景）
tcpdump -i any -nn -A 'tcp port 80 and host <目标IP>'

# 抓包统计（按源 IP 聚合）
tcpdump -r capture.pcap -nn | awk '{print $3}' | sort | uniq -c | sort -rn | head -20
```
