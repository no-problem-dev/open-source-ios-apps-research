# Backyard Birds (Apple公式) - SwiftData & App Intents 完全ガイド

**リポジトリ**: https://github.com/apple/sample-backyard-birds
**実装参考度**: ⭐⭐⭐⭐⭐

---

## なぜ参考になるか

**Apple公式サンプル**として、iOS 17の新機能を「正しく」使う方法を示している。

| 学習ポイント | 重要度 | 理由 |
|-------------|--------|------|
| **SwiftData** | ⭐⭐⭐⭐⭐ | Apple推奨の実装パターン |
| **App Intents** | ⭐⭐⭐⭐⭐ | Siri/ショートカット統合 |
| **WidgetKit** | ⭐⭐⭐⭐⭐ | インタラクティブWidget |
| **TipKit** | ⭐⭐⭐⭐☆ | ユーザーオンボーディング |
| **マルチプラットフォーム** | ⭐⭐⭐⭐☆ | iOS/iPadOS/macOS/watchOS |

---

## 1. SwiftData 完全実装パターン

### モデル定義

```swift
// BackyardBirdsData/Backyard/Backyard.swift
import SwiftData

@Model
public final class Backyard {
    public var name: String
    public var creationDate: Date

    @Relationship(deleteRule: .cascade, inverse: \Bird.backyard)
    public var birds: [Bird] = []

    @Relationship(deleteRule: .cascade, inverse: \Plant.backyard)
    public var plants: [Plant] = []

    // 計算プロパティ（永続化されない）
    public var birdCount: Int { birds.count }

    public init(name: String) {
        self.name = name
        self.creationDate = Date()
    }
}

@Model
public final class Bird {
    public var species: BirdSpecies
    public var nickname: String?
    public var lastSeenDate: Date

    public var backyard: Backyard?

    public init(species: BirdSpecies, in backyard: Backyard) {
        self.species = species
        self.lastSeenDate = Date()
        self.backyard = backyard
    }
}
```

### カスタム型の永続化

```swift
// enumをSwiftDataで保存
public enum BirdSpecies: String, Codable, CaseIterable {
    case robin = "robin"
    case blueJay = "blue_jay"
    case cardinal = "cardinal"
    case sparrow = "sparrow"

    public var displayName: String {
        switch self {
        case .robin: "ロビン"
        case .blueJay: "アオカケス"
        case .cardinal: "ショウジョウコウカンチョウ"
        case .sparrow: "スズメ"
        }
    }
}
```

### ModelContainer設定

```swift
// App起動時の設定
@main
struct BackyardBirdsApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(sharedModelContainer)
    }
}

// 共有コンテナ（App Groupでウィジェットと共有）
let sharedModelContainer: ModelContainer = {
    let schema = Schema([Backyard.self, Bird.self, Plant.self])

    let modelConfiguration = ModelConfiguration(
        schema: schema,
        isStoredInMemoryOnly: false,
        groupContainer: .identifier("group.com.apple.backyard-birds"),
        cloudKitContainerIdentifier: nil
    )

    do {
        return try ModelContainer(
            for: schema,
            configurations: [modelConfiguration]
        )
    } catch {
        fatalError("Could not create ModelContainer: \(error)")
    }
}()
```

### @Query の高度な使用

```swift
struct BirdListView: View {
    // 基本的なクエリ
    @Query(sort: \Bird.lastSeenDate, order: .reverse)
    private var birds: [Bird]

    // フィルタ付きクエリ
    @Query(
        filter: #Predicate<Bird> { bird in
            bird.species == .robin
        },
        sort: \Bird.lastSeenDate
    )
    private var robins: [Bird]

    // 動的フィルタ（init時に設定）
    @Query private var filteredBirds: [Bird]

    init(backyard: Backyard) {
        let backyardID = backyard.persistentModelID
        _filteredBirds = Query(
            filter: #Predicate<Bird> { bird in
                bird.backyard?.persistentModelID == backyardID
            }
        )
    }

    var body: some View {
        List(birds) { bird in
            BirdRow(bird: bird)
        }
    }
}
```

---

## 2. App Intents - Siriショートカット統合

