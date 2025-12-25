# Wikipedia iOS

**スター**: 3,314 ⭐
**リポジトリ**: https://github.com/wikimedia/wikipedia-ios
**ライセンス**: MIT

---

## 概要

Wikipedia iOSは、Wikimedia Foundationによる公式Wikipediaアプリ。オフライン閲覧、多言語対応、地図統合など、豊富な機能を持つ大規模オープンソースプロジェクト。

---

## 主要機能

| 機能 | 説明 |
|------|------|
| 記事閲覧 | 全言語Wikipedia |
| オフライン | 記事保存 |
| 検索 | 高度な検索機能 |
| 探索 | おすすめ記事 |
| 地図 | 近くの記事 |
| 編集 | 記事編集対応 |

---

## 技術スタック

| 項目 | 技術 |
|------|------|
| 言語 | Swift, Objective-C |
| UI | UIKit |
| ストレージ | Core Data |
| ネットワーク | URLSession |
| 地図 | MapKit |
| 多言語 | RTL対応 |

---

## アーキテクチャ

```
wikipedia-ios/
├── Wikipedia/               # メインアプリ
│   ├── Code/                # ソースコード
│   │   ├── Article/         # 記事表示
│   │   ├── Search/          # 検索機能
│   │   ├── Explore/         # 探索フィード
│   │   ├── Places/          # 地図機能
│   │   ├── Saved/           # 保存記事
│   │   └── Settings/        # 設定
│   └── Storyboards/         # UI定義
├── WMF Framework/           # 共有フレームワーク
│   ├── Core Data/           # データモデル
│   ├── Networking/          # API クライアント
│   └── Utilities/           # ユーティリティ
└── WikipediaUnitTests/      # テスト
```

---

## 学習価値

### 大規模アプリ設計

| 分野 | 学習ポイント |
|------|-------------|
| **大規模設計** | 100万行級コードベース |
| **多言語対応** | RTL、ローカライゼーション |
| **オフライン** | Core Data同期 |
| **コンテンツ表示** | リッチテキストレンダリング |
| **アクセシビリティ** | VoiceOver完全対応 |

### コードパターン

**記事表示**:
```swift
// WebView ベースの記事表示
class ArticleViewController: UIViewController {
    private let webView: WKWebView
    private let article: WMFArticle

    override func viewDidLoad() {
        super.viewDidLoad()
        setupWebView()
        loadArticle()
    }

    private func loadArticle() {
        let request = ArticleRequest(title: article.title, language: article.language)

        articleService.fetch(request) { [weak self] result in
            switch result {
            case .success(let content):
                self?.renderArticle(content)
            case .failure(let error):
                self?.showError(error)
            }
        }
    }

    private func renderArticle(_ content: ArticleContent) {
        let html = ArticleRenderer.render(
            content: content,
            theme: theme,
            textSize: textSizeMultiplier
        )
        webView.loadHTMLString(html, baseURL: content.baseURL)
    }
}
```

**オフライン保存**:
```swift
// Core Data を使用したオフライン保存
class SavedArticlesController {
    private let dataStore: MWKDataStore

    func saveArticle(_ article: WMFArticle) async throws {
        // メタデータ保存
        try await dataStore.save { context in
            let savedArticle = SavedArticle(context: context)
            savedArticle.title = article.title
            savedArticle.url = article.url
            savedArticle.savedDate = Date()
        }

        // コンテンツダウンロード
        let content = try await articleService.fetchForOffline(article)
        try await saveContent(content, for: article)

        // 画像ダウンロード
        let images = content.imageURLs
        try await imageCache.prefetch(images)
    }

    func getSavedArticles() -> [SavedArticle] {
        let request: NSFetchRequest<SavedArticle> = SavedArticle.fetchRequest()
        request.sortDescriptors = [NSSortDescriptor(key: "savedDate", ascending: false)]
        return try? dataStore.viewContext.fetch(request) ?? []
    }
}
```

**多言語対応**:
```swift
// RTL対応レイアウト
class LocalizedViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        setupForLanguage()
    }

    private func setupForLanguage() {
        let language = MWKLanguageLinkController.shared.appLanguage

        // RTL言語対応
        if language.isRTL {
            view.semanticContentAttribute = .forceRightToLeft
        } else {
            view.semanticContentAttribute = .forceLeftToRight
        }

        // フォント設定
        applyFont(for: language)
    }

    private func applyFont(for language: MWKLanguageLink) {
        // 言語固有のフォント設定
        let fontFamily = language.preferredFontFamily
        let fontSize = language.preferredFontSize
        // ...
    }
}
```

**探索フィード**:
```swift
// パーソナライズされたフィード
class ExploreFeedController {
    private var feedItems: [FeedItem] = []

    func refresh() async throws {
        let language = currentLanguage

        // 並列フェッチ
        async let featured = fetchFeaturedArticle(language: language)
        async let onThisDay = fetchOnThisDay(language: language)
        async let trending = fetchTrending(language: language)
        async let nearby = fetchNearby()

        feedItems = try await [
            featured,
            onThisDay,
            trending,
            nearby
        ].compactMap { $0 }

        notifyUpdate()
    }
}
```

---

## プロジェクト評価

| 観点 | 評価 | 理由 |
|------|------|------|
| **コード品質** | ★★★★☆ | 大規模だが整理されている |
| **ドキュメント** | ★★★★★ | 貢献ガイド充実 |
| **アクティビティ** | ★★★★★ | 継続的な開発 |
| **学習価値** | ★★★★★ | 大規模アプリの教科書 |
| **実用性** | ★★★★★ | 公式Wikipediaアプリ |

---

## ローカライゼーション

### 対応言語

- 300以上のWikipedia言語
- RTL言語完全対応（アラビア語、ヘブライ語など）
- 動的フォントサイズ

### 実装

```swift
// 多言語文字列
extension String {
    var wmf_localized: String {
        return NSLocalizedString(self, bundle: Bundle.wmf, comment: "")
    }

    func wmf_localizedString(withValue value: String) -> String {
        return String.localizedStringWithFormat(wmf_localized, value)
    }
}
```

---

## アクセシビリティ

```swift
// VoiceOver対応
extension ArticleViewController {
    override func accessibilityPerformMagicTap() -> Bool {
        // 記事の読み上げ開始/停止
        toggleReadAloud()
        return true
    }

    func configureAccessibility() {
        // 見出しジャンプ対応
        webView.accessibilityTraits = .header

        // カスタムアクション
        let actions = [
            UIAccessibilityCustomAction(
                name: "Save article",
                target: self,
                selector: #selector(saveArticle)
            )
        ]
        accessibilityCustomActions = actions
    }
}
```

---

## 開発者向け情報

### ビルド手順

```bash
git clone https://github.com/wikimedia/wikipedia-ios
cd wikipedia-ios
./scripts/setup
open Wikipedia.xcodeproj
```

### 貢献

- 活発なコミュニティ
- Good First Issue ラベル
- 詳細な貢献ガイド

---

## まとめ

Wikipedia iOSは、大規模オープンソースiOSアプリの模範。多言語対応、オフライン機能、アクセシビリティなど、グローバルアプリの要件を網羅。オープンソースコミュニティ運営の参考にもなる。

**推奨対象**:
- 大規模アプリの設計を学びたい開発者
- 多言語・RTL対応を実装したい人
- オフライン機能を実装したい人
- オープンソースプロジェクトに貢献したい人
