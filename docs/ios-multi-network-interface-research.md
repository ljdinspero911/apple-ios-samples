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

| **方案 F：libcurl + CURLOPT_INTERFACE** | libcurl (C) | iOS 9+ | ❌ | 中 | 已有 libcurl 基础设施、跨平台项目 |

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

### 方案 F：libcurl + CURLOPT_INTERFACE 指定网卡（重点方案）

#### 原理

libcurl 提供 `CURLOPT_INTERFACE` 选项，可以将出站连接绑定到指定的网络接口名或 IP 地址。在 iOS 上，WiFi 接口名为 `en0`，蜂窝接口名为 `pdp_ip0`（可能还有 `pdp_ip1` 等）。通过 `getifaddrs()` 枚举当前可用接口和对应 IP 地址，然后用 `CURLOPT_INTERFACE` 将 curl 请求绑定到目标接口，实现完整的 HTTP 请求（含 Header、Cookie、重定向、TLS）且指定网卡。

#### iOS 上的网络接口名称

| 接口名 | 含义 |
|--------|------|
| `en0` | WiFi |
| `pdp_ip0` | 蜂窝数据（主卡，Packet Data Protocol） |
| `pdp_ip1` / `pdp_ip2` / `pdp_ip3` | 蜂窝数据（其他通道 / 副卡） |
| `lo0` | 本地回环 |
| `awdl0` | Apple Wireless Direct Link |
| `ap1` | 热点 Access Point |

> **注意**：Apple 官方声明 BSD 接口名不是 API，不保证永远不变。但实际上 `en0` (WiFi) 和 `pdp_ip0` (蜂窝) 在所有 iOS 版本中一直稳定使用。

#### 集成 libcurl 到 iOS 项目

**方式 1：预编译 XCFramework（推荐）**

