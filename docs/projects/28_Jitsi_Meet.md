# Jitsi Meet iOS

**スター**: 2,500+ ⭐
**リポジトリ**: https://github.com/jitsi/jitsi-meet
**ライセンス**: Apache-2.0

---

## 概要

Jitsi Meetは、オープンソースのビデオ会議プラットフォーム。iOS SDKを提供し、アプリへのビデオ通話機能統合が可能。セルフホスト可能で、プライバシー重視の企業に人気。

---

## 主要機能

| 機能 | 説明 |
|------|------|
| ビデオ会議 | 多人数対応 |
| 画面共有 | iOS画面共有 |
| チャット | テキストチャット |
| 挙手 | 発言リクエスト |
| 録画 | サーバー録画 |
| E2E暗号化 | オプション暗号化 |

---

## 技術スタック

| 項目 | 技術 |
|------|------|
| 言語 | Swift, React Native |
| ビデオ | WebRTC |
| シグナリング | XMPP (Prosody) |
| UI | React Native |
| SDK | JitsiMeetSDK |
| 画面共有 | ReplayKit |

---

## アーキテクチャ

```
jitsi-meet/
├── ios/                     # iOS アプリ
│   ├── app/                 # メインアプリ
│   ├── sdk/                 # JitsiMeetSDK
│   │   ├── JitsiMeet.h
│   │   └── JitsiMeetView.swift
│   └── broadcast/           # 画面共有拡張
├── react/                   # React Native コード
│   ├── features/            # 機能モジュール
│   └── components/          # UIコンポーネント
└── modules/                 # ネイティブモジュール
```

---

## 学習価値

### ビデオ会議実装

| 分野 | 学習ポイント |
|------|-------------|
| **WebRTC** | リアルタイム通信 |
| **SDK設計** | 再利用可能なSDK |
| **React Native** | ネイティブ連携 |
| **画面共有** | ReplayKit活用 |
| **SRTP** | メディア暗号化 |

### コードパターン

**JitsiMeet SDK統合**:
```swift
// SDK統合
import JitsiMeetSDK

class VideoConferenceViewController: UIViewController {
    private var jitsiMeetView: JitsiMeetView?

    func joinMeeting(roomName: String) {
        jitsiMeetView = JitsiMeetView()
        jitsiMeetView?.delegate = self

        let options = JitsiMeetConferenceOptions.fromBuilder { builder in
            builder.room = roomName
            builder.serverURL = URL(string: "https://meet.jit.si")

            // 機能設定
            builder.setFeatureFlag("welcomepage.enabled", withBoolean: false)
            builder.setFeatureFlag("pip.enabled", withBoolean: true)
            builder.setFeatureFlag("call-integration.enabled", withBoolean: true)

            // ユーザー情報
            builder.userInfo = JitsiMeetUserInfo(
                displayName: "User Name",
                email: "user@example.com",
                avatar: URL(string: "https://example.com/avatar.png")
            )
        }

        jitsiMeetView?.join(options)
        view.addSubview(jitsiMeetView!)
    }

    func leaveMeeting() {
        jitsiMeetView?.leave()
    }
}

extension VideoConferenceViewController: JitsiMeetViewDelegate {
    func conferenceJoined(_ data: [AnyHashable: Any]!) {
        print("会議に参加: \(data["url"] ?? "")")
    }

    func conferenceTerminated(_ data: [AnyHashable: Any]!) {
        jitsiMeetView?.removeFromSuperview()
        jitsiMeetView = nil
    }

    func participantJoined(_ data: [AnyHashable: Any]!) {
        print("参加者入室: \(data["participantId"] ?? "")")
    }
}
```

