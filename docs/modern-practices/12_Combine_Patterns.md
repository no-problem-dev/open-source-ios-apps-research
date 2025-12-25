# Combine 活用パターン集

**参考プロジェクト**:
- Kickstarter iOS (ReactiveSwift → Combine移行参考)
- Various open source apps

**実装参考度**: ⭐⭐⭐⭐☆

---

## なぜ重要か

CombineはApple純正のリアクティブフレームワーク。async/awaitと併用することで、複雑な非同期処理を宣言的に記述できる。

| 学習ポイント | 重要度 | 理由 |
|-------------|--------|------|
| **Publisher/Subscriber** | ⭐⭐⭐⭐⭐ | 基本概念 |
| **Operator** | ⭐⭐⭐⭐⭐ | データ変換 |
| **@Published** | ⭐⭐⭐⭐⭐ | SwiftUI連携 |
| **Subject** | ⭐⭐⭐⭐☆ | イベント発行 |
| **Scheduler** | ⭐⭐⭐☆☆ | スレッド制御 |

---

## 1. 基本パターン

### Publisher と Subscriber

```swift
import Combine

class SearchViewModel: ObservableObject {
    @Published var searchText = ""
    @Published private(set) var results: [SearchResult] = []
    @Published private(set) var isSearching = false

    private var cancellables = Set<AnyCancellable>()
    private let searchService: SearchService

    init(searchService: SearchService = SearchServiceImpl()) {
        self.searchService = searchService
        setupBindings()
    }

    private func setupBindings() {
        $searchText
            .debounce(for: .milliseconds(300), scheduler: RunLoop.main)
            .removeDuplicates()
            .filter { !$0.isEmpty }
            .handleEvents(receiveOutput: { [weak self] _ in
                self?.isSearching = true
            })
            .flatMap { [searchService] query in
                searchService.search(query: query)
                    .catch { _ in Just([]) }
            }
            .receive(on: DispatchQueue.main)
            .sink { [weak self] results in
                self?.results = results
                self?.isSearching = false
            }
            .store(in: &cancellables)
    }
}
```

### Subject

```swift
class EventBus {
    static let shared = EventBus()

    // PassthroughSubject: 値を保持しない
    let userLoggedIn = PassthroughSubject<User, Never>()
    let userLoggedOut = PassthroughSubject<Void, Never>()

    // CurrentValueSubject: 最新値を保持
    let connectionStatus = CurrentValueSubject<ConnectionStatus, Never>(.disconnected)

    private init() {}
}

// 使用側
class ProfileViewModel: ObservableObject {
    @Published var user: User?
    private var cancellables = Set<AnyCancellable>()

    init() {
        EventBus.shared.userLoggedIn
            .sink { [weak self] user in
                self?.user = user
            }
            .store(in: &cancellables)

        EventBus.shared.userLoggedOut
            .sink { [weak self] _ in
                self?.user = nil
            }
            .store(in: &cancellables)
    }
}
```

---

## 2. よく使うOperator

### 変換系

```swift
// map: 値を変換
$text
    .map { $0.uppercased() }
    .sink { print($0) }

// compactMap: nilを除外
$optionalValue
    .compactMap { $0 }
    .sink { print($0) }

// flatMap: Publisherを平坦化
$userId
    .flatMap { userId in
        userService.fetchUser(id: userId)
    }
    .sink { user in print(user) }

// scan: 累積
$score
    .scan(0) { total, score in total + score }
    .sink { print("Total: \($0)") }
```

### フィルタリング系

```swift
// filter: 条件に合う値のみ
$text
    .filter { $0.count >= 3 }
    .sink { print($0) }

// removeDuplicates: 重複除去
$value
    .removeDuplicates()
    .sink { print($0) }

// first: 最初の1つ
$events
    .first()
    .sink { print($0) }

// dropFirst: 最初のn個をスキップ
$values
    .dropFirst(2)
    .sink { print($0) }
```

### 時間制御系

```swift
// debounce: 入力が落ち着くまで待つ
$searchText
    .debounce(for: .milliseconds(300), scheduler: RunLoop.main)
    .sink { self.search($0) }

// throttle: 一定間隔で最新値を取得
$scrollPosition
    .throttle(for: .milliseconds(100), scheduler: RunLoop.main, latest: true)
    .sink { self.updateUI($0) }

// delay: 遅延
$value
    .delay(for: .seconds(1), scheduler: RunLoop.main)
    .sink { print($0) }

// timeout: タイムアウト
networkRequest
    .timeout(.seconds(10), scheduler: RunLoop.main)
    .sink(
        receiveCompletion: { completion in
            if case .failure = completion {
                print("Timeout!")
            }
        },
        receiveValue: { print($0) }
    )
```

### 結合系

```swift
// combineLatest: 最新値を結合
Publishers.CombineLatest($username, $password)
    .map { username, password in
        !username.isEmpty && password.count >= 8
    }
    .assign(to: &$isFormValid)

// merge: 複数のPublisherをマージ
Publishers.Merge(
    NotificationCenter.default.publisher(for: .userLoggedIn),
    NotificationCenter.default.publisher(for: .userUpdated)
)
.sink { notification in
    self.refreshUser()
}

// zip: ペアで結合
Publishers.Zip(firstRequest, secondRequest)
    .sink { first, second in
        print("Both completed: \(first), \(second)")
    }
```

