# MVVM 実装パターン比較

**参考プロジェクト**:
- Clean Architecture SwiftUI
- Kickstarter iOS
- Various open source apps

**実装参考度**: ⭐⭐⭐⭐⭐

---

## なぜ重要か

MVVMはSwiftUIと相性の良いアーキテクチャ。ただし実装方法は複数あり、プロジェクトに合った選択が重要。

| パターン | 特徴 | 適したケース |
|---------|------|-------------|
| **@Observable** | iOS 17+、シンプル | 新規プロジェクト |
| **ObservableObject** | iOS 14+、広く普及 | 既存プロジェクト |
| **Input/Output** | テスト容易 | 大規模プロジェクト |
| **Protocol-based** | 柔軟性高 | DI重視 |

---

## 1. @Observable パターン（iOS 17+）

### 基本実装

```swift
import SwiftUI

@Observable
class ItemListViewModel {
    var items: [Item] = []
    var isLoading = false
    var errorMessage: String?

    private let repository: ItemRepository

    init(repository: ItemRepository = ItemRepositoryImpl()) {
        self.repository = repository
    }

    func loadItems() async {
        isLoading = true
        errorMessage = nil

        do {
            items = try await repository.fetchItems()
        } catch {
            errorMessage = error.localizedDescription
        }

        isLoading = false
    }

    func deleteItem(_ item: Item) async {
        do {
            try await repository.delete(item)
            items.removeAll { $0.id == item.id }
        } catch {
            errorMessage = error.localizedDescription
        }
    }
}

struct ItemListView: View {
    @State private var viewModel = ItemListViewModel()

    var body: some View {
        List {
            ForEach(viewModel.items) { item in
                ItemRow(item: item)
            }
            .onDelete { indexSet in
                Task {
                    for index in indexSet {
                        await viewModel.deleteItem(viewModel.items[index])
                    }
                }
            }
        }
        .overlay {
            if viewModel.isLoading {
                ProgressView()
            }
        }
        .alert("Error", isPresented: .constant(viewModel.errorMessage != nil)) {
            Button("OK") { viewModel.errorMessage = nil }
        } message: {
            Text(viewModel.errorMessage ?? "")
        }
        .task {
            await viewModel.loadItems()
        }
    }
}
```

### @ObservationIgnored

```swift
@Observable
class UserViewModel {
    var user: User?
    var isEditing = false

    // 追跡しないプロパティ
    @ObservationIgnored
    private let analyticsService: AnalyticsService

    @ObservationIgnored
    private var cancellables = Set<AnyCancellable>()

    init(analyticsService: AnalyticsService = AnalyticsServiceImpl()) {
        self.analyticsService = analyticsService
    }
}
```

---

## 2. ObservableObject パターン（iOS 14+）

### 基本実装

```swift
class ItemListViewModel: ObservableObject {
    @Published var items: [Item] = []
    @Published var isLoading = false
    @Published var errorMessage: String?

    private let repository: ItemRepository
    private var cancellables = Set<AnyCancellable>()

    init(repository: ItemRepository = ItemRepositoryImpl()) {
        self.repository = repository
    }

    func loadItems() {
        isLoading = true
        errorMessage = nil

        repository.fetchItems()
            .receive(on: DispatchQueue.main)
            .sink(
                receiveCompletion: { [weak self] completion in
                    self?.isLoading = false
                    if case .failure(let error) = completion {
                        self?.errorMessage = error.localizedDescription
                    }
                },
                receiveValue: { [weak self] items in
                    self?.items = items
                }
            )
            .store(in: &cancellables)
    }
}

struct ItemListView: View {
    @StateObject private var viewModel = ItemListViewModel()

    var body: some View {
        List(viewModel.items) { item in
            ItemRow(item: item)
        }
        .onAppear {
            viewModel.loadItems()
        }
    }
}
```

---

## 3. Input/Output パターン

### プロトコル定義

```swift
// Inputs: Viewからの入力
protocol ItemListViewModelInputs {
    func viewDidLoad()
    func refreshTriggered()
    func itemSelected(_ item: Item)
    func deleteButtonTapped(at index: Int)
}

// Outputs: Viewへの出力
protocol ItemListViewModelOutputs {
    var items: [Item] { get }
    var isLoading: Bool { get }
    var errorMessage: String? { get }
    var selectedItem: Item? { get }
}

// Type: インターフェース
protocol ItemListViewModelType: ObservableObject {
    var inputs: ItemListViewModelInputs { get }
    var outputs: ItemListViewModelOutputs { get }
}
```

### 実装

