# Element iOS (Matrix)

**スター**: 1,900+ ⭐
**リポジトリ**: https://github.com/vector-im/element-ios
**ライセンス**: Apache-2.0

---

## 概要

Elementは、分散型通信プロトコルMatrixの公式iOSクライアント。E2E暗号化、音声/ビデオ通話、ブリッジ連携など、エンタープライズグレードのメッセージング機能を提供。Slackの代替として企業でも利用。

---

## 主要機能

| 機能 | 説明 |
|------|------|
| メッセージング | E2Eテキスト、画像、ファイル |
| 通話 | 音声/ビデオ通話 |
| ルーム | グループチャット |
| スペース | ルーム階層管理 |
| ブリッジ | Slack、Discord連携 |
| SSO | エンタープライズ認証 |

---

## 技術スタック

| 項目 | 技術 |
|------|------|
| 言語 | Swift |
| UI | UIKit + SwiftUI |
| プロトコル | Matrix |
| 暗号化 | Olm/Megolm |
| 通話 | WebRTC |
| SDK | matrix-ios-sdk |

---

## アーキテクチャ

```
element-ios/
├── Riot/                    # メインアプリ（レガシー名）
│   ├── Modules/             # 機能モジュール
│   │   ├── Room/            # ルーム
│   │   ├── Call/            # 通話
│   │   └── Settings/        # 設定
│   └── Managers/            # マネージャー
├── RiotSwiftUI/             # SwiftUI コンポーネント
├── MatrixSDK/               # Matrix SDK
└── MatrixKit/               # UI キット
```

---

## 学習価値

### 分散型メッセージング

| 分野 | 学習ポイント |
|------|-------------|
| **Matrix プロトコル** | 分散型通信 |
| **E2E暗号化** | Olm/Megolm |
| **WebRTC** | 音声/ビデオ通話 |
| **大規模アプリ** | モジュール設計 |
| **エンタープライズ** | SSO統合 |

### コードパターン

**Matrix セッション**:
```swift
// Matrix セッション管理
class MatrixSessionManager: ObservableObject {
    @Published var session: MXSession?
    @Published var rooms: [MXRoom] = []

    func login(homeserver: String, username: String, password: String) async throws {
        let restClient = MXRestClient(homeServer: URL(string: homeserver)!)

        let credentials = try await restClient.login(
            type: .password,
            username: username,
            password: password
        )

        let session = MXSession(credentials: credentials)
        session.start { [weak self] in
            self?.session = session
            self?.rooms = session.rooms
        }
    }

    func joinRoom(roomId: String) async throws {
        guard let session = session else { throw MatrixError.notLoggedIn }
        try await session.joinRoom(roomId)
        rooms = session.rooms
    }
}
```

**E2E暗号化メッセージ**:
```swift
// Megolm 暗号化
class EncryptedMessaging {
    private let crypto: MXCrypto

    func sendEncryptedMessage(room: MXRoom, content: String) async throws {
        // ルームの暗号化状態確認
        guard room.summary.isEncrypted else {
            try await room.sendTextMessage(content)
            return
        }

        // セッション鍵の確認
        try await ensureSessionKeys(for: room)

        // 暗号化してメッセージ送信
        let encryptedContent = try await crypto.encryptEventContent(
            ["body": content, "msgtype": "m.text"],
            for: room
        )

        try await room.sendMessage(withContent: encryptedContent)
    }

    private func ensureSessionKeys(for room: MXRoom) async throws {
        let members = room.state.members.members ?? []
        for member in members {
            if !crypto.deviceListForUser(member.userId).isEmpty {
                try await crypto.downloadKeys([member.userId])
            }
        }
    }
}
```

**WebRTC通話**:
```swift
// 音声/ビデオ通話
class CallManager: ObservableObject {
    @Published var activeCall: MXCall?

    private var peerConnection: RTCPeerConnection?
    private var localVideoTrack: RTCVideoTrack?

    func startCall(room: MXRoom, isVideo: Bool) async throws {
        let call = try await room.placeCall(withVideo: isVideo)
        activeCall = call

        call.delegate = self

        // WebRTC セットアップ
        let config = RTCConfiguration()
        config.iceServers = [RTCIceServer(urlStrings: ["stun:stun.matrix.org"])]

        peerConnection = RTCPeerConnectionFactory().peerConnection(
            with: config,
            constraints: RTCMediaConstraints(mandatoryConstraints: nil, optionalConstraints: nil),
            delegate: self
        )

        if isVideo {
            setupLocalVideo()
        }

        let offer = try await peerConnection!.offer(for: RTCMediaConstraints())
        try await peerConnection!.setLocalDescription(offer)
        try await call.sendOffer(offer.sdp)
    }

    func answerCall(_ call: MXCall) async throws {
        activeCall = call

        let answer = try await peerConnection!.answer(for: RTCMediaConstraints())
        try await peerConnection!.setLocalDescription(answer)
        try await call.answer(answer.sdp)
    }
}
```

**デバイス検証**:
```swift
// E2E デバイス検証
class DeviceVerification {
    func verifyDevice(_ device: MXDeviceInfo, using session: MXSession) async throws {
        // SAS（Short Authentication String）検証
        let request = try await session.crypto.requestVerification(
            byToDevice: device.userId,
            deviceIds: [device.deviceId]
        )

        // 絵文字比較
        let transaction = try await request.beginKeyVerification(.sas)

        if let sasTransaction = transaction as? MXSASTransaction {
            let emojis = sasTransaction.sasEmoji
            // UIで絵文字を表示してユーザーに確認
            if await confirmEmojis(emojis) {
                try await sasTransaction.confirmSAS()
            }
        }
    }
}
```

---

## プロジェクト評価

| 観点 | 評価 | 理由 |
|------|------|------|
| **コード品質** | ★★★★☆ | 大規模で複雑 |
| **ドキュメント** | ★★★★★ | Matrix仕様文書 |
| **アクティビティ** | ★★★★★ | 継続的な開発 |
| **学習価値** | ★★★★★ | 分散型メッセージングの教科書 |
| **実用性** | ★★★★★ | エンタープライズ利用 |

---

## Matrix プロトコル

### 主要概念

| 概念 | 説明 |
|------|------|
| Homeserver | ユーザーのサーバー |
| Room | チャットルーム |
| Event | メッセージ/状態変更 |
| Federation | サーバー間連携 |
| Spaces | ルームのグループ化 |

### イベント構造

```swift
// Matrix イベント
struct MatrixEvent: Codable {
    let eventId: String
    let roomId: String
    let sender: String
    let type: String
    let content: [String: Any]
    let originServerTs: Int64
}
```

---

## 開発者向け情報

### ビルド手順

```bash
git clone https://github.com/vector-im/element-ios
cd element-ios
bundle install
bundle exec pod install
open Riot.xcworkspace
```

### 必要要件

- Xcode 15+
- CocoaPods
- Bundler

---

## まとめ

Element iOSは、分散型メッセージングアプリ開発の包括的な参考資料。Matrixプロトコル、E2E暗号化、WebRTC通話など、高度な機能を網羅。エンタープライズアプリ開発にも参考になる。

**推奨対象**:
- 分散型メッセージングを学びたい人
- E2E暗号化を実装したい開発者
- WebRTC通話を統合したい人
- エンタープライズアプリを開発したい人
