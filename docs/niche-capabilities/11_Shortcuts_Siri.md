# Shortcuts / Siri連携

**驚きポイント**: アプリ機能をショートカットAppで自動化

## 参考プロジェクト

| プロジェクト | Stars | 特徴 |
|-------------|-------|------|
| [Actions](https://github.com/sindresorhus/Actions) | 2,917 | 100+のショートカットアクション |
| [Siri Shortcut Example](https://github.com/CoyoteLab/Studies-Siri-Shortcut-iOS-13) | 20 | 基本実装例 |

---

## 1. App Intents（iOS 16+）

### シンプルなIntent

```swift
import AppIntents

struct OpenSettingsIntent: AppIntent {
    static var title: LocalizedStringResource = "設定を開く"
    static var description = IntentDescription("アプリの設定画面を開きます")

    // Siri起動をサポート
    static var openAppWhenRun: Bool = true

    func perform() async throws -> some IntentResult {
        // 設定画面を開く処理
        return .result()
    }
}
```

### パラメータ付きIntent

```swift
struct CreateNoteIntent: AppIntent {
    static var title: LocalizedStringResource = "ノートを作成"

    @Parameter(title: "タイトル")
    var noteTitle: String

    @Parameter(title: "内容", default: "")
    var content: String

    @Parameter(title: "フォルダ")
    var folder: FolderEntity?

    func perform() async throws -> some IntentResult & ReturnsValue<NoteEntity> {
        let note = try await NoteService.shared.create(
            title: noteTitle,
            content: content,
            folder: folder
        )

        return .result(value: NoteEntity(note: note))
    }
}
```

### Entity定義

```swift
struct NoteEntity: AppEntity {
    static var typeDisplayRepresentation = TypeDisplayRepresentation(name: "ノート")

    static var defaultQuery = NoteQuery()

    var id: UUID
    var title: String

    var displayRepresentation: DisplayRepresentation {
        DisplayRepresentation(title: "\(title)")
    }
}

struct NoteQuery: EntityQuery {
    func entities(for identifiers: [UUID]) async throws -> [NoteEntity] {
        // IDでノートを取得
        return []
    }

    func suggestedEntities() async throws -> [NoteEntity] {
        // 最近のノートを提案
        return []
    }
}
```

---

## 2. ショートカット提供

```swift
struct AppShortcuts: AppShortcutsProvider {
    static var appShortcuts: [AppShortcut] {
        AppShortcut(
            intent: CreateNoteIntent(),
            phrases: [
                "新しいノートを\(.applicationName)で作成",
                "\(.applicationName)でメモを取る"
            ],
            shortTitle: "ノート作成",
            systemImageName: "note.text"
        )

        AppShortcut(
            intent: SearchNotesIntent(),
            phrases: [
                "\(.applicationName)で検索",
                "\(.applicationName)のノートを探す"
            ],
            shortTitle: "検索",
            systemImageName: "magnifyingglass"
        )
    }
}
```

---

## 3. ウィジェットとの連携

```swift
import WidgetKit

struct NoteWidget: Widget {
    var body: some WidgetConfiguration {
        AppIntentConfiguration(
            kind: "NoteWidget",
            intent: SelectNoteIntent.self,
            provider: NoteProvider()
        ) { entry in
            NoteWidgetView(entry: entry)
        }
    }
}

struct SelectNoteIntent: WidgetConfigurationIntent {
    static var title: LocalizedStringResource = "ノートを選択"

    @Parameter(title: "ノート")
    var note: NoteEntity?
}
```

---

## 4. Focus Filter

```swift
import AppIntents

struct NoteFocusFilter: SetFocusFilterIntent {
    static var title: LocalizedStringResource = "ノートフィルター"

    @Parameter(title: "表示するフォルダ")
    var folder: FolderEntity?

    @Parameter(title: "通知を非表示", default: false)
    var hideNotifications: Bool

    var displayRepresentation: DisplayRepresentation {
        DisplayRepresentation(title: "フォルダ: \(folder?.name ?? "すべて")")
    }

    func perform() async throws -> some IntentResult {
        // フォーカスモード設定を適用
        await FocusFilterManager.shared.apply(
            folder: folder,
            hideNotifications: hideNotifications
        )
        return .result()
    }
}
```

---

## 5. Actions風：多機能アクション

```swift
// テキスト操作
struct ReverseTextIntent: AppIntent {
    static var title: LocalizedStringResource = "テキストを逆順に"

    @Parameter(title: "テキスト")
    var text: String

    func perform() async throws -> some IntentResult & ReturnsValue<String> {
        let reversed = String(text.reversed())
        return .result(value: reversed)
    }
}

// デバイス情報
struct GetDeviceInfoIntent: AppIntent {
    static var title: LocalizedStringResource = "デバイス情報を取得"

    func perform() async throws -> some IntentResult & ReturnsValue<String> {
        let info = """
        デバイス: \(UIDevice.current.model)
        システム: \(UIDevice.current.systemVersion)
        名前: \(UIDevice.current.name)
        """
        return .result(value: info)
    }
}

// クリップボード操作
struct GetClipboardIntent: AppIntent {
    static var title: LocalizedStringResource = "クリップボードを取得"

    func perform() async throws -> some IntentResult & ReturnsValue<String> {
        let text = UIPasteboard.general.string ?? ""
        return .result(value: text)
    }
}

struct SetClipboardIntent: AppIntent {
    static var title: LocalizedStringResource = "クリップボードに設定"

    @Parameter(title: "テキスト")
    var text: String

    func perform() async throws -> some IntentResult {
        UIPasteboard.general.string = text
        return .result()
    }
}
```

---

## 6. 旧SiriKit（iOS 10-15）

```swift
import Intents

class IntentHandler: INExtension {
    override func handler(for intent: INIntent) -> Any {
        switch intent {
        case is INSendMessageIntent:
            return SendMessageIntentHandler()
        case is INSearchForMessagesIntent:
            return SearchMessagesIntentHandler()
        default:
            return self
        }
    }
}

class SendMessageIntentHandler: NSObject, INSendMessageIntentHandling {
    func handle(intent: INSendMessageIntent) async -> INSendMessageIntentResponse {
        // メッセージ送信処理
        return INSendMessageIntentResponse(code: .success, userActivity: nil)
    }
}
```

---

## まとめ

| API | iOS | 用途 |
|-----|-----|------|
| App Intents | 16+ | 新しいショートカット |
| AppShortcutsProvider | 16+ | Siri音声起動 |
| Focus Filter | 16+ | 集中モード連携 |
| SiriKit | 10+ | 旧Siri連携 |

### 実装のヒント

1. **フレーズ多様性**: 複数の言い回しを登録
2. **エラーハンドリング**: IntentErrorで適切なメッセージ
3. **パラメータ検証**: 必須/オプションを明確に
4. **プレビュー**: ショートカットAppでテスト
