# CarPlay開発

**驚きポイント**: シミュレータでCarPlayアプリをテスト可能

## 参考プロジェクト

| プロジェクト | Stars | 特徴 |
|-------------|-------|------|
| [CarSample](https://github.com/below/CarSample) | 32 | Entitlement不要でテスト |

---

## 1. CarPlayの種類

| カテゴリ | Entitlement | 例 |
|---------|-------------|-----|
| Navigation | 必要（申請） | Google Maps |
| Audio | 必要（申請） | Spotify |
| Communication | 必要（申請） | WhatsApp |
| EV Charging | 必要（申請） | ChargePoint |
| Parking | 必要（申請） | SpotHero |
| Quick Food Ordering | 必要（申請） | Starbucks |

---

## 2. シミュレータでのテスト（CarSample方式）

Entitlementなしでシミュレータテスト可能。

### Info.plist

```xml
<key>UIApplicationSceneManifest</key>
<dict>
    <key>UISceneConfigurations</key>
    <dict>
        <key>CPTemplateApplicationSceneSessionRoleApplication</key>
        <array>
            <dict>
                <key>UISceneClassName</key>
                <string>CPTemplateApplicationScene</string>
                <key>UISceneConfigurationName</key>
                <string>CarPlay</string>
                <key>UISceneDelegateClassName</key>
                <string>$(PRODUCT_MODULE_NAME).CarPlaySceneDelegate</string>
            </dict>
        </array>
    </dict>
</dict>
```

### CarPlaySceneDelegate

```swift
import CarPlay

class CarPlaySceneDelegate: UIResponder, CPTemplateApplicationSceneDelegate {
    var interfaceController: CPInterfaceController?

    func templateApplicationScene(
        _ templateApplicationScene: CPTemplateApplicationScene,
        didConnect interfaceController: CPInterfaceController
    ) {
        self.interfaceController = interfaceController

        let rootTemplate = createRootTemplate()
        interfaceController.setRootTemplate(rootTemplate, animated: true)
    }

    func templateApplicationScene(
        _ templateApplicationScene: CPTemplateApplicationScene,
        didDisconnectInterfaceController interfaceController: CPInterfaceController
    ) {
        self.interfaceController = nil
    }

    private func createRootTemplate() -> CPTemplate {
        let item1 = CPListItem(text: "Item 1", detailText: "Detail")
        item1.handler = { [weak self] _, completion in
            self?.handleItemSelection()
            completion()
        }

        let section = CPListSection(items: [item1])
        let listTemplate = CPListTemplate(title: "My App", sections: [section])

        return listTemplate
    }

    private func handleItemSelection() {
        // アイテム選択時の処理
    }
}
```

---

## 3. テンプレート

### リストテンプレート

```swift
func createListTemplate() -> CPListTemplate {
    let items = (1...5).map { index in
        let item = CPListItem(
            text: "Item \(index)",
            detailText: "Description for item \(index)"
        )
        item.accessoryType = .disclosureIndicator
        return item
    }

    let section = CPListSection(items: items, header: "Section 1", sectionIndexTitle: "S1")
    return CPListTemplate(title: "List", sections: [section])
}
```

### グリッドテンプレート

```swift
func createGridTemplate() -> CPGridTemplate {
    let buttons = ["Home", "Work", "Favorites"].map { title in
        CPGridButton(
            titleVariants: [title],
            image: UIImage(systemName: "star")!
        ) { _ in
            // タップ処理
        }
    }

    return CPGridTemplate(title: "Quick Access", gridButtons: buttons)
}
```

### タブバーテンプレート

```swift
func createTabBarTemplate() -> CPTabBarTemplate {
    let tab1 = createListTemplate()
    tab1.tabTitle = "List"
    tab1.tabImage = UIImage(systemName: "list.bullet")

    let tab2 = createGridTemplate()
    tab2.tabTitle = "Grid"
    tab2.tabImage = UIImage(systemName: "square.grid.2x2")

    return CPTabBarTemplate(templates: [tab1, tab2])
}
```

---

## 4. Now Playing（オーディオアプリ）

```swift
import MediaPlayer

class NowPlayingManager {
    func updateNowPlaying(title: String, artist: String, artwork: UIImage?) {
        var info = [String: Any]()
        info[MPMediaItemPropertyTitle] = title
        info[MPMediaItemPropertyArtist] = artist

        if let artwork = artwork {
            info[MPMediaItemPropertyArtwork] = MPMediaItemArtwork(
                boundsSize: artwork.size
            ) { _ in artwork }
        }

        MPNowPlayingInfoCenter.default().nowPlayingInfo = info
    }

    func setupRemoteCommands() {
        let commandCenter = MPRemoteCommandCenter.shared()

        commandCenter.playCommand.addTarget { _ in
            // 再生
            return .success
        }

        commandCenter.pauseCommand.addTarget { _ in
            // 一時停止
            return .success
        }

        commandCenter.nextTrackCommand.addTarget { _ in
            // 次のトラック
            return .success
        }
    }
}
```

---

## 5. ナビゲーション（地図アプリ）

```swift
import CarPlay
import MapKit

class NavigationManager: NSObject {
    private var mapTemplate: CPMapTemplate?

    func setupMapTemplate() -> CPMapTemplate {
        let template = CPMapTemplate()
        template.mapDelegate = self

        // ナビボタン
        let startButton = CPBarButton(title: "Start") { _ in
            self.startNavigation()
        }
        template.leadingNavigationBarButtons = [startButton]

        self.mapTemplate = template
        return template
    }

    private func startNavigation() {
        let trip = CPTrip(
            origin: MKMapItem.forCurrentLocation(),
            destination: MKMapItem(placemark: MKPlacemark(
                coordinate: CLLocationCoordinate2D(latitude: 35.6812, longitude: 139.7671)
            )),
            routeChoices: []
        )

        mapTemplate?.startNavigationSession(for: trip)
    }
}

extension NavigationManager: CPMapTemplateDelegate {
    func mapTemplate(
        _ mapTemplate: CPMapTemplate,
        panWith direction: CPMapTemplate.PanDirection
    ) {
        // パン操作
    }
}
```

---

## 6. シミュレータ起動

```bash
# CarPlayシミュレータを起動
xcrun simctl io booted launch-carplay
```

Xcode: **I/O** → **External Displays** → **CarPlay**

---

## まとめ

| テンプレート | 用途 |
|-------------|------|
| CPListTemplate | リスト表示 |
| CPGridTemplate | グリッドボタン |
| CPTabBarTemplate | タブ切り替え |
| CPMapTemplate | 地図表示（ナビ） |
| CPNowPlayingTemplate | 再生中（オーディオ） |
| CPPointOfInterestTemplate | POI表示 |

### 制約

- **UIKitビュー不可**: テンプレートのみ
- **Entitlement必須**: App Store申請時
- **シンプルUI**: 運転中の安全性優先
