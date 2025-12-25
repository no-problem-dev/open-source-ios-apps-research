# ニュース&RSSアプリ分析

**調査日**: 2025年12月25日
**対象**: 87件のニュース・RSSアプリ

---

## 概要

RSSリーダー、Hacker Newsクライアント、Redditアプリなど、情報収集系アプリは多数存在。NetNewsWireを筆頭に、高品質なオープンソースリーダーが豊富。

---

## トップアプリ

| スター | アプリ | カテゴリ | 説明 |
|--------|--------|----------|------|
| 15,668 | Readest | 電子書籍 | 多機能リーダー |
| 9,399 | NetNewsWire | RSS | RSS/Atom/JSON Feed |
| 3,746 | For Hacker News | HN | 人気HNクライアント |
| 2,420 | Designer News | ニュース | デザイン系 |
| 2,199 | Twine | RSS | RSSリーダー |
| 1,963 | v2ex | コミュニティ | 中国語テック |
| 1,779 | HN Reader | HN | Hacker News |
| 1,404 | Hacki | HN | HNクライアント |
| 1,307 | HackerNews | HN | シンプルHN |
| 1,287 | reddit-swiftui | Reddit | SwiftUI製 |

---

## カテゴリ別詳細

### RSSリーダー

| アプリ | スター | 特徴 |
|--------|--------|------|
| NetNewsWire | 9,399 | 多形式対応、同期 |
| Twine | 2,199 | モダンUI |
| RSSRead | 859 | オフライン対応 |
| FeedFlow | 719 | SwiftUI、ミニマル |
| Simple Reader | 260 | シンプル |

### Hacker Newsクライアント

| アプリ | スター | 技術 |
|--------|--------|------|
| For Hacker News | 3,746 | Swift |
| HN Reader | 1,779 | Swift |
| Hacki | 1,404 | Swift |
| HackerNews | 1,307 | Swift |
| News/YC | 826 | Swift |
| Emerge Tools HN | 198 | Swift |
| flews | 166 | マルチサービス |

### Redditクライアント

| アプリ | スター | 技術 |
|--------|--------|------|
| reddit-swiftui | 1,287 | SwiftUI |
| Slide for Reddit | 486 | Swift |
| SwipeIt | 442 | Swift |
| Beam | 274 | Swift |
| reddift | 238 | Swift |

### 電子書籍・コミック

| アプリ | スター | 形式 |
|--------|--------|------|
| Readest | 15,668 | 多形式 |
| Kiwix | 660 | Wikipedia |
| GreatReader | 613 | PDF |
| ComicFlow | 362 | コミック |

---

## 技術パターン

### RSS解析

```swift
import FeedKit

class RSSParser {
    func parse(url: URL) async throws -> [FeedItem] {
        let (data, _) = try await URLSession.shared.data(from: url)
        let parser = FeedParser(data: data)

        return try await withCheckedThrowingContinuation { continuation in
            parser.parseAsync { result in
                switch result {
                case .success(let feed):
                    let items = self.extractItems(from: feed)
                    continuation.resume(returning: items)
                case .failure(let error):
                    continuation.resume(throwing: error)
                }
            }
        }
    }

    private func extractItems(from feed: Feed) -> [FeedItem] {
        switch feed {
        case .rss(let rssFeed):
            return rssFeed.items?.map { item in
                FeedItem(
                    title: item.title ?? "",
                    link: item.link ?? "",
                    description: item.description ?? "",
                    pubDate: item.pubDate
                )
            } ?? []
        case .atom(let atomFeed):
            return atomFeed.entries?.map { entry in
                FeedItem(
                    title: entry.title ?? "",
                    link: entry.links?.first?.attributes?.href ?? "",
                    description: entry.summary?.value ?? "",
                    pubDate: entry.published
                )
            } ?? []
        case .json(let jsonFeed):
            return jsonFeed.items?.map { item in
                FeedItem(
                    title: item.title ?? "",
                    link: item.url ?? "",
                    description: item.contentText ?? "",
                    pubDate: item.datePublished
                )
            } ?? []
        }
    }
}

struct FeedItem: Identifiable {
    let id = UUID()
    let title: String
    let link: String
    let description: String
    let pubDate: Date?
}
```

