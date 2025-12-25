# ブラウザ&Webアプリ分析

**調査日**: 2025年12月25日
**対象**: 31件のブラウザ・Webアプリ

---

## 概要

iOSブラウザアプリは、WebKit/WKWebViewの活用からプライバシー保護機能まで多様。Firefox、Braveなどの主要ブラウザからプライバシー特化型まで。

---

## トップアプリ

| スター | アプリ | カテゴリ | 説明 |
|--------|--------|----------|------|
| 12,796 | Firefox | 汎用ブラウザ | 公式アプリ |
| 3,700 | EhPanda | 専門ブラウザ | SwiftUI製 |
| 2,503 | Onion Browser | プライバシー | Tor対応 |
| 1,725 | Brave | プライバシー | 広告ブロック |
| 1,594 | Trust | Web3 | DAppブラウザ |
| 1,254 | Firefox Focus | プライバシー | 追跡防止 |
| 1,244 | Stay | Safari拡張 | アプリジャンプ防止 |
| 749 | FileExplorer | ファイル | 多機能 |
| 660 | Siesta | GitHub | リポジトリ閲覧 |
| 317 | File Browser | ファイル | シンプル |

---

## カテゴリ別詳細

### 汎用ブラウザ

| アプリ | スター | 特徴 |
|--------|--------|------|
| Firefox | 12,796 | Gecko/WebView、同期 |
| Brave | 1,725 | 広告ブロック内蔵 |
| DuckDuckGo | 143 | プライバシー重視 |

### プライバシー特化

| アプリ | スター | 機能 |
|--------|--------|------|
| Onion Browser | 2,503 | Tor統合 |
| Firefox Focus | 1,254 | 追跡防止、自動削除 |
| SnowHaze | 164 | 強力なプライバシー |
| Endless Browser | 270 | セキュリティ重視 |
| Ghostery Dawn | 75 | 追跡ブロック |

### Web3・DApp

| アプリ | スター | 対応 |
|--------|--------|------|
| Trust | 1,594 | Ethereum DApp |

### 専門ブラウザ

| アプリ | スター | 用途 |
|--------|--------|------|
| EhPanda | 3,700 | SwiftUI、API統合 |
| Siesta | 660 | GitHub閲覧 |
| SF Symbols Browser | 137 | アイコン閲覧 |

---

## 技術パターン

### WKWebView基本設定

```swift
import WebKit

class BrowserViewController: UIViewController {
    var webView: WKWebView!

    override func viewDidLoad() {
        super.viewDidLoad()

        let configuration = WKWebViewConfiguration()

        // JavaScript設定
        configuration.defaultWebpagePreferences.allowsContentJavaScript = true

        // ユーザーエージェント
        configuration.applicationNameForUserAgent = "MyBrowser/1.0"

        // データストア（プライベートモード）
        configuration.websiteDataStore = .nonPersistent()

        webView = WKWebView(frame: view.bounds, configuration: configuration)
        webView.navigationDelegate = self
        webView.uiDelegate = self
        view.addSubview(webView)
    }
}

extension BrowserViewController: WKNavigationDelegate {
    func webView(_ webView: WKWebView, decidePolicyFor navigationAction: WKNavigationAction, decisionHandler: @escaping (WKNavigationActionPolicy) -> Void) {
        // ナビゲーション制御
        if let url = navigationAction.request.url {
            if shouldBlock(url) {
                decisionHandler(.cancel)
                return
            }
        }
        decisionHandler(.allow)
    }

    func webView(_ webView: WKWebView, didFinish navigation: WKNavigation!) {
        // ページ読み込み完了
        title = webView.title
    }
}
```

### コンテンツブロッカー

```swift
// コンテンツブロッカー実装
class ContentBlocker {
    static func loadRules() -> String {
        """
        [
            {
                "trigger": {
                    "url-filter": ".*",
                    "resource-type": ["script"],
                    "load-type": ["third-party"]
                },
                "action": {
                    "type": "block"
                }
            },
            {
                "trigger": {
                    "url-filter": ".*tracker.*"
                },
                "action": {
                    "type": "block"
                }
            }
        ]
        """
    }

    static func compile(completion: @escaping (WKContentRuleList?) -> Void) {
        WKContentRuleListStore.default().compileContentRuleList(
            forIdentifier: "AdBlocker",
            encodedContentRuleList: loadRules()
        ) { ruleList, error in
            completion(ruleList)
        }
    }
}

// 使用
ContentBlocker.compile { ruleList in
    if let ruleList = ruleList {
        configuration.userContentController.add(ruleList)
    }
}
```

### タブ管理

```swift
// タブモデル
class Tab: Identifiable, ObservableObject {
    let id = UUID()
    @Published var url: URL?
    @Published var title: String = "New Tab"
    @Published var isLoading = false
    @Published var canGoBack = false
    @Published var canGoForward = false

    weak var webView: WKWebView?
}

// タブマネージャー
class TabManager: ObservableObject {
    @Published var tabs: [Tab] = []
    @Published var activeTabIndex: Int = 0

    var activeTab: Tab? {
        guard tabs.indices.contains(activeTabIndex) else { return nil }
        return tabs[activeTabIndex]
    }

    func newTab(url: URL? = nil) {
        let tab = Tab()
        tab.url = url
        tabs.append(tab)
        activeTabIndex = tabs.count - 1
    }

    func closeTab(at index: Int) {
        tabs.remove(at: index)
        if activeTabIndex >= tabs.count {
            activeTabIndex = max(0, tabs.count - 1)
        }
    }
}
```

