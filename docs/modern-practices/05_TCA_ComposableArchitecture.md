# TCA (The Composable Architecture) 実践ガイド

**参考プロジェクト**:
- Critical Maps: https://github.com/nickolashkraus/critical-maps
- OnlineStoreTCA: https://github.com/nickolashkraus/OnlineStoreTCA
- Das E-Rezept: https://github.com/gematik/E-Rezept-App-iOS

**実装参考度**: ⭐⭐⭐⭐⭐

---

## なぜ参考になるか

TCAは**Point-Free社**が開発したSwiftUI向けアーキテクチャ。状態管理、副作用処理、テスタビリティに優れる。

| 学習ポイント | 重要度 | 理由 |
|-------------|--------|------|
| **単方向データフロー** | ⭐⭐⭐⭐⭐ | 予測可能な状態管理 |
| **Reducer合成** | ⭐⭐⭐⭐⭐ | 機能の分離と結合 |
| **副作用管理** | ⭐⭐⭐⭐⭐ | Effect によるテスタブルな副作用 |
| **依存性注入** | ⭐⭐⭐⭐⭐ | @Dependency によるDI |
| **テスト容易性** | ⭐⭐⭐⭐⭐ | TestStore による完全テスト |

---

## 1. TCA基本構造

### Feature の4要素

```swift
import ComposableArchitecture

// 1. State - 画面の状態
@ObservableState
struct CounterFeature: Reducer {
    struct State: Equatable {
        var count = 0
        var isLoading = false
        var fact: String?
    }

    // 2. Action - ユーザー操作やシステムイベント
    enum Action {
        case incrementButtonTapped
        case decrementButtonTapped
        case factButtonTapped
        case factResponse(String)
    }

    // 3. Dependency - 外部依存
    @Dependency(\.numberFact) var numberFact

    // 4. Reducer - 状態変更ロジック
    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case .incrementButtonTapped:
                state.count += 1
                return .none

            case .decrementButtonTapped:
                state.count -= 1
                return .none

            case .factButtonTapped:
                state.isLoading = true
                return .run { [count = state.count] send in
                    let fact = try await self.numberFact.fetch(count)
                    await send(.factResponse(fact))
                }

            case .factResponse(let fact):
                state.isLoading = false
                state.fact = fact
                return .none
            }
        }
    }
}
```

### View の実装

```swift
struct CounterView: View {
    let store: StoreOf<CounterFeature>

    var body: some View {
        WithViewStore(store, observe: { $0 }) { viewStore in
            VStack {
                HStack {
                    Button("-") {
                        viewStore.send(.decrementButtonTapped)
                    }
                    Text("\(viewStore.count)")
                        .font(.largeTitle)
                    Button("+") {
                        viewStore.send(.incrementButtonTapped)
                    }
                }

                Button("Get fact") {
                    viewStore.send(.factButtonTapped)
                }
                .disabled(viewStore.isLoading)

                if let fact = viewStore.fact {
                    Text(fact)
                        .padding()
                }
            }
        }
    }
}
```

---

## 2. 機能の合成（Composition）

### 親子関係の構築

```swift
// 子Feature
@Reducer
struct CounterRowFeature {
    struct State: Equatable, Identifiable {
        let id: UUID
        var count = 0
    }

    enum Action {
        case incrementButtonTapped
        case decrementButtonTapped
    }

    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case .incrementButtonTapped:
                state.count += 1
                return .none
            case .decrementButtonTapped:
                state.count -= 1
                return .none
            }
        }
    }
}

// 親Feature
@Reducer
struct CounterListFeature {
    struct State: Equatable {
        var counters: IdentifiedArrayOf<CounterRowFeature.State> = []
    }

    enum Action {
        case addButtonTapped
        case counter(id: CounterRowFeature.State.ID, action: CounterRowFeature.Action)
    }

    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case .addButtonTapped:
                state.counters.append(CounterRowFeature.State(id: UUID()))
                return .none
            case .counter:
                return .none
            }
        }
        // 子Reducerを統合
        .forEach(\.counters, action: \.counter) {
            CounterRowFeature()
        }
    }
}
```

