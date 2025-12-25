# Bark

**スター**: 7,209 ⭐
**リポジトリ**: https://github.com/Finb/Bark
**ライセンス**: MIT

---

## 概要

Barkは、カスタムプッシュ通知をiOSデバイスに送信できるアプリ。シンプルなHTTPリクエストで通知を送信でき、サーバー監視、CI/CD通知、IoTアラートなど様々な用途に活用可能。

---

## 主要機能

| 機能 | 説明 |
|------|------|
| HTTP通知 | URLでプッシュ送信 |
| グループ化 | 通知のカテゴリ分類 |
| アーカイブ | 通知履歴保存 |
| カスタムサウンド | 通知音のカスタマイズ |
| 暗号化 | E2E暗号化オプション |
| セルフホスト | 自前サーバー運用 |

---

## 技術スタック

| 項目 | 技術 |
|------|------|
| 言語 | Swift |
| UI | SwiftUI |
| 通知 | APNs |
| サーバー | Go (bark-server) |
| ストレージ | Core Data |
| 暗号化 | CryptoKit |

---

## アーキテクチャ

```
Bark/
├── Bark/                    # iOSアプリ
│   ├── Views/               # SwiftUI ビュー
│   ├── Models/              # データモデル
│   ├── Services/            # 通知サービス
│   └── Extensions/          # 拡張機能
├── NotificationServiceExtension/  # 通知拡張
└── BarkServer/              # Go サーバー（別リポ）
```

---

## 学習価値

### プッシュ通知システム

| 分野 | 学習ポイント |
|------|-------------|
| **APNs** | Apple Push Notification |
| **Notification Extension** | 通知のカスタマイズ |
| **Background Processing** | バックグラウンド処理 |
| **暗号化通知** | セキュア通知 |
| **ディープリンク** | URL Scheme |

### コードパターン

**通知受信**:
```swift
// Notification Service Extension
class NotificationService: UNNotificationServiceExtension {
    override func didReceive(
        _ request: UNNotificationRequest,
        withContentHandler contentHandler: @escaping (UNNotificationContent) -> Void
    ) {
        guard let mutableContent = request.content.mutableCopy() as? UNMutableNotificationContent else {
            contentHandler(request.content)
            return
        }

        // 暗号化されたコンテンツの復号化
        if let encryptedBody = mutableContent.userInfo["encrypted"] as? String {
            mutableContent.body = decrypt(encryptedBody)
        }

        contentHandler(mutableContent)
    }
}
```

**通知送信API**:
```swift
// シンプルな通知送信
struct BarkAPI {
    let baseURL: URL
    let deviceKey: String

    func push(title: String, body: String, group: String? = nil) async throws {
        var components = URLComponents(url: baseURL, resolvingAgainstBaseURL: true)!
        components.path = "/\(deviceKey)/\(title)/\(body)"

        if let group = group {
            components.queryItems = [URLQueryItem(name: "group", value: group)]
        }

        let (_, response) = try await URLSession.shared.data(from: components.url!)
        guard (response as? HTTPURLResponse)?.statusCode == 200 else {
            throw BarkError.pushFailed
        }
    }
}
```

**通知履歴管理**:
```swift
// Core Data で通知アーカイブ
class NotificationStore: ObservableObject {
    @Published var notifications: [SavedNotification] = []

    private let container: NSPersistentContainer

    func save(notification: UNNotification) {
        let context = container.viewContext
        let saved = SavedNotification(context: context)
        saved.title = notification.request.content.title
        saved.body = notification.request.content.body
        saved.date = Date()
        saved.group = notification.request.content.threadIdentifier

        try? context.save()
        fetchNotifications()
    }
}
```

---

## プロジェクト評価

| 観点 | 評価 | 理由 |
|------|------|------|
| **コード品質** | ★★★★☆ | シンプルで明確 |
| **ドキュメント** | ★★★★★ | API仕様が明確 |
| **アクティビティ** | ★★★★☆ | 継続的なメンテナンス |
| **学習価値** | ★★★★★ | 通知システムの教科書 |
| **実用性** | ★★★★★ | 実用的なツール |

---

## 通知カスタマイズ

### パラメータ

| パラメータ | 説明 |
|-----------|------|
| title | 通知タイトル |
| body | 通知本文 |
| group | グループ化キー |
| sound | カスタムサウンド |
| icon | カスタムアイコン |
| url | タップ時の遷移先 |
| isArchive | アーカイブ有無 |

### 使用例

```bash
# 基本的な通知
curl https://api.day.app/YOUR_KEY/タイトル/本文

# グループ化された通知
curl "https://api.day.app/YOUR_KEY/タイトル/本文?group=サーバー監視"

# カスタムサウンド
curl "https://api.day.app/YOUR_KEY/タイトル/本文?sound=alarm"
```

---

## 開発者向け情報

### ビルド手順

```bash
git clone https://github.com/Finb/Bark
cd Bark
open Bark.xcodeproj
```

### セルフホスト

```bash
# bark-server をDockerで起動
docker run -dt --name bark -p 8080:8080 finab/bark-server
```

---

## まとめ

Barkは、iOSプッシュ通知システムを学ぶ最良の教材。APNs統合、Notification Extension、暗号化通知など、通知関連の技術を網羅。DevOps、IoT、監視システムとの連携に最適。

**推奨対象**:
- プッシュ通知システムを学びたい開発者
- サーバー監視・CI/CD通知を実装したい人
- Notification Extensionを学びたい人
- セルフホスト可能な通知基盤を求める人