### 基本的なIntent定義

```swift
// BackyardBirdsData/Intents/AddBirdIntent.swift
import AppIntents

struct AddBirdIntent: AppIntent {
    static var title: LocalizedStringResource = "Add Bird"
    static var description = IntentDescription("Add a bird to your backyard")

    // パラメータ
    @Parameter(title: "Species")
    var species: BirdSpeciesEntity

    @Parameter(title: "Backyard")
    var backyard: BackyardEntity

    // 実行
    func perform() async throws -> some IntentResult {
        let modelContext = ModelContext(sharedModelContainer)

        // Entityから実際のモデルを取得
        guard let backyardModel = try? modelContext.fetch(
            FetchDescriptor<Backyard>(
                predicate: #Predicate { $0.persistentModelID == backyard.id }
            )
        ).first else {
            throw IntentError.backyardNotFound
        }

        let bird = Bird(species: species.species, in: backyardModel)
        modelContext.insert(bird)
        try modelContext.save()

        return .result(value: "Added \(species.displayName)")
    }
}
```

### EntityとQuery

```swift
// BackyardEntity - Siriで選択可能なエンティティ
struct BackyardEntity: AppEntity {
    var id: PersistentIdentifier
    var name: String

    static var typeDisplayRepresentation: TypeDisplayRepresentation {
        TypeDisplayRepresentation(name: "Backyard")
    }

    var displayRepresentation: DisplayRepresentation {
        DisplayRepresentation(title: "\(name)")
    }

    // 検索クエリ
    static var defaultQuery = BackyardQuery()
}

struct BackyardQuery: EntityQuery {
    func entities(for identifiers: [PersistentIdentifier]) async throws -> [BackyardEntity] {
        let context = ModelContext(sharedModelContainer)
        let backyards = try context.fetch(FetchDescriptor<Backyard>())
        return backyards
            .filter { identifiers.contains($0.persistentModelID) }
            .map { BackyardEntity(id: $0.persistentModelID, name: $0.name) }
    }

    func suggestedEntities() async throws -> [BackyardEntity] {
        let context = ModelContext(sharedModelContainer)
        let backyards = try context.fetch(FetchDescriptor<Backyard>())
        return backyards.map { BackyardEntity(id: $0.persistentModelID, name: $0.name) }
    }
}
```

### App Shortcuts

```swift
// アプリショートカット定義
struct BackyardBirdsShortcuts: AppShortcutsProvider {
    static var appShortcuts: [AppShortcut] {
        AppShortcut(
            intent: AddBirdIntent(),
            phrases: [
                "Add a bird to \(.applicationName)",
                "Log a bird sighting in \(.applicationName)"
            ],
            shortTitle: "Add Bird",
            systemImageName: "bird"
        )
    }
}
```

---

## 3. インタラクティブWidget

### Widget定義

```swift
// BackyardBirdsWidget/BackyardBirdsWidget.swift
import WidgetKit
import SwiftUI
import AppIntents

struct BackyardBirdsWidget: Widget {
    var body: some WidgetConfiguration {
        AppIntentConfiguration(
            kind: "BackyardBirds",
            intent: SelectBackyardIntent.self,
            provider: Provider()
        ) { entry in
            BackyardWidgetView(entry: entry)
                .containerBackground(.fill.tertiary, for: .widget)
        }
        .configurationDisplayName("Backyard Birds")
        .description("See your backyard activity")
        .supportedFamilies([.systemSmall, .systemMedium, .systemLarge])
    }
}
```

### インタラクティブボタン

```swift
struct BackyardWidgetView: View {
    var entry: Provider.Entry

    var body: some View {
        VStack {
            Text(entry.backyard.name)
                .font(.headline)

            Text("\(entry.backyard.birdCount) birds")

            // インタラクティブボタン（iOS 17+）
            Button(intent: RefreshBackyardIntent(backyardID: entry.backyard.id)) {
                Label("Refresh", systemImage: "arrow.clockwise")
            }
            .buttonStyle(.borderedProminent)
        }
    }
}

// ボタン用Intent
struct RefreshBackyardIntent: AppIntent {
    static var title: LocalizedStringResource = "Refresh Backyard"

    @Parameter(title: "Backyard ID")
    var backyardID: String

    func perform() async throws -> some IntentResult {
        // データ更新処理
        WidgetCenter.shared.reloadTimelines(ofKind: "BackyardBirds")
        return .result()
    }
}
```

