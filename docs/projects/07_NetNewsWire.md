# NetNewsWire

**スター**: 9,399 ⭐
**リポジトリ**: https://github.com/Ranchero-Software/NetNewsWire
**ライセンス**: MIT

---

## 概要

NetNewsWireは、macOS/iOS向けの老舗RSSリーダー。2002年からの歴史を持ち、現在はオープンソースとして公開。RSS、Atom、JSON Feedをサポートし、複数の同期サービスと連携可能。

---

## 主要機能

| 機能 | 説明 |
|------|------|
| フィード形式 | RSS, Atom, JSON Feed, RSS-in-JSON |
| 同期 | iCloud, Feedbin, Feedly, etc. |
| スマートフィード | 未読、スター、今日 |
| 検索 | 全文検索 |
| Safari拡張 | 購読ボタン |
| ウィジェット | iOS 14+ウィジェット |

---

## 技術スタック

| 項目 | 技術 |
|------|------|
| 言語 | Swift (100%) |
| UI | AppKit (macOS), UIKit (iOS) |
| データ | SQLite (FMDB) |
| 同期 | CloudKit, REST API |
| パース | XMLParser, JSONDecoder |

---

## アーキテクチャ

```
NetNewsWire/
├── Mac/                 # macOSアプリ
├── iOS/                 # iOSアプリ
├── Shared/              # 共有コード
├── Account/             # アカウント管理
├── Articles/            # 記事モデル
├── ArticlesDatabase/    # SQLiteデータベース
├── Parser/              # フィードパーサー
├── RSCore/              # コアユーティリティ
├── RSWeb/               # Web関連
└── Secrets/             # API キー管理
```

---

## 学習価値

### クリーンなSwift設計

| 分野 | 学習ポイント |
|------|-------------|
| **マルチプラットフォーム** | iOS/macOS共通コード |
| **データ層** | SQLite抽象化 |
| **同期** | 複数バックエンド対応 |
| **パース** | XML/JSONフィード解析 |
| **キャッシュ** | 効率的な画像キャッシュ |

### コードパターン

**フィードパース**:
```swift
// RSSパーサー
struct FeedParser {
    func parse(data: Data) throws -> ParsedFeed {
        // 形式を自動検出
        if let rssFeed = try? parseRSS(data) {
            return rssFeed
        }
        if let atomFeed = try? parseAtom(data) {
            return atomFeed
        }
        if let jsonFeed = try? parseJSON(data) {
            return jsonFeed
        }
        throw ParserError.unknownFormat
    }
}
```

**アカウント抽象化**:
```swift
// 複数サービス対応
protocol AccountDelegate {
    func refreshAll() async throws
    func markAsRead(_ article: Article) async throws
    func markAsStarred(_ article: Article, starred: Bool) async throws
}

class FeedbinAccountDelegate: AccountDelegate {
    // Feedbin API実装
}

class iCloudAccountDelegate: AccountDelegate {
    // CloudKit実装
}
```

**スマートフィード**:
```swift
// 動的フィード
class SmartFeed {
    let predicate: (Article) -> Bool

    static let unread = SmartFeed { !$0.isRead }
    static let starred = SmartFeed { $0.isStarred }
    static let today = SmartFeed {
        Calendar.current.isDateInToday($0.datePublished)
    }
}
```

---

## プロジェクト評価

| 観点 | 評価 | 理由 |
|------|------|------|
| **コード品質** | ★★★★★ | 20年の歴史に裏付け |
| **ドキュメント** | ★★★★☆ | 開発ガイドあり |
| **アクティビティ** | ★★★★★ | 継続的なリリース |
| **学習価値** | ★★★★★ | クリーンなSwift設計 |
| **実用性** | ★★★★★ | 実用的なRSSリーダー |

---

## iOS/macOS共通コード

### 共有フレームワーク

| フレームワーク | 役割 |
|---------------|------|
| Account | アカウント管理 |
| Articles | 記事モデル |
| ArticlesDatabase | SQLite操作 |
| Parser | フィード解析 |
| RSCore | ユーティリティ |

### プラットフォーム固有

```swift
#if os(macOS)
import AppKit
typealias PlatformViewController = NSViewController
#else
import UIKit
typealias PlatformViewController = UIViewController
#endif
```

---

## 開発者向け情報

### ビルド手順

```bash
# クローン
git clone https://github.com/Ranchero-Software/NetNewsWire.git

# サブモジュール取得
git submodule update --init --recursive

# Xcodeで開く
open NetNewsWire.xcworkspace
```

### 貢献ガイド

- Help Wanted ラベルの Issue から開始
- コードスタイルガイド遵守
- ユニットテスト必須

---

## まとめ

NetNewsWireは、Swiftの美しいコードと20年以上の歴史を持つ成熟したプロジェクト。マルチプラットフォーム開発、クリーンアーキテクチャ、データ同期などを学ぶ最良の教材。

**推奨対象**:
- クリーンなSwiftコードを学びたい人
- iOS/macOS両対応アプリ開発者
- RSSリーダー開発に興味がある人
- データ同期パターンを学びたい人

