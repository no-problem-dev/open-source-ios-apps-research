# Kickstarter - MVVM + FRP + テスト駆動開発の極み

**リポジトリ**: https://github.com/kickstarter/ios-oss
**実装参考度**: ⭐⭐⭐⭐⭐

---

## なぜ参考になるか

**企業レベルの本番アプリ**のオープンソース実装。テスト駆動開発、ReactiveSwift、厳密なMVVMパターンを学べる。

| 学習ポイント | 重要度 | 理由 |
|-------------|--------|------|
| **MVVM + FRP** | ⭐⭐⭐⭐⭐ | ReactiveSwiftによる完全なリアクティブ実装 |
| **テスト駆動** | ⭐⭐⭐⭐⭐ | 90%以上のテストカバレッジ |
| **Playground駆動開発** | ⭐⭐⭐⭐⭐ | Playgroundでの高速UI開発 |
| **スナップショットテスト** | ⭐⭐⭐⭐☆ | UIのリグレッション防止 |
| **A/Bテスト設計** | ⭐⭐⭐⭐☆ | 本番での機能検証 |

---

## 1. ViewModel設計パターン

### Input/Output Protocol

```swift
// Library/ViewModels/ProjectViewModel.swift
protocol ProjectViewModelInputs {
    func configureWith(project: Project)
    func pledgeButtonTapped()
    func shareButtonTapped()
    func viewDidLoad()
    func viewWillAppear()
}

protocol ProjectViewModelOutputs {
    var projectName: Signal<String, Never> { get }
    var pledgeButtonTitle: Signal<String, Never> { get }
    var pledgeButtonEnabled: Signal<Bool, Never> { get }
    var showShareSheet: Signal<URL, Never> { get }
    var goToPledge: Signal<Project, Never> { get }
}

protocol ProjectViewModelType {
    var inputs: ProjectViewModelInputs { get }
    var outputs: ProjectViewModelOutputs { get }
}

final class ProjectViewModel: ProjectViewModelType,
                              ProjectViewModelInputs,
                              ProjectViewModelOutputs {

    init() {
        // Input → Output の変換ロジック
        let project = self.configureWithProjectProperty.signal.skipNil()

        self.projectName = project.map { $0.name }

        self.pledgeButtonTitle = project.map { project in
            project.state == .live ? "Back this project" : "View rewards"
        }

        self.pledgeButtonEnabled = project.map { $0.state == .live }

        self.showShareSheet = project
            .takeWhen(self.shareButtonTappedProperty.signal)
            .map { $0.urls.web.project }

        self.goToPledge = project
            .takeWhen(self.pledgeButtonTappedProperty.signal)
    }

    // MARK: - Inputs
    private let configureWithProjectProperty = MutableProperty<Project?>(nil)
    func configureWith(project: Project) {
        self.configureWithProjectProperty.value = project
    }

    private let pledgeButtonTappedProperty = MutableProperty(())
    func pledgeButtonTapped() {
        self.pledgeButtonTappedProperty.value = ()
    }

    private let shareButtonTappedProperty = MutableProperty(())
    func shareButtonTapped() {
        self.shareButtonTappedProperty.value = ()
    }

    func viewDidLoad() { /* tracking */ }
    func viewWillAppear() { /* refresh */ }

    // MARK: - Outputs
    let projectName: Signal<String, Never>
    let pledgeButtonTitle: Signal<String, Never>
    let pledgeButtonEnabled: Signal<Bool, Never>
    let showShareSheet: Signal<URL, Never>
    let goToPledge: Signal<Project, Never>

    var inputs: ProjectViewModelInputs { return self }
    var outputs: ProjectViewModelOutputs { return self }
}
```

### ViewControllerでの使用

