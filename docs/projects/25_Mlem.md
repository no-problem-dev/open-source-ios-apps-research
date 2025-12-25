# Mlem

**スター**: 1,100+ ⭐
**リポジトリ**: https://github.com/mlemgroup/mlem
**ライセンス**: GPL-3.0

---

## 概要

Mlemは、iOS向けのネイティブLemmyクライアント。Lemmyは分散型Reddit代替サービスで、MlemはSwiftUIで構築された美しいインターフェースを提供。フェデレーション対応のソーシャルメディアアプリ開発の参考に。

---

## 主要機能

| 機能 | 説明 |
|------|------|
| フィード | ホーム、ローカル、全体 |
| コミュニティ | 購読、ブラウズ |
| 投稿 | テキスト、リンク、画像 |
| コメント | スレッド表示 |
| 投票 | upvote/downvote |
| 検索 | ユーザー、コミュニティ |

---

## 技術スタック

| 項目 | 技術 |
|------|------|
| 言語 | Swift |
| UI | SwiftUI (100%) |
| アーキテクチャ | MVVM |
| ネットワーク | URLSession + async/await |
| 画像 | Nuke |
| Markdown | swift-markdown-ui |

---

## アーキテクチャ

```
Mlem/
├── Mlem/                    # メインアプリ
│   ├── App/                 # アプリ起動
│   ├── Views/               # SwiftUI ビュー
│   │   ├── Tabs/            # タブビュー
│   │   │   ├── Feed/        # フィード
│   │   │   ├── Inbox/       # 受信箱
│   │   │   └── Profile/     # プロフィール
│   │   ├── Shared/          # 共有コンポーネント
│   │   └── Components/      # UIコンポーネント
│   ├── Models/              # データモデル
│   ├── API/                 # Lemmy API
│   └── Managers/            # マネージャー
└── MlemTests/               # テスト
```

---

## 学習価値

### フェデレーション対応アプリ

| 分野 | 学習ポイント |
|------|-------------|
| **SwiftUI** | 複雑なUI実装 |
| **フェデレーション** | 分散型API統合 |
| **Markdown** | リッチテキスト表示 |
| **無限スクロール** | ページネーション |
| **オフライン** | キャッシュ戦略 |

### コードパターン

**フェデレーションAPI**:
```swift
// Lemmy API クライアント
class LemmyAPI {
    private let session: URLSession
    private var instanceURL: URL
    private var authToken: String?

    func getPosts(
        communityId: Int? = nil,
        sort: SortType = .hot,
        page: Int = 1
    ) async throws -> [Post] {
        var components = URLComponents(url: instanceURL.appendingPathComponent("/api/v3/post/list"), resolvingAgainstBaseURL: true)!

        var queryItems: [URLQueryItem] = [
            URLQueryItem(name: "sort", value: sort.rawValue),
            URLQueryItem(name: "page", value: String(page))
        ]

        if let communityId = communityId {
            queryItems.append(URLQueryItem(name: "community_id", value: String(communityId)))
        }

        components.queryItems = queryItems

        var request = URLRequest(url: components.url!)
        if let token = authToken {
            request.addValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        }

        let (data, _) = try await session.data(for: request)
        let response = try JSONDecoder().decode(GetPostsResponse.self, from: data)
        return response.posts
    }

    func vote(postId: Int, score: Int) async throws {
        let body = CreatePostLike(post_id: postId, score: score)

        var request = URLRequest(url: instanceURL.appendingPathComponent("/api/v3/post/like"))
        request.httpMethod = "POST"
        request.httpBody = try JSONEncoder().encode(body)
        request.addValue("application/json", forHTTPHeaderField: "Content-Type")

        if let token = authToken {
            request.addValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        }

        let (_, response) = try await session.data(for: request)
        guard (response as? HTTPURLResponse)?.statusCode == 200 else {
            throw LemmyError.voteFailed
        }
    }
}
```

