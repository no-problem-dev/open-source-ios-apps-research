# WidgetKit & App Intents 実装ガイド

**参考プロジェクト**:
- Backyard Birds (Apple): https://github.com/apple/sample-backyard-birds
- Clendar: https://github.com/nickolashkraus/Clendar
- Loop Habit Tracker: https://github.com/nickolashkraus/loop-habit-tracker

**実装参考度**: ⭐⭐⭐⭐⭐

---

## なぜ重要か

iOS 17でWidgetが**インタラクティブ**になった。App Intentsと組み合わせることで、ウィジェット上で直接操作が可能に。

| 学習ポイント | 重要度 | 理由 |
|-------------|--------|------|
| **Interactive Widget** | ⭐⭐⭐⭐⭐ | iOS 17の目玉機能 |
| **App Intents** | ⭐⭐⭐⭐⭐ | Siri・ショートカット統合 |
| **Timeline Provider** | ⭐⭐⭐⭐☆ | データ更新戦略 |
| **WidgetFamily** | ⭐⭐⭐⭐☆ | サイズ対応 |
| **ConfigurationIntent** | ⭐⭐⭐⭐☆ | ユーザー設定 |

---

## 1. 基本的なWidget

### Widget定義

```swift
import WidgetKit
import SwiftUI

struct MyWidget: Widget {
    let kind: String = "MyWidget"

    var body: some WidgetConfiguration {
        StaticConfiguration(
            kind: kind,
            provider: Provider()
        ) { entry in
            MyWidgetEntryView(entry: entry)
                .containerBackground(.fill.tertiary, for: .widget)
        }
        .configurationDisplayName("My Widget")
        .description("Shows your daily progress")
        .supportedFamilies([.systemSmall, .systemMedium, .systemLarge])
    }
}
```

### Entry と Provider

```swift
struct SimpleEntry: TimelineEntry {
    let date: Date
    let progress: Double
    let streakDays: Int
}

struct Provider: TimelineProvider {
    func placeholder(in context: Context) -> SimpleEntry {
        SimpleEntry(date: Date(), progress: 0.5, streakDays: 7)
    }

    func getSnapshot(in context: Context, completion: @escaping (SimpleEntry) -> ()) {
        let entry = SimpleEntry(date: Date(), progress: 0.75, streakDays: 14)
        completion(entry)
    }

    func getTimeline(in context: Context, completion: @escaping (Timeline<SimpleEntry>) -> ()) {
        var entries: [SimpleEntry] = []

        // 現在から1時間ごとにエントリを生成
        let currentDate = Date()
        for hourOffset in 0 ..< 24 {
            let entryDate = Calendar.current.date(byAdding: .hour, value: hourOffset, to: currentDate)!
            let entry = SimpleEntry(
                date: entryDate,
                progress: fetchProgress(),
                streakDays: fetchStreak()
            )
            entries.append(entry)
        }

        // 1時間後に再度タイムライン取得
        let timeline = Timeline(entries: entries, policy: .after(
            Calendar.current.date(byAdding: .hour, value: 1, to: currentDate)!
        ))
        completion(timeline)
    }

    private func fetchProgress() -> Double {
        // App Groupからデータ取得
        let defaults = UserDefaults(suiteName: "group.com.example.app")
        return defaults?.double(forKey: "progress") ?? 0
    }

    private func fetchStreak() -> Int {
        let defaults = UserDefaults(suiteName: "group.com.example.app")
        return defaults?.integer(forKey: "streak") ?? 0
    }
}
```

### Widget View

```swift
struct MyWidgetEntryView: View {
    @Environment(\.widgetFamily) var family
    var entry: Provider.Entry

    var body: some View {
        switch family {
        case .systemSmall:
            SmallWidgetView(entry: entry)
        case .systemMedium:
            MediumWidgetView(entry: entry)
        case .systemLarge:
            LargeWidgetView(entry: entry)
        default:
            SmallWidgetView(entry: entry)
        }
    }
}

struct SmallWidgetView: View {
    let entry: SimpleEntry

    var body: some View {
        VStack {
            Text("\(Int(entry.progress * 100))%")
                .font(.title)
                .fontWeight(.bold)

            CircularProgressView(progress: entry.progress)

            Text("\(entry.streakDays) days")
                .font(.caption)
        }
    }
}
```

---

## 2. インタラクティブWidget（iOS 17+）

### Button と Toggle