使用 [curl-apple](https://github.com/nicerobot/curl-apple) 或 [libcurl-ios-prebuilt](https://github.com/nicerobot/libcurl-ios-prebuilt-and-buildscripts) 提供的构建脚本：

```bash
# 编译 libcurl 静态库 for iOS (arm64) + 模拟器 (arm64 + x86_64)
# 支持选项：Secure Transport / OpenSSL、HTTP/2、zlib 等
./build.sh --target=ios --tls=secure-transport --enable-http2

# 输出 libcurl.xcframework，拖入 Xcode 即可
```

**方式 2：CocoaPods**

```ruby
# Podfile
pod 'SwiftyCurl', '~> 0.5'
```

**方式 3：手动编译**

```bash
# 交叉编译 for iOS arm64
export CC=$(xcrun -sdk iphoneos -find clang)
export CFLAGS="-arch arm64 -isysroot $(xcrun -sdk iphoneos --show-sdk-path) -miphoneos-version-min=13.0"
./configure --host=arm-apple-darwin --with-secure-transport --enable-static --disable-shared
make
```

#### 完整实现代码

##### 1. 网络接口枚举工具（C / Objective-C）

```c
// CURLInterfaceHelper.h

#import <Foundation/Foundation.h>

typedef NS_ENUM(NSInteger, MPNetworkType) {
    MPNetworkTypeWiFi,
    MPNetworkTypeCellular,
    MPNetworkTypeAny
};

@interface MPNetworkInterface : NSObject
@property (nonatomic, copy) NSString *name;       // e.g. "en0", "pdp_ip0"
@property (nonatomic, copy) NSString *ipv4Address;
@property (nonatomic, copy, nullable) NSString *ipv6Address;
@property (nonatomic, assign) MPNetworkType type;
@property (nonatomic, assign, getter=isUp) BOOL up;
@end

@interface MPInterfaceDetector : NSObject
+ (NSArray<MPNetworkInterface *> *)availableInterfaces;
+ (nullable MPNetworkInterface *)wifiInterface;
+ (nullable MPNetworkInterface *)cellularInterface;
+ (nullable NSString *)interfaceNameForType:(MPNetworkType)type;
+ (nullable NSString *)ipAddressForType:(MPNetworkType)type;
@end
```

```objc
// CURLInterfaceHelper.m

#import "CURLInterfaceHelper.h"
#include <ifaddrs.h>
#include <arpa/inet.h>
#include <net/if.h>

@implementation MPNetworkInterface
@end

@implementation MPInterfaceDetector

+ (NSArray<MPNetworkInterface *> *)availableInterfaces {
    NSMutableArray<MPNetworkInterface *> *result = [NSMutableArray array];
    NSMutableDictionary<NSString *, MPNetworkInterface *> *map = [NSMutableDictionary dictionary];
    
    struct ifaddrs *interfaces = NULL;
    if (getifaddrs(&interfaces) != 0) {
        return result;
    }
    
    for (struct ifaddrs *addr = interfaces; addr != NULL; addr = addr->ifa_next) {
        if (addr->ifa_addr == NULL) continue;
        
        NSString *name = [NSString stringWithUTF8String:addr->ifa_name];
        sa_family_t family = addr->ifa_addr->sa_family;
        
        if (family != AF_INET && family != AF_INET6) continue;
        
        MPNetworkInterface *iface = map[name];
        if (!iface) {
            iface = [[MPNetworkInterface alloc] init];
            iface.name = name;
            iface.up = (addr->ifa_flags & IFF_UP) && (addr->ifa_flags & IFF_RUNNING);
            
            if ([name isEqualToString:@"en0"]) {
                iface.type = MPNetworkTypeWiFi;
            } else if ([name hasPrefix:@"pdp_ip"]) {
                iface.type = MPNetworkTypeCellular;
            } else {
                iface.type = MPNetworkTypeAny;
            }
            
            map[name] = iface;
            [result addObject:iface];
        }
        
        char addrBuf[INET6_ADDRSTRLEN];
        if (family == AF_INET) {
            struct sockaddr_in *sin = (struct sockaddr_in *)addr->ifa_addr;
            inet_ntop(AF_INET, &sin->sin_addr, addrBuf, sizeof(addrBuf));
            iface.ipv4Address = [NSString stringWithUTF8String:addrBuf];
        } else if (family == AF_INET6) {
            struct sockaddr_in6 *sin6 = (struct sockaddr_in6 *)addr->ifa_addr;
            inet_ntop(AF_INET6, &sin6->sin6_addr, addrBuf, sizeof(addrBuf));
            iface.ipv6Address = [NSString stringWithUTF8String:addrBuf];
        }
    }
    
    freeifaddrs(interfaces);
    return result;
}

+ (nullable MPNetworkInterface *)wifiInterface {
    for (MPNetworkInterface *iface in [self availableInterfaces]) {
        if (iface.type == MPNetworkTypeWiFi && iface.isUp && iface.ipv4Address) {
            return iface;
        }
    }
    return nil;
}

+ (nullable MPNetworkInterface *)cellularInterface {
    for (MPNetworkInterface *iface in [self availableInterfaces]) {
        if (iface.type == MPNetworkTypeCellular && iface.isUp && iface.ipv4Address) {
            return iface;
        }
    }
    return nil;
}

+ (nullable NSString *)interfaceNameForType:(MPNetworkType)type {
    switch (type) {
        case MPNetworkTypeWiFi:
            return [self wifiInterface].name;
        case MPNetworkTypeCellular:
            return [self cellularInterface].name;
        case MPNetworkTypeAny:
            return nil;
    }
}

+ (nullable NSString *)ipAddressForType:(MPNetworkType)type {
    switch (type) {
        case MPNetworkTypeWiFi:
            return [self wifiInterface].ipv4Address;
        case MPNetworkTypeCellular:
            return [self cellularInterface].ipv4Address;
        case MPNetworkTypeAny:
            return nil;
    }
}

@end
```

##### 2. libcurl HTTP 客户端（核心实现）

```objc
// MPCurlHTTPClient.h

#import <Foundation/Foundation.h>
#import "CURLInterfaceHelper.h"

@class MPCurlResponse;

typedef void (^MPCurlCompletion)(MPCurlResponse * _Nullable response, NSError * _Nullable error);

@interface MPCurlResponse : NSObject
@property (nonatomic, assign) NSInteger statusCode;
@property (nonatomic, copy) NSDictionary<NSString *, NSString *> *headers;
@property (nonatomic, copy) NSData *body;
@property (nonatomic, copy) NSString *usedInterface;  // 实际使用的网卡
@property (nonatomic, assign) double totalTime;        // 总耗时（秒）
@property (nonatomic, assign) double connectTime;      // 连接耗时
@property (nonatomic, assign) double nameLookupTime;   // DNS 耗时
@end

@interface MPCurlHTTPClient : NSObject

/// 指定网卡发起 GET 请求
- (void)GET:(NSString *)url
  interface:(MPNetworkType)networkType
 completion:(MPCurlCompletion)completion;

/// 指定网卡发起 POST 请求
- (void)POST:(NSString *)url
     headers:(nullable NSDictionary<NSString *, NSString *> *)headers
        body:(nullable NSData *)body
   interface:(MPNetworkType)networkType
  completion:(MPCurlCompletion)completion;

/// 带自动回退的请求：先走 WiFi，超时/失败后自动切蜂窝
- (void)requestWithFallback:(NSString *)url
                     method:(NSString *)method
                    headers:(nullable NSDictionary<NSString *, NSString *> *)headers
                       body:(nullable NSData *)body
               wifiTimeout:(NSTimeInterval)wifiTimeout
                 completion:(MPCurlCompletion)completion;

@end
```

```objc
// MPCurlHTTPClient.m

#import "MPCurlHTTPClient.h"
#include <curl/curl.h>

#pragma mark - Write/Header Callbacks

struct CurlWriteBuffer {
    char *data;
    size_t size;
};

static size_t curl_write_callback(void *contents, size_t size, size_t nmemb, void *userp) {
    size_t totalSize = size * nmemb;
    struct CurlWriteBuffer *buf = (struct CurlWriteBuffer *)userp;
    
    char *ptr = realloc(buf->data, buf->size + totalSize + 1);
    if (!ptr) return 0;
    
    buf->data = ptr;
    memcpy(&(buf->data[buf->size]), contents, totalSize);
    buf->size += totalSize;
    buf->data[buf->size] = 0;
    
    return totalSize;
}

static size_t curl_header_callback(char *buffer, size_t size, size_t nitems, void *userdata) {
    size_t totalSize = size * nitems;
    NSMutableDictionary *headers = (__bridge NSMutableDictionary *)userdata;
    
    NSString *line = [[NSString alloc] initWithBytes:buffer length:totalSize encoding:NSUTF8StringEncoding];
    if (!line) return totalSize;
    
    NSRange colonRange = [line rangeOfString:@":"];
    if (colonRange.location != NSNotFound) {
        NSString *key = [[line substringToIndex:colonRange.location]
                         stringByTrimmingCharactersInSet:[NSCharacterSet whitespaceAndNewlineCharacterSet]];
        NSString *value = [[line substringFromIndex:colonRange.location + 1]
                           stringByTrimmingCharactersInSet:[NSCharacterSet whitespaceAndNewlineCharacterSet]];
        if (key.length > 0) {
            headers[key] = value;
        }
    }
    
    return totalSize;
}

#pragma mark - MPCurlResponse

@implementation MPCurlResponse
@end

#pragma mark - MPCurlHTTPClient

@implementation MPCurlHTTPClient {
    dispatch_queue_t _curlQueue;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _curlQueue = dispatch_queue_create("com.multipath.curl", DISPATCH_QUEUE_CONCURRENT);
        curl_global_init(CURL_GLOBAL_DEFAULT);
    }
    return self;
}

- (void)dealloc {
    curl_global_cleanup();
}

#pragma mark - Public API

- (void)GET:(NSString *)url
  interface:(MPNetworkType)networkType
 completion:(MPCurlCompletion)completion {
    [self performRequest:url
                  method:@"GET"
                 headers:nil
                    body:nil
               interface:networkType
                 timeout:30
              completion:completion];
}

- (void)POST:(NSString *)url
     headers:(NSDictionary<NSString *, NSString *> *)headers
        body:(NSData *)body
   interface:(MPNetworkType)networkType
  completion:(MPCurlCompletion)completion {
    [self performRequest:url
                  method:@"POST"
                 headers:headers
                    body:body
               interface:networkType
                 timeout:30
              completion:completion];
}

- (void)requestWithFallback:(NSString *)url
                     method:(NSString *)method
                    headers:(NSDictionary<NSString *, NSString *> *)headers
                       body:(NSData *)body
               wifiTimeout:(NSTimeInterval)wifiTimeout
                 completion:(MPCurlCompletion)completion {
    
    // 先尝试 WiFi
    [self performRequest:url
                  method:method
                 headers:headers
                    body:body
               interface:MPNetworkTypeWiFi
                 timeout:wifiTimeout
              completion:^(MPCurlResponse *response, NSError *error) {
        
        if (error && [MPInterfaceDetector cellularInterface]) {
            NSLog(@"[MultiPath] WiFi request failed (%@), falling back to cellular", error.localizedDescription);
            
            // WiFi 失败，回退到蜂窝
            [self performRequest:url
                          method:method
                         headers:headers
                            body:body
                       interface:MPNetworkTypeCellular
                         timeout:30
                      completion:completion];
        } else {
            completion(response, error);
        }
    }];
}

#pragma mark - Core curl execution

- (void)performRequest:(NSString *)url
                method:(NSString *)method
               headers:(NSDictionary<NSString *, NSString *> *)headers
                  body:(NSData *)body
             interface:(MPNetworkType)networkType
               timeout:(NSTimeInterval)timeout
            completion:(MPCurlCompletion)completion {
    
    dispatch_async(_curlQueue, ^{
        CURL *curl = curl_easy_init();
        if (!curl) {
            NSError *err = [NSError errorWithDomain:@"MPCurl" code:-1 userInfo:@{
                NSLocalizedDescriptionKey: @"curl_easy_init failed"
            }];
            dispatch_async(dispatch_get_main_queue(), ^{ completion(nil, err); });
            return;
        }
        
        // ---- URL ----
        curl_easy_setopt(curl, CURLOPT_URL, [url UTF8String]);
        
        // ---- 指定网络接口（核心） ----
        NSString *interfaceName = [MPInterfaceDetector interfaceNameForType:networkType];
        NSString *interfaceIP = [MPInterfaceDetector ipAddressForType:networkType];
        
        if (interfaceName) {
            // 方式 1：用接口名绑定（推荐）
            NSString *ifSpec = [NSString stringWithFormat:@"if!%@", interfaceName];
            curl_easy_setopt(curl, CURLOPT_INTERFACE, [ifSpec UTF8String]);
            
            NSLog(@"[MultiPath] Binding to interface: %@ (%@)", interfaceName, interfaceIP ?: @"no IP");
        } else if (interfaceIP) {
            // 方式 2：用 IP 地址绑定
            NSString *hostSpec = [NSString stringWithFormat:@"host!%@", interfaceIP];
            curl_easy_setopt(curl, CURLOPT_INTERFACE, [hostSpec UTF8String]);
        }
        // networkType == MPNetworkTypeAny 时不设置 CURLOPT_INTERFACE，由系统路由
        
        // ---- Method ----
        if ([method isEqualToString:@"POST"]) {
            curl_easy_setopt(curl, CURLOPT_POST, 1L);
            if (body) {
                curl_easy_setopt(curl, CURLOPT_POSTFIELDS, body.bytes);
                curl_easy_setopt(curl, CURLOPT_POSTFIELDSIZE, (long)body.length);
            }
        } else if ([method isEqualToString:@"PUT"]) {
            curl_easy_setopt(curl, CURLOPT_CUSTOMREQUEST, "PUT");
            if (body) {
                curl_easy_setopt(curl, CURLOPT_POSTFIELDS, body.bytes);
                curl_easy_setopt(curl, CURLOPT_POSTFIELDSIZE, (long)body.length);
            }
        } else if ([method isEqualToString:@"DELETE"]) {
            curl_easy_setopt(curl, CURLOPT_CUSTOMREQUEST, "DELETE");
        }
        // GET is the default
        
        // ---- Headers ----
        struct curl_slist *headerList = NULL;
        for (NSString *key in headers) {
            NSString *header = [NSString stringWithFormat:@"%@: %@", key, headers[key]];
            headerList = curl_slist_append(headerList, [header UTF8String]);
        }
        if (headerList) {
            curl_easy_setopt(curl, CURLOPT_HTTPHEADER, headerList);
        }
        
        // ---- TLS (使用系统 CA 证书) ----
        curl_easy_setopt(curl, CURLOPT_SSL_VERIFYPEER, 1L);
        curl_easy_setopt(curl, CURLOPT_SSL_VERIFYHOST, 2L);
        // iOS Secure Transport 后端会自动使用系统信任链
        
        // ---- 超时 ----
        curl_easy_setopt(curl, CURLOPT_TIMEOUT, (long)timeout);
        curl_easy_setopt(curl, CURLOPT_CONNECTTIMEOUT, (long)MIN(timeout, 10));
        
        // ---- Follow redirects ----
        curl_easy_setopt(curl, CURLOPT_FOLLOWLOCATION, 1L);
        curl_easy_setopt(curl, CURLOPT_MAXREDIRS, 10L);
        
        // ---- Response body buffer ----
        struct CurlWriteBuffer bodyBuf = { .data = malloc(1), .size = 0 };
        curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, curl_write_callback);
        curl_easy_setopt(curl, CURLOPT_WRITEDATA, &bodyBuf);
        
        // ---- Response headers ----
        NSMutableDictionary *respHeaders = [NSMutableDictionary dictionary];
        curl_easy_setopt(curl, CURLOPT_HEADERFUNCTION, curl_header_callback);
        curl_easy_setopt(curl, CURLOPT_HEADERDATA, (__bridge void *)respHeaders);
        
        // ---- DNS 缓存（减少重复解析开销） ----
        curl_easy_setopt(curl, CURLOPT_DNS_CACHE_TIMEOUT, 300L);
        
        // ---- Execute ----
        CURLcode res = curl_easy_perform(curl);
        
        if (res != CURLE_OK) {
            free(bodyBuf.data);
            if (headerList) curl_slist_free_all(headerList);
            
            const char *errStr = curl_easy_strerror(res);
            NSError *err = [NSError errorWithDomain:@"MPCurl" code:res userInfo:@{
                NSLocalizedDescriptionKey: [NSString stringWithUTF8String:errStr],
                @"CURLcode": @(res),
                @"interface": interfaceName ?: @"default"
            }];
            
            curl_easy_cleanup(curl);
            dispatch_async(dispatch_get_main_queue(), ^{ completion(nil, err); });
            return;
        }
        
        // ---- Build response ----
        MPCurlResponse *response = [[MPCurlResponse alloc] init];
        
        long httpCode = 0;
        curl_easy_getinfo(curl, CURLINFO_RESPONSE_CODE, &httpCode);
        response.statusCode = httpCode;
        
        response.headers = [respHeaders copy];
        response.body = [NSData dataWithBytes:bodyBuf.data length:bodyBuf.size];
        response.usedInterface = interfaceName ?: @"default";
        
        double totalTime = 0, connectTime = 0, dnsTime = 0;
        curl_easy_getinfo(curl, CURLINFO_TOTAL_TIME, &totalTime);
        curl_easy_getinfo(curl, CURLINFO_CONNECT_TIME, &connectTime);
        curl_easy_getinfo(curl, CURLINFO_NAMELOOKUP_TIME, &dnsTime);
        response.totalTime = totalTime;
        response.connectTime = connectTime;
        response.nameLookupTime = dnsTime;
        
        // ---- Cleanup ----
        free(bodyBuf.data);
        if (headerList) curl_slist_free_all(headerList);
        curl_easy_cleanup(curl);
        
        dispatch_async(dispatch_get_main_queue(), ^{ completion(response, nil); });
    });
}

@end
```

##### 3. 带智能回退的高层封装

```objc
// MPSmartHTTPClient.h — 智能多网卡 HTTP 客户端

#import <Foundation/Foundation.h>
#import "MPCurlHTTPClient.h"

/// 请求策略
typedef NS_ENUM(NSInteger, MPRequestStrategy) {
    MPRequestStrategyWiFiOnly,          // 只走 WiFi
    MPRequestStrategyCellularOnly,      // 只走蜂窝
    MPRequestStrategyWiFiWithFallback,  // WiFi 优先，失败回退蜂窝
    MPRequestStrategyFastestWins,       // WiFi 和蜂窝竞速，取先到者
    MPRequestStrategyAuto,             // 根据当前网络质量自动选择
};

@interface MPSmartHTTPClient : NSObject

@property (nonatomic, assign) NSTimeInterval wifiFallbackTimeout; // WiFi 超时阈值，默认 5s
@property (nonatomic, assign) NSTimeInterval cellularTimeout;     // 蜂窝超时，默认 30s

- (void)request:(NSString *)url
         method:(NSString *)method
        headers:(nullable NSDictionary<NSString *, NSString *> *)headers
           body:(nullable NSData *)body
       strategy:(MPRequestStrategy)strategy
     completion:(MPCurlCompletion)completion;

@end
```

```objc
// MPSmartHTTPClient.m

#import "MPSmartHTTPClient.h"

@implementation MPSmartHTTPClient {
    MPCurlHTTPClient *_curlClient;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _curlClient = [[MPCurlHTTPClient alloc] init];
        _wifiFallbackTimeout = 5.0;
        _cellularTimeout = 30.0;
    }
    return self;
}

- (void)request:(NSString *)url
         method:(NSString *)method
        headers:(NSDictionary<NSString *, NSString *> *)headers
           body:(NSData *)body
       strategy:(MPRequestStrategy)strategy
     completion:(MPCurlCompletion)completion {
    
    switch (strategy) {
        case MPRequestStrategyWiFiOnly:
            [_curlClient performRequest:url method:method headers:headers body:body
                              interface:MPNetworkTypeWiFi timeout:self.cellularTimeout
                             completion:completion];
            break;
            
        case MPRequestStrategyCellularOnly:
            [_curlClient performRequest:url method:method headers:headers body:body
                              interface:MPNetworkTypeCellular timeout:self.cellularTimeout
                             completion:completion];
            break;
            
        case MPRequestStrategyWiFiWithFallback:
            [_curlClient requestWithFallback:url method:method headers:headers body:body
                                wifiTimeout:self.wifiFallbackTimeout completion:completion];
            break;
            
        case MPRequestStrategyFastestWins:
            [self raceRequest:url method:method headers:headers body:body completion:completion];
            break;
            
        case MPRequestStrategyAuto:
            [self autoRequest:url method:method headers:headers body:body completion:completion];
            break;
    }
}

/// 竞速：同时走 WiFi 和蜂窝，取先返回者
- (void)raceRequest:(NSString *)url
             method:(NSString *)method
            headers:(NSDictionary *)headers
               body:(NSData *)body
         completion:(MPCurlCompletion)completion {
    
    __block BOOL completed = NO;
    __block NSLock *lock = [[NSLock alloc] init];
    
    void (^onceCompletion)(MPCurlResponse *, NSError *) = ^(MPCurlResponse *resp, NSError *err) {
        [lock lock];
        if (!completed) {
            completed = YES;
            [lock unlock];
            completion(resp, err);
        } else {
            [lock unlock];
        }
    };
    
    // WiFi 路径
    [_curlClient performRequest:url method:method headers:headers body:body
                      interface:MPNetworkTypeWiFi timeout:self.wifiFallbackTimeout
                     completion:^(MPCurlResponse *resp, NSError *err) {
        if (!err) {
            onceCompletion(resp, nil);
        }
    }];
    
    // 蜂窝路径（延迟 200ms 启动，给 WiFi 先机）
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.2 * NSEC_PER_SEC)),
                   dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [self->_curlClient performRequest:url method:method headers:headers body:body
                          interface:MPNetworkTypeCellular timeout:self.cellularTimeout
                         completion:^(MPCurlResponse *resp, NSError *err) {
            onceCompletion(resp, err);
        }];
    });
    
    // 兜底超时
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(self.cellularTimeout * NSEC_PER_SEC)),
                   dispatch_get_main_queue(), ^{
        NSError *err = [NSError errorWithDomain:@"MPCurl" code:-2 userInfo:@{
            NSLocalizedDescriptionKey: @"Both WiFi and cellular timed out"
        }];
        onceCompletion(nil, err);
    });
}

/// 自动策略：检测网络状态后选择最优路径
- (void)autoRequest:(NSString *)url
             method:(NSString *)method
            headers:(NSDictionary *)headers
               body:(NSData *)body
         completion:(MPCurlCompletion)completion {
    
    MPNetworkInterface *wifi = [MPInterfaceDetector wifiInterface];
    MPNetworkInterface *cell = [MPInterfaceDetector cellularInterface];
    
    if (wifi && cell) {
        // 两个都可用 → WiFi 优先 + 回退
        [_curlClient requestWithFallback:url method:method headers:headers body:body
                            wifiTimeout:self.wifiFallbackTimeout completion:completion];
    } else if (wifi) {
        // 只有 WiFi
        [_curlClient performRequest:url method:method headers:headers body:body
                          interface:MPNetworkTypeWiFi timeout:self.cellularTimeout
                         completion:completion];
    } else if (cell) {
        // 只有蜂窝
        [_curlClient performRequest:url method:method headers:headers body:body
                          interface:MPNetworkTypeCellular timeout:self.cellularTimeout
                         completion:completion];
    } else {
        // 无网络
        NSError *err = [NSError errorWithDomain:@"MPCurl" code:-3 userInfo:@{
            NSLocalizedDescriptionKey: @"No network interface available"
        }];
        completion(nil, err);
    }
}

@end
```

##### 4. Swift 封装层（可选）

```swift
// MultiPathCurl.swift — Swift 友好的封装

import Foundation

enum NetworkInterface {
    case wifi
    case cellular
    case any
    
    var mpType: MPNetworkType {
        switch self {
        case .wifi:     return .wifi
        case .cellular: return .cellular
        case .any:      return .any
        }
    }
}

enum RequestStrategy {
    case wifiOnly
    case cellularOnly
    case wifiWithFallback(timeout: TimeInterval)
    case race
    case auto
}

class MultiPathCurl {
    
    static let shared = MultiPathCurl()
    
    private let client = MPSmartHTTPClient()
    
    /// 获取当前可用网络接口信息
    var availableInterfaces: [String: String] {
        var result: [String: String] = [:]
        if let wifi = MPInterfaceDetector.wifiInterface() {
            result["wifi"] = "\(wifi.name ?? "en0") (\(wifi.ipv4Address ?? ""))"
        }
        if let cell = MPInterfaceDetector.cellularInterface() {
            result["cellular"] = "\(cell.name ?? "pdp_ip0") (\(cell.ipv4Address ?? ""))"
        }
        return result
    }
    
    /// 指定网卡发起 GET
    func get(_ url: String,
             via interface: NetworkInterface = .any,
             completion: @escaping (Data?, Int, Error?) -> Void) {
        
        client.request(url, method: "GET", headers: nil, body: nil,
                       strategy: .wiFiWithFallback) { response, error in
            completion(response?.body, Int(response?.statusCode ?? 0), error)
        }
    }
    
    /// 带策略的请求
    func request(_ url: String,
                 method: String = "GET",
                 headers: [String: String]? = nil,
                 body: Data? = nil,
                 strategy: RequestStrategy = .auto,
                 completion: @escaping (MPCurlResponse?, Error?) -> Void) {
        
        let mpStrategy: MPRequestStrategy
        switch strategy {
        case .wifiOnly:
            mpStrategy = .wiFiOnly
        case .cellularOnly:
            mpStrategy = .cellularOnly
        case .wifiWithFallback(let timeout):
            client.wifiFallbackTimeout = timeout
            mpStrategy = .wiFiWithFallback
        case .race:
            mpStrategy = .fastestWins
        case .auto:
            mpStrategy = .auto
        }
        
        client.request(url, method: method, headers: headers, body: body,
                       strategy: mpStrategy, completion: completion)
    }
}
```

##### 5. 使用示例

```objc
// Objective-C 使用示例

MPCurlHTTPClient *client = [[MPCurlHTTPClient alloc] init];

// 示例 1：强制走蜂窝网络
[client GET:@"https://api.example.com/data"
  interface:MPNetworkTypeCellular
 completion:^(MPCurlResponse *response, NSError *error) {
    if (error) {
        NSLog(@"蜂窝请求失败: %@", error);
        return;
    }
    NSLog(@"状态码: %ld, 使用网卡: %@, 耗时: %.3fs",
          (long)response.statusCode, response.usedInterface, response.totalTime);
}];

// 示例 2：强制走 WiFi
[client GET:@"https://api.example.com/data"
  interface:MPNetworkTypeWiFi
 completion:^(MPCurlResponse *response, NSError *error) {
    // ...
}];

// 示例 3：WiFi 优先，5 秒超时后自动切蜂窝
[client requestWithFallback:@"https://api.example.com/data"
                     method:@"GET"
                    headers:nil
                       body:nil
               wifiTimeout:5.0
                 completion:^(MPCurlResponse *response, NSError *error) {
    NSLog(@"最终使用网卡: %@", response.usedInterface);
}];

// 示例 4：查看当前可用接口
NSArray *interfaces = [MPInterfaceDetector availableInterfaces];
for (MPNetworkInterface *iface in interfaces) {
    NSLog(@"接口: %@, IP: %@, 类型: %ld, 状态: %@",
          iface.name, iface.ipv4Address, (long)iface.type,
          iface.isUp ? @"UP" : @"DOWN");
}
```

```swift
// Swift 使用示例

let curl = MultiPathCurl.shared

// 查看可用接口
print("可用网络: \(curl.availableInterfaces)")

// WiFi 优先 + 3 秒超时回退蜂窝
curl.request("https://api.example.com/data",
             strategy: .wifiWithFallback(timeout: 3)) { response, error in
    guard let resp = response else {
        print("请求失败: \(error?.localizedDescription ?? "")")
        return
    }
    print("HTTP \(resp.statusCode) via \(resp.usedInterface), \(resp.totalTime)s")
}

// 竞速模式
curl.request("https://api.example.com/critical-data",
             strategy: .race) { response, error in
    // 哪个网卡先返回就用哪个
}
```

#### CURLOPT_INTERFACE 在 iOS 上的重要注意事项

##### 已知限制：蜂窝唤醒问题

**POSIX socket 绑定（libcurl 底层使用的方式）不会主动唤醒 iOS 蜂窝无线电硬件**。这意味着：

| 场景 | CURLOPT_INTERFACE 表现 |
|------|----------------------|
| WiFi + 蜂窝同时在线（最常见） | ✅ 正常工作 |
| 只有蜂窝（无 WiFi） | ✅ 正常工作 |
| WiFi 在线，蜂窝休眠 | ⚠️ 可能失败（蜂窝未唤醒） |
| 蜂窝被系统节能关闭 | ❌ 失败 |

##### 解决蜂窝唤醒问题的方案

```objc
// 方案 1（推荐）：先用 NWConnection 唤醒蜂窝，再用 curl 请求

#import <Network/Network.h>

- (void)ensureCellularActiveWithCompletion:(void (^)(BOOL success))completion {
    NWParameters *params = [NWParameters new];
    params.requiredInterfaceType = NWInterfaceTypeCellular;
    params.prohibitExpensivePaths = NO;
    
    nw_connection_t conn = nw_connection_create(
        nw_endpoint_create_host("connectivity-check.ubuntu.com", "80"),
        nw_parameters_create_secure_tcp(
            NW_PARAMETERS_DISABLE_PROTOCOL,
            NW_PARAMETERS_DEFAULT_CONFIGURATION
        )
    );
    
    // 简化：用 Swift 写更清晰（见下方 Swift 版）
}

// Swift 版本
func ensureCellularActive() async -> Bool {
    let params = NWParameters.tcp
    params.requiredInterfaceType = .cellular
    params.prohibitExpensivePaths = false
    
    let connection = NWConnection(
        host: "connectivity-check.ubuntu.com",
        port: 80,
        using: params
    )
    
    return await withCheckedContinuation { continuation in
        connection.stateUpdateHandler = { state in
            switch state {
            case .ready:
                connection.cancel()
                continuation.resume(returning: true)
            case .failed, .cancelled:
                continuation.resume(returning: false)
            default:
                break
            }
        }
        connection.start(queue: .global())
        
        // 超时保护
        DispatchQueue.global().asyncAfter(deadline: .now() + 3) {
            connection.cancel()
        }
    }
}

// 使用：先唤醒蜂窝，再走 curl
func requestViaCellularWithWakeup(url: String) async {
    let woken = await ensureCellularActive()
    if woken {
        MultiPathCurl.shared.request(url, strategy: .cellularOnly) { resp, err in
            // 蜂窝已激活，curl 绑定 pdp_ip0 可正常工作
        }
    }
}
```

```objc
// 方案 2：用 CURLOPT_INTERFACE 绑定 IP 而非接口名
// 有时绑定 IP 比绑定接口名更可靠

NSString *cellularIP = [MPInterfaceDetector ipAddressForType:MPNetworkTypeCellular];
if (cellularIP) {
    NSString *hostSpec = [NSString stringWithFormat:@"host!%@", cellularIP];
    curl_easy_setopt(curl, CURLOPT_INTERFACE, [hostSpec UTF8String]);
}
```

#### libcurl vs NWConnection 对比

| 特性 | libcurl + CURLOPT_INTERFACE | NWConnection + requiredInterfaceType |
|------|---------------------------|-------------------------------------|
| HTTP 协议支持 | ✅ 完整（HTTP/1.1, HTTP/2, HTTP/3） | ❌ 需手动拼装 |
| TLS 支持 | ✅ 内置（Secure Transport / OpenSSL） | ✅ 内置 |
| Cookie 管理 | ✅ 内置 | ❌ 需手动 |
| 重定向跟随 | ✅ 内置 | ❌ 需手动 |
| 代理支持 | ✅ 内置 | ❌ 需手动 |
| 蜂窝唤醒 | ⚠️ 不保证 | ✅ 系统级保证 |
| 跨平台 | ✅ C 库，全平台可用 | ❌ Apple only |
| 包体积 | +1~3MB（静态链接） | 0（系统框架） |
| API 稳定性 | ✅ 非常稳定 | ✅ 稳定 |
| 接口选择粒度 | 接口名或 IP | 接口类型（wifi/cellular） |

#### 优点

- **完整的 HTTP 客户端**：不需要手动拼 HTTP 报文，支持 HTTP/1.1、HTTP/2、HTTP/3
- **成熟稳定**：libcurl 有 25+ 年历史，生产级可靠性
- **精确到接口名**：可以指定 `en0`、`pdp_ip0` 这样的具体接口名
- **跨平台**：同一套 curl 代码可在 iOS、Android (NDK)、Linux、macOS 上运行
- **丰富的调试信息**：`CURLINFO_*` 提供 DNS 时间、连接时间、TLS 握手时间等
- **Cookie 和重定向**：内置支持，无需手写

#### 缺点

- **需要额外集成 libcurl**：增加 ~1-3MB 包体积
- **蜂窝唤醒问题**：POSIX socket bind 不会唤醒 iOS 蜂窝硬件，需配合 NWConnection 预热
- **接口名不是官方 API**：Apple 不保证 `en0`/`pdp_ip0` 永远不变（实际一直稳定）
- **C API**：需要 Objective-C 或 Swift 桥接封装
- **不支持后台传输**：URLSession 的后台传输能力 curl 不具备

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
│   │       └── 否 → 往下看
│   └──
└── 否（不能改服务端）或服务端无法改造
    ├── 需要完整的 HTTP 能力（Cookie、重定向、HTTP/2）？
    │   ├── 是 → 方案 F（libcurl + CURLOPT_INTERFACE）⭐
    │   └── 否（只需简单请求）→ 方案 B（NWConnection）
    ├── 已有 libcurl / 跨平台 C++ 网络层？
    │   └── 是 → 方案 F（libcurl），可复用现有代码
    ├── 纯 Swift/ObjC 项目，不想引入 C 依赖？
    │   └── 是 → 方案 C（NWPathMonitor + URLSession 回退）
    └── 对延迟极度敏感？
        └── 是 → 方案 D（竞速连接）或方案 F 的竞速模式
```

### 推荐方案

#### 首选：方案 F（libcurl + CURLOPT_INTERFACE）

如果项目可以接入 libcurl（约 1-3MB 包体积增加），**方案 F 是最完整的选择**：

1. **完整 HTTP 客户端**：不需要手动拼 HTTP 报文，Cookie、重定向、HTTP/2 全部内置
2. **精确到接口名**：`CURLOPT_INTERFACE` 直接指定 `en0`（WiFi）或 `pdp_ip0`（蜂窝）
3. **不需要服务端改造**
4. **跨平台**：同一套代码可移植到 Android NDK
5. **丰富的性能指标**：DNS 时间、TCP 连接时间、TLS 握手时间等

> **注意**：需配合 `NWConnection` 解决蜂窝唤醒问题（详见方案 F 代码）

#### 备选：方案 C（NWPathMonitor + URLSession 回退）

如果不想引入 libcurl 依赖，**方案 C** 是纯 Apple API 的最佳选择：

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
