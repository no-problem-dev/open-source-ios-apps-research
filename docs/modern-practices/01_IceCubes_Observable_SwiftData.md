# Ice Cubes - iOS 17+ モダンSwiftUIの教科書

**リポジトリ**: https://github.com/Dimillian/IceCubesApp
**作者**: Thomas Ricouard (@Dimillian)
**実装参考度**: ⭐⭐⭐⭐⭐

---

## なぜ参考になるか

Ice Cubesは**iOS 17+の最新機能を実際のプロダクションアプリで活用**している最良の例。

| 学習ポイント | 重要度 | 理由 |
|-------------|--------|------|
| **@Observable** | ⭐⭐⭐⭐⭐ | Observation frameworkの実践的な使い方 |
| **SwiftData** | ⭐⭐⭐⭐⭐ | Core Data代替の実装パターン |
| **SPMマルチパッケージ** | ⭐⭐⭐⭐⭐ | モジュール分割の設計 |
| **async/await** | ⭐⭐⭐⭐☆ | Swift Concurrency活用 |
| **NavigationStack** | ⭐⭐⭐⭐☆ | 型安全ナビゲーション |

---

## 1. @Observable パターン

### 従来の ObservableObject vs @Observable

**❌ 従来（iOS 16以前）**:
```swift
class TimelineViewModel: ObservableObject {
    @Published var statuses: [Status] = []
    @Published var isLoading = false
    @Published var error: Error?
}

struct TimelineView: View {
    @StateObject var viewModel = TimelineViewModel()
    // 全プロパティの変更で再描画される
}
```

**✅ Ice Cubesのアプローチ（iOS 17+）**:
```swift
@Observable
class TimelineViewModel {
    var statuses: [Status] = []
    var isLoading = false
    var error: Error?

    // @Publishedが不要、アクセスされたプロパティのみ追跡
}

struct TimelineView: View {
    @State var viewModel = TimelineViewModel()
    // statusesが変更されたときのみ、statusesを使う部分だけ再描画
}
```

### Ice Cubesでの実装例

```swift
// Packages/Status/Sources/Status/StatusDataController.swift
@Observable
public class StatusDataController {
    public var isBookmarked: Bool
    public var isFavorited: Bool
    public var isReblogged: Bool
    public var reblogsCount: Int
    public var favoritesCount: Int
    public var repliesCount: Int

    private let status: Status
    private let client: Client

    public init(account: Account, status: Status, client: Client) {
        self.status = status
        self.client = client
        // 初期値設定
        self.isBookmarked = status.bookmarked == true
        self.isFavorited = status.favourited == true
        // ...
    }

    public func toggleFavorite() async {
        // 楽観的更新
        isFavorited.toggle()
        favoritesCount += isFavorited ? 1 : -1

        do {
            let endpoint = isFavorited
                ? Statuses.favorite(id: status.id)
                : Statuses.unfavorite(id: status.id)
            _ = try await client.post(endpoint: endpoint)
        } catch {
            // ロールバック
            isFavorited.toggle()
            favoritesCount += isFavorited ? 1 : -1
        }
    }
}
```

### 取り入れ方

1. `@ObservableObject` → `@Observable` に置き換え
2. `@Published` を削除
3. `@StateObject` → `@State` に変更
4. 必要に応じて `@ObservationIgnored` で追跡除外

---

## 2. SwiftData 実装パターン

### モデル定義

```swift
// Packages/Models/Sources/Models/SwiftData/Draft.swift
import SwiftData

@Model
public class Draft {
    public var content: String
    public var createdAt: Date
    @Relationship(deleteRule: .cascade)
    public var attachments: [DraftAttachment]

    public init(content: String) {
        self.content = content
        self.createdAt = Date()
        self.attachments = []
    }
}

@Model
public class DraftAttachment {
    public var data: Data
    public var type: String

    public init(data: Data, type: String) {
        self.data = data
        self.type = type
    }
}
```

### コンテナ設定

```swift
// App起動時
@main
struct IceCubesApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(for: [Draft.self, DraftAttachment.self])
    }
}
```

### @Query によるフェッチ

```swift
struct DraftsView: View {
    @Query(sort: \Draft.createdAt, order: .reverse)
    private var drafts: [Draft]

    @Environment(\.modelContext) private var modelContext

    var body: some View {
        List(drafts) { draft in
            DraftRow(draft: draft)
        }
        .onDelete { indexSet in
            for index in indexSet {
                modelContext.delete(drafts[index])
            }
        }
    }
}
```

### 取り入れ方

1. Core Data の `NSManagedObject` → `@Model` に置き換え
2. `@FetchRequest` → `@Query` に置き換え
3. `NSManagedObjectContext` → `ModelContext` に置き換え
4. マイグレーションは `Schema` と `SchemaMigrationPlan` で管理

---

## 3. SPMマルチパッケージアーキテクチャ

### Ice Cubesのパッケージ構成

```
IceCubesApp/
├── IceCubesApp/              # メインアプリ（薄いシェル）
├── Packages/
│   ├── Account/              # アカウント機能
│   ├── AppAccount/           # マルチアカウント管理
│   ├── Conversations/        # DM機能
│   ├── DesignSystem/         # UIコンポーネント、テーマ
│   ├── Env/                  # 環境変数、設定
│   ├── Explore/              # 探索機能
│   ├── Lists/                # リスト機能
│   ├── Models/               # データモデル（SwiftData含む）
│   ├── Network/              # APIクライアント
│   ├── Notifications/        # 通知機能
│   ├── Status/               # 投稿表示・作成
│   └── Timeline/             # タイムライン
```

