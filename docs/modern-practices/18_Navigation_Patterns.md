# ナビゲーション設計パターン

**参考プロジェクト**:
- Ice Cubes
- Clean Architecture SwiftUI

**実装参考度**: ⭐⭐⭐⭐⭐

---

## なぜ重要か

iOS 16以降の**NavigationStack**は型安全なナビゲーションを実現。deep linkやプログラマティックナビゲーションが容易に。

| 学習ポイント | 重要度 | 理由 |
|-------------|--------|------|
| **NavigationStack** | ⭐⭐⭐⭐⭐ | 型安全ナビゲーション |
| **NavigationPath** | ⭐⭐⭐⭐⭐ | プログラマティック制御 |
| **navigationDestination** | ⭐⭐⭐⭐⭐ | 画面遷移定義 |
| **Coordinator** | ⭐⭐⭐⭐☆ | 大規模アプリ向け |
| **Deep Link** | ⭐⭐⭐⭐☆ | URL対応 |

---

## 1. NavigationStack基本

### 基本的な使用法

```swift
struct ContentView: View {
    var body: some View {
        NavigationStack {
            List {
                NavigationLink("Settings", value: Route.settings)
                NavigationLink("Profile", value: Route.profile("user123"))
            }
            .navigationTitle("Home")
            .navigationDestination(for: Route.self) { route in
                switch route {
                case .settings:
                    SettingsView()
                case .profile(let userId):
                    ProfileView(userId: userId)
                case .detail(let item):
                    DetailView(item: item)
                }
            }
        }
    }
}

enum Route: Hashable {
    case settings
    case profile(String)
    case detail(Item)
}
```

### NavigationPath

```swift
class NavigationManager: ObservableObject {
    @Published var path = NavigationPath()

    func push(_ route: Route) {
        path.append(route)
    }

    func pop() {
        guard !path.isEmpty else { return }
        path.removeLast()
    }

    func popToRoot() {
        path = NavigationPath()
    }

    func popTo(_ count: Int) {
        guard path.count >= count else { return }
        path.removeLast(count)
    }
}

struct RootView: View {
    @StateObject private var navigationManager = NavigationManager()

    var body: some View {
        NavigationStack(path: $navigationManager.path) {
            HomeView()
                .navigationDestination(for: Route.self) { route in
                    destinationView(for: route)
                }
        }
        .environmentObject(navigationManager)
    }

    @ViewBuilder
    private func destinationView(for route: Route) -> some View {
        switch route {
        case .settings:
            SettingsView()
        case .profile(let userId):
            ProfileView(userId: userId)
        case .detail(let item):
            DetailView(item: item)
        }
    }
}

// 使用例
struct ItemView: View {
    @EnvironmentObject var navigationManager: NavigationManager
    let item: Item

    var body: some View {
        VStack {
            Text(item.name)
            Button("View Details") {
                navigationManager.push(.detail(item))
            }
            Button("Back to Root") {
                navigationManager.popToRoot()
            }
        }
    }
}
```

---

## 2. Coordinator パターン

### Coordinator定義

```swift
@MainActor
class AppCoordinator: ObservableObject {
    @Published var path = NavigationPath()
    @Published var sheet: Sheet?
    @Published var fullScreenCover: FullScreenCover?

    enum Sheet: Identifiable {
        case addItem
        case editItem(Item)

        var id: String {
            switch self {
            case .addItem: "addItem"
            case .editItem(let item): "editItem_\(item.id)"
            }
        }
    }

    enum FullScreenCover: Identifiable {
        case camera
        case imageViewer(Image)

        var id: String {
            switch self {
            case .camera: "camera"
            case .imageViewer: "imageViewer"
            }
        }
    }

    // Navigation
    func navigate(to route: Route) {
        path.append(route)
    }

    func pop() {
        guard !path.isEmpty else { return }
        path.removeLast()
    }

    func popToRoot() {
        path = NavigationPath()
    }

    // Sheets
    func presentSheet(_ sheet: Sheet) {
        self.sheet = sheet
    }

    func dismissSheet() {
        sheet = nil
    }

    // Full Screen Covers
    func presentFullScreen(_ cover: FullScreenCover) {
        fullScreenCover = cover
    }

    func dismissFullScreen() {
        fullScreenCover = nil
    }
}
```

### Coordinator使用

```swift
struct CoordinatedApp: View {
    @StateObject private var coordinator = AppCoordinator()

    var body: some View {
        NavigationStack(path: $coordinator.path) {
            HomeView()
                .navigationDestination(for: Route.self) { route in
                    routeView(route)
                }
        }
        .sheet(item: $coordinator.sheet) { sheet in
            sheetView(sheet)
        }
        .fullScreenCover(item: $coordinator.fullScreenCover) { cover in
            fullScreenView(cover)
        }
        .environmentObject(coordinator)
    }

    @ViewBuilder
    private func routeView(_ route: Route) -> some View {
        switch route {
        case .settings:
            SettingsView()
        case .profile(let userId):
            ProfileView(userId: userId)
        case .detail(let item):
            DetailView(item: item)
        }
    }

    @ViewBuilder
    private func sheetView(_ sheet: AppCoordinator.Sheet) -> some View {
        switch sheet {
        case .addItem:
            AddItemView()
        case .editItem(let item):
            EditItemView(item: item)
        }
    }

    @ViewBuilder
    private func fullScreenView(_ cover: AppCoordinator.FullScreenCover) -> some View {
        switch cover {
        case .camera:
            CameraView()
        case .imageViewer(let image):
            ImageViewerView(image: image)
        }
    }
}
```

