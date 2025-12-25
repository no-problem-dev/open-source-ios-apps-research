# Clean Architecture for SwiftUI

**スター**: 6,436 ⭐
**リポジトリ**: https://github.com/nickolashkraus/clean-architecture-swiftui
**ライセンス**: MIT

---

## 概要

Clean Architecture for SwiftUIは、Robert C. Martin（Uncle Bob）のClean Architectureを SwiftUI に適用したサンプルプロジェクト。テスタビリティ、保守性、依存性の分離を重視した設計パターンを実装。

---

## 主要機能

| 機能 | 説明 |
|------|------|
| 国リスト | REST API連携 |
| 国詳細 | 詳細情報表示 |
| 検索 | フィルタリング |
| オフライン | Core Data キャッシュ |
| テスト | 全レイヤーテスト |

---

## 技術スタック

| 項目 | 技術 |
|------|------|
| 言語 | Swift |
| UI | SwiftUI |
| アーキテクチャ | Clean Architecture |
| 永続化 | Core Data |
| ネットワーク | URLSession + Combine |
| DI | 手動依存注入 |
| テスト | XCTest |

---

## アーキテクチャ

```
CleanArchitectureSwiftUI/
├── Domain/                  # ドメイン層
│   ├── Entities/            # エンティティ
│   ├── UseCases/            # ユースケース
│   └── Interfaces/          # リポジトリインターフェース
├── Data/                    # データ層
│   ├── Repositories/        # リポジトリ実装
│   ├── DataSources/         # データソース
│   │   ├── Remote/          # API
│   │   └── Local/           # Core Data
│   └── Mappers/             # マッパー
├── Presentation/            # プレゼンテーション層
│   ├── Views/               # SwiftUI ビュー
│   ├── ViewModels/          # ビューモデル
│   └── UIComponents/        # 再利用UIコンポーネント
└── Infrastructure/          # インフラ層
    ├── DI/                  # 依存性注入
    └── Network/             # ネットワーク基盤
```

---

## 学習価値

### Clean Architecture 実装

| 分野 | 学習ポイント |
|------|-------------|
| **レイヤー分離** | 依存性ルール遵守 |
| **依存性逆転** | 抽象に依存 |
| **テスタビリティ** | 各層の単体テスト |
| **SOLID原則** | 全原則の適用 |
| **UseCase** | ビジネスロジック分離 |

### コードパターン

**エンティティ（ドメイン層）**:
```swift
// ドメインエンティティ - 外部依存なし
struct Country: Equatable, Identifiable {
    let id: String
    let name: String
    let flag: String
    let population: Int
    let capital: String
    let languages: [Language]
}

struct Language: Equatable {
    let name: String
    let nativeName: String
}
```

**ユースケース（ドメイン層）**:
```swift
// ユースケースインターフェース
protocol GetCountriesUseCase {
    func execute() async throws -> [Country]
}

// ユースケース実装
final class GetCountriesUseCaseImpl: GetCountriesUseCase {
    private let repository: CountryRepository

    init(repository: CountryRepository) {
        self.repository = repository
    }

    func execute() async throws -> [Country] {
        return try await repository.getCountries()
    }
}
```

**リポジトリインターフェース（ドメイン層）**:
```swift
// リポジトリ抽象 - ドメイン層で定義
protocol CountryRepository {
    func getCountries() async throws -> [Country]
    func getCountry(id: String) async throws -> Country
    func searchCountries(query: String) async throws -> [Country]
}
```

**リポジトリ実装（データ層）**:
```swift
// リポジトリ実装 - データ層
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
        do {
            let countries = try await remoteDataSource.fetchCountries()
            await localDataSource.cache(countries)
            return countries
        } catch {
            return try await localDataSource.getCachedCountries()
        }
    }
}
```

**ViewModel（プレゼンテーション層）**:
```swift
// ViewModel - UIロジック
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
        defer { isLoading = false }

        do {
            countries = try await getCountriesUseCase.execute()
        } catch {
            self.error = error
        }
    }
}
```

