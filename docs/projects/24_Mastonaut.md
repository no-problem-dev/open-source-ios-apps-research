# Mastonaut

**スター**: 1,400+ ⭐
**リポジトリ**: https://github.com/chucker/Mastonaut
**ライセンス**: GPL-3.0

---

## 概要

Mastonautは、macOS向けのネイティブMastodonクライアント。AppKitで構築され、マルチカラムレイアウト、複数アカウント対応、キーボードナビゲーションなど、デスクトップアプリとしての使いやすさを重視。

---

## 主要機能

| 機能 | 説明 |
|------|------|
| マルチカラム | TweetDeck風レイアウト |
| マルチアカウント | 複数インスタンス対応 |
| 通知 | macOS通知統合 |
| キーボード | 完全キーボード操作 |
| ドラッグ&ドロップ | 画像添付 |
| ダークモード | システム連動 |

---

## 技術スタック

| 項目 | 技術 |
|------|------|
| 言語 | Swift |
| UI | AppKit |
| ネットワーク | URLSession |
| ストレージ | Core Data |
| 認証 | OAuth 2.0 |
| 通知 | UserNotifications |

---

## アーキテクチャ

```
Mastonaut/
├── Mastonaut/               # メインアプリ
│   ├── View Controllers/    # NSViewController
│   │   ├── Timeline/        # タイムライン
│   │   ├── Compose/         # 投稿作成
│   │   └── Profile/         # プロフィール
│   ├── Views/               # NSView サブクラス
│   ├── Models/              # データモデル
│   └── Networking/          # API クライアント
├── MastodonKit/             # Mastodon API ライブラリ
└── CoreTootin/              # 共有フレームワーク
```

---

## 学習価値

### macOSネイティブアプリ

| 分野 | 学習ポイント |
|------|-------------|
| **AppKit** | ネイティブmacOS UI |
| **マルチカラム** | NSSplitView活用 |
| **キーボード** | レスポンダーチェーン |
| **OAuth** | 認証フロー |
| **ドラッグ&ドロップ** | NSPasteboard |

### コードパターン

**マルチカラムレイアウト**:
```swift
// TweetDeck風マルチカラム
class ColumnsViewController: NSViewController {
    private var columns: [ColumnViewController] = []
    private let splitView = NSSplitView()

    override func loadView() {
        splitView.isVertical = true
        splitView.dividerStyle = .thin
        view = splitView
    }

    func addColumn(type: ColumnType) {
        let column = ColumnViewController(type: type)
        columns.append(column)

        addChild(column)
        splitView.addArrangedSubview(column.view)

        column.view.widthAnchor.constraint(greaterThanOrEqualToConstant: 300).isActive = true
    }

    func removeColumn(at index: Int) {
        let column = columns.remove(at: index)
        column.view.removeFromSuperview()
        column.removeFromParent()
    }
}

class ColumnViewController: NSViewController {
    let type: ColumnType
    private let tableView = NSTableView()
    private var statuses: [Status] = []

    init(type: ColumnType) {
        self.type = type
        super.init(nibName: nil, bundle: nil)
    }

    override func loadView() {
        let scrollView = NSScrollView()
        scrollView.documentView = tableView
        view = scrollView

        tableView.delegate = self
        tableView.dataSource = self
    }
}
```

**キーボードナビゲーション**:
```swift
// レスポンダーチェーン活用
class TimelineViewController: NSViewController {
    override var acceptsFirstResponder: Bool { true }

    override func keyDown(with event: NSEvent) {
        switch event.charactersIgnoringModifiers {
        case "j": // 次のステータス
            selectNextStatus()
        case "k": // 前のステータス
            selectPreviousStatus()
        case "r": // 返信
            replyToSelectedStatus()
        case "f": // お気に入り
            favoriteSelectedStatus()
        case "b": // ブースト
            boostSelectedStatus()
        default:
            super.keyDown(with: event)
        }
    }

    @IBAction func compose(_ sender: Any?) {
        let composeWindow = ComposeWindowController()
        composeWindow.showWindow(nil)
    }
}

// メニューバーキーボードショートカット
extension AppDelegate {
    func setupKeyboardShortcuts() {
        NSEvent.addLocalMonitorForEvents(matching: .keyDown) { event in
            if event.modifierFlags.contains(.command) {
                switch event.charactersIgnoringModifiers {
                case "n": // 新規投稿
                    self.newPost(nil)
                    return nil
                default:
                    break
                }
            }
            return event
        }
    }
}
```