### Hacker News API

```swift
// Hacker News Firebase API
class HackerNewsAPI {
    private let baseURL = "https://hacker-news.firebaseio.com/v0"

    func fetchTopStories() async throws -> [Int] {
        let url = URL(string: "\(baseURL)/topstories.json")!
        let (data, _) = try await URLSession.shared.data(from: url)
        return try JSONDecoder().decode([Int].self, from: data)
    }

    func fetchItem(id: Int) async throws -> HNItem {
        let url = URL(string: "\(baseURL)/item/\(id).json")!
        let (data, _) = try await URLSession.shared.data(from: url)
        return try JSONDecoder().decode(HNItem.self, from: data)
    }

    func fetchStories(type: StoryType, limit: Int = 30) async throws -> [HNItem] {
        let ids = try await fetchTopStories()
        let limitedIds = Array(ids.prefix(limit))

        return try await withThrowingTaskGroup(of: HNItem.self) { group in
            for id in limitedIds {
                group.addTask {
                    try await self.fetchItem(id: id)
                }
            }

            var items: [HNItem] = []
            for try await item in group {
                items.append(item)
            }
            return items.sorted { $0.score ?? 0 > $1.score ?? 0 }
        }
    }
}

struct HNItem: Codable, Identifiable {
    let id: Int
    let title: String?
    let url: String?
    let text: String?
    let by: String?
    let score: Int?
    let time: Int?
    let descendants: Int?
    let kids: [Int]?
    let type: String?
}
```

### Reddit API

```swift
// Reddit API統合
class RedditAPI {
    private let clientId: String
    private var accessToken: String?

    func fetchPosts(subreddit: String) async throws -> [RedditPost] {
        let url = URL(string: "https://www.reddit.com/r/\(subreddit)/hot.json")!
        var request = URLRequest(url: url)
        request.setValue("iOS:MyApp:1.0 (by /u/username)", forHTTPHeaderField: "User-Agent")

        let (data, _) = try await URLSession.shared.data(for: request)
        let response = try JSONDecoder().decode(RedditResponse.self, from: data)

        return response.data.children.map { $0.data }
    }
}

struct RedditResponse: Codable {
    let data: RedditData
}

struct RedditData: Codable {
    let children: [RedditChild]
}

struct RedditChild: Codable {
    let data: RedditPost
}

struct RedditPost: Codable, Identifiable {
    let id: String
    let title: String
    let selftext: String?
    let author: String
    let score: Int
    let numComments: Int
    let url: String?
    let thumbnail: String?

    enum CodingKeys: String, CodingKey {
        case id, title, selftext, author, score, url, thumbnail
        case numComments = "num_comments"
    }
}
```

### フィード同期

```swift
// 同期管理
class FeedSyncManager: ObservableObject {
    @Published var feeds: [Feed] = []
    @Published var unreadCount: Int = 0

    private let storage: FeedStorage
    private let refreshInterval: TimeInterval = 900 // 15分

    func addFeed(url: URL) async throws {
        let parser = RSSParser()
        let items = try await parser.parse(url: url)

        let feed = Feed(url: url, items: items, lastUpdated: Date())
        feeds.append(feed)
        try await storage.save(feeds)
    }

    func refreshAll() async {
        await withTaskGroup(of: Void.self) { group in
            for index in feeds.indices {
                group.addTask {
                    do {
                        let parser = RSSParser()
                        let items = try await parser.parse(url: self.feeds[index].url)
                        await MainActor.run {
                            self.feeds[index].items = items
                            self.feeds[index].lastUpdated = Date()
                        }
                    } catch {
                        print("Failed to refresh: \(error)")
                    }
                }
            }
        }
        updateUnreadCount()
    }

    private func updateUnreadCount() {
        unreadCount = feeds.flatMap(\.items).filter { !$0.isRead }.count
    }
}
```

---

## SwiftUI実装

### ニュースフィード