**依存性注入**:
```swift
// DIコンテナ
final class DependencyContainer {
    // Data Sources
    lazy var remoteDataSource: CountryRemoteDataSource = {
        CountryRemoteDataSourceImpl(networkService: networkService)
    }()

    lazy var localDataSource: CountryLocalDataSource = {
        CountryLocalDataSourceImpl(coreDataStack: coreDataStack)
    }()

    // Repositories
    lazy var countryRepository: CountryRepository = {
        CountryRepositoryImpl(
            remoteDataSource: remoteDataSource,
            localDataSource: localDataSource
        )
    }()

    // Use Cases
    lazy var getCountriesUseCase: GetCountriesUseCase = {
        GetCountriesUseCaseImpl(repository: countryRepository)
    }()

    // ViewModels
    func makeCountryListViewModel() -> CountryListViewModel {
        CountryListViewModel(getCountriesUseCase: getCountriesUseCase)
    }
}
```

---

## プロジェクト評価

| 観点 | 評価 | 理由 |
|------|------|------|
| **コード品質** | ★★★★★ | 教科書的な実装 |
| **ドキュメント** | ★★★★★ | アーキテクチャ解説 |
| **アクティビティ** | ★★★★☆ | 定期的な更新 |
| **学習価値** | ★★★★★ | CA学習の決定版 |
| **実用性** | ★★★★☆ | テンプレートとして使用可 |

---

## テスト戦略

### 各層のテスト

```swift
// ユースケーステスト
class GetCountriesUseCaseTests: XCTestCase {
    func testExecuteReturnsCountries() async throws {
        // Given
        let mockRepository = MockCountryRepository()
        mockRepository.countries = [Country.mock]
        let useCase = GetCountriesUseCaseImpl(repository: mockRepository)

        // When
        let result = try await useCase.execute()

        // Then
        XCTAssertEqual(result.count, 1)
        XCTAssertEqual(result.first?.name, Country.mock.name)
    }
}

// ViewModelテスト
class CountryListViewModelTests: XCTestCase {
    func testLoadCountriesUpdatesState() async {
        // Given
        let mockUseCase = MockGetCountriesUseCase()
        mockUseCase.result = [Country.mock]
        let viewModel = CountryListViewModel(getCountriesUseCase: mockUseCase)

        // When
        await viewModel.loadCountries()

        // Then
        XCTAssertFalse(viewModel.isLoading)
        XCTAssertEqual(viewModel.countries.count, 1)
    }
}
```

---

## 依存性ルール

```
┌─────────────────────────────────┐
│        Presentation Layer       │
│     (Views, ViewModels)         │
│              ↓                  │
├─────────────────────────────────┤
│         Domain Layer            │
│  (Entities, UseCases, Repos*)   │
│              ↑                  │
├─────────────────────────────────┤
│          Data Layer             │
│ (Repositories, DataSources)     │
└─────────────────────────────────┘

* Repos = Repository Interfaces (抽象)
矢印は依存方向を示す
内側のレイヤーは外側を知らない
```

---

## 開発者向け情報

### ビルド手順

```bash
git clone https://github.com/nickolashkraus/clean-architecture-swiftui
cd clean-architecture-swiftui
open CleanArchitectureSwiftUI.xcodeproj
```

### プロジェクトテンプレート

このプロジェクトを新規プロジェクトのテンプレートとして使用可能。

---

## まとめ

Clean Architecture for SwiftUIは、SwiftUIにClean Architectureを適用する模範的な実装。レイヤー分離、依存性逆転、テスタビリティなど、スケーラブルなアプリ設計の教科書。大規模プロジェクトの設計に必須の知識を学べる。

**推奨対象**:
- Clean Architectureを SwiftUI で実践したい開発者
- テスタブルなアプリ設計を学びたい人
- 大規模プロジェクトの設計パターンを参考にしたい人
- SOLID原則の実践例を見たい人