```swift
import AppIntents

// App Intent定義
struct ToggleCompletionIntent: AppIntent {
    static var title: LocalizedStringResource = "Toggle Completion"

    @Parameter(title: "Task ID")
    var taskID: String

    func perform() async throws -> some IntentResult {
        // タスクの完了状態を切り替え
        let defaults = UserDefaults(suiteName: "group.com.example.app")
        var completed = defaults?.stringArray(forKey: "completedTasks") ?? []

        if completed.contains(taskID) {
            completed.removeAll { $0 == taskID }
        } else {
            completed.append(taskID)
        }

        defaults?.set(completed, forKey: "completedTasks")

        // ウィジェットを更新
        WidgetCenter.shared.reloadTimelines(ofKind: "TaskWidget")

        return .result()
    }
}

// Widget View
struct TaskWidgetView: View {
    let task: TaskItem

    var body: some View {
        HStack {
            // インタラクティブボタン
            Button(intent: ToggleCompletionIntent(taskID: task.id)) {
                Image(systemName: task.isCompleted ? "checkmark.circle.fill" : "circle")
                    .foregroundStyle(task.isCompleted ? .green : .gray)
            }
            .buttonStyle(.plain)

            Text(task.title)
                .strikethrough(task.isCompleted)

            Spacer()
        }
    }
}
```

### Toggle の使用

```swift
struct HabitToggleIntent: AppIntent {
    static var title: LocalizedStringResource = "Toggle Habit"

    @Parameter(title: "Habit ID")
    var habitID: String

    @Parameter(title: "Is Completed")
    var isCompleted: Bool

    func perform() async throws -> some IntentResult {
        // 習慣の完了状態を保存
        saveHabitCompletion(habitID: habitID, completed: isCompleted)
        WidgetCenter.shared.reloadTimelines(ofKind: "HabitWidget")
        return .result()
    }
}

struct HabitWidgetView: View {
    let habit: Habit

    var body: some View {
        Toggle(
            isOn: habit.isCompletedToday,
            intent: HabitToggleIntent(habitID: habit.id, isCompleted: !habit.isCompletedToday)
        ) {
            Text(habit.name)
        }
        .toggleStyle(.button)
    }
}
```

---

## 3. App Intents 詳細

### 基本的なIntent

```swift
import AppIntents

struct AddTaskIntent: AppIntent {
    static var title: LocalizedStringResource = "Add Task"
    static var description = IntentDescription("Add a new task to your list")

    // パラメータ
    @Parameter(title: "Task Title")
    var title: String

    @Parameter(title: "Priority", default: .medium)
    var priority: TaskPriority

    @Parameter(title: "Due Date")
    var dueDate: Date?

    // 実行
    func perform() async throws -> some IntentResult & ReturnsValue<String> {
        let task = createTask(title: title, priority: priority, dueDate: dueDate)
        return .result(value: "Created: \(task.title)")
    }
}

// 優先度のenum
enum TaskPriority: String, AppEnum {
    case low, medium, high

    static var typeDisplayRepresentation: TypeDisplayRepresentation = "Priority"

    static var caseDisplayRepresentations: [TaskPriority: DisplayRepresentation] = [
        .low: "Low",
        .medium: "Medium",
        .high: "High"
    ]
}
```

### Entity と Query

```swift
// Siriで選択可能なエンティティ
struct TaskEntity: AppEntity {
    var id: String
    var title: String
    var isCompleted: Bool

    static var typeDisplayRepresentation: TypeDisplayRepresentation = "Task"

    var displayRepresentation: DisplayRepresentation {
        DisplayRepresentation(title: "\(title)")
    }

    // 検索クエリ
    static var defaultQuery = TaskQuery()
}

struct TaskQuery: EntityQuery {
    func entities(for identifiers: [String]) async throws -> [TaskEntity] {
        // IDでタスクを取得
        return fetchTasks(ids: identifiers).map { task in
            TaskEntity(id: task.id, title: task.title, isCompleted: task.isCompleted)
        }
    }

    func suggestedEntities() async throws -> [TaskEntity] {
        // 候補として表示するタスク
        return fetchRecentTasks().map { task in
            TaskEntity(id: task.id, title: task.title, isCompleted: task.isCompleted)
        }
    }
}

// Entityを使うIntent
struct CompleteTaskIntent: AppIntent {
    static var title: LocalizedStringResource = "Complete Task"

    @Parameter(title: "Task")
    var task: TaskEntity

    func perform() async throws -> some IntentResult {
        markTaskComplete(id: task.id)
        return .result()
    }
}
```

### App Shortcuts

```swift
struct MyAppShortcuts: AppShortcutsProvider {
    static var appShortcuts: [AppShortcut] {
        AppShortcut(
            intent: AddTaskIntent(),
            phrases: [
                "Add a task to \(.applicationName)",
                "Create a new task in \(.applicationName)",
                "New task in \(.applicationName)"
            ],
            shortTitle: "Add Task",
            systemImageName: "plus.circle"
        )

        AppShortcut(
            intent: ShowTodayIntent(),
            phrases: [
                "Show today's tasks in \(.applicationName)",
                "What's on my list in \(.applicationName)"
            ],
            shortTitle: "Today's Tasks",
            systemImageName: "calendar"
        )
    }
}
```

---

## 4. 設定可能なWidget

### AppIntentConfiguration

