# Passepartout

**スター**: 3,500+ ⭐
**リポジトリ**: https://github.com/passepartoutvpn/passepartout-apple
**ライセンス**: GPL-3.0

---

## 概要

Passepartoutは、iOS/macOS向けのOpenVPN/WireGuardクライアント。ユーザーフレンドリーなインターフェースで、複数のVPNプロトコルと設定プロファイルを管理可能。プライバシー重視の設計。

---

## 主要機能

| 機能 | 説明 |
|------|------|
| OpenVPN | 完全対応 |
| WireGuard | ネイティブ対応 |
| プロファイル | 複数設定管理 |
| プロバイダー | 主要VPNサービス対応 |
| Siri | ショートカット対応 |
| iCloud同期 | 設定同期 |

---

## 技術スタック

| 項目 | 技術 |
|------|------|
| 言語 | Swift |
| UI | SwiftUI |
| VPN | NetworkExtension |
| OpenVPN | TunnelKit |
| WireGuard | WireGuardKit |
| 同期 | Core Data + iCloud |

---

## アーキテクチャ

```
passepartout-apple/
├── Passepartout/            # メインアプリ
│   ├── App/                 # アプリ起動
│   ├── Views/               # SwiftUI ビュー
│   └── ViewModels/          # ビューモデル
├── PassepartoutLibrary/     # 共有ライブラリ
│   ├── Core/                # コアロジック
│   ├── Providers/           # VPNプロバイダー
│   └── Services/            # サービス層
├── TunnelKit/               # OpenVPN実装
└── Tunnels/                 # トンネル拡張
    ├── OpenVPNTunnel/
    └── WireGuardTunnel/
```

---

## 学習価値

### VPNクライアント設計

| 分野 | 学習ポイント |
|------|-------------|
| **マルチプロトコル** | OpenVPN + WireGuard |
| **SwiftUI** | 設定UI実装 |
| **Core Data** | プロファイル管理 |
| **iCloud同期** | 設定同期 |
| **Siri統合** | ショートカット |

### コードパターン

**VPNマネージャー**:
```swift
// マルチプロトコルVPN管理
class VPNManager: ObservableObject {
    @Published var status: VPNStatus = .disconnected
    @Published var activeProfile: Profile?

    private var tunnelManager: NETunnelProviderManager?

    func connect(profile: Profile) async throws {
        // プロファイルに応じたトンネル設定
        let protocol = buildProtocol(for: profile)

        tunnelManager = NETunnelProviderManager()
        tunnelManager?.protocolConfiguration = protocol
        tunnelManager?.localizedDescription = profile.name
        tunnelManager?.isEnabled = true

        try await tunnelManager?.saveToPreferences()
        try await tunnelManager?.loadFromPreferences()

        try tunnelManager?.connection.startVPNTunnel()
        activeProfile = profile
    }

    private func buildProtocol(for profile: Profile) -> NETunnelProviderProtocol {
        let proto = NETunnelProviderProtocol()

        switch profile.vpnType {
        case .openVPN:
            proto.providerBundleIdentifier = "com.passepartout.OpenVPNTunnel"
            proto.providerConfiguration = profile.openVPNConfiguration
        case .wireGuard:
            proto.providerBundleIdentifier = "com.passepartout.WireGuardTunnel"
            proto.providerConfiguration = profile.wireGuardConfiguration
        }

        return proto
    }
}
```

**プロファイル管理**:
```swift
// Core Data プロファイル
@objc(Profile)
class Profile: NSManagedObject {
    @NSManaged var id: UUID
    @NSManaged var name: String
    @NSManaged var vpnType: VPNType
    @NSManaged var serverAddress: String
    @NSManaged var configurationData: Data

    var openVPNConfiguration: [String: Any]? {
        guard vpnType == .openVPN else { return nil }
        return try? JSONSerialization.jsonObject(with: configurationData) as? [String: Any]
    }

    var wireGuardConfiguration: [String: Any]? {
        guard vpnType == .wireGuard else { return nil }
        return try? JSONSerialization.jsonObject(with: configurationData) as? [String: Any]
    }
}

class ProfileManager: ObservableObject {
    @Published var profiles: [Profile] = []

    private let container: NSPersistentCloudKitContainer

    init() {
        container = NSPersistentCloudKitContainer(name: "Passepartout")
        container.loadPersistentStores { _, error in
            if let error = error {
                fatalError("Core Data load failed: \(error)")
            }
        }
        container.viewContext.automaticallyMergesChangesFromParent = true
        fetchProfiles()
    }

    func importProfile(from url: URL) throws {
        let data = try Data(contentsOf: url)
        let profile = Profile(context: container.viewContext)
        profile.id = UUID()
        profile.name = url.deletingPathExtension().lastPathComponent
        profile.configurationData = data
        profile.vpnType = detectType(from: data)

        try container.viewContext.save()
        fetchProfiles()
    }
}
```

