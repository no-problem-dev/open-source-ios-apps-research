# Firefox iOS

**スター**: 12,796 ⭐
**リポジトリ**: https://github.com/mozilla-mobile/firefox-ios
**ライセンス**: MPL-2.0

---

## 概要

Mozilla公式のiOS向けFirefoxブラウザ。WebKitベースながら、Firefoxの特徴であるプライバシー保護、同期機能、拡張機能を提供。大規模Swift/UIKitプロジェクトの代表例。

---

## 主要機能

| 機能 | 説明 |
|------|------|
| ブラウジング | WKWebViewベース |
| Firefox Sync | ブックマーク・履歴同期 |
| トラッキング防止 | Enhanced Tracking Protection |
| プライベートモード | 追跡なしブラウジング |
| 拡張機能 | 一部アドオン対応 |
| リーダーモード | 読みやすい表示 |

---

## 技術スタック

| 項目 | 技術 |
|------|------|
| 言語 | Swift (99%) |
| UI | UIKit, SwiftUI (部分) |
| 依存管理 | Carthage |
| ネットワーク | Alamofire |
| レイアウト | SnapKit |
| 電話番号 | libphonenumber |

---

## アーキテクチャ

```
firefox-ios/
├── Client/              # メインアプリ
│   ├── Frontend/        # UI層
│   │   ├── Browser/     # ブラウザUI
│   │   ├── Home/        # ホーム画面
│   │   └── Settings/    # 設定
│   ├── Application/     # アプリケーション層
│   └── Extensions/      # 拡張機能
├── Storage/             # データ永続化
├── Sync/                # Firefox Sync
├── Shared/              # 共有コード
└── WidgetKit/           # ウィジェット
```

---

## 学習価値

### 大規模プロジェクト設計

| 分野 | 学習ポイント |
|------|-------------|
| **モジュール化** | 機能別の分離 |
| **テスト** | 包括的なテストスイート |
| **CI/CD** | GitHub Actions |
| **ローカライゼーション** | 多言語対応 |
| **アクセシビリティ** | VoiceOver完全対応 |

### コードパターン

**タブ管理**:
```swift
class TabManager {
    private(set) var tabs: [Tab] = []
    var selectedTab: Tab?

    func addTab(url: URL? = nil) -> Tab {
        let tab = Tab(configuration: webViewConfig)
        tabs.append(tab)
        return tab
    }

    func removeTab(_ tab: Tab) {
        tabs.removeAll { $0 === tab }
        tab.close()
    }
}
```

**トラッキング防止**:
```swift
class TrackingProtection {
    func shouldBlock(request: URLRequest) -> Bool {
        guard let host = request.url?.host else { return false }
        return blockList.contains(host)
    }
}
```

---

## プロジェクト評価

| 観点 | 評価 | 理由 |
|------|------|------|
| **コード品質** | ★★★★★ | Mozilla品質基準 |
| **ドキュメント** | ★★★★★ | 詳細な開発ガイド |
| **アクティビティ** | ★★★★★ | 毎日のコミット |
| **学習価値** | ★★★★★ | 大規模プロジェクトの手本 |
| **実用性** | ★★★★★ | 実際のApp Store公開 |

---

## 開発者向け情報

### ビルド手順

```bash
# 1. リポジトリクローン
git clone https://github.com/mozilla-mobile/firefox-ios

# 2. 依存関係インストール
./bootstrap.sh

# 3. Xcodeで開く
open Client.xcodeproj
```

### 貢献ガイド

- Good First Issues ラベル活用
- コードレビュープロセス明確
- 自動テスト必須

---

## 関連リソース

- [開発者ドキュメント](https://github.com/mozilla-mobile/firefox-ios/wiki)
- [Mozilla Developer Network](https://developer.mozilla.org/)
- [貢献ガイド](https://github.com/mozilla-mobile/firefox-ios/blob/main/CONTRIBUTING.md)

---

## まとめ

Firefox iOSは、大規模なSwiftプロジェクトの最良の参考例の一つ。モジュール化、テスト、CI/CD、アクセシビリティなど、プロフェッショナルなiOS開発のあらゆる側面を学べる。

**推奨対象**:
- 大規模プロジェクト設計を学びたい開発者
- ブラウザ開発に興味がある人
- チーム開発のベストプラクティスを知りたい人

