# SimpleX Chat

**スター**: 10,062 ⭐
**リポジトリ**: https://github.com/simplex-chat/simplex-chat
**ライセンス**: AGPL-3.0

---

## 概要

SimpleX Chatは、ユーザーIDを持たない革新的なプライバシー重視メッセージングアプリ。電話番号、ユーザー名、その他の識別子なしで動作し、メタデータ保護において最高レベルのプライバシーを提供。

---

## 主要機能

| 機能 | 説明 |
|------|------|
| IDなし | ユーザー識別子不要 |
| E2E暗号化 | 完全な暗号化 |
| 分散型 | P2P的な設計 |
| グループチャット | 暗号化グループ |
| 音声メッセージ | 暗号化音声 |
| ファイル共有 | セキュアな転送 |

---

## 技術スタック

| 項目 | 技術 |
|------|------|
| 言語 | Swift, Haskell (バックエンド) |
| UI | SwiftUI |
| プロトコル | SimpleX Messaging Protocol |
| 暗号化 | NaCl, Double Ratchet |
| ネットワーク | Tor対応 |

---

## 革新的設計

### ユーザーIDなしの仕組み

```
従来のメッセージング:
User A (ID: alice123) → Server → User B (ID: bob456)

SimpleX:
User A → [一時的接続ID] → Relay → [別の一時的ID] → User B
         ↑ 各会話で異なるID
```

### SimpleX Messaging Protocol

| 特徴 | 説明 |
|------|------|
| ペアワイズID | 各会話で異なる識別子 |
| リレーサーバー | メッセージ中継のみ |
| メタデータ保護 | 誰と通信しているか不明 |
| 分散化 | 自己ホスト可能 |

---

## 学習価値

### プライバシー設計

| 分野 | 学習ポイント |
|------|-------------|
| **プロトコル設計** | 独自プロトコル実装 |
| **SwiftUI** | モダンUI設計 |
| **暗号化** | 高度な暗号技術 |
| **分散システム** | P2P的アーキテクチャ |
| **Tor統合** | 匿名ネットワーク |

### コードパターン

**接続確立**:
```swift
// QRコードで接続
struct ConnectionView: View {
    @State private var invitationLink: String = ""

    var body: some View {
        VStack {
            // QRコード生成
            QRCodeView(content: generateInvitation())

            // または招待リンク共有
            ShareLink(item: invitationLink)
        }
    }

    func generateInvitation() -> String {
        // 一時的な接続IDを生成
        return SimpleXAPI.createInvitation()
    }
}
```

**メッセージ送信**:
```swift
class ChatController: ObservableObject {
    @Published var messages: [ChatMessage] = []

    func sendMessage(_ text: String, to contact: Contact) async throws {
        // E2E暗号化してリレー経由で送信
        let encrypted = try encrypt(text, for: contact)
        try await SimpleXAPI.send(encrypted, via: contact.relayServer)
    }
}
```

---

## プロジェクト評価

| 観点 | 評価 | 理由 |
|------|------|------|
| **コード品質** | ★★★★★ | 革新的な設計 |
| **ドキュメント** | ★★★★★ | プロトコル文書完備 |
| **アクティビティ** | ★★★★★ | 活発な開発 |
| **学習価値** | ★★★★★ | プライバシー設計の先端 |
| **実用性** | ★★★★☆ | ユーザー基盤拡大中 |

---

## SwiftUI実装

### チャットUI

```swift
struct ChatView: View {
    @StateObject var chat: ChatController
    @State private var messageText = ""

    var body: some View {
        VStack {
            ScrollView {
                LazyVStack {
                    ForEach(chat.messages) { message in
                        MessageBubble(message: message)
                    }
                }
            }

            HStack {
                TextField("メッセージ", text: $messageText)
                Button("送信") {
                    Task {
                        try await chat.sendMessage(messageText)
                        messageText = ""
                    }
                }
            }
        }
    }
}
```

---

## 開発者向け情報

### セルフホスト

SimpleXリレーサーバーは自己ホスト可能：

```bash
# サーバー起動
docker run -d simplex/smp-server
```

### iOSアプリビルド

```bash
# Xcodeでビルド
open apps/ios/SimpleX.xcodeproj
```

---

## まとめ

SimpleX Chatは、プライバシー保護の新しいパラダイムを示すプロジェクト。ユーザーIDなしでの通信という革新的なアプローチは、次世代のプライバシー重視アプリの参考になる。

**推奨対象**:
- プライバシー技術研究者
- 分散システム開発者
- SwiftUI + セキュリティ学習者

