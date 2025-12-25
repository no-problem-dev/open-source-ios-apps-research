# Signal iOS

**スター**: 11,736 ⭐
**リポジトリ**: https://github.com/signalapp/Signal-iOS
**ライセンス**: GPL-3.0

---

## 概要

Signalは、世界で最も信頼されるエンドツーエンド暗号化メッセージングアプリ。プライバシーとセキュリティを最優先に設計され、メッセージ、通話、ビデオ通話すべてが暗号化される。

---

## 主要機能

| 機能 | 説明 |
|------|------|
| E2E暗号化 | Signal Protocol |
| メッセージ | テキスト、画像、動画、音声 |
| 音声通話 | 暗号化された通話 |
| ビデオ通話 | グループビデオ対応 |
| 消えるメッセージ | 時限削除 |
| 画面セキュリティ | スクリーンショット防止 |

---

## 技術スタック

| 項目 | 技術 |
|------|------|
| 言語 | Objective-C, Swift |
| 暗号化 | Signal Protocol, OpenSSL |
| UI | UIKit |
| 依存管理 | Carthage |
| レイアウト | PureLayout |
| WebSocket | SocketRocket |

---

## アーキテクチャ

```
Signal-iOS/
├── Signal/              # メインアプリ
│   ├── src/             # ソースコード
│   │   ├── Messages/    # メッセージ処理
│   │   ├── Calls/       # 通話機能
│   │   └── ViewControllers/
├── SignalServiceKit/    # バックエンド通信
├── SignalMessaging/     # メッセージング共通
└── SignalCoreKit/       # コア機能
```

---

## 学習価値

### セキュリティ実装

| 分野 | 学習ポイント |
|------|-------------|
| **暗号化** | Signal Protocol実装 |
| **鍵管理** | 公開鍵インフラ |
| **セキュアストレージ** | Keychain活用 |
| **プロトコル** | 独自通信プロトコル |
| **認証** | 電話番号認証 |

### コードパターン

**暗号化メッセージ送信**:
```objc
// メッセージ暗号化
- (TSOutgoingMessage *)sendMessage:(NSString *)body
                      toThread:(TSThread *)thread {
    TSOutgoingMessage *message = [[TSOutgoingMessage alloc]
        initWithTimestamp:[NSDate ows_millisecondTimeStamp]
        inThread:thread
        messageBody:body];

    // Signal Protocol で暗号化
    [self.messageSender sendMessage:message
                       success:^{ /* 成功 */ }
                       failure:^(NSError *error) { /* 失敗 */ }];

    return message;
}
```

**セキュアストレージ**:
```swift
// Keychainアクセス
class SecureStorage {
    func storeKey(_ key: Data, for identifier: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassKey,
            kSecAttrApplicationTag as String: identifier,
            kSecValueData as String: key,
            kSecAttrAccessible as String: kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly
        ]
        SecItemAdd(query as CFDictionary, nil)
    }
}
```

---

## プロジェクト評価

| 観点 | 評価 | 理由 |
|------|------|------|
| **コード品質** | ★★★★★ | セキュリティ監査済み |
| **ドキュメント** | ★★★★☆ | 技術文書充実 |
| **アクティビティ** | ★★★★★ | 継続的な開発 |
| **学習価値** | ★★★★★ | セキュリティの教科書 |
| **実用性** | ★★★★★ | 世界中で利用 |

---

## セキュリティ特徴

| 項目 | 実装 |
|------|------|
| プロトコル | Signal Protocol (Double Ratchet) |
| 前方秘匿性 | 完全前方秘匿性 |
| 鍵交換 | X3DH |
| 暗号化 | AES-256, Curve25519 |
| メタデータ保護 | Sealed Sender |

---

## 開発者向け情報

### ビルド要件

```bash
# CocoaPodsインストール
sudo gem install cocoapods

# 依存関係インストール
pod install

# Xcodeで開く
open Signal.xcworkspace
```

### 貢献について

- セキュリティ関連の変更は厳格なレビュー
- バグバウンティプログラム存在

---

## まとめ

Signalは、モバイルセキュリティの最高峰を学べるプロジェクト。暗号化プロトコル、セキュアな通信、鍵管理など、セキュリティエンジニアにとって必須の知識が詰まっている。

**推奨対象**:
- セキュリティエンジニア
- 暗号化実装を学びたい開発者
- プライバシー重視アプリ開発者