```swift
final class ItemListViewModel: ItemListViewModelType,
                               ItemListViewModelInputs,
                               ItemListViewModelOutputs {

    // MARK: - Outputs
    @Published private(set) var items: [Item] = []
    @Published private(set) var isLoading = false
    @Published private(set) var errorMessage: String?
    @Published private(set) var selectedItem: Item?

    // MARK: - Dependencies
    private let repository: ItemRepository

    // MARK: - Init
    init(repository: ItemRepository = ItemRepositoryImpl()) {
        self.repository = repository
    }

    // MARK: - Inputs
    func viewDidLoad() {
        loadItems()
    }

    func refreshTriggered() {
        loadItems()
    }

    func itemSelected(_ item: Item) {
        selectedItem = item
    }

    func deleteButtonTapped(at index: Int) {
        guard index < items.count else { return }
        let item = items[index]
        deleteItem(item)
    }

    // MARK: - Protocol Conformance
    var inputs: ItemListViewModelInputs { self }
    var outputs: ItemListViewModelOutputs { self }

    // MARK: - Private
    private func loadItems() {
        isLoading = true
        Task {
            do {
                items = try await repository.fetchItems()
            } catch {
                errorMessage = error.localizedDescription
            }
            isLoading = false
        }
    }

    private func deleteItem(_ item: Item) {
        Task {
            do {
                try await repository.delete(item)
                items.removeAll { $0.id == item.id }
            } catch {
                errorMessage = error.localizedDescription
            }
        }
    }
}
```

### View実装

```swift
struct ItemListView<VM: ItemListViewModelType>: View {
    @StateObject var viewModel: VM

    var body: some View {
        List {
            ForEach(viewModel.outputs.items) { item in
                ItemRow(item: item)
                    .onTapGesture {
                        viewModel.inputs.itemSelected(item)
                    }
            }
            .onDelete { indexSet in
                indexSet.forEach {
                    viewModel.inputs.deleteButtonTapped(at: $0)
                }
            }
        }
        .refreshable {
            viewModel.inputs.refreshTriggered()
        }
        .onAppear {
            viewModel.inputs.viewDidLoad()
        }
    }
}
```

---

## 4. Protocol-based DI パターン

### プロトコル定義

```swift
// ViewModel Protocol
protocol ItemListViewModelProtocol: ObservableObject {
    var items: [Item] { get }
    var isLoading: Bool { get }

    func loadItems() async
    func deleteItem(_ item: Item) async
}

// Repository Protocol
protocol ItemRepository {
    func fetchItems() async throws -> [Item]
    func delete(_ item: Item) async throws
}
```

### 実装とモック

```swift
// 本番実装
final class ItemListViewModelImpl: ItemListViewModelProtocol {
    @Published private(set) var items: [Item] = []
    @Published private(set) var isLoading = false

    private let repository: ItemRepository

    init(repository: ItemRepository) {
        self.repository = repository
    }

    @MainActor
    func loadItems() async {
        isLoading = true
        defer { isLoading = false }

        do {
            items = try await repository.fetchItems()
        } catch {
            // Error handling
        }
    }

    @MainActor
    func deleteItem(_ item: Item) async {
        do {
            try await repository.delete(item)
            items.removeAll { $0.id == item.id }
        } catch {
            // Error handling
        }
    }
}

// テスト用モック
final class MockItemListViewModel: ItemListViewModelProtocol {
    @Published var items: [Item] = []
    @Published var isLoading = false

    var loadItemsCalled = false
    var deleteItemCalled = false

    func loadItems() async {
        loadItemsCalled = true
    }

    func deleteItem(_ item: Item) async {
        deleteItemCalled = true
        items.removeAll { $0.id == item.id }
    }
}
```

### View（ジェネリック）

```swift
struct ItemListView<VM: ItemListViewModelProtocol>: View {
    @StateObject var viewModel: VM

    var body: some View {
        List(viewModel.items) { item in
            ItemRow(item: item)
        }
        .task {
            await viewModel.loadItems()
        }
    }
}

// 使用
ItemListView(viewModel: ItemListViewModelImpl(repository: ItemRepositoryImpl()))

// プレビュー
#Preview {
    let mockVM = MockItemListViewModel()
    mockVM.items = [Item.sample]
    return ItemListView(viewModel: mockVM)
}
```

---

## 5. パターン比較

| 観点 | @Observable | ObservableObject | Input/Output |
|------|-------------|------------------|--------------|
| **iOS要件** | 17+ | 14+ | 14+ |
| **コード量** | 少 | 中 | 多 |
| **テスト** | 普通 | 普通 | 容易 |
| **型安全** | 高 | 高 | 最高 |
| **学習コスト** | 低 | 低 | 中 |
| **再描画効率** | 最高 | 普通 | 普通 |

### 選択ガイドライン

```
新規プロジェクト（iOS 17+）
├─ シンプルなアプリ → @Observable
├─ 中規模アプリ → @Observable + Protocol DI
└─ 大規模/テスト重視 → Input/Output

既存プロジェクト（iOS 14+）
├─ シンプルなアプリ → ObservableObject
├─ 中規模アプリ → ObservableObject + Protocol DI
└─ 大規模/テスト重視 → Input/Output
```

---

## まとめ

| パターン | 推奨ケース | 優先度 |
|---------|-----------|--------|
| **@Observable** | iOS 17+新規開発 | 🔴 必須 |
| **Protocol DI** | テスト・モック | 🟡 推奨 |
| **Input/Output** | 大規模・チーム開発 | 🟢 参考 |
| **ObservableObject** | レガシー対応 | 🟢 参考 |