```swift
// 設定用Intent
struct SelectCategoryIntent: WidgetConfigurationIntent {
    static var title: LocalizedStringResource = "Select Category"
    static var description = IntentDescription("Choose which category to display")

    @Parameter(title: "Category")
    var category: CategoryEntity?
}

// Widget定義
struct ConfigurableWidget: Widget {
    let kind: String = "ConfigurableWidget"

    var body: some WidgetConfiguration {
        AppIntentConfiguration(
            kind: kind,
            intent: SelectCategoryIntent.self,
            provider: ConfigurableProvider()
        ) { entry in
            ConfigurableWidgetView(entry: entry)
                .containerBackground(.fill.tertiary, for: .widget)
        }
        .configurationDisplayName("Category Widget")
        .description("Shows tasks from a specific category")
        .supportedFamilies([.systemSmall, .systemMedium])
    }
}

// Provider
struct ConfigurableProvider: AppIntentTimelineProvider {
    func placeholder(in context: Context) -> CategoryEntry {
        CategoryEntry(date: Date(), category: nil, tasks: [])
    }

    func snapshot(for configuration: SelectCategoryIntent, in context: Context) async -> CategoryEntry {
        let tasks = await fetchTasks(for: configuration.category)
        return CategoryEntry(date: Date(), category: configuration.category, tasks: tasks)
    }

    func timeline(for configuration: SelectCategoryIntent, in context: Context) async -> Timeline<CategoryEntry> {
        let tasks = await fetchTasks(for: configuration.category)
        let entry = CategoryEntry(date: Date(), category: configuration.category, tasks: tasks)

        return Timeline(entries: [entry], policy: .after(
            Calendar.current.date(byAdding: .hour, value: 1, to: Date())!
        ))
    }
}
```

---

## 5. データ共有（App Group）

### UserDefaults共有

```swift
// メインアプリ
class DataManager {
    static let shared = DataManager()

    private let defaults = UserDefaults(suiteName: "group.com.example.app")!

    func saveProgress(_ progress: Double) {
        defaults.set(progress, forKey: "progress")
        // ウィジェットに更新を通知
        WidgetCenter.shared.reloadTimelines(ofKind: "ProgressWidget")
    }

    func getProgress() -> Double {
        defaults.double(forKey: "progress")
    }
}

// ウィジェット
struct WidgetProvider: TimelineProvider {
    private let defaults = UserDefaults(suiteName: "group.com.example.app")!

    func getTimeline(in context: Context, completion: @escaping (Timeline<Entry>) -> ()) {
        let progress = defaults.double(forKey: "progress")
        // ...
    }
}
```

### SwiftData共有

```swift
// 共有ModelContainer
let sharedModelContainer: ModelContainer = {
    let schema = Schema([Task.self])
    let config = ModelConfiguration(
        schema: schema,
        groupContainer: .identifier("group.com.example.app")
    )
    return try! ModelContainer(for: schema, configurations: config)
}()

// ウィジェットでの使用
struct WidgetProvider: TimelineProvider {
    func getTimeline(in context: Context, completion: @escaping (Timeline<Entry>) -> ()) {
        let context = sharedModelContainer.mainContext
        let tasks = try? context.fetch(FetchDescriptor<Task>())
        // ...
    }
}
```

---

## 6. タイムライン更新戦略

### 更新ポリシー

```swift
// 特定時刻に更新
let nextMidnight = Calendar.current.startOfDay(for: Date().addingTimeInterval(86400))
let timeline = Timeline(entries: entries, policy: .after(nextMidnight))

// 即座に次の更新を要求
let timeline = Timeline(entries: entries, policy: .atEnd)

// システム任せ（デフォルト）
let timeline = Timeline(entries: entries, policy: .never)
```

### アプリからの更新トリガー

```swift
// 特定ウィジェットを更新
WidgetCenter.shared.reloadTimelines(ofKind: "MyWidget")

// すべてのウィジェットを更新
WidgetCenter.shared.reloadAllTimelines()

// 現在の設定を取得
WidgetCenter.shared.getCurrentConfigurations { result in
    switch result {
    case .success(let infos):
        for info in infos {
            print("Widget: \(info.kind), Family: \(info.family)")
        }
    case .failure(let error):
        print("Error: \(error)")
    }
}
```

---

## まとめ：WidgetKit + App Intents

| 機能 | 用途 | 優先度 |
|------|------|--------|
| **Interactive Widget** | ウィジェット上での直接操作 | 🔴 必須（iOS 17+） |
| **App Intents** | Siri・ショートカット統合 | 🔴 必須 |
| **AppIntentConfiguration** | ユーザー設定可能Widget | 🟡 推奨 |
| **App Group** | アプリ・ウィジェット間データ共有 | 🟡 推奨 |
| **Timeline戦略** | 効率的な更新 | 🟢 参考 |

### 実装チェックリスト

1. App Group entitlement を追加
2. Widget Extension を作成
3. TimelineProvider を実装
4. インタラクティブ操作用の AppIntent を定義
5. App Shortcuts で Siri フレーズを登録
