# Notification Extension 実装ガイド

**参考プロジェクト**:
- 各種メッセージングアプリ
- ニュースアプリ

**実装参考度**: ⭐⭐⭐⭐☆

---

## なぜ重要か

Notification Extensionは**リッチな通知UI**を提供。画像、動画、カスタムUIでユーザーエンゲージメントを向上。

| 学習ポイント | 重要度 | 理由 |
|-------------|--------|------|
| **Service Extension** | ⭐⭐⭐⭐⭐ | ペイロード加工 |
| **Content Extension** | ⭐⭐⭐⭐⭐ | カスタムUI |
| **メディア添付** | ⭐⭐⭐⭐☆ | 画像・動画 |
| **インタラクション** | ⭐⭐⭐☆☆ | ボタン・入力 |

---

## 1. Notification Service Extension

### 用途

- ペイロードの加工
- 画像/動画のダウンロード
- 暗号化コンテンツの復号
- コンテンツの変更

### 作成

```
File → New → Target → Notification Service Extension
```

### 実装

```swift
import UserNotifications

class NotificationService: UNNotificationServiceExtension {
    var contentHandler: ((UNNotificationContent) -> Void)?
    var bestAttemptContent: UNMutableNotificationContent?

    override func didReceive(
        _ request: UNNotificationRequest,
        withContentHandler contentHandler: @escaping (UNNotificationContent) -> Void
    ) {
        self.contentHandler = contentHandler
        bestAttemptContent = (request.content.mutableCopy() as? UNMutableNotificationContent)

        guard let bestAttemptContent = bestAttemptContent else {
            contentHandler(request.content)
            return
        }

        // タイトルを加工
        bestAttemptContent.title = "[New] " + bestAttemptContent.title

        // 画像を添付
        if let imageURLString = bestAttemptContent.userInfo["imageURL"] as? String,
           let imageURL = URL(string: imageURLString) {
            downloadImage(from: imageURL) { attachment in
                if let attachment = attachment {
                    bestAttemptContent.attachments = [attachment]
                }
                contentHandler(bestAttemptContent)
            }
        } else {
            contentHandler(bestAttemptContent)
        }
    }

    override func serviceExtensionTimeWillExpire() {
        // タイムアウト時（約30秒）
        if let contentHandler = contentHandler,
           let bestAttemptContent = bestAttemptContent {
            contentHandler(bestAttemptContent)
        }
    }

    private func downloadImage(
        from url: URL,
        completion: @escaping (UNNotificationAttachment?) -> Void
    ) {
        let task = URLSession.shared.downloadTask(with: url) { location, _, error in
            guard let location = location, error == nil else {
                completion(nil)
                return
            }

            // 一時ファイルにコピー
            let tempDirectory = FileManager.default.temporaryDirectory
            let tempFile = tempDirectory.appendingPathComponent(UUID().uuidString + ".jpg")

            do {
                try FileManager.default.moveItem(at: location, to: tempFile)
                let attachment = try UNNotificationAttachment(
                    identifier: UUID().uuidString,
                    url: tempFile,
                    options: nil
                )
                completion(attachment)
            } catch {
                completion(nil)
            }
        }
        task.resume()
    }
}
```

### Push Payload

```json
{
    "aps": {
        "alert": {
            "title": "新しいメッセージ",
            "body": "Hello!"
        },
        "mutable-content": 1
    },
    "imageURL": "https://example.com/image.jpg"
}
```

---

## 2. Notification Content Extension

### 用途

- カスタム通知UI
- インタラクティブ要素
- 動画再生
- 地図表示

### 作成

```
File → New → Target → Notification Content Extension
```

### Info.plist設定

```xml
<key>NSExtension</key>
<dict>
    <key>NSExtensionAttributes</key>
    <dict>
        <!-- カテゴリID -->
        <key>UNNotificationExtensionCategory</key>
        <string>MESSAGE_CATEGORY</string>

        <!-- デフォルトUIを非表示 -->
        <key>UNNotificationExtensionDefaultContentHidden</key>
        <true/>

        <!-- 初期サイズ比率 -->
        <key>UNNotificationExtensionInitialContentSizeRatio</key>
        <real>0.5</real>

        <!-- ユーザー操作を許可 -->
        <key>UNNotificationExtensionUserInteractionEnabled</key>
        <true/>
    </dict>
    <key>NSExtensionMainStoryboard</key>
    <string>MainInterface</string>
    <key>NSExtensionPointIdentifier</key>
    <string>com.apple.usernotifications.content-extension</string>
</dict>
```

### UIKit実装