**画面共有（ReplayKit）**:
```swift
// Broadcast Upload Extension
class SampleHandler: RPBroadcastSampleHandler {
    private var webSocket: URLSessionWebSocketTask?

    override func broadcastStarted(withSetupInfo setupInfo: [String: NSObject]?) {
        // WebSocket接続
        let session = URLSession.shared
        let url = URL(string: "wss://meet.jit.si/broadcast")!
        webSocket = session.webSocketTask(with: url)
        webSocket?.resume()
    }

    override func processSampleBuffer(_ sampleBuffer: CMSampleBuffer, with sampleBufferType: RPSampleBufferType) {
        switch sampleBufferType {
        case .video:
            sendVideoFrame(sampleBuffer)
        case .audioApp:
            sendAudioFrame(sampleBuffer, isApp: true)
        case .audioMic:
            sendAudioFrame(sampleBuffer, isApp: false)
        @unknown default:
            break
        }
    }

    private func sendVideoFrame(_ sampleBuffer: CMSampleBuffer) {
        guard let imageBuffer = CMSampleBufferGetImageBuffer(sampleBuffer) else { return }

        // フレームをエンコードして送信
        let encoder = VideoEncoder()
        let encodedData = encoder.encode(imageBuffer)

        webSocket?.send(.data(encodedData)) { error in
            if let error = error {
                print("送信エラー: \(error)")
            }
        }
    }
}
```

**CallKit統合**:
```swift
// CallKit で着信表示
class CallKitManager: NSObject, CXProviderDelegate {
    private let provider: CXProvider
    private let callController = CXCallController()

    override init() {
        let config = CXProviderConfiguration()
        config.supportsVideo = true
        config.supportedHandleTypes = [.generic]
        config.iconTemplateImageData = UIImage(named: "CallKitIcon")?.pngData()

        provider = CXProvider(configuration: config)
        super.init()
        provider.setDelegate(self, queue: nil)
    }

    func reportIncomingCall(uuid: UUID, roomName: String) {
        let update = CXCallUpdate()
        update.remoteHandle = CXHandle(type: .generic, value: roomName)
        update.hasVideo = true

        provider.reportNewIncomingCall(with: uuid, update: update) { error in
            if let error = error {
                print("着信報告エラー: \(error)")
            }
        }
    }

    func providerDidReset(_ provider: CXProvider) {
        // 通話リセット
    }

    func provider(_ provider: CXProvider, perform action: CXAnswerCallAction) {
        // 応答処理
        JitsiMeet.shared.joinRoom(action.callUUID.uuidString)
        action.fulfill()
    }
}
```

---

## プロジェクト評価

| 観点 | 評価 | 理由 |
|------|------|------|
| **コード品質** | ★★★★☆ | 大規模プロジェクト |
| **ドキュメント** | ★★★★★ | SDK文書充実 |
| **アクティビティ** | ★★★★★ | 継続的な開発 |
| **学習価値** | ★★★★★ | ビデオ会議の教科書 |
| **実用性** | ★★★★★ | エンタープライズ利用 |

---

## SDK統合オプション

### 機能フラグ

| フラグ | 説明 |
|-------|------|
| pip.enabled | Picture-in-Picture |
| call-integration.enabled | CallKit統合 |
| chat.enabled | チャット機能 |
| recording.enabled | 録画機能 |
| live-streaming.enabled | ライブ配信 |

---

## セルフホスト

```bash
# Docker でJitsiサーバー起動
git clone https://github.com/jitsi/docker-jitsi-meet
cd docker-jitsi-meet
./gen-passwords.sh
docker-compose up -d
```

---

## 開発者向け情報

### ビルド手順

```bash
git clone https://github.com/jitsi/jitsi-meet
cd jitsi-meet
npm install
cd ios
pod install
open jitsi-meet.xcworkspace
```

### 必要要件

- Xcode 15+
- Node.js
- CocoaPods

---

## まとめ

Jitsi Meetは、ビデオ会議機能をアプリに統合する最良の参考資料。WebRTC、画面共有、CallKit統合など、リアルタイム通信の技術を網羅。SDKの設計パターンも参考になる。

**推奨対象**:
- ビデオ会議機能を統合したい開発者
- WebRTCを学びたい人
- SDK設計を参考にしたい人
- 画面共有を実装したい人