**OAuth認証フロー**:
```swift
// Mastodon OAuth 2.0
class AuthenticationService {
    func authenticate(instance: String) async throws -> Account {
        // 1. アプリ登録
        let app = try await registerApp(on: instance)

        // 2. 認証URL生成
        let authURL = buildAuthURL(instance: instance, clientId: app.clientId)

        // 3. ブラウザで認証
        let authCode = try await openBrowserAndGetCode(authURL)

        // 4. トークン取得
        let token = try await getAccessToken(
            instance: instance,
            clientId: app.clientId,
            clientSecret: app.clientSecret,
            code: authCode
        )

        // 5. アカウント情報取得
        let account = try await verifyCredentials(instance: instance, token: token)

        // 6. Keychainに保存
        try KeychainService.save(token: token, for: account.id)

        return account
    }

    private func openBrowserAndGetCode(_ url: URL) async throws -> String {
        NSWorkspace.shared.open(url)

        // コールバック待機
        return try await withCheckedThrowingContinuation { continuation in
            callbackHandler = { code in
                continuation.resume(returning: code)
            }
        }
    }
}
```

**ドラッグ&ドロップ**:
```swift
// 画像ドラッグ&ドロップ
class ComposeView: NSView {
    override init(frame: CGRect) {
        super.init(frame: frame)
        registerForDraggedTypes([.fileURL, .png, .tiff])
    }

    override func draggingEntered(_ sender: NSDraggingInfo) -> NSDragOperation {
        if hasImageFiles(sender) {
            return .copy
        }
        return []
    }

    override func performDragOperation(_ sender: NSDraggingInfo) -> Bool {
        guard let urls = sender.draggingPasteboard.readObjects(
            forClasses: [NSURL.self],
            options: [.urlReadingFileURLsOnly: true]
        ) as? [URL] else {
            return false
        }

        let imageURLs = urls.filter { isImageFile($0) }
        attachImages(imageURLs)
        return true
    }
}
```

---

## プロジェクト評価

| 観点 | 評価 | 理由 |
|------|------|------|
| **コード品質** | ★★★★☆ | 良質なAppKitコード |
| **ドキュメント** | ★★★☆☆ | 基本的なREADME |
| **アクティビティ** | ★★★☆☆ | メンテナンス中 |
| **学習価値** | ★★★★★ | AppKit学習に最適 |
| **実用性** | ★★★★☆ | 実用的なクライアント |

---

## macOS統合

### 通知センター

```swift
// macOS通知
class NotificationService {
    func showNotification(status: Status) {
        let content = UNMutableNotificationContent()
        content.title = status.account.displayName
        content.body = status.content.strippingHTML()
        content.userInfo = ["statusId": status.id]

        let request = UNNotificationRequest(
            identifier: status.id,
            content: content,
            trigger: nil
        )

        UNUserNotificationCenter.current().add(request)
    }
}
```

### Touch Bar

```swift
// Touch Bar対応
extension TimelineViewController: NSTouchBarDelegate {
    override func makeTouchBar() -> NSTouchBar? {
        let touchBar = NSTouchBar()
        touchBar.delegate = self
        touchBar.defaultItemIdentifiers = [
            .compose, .reply, .favorite, .boost
        ]
        return touchBar
    }

    func touchBar(
        _ touchBar: NSTouchBar,
        makeItemForIdentifier identifier: NSTouchBarItem.Identifier
    ) -> NSTouchBarItem? {
        switch identifier {
        case .compose:
            let item = NSButtonTouchBarItem(
                identifier: identifier,
                image: NSImage(systemSymbolName: "square.and.pencil", accessibilityDescription: nil)!,
                target: self,
                action: #selector(compose)
            )
            return item
        default:
            return nil
        }
    }
}
```

---

## 開発者向け情報

### ビルド手順

```bash
git clone https://github.com/chucker/Mastonaut
cd Mastonaut
open Mastonaut.xcodeproj
```

### 必要要件

- Xcode 14+
- macOS 12+

---

## まとめ

Mastonautは、AppKitを使用したネイティブmacOSアプリの優れた実装例。マルチカラムレイアウト、キーボードナビゲーション、OAuth認証など、デスクトップアプリ開発のパターンを学べる。

**推奨対象**:
- AppKit でmacOSアプリを開発したい人
- マルチカラムUIを実装したい開発者
- OAuth認証フローを学びたい人
- キーボードナビゲーションを実装したい人
