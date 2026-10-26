# qTunnel iOS Client App 功能规格文档

## 1. 项目概述

### 1.1 产品描述
qTunnel-iOS 是一个 iOS 客户端应用，作为加密代理通道，将设备上的流量通过 qTunnel 协议转发到远程服务器。

### 1.2 核心功能
- **流量加密转发**: 捕获设备流量，通过 qTunnel 协议加密后转发到服务端
- **协议支持**: 兼容 qTunnel 的 RC4 和 AES256-CFB 加密
- **本地代理**: 作为 SOCKS5/HTTP 代理服务器，接收其他 app 的流量

### 1.3 使用场景
- 保护公共 WiFi 下的网络通信安全
- 企业内网远程访问
- 隐私保护，加密网络流量

---

## 2. 系统架构

### 2.1 整体架构

```
+------------------+      +------------------+      +------------------+
|   Target App     | ---> | qTunnel-iOS      | ---> |   qTunnel Server |
| (Browser, etc.)  |      | (Local Proxy)    |      |   (Remote)       |
+------------------+      +------------------+      +------------------+
                                  |
                                  v
                         +------------------+
                         |   qTunnel Protocol|
                         |   (RC4/AES256)    |
                         +------------------+
```

### 2.2 模块划分

```
qTunnel-iOS/
├── Core/
│   ├── TunnelClient       # qTunnel 协议客户端
│   ├── Cipher             # 加密模块 (RC4/AES256-CFB)
│   ├── ProxyServer        # 本地 SOCKS5/HTTP 代理
│   └── ConnectionManager  # 连接池管理
├── Network/
│   ├── PacketCapture      # 流量捕获 (Packet Tunnel Provider)
│   ├── DNSServer          # DNS 代理
│   └── NetworkExtension   # iOS Network Extension
├── UI/
│   ├── MainViewController # 主界面
│   ├── SettingsViewController # 设置界面
│   ├── ServerListViewController # 服务器管理
│   └── Components/        # 通用组件
├── Storage/
│   ├── ServerConfig       # 服务器配置存储
│   ├── KeychainManager    # 密钥链管理 (密码存储)
│   └── PreferencesManager # 用户偏好设置
└── Utils/
    ├── Logger             # 日志系统
    ├── Extensions         # 扩展方法
    └── Validation         # 输入验证
```

---

## 3. 功能详细说明

### 3.1 核心功能模块

#### 3.1.1 qTunnel 协议客户端 (TunnelClient)

**功能**: 实现 qTunnel 协议的客户端功能

**接口定义**:
```swift
class TunnelClient {
    // 初始化
    init(serverAddress: String, port: Int, secret: String, cryptoMethod: CryptoMethod)

    // 连接管理
    func connect() throws
    func disconnect()
    var isConnected: Bool

    // 数据传输
    func send(_ data: Data) throws -> Data
    func sendSync(_ data: Data) async throws -> Data

    // 状态回调
    var onConnect: (() -> Void)?
    var onDisconnect: ((Error?) -> Void)?
    var onError: ((Error) -> Void)?
}

enum CryptoMethod {
    case rc4
    case aes256cfb
}
```

**加密实现**:
```swift
class QTCipher {
    init(secret: String, method: CryptoMethod)

    //解密
    func encrypt(_ data: 加密/ Data) -> Data
    func decrypt(_ data: Data) -> Data

    // 密钥派生 (MD5迭代)
    private func secretToKey(_ secret: [UInt8], size: Int) -> [UInt8]
}
```

#### 3.1.2 本地代理服务器 (ProxyServer)

**功能**: 在本地监听端口，接收其他 app 的流量

**支持协议**:
- SOCKS5 代理
- HTTP 代理 (可选)

**接口定义**:
```swift
class ProxyServer {
    // 启动/停止
    func start(port: Int) throws
    func stop()

    // 配置
    var proxyType: ProxyType
    var listeningPort: Int { get }

    // 回调
    var onRequest: ((ProxyRequest) -> Void)?
    var onConnection: ((String, Int) -> Void)?
    var onError: ((Error) -> Void)?
}

enum ProxyType {
    case socks5
    case http
}
```

