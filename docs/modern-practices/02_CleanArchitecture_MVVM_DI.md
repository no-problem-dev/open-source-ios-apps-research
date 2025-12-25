# Clean Architecture for SwiftUI - 設計パターンの教科書

**リポジトリ**: https://github.com/nickolashkraus/clean-architecture-swiftui
**実装参考度**: ⭐⭐⭐⭐⭐

---

## なぜ参考になるか

Clean Architectureを**SwiftUIに正しく適用した模範的実装**。テスト駆動開発、依存性注入、レイヤー分離を学べる。

| 学習ポイント | 重要度 | 理由 |
|-------------|--------|------|
| **レイヤー分離** | ⭐⭐⭐⭐⭐ | 責務の明確な分離 |
| **依存性逆転** | ⭐⭐⭐⭐⭐ | テスタブルな設計 |
| **UseCase** | ⭐⭐⭐⭐⭐ | ビジネスロジック分離 |
| **Repository** | ⭐⭐⭐⭐⭐ | データアクセス抽象化 |
| **DIコンテナ** | ⭐⭐⭐⭐☆ | 依存性管理 |

---

## 1. レイヤー構成

### 全体像

```
┌─────────────────────────────────────┐
│         Presentation Layer          │
│        (View, ViewModel)            │
│                 │                   │
│                 ▼                   │
├─────────────────────────────────────┤
│           Domain Layer              │
│  (Entity, UseCase, Repository IF)   │
│                 ▲                   │
│                 │                   │
├─────────────────────────────────────┤
│            Data Layer               │
│  (Repository Impl, DataSource)      │
└─────────────────────────────────────┘

※ 矢印は依存方向。内側は外側を知らない。
```

### ディレクトリ構成

```
CleanArchitectureSwiftUI/
├── Domain/                  # ドメイン層（ビジネスロジック）
│   ├── Entities/            # ビジネスエンティティ
│   │   └── Country.swift
│   ├── UseCases/            # ユースケース
│   │   └── GetCountriesUseCase.swift
│   └── Interfaces/          # リポジトリインターフェース
│       └── CountryRepository.swift
│
├── Data/                    # データ層
│   ├── Repositories/        # リポジトリ実装
│   │   └── CountryRepositoryImpl.swift
│   └── DataSources/         # データソース
│       ├── Remote/          # API
│       └── Local/           # Core Data
│
├── Presentation/            # プレゼンテーション層
│   ├── Views/               # SwiftUI View
│   ├── ViewModels/          # ViewModel
│   └── UIComponents/        # 再利用UI
│
└── Infrastructure/          # インフラ層
    ├── DI/                  # 依存性注入
    └── Network/             # ネットワーク基盤
```

---

## 2. Domain Layer（最重要）

### Entity - 純粋なビジネスモデル

```swift
// Domain/Entities/Country.swift
// 外部フレームワークに依存しない純粋な構造体
struct Country: Equatable, Identifiable {
    let id: String
    let name: String
    let capital: String
    let population: Int
    let flag: String

    // ビジネスルールはここに
    var isLargeCountry: Bool {
        population > 100_000_000
    }
}
```

**ポイント**:
- Codableなどのフレームワーク依存を避ける
- ビジネスルールのみを持つ
- テストしやすい純粋な構造体

### UseCase - ビジネスロジック

```swift
// Domain/UseCases/GetCountriesUseCase.swift

// プロトコル定義
protocol GetCountriesUseCase {
    func execute() async throws -> [Country]
}

// 実装
final class GetCountriesUseCaseImpl: GetCountriesUseCase {
    private let repository: CountryRepository

    init(repository: CountryRepository) {
        self.repository = repository
    }

    func execute() async throws -> [Country] {
        let countries = try await repository.getCountries()
        // ビジネスロジック: 人口順でソート
        return countries.sorted { $0.population > $1.population }
    }
}
```

**ポイント**:
- 1つのUseCaseは1つの処理のみ
- Repositoryインターフェースに依存（実装ではない）
- ビジネスルールをここに集約

### Repository Interface - データアクセス抽象化

```swift
// Domain/Interfaces/CountryRepository.swift
// ドメイン層で定義（実装はData層）
protocol CountryRepository {
    func getCountries() async throws -> [Country]
    func getCountry(id: String) async throws -> Country
    func searchCountries(query: String) async throws -> [Country]
}
```

---

## 3. Data Layer

### Repository Implementation

```swift
// Data/Repositories/CountryRepositoryImpl.swift
final class CountryRepositoryImpl: CountryRepository {
    private let remoteDataSource: CountryRemoteDataSource
    private let localDataSource: CountryLocalDataSource

    init(
        remoteDataSource: CountryRemoteDataSource,
        localDataSource: CountryLocalDataSource
    ) {
        self.remoteDataSource = remoteDataSource
        self.localDataSource = localDataSource
    }

    func getCountries() async throws -> [Country] {
        // オンライン時: リモートから取得してキャッシュ
        // オフライン時: ローカルキャッシュから取得
        do {
            let dtos = try await remoteDataSource.fetchCountries()
            let countries = dtos.map { $0.toDomain() }
            await localDataSource.cache(countries)
            return countries
        } catch {
            // フォールバック: キャッシュから取得
            return try await localDataSource.getCachedCountries()
        }
    }
}
```

### DTO (Data Transfer Object)

```swift
// Data/DataSources/Remote/CountryDTO.swift
struct CountryDTO: Decodable {
    let cca3: String
    let name: NameDTO
    let capital: [String]?
    let population: Int
    let flag: String

    struct NameDTO: Decodable {
        let common: String
    }

    // DTOからDomainエンティティへの変換
    func toDomain() -> Country {
        Country(
            id: cca3,
            name: name.common,
            capital: capital?.first ?? "",
            population: population,
            flag: flag
        )
    }
}
```

