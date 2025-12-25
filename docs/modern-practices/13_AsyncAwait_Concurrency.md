# Swift Concurrency 実践ガイド

**参考プロジェクト**:
- Ice Cubes
- Various modern open source apps

**実装参考度**: ⭐⭐⭐⭐⭐

---

## なぜ重要か

Swift Concurrencyは**async/await、Actor、TaskGroup**など、安全で効率的な並行処理を実現する機能群。iOS 15+で利用可能。

| 学習ポイント | 重要度 | 理由 |
|-------------|--------|------|
| **async/await** | ⭐⭐⭐⭐⭐ | 基本的な非同期処理 |
| **Task** | ⭐⭐⭐⭐⭐ | 非同期処理の起動 |
| **Actor** | ⭐⭐⭐⭐⭐ | データ競合防止 |
| **TaskGroup** | ⭐⭐⭐⭐☆ | 並列処理 |
| **AsyncSequence** | ⭐⭐⭐⭐☆ | 非同期ストリーム |

---

## 1. async/await 基本

### 関数定義

```swift
// 非同期関数
func fetchUser(id: String) async throws -> User {
    let url = URL(string: "https://api.example.com/users/\(id)")!
    let (data, response) = try await URLSession.shared.data(from: url)

    guard let httpResponse = response as? HTTPURLResponse,
          httpResponse.statusCode == 200 else {
        throw APIError.invalidResponse
    }

    return try JSONDecoder().decode(User.self, from: data)
}

// 呼び出し
Task {
    do {
        let user = try await fetchUser(id: "123")
        print(user.name)
    } catch {
        print("Error: \(error)")
    }
}
```

### SwiftUIでの使用

```swift
struct UserProfileView: View {
    @State private var user: User?
    @State private var isLoading = false
    @State private var error: Error?

    let userId: String

    var body: some View {
        Group {
            if isLoading {
                ProgressView()
            } else if let user = user {
                UserContent(user: user)
            } else if let error = error {
                ErrorView(error: error)
            }
        }
        .task {
            await loadUser()
        }
    }

    private func loadUser() async {
        isLoading = true
        defer { isLoading = false }

        do {
            user = try await fetchUser(id: userId)
        } catch {
            self.error = error
        }
    }
}
```

---

## 2. Task

### Task の種類

```swift
// 通常のTask
Task {
    let result = try await someAsyncWork()
    print(result)
}

// デタッチドTask（親Taskと独立）
Task.detached {
    let result = try await heavyWork()
    print(result)
}

// 優先度付きTask
Task(priority: .high) {
    await urgentWork()
}

Task(priority: .background) {
    await backgroundWork()
}
```

### キャンセル

```swift
class DownloadManager {
    private var downloadTask: Task<Data, Error>?

    func startDownload(url: URL) {
        downloadTask = Task {
            try await downloadData(from: url)
        }
    }

    func cancelDownload() {
        downloadTask?.cancel()
    }

    private func downloadData(from url: URL) async throws -> Data {
        var data = Data()

        for try await chunk in url.resourceBytes {
            // キャンセルチェック
            try Task.checkCancellation()

            data.append(contentsOf: chunk)
        }

        return data
    }
}
```

### Task値の取得

```swift
let task = Task {
    try await fetchUser(id: "123")
}

// 後で結果を取得
let user = try await task.value
```

---

## 3. Actor

### 基本的なActor

```swift
actor BankAccount {
    private var balance: Double = 0

    func deposit(_ amount: Double) {
        balance += amount
    }

    func withdraw(_ amount: Double) throws -> Double {
        guard balance >= amount else {
            throw BankError.insufficientFunds
        }
        balance -= amount
        return amount
    }

    func getBalance() -> Double {
        balance
    }
}

// 使用
let account = BankAccount()
await account.deposit(100)
let balance = await account.getBalance()
```

### nonisolated

```swift
actor DataCache {
    private var cache: [String: Data] = [:]

    // isolated: 通常のメソッド
    func store(_ data: Data, for key: String) {
        cache[key] = data
    }

    func retrieve(for key: String) -> Data? {
        cache[key]
    }

    // nonisolated: 同期的にアクセス可能
    nonisolated let cacheDirectory: URL = {
        FileManager.default.urls(for: .cachesDirectory, in: .userDomainMask)[0]
    }()

    nonisolated func cacheFilePath(for key: String) -> URL {
        cacheDirectory.appendingPathComponent(key)
    }
}
```

### @MainActor

```swift
// クラス全体をMainActorに
@MainActor
class ViewModel: ObservableObject {
    @Published var items: [Item] = []
    @Published var isLoading = false

    func loadItems() async {
        isLoading = true
        defer { isLoading = false }

        items = await fetchItems()
    }
}

// メソッド単位
class DataManager {
    @MainActor
    func updateUI(with data: Data) {
        // UIの更新は必ずメインスレッド
    }

    func processData() async {
        let data = await fetchData()

        // メインスレッドに戻る
        await MainActor.run {
            updateUI(with: data)
        }
    }
}
```

---