### 親Viewの実装

```swift
struct CounterListView: View {
    let store: StoreOf<CounterListFeature>

    var body: some View {
        List {
            ForEachStore(
                store.scope(
                    state: \.counters,
                    action: \.counter
                )
            ) { rowStore in
                CounterRowView(store: rowStore)
            }
        }
        .toolbar {
            Button("Add") {
                store.send(.addButtonTapped)
            }
        }
    }
}
```

---

## 3. 副作用（Effect）の管理

### 非同期処理

```swift
@Reducer
struct SearchFeature {
    struct State: Equatable {
        var query = ""
        var results: [SearchResult] = []
        var isSearching = false
    }

    enum Action {
        case queryChanged(String)
        case searchResponse([SearchResult])
        case searchFailed
    }

    @Dependency(\.searchClient) var searchClient
    @Dependency(\.continuousClock) var clock

    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case .queryChanged(let query):
                state.query = query

                guard !query.isEmpty else {
                    state.results = []
                    return .cancel(id: CancelID.search)
                }

                state.isSearching = true
                return .run { send in
                    // デバウンス
                    try await clock.sleep(for: .milliseconds(300))
                    let results = try await searchClient.search(query)
                    await send(.searchResponse(results))
                } catch: { error, send in
                    await send(.searchFailed)
                }
                .cancellable(id: CancelID.search, cancelInFlight: true)

            case .searchResponse(let results):
                state.isSearching = false
                state.results = results
                return .none

            case .searchFailed:
                state.isSearching = false
                return .none
            }
        }
    }

    enum CancelID { case search }
}
```

### Effectの種類

```swift
// 1. 何もしない
return .none

// 2. 即座にActionを送信
return .send(.someAction)

// 3. 非同期処理
return .run { send in
    let result = try await apiClient.fetch()
    await send(.response(result))
}

// 4. キャンセル可能な処理
return .run { send in
    // ...
}
.cancellable(id: CancelID.fetch)

// 5. キャンセル
return .cancel(id: CancelID.fetch)

// 6. 複数のEffectを結合
return .merge(
    .send(.action1),
    .run { send in await send(.action2) }
)

// 7. 順次実行
return .concatenate(
    .send(.startLoading),
    .run { send in
        let data = try await fetch()
        await send(.loaded(data))
    }
)
```

---

## 4. 依存性管理（@Dependency）

### 依存性の定義

```swift
// Dependencies/NumberFactClient.swift
struct NumberFactClient {
    var fetch: @Sendable (Int) async throws -> String
}

extension NumberFactClient: DependencyKey {
    static let liveValue = Self(
        fetch: { number in
            let url = URL(string: "http://numbersapi.com/\(number)")!
            let (data, _) = try await URLSession.shared.data(from: url)
            return String(decoding: data, as: UTF8.self)
        }
    )

    static let testValue = Self(
        fetch: { number in
            "\(number) is a great number!"
        }
    )

    static let previewValue = Self(
        fetch: { number in
            "Preview fact for \(number)"
        }
    )
}

extension DependencyValues {
    var numberFact: NumberFactClient {
        get { self[NumberFactClient.self] }
        set { self[NumberFactClient.self] = newValue }
    }
}
```

### Reducerでの使用

```swift
@Reducer
struct FactFeature {
    @Dependency(\.numberFact) var numberFact

    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case .fetchFact:
                return .run { [number = state.number] send in
                    let fact = try await numberFact.fetch(number)
                    await send(.factLoaded(fact))
                }
            // ...
            }
        }
    }
}
```

---

## 5. テスト

### TestStoreによる完全テスト