**ポイント**:
- DTOはAPI仕様に合わせる（Codable）
- toDomain()でドメインエンティティに変換
- ドメインをAPIの変更から保護

---

## 4. Presentation Layer

### ViewModel

```swift
// Presentation/ViewModels/CountryListViewModel.swift
@MainActor
final class CountryListViewModel: ObservableObject {
    @Published private(set) var countries: [Country] = []
    @Published private(set) var isLoading = false
    @Published private(set) var error: Error?

    private let getCountriesUseCase: GetCountriesUseCase

    init(getCountriesUseCase: GetCountriesUseCase) {
        self.getCountriesUseCase = getCountriesUseCase
    }

    func loadCountries() async {
        isLoading = true
        error = nil

        do {
            countries = try await getCountriesUseCase.execute()
        } catch {
            self.error = error
        }

        isLoading = false
    }
}
```

### View

```swift
// Presentation/Views/CountryListView.swift
struct CountryListView: View {
    @StateObject var viewModel: CountryListViewModel

    var body: some View {
        Group {
            if viewModel.isLoading {
                ProgressView()
            } else if let error = viewModel.error {
                ErrorView(error: error, retry: {
                    Task { await viewModel.loadCountries() }
                })
            } else {
                List(viewModel.countries) { country in
                    CountryRow(country: country)
                }
            }
        }
        .task {
            await viewModel.loadCountries()
        }
    }
}
```

---

## 5. Dependency Injection

### DIコンテナ

```swift
// Infrastructure/DI/DependencyContainer.swift
final class DependencyContainer {
    // MARK: - Network
    lazy var networkService: NetworkService = {
        NetworkServiceImpl(session: .shared)
    }()

    // MARK: - Data Sources
    lazy var countryRemoteDataSource: CountryRemoteDataSource = {
        CountryRemoteDataSourceImpl(networkService: networkService)
    }()

    lazy var countryLocalDataSource: CountryLocalDataSource = {
        CountryLocalDataSourceImpl(coreDataStack: coreDataStack)
    }()

    // MARK: - Repositories
    lazy var countryRepository: CountryRepository = {
        CountryRepositoryImpl(
            remoteDataSource: countryRemoteDataSource,
            localDataSource: countryLocalDataSource
        )
    }()

    // MARK: - Use Cases
    lazy var getCountriesUseCase: GetCountriesUseCase = {
        GetCountriesUseCaseImpl(repository: countryRepository)
    }()

    // MARK: - ViewModels
    func makeCountryListViewModel() -> CountryListViewModel {
        CountryListViewModel(getCountriesUseCase: getCountriesUseCase)
    }
}
```

### アプリでの使用

```swift
@main
struct CleanArchitectureApp: App {
    let container = DependencyContainer()

    var body: some Scene {
        WindowGroup {
            CountryListView(
                viewModel: container.makeCountryListViewModel()
            )
        }
    }
}
```

---

## 6. テスト戦略

### UseCaseのテスト

```swift
// Tests/UseCaseTests/GetCountriesUseCaseTests.swift
final class GetCountriesUseCaseTests: XCTestCase {
    func testExecuteReturnsCountriesSortedByPopulation() async throws {
        // Arrange
        let mockRepository = MockCountryRepository()
        mockRepository.countries = [
            Country(id: "1", name: "Small", capital: "", population: 100, flag: ""),
            Country(id: "2", name: "Large", capital: "", population: 1000, flag: ""),
        ]
        let useCase = GetCountriesUseCaseImpl(repository: mockRepository)

        // Act
        let result = try await useCase.execute()

        // Assert
        XCTAssertEqual(result.first?.name, "Large") // 人口順
    }
}

// Mock
class MockCountryRepository: CountryRepository {
    var countries: [Country] = []

    func getCountries() async throws -> [Country] {
        return countries
    }

    func getCountry(id: String) async throws -> Country {
        countries.first { $0.id == id }!
    }

    func searchCountries(query: String) async throws -> [Country] {
        countries.filter { $0.name.contains(query) }
    }
}
```

### ViewModelのテスト

```swift
final class CountryListViewModelTests: XCTestCase {
    @MainActor
    func testLoadCountriesUpdatesState() async {
        // Arrange
        let mockUseCase = MockGetCountriesUseCase()
        mockUseCase.result = [Country.mock]
        let viewModel = CountryListViewModel(getCountriesUseCase: mockUseCase)

        // Act
        await viewModel.loadCountries()

        // Assert
        XCTAssertFalse(viewModel.isLoading)
        XCTAssertEqual(viewModel.countries.count, 1)
        XCTAssertNil(viewModel.error)
    }
}
```

---

## まとめ：Clean Architectureから学ぶべきこと

| 原則 | 実践方法 | メリット |
|------|---------|---------|
| **単一責任** | 1クラス1責務 | 変更の影響範囲が限定 |
| **依存性逆転** | インターフェースに依存 | テスト容易性 |
| **開放閉鎖** | 拡張に開いて修正に閉じる | 機能追加が容易 |

### 即座に取り入れられること

1. DTO ↔ Domain Entity の変換パターン
2. UseCase によるビジネスロジック分離
3. Repository インターフェースによるデータアクセス抽象化
4. DIコンテナによる依存性管理

### このパターンを使うべきケース

- 中〜大規模アプリ
- テストを重視するプロジェクト
- 長期メンテナンスが必要なアプリ
- チーム開発

### 使わなくてもいいケース

- 小規模なプロトタイプ
- 短命なアプリ
- 1人開発の小さなツール