---

## 3. エラーハンドリング

### catch

```swift
networkService.fetchData()
    .catch { error -> Just<Data> in
        print("Error: \(error)")
        return Just(Data())  // フォールバック値
    }
    .sink { data in
        process(data)
    }
```

### retry

```swift
networkService.fetchData()
    .retry(3)  // 3回リトライ
    .catch { _ in Just(Data()) }
    .sink { data in
        process(data)
    }
```

### replaceError

```swift
networkService.fetchData()
    .replaceError(with: defaultData)
    .sink { data in
        process(data)
    }
```

### mapError

```swift
networkService.fetchData()
    .mapError { networkError -> AppError in
        switch networkError {
        case .timeout: return .networkTimeout
        case .notFound: return .dataNotFound
        default: return .unknown
        }
    }
    .sink(
        receiveCompletion: { completion in
            if case .failure(let appError) = completion {
                handle(appError)
            }
        },
        receiveValue: { data in
            process(data)
        }
    )
```

---

## 4. SwiftUI連携

### @Published + View

```swift
class FormViewModel: ObservableObject {
    @Published var email = ""
    @Published var password = ""
    @Published private(set) var isValid = false
    @Published private(set) var emailError: String?

    private var cancellables = Set<AnyCancellable>()

    init() {
        // バリデーション
        Publishers.CombineLatest($email, $password)
            .map { email, password in
                self.validateEmail(email) && password.count >= 8
            }
            .assign(to: &$isValid)

        // メールバリデーション
        $email
            .debounce(for: .milliseconds(500), scheduler: RunLoop.main)
            .map { email -> String? in
                guard !email.isEmpty else { return nil }
                return self.validateEmail(email) ? nil : "Invalid email"
            }
            .assign(to: &$emailError)
    }

    private func validateEmail(_ email: String) -> Bool {
        email.contains("@") && email.contains(".")
    }
}

struct FormView: View {
    @StateObject private var viewModel = FormViewModel()

    var body: some View {
        Form {
            TextField("Email", text: $viewModel.email)
            if let error = viewModel.emailError {
                Text(error).foregroundColor(.red)
            }

            SecureField("Password", text: $viewModel.password)

            Button("Submit") {
                submit()
            }
            .disabled(!viewModel.isValid)
        }
    }
}
```

### assign(to:)

```swift
class CounterViewModel: ObservableObject {
    @Published var count = 0

    private var cancellables = Set<AnyCancellable>()

    init() {
        // 古い方法: store(in:) が必要
        Timer.publish(every: 1, on: .main, in: .common)
            .autoconnect()
            .map { _ in self.count + 1 }
            .sink { [weak self] in self?.count = $0 }
            .store(in: &cancellables)

        // 新しい方法: メモリリークなし
        Timer.publish(every: 1, on: .main, in: .common)
            .autoconnect()
            .map { [weak self] _ in (self?.count ?? 0) + 1 }
            .assign(to: &$count)  // & で参照渡し
    }
}
```

---

## 5. async/await との連携

### Publisherからasync

```swift
extension Publisher {
    func asyncFirst() async throws -> Output {
        try await withCheckedThrowingContinuation { continuation in
            var cancellable: AnyCancellable?
            cancellable = first()
                .sink(
                    receiveCompletion: { completion in
                        switch completion {
                        case .finished:
                            break
                        case .failure(let error):
                            continuation.resume(throwing: error)
                        }
                        cancellable?.cancel()
                    },
                    receiveValue: { value in
                        continuation.resume(returning: value)
                    }
                )
        }
    }
}

// 使用
let result = try await networkPublisher.asyncFirst()
```

### AsyncSequenceからPublisher

```swift
extension Publisher where Failure == Never {
    var values: AsyncPublisher<Self> {
        AsyncPublisher(self)
    }
}

// 使用
for await value in $searchText.values {
    print("Search: \(value)")
}
```

---

## まとめ：Combine vs async/await

| 用途 | Combine | async/await |
|------|---------|-------------|
| **単発リクエスト** | ⭕ | ◎ より簡単 |
| **ストリーム処理** | ◎ 最適 | ⭕ AsyncSequence |
| **UI バインディング** | ◎ @Published | △ |
| **複雑な変換** | ◎ Operator豊富 | △ |
| **キャンセル** | ⭕ AnyCancellable | ◎ Task |
| **エラーハンドリング** | ⭕ | ◎ try/catch |

### 使い分けガイドライン

```
単発の非同期処理 → async/await
├─ API呼び出し
├─ ファイル読み込み
└─ データベースクエリ

継続的なストリーム → Combine
├─ UIイベント（テキスト入力、スクロール）
├─ タイマー
├─ 複数値の結合（CombineLatest）
└─ リアルタイム更新（WebSocket）

SwiftUI状態管理 → @Published + Combine
```