```swift
final class ProjectViewController: UIViewController {
    private let viewModel: ProjectViewModelType = ProjectViewModel()

    override func viewDidLoad() {
        super.viewDidLoad()

        // Outputsをバインド
        self.viewModel.outputs.projectName
            .observeForUI()
            .observeValues { [weak self] name in
                self?.titleLabel.text = name
            }

        self.viewModel.outputs.pledgeButtonTitle
            .observeForUI()
            .observeValues { [weak self] title in
                self?.pledgeButton.setTitle(title, for: .normal)
            }

        self.viewModel.outputs.goToPledge
            .observeForControllerAction()
            .observeValues { [weak self] project in
                self?.goToPledge(project)
            }

        // Inputsを送信
        self.viewModel.inputs.viewDidLoad()
    }

    @IBAction func pledgeButtonTapped() {
        self.viewModel.inputs.pledgeButtonTapped()
    }
}
```

---

## 2. テスト駆動開発

### ViewModelのテスト

```swift
// KsApiTests/ViewModels/ProjectViewModelTests.swift
final class ProjectViewModelTests: TestCase {
    let vm: ProjectViewModelType = ProjectViewModel()

    func testProjectName() {
        let project = .template |> Project.lens.name .~ "Cool Project"

        self.vm.inputs.configureWith(project: project)
        self.vm.inputs.viewDidLoad()

        self.projectName.assertValues(["Cool Project"])
    }

    func testPledgeButton_LiveProject() {
        let liveProject = .template |> Project.lens.state .~ .live

        self.vm.inputs.configureWith(project: liveProject)

        self.pledgeButtonTitle.assertValues(["Back this project"])
        self.pledgeButtonEnabled.assertValues([true])
    }

    func testPledgeButton_EndedProject() {
        let endedProject = .template |> Project.lens.state .~ .successful

        self.vm.inputs.configureWith(project: endedProject)

        self.pledgeButtonTitle.assertValues(["View rewards"])
        self.pledgeButtonEnabled.assertValues([false])
    }

    func testGoToPledge() {
        let project = Project.template

        self.vm.inputs.configureWith(project: project)
        self.vm.inputs.pledgeButtonTapped()

        self.goToPledge.assertValues([project])
    }
}
```

### テスト用のLensパターン

```swift
// Lensを使ったテストデータ作成
extension Project {
    enum lens {
        static let name = Lens<Project, String>(
            get: { $0.name },
            set: { Project(id: $1.id, name: $0, state: $1.state, /* ... */) }
        )

        static let state = Lens<Project, State>(
            get: { $0.state },
            set: { Project(id: $1.id, name: $1.name, state: $0, /* ... */) }
        )
    }
}

// 使用例
let project = Project.template
    |> Project.lens.name .~ "My Project"
    |> Project.lens.state .~ .live
```

---

## 3. スナップショットテスト

### UI検証の自動化

```swift
// KickstarterTests/Views/ProjectCardViewTests.swift
import SnapshotTesting

final class ProjectCardViewTests: TestCase {
    func testProjectCard_LiveProject() {
        let project = Project.template |> Project.lens.state .~ .live
        let view = ProjectCardView(frame: CGRect(x: 0, y: 0, width: 320, height: 200))
        view.configure(with: project)

        assertSnapshot(matching: view, as: .image)
    }

    func testProjectCard_SuccessfulProject() {
        let project = Project.template |> Project.lens.state .~ .successful
        let view = ProjectCardView(frame: CGRect(x: 0, y: 0, width: 320, height: 200))
        view.configure(with: project)

        assertSnapshot(matching: view, as: .image)
    }

    func testProjectCard_DarkMode() {
        let project = Project.template
        let view = ProjectCardView(frame: CGRect(x: 0, y: 0, width: 320, height: 200))
        view.configure(with: project)
        view.overrideUserInterfaceStyle = .dark

        assertSnapshot(matching: view, as: .image)
    }
}
```

---

## 4. Playground駆動開発

### 高速UIイテレーション

```swift
// Kickstarter-iOS.playground/Sources/PlaygroundHelpers.swift
import PlaygroundSupport
import UIKit

public func playgroundControllers(
    device: Device = .phone4_7inch,
    orientation: Orientation = .portrait,
    child: UIViewController
) -> UIViewController {

    let parent = UIViewController()
    parent.addChild(child)
    parent.view.addSubview(child.view)

    child.view.frame = device.frame(orientation)
    parent.preferredContentSize = child.view.frame.size

    return parent
}

// Playgroundでの使用
let vc = ProjectViewController.instantiate()
vc.configure(with: Project.template)

PlaygroundPage.current.liveView = playgroundControllers(
    device: .phone4_7inch,
    orientation: .portrait,
    child: vc
)
```