**Siri ショートカット**:
```swift
// Siri統合
class IntentHandler: INExtension {
    override func handler(for intent: INIntent) -> Any {
        switch intent {
        case is ConnectVPNIntent:
            return ConnectVPNIntentHandler()
        case is DisconnectVPNIntent:
            return DisconnectVPNIntentHandler()
        default:
            return self
        }
    }
}

class ConnectVPNIntentHandler: NSObject, ConnectVPNIntentHandling {
    func handle(intent: ConnectVPNIntent) async -> ConnectVPNIntentResponse {
        guard let profileName = intent.profileName else {
            return ConnectVPNIntentResponse(code: .failure, userActivity: nil)
        }

        do {
            let profile = try ProfileManager.shared.profile(named: profileName)
            try await VPNManager.shared.connect(profile: profile)
            return ConnectVPNIntentResponse(code: .success, userActivity: nil)
        } catch {
            return ConnectVPNIntentResponse(code: .failure, userActivity: nil)
        }
    }
}
```

---

## プロジェクト評価

| 観点 | 評価 | 理由 |
|------|------|------|
| **コード品質** | ★★★★★ | モダンSwiftUI設計 |
| **ドキュメント** | ★★★★☆ | ユーザーガイド充実 |
| **アクティビティ** | ★★★★★ | 継続的な開発 |
| **学習価値** | ★★★★★ | VPNクライアントの模範 |
| **実用性** | ★★★★★ | App Store公開中 |

---

## TunnelKit統合

### OpenVPN実装

```swift
// TunnelKit ベースのOpenVPN
class OpenVPNTunnelProvider: NEPacketTunnelProvider {
    private var vpnAdapter: OpenVPNAdapter?

    override func startTunnel(
        options: [String: NSObject]?,
        completionHandler: @escaping (Error?) -> Void
    ) {
        let config = parseConfiguration(from: protocolConfiguration)

        vpnAdapter = OpenVPNAdapter()
        vpnAdapter?.delegate = self

        do {
            try vpnAdapter?.apply(configuration: config)
            vpnAdapter?.connect(using: packetFlow)
            completionHandler(nil)
        } catch {
            completionHandler(error)
        }
    }
}
```

---

## iCloud同期

```swift
// CloudKit 同期設定
class CloudSyncManager {
    private let container: NSPersistentCloudKitContainer

    func enableSync() {
        let storeDescription = container.persistentStoreDescriptions.first
        storeDescription?.cloudKitContainerOptions = NSPersistentCloudKitContainerOptions(
            containerIdentifier: "iCloud.com.passepartout"
        )
    }

    func handleRemoteChange(_ notification: Notification) {
        container.viewContext.perform {
            self.container.viewContext.refreshAllObjects()
        }
    }
}
```

---

## 開発者向け情報

### ビルド手順

```bash
git clone https://github.com/passepartoutvpn/passepartout-apple
cd passepartout-apple
git submodule update --init --recursive
open Passepartout.xcworkspace
```

### 必要要件

- Xcode 15+
- TunnelKit
- WireGuardKit

---

## まとめ

Passepartoutは、マルチプロトコルVPNクライアントの優れた実装例。SwiftUI、Core Data + iCloud、Siri統合など、モダンなiOS開発パターンを網羅。VPNアプリ開発の参考に最適。

**推奨対象**:
- VPNクライアントを開発したい人
- SwiftUI + Core Data設計を学びたい開発者
- iCloud同期を実装したい人
- Siriショートカット統合を学びたい人