### SwiftUIブラウザ

```swift
import SwiftUI
import WebKit

struct WebView: UIViewRepresentable {
    let url: URL
    @Binding var isLoading: Bool
    @Binding var title: String

    func makeUIView(context: Context) -> WKWebView {
        let webView = WKWebView()
        webView.navigationDelegate = context.coordinator
        return webView
    }

    func updateUIView(_ webView: WKWebView, context: Context) {
        if webView.url != url {
            webView.load(URLRequest(url: url))
        }
    }

    func makeCoordinator() -> Coordinator {
        Coordinator(self)
    }

    class Coordinator: NSObject, WKNavigationDelegate {
        var parent: WebView

        init(_ parent: WebView) {
            self.parent = parent
        }

        func webView(_ webView: WKWebView, didStartProvisionalNavigation navigation: WKNavigation!) {
            parent.isLoading = true
        }

        func webView(_ webView: WKWebView, didFinish navigation: WKNavigation!) {
            parent.isLoading = false
            parent.title = webView.title ?? ""
        }
    }
}
```

---

## プライバシー機能

### 追跡防止

```swift
// Intelligent Tracking Prevention
class PrivacyManager {
    func configurePrivacy(for configuration: WKWebViewConfiguration) {
        // Cookie設定
        configuration.websiteDataStore.httpCookieStore.setCookie(
            HTTPCookie(properties: [
                .name: "tracking",
                .value: "disabled",
                .domain: ".example.com",
                .path: "/"
            ])!
        )

        // 追跡防止
        let preferences = WKWebpagePreferences()
        preferences.allowsContentJavaScript = false
        configuration.defaultWebpagePreferences = preferences
    }

    func clearBrowsingData() async {
        let dataStore = WKWebsiteDataStore.default()
        let types = WKWebsiteDataStore.allWebsiteDataTypes()
        let dateFrom = Date(timeIntervalSince1970: 0)

        await dataStore.removeData(ofTypes: types, modifiedSince: dateFrom)
    }
}
```

### Tor統合

```swift
// Onion Browserパターン
class TorManager {
    var torController: TorController?

    func startTor(completion: @escaping (Bool) -> Void) {
        // Tor設定
        let config = TorConfiguration()
        config.socksPort = 9050
        config.controlPort = 9051

        // 起動
        torController = TorController(configuration: config)
        torController?.connect { success in
            completion(success)
        }
    }

    func createTorSession() -> URLSession {
        let config = URLSessionConfiguration.default
        config.connectionProxyDictionary = [
            kCFStreamPropertySOCKSProxyHost: "127.0.0.1",
            kCFStreamPropertySOCKSProxyPort: 9050
        ]
        return URLSession(configuration: config)
    }
}
```

---

## Safari拡張

### 共有拡張

```swift
// Share Extension for Safari
class ShareViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()

        guard let item = extensionContext?.inputItems.first as? NSExtensionItem,
              let attachment = item.attachments?.first else {
            return
        }

        if attachment.hasItemConformingToTypeIdentifier(kUTTypeURL as String) {
            attachment.loadItem(forTypeIdentifier: kUTTypeURL as String) { item, error in
                if let url = item as? URL {
                    self.handleURL(url)
                }
            }
        }
    }
}
```

---

## Swiftパッケージ機会

### WebViewKit

**機能**:
- WKWebView簡略化
- SwiftUI統合
- ナビゲーション管理
- JavaScript橋渡し

### PrivacyBrowserKit

**機能**:
- コンテンツブロック
- 追跡防止
- Cookie管理
- 履歴自動削除

### TabManagerKit

**機能**:
- タブ管理
- セッション復元
- メモリ最適化
- スワイプジェスチャー

---

## 学習リソース

| アプリ | 学習ポイント |
|--------|-------------|
| Firefox | 大規模ブラウザ設計 |
| Onion Browser | Tor統合、プライバシー |
| EhPanda | SwiftUI + WebView |
| Brave | コンテンツブロック |

---

## 開発上の考慮事項

### パフォーマンス

| 項目 | 対策 |
|------|------|
| メモリ | 非表示タブのサスペンド |
| レンダリング | プロセス分離 |
| キャッシュ | 適切なキャッシュ戦略 |
| JavaScript | 必要時のみ有効 |

### セキュリティ

| 項目 | 対応 |
|------|------|
| HTTPS | 強制リダイレクト |
| 証明書 | ピンニング検討 |
| JavaScript | サンドボックス |
| Cookie | 適切な有効期限 |

---

## トレンド

### 成長中
- プライバシー強化
- Web3/DApp対応
- SwiftUI統合

### 安定
- 基本ブラウザ機能
- タブ管理

### 注目
- Safari拡張エコシステム
- コンテンツブロッカー
- 分散型Web