---

## 5. Environment パターン

### 依存性注入

```swift
// Library/Environment.swift
struct Environment {
    let apiService: ServiceType
    let currentUser: User?
    let dateType: DateProtocol.Type
    let language: Language
    let locale: Locale
    let mainBundle: BundleType
    let scheduler: DateScheduler
    let ubiquitousStore: KeyValueStoreType
    let userDefaults: KeyValueStoreType

    init(
        apiService: ServiceType = Service(),
        currentUser: User? = nil,
        dateType: DateProtocol.Type = Date.self,
        language: Language = .en,
        locale: Locale = .current,
        mainBundle: BundleType = Bundle.main,
        scheduler: DateScheduler = QueueScheduler.main,
        ubiquitousStore: KeyValueStoreType = NSUbiquitousKeyValueStore.default,
        userDefaults: KeyValueStoreType = UserDefaults.standard
    ) {
        self.apiService = apiService
        self.currentUser = currentUser
        self.dateType = dateType
        self.language = language
        self.locale = locale
        self.mainBundle = mainBundle
        self.scheduler = scheduler
        self.ubiquitousStore = ubiquitousStore
        self.userDefaults = userDefaults
    }
}

// グローバルアクセス
private var _currentEnvironment = Environment()
var AppEnvironment: Environment {
    get { return _currentEnvironment }
    set { _currentEnvironment = newValue }
}

// テスト用
extension Environment {
    static func pushEnvironment(_ env: Environment) { /* ... */ }
    static func popEnvironment() { /* ... */ }
}
```

### テストでの環境切り替え

```swift
final class ProjectViewModelTests: TestCase {
    override func setUp() {
        super.setUp()

        // テスト用環境を設定
        AppEnvironment.pushEnvironment(
            Environment(
                apiService: MockService(),
                currentUser: User.template,
                dateType: MockDate.self,
                scheduler: self.scheduler
            )
        )
    }

    override func tearDown() {
        AppEnvironment.popEnvironment()
        super.tearDown()
    }
}
```

---

## 6. A/Bテスト設計

### 機能フラグ

```swift
// Library/Experiments/OptimizelyFeature.swift
enum OptimizelyFeature: String {
    case pledgeViewCTAEnabled = "ios_pledge_view_cta_enabled"
    case nativeRiskMessaging = "ios_native_risk_messaging"
    case newPaymentFlow = "ios_new_payment_flow"

    func isEnabled(for user: User?) -> Bool {
        let client = AppEnvironment.current.optimizelyClient
        let userId = user?.id.description ?? UUID().uuidString

        return client?.isFeatureEnabled(
            featureKey: self.rawValue,
            userId: userId
        ) ?? false
    }
}

// 使用例
if OptimizelyFeature.newPaymentFlow.isEnabled(for: currentUser) {
    // 新しい支払いフロー
} else {
    // 既存の支払いフロー
}
```

---

## まとめ：Kickstarterから学ぶべきこと

| 技術 | 学習ポイント | 優先度 |
|------|-------------|--------|
| **Input/Output VM** | テスタブルなViewModel設計 | 🔴 必須 |
| **ReactiveSwift** | リアクティブプログラミング | 🟡 推奨 |
| **スナップショットテスト** | UIリグレッション防止 | 🟡 推奨 |
| **Playground開発** | 高速UIイテレーション | 🟢 参考 |
| **Environment DI** | 依存性注入パターン | 🟡 推奨 |

### 即座に取り入れられること

1. Input/Output Protocol によるViewModel設計
2. Lensパターンでのテストデータ生成
3. スナップショットテストでのUI検証
4. Environment によるグローバル依存性管理

### このパターンを使うべきケース

- テストカバレッジを重視するプロジェクト
- チーム開発で品質管理が必要
- 長期メンテナンスが前提のアプリ
- A/Bテストを頻繁に行うサービス