```swift
import UIKit
import UserNotifications
import UserNotificationsUI

class NotificationViewController: UIViewController, UNNotificationContentExtension {
    @IBOutlet weak var imageView: UIImageView!
    @IBOutlet weak var titleLabel: UILabel!
    @IBOutlet weak var bodyLabel: UILabel!
    @IBOutlet weak var likeButton: UIButton!

    private var notification: UNNotification?

    override func viewDidLoad() {
        super.viewDidLoad()
        setupUI()
    }

    private func setupUI() {
        likeButton.addTarget(self, action: #selector(likeButtonTapped), for: .touchUpInside)
    }

    func didReceive(_ notification: UNNotification) {
        self.notification = notification
        let content = notification.request.content

        titleLabel.text = content.title
        bodyLabel.text = content.body

        // 添付画像を表示
        if let attachment = content.attachments.first,
           attachment.url.startAccessingSecurityScopedResource() {
            defer { attachment.url.stopAccessingSecurityScopedResource() }

            if let data = try? Data(contentsOf: attachment.url),
               let image = UIImage(data: data) {
                imageView.image = image
            }
        }
    }

    func didReceive(
        _ response: UNNotificationResponse,
        completionHandler completion: @escaping (UNNotificationContentExtensionResponseOption) -> Void
    ) {
        switch response.actionIdentifier {
        case "LIKE_ACTION":
            // いいね処理
            likeButton.setTitle("❤️ Liked!", for: .normal)
            completion(.doNotDismiss)

        case "REPLY_ACTION":
            if let textResponse = response as? UNTextInputNotificationResponse {
                // 返信処理
                sendReply(textResponse.userText)
            }
            completion(.dismiss)

        default:
            completion(.dismissAndForwardAction)
        }
    }

    @objc private func likeButtonTapped() {
        likeButton.setTitle("❤️ Liked!", for: .normal)
        // API呼び出しなど
    }

    private func sendReply(_ text: String) {
        // 返信送信処理
    }
}
```

---

## 3. SwiftUI版 Content Extension

```swift
import SwiftUI
import UserNotifications
import UserNotificationsUI

class NotificationViewController: UIViewController, UNNotificationContentExtension {
    override func viewDidLoad() {
        super.viewDidLoad()
    }

    func didReceive(_ notification: UNNotification) {
        let content = notification.request.content
        let hostingController = UIHostingController(
            rootView: NotificationContentView(
                title: content.title,
                body: content.body,
                attachment: content.attachments.first
            )
        )

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

struct NotificationContentView: View {
    let title: String
    let body: String
    let attachment: UNNotificationAttachment?

    var body: some View {
        VStack(alignment: .leading, spacing: 12) {
            if let attachment = attachment,
               let image = loadImage(from: attachment) {
                Image(uiImage: image)
                    .resizable()
                    .aspectRatio(contentMode: .fill)
                    .frame(height: 150)
                    .clipped()
            }

            VStack(alignment: .leading, spacing: 4) {
                Text(title)
                    .font(.headline)

                Text(body)
                    .font(.body)
                    .foregroundColor(.secondary)
            }
            .padding(.horizontal)
        }
        .padding(.vertical)
    }

    private func loadImage(from attachment: UNNotificationAttachment) -> UIImage? {
        guard attachment.url.startAccessingSecurityScopedResource() else {
            return nil
        }
        defer { attachment.url.stopAccessingSecurityScopedResource() }

        guard let data = try? Data(contentsOf: attachment.url) else {
            return nil
        }
        return UIImage(data: data)
    }
}
```

---

## 4. カテゴリとアクション設定

### メインアプリでの設定

```swift
import UserNotifications

class NotificationManager {
    static let shared = NotificationManager()

    func registerCategories() {
        // アクション定義
        let likeAction = UNNotificationAction(
            identifier: "LIKE_ACTION",
            title: "❤️ いいね",
            options: [.foreground]
        )

        let replyAction = UNTextInputNotificationAction(
            identifier: "REPLY_ACTION",
            title: "返信",
            options: [],
            textInputButtonTitle: "送信",
            textInputPlaceholder: "メッセージを入力..."
        )

        let deleteAction = UNNotificationAction(
            identifier: "DELETE_ACTION",
            title: "削除",
            options: [.destructive, .authenticationRequired]
        )

        // カテゴリ定義
        let messageCategory = UNNotificationCategory(
            identifier: "MESSAGE_CATEGORY",
            actions: [likeAction, replyAction, deleteAction],
            intentIdentifiers: [],
            hiddenPreviewsBodyPlaceholder: "新しいメッセージ",
            options: .customDismissAction
        )

        UNUserNotificationCenter.current().setNotificationCategories([messageCategory])
    }
}
```

---

## 5. 動画付き通知

```swift
class NotificationService: UNNotificationServiceExtension {
    private func downloadVideo(
        from url: URL,
        completion: @escaping (UNNotificationAttachment?) -> Void
    ) {
        let task = URLSession.shared.downloadTask(with: url) { location, _, error in
            guard let location = location, error == nil else {
                completion(nil)
                return
            }

            let tempFile = FileManager.default.temporaryDirectory
                .appendingPathComponent(UUID().uuidString + ".mp4")

            do {
                try FileManager.default.moveItem(at: location, to: tempFile)

                // サムネイルオプション
                let options: [String: Any] = [
                    UNNotificationAttachmentOptionsThumbnailTimeKey: 0,  // サムネイル取得位置
                    UNNotificationAttachmentOptionsThumbnailClippingRectKey:
                        CGRect(x: 0, y: 0, width: 1, height: 1).dictionaryRepresentation
                ]

                let attachment = try UNNotificationAttachment(
                    identifier: UUID().uuidString,
                    url: tempFile,
                    options: options
                )
                completion(attachment)
            } catch {
                completion(nil)
            }
        }
        task.resume()
    }
}
```

---

## まとめ：Notification Extension

| Extension | 用途 | 優先度 |
|-----------|------|--------|
| **Service Extension** | ペイロード加工、メディアDL | 🔴 必須 |
| **Content Extension** | カスタムUI | 🟡 推奨 |
| **カテゴリ/アクション** | インタラクション | 🟡 推奨 |

### 制限事項

1. **Service Extension**: 約30秒のタイムアウト
2. **Content Extension**: メモリ制限あり
3. **mutable-content**: 必ず1を設定
4. **カテゴリID**: Push payloadとInfo.plistで一致させる