```swift
struct FeedListView: View {
    @StateObject var viewModel = FeedViewModel()

    var body: some View {
        NavigationStack {
            List {
                ForEach(viewModel.items) { item in
                    NavigationLink(destination: ArticleView(item: item)) {
                        FeedItemRow(item: item)
                    }
                }
            }
            .refreshable {
                await viewModel.refresh()
            }
            .navigationTitle("フィード")
        }
    }
}

struct FeedItemRow: View {
    let item: FeedItem

    var body: some View {
        VStack(alignment: .leading, spacing: 8) {
            Text(item.title)
                .font(.headline)
                .lineLimit(2)

            Text(item.description)
                .font(.subheadline)
                .foregroundColor(.secondary)
                .lineLimit(3)

            HStack {
                if let date = item.pubDate {
                    Text(date, style: .relative)
                        .font(.caption)
                        .foregroundColor(.secondary)
                }
                Spacer()
            }
        }
        .padding(.vertical, 4)
    }
}
```

### Hacker Newsスタイル

```swift
struct HNStoryRow: View {
    let story: HNItem

    var body: some View {
        VStack(alignment: .leading, spacing: 8) {
            // タイトル
            Text(story.title ?? "")
                .font(.headline)

            // メタ情報
            HStack(spacing: 16) {
                Label("\(story.score ?? 0)", systemImage: "arrow.up")
                Label("\(story.descendants ?? 0)", systemImage: "bubble.right")
                Text(story.by ?? "")
                    .foregroundColor(.orange)
            }
            .font(.caption)
            .foregroundColor(.secondary)

            // ドメイン
            if let url = story.url, let host = URL(string: url)?.host {
                Text(host)
                    .font(.caption2)
                    .foregroundColor(.blue)
            }
        }
    }
}
```

---

## オフライン対応

```swift
// 記事キャッシュ
class ArticleCache {
    private let cacheDirectory: URL

    init() {
        cacheDirectory = FileManager.default.urls(
            for: .cachesDirectory,
            in: .userDomainMask
        )[0].appendingPathComponent("articles")

        try? FileManager.default.createDirectory(
            at: cacheDirectory,
            withIntermediateDirectories: true
        )
    }

    func cache(article: Article) async throws {
        let data = try JSONEncoder().encode(article)
        let filename = article.id.uuidString + ".json"
        let fileURL = cacheDirectory.appendingPathComponent(filename)
        try data.write(to: fileURL)
    }

    func retrieve(id: UUID) throws -> Article? {
        let filename = id.uuidString + ".json"
        let fileURL = cacheDirectory.appendingPathComponent(filename)

        guard FileManager.default.fileExists(atPath: fileURL.path) else {
            return nil
        }

        let data = try Data(contentsOf: fileURL)
        return try JSONDecoder().decode(Article.self, from: data)
    }

    func clearOldCache(olderThan days: Int) {
        let cutoff = Date().addingTimeInterval(-TimeInterval(days * 86400))
        // 古いファイルを削除
    }
}
```

---

## Swiftパッケージ機会

### FeedParserKit

**機能**:
- RSS/Atom/JSON Feed解析
- 自動フォーマット検出
- エラーハンドリング
- SwiftUIプレビュー

### NewsAggregatorKit

**機能**:
- マルチソース統合
- 同期管理
- オフラインキャッシュ
- 通知統合

### HackerNewsKit

**機能**:
- Firebase API統合
- コメントツリー
- ユーザー情報
- キャッシュ

---

## 学習リソース

| アプリ | 学習ポイント |
|--------|-------------|
| NetNewsWire | 大規模RSSリーダー設計 |
| reddit-swiftui | SwiftUI + API統合 |
| Hacki | HN API使用 |
| FeedFlow | ミニマルSwiftUI |

---

## 開発上の考慮事項

### パフォーマンス

| 項目 | 対策 |
|------|------|
| 並列取得 | async/await + TaskGroup |
| キャッシュ | 適切な期限設定 |
| 画像 | 遅延読み込み |
| リスト | LazyVStack使用 |

### ユーザー体験

| 項目 | 実装 |
|------|------|
| Pull to Refresh | .refreshable |
| 未読管理 | バッジ表示 |
| オフライン | キャッシュ活用 |
| 共有 | ShareLink |

---

## トレンド

### 成長中
- SwiftUI製リーダー
- マルチソース統合
- AI要約機能

### 安定
- RSSリーダー
- HNクライアント
- Redditクライアント

### 注目
- アルゴリズムフィード
- プライバシー重視
- 分散型ニュース

