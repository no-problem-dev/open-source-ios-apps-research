# Ice Cubes

**スター**: 6,748 ⭐
**リポジトリ**: https://github.com/Dimillian/IceCubesApp
**ライセンス**: Apache-2.0

---

## 概要

Ice Cubesは、SwiftUIで構築されたMastodonクライアント。Thomas Ricouard（@Dimillian）による作品で、モダンSwiftUI開発のベストプラクティスを体現。iOS、iPadOS、macOS、visionOSに対応。

---

## 主要機能

| 機能 | 説明 |
|------|------|
| タイムライン | ホーム、ローカル、連合 |
| 投稿 | テキスト、画像、投票 |
| 通知 | プッシュ通知対応 |
| 検索 | ユーザー、ハッシュタグ |
| マルチアカウント | 複数インスタンス |
| テーマ | カスタムテーマ |

---

## 技術スタック

| 項目 | 技術 |
|------|------|
| 言語 | Swift |
| UI | SwiftUI (100%) |
| アーキテクチャ | MVVM + @Observable |
| ネットワーク | URLSession + async/await |
| ストレージ | SwiftData |
| 通知 | APNs |

---

## アーキテクチャ

```
IceCubesApp/
├── IceCubes/            # メインアプリ
│   ├── App/             # アプリエントリ
│   └── Env/             # 環境オブジェクト
├── Packages/
│   ├── Account/         # アカウント機能
│   ├── Timeline/        # タイムライン
│   ├── Status/          # 投稿
│   ├── Network/         # APIクライアント
│   ├── Models/          # データモデル
│   └── DesignSystem/    # UIコンポーネント
└── IceCubesNotifications/ # 通知拡張
```

---

## 学習価値

### モダンSwiftUIの教科書

| 分野 | 学習ポイント |
|------|-------------|
| **@Observable** | iOS 17+新API |
| **SwiftData** | 永続化 |
| **Navigation** | NavigationStack |
| **SPM** | マルチパッケージ構成 |
| **マルチプラットフォーム** | iOS/macOS/visionOS |

### コードパターン

**@Observable ViewModel**:
```swift
@Observable
class TimelineViewModel {
    var statuses: [Status] = []
    var isLoading = false
    var error: Error?

    private let client: Client

    init(client: Client) {
        self.client = client
    }

    func fetchTimeline() async {
        isLoading = true
        defer { isLoading = false }

        do {
            statuses = try await client.get(endpoint: Timelines.home)
        } catch {
            self.error = error
        }
    }
}
```

**SwiftUI View**:
```swift
struct TimelineView: View {
    @State private var viewModel: TimelineViewModel

    init(client: Client) {
        _viewModel = State(initialValue: TimelineViewModel(client: client))
    }

    var body: some View {
        List {
            ForEach(viewModel.statuses) { status in
                StatusRowView(status: status)
            }
        }
        .refreshable {
            await viewModel.fetchTimeline()
        }
        .task {
            await viewModel.fetchTimeline()
        }
    }
}
```

**デザインシステム**:
```swift
// DesignSystem パッケージ
public struct Theme: Sendable {
    public let primaryColor: Color
    public let secondaryColor: Color
    public let tintColor: Color

    public static let `default` = Theme(
        primaryColor: .primary,
        secondaryColor: .secondary,
        tintColor: .accentColor
    )
}

public struct StatusRowView: View {
    let status: Status
    @Environment(Theme.self) private var theme

    public var body: some View {
        VStack(alignment: .leading) {
            Text(status.account.displayName)
                .foregroundStyle(theme.primaryColor)
            Text(status.content)
        }
    }
}
```

---

## SPMマルチパッケージ構成

### パッケージ分割

| パッケージ | 役割 |
|-----------|------|
| Models | Codableモデル |
| Network | APIクライアント |
| Account | アカウント管理 |
| Timeline | タイムライン表示 |
| Status | 投稿関連 |
| DesignSystem | UI部品 |

### 依存関係

```swift
// Package.swift
let package = Package(
    name: "Timeline",
    dependencies: [
        .package(path: "../Models"),
        .package(path: "../Network"),
        .package(path: "../DesignSystem"),
    ],
    targets: [
        .target(
            name: "Timeline",
            dependencies: ["Models", "Network", "DesignSystem"]
        )
    ]
)
```

---

## プロジェクト評価

| 観点 | 評価 | 理由 |
|------|------|------|
| **コード品質** | ★★★★★ | 最新SwiftUIベストプラクティス |
| **ドキュメント** | ★★★★☆ | コードが読みやすい |
| **アクティビティ** | ★★★★★ | 非常に活発 |
| **学習価値** | ★★★★★ | SwiftUI学習の最良教材 |
| **実用性** | ★★★★★ | App Store公開中 |

---

## visionOS対応

```swift
#if os(visionOS)
struct TimelineView: View {
    var body: some View {
        NavigationSplitView {
            SidebarView()
        } detail: {
            TimelineDetailView()
        }
        .ornament(attachmentAnchor: .scene(.bottom)) {
            ComposeButton()
        }
    }
}
#endif
```

---

## 開発者向け情報

### ビルド手順

```bash
git clone https://github.com/Dimillian/IceCubesApp
cd IceCubesApp
open IceCubes.xcodeproj
```

### 貢献ポイント

- 翻訳（Localizable.strings）
- 新機能提案
- バグ修正

---

## まとめ

Ice Cubesは、2024年以降のSwiftUI開発の模範。@Observable、SwiftData、NavigationStack、SPMマルチパッケージなど、最新のSwift/SwiftUI機能を網羅。SwiftUI学習者必見のプロジェクト。

**推奨対象**:
- SwiftUIを深く学びたい開発者
- iOS 17+の新機能を学びたい人
- SPMでのモジュール化を参考にしたい人
- マルチプラットフォーム対応を学びたい人