**SOCKS5 认证流程**:
```
1. 客户端连接 → Greeting
2. 服务器 → Auth挑   3. 客户端 → Auth Response
4. 服务器 → Auth Success
5. 客户端 → Connection Request (目标地址:端口)
6. 服务器 → Connection Reply
7. 开始数据传输 (通过 TunnelClient 加密转发)
```

#### 3.1.3 流量捕获 (PacketTunnelProvider)

**功能**: 使用 iOS Network Extension 捕获设备全局流量

**配置**:
```swift
class PacketTunnelProvider: NEPacketTunnelProvider {
    override func startTunnel(options: [String : NSObject]?, completionHandler: @escaping (Error?) -> Void) {
        // 1. 配置 TUN 接口
        // 2. 启动本地代理服务器
        // 3. 连接到 qTunnel 服务器
        // 4. 开始流量转发
    }

    override func stopTunnel(with reason: NEProviderStopReason, completionHandler: @escaping () -> Void) {
        // 清理连接
    }
}
```

### 3.2 用户界面

#### 3.2.1 主界面 (MainViewController)

**功能**: 显示连接状态、快捷操作

**布局**:
```
+------------------+
| [状态指示器]     |
|   已连接 / 未连接 |
+------------------+
|                  |
|  [  开启隧道  ]  |
|                  |
+------------------+
| 服务器: example.com:9001 |
| 方法: RC4          |
+------------------+
| [流量统计]        |
| 上传: 1.2MB       |
| 下载: 5.8MB       |
+------------------+
| [  设置  ]        |
+------------------+
```

**功能**:
- 显示当前连接状态
- 一键连接/断开
- 显示服务器信息
- 显示流量统计
- 入口跳转到设置

#### 3.2.2 设置界面 (SettingsViewController)

**功能**: 配置服务器、加密参数、偏好设置

**配置项**:
```
服务器配置:
├── 地址: [文本输入]
├── 端口: [数字输入]
├── Secret: [密码输入]
└── 加密方法: [RC4 / AES256-CFB]

高级选项:
├── 自动重连: [开关]
├── 连接超时: [数字输入] 秒
├── 心跳间隔: [数字输入] 秒
└── 日志级别: [Debug / Info / Warning / Error]

账户:
├── 移除服务器配置
└── 导出配置
```

#### 3.2.3 服务器管理 (ServerListViewController)

**功能**: 添加、编辑、删除服务器配置

**列表项**:
```
[服务器名称] [地址:端口] [加密方法]
[编辑] [删除] [连接]
```

**操作**:
- 添加新服务器
- 编辑现有服务器
- 删除服务器
- 快速切换连接

### 3.3 数据存储

#### 3.3.1 服务器配置 (ServerConfig)

**数据结构**:
```swift
struct ServerConfig: Codable {
    var id: String
    var name: String
    var address: String
    var port: Int
    var secret: String  // 存储到 Keychain
    var cryptoMethod: CryptoMethod
    var isEnabled: Bool
    var lastConnected: Date?
    var createdAt: Date
    var updatedAt: Date
}
```

**存储位置**:
- 配置元数据: `UserDefaults` 或 `plist` 文件
- Secret 密码: `Keychain`

#### 3.3.2 Keychain 管理

**存储内容**:
- 服务器 secret
- 可能的证书/认证信息

**安全措施**:
- 使用 iOS Keychain Services
- 设置 `kSecAttrAccessibleWhenUnlockedThisDeviceOnly`
- 敏感数据加密存储

---

## 4. 非功能需求

### 4.1 性能需求
- 连接建立时间 < 3 秒
- 加密吞吐量 > 10 MB/s
- 内存占用 < 50 MB
- 电池消耗 < 5%/小时 (持续运行)

### 4.2 兼容性需求
- iOS 15.0+
- 支持 iPhone 和 iPad
- 支持 IPv4 和 IPv6

### 4.3 安全需求
- Secret 存储在 Keychain，不存储在明文
- 支持后台运行 (Network Extension)
- TLS 证书验证 (如有 HTTPS 后端)

### 4.4 隐私需求
- 不收集用户流量数据
- 不记录访问记录
- 本地日志可手动清除

---

## 5. 技术选型

### 5.1 依赖库