### Package.swift の構造

```swift
// Packages/Network/Package.swift
let package = Package(
    name: "Network",
    platforms: [.iOS(.v17), .visionOS(.v1)],
    products: [
        .library(name: "Network", targets: ["Network"]),
    ],
    dependencies: [
        .package(name: "Models", path: "../Models"),
    ],
    targets: [
        .target(
            name: "Network",
            dependencies: ["Models"]
        ),
    ]
)
```

### 依存関係のルール

```
    ┌─────────────────┐
    │   IceCubesApp   │  ← メインアプリ
    └────────┬────────┘
             │
    ┌────────┴────────┐
    │  Feature Layer  │  ← Timeline, Status, Account等
    └────────┬────────┘
             │
    ┌────────┴────────┐
    │   Core Layer    │  ← Network, DesignSystem
    └────────┬────────┘
             │
    ┌────────┴────────┐
    │   Foundation    │  ← Models, Env
    └─────────────────┘
```

### 取り入れ方

1. 機能単位でパッケージを分割
2. 依存方向は「上→下」のみ
3. 循環依存を避けるためにプロトコルを活用
4. 各パッケージは独立してテスト可能に

---

## 4. EnvironmentKey パターン

### 依存性注入の実装

```swift
// Packages/Env/Sources/Env/CurrentAccount.swift
@Observable
public class CurrentAccount {
    public var account: Account?
    public var lists: [List] = []
    public var tags: [Tag] = []

    private let client: Client

    public func fetchCurrentAccount() async {
        do {
            account = try await client.get(endpoint: Accounts.verifyCredentials)
        } catch {
            account = nil
        }
    }
}

// Environment Key
private struct CurrentAccountKey: EnvironmentKey {
    static let defaultValue: CurrentAccount = CurrentAccount()
}

extension EnvironmentValues {
    public var currentAccount: CurrentAccount {
        get { self[CurrentAccountKey.self] }
        set { self[CurrentAccountKey.self] = newValue }
    }
}
```

### 使用方法

```swift
// 親ビュー
ContentView()
    .environment(currentAccount)

// 子ビュー
struct ProfileView: View {
    @Environment(\.currentAccount) var currentAccount

    var body: some View {
        if let account = currentAccount.account {
            Text(account.displayName)
        }
    }
}
```

---

## 5. API クライアント設計

### エンドポイント定義

```swift
// Packages/Network/Sources/Network/Endpoint/Statuses.swift
public enum Statuses: Endpoint {
    case status(id: String)
    case context(id: String)
    case favorite(id: String)
    case unfavorite(id: String)
    case reblog(id: String)
    case unreblog(id: String)

    public var path: String {
        switch self {
        case .status(let id): "/api/v1/statuses/\(id)"
        case .context(let id): "/api/v1/statuses/\(id)/context"
        case .favorite(let id): "/api/v1/statuses/\(id)/favourite"
        case .unfavorite(let id): "/api/v1/statuses/\(id)/unfavourite"
        case .reblog(let id): "/api/v1/statuses/\(id)/reblog"
        case .unreblog(let id): "/api/v1/statuses/\(id)/unreblog"
        }
    }

    public var method: HTTPMethod {
        switch self {
        case .status, .context: .get
        case .favorite, .unfavorite, .reblog, .unreblog: .post
        }
    }
}
```

### クライアント実装

```swift
// Packages/Network/Sources/Network/Client.swift
@Observable
public class Client {
    public let server: String
    public var oauthToken: OauthToken?

    private let session: URLSession

    public func get<T: Decodable>(endpoint: Endpoint) async throws -> T {
        let request = makeRequest(endpoint: endpoint)
        let (data, response) = try await session.data(for: request)

        guard let httpResponse = response as? HTTPURLResponse,
              (200...299).contains(httpResponse.statusCode) else {
            throw ClientError.invalidResponse
        }

        return try decoder.decode(T.self, from: data)
    }
}
```

---

## まとめ：Ice Cubesから学ぶべきこと

| 技術 | 学習ポイント | 優先度 |
|------|-------------|--------|
| @Observable | Observation framework の実践的使用 | 🔴 必須 |
| SwiftData | Core Data からの移行パターン | 🔴 必須 |
| SPMマルチパッケージ | モジュール分割設計 | 🟡 推奨 |
| EnvironmentKey | 依存性注入パターン | 🟡 推奨 |
| async/await | ネットワーク処理 | 🟢 参考 |

### 即座に取り入れられること

1. `@Observable` を使った ViewModel の実装
2. `@Query` を使った SwiftData フェッチ
3. EnvironmentKey を使った依存性注入
4. 楽観的更新パターン（先にUIを更新、失敗したらロールバック）

### リポジトリを読む順序

1. `Packages/Models/` - データモデルの定義
2. `Packages/Network/` - APIクライアント
3. `Packages/Status/` - 投稿表示の実装
4. `Packages/Timeline/` - タイムライン表示
5. `IceCubesApp/` - アプリのエントリーポイント
