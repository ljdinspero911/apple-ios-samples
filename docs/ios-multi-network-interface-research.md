# iOS 多网卡（WiFi / 蜂窝网络切换与并发）方案调研

## 一、需求概述

当 WiFi 信号差或连接质量不佳时，自动或手动切换到蜂窝网络（4G/5G）发起请求，保证网络请求的可靠性和低延迟。核心诉求：

1. **检测 WiFi 质量**：判断当前 WiFi 是否可用、质量是否可接受
2. **蜂窝回退**：WiFi 不佳时切换到蜂窝网络
3. **无感切换**：尽量不中断正在进行的请求
4. **可控性**：能够针对特定请求指定使用哪种网络

---

## 二、方案总览

| 方案 | API 层级 | 最低系统版本 | 需要服务端支持 | 复杂度 | 推荐场景 |
|------|---------|------------|-------------|--------|---------|
| **方案 A：Multipath TCP (MPTCP)** | URLSession | iOS 11 | ✅ 需要 | 低 | 长连接、流媒体、IM |
| **方案 B：Network.framework + NWConnection** | Network.framework | iOS 12 | ❌ | 中 | 自定义 TCP/UDP/QUIC 通信 |
| **方案 C：NWPathMonitor 检测 + URLSession 切换** | Network + Foundation | iOS 12 | ❌ | 中 | HTTP API 请求 |
| **方案 D：Happy Eyeballs / 竞速连接** | Network.framework | iOS 12 | ❌ | 高 | 对延迟极度敏感的场景 |
| **方案 E：QUIC 连接迁移** | Network.framework | iOS 15 | ✅ 需要 | 中 | 新项目、可控服务端 |

---

## 三、各方案详细分析

### 方案 A：Multipath TCP (MPTCP)

#### 原理

MPTCP 是 TCP 的扩展（RFC 8684），允许一个 TCP 连接同时使用多条路径（WiFi + 蜂窝）。iOS 从 iOS 11 起在 URLSession 中原生支持。

系统会在 WiFi 上建立主连接，同时在蜂窝网络上建立备用连接。当 WiFi 不可用或响应慢时，自动切换到蜂窝路径，对应用层完全透明。

#### 实现方式

```swift
// 1. 在 Xcode 中启用 Multipath 能力
//    Signing & Capabilities → + Capability → Multipath
//    自动添加 entitlement: com.apple.developer.networking.multipath

// 2. 配置 URLSession
let config = URLSessionConfiguration.default

// 三种模式：
// .handover   — WiFi 为主，WiFi 不可用时切换到蜂窝（推荐）
// .interactive — 低延迟优先，可能同时使用两条路径
// .aggregate  — 聚合带宽（受限于 Apple 策略，实际效果有限）
config.multipathServiceType = .handover

let session = URLSession(configuration: config)

// 3. 正常发起请求，MPTCP 在传输层自动处理切换
let task = session.dataTask(with: url) { data, response, error in
    // ...
}
task.resume()
```

#### 前置条件

- Xcode 中启用 `com.apple.developer.networking.multipath` entitlement
- **服务端必须支持 MPTCP**（Linux 内核 5.6+ 原生支持，Nginx 1.21.4+ 支持）
- 设备蜂窝数据需开启
- 中间网络设备（防火墙、负载均衡器）需允许 TCP Option 30

#### 优点

- **实现最简单**：只需配置 URLSessionConfiguration，无需改动业务代码
- **系统级支持**：Apple Siri、Apple Music 已在使用
- **无感切换**：应用层完全透明

#### 缺点

- **服务端必须支持 MPTCP**：这是最大障碍，很多服务端不支持
- 中间设备可能剥离 TCP Option 30，导致回退到普通 TCP
- `.aggregate` 模式实际体验不如预期，Apple 对其限制较多
- 无法精细控制"什么时候该切换"

#### 参考

