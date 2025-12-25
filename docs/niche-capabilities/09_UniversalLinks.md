# Universal Links / Deep Link

**驚きポイント**: URLからアプリ内の特定画面に直接遷移

## 参考プロジェクト

| プロジェクト | Stars | 特徴 |
|-------------|-------|------|
| [Knil](https://github.com/ethanhuang13/knil) | 767 | Universal Linksテストツール |
| [iOSAppsInfo](https://github.com/wujianguo/iOSAppsInfo) | 344 | URL Scheme一覧表示 |

---

## 1. Universal Links設定

### apple-app-site-association

```json
{
  "applinks": {
    "apps": [],
    "details": [
      {
        "appID": "TEAMID.com.example.app",
        "paths": [
          "/products/*",
          "/users/*",
          "/share/*"
        ]
      }
    ]
  }
}
```

**配置場所**: `https://example.com/.well-known/apple-app-site-association`

### Associated Domains (Entitlements)

```xml
<key>com.apple.developer.associated-domains</key>
<array>
    <string>applinks:example.com</string>
    <string>applinks:www.example.com</string>
</array>
```

---

## 2. アプリ側の処理

### SceneDelegate

```swift
class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    func scene(
        _ scene: UIScene,
        continue userActivity: NSUserActivity
    ) {
        guard userActivity.activityType == NSUserActivityTypeBrowsingWeb,
              let url = userActivity.webpageURL else { return }

        handleDeepLink(url)
    }

    private func handleDeepLink(_ url: URL) {
        let pathComponents = url.pathComponents

        switch pathComponents.first {
        case "products":
            if let productId = pathComponents[safe: 1] {
                navigateToProduct(productId)
            }
        case "users":
            if let userId = pathComponents[safe: 1] {
                navigateToUser(userId)
            }
        default:
            break
        }
    }
}
```

### SwiftUI App

```swift
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
                .onOpenURL { url in
                    handleURL(url)
                }
        }
    }

    private func handleURL(_ url: URL) {
        // URL処理
    }
}
```

---

## 3. URL Scheme（カスタムスキーム）

### Info.plist

```xml
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>myapp</string>
        </array>
        <key>CFBundleURLName</key>
        <string>com.example.myapp</string>
    </dict>
</array>
```

### URL構造設計

```
myapp://products/123
myapp://users/profile/456
myapp://settings?tab=notifications
```

### パース

```swift
struct DeepLinkParser {
    enum Route {
        case product(id: String)
        case user(id: String)
        case settings(tab: String?)
        case unknown
    }

    static func parse(_ url: URL) -> Route {
        guard url.scheme == "myapp" else { return .unknown }

        let host = url.host
        let pathComponents = url.pathComponents.filter { $0 != "/" }
        let queryItems = URLComponents(url: url, resolvingAgainstBaseURL: false)?.queryItems

        switch host {
        case "products":
            guard let id = pathComponents.first else { return .unknown }
            return .product(id: id)

        case "users":
            guard let id = pathComponents.first else { return .unknown }
            return .user(id: id)

        case "settings":
            let tab = queryItems?.first(where: { $0.name == "tab" })?.value
            return .settings(tab: tab)

        default:
            return .unknown
        }
    }
}
```

---

## 4. 他アプリを開く

```swift
class AppLauncher {
    // 他アプリを開く
    func openApp(scheme: String) {
        guard let url = URL(string: "\(scheme)://"),
              UIApplication.shared.canOpenURL(url) else {
            return
        }

        UIApplication.shared.open(url)
    }

    // Safariで開く
    func openInSafari(_ urlString: String) {
        guard let url = URL(string: urlString) else { return }
        UIApplication.shared.open(url)
    }

    // メールアプリ
    func openMail(to: String, subject: String, body: String) {
        let encodedSubject = subject.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? ""
        let encodedBody = body.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? ""

        if let url = URL(string: "mailto:\(to)?subject=\(encodedSubject)&body=\(encodedBody)") {
            UIApplication.shared.open(url)
        }
    }

    // 電話
    func call(_ number: String) {
        if let url = URL(string: "tel:\(number)") {
            UIApplication.shared.open(url)
        }
    }

    // マップ
    func openMaps(query: String) {
        let encoded = query.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? ""
        if let url = URL(string: "maps://?q=\(encoded)") {
            UIApplication.shared.open(url)
        }
    }
}
```

### Info.plist（他アプリ確認用）

```xml
<key>LSApplicationQueriesSchemes</key>
<array>
    <string>twitter</string>
    <string>instagram</string>
    <string>fb</string>
</array>
```

---

## 5. Knil式テスト

```swift
class UniversalLinkTester {
    // Associated Domainsの検証
    func validateAASA(domain: String) async throws -> Bool {
        let urls = [
            "https://\(domain)/.well-known/apple-app-site-association",
            "https://\(domain)/apple-app-site-association"
        ]

        for urlString in urls {
            guard let url = URL(string: urlString) else { continue }

            do {
                let (data, response) = try await URLSession.shared.data(from: url)

                guard let httpResponse = response as? HTTPURLResponse,
                      httpResponse.statusCode == 200 else { continue }

                // Content-Typeチェック
                let contentType = httpResponse.value(forHTTPHeaderField: "Content-Type") ?? ""
                guard contentType.contains("application/json") else { continue }

                // JSONパース
                let json = try JSONSerialization.jsonObject(with: data)
                if let dict = json as? [String: Any],
                   dict["applinks"] != nil {
                    return true
                }
            } catch {
                continue
            }
        }

        return false
    }

    // インストール済みアプリのURL Scheme取得
    func getInstalledAppSchemes() -> [String] {
        // プライベートAPIが必要（App Store不可）
        return []
    }
}
```

---

## 6. Handoff

```swift
class HandoffManager {
    func startActivity(type: String, userInfo: [String: Any]) {
        let activity = NSUserActivity(activityType: type)
        activity.title = "Continue Reading"
        activity.userInfo = userInfo
        activity.isEligibleForHandoff = true
        activity.webpageURL = URL(string: "https://example.com/article/123")

        activity.becomeCurrent()
    }
}

// Info.plist
// <key>NSUserActivityTypes</key>
// <array>
//     <string>com.example.app.reading</string>
// </array>
```

---

## まとめ

| 方式 | セキュリティ | 用途 |
|------|------------|------|
| Universal Links | 高（ドメイン認証） | Web ↔ App連携 |
| URL Scheme | 低（重複可能） | App ↔ App連携 |
| Handoff | 高（同一iCloud） | デバイス間継続 |

### 設計のヒント

1. **フォールバック**: アプリ未インストール時はWebへ
2. **パラメータ検証**: 不正なURLを拒否
3. **状態復元**: ディープリンクで中断した作業を復元
4. **アナリティクス**: どのリンクから来たか追跡