### Timeline Provider

```swift
struct Provider: AppIntentTimelineProvider {
    func placeholder(in context: Context) -> Entry {
        Entry(date: Date(), backyard: .placeholder)
    }

    func snapshot(for configuration: SelectBackyardIntent, in context: Context) async -> Entry {
        await entry(for: configuration)
    }

    func timeline(for configuration: SelectBackyardIntent, in context: Context) async -> Timeline<Entry> {
        let entry = await entry(for: configuration)

        // 1時間ごとに更新
        let nextUpdate = Calendar.current.date(byAdding: .hour, value: 1, to: Date())!
        return Timeline(entries: [entry], policy: .after(nextUpdate))
    }

    private func entry(for configuration: SelectBackyardIntent) async -> Entry {
        // SwiftDataから取得
        let context = ModelContext(sharedModelContainer)
        // ...
    }
}
```

---

## 4. TipKit - ユーザーガイダンス

### Tip定義

```swift
import TipKit

struct AddBirdTip: Tip {
    var title: Text {
        Text("Add Your First Bird")
    }

    var message: Text? {
        Text("Tap the + button to log a bird sighting")
    }

    var image: Image? {
        Image(systemName: "bird")
    }

    // 表示条件
    var rules: [Rule] {
        // 鳥が0匹のときだけ表示
        #Rule(Self.$birdCount) { $0 == 0 }
    }

    // パラメータ
    @Parameter
    static var birdCount: Int = 0
}
```

### Tipの表示

```swift
struct BirdListView: View {
    let addBirdTip = AddBirdTip()

    var body: some View {
        List {
            // インライン表示
            TipView(addBirdTip)

            ForEach(birds) { bird in
                BirdRow(bird: bird)
            }
        }
        .toolbar {
            ToolbarItem {
                Button(action: addBird) {
                    Image(systemName: "plus")
                }
                // ポップオーバー表示
                .popoverTip(addBirdTip)
            }
        }
    }

    func addBird() {
        // Tipを非表示にする
        addBirdTip.invalidate(reason: .actionPerformed)
    }
}
```

### TipKit設定

```swift
@main
struct BackyardBirdsApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
                .task {
                    try? Tips.configure([
                        .displayFrequency(.immediate),
                        .datastoreLocation(.applicationDefault)
                    ])
                }
        }
    }
}
```

---

## 5. App Group でデータ共有

### 設定

```swift
// メインアプリとウィジェットでデータを共有

// 1. Xcode で App Group を追加
// Target → Signing & Capabilities → + App Groups
// group.com.example.backyardbirds

// 2. ModelContainer で groupContainer を指定
let modelConfiguration = ModelConfiguration(
    schema: schema,
    groupContainer: .identifier("group.com.example.backyardbirds")
)

// 3. UserDefaults も共有可能
let sharedDefaults = UserDefaults(suiteName: "group.com.example.backyardbirds")
```

---

## まとめ：Backyard Birdsから学ぶべきこと

| 技術 | 実装ポイント | 優先度 |
|------|-------------|--------|
| **SwiftData** | @Model, @Query, App Group共有 | 🔴 必須 |
| **App Intents** | Siri統合、ショートカット | 🔴 必須 |
| **WidgetKit** | インタラクティブWidget | 🟡 推奨 |
| **TipKit** | ユーザーオンボーディング | 🟢 参考 |

### 即座に取り入れられること

1. SwiftData の @Model, @Relationship 定義
2. @Query の動的フィルタリング
3. App Intents でSiriショートカット対応
4. インタラクティブWidgetの実装
5. App Groupでウィジェットとデータ共有

### Apple公式サンプルを読むメリット

- **正しい実装パターン**が学べる
- **非推奨のコード**が含まれない
- **最新iOS**の機能を網羅
- **ドキュメント**と連携している
