# SwiftData 実装パターン集

**参考プロジェクト**:
- Apple公式サンプル: Backyard Birds, Food Truck
- Ice Cubes: https://github.com/Dimillian/IceCubesApp
- Chronicling America: https://github.com/TheOfficialSK/ChroniclingAmerica

**実装参考度**: ⭐⭐⭐⭐⭐

---

## なぜ重要か

SwiftDataはiOS 17で導入された**Core Dataの後継**。SwiftUIとシームレスに統合され、宣言的にデータモデルを定義できる。

| 学習ポイント | 重要度 | 理由 |
|-------------|--------|------|
| **@Model** | ⭐⭐⭐⭐⭐ | 永続化モデルの定義 |
| **@Query** | ⭐⭐⭐⭐⭐ | 宣言的データフェッチ |
| **@Relationship** | ⭐⭐⭐⭐⭐ | リレーション管理 |
| **ModelContainer** | ⭐⭐⭐⭐☆ | データストア設定 |
| **マイグレーション** | ⭐⭐⭐⭐☆ | スキーマ変更対応 |

---

## 1. 基本的なモデル定義

### @Model の使用

```swift
import SwiftData

@Model
final class Item {
    var title: String
    var content: String
    var createdAt: Date
    var isPinned: Bool

    // 計算プロパティ（永続化されない）
    var formattedDate: String {
        createdAt.formatted(date: .abbreviated, time: .shortened)
    }

    init(title: String, content: String = "", isPinned: Bool = false) {
        self.title = title
        self.content = content
        self.createdAt = Date()
        self.isPinned = isPinned
    }
}
```

### カスタム型の永続化

```swift
// Codable enumは自動的に永続化可能
@Model
final class Task {
    var title: String
    var priority: Priority
    var status: Status

    enum Priority: String, Codable, CaseIterable {
        case low, medium, high

        var color: Color {
            switch self {
            case .low: .green
            case .medium: .orange
            case .high: .red
            }
        }
    }

    enum Status: String, Codable {
        case todo, inProgress, done
    }
}

// カスタム型もCodableで対応
@Model
final class Location {
    var name: String
    var coordinate: Coordinate

    struct Coordinate: Codable {
        var latitude: Double
        var longitude: Double
    }
}
```

---

## 2. リレーション定義

### 一対多

```swift
@Model
final class Folder {
    var name: String
    var createdAt: Date

    // 一対多（Folderが削除されたらNotesも削除）
    @Relationship(deleteRule: .cascade, inverse: \Note.folder)
    var notes: [Note] = []

    var noteCount: Int { notes.count }
}

@Model
final class Note {
    var title: String
    var content: String

    // 多対一
    var folder: Folder?
}
```

### 多対多

```swift
@Model
final class Tag {
    var name: String
    var color: String

    @Relationship(inverse: \Article.tags)
    var articles: [Article] = []
}

@Model
final class Article {
    var title: String
    var content: String
    var tags: [Tag] = []
}
```

### 削除ルール

```swift
@Model
final class Parent {
    // cascade: 親削除時に子も削除
    @Relationship(deleteRule: .cascade)
    var children: [Child] = []

    // nullify: 親削除時に子の参照をnilに
    @Relationship(deleteRule: .nullify)
    var optionalChildren: [Child] = []

    // deny: 子がある場合は親を削除不可
    @Relationship(deleteRule: .deny)
    var requiredChildren: [Child] = []

    // noAction: 何もしない（デフォルト）
    @Relationship(deleteRule: .noAction)
    var unmanaged: [Child] = []
}
```

---

## 3. @Query パターン

### 基本的なクエリ

```swift
struct ItemListView: View {
    // 全件取得
    @Query private var items: [Item]

    // ソート
    @Query(sort: \Item.createdAt, order: .reverse)
    private var sortedItems: [Item]

    // フィルタ
    @Query(filter: #Predicate<Item> { $0.isPinned })
    private var pinnedItems: [Item]

    var body: some View {
        List(items) { item in
            ItemRow(item: item)
        }
    }
}
```

### 複合条件

```swift
struct TaskListView: View {
    @Query(
        filter: #Predicate<Task> { task in
            task.status != .done && task.priority == .high
        },
        sort: [
            SortDescriptor(\Task.priority, order: .reverse),
            SortDescriptor(\Task.createdAt)
        ]
    )
    private var urgentTasks: [Task]
}
```

### 動的フィルタ

```swift
struct FolderDetailView: View {
    let folder: Folder

    @Query private var notes: [Note]

    init(folder: Folder) {
        self.folder = folder

        // 動的にフィルタを設定
        let folderID = folder.persistentModelID
        _notes = Query(
            filter: #Predicate<Note> { note in
                note.folder?.persistentModelID == folderID
            },
            sort: \Note.createdAt,
            order: .reverse
        )
    }

    var body: some View {
        List(notes) { note in
            NoteRow(note: note)
        }
    }
}
```

### アニメーション付きクエリ

```swift
struct AnimatedListView: View {
    @Query(sort: \Item.createdAt, animation: .smooth)
    private var items: [Item]

    var body: some View {
        List(items) { item in
            ItemRow(item: item)
        }
    }
}
```

---

## 4. ModelContainer設定

### 基本設定

```swift
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(for: [Item.self, Folder.self, Note.self])
    }
}
```

### カスタム設定

