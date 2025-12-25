# MovieSwiftUI

**スター**: 6,524 ⭐
**リポジトリ**: https://github.com/Dimillian/MovieSwiftUI
**ライセンス**: Apache-2.0

---

## 概要

MovieSwiftUIは、Thomas Ricouard（@Dimillian）によるSwiftUIデモアプリ。The Movie Database (TMDB) APIを使用し、SwiftUIの機能を包括的にデモンストレーション。SwiftUI学習の定番教材。

---

## 主要機能

| 機能 | 説明 |
|------|------|
| 映画検索 | TMDB API連携 |
| 映画詳細 | キャスト、レビュー |
| お気に入り | ローカル保存 |
| ディスカバー | ジャンル別探索 |
| カスタムリスト | 映画リスト作成 |
| マルチプラットフォーム | iOS/iPadOS/macOS |

---

## 技術スタック

| 項目 | 技術 |
|------|------|
| 言語 | Swift |
| UI | SwiftUI (100%) |
| アーキテクチャ | Redux-like + Combine |
| ネットワーク | URLSession + Combine |
| 状態管理 | FluxStore |
| 画像 | Nuke |

---

## アーキテクチャ

```
MovieSwiftUI/
├── MovieSwift/              # メインアプリ
│   ├── views/               # SwiftUI ビュー
│   │   ├── movies/          # 映画関連
│   │   ├── people/          # 俳優関連
│   │   └── shared/          # 共通コンポーネント
│   ├── flux/                # Flux アーキテクチャ
│   │   ├── actions/         # アクション
│   │   ├── reducers/        # リデューサー
│   │   └── state/           # 状態
│   └── services/            # API サービス
└── Packages/                # SPM パッケージ
```

---

## 学習価値

### SwiftUI + Flux パターン

| 分野 | 学習ポイント |
|------|-------------|
| **SwiftUI** | 全機能デモ |
| **Flux/Redux** | 単方向データフロー |
| **Combine** | リアクティブ処理 |
| **マルチプラットフォーム** | 適応型レイアウト |
| **API連携** | REST API統合 |

### コードパターン

**Flux Store**:
```swift
// 単方向データフロー
final class Store<State: FluxState>: ObservableObject {
    @Published private(set) var state: State

    private let reducer: Reducer<State>
    private var cancellables = Set<AnyCancellable>()

    init(state: State, reducer: @escaping Reducer<State>) {
        self.state = state
        self.reducer = reducer
    }

    func dispatch(action: Action) {
        DispatchQueue.main.async {
            self.state = self.reducer(self.state, action)
        }
    }
}

// 使用例
struct AppState: FluxState {
    var moviesState = MoviesState()
    var peopleState = PeopleState()
}

let store = Store<AppState>(
    state: AppState(),
    reducer: appStateReducer
)
```

**Reducer**:
```swift
// 状態更新ロジック
func moviesStateReducer(state: MoviesState, action: Action) -> MoviesState {
    var state = state

    switch action {
    case let action as MoviesActions.SetMovies:
        state.movies[action.category] = action.movies
    case let action as MoviesActions.SetDetail:
        state.movieDetails[action.movie] = action.detail
    case let action as MoviesActions.AddToWishlist:
        state.wishlist.insert(action.movie)
    default:
        break
    }

    return state
}
```

**SwiftUI View + Store**:
```swift
// Store を使用したビュー
struct MovieListView: View {
    @EnvironmentObject var store: Store<AppState>
    let category: MovieCategory

    var movies: [Movie] {
        store.state.moviesState.movies[category] ?? []
    }

    var body: some View {
        List(movies) { movie in
            NavigationLink(destination: MovieDetailView(movie: movie)) {
                MovieRow(movie: movie)
            }
        }
        .onAppear {
            store.dispatch(action: MoviesActions.FetchMovies(category: category))
        }
    }
}
```

**API Service with Combine**:
```swift
// Combine ベースの API クライアント
class APIService {
    private let baseURL = "https://api.themoviedb.org/3"

    func fetch<T: Decodable>(endpoint: Endpoint) -> AnyPublisher<T, Error> {
        let url = URL(string: "\(baseURL)/\(endpoint.path)")!
        var request = URLRequest(url: url)
        request.addValue(apiKey, forHTTPHeaderField: "Authorization")

        return URLSession.shared.dataTaskPublisher(for: request)
            .map(\.data)
            .decode(type: T.self, decoder: JSONDecoder())
            .receive(on: DispatchQueue.main)
            .eraseToAnyPublisher()
    }
}
```

---

## プロジェクト評価

| 観点 | 評価 | 理由 |
|------|------|------|
| **コード品質** | ★★★★★ | SwiftUIベストプラクティス |
| **ドキュメント** | ★★★★☆ | コードが自己文書化 |
| **アクティビティ** | ★★★★☆ | 定期的な更新 |
| **学習価値** | ★★★★★ | SwiftUI学習の決定版 |
| **実用性** | ★★★★☆ | デモアプリとして秀逸 |

---

## SwiftUI機能デモ

### 含まれる機能

| 機能 | 実装例 |
|------|--------|
| NavigationStack | 映画詳細遷移 |
| List | 映画リスト |
| LazyVGrid | グリッドレイアウト |
| Sheet | モーダル表示 |
| SearchBar | 検索UI |
| AsyncImage | 画像読み込み |
| Animation | カスタムアニメーション |

### マルチプラットフォーム

```swift
struct MovieDetailView: View {
    let movie: Movie

    var body: some View {
        #if os(iOS)
        iOSDetailView(movie: movie)
        #elseif os(macOS)
        macOSDetailView(movie: movie)
        #endif
    }
}
```

---

## 開発者向け情報

### ビルド手順

```bash
git clone https://github.com/Dimillian/MovieSwiftUI
cd MovieSwiftUI
# TMDB APIキーを設定
open MovieSwift.xcodeproj
```

### API キー設定

1. TMDB でアカウント作成
2. APIキー取得
3. プロジェクトに設定

---

## まとめ

MovieSwiftUIは、SwiftUIの機能を包括的に学べる教科書的プロジェクト。Flux/Reduxパターン、Combine、マルチプラットフォーム対応など、実践的なSwiftUI開発を網羅。Ice Cubesと同じ作者による高品質なコード。

**推奨対象**:
- SwiftUIを体系的に学びたい開発者
- Flux/Reduxパターンを理解したい人
- Combine + SwiftUI連携を学びたい人
- API連携アプリの設計を参考にしたい人
