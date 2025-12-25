# Share Extension 実装ガイド

**参考プロジェクト**:
- 各種ノートアプリ
- SNSアプリ

**実装参考度**: ⭐⭐⭐⭐☆

---

## なぜ重要か

Share Extensionは他アプリから**コンテンツを受け取る**ための仕組み。ユーザー獲得経路として重要。

| 学習ポイント | 重要度 | 理由 |
|-------------|--------|------|
| **NSExtensionItem** | ⭐⭐⭐⭐⭐ | 共有データの受け取り |
| **App Group** | ⭐⭐⭐⭐⭐ | データ共有 |
| **UI実装** | ⭐⭐⭐⭐☆ | SwiftUI対応 |
| **Activation Rules** | ⭐⭐⭐☆☆ | フィルタリング |

---

## 1. Extension作成

### Xcodeで追加

```
File → New → Target → Share Extension
```

### Info.plist設定

```xml
<key>NSExtension</key>
<dict>
    <key>NSExtensionAttributes</key>
    <dict>
        <key>NSExtensionActivationRule</key>
        <dict>
            <!-- URLを1つ受け取る -->
            <key>NSExtensionActivationSupportsWebURLWithMaxCount</key>
            <integer>1</integer>

            <!-- テキストを受け取る -->
            <key>NSExtensionActivationSupportsText</key>
            <true/>

            <!-- 画像を最大10枚 -->
            <key>NSExtensionActivationSupportsImageWithMaxCount</key>
            <integer>10</integer>
        </dict>
    </dict>
    <key>NSExtensionMainStoryboard</key>
    <string>MainInterface</string>
    <key>NSExtensionPointIdentifier</key>
    <string>com.apple.share-services</string>
</dict>
```

---

## 2. 基本的な実装

### UIKit版

```swift
import UIKit
import UniformTypeIdentifiers

class ShareViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        setupUI()
        loadSharedContent()
    }

    private func setupUI() {
        view.backgroundColor = .systemBackground

        let saveButton = UIBarButtonItem(
            title: "保存",
            style: .done,
            target: self,
            action: #selector(saveAction)
        )
        navigationItem.rightBarButtonItem = saveButton

        let cancelButton = UIBarButtonItem(
            title: "キャンセル",
            style: .plain,
            target: self,
            action: #selector(cancelAction)
        )
        navigationItem.leftBarButtonItem = cancelButton
    }

    private func loadSharedContent() {
        guard let extensionItems = extensionContext?.inputItems as? [NSExtensionItem] else {
            return
        }

        for item in extensionItems {
            guard let attachments = item.attachments else { continue }

            for attachment in attachments {
                // URL
                if attachment.hasItemConformingToTypeIdentifier(UTType.url.identifier) {
                    loadURL(from: attachment)
                }

                // テキスト
                if attachment.hasItemConformingToTypeIdentifier(UTType.plainText.identifier) {
                    loadText(from: attachment)
                }

                // 画像
                if attachment.hasItemConformingToTypeIdentifier(UTType.image.identifier) {
                    loadImage(from: attachment)
                }
            }
        }
    }

    private func loadURL(from attachment: NSItemProvider) {
        attachment.loadItem(forTypeIdentifier: UTType.url.identifier) { [weak self] item, error in
            guard let url = item as? URL else { return }
            DispatchQueue.main.async {
                self?.handleURL(url)
            }
        }
    }

    private func loadText(from attachment: NSItemProvider) {
        attachment.loadItem(forTypeIdentifier: UTType.plainText.identifier) { [weak self] item, error in
            guard let text = item as? String else { return }
            DispatchQueue.main.async {
                self?.handleText(text)
            }
        }
    }

    private func loadImage(from attachment: NSItemProvider) {
        attachment.loadItem(forTypeIdentifier: UTType.image.identifier) { [weak self] item, error in
            var image: UIImage?

            if let url = item as? URL {
                image = UIImage(contentsOfFile: url.path)
            } else if let data = item as? Data {
                image = UIImage(data: data)
            } else if let img = item as? UIImage {
                image = img
            }

            guard let loadedImage = image else { return }

            DispatchQueue.main.async {
                self?.handleImage(loadedImage)
            }
        }
    }

    private func handleURL(_ url: URL) {
        // URLの処理
        print("Received URL: \(url)")
    }

    private func handleText(_ text: String) {
        // テキストの処理
        print("Received text: \(text)")
    }

    private func handleImage(_ image: UIImage) {
        // 画像の処理
        print("Received image: \(image.size)")
    }

    @objc private func saveAction() {
        // 保存処理
        saveToAppGroup()

        extensionContext?.completeRequest(returningItems: nil)
    }

    @objc private func cancelAction() {
        extensionContext?.cancelRequest(withError: NSError(
            domain: "ShareExtension",
            code: 0,
            userInfo: nil
        ))
    }

    private func saveToAppGroup() {
        let defaults = UserDefaults(suiteName: "group.com.example.app")
        // データを保存
    }
}
```

---

## 3. SwiftUI版

### Hosting Controller