---

## 3. Deep Link対応

### URL解析

```swift
enum DeepLink {
    case home
    case item(id: String)
    case profile(userId: String)
    case settings

    init?(url: URL) {
        guard let components = URLComponents(url: url, resolvingAgainstBaseURL: true),
              let host = components.host else {
            return nil
        }

        let pathComponents = components.path.split(separator: "/").map(String.init)

        switch host {
        case "item":
            guard let id = pathComponents.first else { return nil }
            self = .item(id: id)
        case "profile":
            guard let userId = pathComponents.first else { return nil }
            self = .profile(userId: userId)
        case "settings":
            self = .settings
        default:
            self = .home
        }
    }

    func toRoute() -> Route? {
        switch self {
        case .home:
            return nil
        case .item(let id):
            return .detail(Item(id: id, name: ""))  // 実際にはfetchする
        case .profile(let userId):
            return .profile(userId)
        case .settings:
            return .settings
        }
    }
}
```

### App対応

```swift
@main
struct MyApp: App {
    @StateObject private var coordinator = AppCoordinator()

    var body: some Scene {
        WindowGroup {
            CoordinatedApp()
                .onOpenURL { url in
                    handleDeepLink(url)
                }
        }
    }

    private func handleDeepLink(_ url: URL) {
        guard let deepLink = DeepLink(url: url),
              let route = deepLink.toRoute() else {
            return
        }

        // ルートにいったん戻ってから遷移
        coordinator.popToRoot()
        DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
            coordinator.navigate(to: route)
        }
    }
}
```

---

## 4. TabView + Navigation

### Tab管理

```swift
enum Tab: Hashable {
    case home
    case search
    case profile
}

class TabCoordinator: ObservableObject {
    @Published var selectedTab: Tab = .home
    @Published var homePath = NavigationPath()
    @Published var searchPath = NavigationPath()
    @Published var profilePath = NavigationPath()

    func resetCurrentTab() {
        switch selectedTab {
        case .home:
            homePath = NavigationPath()
        case .search:
            searchPath = NavigationPath()
        case .profile:
            profilePath = NavigationPath()
        }
    }
}

struct MainTabView: View {
    @StateObject private var coordinator = TabCoordinator()

    var body: some View {
        TabView(selection: $coordinator.selectedTab) {
            NavigationStack(path: $coordinator.homePath) {
                HomeView()
                    .navigationDestination(for: Route.self) { route in
                        routeView(route)
                    }
            }
            .tabItem { Label("Home", systemImage: "house") }
            .tag(Tab.home)

            NavigationStack(path: $coordinator.searchPath) {
                SearchView()
                    .navigationDestination(for: Route.self) { route in
                        routeView(route)
                    }
            }
            .tabItem { Label("Search", systemImage: "magnifyingglass") }
            .tag(Tab.search)

            NavigationStack(path: $coordinator.profilePath) {
                ProfileView()
                    .navigationDestination(for: Route.self) { route in
                        routeView(route)
                    }
            }
            .tabItem { Label("Profile", systemImage: "person") }
            .tag(Tab.profile)
        }
        .environmentObject(coordinator)
    }
}
```

---

## 5. NavigationPath永続化

### Codable対応

```swift
extension NavigationPath {
    func encoded() -> Data? {
        try? JSONEncoder().encode(codable)
    }

    init(from data: Data) {
        if let codable = try? JSONDecoder().decode(
            NavigationPath.CodableRepresentation.self,
            from: data
        ) {
            self.init(codable)
        } else {
            self.init()
        }
    }
}

class PersistentNavigationManager: ObservableObject {
    @Published var path: NavigationPath {
        didSet {
            savePath()
        }
    }

    private let key = "navigationPath"

    init() {
        if let data = UserDefaults.standard.data(forKey: key) {
            path = NavigationPath(from: data)
        } else {
            path = NavigationPath()
        }
    }

    private func savePath() {
        if let data = path.encoded() {
            UserDefaults.standard.set(data, forKey: key)
        }
    }
}
```

---

## まとめ：ナビゲーションパターン

| パターン | 用途 | 優先度 |
|---------|------|--------|
| **NavigationStack** | 基本ナビゲーション | 🔴 必須 |
| **NavigationPath** | プログラマティック制御 | 🔴 必須 |
| **Coordinator** | 大規模アプリ | 🟡 推奨 |
| **Deep Link** | URL対応 | 🟡 推奨 |
| **状態永続化** | 復元機能 | 🟢 参考 |

### 設計原則

1. **Route型を定義**: 型安全なナビゲーション
2. **Coordinatorで一元管理**: 複雑なフローに対応
3. **Tab毎に独立Path**: 各タブの状態を保持
4. **Deep Link対応**: ユーザー体験向上