**無限スクロール**:
```swift
// ページネーション付きフィード
class FeedViewModel: ObservableObject {
    @Published var posts: [Post] = []
    @Published var isLoading = false
    @Published var hasMorePages = true

    private var currentPage = 1
    private let api: LemmyAPI

    func loadInitialPosts() async {
        currentPage = 1
        posts = []
        await loadMorePosts()
    }

    func loadMorePosts() async {
        guard !isLoading && hasMorePages else { return }

        isLoading = true
        defer { isLoading = false }

        do {
            let newPosts = try await api.getPosts(page: currentPage)
            posts.append(contentsOf: newPosts)
            currentPage += 1
            hasMorePages = !newPosts.isEmpty
        } catch {
            // エラーハンドリング
        }
    }

    func loadMoreIfNeeded(currentPost: Post) {
        guard let index = posts.firstIndex(where: { $0.id == currentPost.id }) else { return }

        // 残り5件で次ページ読み込み
        if index >= posts.count - 5 {
            Task { await loadMorePosts() }
        }
    }
}
```

**コメントスレッド表示**:
```swift
// ネスト構造コメント表示
struct CommentTreeView: View {
    let comments: [Comment]
    let depth: Int

    var body: some View {
        ForEach(comments) { comment in
            VStack(alignment: .leading, spacing: 0) {
                CommentView(comment: comment, depth: depth)

                if !comment.replies.isEmpty {
                    CommentTreeView(
                        comments: comment.replies,
                        depth: depth + 1
                    )
                }
            }
        }
    }
}

struct CommentView: View {
    let comment: Comment
    let depth: Int

    private var indentWidth: CGFloat {
        CGFloat(depth) * 16
    }

    var body: some View {
        HStack(alignment: .top, spacing: 0) {
            // インデントライン
            if depth > 0 {
                Rectangle()
                    .fill(Color.gray.opacity(0.3))
                    .frame(width: 2)
                    .padding(.leading, indentWidth - 8)
            }

            VStack(alignment: .leading) {
                // ユーザー名
                HStack {
                    Text(comment.creator.name)
                        .font(.caption)
                        .fontWeight(.semibold)
                    Text(comment.counts.score.description)
                        .font(.caption2)
                }

                // コンテンツ
                Markdown(comment.content)
                    .font(.body)

                // アクションバー
                CommentActionsView(comment: comment)
            }
            .padding(.leading, 8)
        }
        .padding(.vertical, 8)
    }
}
```

**投票システム**:
```swift
// 投票UI
struct VoteButtons: View {
    @ObservedObject var viewModel: PostViewModel

    var body: some View {
        HStack(spacing: 4) {
            Button {
                Task { await viewModel.upvote() }
            } label: {
                Image(systemName: viewModel.myVote == 1 ? "arrow.up.circle.fill" : "arrow.up.circle")
                    .foregroundColor(viewModel.myVote == 1 ? .orange : .gray)
            }

            Text(viewModel.score.description)
                .font(.caption)
                .monospacedDigit()

            Button {
                Task { await viewModel.downvote() }
            } label: {
                Image(systemName: viewModel.myVote == -1 ? "arrow.down.circle.fill" : "arrow.down.circle")
                    .foregroundColor(viewModel.myVote == -1 ? .blue : .gray)
            }
        }
    }
}
```

---

## プロジェクト評価

| 観点 | 評価 | 理由 |
|------|------|------|
| **コード品質** | ★★★★★ | クリーンなSwiftUI |
| **ドキュメント** | ★★★★☆ | 貢献ガイド充実 |
| **アクティビティ** | ★★★★★ | 活発な開発 |
| **学習価値** | ★★★★★ | フェデレーションアプリの参考 |
| **実用性** | ★★★★☆ | 実用的なLemmyクライアント |

---

## Markdown表示

```swift
// swift-markdown-ui 統合
import MarkdownUI

struct PostContentView: View {
    let content: String

    var body: some View {
        Markdown(content)
            .markdownTheme(.gitHub)
            .markdownCodeSyntaxHighlighter(.splash(theme: .sunset(withFont: .init(size: 14))))
    }
}
```

---

## 開発者向け情報

### ビルド手順

```bash
git clone https://github.com/mlemgroup/mlem
cd mlem
open Mlem.xcodeproj
```

### 貢献

- 活発なコミュニティ
- Good First Issue ラベル
- Discord サーバー

---

## まとめ

Mlemは、SwiftUIで構築されたフェデレーション対応ソーシャルメディアクライアントの優れた実装例。無限スクロール、コメントスレッド、投票システムなど、Redditライクなアプリのパターンを学べる。

**推奨対象**:
- SwiftUIで複雑なUIを実装したい開発者
- フェデレーションAPIを統合したい人
- ソーシャルメディアアプリを開発したい人
- Markdownレンダリングを実装したい人
