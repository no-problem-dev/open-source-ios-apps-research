# Kickstarter iOS

**スター**: 8,601 ⭐
**リポジトリ**: https://github.com/kickstarter/ios-oss
**ライセンス**: Apache-2.0

---

## 概要

Kickstarter公式iOSアプリのオープンソース版。関数型リアクティブプログラミング（FRP）とMVVMアーキテクチャの模範的実装として知られ、テスト駆動開発の教科書的存在。

---

## 主要機能

| 機能 | 説明 |
|------|------|
| プロジェクト閲覧 | 発見、検索、カテゴリ |
| 支援 | Stripe決済 |
| 更新情報 | プロジェクト進捗 |
| メッセージ | クリエイターとの連絡 |
| プロフィール | 支援履歴管理 |
| プッシュ通知 | 更新通知 |

---

## 技術スタック

| 項目 | 技術 |
|------|------|
| 言語 | Swift |
| アーキテクチャ | MVVM + FRP |
| リアクティブ | ReactiveSwift |
| ネットワーク | Alamofire |
| 決済 | Stripe |
| テスト | ios-snapshot-test-case |
| 認証 | 1Password |

---

## アーキテクチャ

```
Kickstarter-iOS/
├── Kickstarter-iOS/      # メインアプリ
│   ├── Features/         # 機能モジュール
│   │   ├── Discovery/    # プロジェクト発見
│   │   ├── Project/      # プロジェクト詳細
│   │   └── Checkout/     # 決済フロー
│   └── Library/          # 共通ライブラリ
├── Library/              # ビジネスロジック
│   ├── ViewModels/       # MVVM ViewModel
│   └── Services/         # APIサービス
└── KsApi/                # APIクライアント
```

---

## 学習価値

### MVVM + FRPの模範

| 分野 | 学習ポイント |
|------|-------------|
| **MVVM** | ViewModelの完璧な分離 |
| **ReactiveSwift** | Signal/Property活用 |
| **テスト** | 高いテストカバレッジ |
| **スナップショット** | UIのビジュアルテスト |
| **依存注入** | Environment パターン |

### コードパターン

**ViewModel定義**:
```swift
protocol ProjectViewModelInputs {
    func configureWith(project: Project)
    func pledgeButtonTapped()
}

protocol ProjectViewModelOutputs {
    var projectName: Signal<String, Never> { get }
    var pledgeButtonTitle: Signal<String, Never> { get }
    var goToPledge: Signal<Project, Never> { get }
}

protocol ProjectViewModelType {
    var inputs: ProjectViewModelInputs { get }
    var outputs: ProjectViewModelOutputs { get }
}

class ProjectViewModel: ProjectViewModelType, ProjectViewModelInputs, ProjectViewModelOutputs {
    init() {
        self.projectName = self.projectProperty.signal
            .skipNil()
            .map { $0.name }

        self.goToPledge = self.pledgeButtonTappedProperty.signal
            .withLatest(from: self.projectProperty.signal.skipNil())
            .map(second)
    }

    // Inputs
    private let projectProperty = MutableProperty<Project?>(nil)
    func configureWith(project: Project) {
        self.projectProperty.value = project
    }

    private let pledgeButtonTappedProperty = MutableProperty(())
    func pledgeButtonTapped() {
        self.pledgeButtonTappedProperty.value = ()
    }

    // Outputs
    let projectName: Signal<String, Never>
    let pledgeButtonTitle: Signal<String, Never>
    let goToPledge: Signal<Project, Never>

    var inputs: ProjectViewModelInputs { return self }
    var outputs: ProjectViewModelOutputs { return self }
}
```

**Environment パターン**:
```swift
struct Environment {
    let apiService: ServiceType
    let currentUser: User?
    let scheduler: DateScheduler
    let ubiquitousStore: KeyValueStoreType

    static let production = Environment(
        apiService: Service(),
        currentUser: nil,
        scheduler: QueueScheduler.main,
        ubiquitousStore: NSUbiquitousKeyValueStore.default
    )
}

// テスト用モック
let testEnv = Environment(
    apiService: MockService(),
    currentUser: User.template,
    scheduler: TestScheduler(),
    ubiquitousStore: MockKeyValueStore()
)
```

---

## テスト戦略

### スナップショットテスト

```swift
class ProjectViewControllerTests: TestCase {
    func testProjectView() {
        let vc = ProjectViewController.instantiate()
        vc.viewModel.inputs.configureWith(project: .template)

        FBSnapshotVerifyView(vc.view)
    }

    func testProjectView_BackedState() {
        let project = Project.template |> Project.lens.personalization.isBacking .~ true
        let vc = ProjectViewController.instantiate()
        vc.viewModel.inputs.configureWith(project: project)

        FBSnapshotVerifyView(vc.view)
    }
}
```

### ViewModelテスト

```swift
class ProjectViewModelTests: TestCase {
    let vm = ProjectViewModel()

    func testProjectName() {
        self.vm.inputs.configureWith(project: .template |> Project.lens.name .~ "Test Project")

        self.vm.outputs.projectName
            .observe { XCTAssertEqual($0.value, "Test Project") }
    }
}
```

---

## プロジェクト評価

| 観点 | 評価 | 理由 |
|------|------|------|
| **コード品質** | ★★★★★ | 業界最高水準 |
| **ドキュメント** | ★★★★★ | 設計思想が明確 |
| **アクティビティ** | ★★★★☆ | 継続的なメンテナンス |
| **学習価値** | ★★★★★ | MVVM/FRPの教科書 |
| **実用性** | ★★★★★ | 本番アプリ |

---

## 開発者向け情報

### ビルド手順

```bash
git clone https://github.com/kickstarter/ios-oss
cd ios-oss
make bootstrap
open Kickstarter.xcworkspace
```

### 設計原則

- **純粋関数**: 副作用を分離
- **テスタビリティ**: すべてが注入可能
- **型安全**: Lens活用
- **イミュータブル**: 状態変更の追跡

---

## まとめ

Kickstarter iOSは、MVVMとリアクティブプログラミングの最も洗練された実装例。テスト戦略、依存注入、関数型プログラミングのベストプラクティスが詰まった、iOS開発者必見のプロジェクト。

**推奨対象**:
- MVVM/FRPを深く学びたい開発者
- テスト駆動開発を実践したい人
- 大規模アプリの設計を参考にしたい人

