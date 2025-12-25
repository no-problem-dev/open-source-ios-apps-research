# Mullvad VPN

**スター**: 6,537 ⭐
**リポジトリ**: https://github.com/mullvad/mullvadvpn-app
**ライセンス**: GPL-3.0

---

## 概要

Mullvad VPNは、プライバシー重視のVPNサービスの公式クライアント。iOS、macOS、Windows、Linux、Androidに対応。WireGuardプロトコルをサポートし、匿名性を重視した設計。

---

## 主要機能

| 機能 | 説明 |
|------|------|
| WireGuard | 高速VPNプロトコル |
| Kill Switch | 接続断時の保護 |
| Split Tunneling | アプリ別ルーティング |
| マルチホップ | 多段接続 |
| 匿名アカウント | 番号のみのアカウント |
| DNS設定 | カスタムDNS |

---

## 技術スタック

| 項目 | 技術 |
|------|------|
| 言語 | Swift, Rust |
| UI | UIKit |
| VPNプロトコル | WireGuard |
| ネットワーク | NetworkExtension |
| 暗号化 | libsodium |
| 共有コード | Rust FFI |

---

## アーキテクチャ

```
mullvadvpn-app/
├── ios/                     # iOSアプリ
│   ├── MullvadVPN/          # メインアプリ
│   ├── PacketTunnel/        # VPN拡張
│   └── MullvadREST/         # API クライアント
├── mullvad-daemon/          # Rust デーモン
├── talpid-core/             # VPN コア（Rust）
└── wireguard/               # WireGuard実装
```

---

## 学習価値

### VPN実装の教科書

| 分野 | 学習ポイント |
|------|-------------|
| **NetworkExtension** | VPNトンネル実装 |
| **WireGuard** | モダンVPNプロトコル |
| **Rust FFI** | Swift-Rust連携 |
| **セキュリティ** | 暗号化・認証 |
| **マルチプラットフォーム** | コード共有戦略 |

### コードパターン

**VPNトンネル管理**:
```swift
// Packet Tunnel Provider
class PacketTunnelProvider: NEPacketTunnelProvider {
    private var wireguardAdapter: WireGuardAdapter?

    override func startTunnel(
        options: [String: NSObject]?,
        completionHandler: @escaping (Error?) -> Void
    ) {
        let config = parseConfiguration(options)

        wireguardAdapter = WireGuardAdapter(with: self) { status in
            switch status {
            case .connected:
                completionHandler(nil)
            case .failed(let error):
                completionHandler(error)
            }
        }

        wireguardAdapter?.start(tunnelConfiguration: config)
    }

    override func stopTunnel(
        with reason: NEProviderStopReason,
        completionHandler: @escaping () -> Void
    ) {
        wireguardAdapter?.stop { completionHandler() }
    }
}
```

**Rust FFI連携**:
```swift
// Swift から Rust 関数呼び出し
class MullvadCore {
    func connect(to server: Server) async throws {
        try await withCheckedThrowingContinuation { continuation in
            mullvad_connect(
                server.address.cString,
                server.port,
                { result in
                    if result.success {
                        continuation.resume()
                    } else {
                        continuation.resume(throwing: VPNError.connectionFailed)
                    }
                }
            )
        }
    }
}
```

**VPN状態管理**:
```swift
// VPN状態の監視
class VPNStateManager: ObservableObject {
    @Published var state: VPNState = .disconnected

    private var observer: NSObjectProtocol?

    init() {
        observer = NotificationCenter.default.addObserver(
            forName: .NEVPNStatusDidChange,
            object: nil,
            queue: .main
        ) { [weak self] notification in
            guard let connection = notification.object as? NEVPNConnection else { return }
            self?.updateState(from: connection.status)
        }
    }

    private func updateState(from status: NEVPNStatus) {
        switch status {
        case .connected: state = .connected
        case .connecting: state = .connecting
        case .disconnected: state = .disconnected
        case .disconnecting: state = .disconnecting
        default: state = .unknown
        }
    }
}
```

---

## プロジェクト評価

| 観点 | 評価 | 理由 |
|------|------|------|
| **コード品質** | ★★★★★ | セキュリティ重視 |
| **ドキュメント** | ★★★★☆ | アーキテクチャ文書 |
| **アクティビティ** | ★★★★★ | 継続的な開発 |
| **学習価値** | ★★★★★ | VPN実装の模範 |
| **実用性** | ★★★★★ | 商用VPNサービス |

---

## NetworkExtension

### VPN構成

```swift
// VPN設定
func configureVPN() async throws {
    let manager = NETunnelProviderManager()

    let proto = NETunnelProviderProtocol()
    proto.providerBundleIdentifier = "net.mullvad.vpn.PacketTunnel"
    proto.serverAddress = "mullvad.net"
    proto.providerConfiguration = [
        "privateKey": privateKey,
        "serverPublicKey": serverPublicKey,
        "endpoint": endpoint
    ]

    manager.protocolConfiguration = proto
    manager.localizedDescription = "Mullvad VPN"
    manager.isEnabled = true

    try await manager.saveToPreferences()
    try await manager.loadFromPreferences()
}
```

---

## WireGuard統合

```swift
// WireGuard設定
struct WireGuardConfiguration {
    let privateKey: PrivateKey
    let addresses: [IPAddressRange]
    let dns: [DNSServer]
    let peers: [Peer]

    struct Peer {
        let publicKey: PublicKey
        let endpoint: Endpoint
        let allowedIPs: [IPAddressRange]
    }
}
```

---

## 開発者向け情報

### ビルド手順

```bash
# Rust ツールチェーンが必要
git clone https://github.com/mullvad/mullvadvpn-app
cd mullvadvpn-app/ios
./build.sh
open MullvadVPN.xcodeproj
```

### 必要要件

- Xcode 15+
- Rust 1.70+
- iOS 15+

---

## まとめ

Mullvad VPNは、プロダクション品質のVPN実装を学ぶ最良の教材。NetworkExtension、WireGuard、Rust FFI連携など、高度なネットワーキング技術を網羅。セキュリティ重視の設計は、プライバシーアプリ開発の参考になる。

**推奨対象**:
- VPN/ネットワーク拡張を学びたい開発者
- Swift-Rust連携を実装したい人
- セキュリティ重視のアプリ設計を参考にしたい人
- WireGuardプロトコルを理解したい人