| 库名 | 用途 | 许可证 |
|------|------|--------|
| CocoaAsyncSocket | TCP Socket 通信 | MIT |
| Cryptor | 通用加密接口 | Apache 2.0 |
| KeychainAccess | Keychain 封装 | MIT |
| SnapKit | Auto Layout | MIT |

### 5.2 iOS 系统框架

| 框架 | 用途 |
|------|------|
| NetworkExtension | VPN/代理功能 |
| Security | Keychain 访问 |
| SystemConfiguration | 网络状态检测 |
| Foundation | 基础功能 |

---

## 6. 项目结构

```
qTunnel-iOS/
├── project.yml                 # XcodeGen 配置
├── Makefile                    # 构建脚本
├── Podfile                     # CocoaPods 依赖
├── setup.sh                    # 环境初始化脚本
├── qTunnel-iOS/
│   ├── App/
│   │   ├── AppDelegate.swift
│   │   ├── SceneDelegate.swift
│   │   └── Info.plist
│   ├── Core/
│   │   ├── TunnelClient.swift
│   │   ├── Cipher/
│   │   │   ├── Cipher.swift
│   │   │   ├── RC4Cipher.swift
│   │   │   └── AES256Cipher.swift
│   │   ├── ProxyServer/
│   │   │   ├── ProxyServer.swift
│   │   │   ├── SOCKS5Server.swift
│   │   │   └── HTTPServer.swift
│   │   └── ConnectionManager.swift
│   ├── Network/
│   │   ├── PacketTunnelProvider.swift
│   │   ├── DNSServer.swift
│   │   └── NetworkExtension.swift
│   ├── UI/
│   │   ├── Main/
│   │   │   ├── MainViewController.swift
│   │   │   ├── MainViewModel.swift
│   │   │   └── MainView.swift
│   │   ├── Settings/
│   │   │   ├── SettingsViewController.swift
│   │   │   ├── ServerListViewController.swift
│   │   │   └── ServerEditViewController.swift
│   │   └── Components/
│   │       ├── StatusIndicator.swift
│   │       ├── TrafficChart.swift
│   │       └── GradientButton.swift
│   ├── Storage/
│   │   ├── ServerConfigStorage.swift
│   │   ├── KeychainManager.swift
│   │   └── PreferencesManager.swift
│   ├── Utils/
│   │   ├── Logger.swift
│   │   ├── Extensions/
│   │   │   ├── Data+Extension.swift
│   │   │   └── String+Extension.swift
│   │   └── Validation.swift
│   └── Resources/
│       ├── Assets.xcassets
│       └── LaunchScreen.storyboard
├── qTunnel-iOS Extension/
│   ├── PacketTunnelProvider.swift
│   ├── Info.plist
│   └── Resources/
└── qTunnel-iOSTests/
    ├── CipherTests.swift
    ├── ProxyTests.swift
    └── TunnelClientTests.swift
```

---

## 7. 开发里程碑

### Phase 1: 基础框架
- [ ] 创建项目结构
- [ ] 配置 XcodeGen
- [ ] 实现 Cipher 加密模块 (RC4/AES256)
- [ ] 实现 TunnelClient

### Phase 2: 代理服务器
- [ ] 实现 SOCKS5 代理服务器
- [ ] 实现本地流量转发
- [ ] 集成 TunnelClient 与 ProxyServer

### Phase 3: UI 开发
- [ ] 主界面 (连接状态、控制按钮)
- [ ] 设置界面 (服务器配置)
- [ ] 服务器列表管理

### Phase 4: Network Extension
- [ ] 实现 PacketTunnelProvider
- [ ] 配置 Network Extension 权限
- [ ] 测试全局流量捕获

### Phase 5: 测试与优化
- [ ] 单元测试
- [ ] 性能测试
- [ ] 稳定性优化
- [ ] App Store 提交准备

---

## 8. 附录

### 8.1 qTunnel 协议参考

详见 `PROTOCOL.md`

### 8.2 相关文档
- Apple Network Extension 文档
- iOS Keychain Services 文档
- SOCKS5 协议规范 (RFC 1928)

### 8.3 参考项目
- Outline VPN (Network Extension 实现)
- Surge (iOS 代理工具)