- [Apple 官方文档: Improving network reliability using Multipath TCP](https://developer.apple.com/documentation/foundation/improving-network-reliability-using-multipath-tcp)
- [Signal iOS MPTCP PR](https://github.com/signalapp/Signal-iOS/pull/5848)

---

### 方案 B：Network.framework + NWConnection 指定网络接口

#### 原理

通过 `NWParameters.requiredInterfaceType` 强制指定连接使用 `.cellular` 或 `.wifi` 接口。在应用层维护两条连接或在检测到 WiFi 差时主动建立蜂窝连接。

#### 实现方式

```swift
import Network

// 创建强制走蜂窝的连接
func createCellularConnection(host: String, port: UInt16) -> NWConnection {
    let params = NWParameters.tls
    params.requiredInterfaceType = .cellular
    params.prohibitExpensivePaths = false       // 蜂窝被系统标记为 expensive
    params.prohibitedInterfaceTypes = [.wifi]   // 明确禁止 WiFi
    
    let connection = NWConnection(
        host: NWEndpoint.Host(host),
        port: NWEndpoint.Port(rawValue: port)!,
        using: params
    )
    
    connection.stateUpdateHandler = { state in
        switch state {
        case .ready:
            print("蜂窝连接就绪")
        case .failed(let error):
            print("蜂窝连接失败: \(error)")
        case .waiting(let error):
            print("等待蜂窝网络: \(error)")
        default:
            break
        }
    }
    
    connection.start(queue: .global())
    return connection
}

// 创建强制走 WiFi 的连接
func createWiFiConnection(host: String, port: UInt16) -> NWConnection {
    let params = NWParameters.tls
    params.requiredInterfaceType = .wifi
    
    let connection = NWConnection(
        host: NWEndpoint.Host(host),
        port: NWEndpoint.Port(rawValue: port)!,
        using: params
    )
    connection.start(queue: .global())
    return connection
}
```

#### 封装 HTTP 请求（基于 NWConnection）

NWConnection 是传输层 API，不直接支持 HTTP。需要自行拼装 HTTP 报文或结合其他框架：

```swift
// 方式 1：手动拼装 HTTP 请求（适合简单场景）
func sendHTTPRequest(connection: NWConnection, 
                     method: String, 
                     path: String, 
                     host: String,
                     body: Data? = nil,
                     completion: @escaping (Data?, Error?) -> Void) {
    
    var request = "\(method) \(path) HTTP/1.1\r\n"
    request += "Host: \(host)\r\n"
    request += "Connection: close\r\n"
    if let body = body {
        request += "Content-Length: \(body.count)\r\n"
    }
    request += "\r\n"
    
    var requestData = request.data(using: .utf8)!
    if let body = body {
        requestData.append(body)
    }
    
    connection.send(content: requestData, completion: .contentProcessed { error in
        if let error = error {
            completion(nil, error)
            return
        }
        
        connection.receive(minimumIncompleteLength: 1, 
                          maximumLength: 65536) { data, _, _, error in
            completion(data, error)
        }
    })
}

// 方式 2：结合 URLProtocol（更实用，见方案 C）
```

#### 优点

- **不需要服务端修改**
- **精确控制使用哪个网络接口**
- 可以同时建立 WiFi 和蜂窝连接做竞速

#### 缺点

- **NWConnection 是传输层 API**，不直接支持 HTTP 语义（Header、Cookie、重定向、缓存等）
- 需要大量封装工作才能替代 URLSession 的能力
- 需自行处理 TLS/证书验证细节
- 不能限定具体接口名称（如 `pdp_ip0` vs `en0`），只能限定接口类型

---

### 方案 C：NWPathMonitor 检测 + URLSession 动态切换（推荐）

#### 原理

使用 `NWPathMonitor` 监控网络状态和质量，当 WiFi 不佳时，通过独立的 URLSession（配置 `allowsCellularAccess` / 自定义 DNS 解析 / 底层 socket 绑定）发起蜂窝请求。

这是**最实用的方案**，兼顾了 URLSession 丰富的 HTTP 能力和网络接口选择的灵活性。

#### 实现方式

```swift
import Network
import Foundation

// MARK: - 网络质量监控器

class NetworkQualityMonitor {
    
    private let pathMonitor = NWPathMonitor()
    private let wifiMonitor = NWPathMonitor(requiredInterfaceType: .wifi)
    private let cellularMonitor = NWPathMonitor(requiredInterfaceType: .cellular)
    private let queue = DispatchQueue(label: "NetworkMonitor")
    
    enum NetworkPreference {
        case wifi
        case cellular
        case any
    }
    
    private(set) var currentPreference: NetworkPreference = .wifi
    private(set) var isWiFiAvailable = false
    private(set) var isCellularAvailable = false
    
    var onPreferenceChanged: ((NetworkPreference) -> Void)?
    
    func start() {
        wifiMonitor.pathUpdateHandler = { [weak self] path in
            self?.isWiFiAvailable = (path.status == .satisfied)
            self?.evaluatePreference()
        }
        
        cellularMonitor.pathUpdateHandler = { [weak self] path in
            self?.isCellularAvailable = (path.status == .satisfied)
            self?.evaluatePreference()
        }
        
        wifiMonitor.start(queue: queue)
        cellularMonitor.start(queue: queue)
    }
    
    private func evaluatePreference() {
        let oldPreference = currentPreference
        
        if isWiFiAvailable && isCellularAvailable {
            // 两者都可用时，需要进一步判断 WiFi 质量
            // 可以通过测速、RTT 探测等手段
            currentPreference = .wifi  // 默认优先 WiFi
        } else if isWiFiAvailable {
            currentPreference = .wifi
        } else if isCellularAvailable {
            currentPreference = .cellular
        } else {
            currentPreference = .any
        }
        
        if oldPreference != currentPreference {
            onPreferenceChanged?(currentPreference)
        }
    }
    
    func stop() {
        wifiMonitor.cancel()
        cellularMonitor.cancel()
    }
}

// MARK: - 多网卡 HTTP 客户端

class MultiPathHTTPClient {
    
    private let monitor = NetworkQualityMonitor()
    
    // 默认 session（系统自动选择最优接口）
    private lazy var defaultSession: URLSession = {
        let config = URLSessionConfiguration.default
        config.timeoutIntervalForRequest = 15
        config.waitsForConnectivity = true
        return URLSession(configuration: config)
    }()
    
    // 仅蜂窝 session 的 NWConnection 池（用于底层回退）
    private let cellularQueue = DispatchQueue(label: "CellularHTTP")
    
    init() {
        monitor.start()
    }
    
    /// 发起带有自动回退逻辑的请求
    func request(_ urlRequest: URLRequest, 
                 timeout: TimeInterval = 10,
                 completion: @escaping (Data?, URLResponse?, Error?) -> Void) {
        
        let startTime = CFAbsoluteTimeGetCurrent()
        
        // 第一次尝试：使用默认（WiFi 优先）
        let task = defaultSession.dataTask(with: urlRequest) { 
            [weak self] data, response, error in
            
            let elapsed = CFAbsoluteTimeGetCurrent() - startTime
            
            // 判断是否需要回退到蜂窝
            if let self = self, self.shouldFallbackToCellular(
                error: error, elapsed: elapsed, timeout: timeout
            ) {
                self.requestViaCellular(urlRequest, completion: completion)
            } else {
                completion(data, response, error)
            }
        }
        task.resume()
    }
    
    private func shouldFallbackToCellular(error: Error?, 
                                           elapsed: TimeInterval,
                                           timeout: TimeInterval) -> Bool {
        guard monitor.isCellularAvailable else { return false }
        
        if let urlError = error as? URLError {
            switch urlError.code {
            case .timedOut, 
                 .cannotConnectToHost, 
                 .networkConnectionLost,
                 .notConnectedToInternet:
                return true
            default:
                return false
            }
        }
        
        // 如果请求虽然没出错但耗时过长，也可考虑回退
        // （此逻辑适合竞速模式，见方案 D）
        return false
    }
    
    /// 通过 NWConnection 走蜂窝发起请求
    private func requestViaCellular(_ urlRequest: URLRequest,
                                     completion: @escaping (Data?, URLResponse?, Error?) -> Void) {
        guard let url = urlRequest.url,
              let host = url.host else {
            completion(nil, nil, URLError(.badURL))
            return
        }
        
        let port = UInt16(url.port ?? (url.scheme == "https" ? 443 : 80))
        
        let tlsParams = NWParameters.tls
        tlsParams.requiredInterfaceType = .cellular
        tlsParams.prohibitExpensivePaths = false
        tlsParams.prohibitedInterfaceTypes = [.wifi]
        
        let connection = NWConnection(
            host: .init(host),
            port: .init(rawValue: port)!,
            using: tlsParams
        )
        
        connection.stateUpdateHandler = { state in
            switch state {
            case .ready:
                // 连接就绪，发送 HTTP 请求
                let httpData = self.buildHTTPRequest(from: urlRequest)
                connection.send(content: httpData, completion: .contentProcessed { error in
                    if let error = error {
                        completion(nil, nil, error)
                        connection.cancel()
                        return
                    }
                    self.receiveHTTPResponse(connection: connection, completion: completion)
                })
            case .failed(let error):
                completion(nil, nil, error)
            default:
                break
            }
        }
        
        connection.start(queue: cellularQueue)
    }
    
    private func buildHTTPRequest(from request: URLRequest) -> Data {
        // 简化实现，生产环境需要完整的 HTTP 报文构建
        let method = request.httpMethod ?? "GET"
        let path = request.url?.path ?? "/"
        let host = request.url?.host ?? ""
        
        var raw = "\(method) \(path) HTTP/1.1\r\nHost: \(host)\r\n"
        request.allHTTPHeaderFields?.forEach { key, value in
            raw += "\(key): \(value)\r\n"
        }
        if let body = request.httpBody {
            raw += "Content-Length: \(body.count)\r\n"
        }
        raw += "\r\n"
        
        var data = raw.data(using: .utf8)!
        if let body = request.httpBody {
            data.append(body)
        }
        return data
    }
    
    private func receiveHTTPResponse(connection: NWConnection,
                                      completion: @escaping (Data?, URLResponse?, Error?) -> Void) {
        connection.receive(minimumIncompleteLength: 1, maximumLength: 65536) { data, _, isComplete, error in
            completion(data, nil, error)
            if isComplete {
                connection.cancel()
            }
        }
    }
}
```

#### WiFi 质量检测策略

```swift
// MARK: - WiFi 质量探测

class WiFiQualityProber {
    
    /// 通过小包 RTT 探测 WiFi 质量
    func probeWiFiLatency(host: String, 
                          port: UInt16 = 443,
                          completion: @escaping (TimeInterval?) -> Void) {
        let params = NWParameters.tcp
        params.requiredInterfaceType = .wifi
        
        let start = CFAbsoluteTimeGetCurrent()
        let connection = NWConnection(
            host: .init(host),
            port: .init(rawValue: port)!,
            using: params
        )
        
        connection.stateUpdateHandler = { state in
            switch state {
            case .ready:
                let rtt = CFAbsoluteTimeGetCurrent() - start
                completion(rtt)
                connection.cancel()
            case .failed:
                completion(nil) // WiFi 不可达
                connection.cancel()
            case .waiting:
                // WiFi 很慢，仍在等待
                DispatchQueue.global().asyncAfter(deadline: .now() + 3) {
                    completion(nil)
                    connection.cancel()
                }
            default:
                break
            }
        }
        
        connection.start(queue: .global())
    }
    
    /// 综合评估：RTT > 阈值 或 连接失败，则建议切换蜂窝
    func evaluateAndDecide(host: String,
                           rttThreshold: TimeInterval = 2.0,
                           completion: @escaping (Bool) -> Void) {
        probeWiFiLatency(host: host) { rtt in
            if let rtt = rtt, rtt < rttThreshold {
                completion(false) // WiFi 尚可，不需要切换
            } else {
                completion(true)  // 建议切换到蜂窝
            }
        }
    }
}
```

#### 优点

- **不需要服务端修改**
- **保留 URLSession 的全部 HTTP 能力**用于默认路径
- 可自定义切换策略（RTT、错误率、超时时间等）
- 渐进式集成，不影响现有代码

#### 缺点

- 蜂窝回退路径使用 NWConnection，需自行处理 HTTP 协议
- 切换有延迟（先失败再重试），不如 MPTCP 平滑
- WiFi 质量检测本身也需要消耗时间和流量

---

### 方案 D：Happy Eyeballs / 竞速连接

#### 原理

同时在 WiFi 和蜂窝上发起连接，取先成功的那个。类似于 DNS Happy Eyeballs（RFC 8305）的思路。

#### 实现方式

```swift
class RacingConnectionManager {
    
    /// 在 WiFi 和蜂窝上同时发起连接，取先成功者
    func raceConnection(host: String, 
                        port: UInt16,
                        completion: @escaping (NWConnection?, NWInterface.InterfaceType?) -> Void) {
        
        var completed = false
        let lock = NSLock()
        
        func completeOnce(_ connection: NWConnection, _ type: NWInterface.InterfaceType) {
            lock.lock()
            defer { lock.unlock() }
            guard !completed else {
                connection.cancel()
                return
            }
            completed = true
            completion(connection, type)
        }
        
        // WiFi 路径
        let wifiParams = NWParameters.tls
        wifiParams.requiredInterfaceType = .wifi
        let wifiConn = NWConnection(host: .init(host), port: .init(rawValue: port)!, using: wifiParams)
        
        wifiConn.stateUpdateHandler = { state in
            if case .ready = state {
                completeOnce(wifiConn, .wifi)
            }
        }
        
        // 蜂窝路径
        let cellParams = NWParameters.tls
        cellParams.requiredInterfaceType = .cellular
        cellParams.prohibitExpensivePaths = false
        let cellConn = NWConnection(host: .init(host), port: .init(rawValue: port)!, using: cellParams)
        
        cellConn.stateUpdateHandler = { state in
            if case .ready = state {
                completeOnce(cellConn, .cellular)
            }
        }
        
        let queue = DispatchQueue(label: "RacingConnection")
        wifiConn.start(queue: queue)
        
        // 延迟 250ms 启动蜂窝（给 WiFi 一个先机）
        queue.asyncAfter(deadline: .now() + 0.25) {
            cellConn.start(queue: queue)
        }
    }
}
```

#### 优点

- 延迟最低：取最快路径
- 对用户体验影响最小

#### 缺点

- **浪费流量**：每次请求消耗双倍连接建立成本
- 实现复杂度高
- 同样面临 NWConnection 不直接支持 HTTP 的问题
- 需要妥善处理失败连接的清理

---

### 方案 E：QUIC 连接迁移

#### 原理

QUIC 协议天然支持连接迁移（Connection Migration），因为 QUIC 连接用 Connection ID 标识，不依赖四元组（IP:Port 对）。当设备从 WiFi 切换到蜂窝时，QUIC 可以无缝迁移。

#### 实现方式

```swift
import Network

// iOS 15+ 支持 NWProtocolQUIC
func createQUICConnection(host: String, port: UInt16) -> NWConnection? {
    guard #available(iOS 15, *) else { return nil }
    
    let quicParams = NWParameters.quic(alpn: ["h3"])  // HTTP/3
    // QUIC 连接迁移由系统自动处理
    
    let connection = NWConnection(
        host: .init(host),
        port: .init(rawValue: port)!,
        using: quicParams
    )
    
    connection.betterPathUpdateHandler = { betterPathAvailable in
        if betterPathAvailable {
            // 系统发现了更好的路径（如从差的 WiFi 到好的蜂窝）
            // QUIC 会自动迁移
            print("发现更好的网络路径，自动迁移中...")
        }
    }
    
    connection.start(queue: .global())
    return connection
}
```

#### 优点

- QUIC 天然支持连接迁移，无需应用层处理
- 0-RTT 重连，切换速度极快
- HTTP/3 是未来趋势

#### 缺点

- **需要服务端支持 HTTP/3 / QUIC**
- iOS 15+ 才能使用
- NWProtocolQUIC 的 API 仍在演进中
- 对既有 HTTP/1.1、HTTP/2 基础设施改造较大

---

## 四、方案选型建议

### 决策树

```
是否可以改造服务端？
├── 是
│   ├── 服务端能支持 MPTCP？
│   │   ├── 是 → 方案 A（MPTCP），最简单
│   │   └── 否，但能支持 HTTP/3？
│   │       ├── 是 → 方案 E（QUIC 连接迁移）
│   │       └── 否 → 方案 C（NWPathMonitor + 回退）
│   └──
└── 否（不能改服务端）
    ├── 只需要 HTTP 请求？
    │   ├── 是 → 方案 C（NWPathMonitor + 回退）
    │   └── 否（自定义协议）→ 方案 B（NWConnection 直接指定接口）
    └── 对延迟极度敏感？
        └── 是 → 方案 D（竞速连接）
```

### 推荐方案：方案 C（NWPathMonitor + URLSession 回退）

对于大多数 iOS 项目，**方案 C** 是最务实的选择：

1. **不需要服务端改造**
2. **主路径保持 URLSession 全部能力**（Cookie、缓存、重定向、证书验证、后台传输）
3. **回退路径可按需扩展**：可以先实现简单的超时重试，后续再优化为 NWConnection 蜂窝直连
4. **渐进式接入**：可以只对关键 API 启用多网卡逻辑

### 推荐的渐进式实施路径

```
阶段 1：网络监控基础设施
  ├── 集成 NWPathMonitor，监控 WiFi / 蜂窝可用性
  ├── 实现 WiFi 质量探测（TCP 连接 RTT）
  └── 建立网络状态上报机制

阶段 2：超时回退
  ├── 对关键 API 实现"WiFi 超时后用蜂窝重试"
  ├── 使用两个 URLSession：默认 + 仅蜂窝（通过 CONNECT 代理或 NWConnection）
  └── 收集回退触发率数据

阶段 3：智能切换
  ├── 基于历史 RTT 和错误率建立 WiFi 质量评分模型
  ├── 对评分低于阈值的 WiFi 直接走蜂窝
  └── 引入竞速策略（方案 D）用于首屏请求

阶段 4（可选）：服务端协同
  ├── 服务端启用 MPTCP → 对核心链路启用方案 A
  └── 或服务端启用 HTTP/3 → 利用 QUIC 连接迁移
```

---

## 五、关键注意事项

### 1. Apple 审核

- 使用 `NWPathMonitor`、`NWConnection`、`URLSession.multipathServiceType` 均为公开 API，**不会触发审核问题**
- **MPTCP 需要 entitlement**：`com.apple.developer.networking.multipath`，需在 Apple Developer Portal 申请
- 不要使用私有 API（如 `CTCellularData` 的私有方法）来绕过系统网络选择

### 2. 用户流量消耗

- 蜂窝回退会消耗用户流量，**必须提供用户开关**
- 建议参考系统的"WiFi 助理"（WiFi Assist）功能设计：
  - 默认关闭或仅在 WiFi 极差时启用
  - 明确告知用户可能使用蜂窝数据
  - 提供流量统计

### 3. 低电量模式

- `NWPathMonitor` 的 `path.isConstrained` 为 `true` 时表示低数据模式
- 低电量模式下应减少探测频率，避免蜂窝回退

### 4. VPN 场景

- 当 VPN 开启时，所有流量可能被路由到 VPN 隧道
- `NWConnection` 的 `requiredInterfaceType` 在 VPN 环境下行为可能不同
- 需要测试 VPN 开启时多网卡逻辑是否正常

### 5. 双卡设备

- iPhone 支持双卡（双 SIM），蜂窝接口可能对应不同运营商
- `NWParameters.requiredInterfaceType = .cellular` 会使用系统默认的蜂窝接口
- 不能通过公开 API 指定使用哪张 SIM 卡

---

## 六、相关 Apple 文档与资源

| 资源 | 链接 |
|------|-----|
| Improving network reliability using Multipath TCP | https://developer.apple.com/documentation/foundation/improving-network-reliability-using-multipath-tcp |
| NWConnection | https://developer.apple.com/documentation/network/nwconnection |
| NWParameters | https://developer.apple.com/documentation/network/nwparameters |
| NWPathMonitor | https://developer.apple.com/documentation/network/nwpathmonitor |
| NWProtocolQUIC | https://developer.apple.com/documentation/network/nwprotocolquic |
| WWDC: Adapt to changing network conditions | https://developer.apple.com/videos/play/tech-talks/111378/ |
| URLSessionConfiguration.multipathServiceType | https://developer.apple.com/documentation/foundation/urlsessionconfiguration/multipathservicetype |

---

## 七、本仓库已有相关参考代码

本仓库（Apple iOS 示例代码合集）中包含以下网络相关示例：

- `Reachability/` — 经典的网络可达性检测，可用于判断 WiFi / 蜂窝是否可用（已被 `NWPathMonitor` 取代）
- `SimpleURLConnections/` — URLConnection 基础用法
- `AdvancedURLConnections/` — URL 连接高级用法（认证、证书）
- `CustomHTTPProtocol/` — 自定义 URLProtocol 拦截流量，可参考用于实现蜂窝回退拦截器
- `SimpleNetworkStreams/` — NSStream 网络通信
- `SimpleTunnelCustomizedNetworkingUsingtheNetworkExtensionFramework/` — Network Extension 隧道实现，展示了底层网络控制的模式