```swift
import SwiftUI

class ShareViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()

        let contentView = ShareView(extensionContext: extensionContext)
        let hostingController = UIHostingController(rootView: contentView)

        addChild(hostingController)
        view.addSubview(hostingController.view)
        hostingController.view.translatesAutoresizingMaskIntoConstraints = false

        NSLayoutConstraint.activate([
            hostingController.view.topAnchor.constraint(equalTo: view.topAnchor),
            hostingController.view.bottomAnchor.constraint(equalTo: view.bottomAnchor),
            hostingController.view.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            hostingController.view.trailingAnchor.constraint(equalTo: view.trailingAnchor)
        ])

        hostingController.didMove(toParent: self)
    }
}

struct ShareView: View {
    let extensionContext: NSExtensionContext?

    @State private var sharedURL: URL?
    @State private var sharedText: String = ""
    @State private var note: String = ""

    var body: some View {
        NavigationStack {
            Form {
                if let url = sharedURL {
                    Section("共有URL") {
                        Text(url.absoluteString)
                    }
                }

                if !sharedText.isEmpty {
                    Section("共有テキスト") {
                        Text(sharedText)
                    }
                }

                Section("メモ") {
                    TextField("メモを追加...", text: $note)
                }
            }
            .navigationTitle("保存")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("キャンセル") {
                        cancel()
                    }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("保存") {
                        save()
                    }
                }
            }
        }
        .task {
            await loadSharedContent()
        }
    }

    private func loadSharedContent() async {
        guard let items = extensionContext?.inputItems as? [NSExtensionItem] else {
            return
        }

        for item in items {
            guard let attachments = item.attachments else { continue }

            for attachment in attachments {
                if attachment.hasItemConformingToTypeIdentifier(UTType.url.identifier) {
                    if let url = try? await attachment.loadItem(
                        forTypeIdentifier: UTType.url.identifier
                    ) as? URL {
                        sharedURL = url
                    }
                }

                if attachment.hasItemConformingToTypeIdentifier(UTType.plainText.identifier) {
                    if let text = try? await attachment.loadItem(
                        forTypeIdentifier: UTType.plainText.identifier
                    ) as? String {
                        sharedText = text
                    }
                }
            }
        }
    }

    private func save() {
        // App Groupに保存
        let defaults = UserDefaults(suiteName: "group.com.example.app")
        defaults?.set(sharedURL?.absoluteString, forKey: "lastSharedURL")
        defaults?.set(note, forKey: "lastNote")

        extensionContext?.completeRequest(returningItems: nil)
    }

    private func cancel() {
        extensionContext?.cancelRequest(withError: NSError(
            domain: "ShareExtension",
            code: 0
        ))
    }
}
```

---

## 4. App Groupでデータ共有

### 設定

```
1. メインアプリのCapabilities → App Groups → ON
2. Share ExtensionのCapabilities → App Groups → ON
3. 同じGroup ID（例: group.com.example.app）を選択
```

### 共有データモデル

```swift
// 共有フレームワークまたは両方のターゲットに含める
struct SharedItem: Codable {
    let url: URL?
    let text: String?
    let note: String
    let createdAt: Date
}

class SharedDataManager {
    private let suiteName = "group.com.example.app"
    private let key = "sharedItems"

    func save(_ item: SharedItem) {
        let defaults = UserDefaults(suiteName: suiteName)
        var items = loadAll()
        items.append(item)

        if let data = try? JSONEncoder().encode(items) {
            defaults?.set(data, forKey: key)
        }
    }

    func loadAll() -> [SharedItem] {
        let defaults = UserDefaults(suiteName: suiteName)
        guard let data = defaults?.data(forKey: key),
              let items = try? JSONDecoder().decode([SharedItem].self, from: data) else {
            return []
        }
        return items
    }
}
```

---

## 5. Activation Rules詳細

### 複合ルール

```xml
<key>NSExtensionActivationRule</key>
<string>
SUBQUERY (
    extensionItems,
    $extensionItem,
    SUBQUERY (
        $extensionItem.attachments,
        $attachment,
        ANY $attachment.registeredTypeIdentifiers UTI-CONFORMS-TO "public.url"
    ).@count == $extensionItem.attachments.@count
).@count == 1
</string>
```

### よく使うルール

```xml
<!-- URLのみ（1つ） -->
<key>NSExtensionActivationSupportsWebURLWithMaxCount</key>
<integer>1</integer>

<!-- テキストのみ -->
<key>NSExtensionActivationSupportsText</key>
<true/>

<!-- 画像（複数可） -->
<key>NSExtensionActivationSupportsImageWithMaxCount</key>
<integer>10</integer>

<!-- ファイル -->
<key>NSExtensionActivationSupportsFileWithMaxCount</key>
<integer>5</integer>
```

---

## まとめ：Share Extension

| 機能 | 用途 | 優先度 |
|------|------|--------|
| **NSItemProvider** | データ受け取り | 🔴 必須 |
| **App Group** | メインアプリとの共有 | 🔴 必須 |
| **Activation Rules** | 対応形式の指定 | 🟡 推奨 |
| **SwiftUI対応** | モダンUI | 🟢 参考 |

### 注意点

1. **メモリ制限**: Extension は 120MB まで
2. **実行時間**: 数秒以内に完了する必要あり
3. **バックグラウンド**: 長時間処理は避ける
4. **App Group**: データ共有の設定を忘れずに