```swift
@main
struct MyApp: App {
    var sharedModelContainer: ModelContainer = {
        let schema = Schema([
            Item.self,
            Folder.self,
            Note.self
        ])

        let modelConfiguration = ModelConfiguration(
            schema: schema,
            isStoredInMemoryOnly: false,
            allowsSave: true
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

    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(sharedModelContainer)
    }
}
```

### App Groupでウィジェットと共有

```swift
let sharedModelContainer: ModelContainer = {
    let schema = Schema([Item.self])

    let modelConfiguration = ModelConfiguration(
        schema: schema,
        isStoredInMemoryOnly: false,
        groupContainer: .identifier("group.com.example.myapp")
    )

    return try! ModelContainer(
        for: schema,
        configurations: [modelConfiguration]
    )
}()
```

---

## 5. ModelContext操作

### CRUD操作

```swift
struct ContentView: View {
    @Environment(\.modelContext) private var modelContext
    @Query private var items: [Item]

    var body: some View {
        List {
            ForEach(items) { item in
                ItemRow(item: item)
            }
            .onDelete(perform: deleteItems)
        }
        .toolbar {
            Button(action: addItem) {
                Label("Add", systemImage: "plus")
            }
        }
    }

    // Create
    private func addItem() {
        let newItem = Item(title: "New Item")
        modelContext.insert(newItem)
        // 自動保存（明示的save不要）
    }

    // Update
    private func updateItem(_ item: Item) {
        item.title = "Updated"
        item.isPinned.toggle()
        // 変更は自動追跡、自動保存
    }

    // Delete
    private func deleteItems(at offsets: IndexSet) {
        for index in offsets {
            modelContext.delete(items[index])
        }
    }
}
```

### FetchDescriptor（手動フェッチ）

```swift
func searchItems(keyword: String) -> [Item] {
    let descriptor = FetchDescriptor<Item>(
        predicate: #Predicate { item in
            item.title.contains(keyword) ||
            item.content.contains(keyword)
        },
        sortBy: [SortDescriptor(\Item.createdAt, order: .reverse)]
    )
    descriptor.fetchLimit = 20

    do {
        return try modelContext.fetch(descriptor)
    } catch {
        return []
    }
}
```

### バッチ操作

```swift
// 全件削除
func deleteAllItems() throws {
    try modelContext.delete(model: Item.self)
}

// 条件削除
func deleteOldItems() throws {
    let oneMonthAgo = Calendar.current.date(byAdding: .month, value: -1, to: Date())!

    try modelContext.delete(
        model: Item.self,
        where: #Predicate { $0.createdAt < oneMonthAgo }
    )
}
```

---

## 6. マイグレーション

### スキーマバージョン管理

```swift
enum SchemaV1: VersionedSchema {
    static var versionIdentifier = Schema.Version(1, 0, 0)
    static var models: [any PersistentModel.Type] {
        [ItemV1.self]
    }

    @Model
    final class ItemV1 {
        var title: String
        var createdAt: Date
    }
}

enum SchemaV2: VersionedSchema {
    static var versionIdentifier = Schema.Version(2, 0, 0)
    static var models: [any PersistentModel.Type] {
        [Item.self]  // 新しいモデル
    }
}

enum MigrationPlan: SchemaMigrationPlan {
    static var schemas: [any VersionedSchema.Type] {
        [SchemaV1.self, SchemaV2.self]
    }

    static var stages: [MigrationStage] {
        [migrateV1toV2]
    }

    static let migrateV1toV2 = MigrationStage.custom(
        fromVersion: SchemaV1.self,
        toVersion: SchemaV2.self
    ) { context in
        // カスタムマイグレーションロジック
        let items = try context.fetch(FetchDescriptor<SchemaV1.ItemV1>())
        for oldItem in items {
            // 変換処理
        }
        try context.save()
    }
}

// 使用
let container = try ModelContainer(
    for: Schema.current,
    migrationPlan: MigrationPlan.self
)
```

---

## 7. プレビューでの使用

### プレビュー用コンテナ

```swift
struct ItemRow: View {
    let item: Item

    var body: some View {
        VStack(alignment: .leading) {
            Text(item.title)
                .font(.headline)
            Text(item.formattedDate)
                .font(.caption)
        }
    }
}

#Preview {
    let config = ModelConfiguration(isStoredInMemoryOnly: true)
    let container = try! ModelContainer(for: Item.self, configurations: config)

    // サンプルデータ
    let sampleItem = Item(title: "Sample", content: "Preview content")
    container.mainContext.insert(sampleItem)

    return ItemRow(item: sampleItem)
        .modelContainer(container)
}
```

---

## まとめ：SwiftDataパターン

| パターン | 用途 | 優先度 |
|---------|------|--------|
| **@Model + @Query** | 基本的なCRUD | 🔴 必須 |
| **@Relationship** | データ関連付け | 🔴 必須 |
| **動的@Query** | 条件付きフェッチ | 🟡 推奨 |
| **App Group共有** | ウィジェット対応 | 🟡 推奨 |
| **マイグレーション** | スキーマ更新 | 🟢 参考 |

### Core Dataからの移行チェックリスト

1. `NSManagedObject` → `@Model`
2. `@FetchRequest` → `@Query`
3. `NSManagedObjectContext` → `ModelContext`
4. `NSPersistentContainer` → `ModelContainer`
5. `.xcdatamodel` → Swiftコード
