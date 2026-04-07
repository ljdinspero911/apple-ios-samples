# Apple URLSession Multipath TCP (MPTCP) 深度分析

## 目录

1. [概述](#1-概述)
2. [核心原理](#2-核心原理)
3. [四种服务类型详解](#3-四种服务类型详解)
4. [iOS 客户端实现](#4-ios-客户端实现)
5. [后台服务端支持](#5-后台服务端支持)
6. [验证与测试](#6-验证与测试)
7. [限制与注意事项](#7-限制与注意事项)
8. [架构图](#8-架构图)

---

## 1. 概述

### 什么是 Multipath TCP

Multipath TCP（MPTCP）是标准 TCP 的扩展协议，定义在 [RFC 8684](https://www.rfc-editor.org/rfc/rfc8684.html) 中。它允许单个 TCP 连接同时使用多个网络接口（如 Wi-Fi 和蜂窝网络）进行数据传输，从而实现：

- **无缝切换（Seamless Handover）**：用户从 Wi-Fi 覆盖区域移动到蜂窝网络时，连接不中断
- **最佳路径选择（Best Path Selection）**：根据延迟、丢包率等条件选择最佳网络路径
- **带宽聚合（Bandwidth Aggregation）**：同时使用多条路径增加总吞吐量

### Apple 的实现

Apple 从 **iOS 11** 开始，在 `URLSessionConfiguration` 中引入了 `multipathServiceType` 属性，使开发者可以通过简单的配置启用 MPTCP。这是一个**仅 iOS 可用**的特性（macOS 上的 URLSession 不支持）。

### 实际场景

```
用户在咖啡店使用 Wi-Fi 视频通话
    ↓
走出咖啡店，Wi-Fi 信号逐渐减弱
    ↓
MPTCP 检测到 Wi-Fi 质量下降
    ↓
自动在蜂窝网络上建立子流（subflow）
    ↓
流量无缝切换到蜂窝网络
    ↓
视频通话全程不中断
```

---

## 2. 核心原理

### 2.1 协议层面

MPTCP 工作在传输层，通过 TCP 选项字段（TCP Option Field）进行协商：

```
┌─────────────────────────────────────────────────┐
│                 应用层 (HTTP/HTTPS)               │
├─────────────────────────────────────────────────┤
│              MPTCP 层 (连接管理/调度)             │
├──────────────┬──────────────┬───────────────────┤
│   子流 1      │   子流 2     │   子流 N          │
│  (TCP over   │  (TCP over   │  (TCP over        │
│   Wi-Fi)     │   Cellular)  │   其他接口)        │
├──────────────┴──────────────┴───────────────────┤
│                    IP 层                          │
├──────────────┬──────────────────────────────────┤
│   Wi-Fi 接口  │       蜂窝网络接口                │
└──────────────┴──────────────────────────────────┘
```

### 2.2 连接建立流程

```
客户端 (iOS)                          服务器 (MPTCP-enabled)
    │                                       │
    │── SYN + MP_CAPABLE ─────────────────→ │  1. 初始握手（Wi-Fi）
    │←── SYN+ACK + MP_CAPABLE ────────────  │
    │── ACK + MP_CAPABLE ─────────────────→ │
    │                                       │
    │    ═══ 主子流已建立 (Wi-Fi) ═══        │
    │                                       │
    │── SYN + MP_JOIN ────────────────────→ │  2. 建立第二子流（蜂窝）
    │←── SYN+ACK + MP_JOIN ───────────────  │
    │── ACK + MP_JOIN ────────────────────→ │
    │                                       │
    │    ═══ 备用子流已建立 (Cellular) ═══   │
    │                                       │
    │←─────── 数据双向传输 ─────────────→   │  3. 按策略调度数据
    │                                       │
```

**关键 TCP 选项：**

| 选项 | Kind 值 | 用途 |
|------|---------|------|
| `MP_CAPABLE` | 30 (subtype 0) | 初始连接协商，声明 MPTCP 支持 |
| `MP_JOIN` | 30 (subtype 1) | 将新子流加入已有 MPTCP 连接 |
| `DSS` | 30 (subtype 2) | 数据序列信号，映射子流序号到连接级序号 |
| `ADD_ADDR` | 30 (subtype 3) | 通告可用的额外地址 |
| `REMOVE_ADDR` | 30 (subtype 4) | 移除之前通告的地址 |
| `MP_PRIO` | 30 (subtype 5) | 修改子流优先级 |
| `MP_FAIL` | 30 (subtype 6) | 回退到普通 TCP |
| `MP_FASTCLOSE` | 30 (subtype 7) | 快速关闭连接 |

### 2.3 降级兼容

如果服务器或中间设备（middlebox）不支持 MPTCP：

```
客户端                               服务器 (不支持 MPTCP)
    │                                       │
    │── SYN + MP_CAPABLE ─────────────────→ │
    │←── SYN+ACK (无 MPTCP 选项) ─────────  │  服务器忽略 MP_CAPABLE
    │── ACK ──────────────────────────────→ │
    │                                       │
    │    ═══ 降级为普通 TCP 连接 ═══         │
```

这种降级是**完全透明**的，对应用层没有任何影响，性能损耗极小。

---

## 3. 四种服务类型详解

`URLSessionConfiguration.MultipathServiceType` 枚举定义了四种模式：

### 3.1 `.none`（默认）

```swift
configuration.multipathServiceType = .none
```

- **行为**：不启用 MPTCP，使用标准 TCP
- **适用场景**：不需要多路径的常规网络请求

### 3.2 `.handover`（推荐）

```swift
configuration.multipathServiceType = .handover
```

- **行为**：
  - 优先使用 Wi-Fi
  - Wi-Fi 信号劣化时，**无缝切换**到蜂窝网络
  - 切换过程中连接不中断、不重建
  - Wi-Fi 恢复后，自动切回 Wi-Fi
- **数据消耗**：仅在 Wi-Fi 不可用时才使用蜂窝数据，**不会同时使用两个网络**
- **适用场景**：
  - 长连接（WebSocket、HTTP/2 长轮询）
  - 即时通讯应用
  - 实时音视频通话
  - 地图导航
  - 任何需要连接稳定性的场景
- **Apple 推荐**：大多数应用应选择此模式

### 3.3 `.interactive`

```swift
configuration.multipathServiceType = .interactive
```

- **行为**：
  - 始终使用**延迟最低**的网络接口
  - 持续探测各路径的延迟
  - 动态切换到响应更快的路径
- **数据消耗**：可能会同时在两个接口上产生探测流量
- **适用场景**：
  - 对延迟极度敏感的应用
  - 实时游戏
  - 股票交易应用
  - 交互式协作工具

### 3.4 `.aggregate`

```swift
configuration.multipathServiceType = .aggregate
```

- **行为**：
  - **同时**使用 Wi-Fi 和蜂窝网络
  - 将数据分散到多条路径上并行传输
  - 聚合多个接口的带宽
  - 同时最大化吞吐量和最小化延迟
- **数据消耗**：**最大**，持续使用蜂窝数据
- **适用场景**：
  - 大文件下载
  - 需要最大带宽的场景
- **注意**：此模式需要 MPTCP entitlement，且受 Wi-Fi Assist 限制

### 四种模式对比

| 特性 | none | handover | interactive | aggregate |
|------|------|----------|-------------|-----------|
| MPTCP 启用 | ❌ | ✅ | ✅ | ✅ |
| 蜂窝数据消耗 | 无 | 仅切换时 | 探测流量 | 持续使用 |
| 连接稳定性 | 一般 | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ |
| 延迟优化 | 无 | 一般 | ⭐⭐⭐ | ⭐⭐ |
| 吞吐量 | 单路径 | 单路径 | 单路径(最快) | 多路径聚合 |
| 推荐程度 | 默认 | **最推荐** | 特定场景 | 特定场景 |

---

## 4. iOS 客户端实现

### 4.1 启用 Entitlement

在 Xcode 项目中：

1. 选择 App Target → **Signing & Capabilities**
2. 点击 **+ Capability**
3. 添加 **Multipath** entitlement

或直接编辑 `.entitlements` 文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>com.apple.developer.networking.multipath</key>
    <true/>
</dict>
</plist>
```

### 4.2 基本使用

```swift
import Foundation

// 1. 创建配置并启用 MPTCP
let configuration = URLSessionConfiguration.default
configuration.multipathServiceType = .handover

// 2. 创建 URLSession
let session = URLSession(configuration: configuration)

// 3. 正常发起网络请求 — 无需其他改动
let url = URL(string: "https://api.example.com/data")!
let task = session.dataTask(with: url) { data, response, error in
    if let error = error {
        print("请求失败: \(error)")
        return
    }
    if let data = data {
        print("收到 \(data.count) 字节")
    }
}
task.resume()
```

### 4.3 带 Delegate 的高级用法

```swift
class NetworkManager: NSObject, URLSessionDelegate, URLSessionTaskDelegate {

    private lazy var session: URLSession = {
        let config = URLSessionConfiguration.default
        config.multipathServiceType = .handover
        config.waitsForConnectivity = true
        config.timeoutIntervalForResource = 300
        return URLSession(configuration: config, delegate: self, delegateQueue: nil)
    }()

    func fetchData(from url: URL) {
        let task = session.dataTask(with: url) { data, response, error in
            // 处理响应
        }
        task.resume()
    }

    // 监控连接状态变化
    func urlSession(_ session: URLSession,
                    taskIsWaitingForConnectivity task: URLSessionTask) {
        print("等待网络连接...")
    }

    func urlSession(_ session: URLSession,
                    task: URLSessionTask,
                    didFinishCollecting metrics: URLSessionTaskMetrics) {
        for metric in metrics.transactionMetrics {
            print("协议: \(metric.networkProtocolName ?? "unknown")")
            print("是否多路径: \(metric.isMultipath)")
            print("接口: \(metric.localAddress ?? "N/A")")
            if metric.isMultipath {
                print("✅ MPTCP 连接成功")
            }
        }
    }
}
```

### 4.4 结合 Reachability 的最佳实践

```swift
import Network

class AdaptiveNetworkManager {

    private let monitor = NWPathMonitor()
    private var currentSession: URLSession?

    init() {
        monitor.pathUpdateHandler = { [weak self] path in
            self?.updateConfiguration(for: path)
        }
        monitor.start(queue: DispatchQueue.global(qos: .utility))
    }

    private func updateConfiguration(for path: NWPath) {
        let config = URLSessionConfiguration.default

        if path.availableInterfaces.count > 1 {
            // 多个接口可用时启用 MPTCP
            config.multipathServiceType = .handover
            print("多网卡可用，启用 MPTCP handover")
        } else {
            config.multipathServiceType = .none
            print("仅单网卡，使用标准 TCP")
        }

        currentSession?.invalidateAndCancel()
        currentSession = URLSession(configuration: config)
    }
}
```

---

## 5. 后台服务端支持

**关键要点：MPTCP 要求服务器端也支持 MPTCP 协议。** 如果服务器不支持，连接会降级为普通 TCP。

### 5.1 Linux 内核配置

MPTCP 从 Linux 内核 **5.6** 开始内置支持（MPTCPv1）。

#### 检查内核是否支持

```bash
# 检查内核版本
uname -r

# 检查 MPTCP 是否编译进内核
sysctl net.mptcp.enabled 2>/dev/null
# 输出 net.mptcp.enabled = 1 表示已启用

# 如果命令不存在，检查内核配置
grep MPTCP /boot/config-$(uname -r)
# CONFIG_MPTCP=y
# CONFIG_MPTCP_IPV6=y
```

#### 启用 MPTCP

```bash
# 启用 MPTCP（临时）
sysctl -w net.mptcp.enabled=1

# 启用 MPTCP（永久）
echo "net.mptcp.enabled=1" > /etc/sysctl.d/90-enable-MPTCP.conf
sysctl -p /etc/sysctl.d/90-enable-MPTCP.conf
```

#### 配置路径管理器（Path Manager）

```bash
# 查看当前路径管理器类型
sysctl net.mptcp.pm_type
# 0 = 内核内置（in-kernel），相同规则应用于所有连接
# 1 = 用户空间（userspace），由 mptcpd 守护进程控制

# 添加端点（endpoint）供子流使用
# 以下命令将服务器的第二个 IP 地址声明为可用的 MPTCP 端点
ip mptcp endpoint add 192.168.1.100 dev eth0 signal
ip mptcp endpoint add 10.0.0.100 dev eth1 signal subflow

# 查看已配置的端点
ip mptcp endpoint show

# 设置最大子流数
ip mptcp limits set subflow 4 add_addr_accepted 4
```

#### 配置包调度器（Packet Scheduler）

```bash
# 查看当前调度器
sysctl net.mptcp.scheduler

# 可选调度器:
# "default" - 默认调度器，优先使用最低延迟的子流
# "redundant" - 冗余调度器，在所有子流上发送相同数据
```

### 5.2 Nginx 配置

Nginx 从 **1.29.7** 版本开始原生支持 MPTCP。

#### Nginx 1.29.7+

```nginx
http {
    server {
        # 添加 multipath 标志即可启用 MPTCP
        listen 443 ssl multipath;
        listen [::]:443 ssl multipath;

        server_name api.example.com;

        ssl_certificate     /etc/ssl/certs/server.crt;
        ssl_certificate_key /etc/ssl/private/server.key;

        location / {
            proxy_pass http://backend;
        }
    }
}
```

#### 旧版 Nginx（使用 mptcpize）

对于不原生支持 MPTCP 的 Nginx 版本，可以使用 `mptcpize` 工具：

```bash
# 安装 mptcpize（mptcpd 包的一部分）
apt install mptcpd  # Debian/Ubuntu
# 或
dnf install mptcpd  # RHEL/Fedora

# 使用 mptcpize 包装 Nginx 启动
mptcpize run nginx -g "daemon off;"
```

`mptcpize` 通过 `LD_PRELOAD` 机制将 `IPPROTO_TCP` 套接字自动替换为 `IPPROTO_MPTCP`。

### 5.3 其他 Web 服务器 / 负载均衡器

| 服务器/负载均衡器 | MPTCP 支持 | 备注 |
|-------------------|-----------|------|
| **Nginx 1.29.7+** | ✅ 原生 | `listen` 指令加 `multipath` |
| **HAProxy** | ✅ 原生 | 较新版本支持 |
| **F5 BIG-IP** | ✅ 原生 | 商业负载均衡器，完整支持 |
| **Citrix ADC** | ✅ 原生 | 商业负载均衡器 |
| **Apache** | ⚠️ mptcpize | 通过 mptcpize 启动 |
| **自定义服务** | ⚠️ 需代码修改 | 使用 `IPPROTO_MPTCP` 创建套接字 |

### 5.4 自定义服务端代码支持

#### Go 服务端

```go
package main

import (
    "fmt"
    "net"
    "syscall"
)

const IPPROTO_MPTCP = 262

func main() {
    // 使用 MPTCP 协议创建监听器
    lc := net.ListenConfig{
        Control: func(network, address string, c syscall.RawConn) error {
            return c.Control(func(fd uintptr) {
                // 设置 MPTCP 协议
                syscall.SetsockoptInt(int(fd), syscall.SOL_TCP,
                    0x2a, 1) // TCP_IS_MPTCP
            })
        },
    }

    listener, err := lc.Listen(context.Background(), "tcp", ":8080")
    if err != nil {
        panic(err)
    }
    defer listener.Close()

    fmt.Println("MPTCP server listening on :8080")
    // ... 处理连接
}
```

Go 1.21+ 原生支持，只需设置环境变量：

```bash
GODEBUG=multipathtcp=1 go run server.go
```

#### Python 服务端

```python
import socket

IPPROTO_MPTCP = 262

# 创建 MPTCP 套接字
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM, IPPROTO_MPTCP)
sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
sock.bind(('0.0.0.0', 8080))
sock.listen(5)

print("MPTCP server listening on :8080")

while True:
    conn, addr = sock.accept()
    print(f"Connection from {addr}")
    # ... 处理连接
```

### 5.5 完整的服务端部署清单

```bash
#!/bin/bash
# deploy-mptcp.sh - MPTCP 服务端部署脚本

set -e

echo "=== MPTCP 服务端部署 ==="

# 1. 检查内核版本
KERNEL_VERSION=$(uname -r | cut -d. -f1-2)
echo "内核版本: $KERNEL_VERSION"
if (( $(echo "$KERNEL_VERSION < 5.6" | bc -l) )); then
    echo "❌ 内核版本过低，需要 >= 5.6"
    exit 1
fi
echo "✅ 内核版本满足要求"

# 2. 检查 MPTCP 编译支持
if ! grep -q "CONFIG_MPTCP=y" /boot/config-$(uname -r) 2>/dev/null; then
    echo "⚠️  MPTCP 可能未编译进内核，尝试启用..."
fi

# 3. 启用 MPTCP
sysctl -w net.mptcp.enabled=1
echo "net.mptcp.enabled=1" > /etc/sysctl.d/90-mptcp.conf
echo "✅ MPTCP 已启用"

# 4. 配置端点
for iface in $(ip -o link show up | awk -F': ' '{print $2}' | grep -v lo); do
    ip_addr=$(ip -4 addr show $iface | grep -oP '(?<=inet\s)\d+(\.\d+){3}')
    if [ -n "$ip_addr" ]; then
        ip mptcp endpoint add $ip_addr dev $iface signal subflow 2>/dev/null || true
        echo "  添加端点: $ip_addr ($iface)"
    fi
done

# 5. 设置子流限制
ip mptcp limits set subflow 8 add_addr_accepted 8
echo "✅ 子流限制已设置"

# 6. 验证配置
echo ""
echo "=== 配置验证 ==="
echo "MPTCP 状态: $(sysctl -n net.mptcp.enabled)"
echo "端点列表:"
ip mptcp endpoint show
echo "子流限制:"
ip mptcp limits show
```

---

## 6. 验证与测试

### 6.1 客户端验证

#### 方法一：URLSessionTaskMetrics

最直接的验证方式，通过 `URLSessionTaskMetrics` 检查连接是否使用了 MPTCP：

```swift
func urlSession(_ session: URLSession,
                task: URLSessionTask,
                didFinishCollecting metrics: URLSessionTaskMetrics) {

    for transaction in metrics.transactionMetrics {
        // 检查是否使用了 MPTCP
        let isMultipath = transaction.isMultipath
        let protocol = transaction.networkProtocolName ?? "unknown"
        let localAddr = transaction.localAddress ?? "N/A"
        let remoteAddr = transaction.remoteAddress ?? "N/A"

        print("""
        ───────────────────────────
        协议: \(protocol)
        多路径: \(isMultipath ? "✅ 是" : "❌ 否")
        本地地址: \(localAddr)
        远程地址: \(remoteAddr)
        ───────────────────────────
        """)
    }
}
```

#### 方法二：在线检测服务

使用 `http://amiusingmptcp.de` 进行检测：

```swift
let config = URLSessionConfiguration.default
config.multipathServiceType = .handover

let session = URLSession(configuration: config)
let url = URL(string: "http://amiusingmptcp.de")!

session.dataTask(with: url) { data, response, error in
    if let data = data, let body = String(data: data, encoding: .utf8) {
        if body.contains("You are using MPTCP") {
            print("✅ MPTCP 正在工作")
        } else {
            print("❌ MPTCP 未生效")
        }
    }
}.resume()
```

#### 方法三：Network Link Conditioner

在 iOS 开发者设备上：

1. **设置 → 开发者 → Network Link Conditioner**
2. 创建自定义 Profile 模拟 Wi-Fi 劣化：
   - 设置较高的丢包率（如 20%）
   - 设置带宽限制（如 100 kbps）
3. 观察应用是否自动切换到蜂窝网络

#### 方法四：开发者设置禁用 Wi-Fi Assist 限制

在测试设备上：

**设置 → 开发者 → Networking → Multipath Networking**

启用后可以绕过 Wi-Fi Assist 的数据限制，更容易触发 MPTCP 行为。

### 6.2 服务端验证

#### 方法一：ss 命令查看 MPTCP 连接

```bash
# 查看所有 MPTCP 连接
ss -M

# 查看详细的 MPTCP 连接信息
ss -tMi

# 输出示例:
# ESTAB  0  0  192.168.1.100:443  10.0.0.50:54321
#   subflow  mptcp_info:(subflows:2, ... )
```

#### 方法二：nstat 统计

```bash
# 查看 MPTCP 相关统计
nstat -az | grep -i mptcp

# 关键指标:
# MPTcpExtMPCapableSYNRX    - 收到的 MP_CAPABLE SYN 数量
# MPTcpExtMPCapableSYNTX    - 发送的 MP_CAPABLE SYN 数量
# MPTcpExtMPCapableACKRX    - 收到的 MP_CAPABLE ACK 数量
# MPTcpExtMPJoinSynRx       - 收到的 MP_JOIN SYN 数量（新子流）
# MPTcpExtMPFallbackTokenInit - 回退到 TCP 的次数
```

#### 方法三：tcpdump 抓包分析

```bash
# 抓取 MPTCP 相关的 TCP 选项
tcpdump -i any -nn 'tcp[tcpflags] & (tcp-syn) != 0' -v port 443

# 使用 Wireshark 过滤器
# mptcp
# tcp.options.mptcp
# tcp.options.mptcp.subtype == 0   (MP_CAPABLE)
# tcp.options.mptcp.subtype == 1   (MP_JOIN)
```

#### 方法四：内核跟踪

```bash
# 使用 ftrace 跟踪 MPTCP 事件
echo 1 > /sys/kernel/debug/tracing/events/mptcp/enable
cat /sys/kernel/debug/tracing/trace_pipe

# 或使用 bpftrace
bpftrace -e 'tracepoint:mptcp:* { printf("%s\n", probe); }'
```

### 6.3 端到端验证方案

```
┌─────────────────────────────────────────────────────┐
│                   验证测试矩阵                        │
├──────────────┬──────────────────────────────────────┤
│  测试场景     │  验证方法                              │
├──────────────┼──────────────────────────────────────┤
│ MPTCP 协商   │ tcpdump 检查 MP_CAPABLE 选项          │
│ 成功         │ URLSessionTaskMetrics.isMultipath     │
│              │ amiusingmptcp.de 在线检测             │
├──────────────┼──────────────────────────────────────┤
│ 无缝切换     │ 1. 建立长连接传输大文件                │
│ (handover)   │ 2. 关闭 Wi-Fi                        │
│              │ 3. 确认传输不中断且无错误              │
│              │ 4. ss -M 确认子流切换                 │
├──────────────┼──────────────────────────────────────┤
│ 低延迟选择   │ 1. 使用 Network Link Conditioner      │
│ (interactive)│    人为增加 Wi-Fi 延迟                │
│              │ 2. 确认流量切换到蜂窝                 │
│              │ 3. 测量实际延迟变化                    │
├──────────────┼──────────────────────────────────────┤
│ 带宽聚合     │ 1. 使用 aggregate 模式下载大文件       │
│ (aggregate)  │ 2. 比较单路径 vs 多路径吞吐量         │
│              │ 3. nstat 确认两个接口均有流量          │
├──────────────┼──────────────────────────────────────┤
│ 降级兼容     │ 1. 连接不支持 MPTCP 的服务器          │
│              │ 2. 确认连接正常建立（降级为 TCP）      │
│              │ 3. tcpdump 确认 SYN 有 MP_CAPABLE     │
│              │    但 SYN+ACK 无 MPTCP 选项           │
├──────────────┼──────────────────────────────────────┤
│ 性能基准     │ 1. 分别测试各模式的吞吐量/延迟        │
│              │ 2. 与纯 TCP 进行 A/B 对比             │
│              │ 3. 在弱网环境下重复测试               │
└──────────────┴──────────────────────────────────────┘
```

### 6.4 自动化测试脚本（服务端）

```bash
#!/bin/bash
# test-mptcp-server.sh - 验证服务器 MPTCP 支持

SERVER_HOST="${1:-localhost}"
SERVER_PORT="${2:-443}"

echo "=== MPTCP 服务端验证 ==="
echo "目标: $SERVER_HOST:$SERVER_PORT"
echo ""

# 1. 检查内核支持
echo "[1/5] 检查内核 MPTCP 支持..."
if sysctl net.mptcp.enabled 2>/dev/null | grep -q "= 1"; then
    echo "  ✅ MPTCP 已在内核中启用"
else
    echo "  ❌ MPTCP 未启用"
fi

# 2. 检查端点配置
echo "[2/5] 检查端点配置..."
ENDPOINTS=$(ip mptcp endpoint show 2>/dev/null | wc -l)
echo "  已配置 $ENDPOINTS 个端点"
ip mptcp endpoint show 2>/dev/null | while read line; do
    echo "    $line"
done

# 3. 检查子流限制
echo "[3/5] 检查子流限制..."
ip mptcp limits show 2>/dev/null

# 4. 测试 MPTCP 连接（需要支持 MPTCP 的客户端工具）
echo "[4/5] 尝试建立 MPTCP 连接..."
if command -v mptcpize &> /dev/null; then
    mptcpize run curl -s -o /dev/null -w "HTTP状态码: %{http_code}\n" \
        "https://$SERVER_HOST:$SERVER_PORT/" --max-time 10 2>/dev/null
    echo "  (使用 mptcpize + curl 测试)"
else
    echo "  ⚠️ mptcpize 未安装，跳过连接测试"
    echo "  安装: apt install mptcpd"
fi

# 5. 检查 MPTCP 统计
echo "[5/5] MPTCP 统计..."
nstat -az 2>/dev/null | grep -i mptcp | head -20

echo ""
echo "=== 验证完成 ==="
```

---

## 7. 限制与注意事项

### 7.1 平台限制

| 限制 | 说明 |
|------|------|
| **仅 iOS** | macOS 上的 `URLSession` 不支持 `multipathServiceType` |
| **iOS 11+** | 需要 iOS 11.0 或更高版本 |
| **需要 Entitlement** | 必须在 Xcode 中添加 Multipath 权限 |
| **仅 URLSession** | 不适用于 `NWConnection` 等其他网络 API（NWConnection 有自己的多路径支持） |

### 7.2 Wi-Fi Assist 限制

- **后台限制**：Wi-Fi Assist 阻止应用在后台使用蜂窝数据进行 MPTCP
- **流量上限**：Wi-Fi Assist 限制应用通过蜂窝网络发送的数据量；达到上限时 MPTCP 被禁用
- **测试绕过**：在开发者设备上，可通过 **设置 → 开发者** 禁用数据限制

### 7.3 服务端要求

- 服务器**必须**支持 MPTCP，否则会降级为普通 TCP
- Linux 内核需要 **≥ 5.6** 且编译了 MPTCP 模块
- 负载均衡器/反向代理需要透传或支持 MPTCP
- 中间设备（防火墙、NAT）不能剥离 TCP 选项

### 7.4 安全考虑

- MPTCP 使用 HMAC-SHA256 进行子流认证，防止劫持
- 但增加了攻击面（多个 IP 地址暴露）
- 防火墙规则需要考虑 MPTCP 的 MP_JOIN 握手

### 7.5 常见问题排查

```
问题：MPTCP 不生效
├─ 检查：客户端是否添加了 Entitlement？
├─ 检查：URLSessionConfiguration 是否设置了 multipathServiceType？
├─ 检查：服务器是否支持 MPTCP？（用 tcpdump 检查 SYN+ACK）
├─ 检查：中间设备是否剥离了 TCP 选项？
└─ 检查：Wi-Fi Assist 是否已达到流量上限？

问题：连接频繁中断
├─ 检查：服务器的 MPTCP 路径管理器配置
├─ 检查：子流数限制是否过小
└─ 检查：防火墙是否阻止了 MP_JOIN

问题：蜂窝数据消耗过多
├─ 检查：是否使用了 aggregate 模式（改用 handover）
├─ 检查：Wi-Fi 信号是否持续不稳定
└─ 建议：在 Wi-Fi 稳定时使用 none 模式
```

---

## 8. 架构图

### 完整的 MPTCP 系统架构

```
                        ┌─────────────────────────────────┐
                        │         iOS 应用                  │
                        │                                   │
                        │   URLSessionConfiguration        │
                        │   .multipathServiceType = .handover│
                        │                                   │
                        │   ┌───────────────────────────┐  │
                        │   │      URLSession            │  │
                        │   │   (MPTCP 连接管理)         │  │
                        │   └─────────┬─────────────────┘  │
                        └─────────────┼────────────────────┘
                                      │
                    ┌─────────────────┼────────────────────┐
                    │                 │                      │
              ┌─────┴──────┐   ┌─────┴──────┐              │
              │  子流 1     │   │  子流 2     │              │
              │  (Wi-Fi)   │   │  (蜂窝)     │              │
              │  TCP + MPTCP│   │  TCP + MPTCP│              │
              └─────┬──────┘   └─────┬──────┘              │
                    │                 │                      │
              ┌─────┴──────┐   ┌─────┴──────┐              │
              │  Wi-Fi      │   │  蜂窝       │              │
              │  接口       │   │  接口       │              │
              │  (en0)     │   │  (pdp_ip0)  │              │
              └─────┬──────┘   └─────┬──────┘              │
                    │                 │                      │
                    │                 │                      │
         ┌──────────┘                 └───────────┐        │
         │                                         │        │
    ┌────┴────┐                              ┌────┴────┐   │
    │ Wi-Fi   │                              │ 蜂窝基站 │   │
    │ 路由器  │                              │         │   │
    └────┬────┘                              └────┬────┘   │
         │                                         │        │
         └──────────────┐         ┌───────────────┘        │
                        │         │                         │
                   ┌────┴─────────┴────┐                   │
                   │     互联网         │                   │
                   └─────────┬─────────┘                   │
                             │                              │
                   ┌─────────┴─────────┐                   │
                   │  负载均衡器         │                   │
                   │  (MPTCP 支持)      │                   │
                   │  如 Nginx + multipath │                │
                   └─────────┬─────────┘                   │
                             │                              │
                   ┌─────────┴─────────┐                   │
                   │  后端服务器         │                   │
                   │  Linux Kernel ≥ 5.6 │                  │
                   │  MPTCP enabled     │                   │
                   └───────────────────┘                   │
```

### Handover 模式时序

```
时间 ──────────────────────────────────────────────────→

Wi-Fi 信号   ████████████████████▓▓▓▓▒▒▒░░░░            
蜂窝 信号                               ░░░▒▒▒▓▓▓███████

连接状态:

子流1(Wi-Fi)  ═══════════════════════╗
                                      ╠══ 重叠切换期
子流2(蜂窝)                       ╔══╝══════════════════

数据流向    Wi-Fi ──────────→  双路径  ──────→ 蜂窝
            (主路径)          (切换中)        (主路径)

应用层视角  ─────────── 无感知，连续数据流 ──────────────
```

---

## 参考资料

- [Apple Developer: Improving network reliability using Multipath TCP](https://developer.apple.com/documentation/foundation/improving-network-reliability-using-multipath-tcp)
- [Apple Developer: URLSessionConfiguration.MultipathServiceType](https://developer.apple.com/documentation/foundation/urlsessionconfiguration/multipathservicetype-swift.enum)
- [RFC 8684: TCP Extensions for Multipath Operation with Multiple Addresses](https://www.rfc-editor.org/rfc/rfc8684.html)
- [Linux Kernel MPTCP Documentation](https://www.kernel.org/doc/html/v6.12/networking/mptcp.html)
- [MPTCP.dev - Multipath TCP for Linux](https://www.mptcp.dev/)
- [Nginx MPTCP Support (PR #130)](https://github.com/nginx/nginx/pull/130)
- [MultipathTCP iOS Sample (GitHub)](https://github.com/below/MultipathTCP)