## 4. TaskGroup

### 並列実行

```swift
func fetchAllUsers(ids: [String]) async throws -> [User] {
    try await withThrowingTaskGroup(of: User.self) { group in
        for id in ids {
            group.addTask {
                try await fetchUser(id: id)
            }
        }

        var users: [User] = []
        for try await user in group {
            users.append(user)
        }
        return users
    }
}
```

### 部分的な成功

```swift
func fetchUsersWithPartialSuccess(ids: [String]) async -> [User] {
    await withTaskGroup(of: User?.self) { group in
        for id in ids {
            group.addTask {
                try? await fetchUser(id: id)
            }
        }

        var users: [User] = []
        for await user in group {
            if let user = user {
                users.append(user)
            }
        }
        return users
    }
}
```

### 並列数の制限

```swift
func downloadImages(urls: [URL]) async throws -> [UIImage] {
    try await withThrowingTaskGroup(of: (Int, UIImage).self) { group in
        let maxConcurrent = 5
        var images: [UIImage?] = Array(repeating: nil, count: urls.count)

        for (index, url) in urls.enumerated() {
            // 並列数を制限
            if index >= maxConcurrent {
                if let (completedIndex, image) = try await group.next() {
                    images[completedIndex] = image
                }
            }

            group.addTask {
                let image = try await downloadImage(from: url)
                return (index, image)
            }
        }

        // 残りを収集
        for try await (index, image) in group {
            images[index] = image
        }

        return images.compactMap { $0 }
    }
}
```

---

## 5. AsyncSequence

### 基本的な使用

```swift
// URLSession.bytes
func streamData(from url: URL) async throws {
    let (bytes, _) = try await URLSession.shared.bytes(from: url)

    for try await byte in bytes {
        process(byte)
    }
}

// NotificationCenter
func observeNotifications() async {
    let notifications = NotificationCenter.default.notifications(
        named: UIApplication.didBecomeActiveNotification
    )

    for await notification in notifications {
        print("App became active: \(notification)")
    }
}
```

### カスタムAsyncSequence

```swift
struct CountdownSequence: AsyncSequence {
    typealias Element = Int

    let start: Int

    struct AsyncIterator: AsyncIteratorProtocol {
        var current: Int

        mutating func next() async -> Int? {
            guard current > 0 else { return nil }
            try? await Task.sleep(nanoseconds: 1_000_000_000)
            defer { current -= 1 }
            return current
        }
    }

    func makeAsyncIterator() -> AsyncIterator {
        AsyncIterator(current: start)
    }
}

// 使用
for await count in CountdownSequence(start: 10) {
    print(count)
}
```

### AsyncStream

```swift
func locationUpdates() -> AsyncStream<CLLocation> {
    AsyncStream { continuation in
        let locationManager = CLLocationManager()
        let delegate = LocationDelegate { location in
            continuation.yield(location)
        }

        continuation.onTermination = { _ in
            locationManager.stopUpdatingLocation()
        }

        locationManager.delegate = delegate
        locationManager.startUpdatingLocation()
    }
}

// 使用
for await location in locationUpdates() {
    print("Location: \(location)")
}
```

---

## 6. Sendable

### Sendable プロトコル

```swift
// 値型は自動的にSendable
struct User: Sendable {
    let id: String
    let name: String
}

// クラスは明示的に@unchecked Sendable
final class DataManager: @unchecked Sendable {
    private let lock = NSLock()
    private var data: [String: Data] = [:]

    func store(_ data: Data, for key: String) {
        lock.lock()
        defer { lock.unlock() }
        self.data[key] = data
    }
}

// Actorは自動的にSendable
actor Cache: Sendable {
    private var storage: [String: Data] = [:]
    // ...
}
```

### @Sendable クロージャ

```swift
// Task内のクロージャは@Sendable
Task {
    // このクロージャは@Sendable
    // 可変なキャプチャは不可
}

// 明示的に@Sendable
func performAsync(_ operation: @Sendable () async -> Void) async {
    await operation()
}
```

---

## まとめ：Swift Concurrency

| 機能 | 用途 | 優先度 |
|------|------|--------|
| **async/await** | 非同期処理の基本 | 🔴 必須 |
| **Task** | 非同期処理の起動・管理 | 🔴 必須 |
| **@MainActor** | UI更新の保証 | 🔴 必須 |
| **Actor** | データ競合防止 | 🟡 推奨 |
| **TaskGroup** | 並列処理 | 🟡 推奨 |
| **AsyncSequence** | 非同期ストリーム | 🟢 参考 |

### マイグレーションガイド

```
Completion Handler → async/await
├─ func fetch(completion: @escaping (Result) -> Void)
└─ func fetch() async throws -> Data

GCD → Task/Actor
├─ DispatchQueue.main.async { }
└─ await MainActor.run { } or @MainActor

Lock → Actor
├─ NSLock, DispatchQueue(label:)
└─ actor SafeContainer { }

OperationQueue → TaskGroup
├─ OperationQueue().addOperations([])
└─ await withTaskGroup { }
```