```swift
import XCTest
@testable import MyFeature
import ComposableArchitecture

final class CounterFeatureTests: XCTestCase {
    @MainActor
    func testIncrement() async {
        let store = TestStore(
            initialState: CounterFeature.State()
        ) {
            CounterFeature()
        }

        await store.send(.incrementButtonTapped) {
            $0.count = 1
        }
        await store.send(.incrementButtonTapped) {
            $0.count = 2
        }
    }

    @MainActor
    func testFetchFact() async {
        let store = TestStore(
            initialState: CounterFeature.State(count: 42)
        ) {
            CounterFeature()
        } withDependencies: {
            // テスト用の依存性を注入
            $0.numberFact.fetch = { number in
                "\(number) is the answer"
            }
        }

        await store.send(.factButtonTapped) {
            $0.isLoading = true
        }

        await store.receive(.factResponse("42 is the answer")) {
            $0.isLoading = false
            $0.fact = "42 is the answer"
        }
    }

    @MainActor
    func testDebounce() async {
        let clock = TestClock()

        let store = TestStore(
            initialState: SearchFeature.State()
        ) {
            SearchFeature()
        } withDependencies: {
            $0.continuousClock = clock
            $0.searchClient.search = { _ in [] }
        }

        await store.send(.queryChanged("a")) {
            $0.query = "a"
            $0.isSearching = true
        }

        // デバウンス時間を進める
        await clock.advance(by: .milliseconds(300))

        await store.receive(.searchResponse([])) {
            $0.isSearching = false
        }
    }
}
```

---

## 6. ナビゲーション

### Tree-based Navigation

```swift
@Reducer
struct AppFeature {
    struct State: Equatable {
        var path = StackState<Path.State>()
    }

    enum Action {
        case path(StackAction<Path.State, Path.Action>)
        case goToDetailTapped(Item)
    }

    @Reducer
    struct Path {
        enum State: Equatable {
            case detail(DetailFeature.State)
            case edit(EditFeature.State)
        }

        enum Action {
            case detail(DetailFeature.Action)
            case edit(EditFeature.Action)
        }

        var body: some ReducerOf<Self> {
            Scope(state: \.detail, action: \.detail) {
                DetailFeature()
            }
            Scope(state: \.edit, action: \.edit) {
                EditFeature()
            }
        }
    }

    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case .goToDetailTapped(let item):
                state.path.append(.detail(DetailFeature.State(item: item)))
                return .none
            case .path:
                return .none
            }
        }
        .forEach(\.path, action: \.path) {
            Path()
        }
    }
}
```

### View実装

```swift
struct AppView: View {
    let store: StoreOf<AppFeature>

    var body: some View {
        NavigationStack(
            path: store.scope(state: \.path, action: \.path)
        ) {
            HomeView(store: store)
        } destination: { store in
            switch store.state {
            case .detail:
                CaseLet(
                    /AppFeature.Path.State.detail,
                    action: AppFeature.Path.Action.detail
                ) { store in
                    DetailView(store: store)
                }
            case .edit:
                CaseLet(
                    /AppFeature.Path.State.edit,
                    action: AppFeature.Path.Action.edit
                ) { store in
                    EditView(store: store)
                }
            }
        }
    }
}
```

---

## まとめ：TCAから学ぶべきこと

| 概念 | 学習ポイント | 優先度 |
|------|-------------|--------|
| **単方向データフロー** | State → View → Action → Reducer | 🔴 必須 |
| **Reducer合成** | .forEach, Scope による機能分割 | 🔴 必須 |
| **Effect** | 副作用の完全な制御とテスト | 🔴 必須 |
| **@Dependency** | テスト可能な依存性注入 | 🟡 推奨 |
| **TestStore** | 状態変化の厳密なテスト | 🟡 推奨 |

### TCAを使うべきケース

- 複雑な状態管理が必要なアプリ
- テストを最優先するプロジェクト
- 予測可能な動作が必要
- チーム開発で一貫性が重要

### TCAを使わない方がいいケース

- シンプルなアプリ（学習コストが高い）
- プロトタイプや短期プロジェクト
- SwiftUIに不慣れなチーム
