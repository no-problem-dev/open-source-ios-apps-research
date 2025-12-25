# Delta

**スター**: 4,300+ ⭐
**リポジトリ**: https://github.com/rileytestut/Delta
**ライセンス**: AGPL-3.0

---

## 概要

Deltaは、Riley Testut氏によるiOS向けオールインワンゲームエミュレータ。AltStoreと連携し、NES、SNES、N64、GBA等のクラシックゲームをプレイ可能。美しいUIとシームレスな体験を提供。

---

## 主要機能

| 機能 | 説明 |
|------|------|
| マルチシステム | NES, SNES, N64, GBA等 |
| コントローラー | MFi, Bluetooth対応 |
| セーブステート | どこでもセーブ |
| チート | チートコード対応 |
| AirPlay | 外部ディスプレイ出力 |
| iCloud同期 | セーブデータ同期 |

---

## 技術スタック

| 項目 | 技術 |
|------|------|
| 言語 | Swift |
| UI | UIKit |
| エミュレーション | DeltaCore |
| グラフィック | Metal |
| オーディオ | AVAudioEngine |
| 配布 | AltStore |

---

## アーキテクチャ

```
Delta/
├── Delta/                   # メインアプリ
│   ├── Game Selection/      # ゲーム選択UI
│   ├── Emulation/           # エミュレーション画面
│   ├── Pause Menu/          # ポーズメニュー
│   └── Settings/            # 設定
├── DeltaCore/               # コアフレームワーク
│   ├── Model/               # データモデル
│   ├── Emulator Core/       # エミュレータ基盤
│   └── UI/                  # UI コンポーネント
└── Systems/                 # 各システムコア
    ├── NES/
    ├── SNES/
    └── GBA/
```

---

## 学習価値

### モダンエミュレータ設計

| 分野 | 学習ポイント |
|------|-------------|
| **プロトコル指向** | 拡張可能な設計 |
| **Metal** | 高性能レンダリング |
| **タッチ入力** | バーチャルコントローラー |
| **サードパーティ配布** | AltStore連携 |
| **Core Data** | ゲームライブラリ管理 |

### コードパターン

**DeltaCore プロトコル**:
```swift
// 拡張可能なエミュレータコア設計
public protocol EmulatorCore: AnyObject {
    var state: EmulatorState { get }
    var videoManager: VideoManager { get }
    var audioManager: AudioManager { get }

    func start()
    func pause()
    func resume()
    func stop()

    var gameType: GameType { get }
    var frameDuration: TimeInterval { get }
}

public protocol VideoManager {
    var videoFormat: VideoFormat { get }
    var videoBuffer: UnsafeMutablePointer<UInt8> { get }
    func prepare()
    func processFrame()
}

public protocol AudioManager {
    var audioFormat: AVAudioFormat { get }
    func start()
    func stop()
}
```

**バーチャルコントローラー**:
```swift
// カスタムタッチコントローラー
class ControllerView: UIView {
    private var buttonViews: [ControllerButton: UIView] = [:]
    private var dpadView: DPadView?

    weak var delegate: ControllerViewDelegate?

    override func touchesBegan(_ touches: Set<UITouch>, with event: UIEvent?) {
        for touch in touches {
            let location = touch.location(in: self)

            // D-Pad チェック
            if let direction = dpadView?.direction(at: location) {
                delegate?.controllerView(self, didActivate: direction)
            }

            // ボタンチェック
            for (button, view) in buttonViews {
                if view.frame.contains(location) {
                    activateButton(button)
                    break
                }
            }
        }
    }

    private func activateButton(_ button: ControllerButton) {
        delegate?.controllerView(self, didActivate: button)
        UIImpactFeedbackGenerator(style: .light).impactOccurred()
    }
}
```

**ゲームライブラリ管理**:
```swift
// Core Data ベースのゲーム管理
class GameLibrary: ObservableObject {
    @Published var games: [Game] = []

    private let persistentContainer: NSPersistentContainer

    func importGame(at url: URL) throws {
        let context = persistentContainer.viewContext

        // ROMファイルをコピー
        let gamesDirectory = FileManager.default.gamesDirectory
        let destinationURL = gamesDirectory.appendingPathComponent(url.lastPathComponent)
        try FileManager.default.copyItem(at: url, to: destinationURL)

        // Core Data エントリ作成
        let game = Game(context: context)
        game.identifier = UUID()
        game.name = url.deletingPathExtension().lastPathComponent
        game.filename = url.lastPathComponent
        game.type = GameType(fileExtension: url.pathExtension)
        game.artworkURL = fetchArtwork(for: game)

        try context.save()
        fetchGames()
    }

    private func fetchArtwork(for game: Game) -> URL? {
        // ゲームデータベースからアートワーク取得
        let api = GameDatabaseAPI()
        return api.artwork(for: game.name)
    }
}
```

**セーブステート管理**:
```swift
// セーブステート機能
class SaveStateManager {
    let game: Game
    private let core: EmulatorCore

    func saveState(slot: Int) throws {
        let stateData = core.saveState()

        let saveStateURL = saveStateURL(for: slot)
        try stateData.write(to: saveStateURL)

        // スクリーンショット保存
        let screenshot = core.videoManager.captureScreenshot()
        let screenshotURL = screenshotURL(for: slot)
        try screenshot.pngData()?.write(to: screenshotURL)

        // メタデータ更新
        updateMetadata(slot: slot, date: Date())
    }

    func loadState(slot: Int) throws {
        let saveStateURL = saveStateURL(for: slot)
        let stateData = try Data(contentsOf: saveStateURL)
        try core.loadState(stateData)
    }

    func quickSave() throws {
        try saveState(slot: 0)
    }

    func quickLoad() throws {
        try loadState(slot: 0)
    }
}
```

---

## プロジェクト評価

| 観点 | 評価 | 理由 |
|------|------|------|
| **コード品質** | ★★★★★ | モダンSwift設計 |
| **ドキュメント** | ★★★★☆ | 開発者ガイド |
| **アクティビティ** | ★★★★★ | 活発な開発 |
| **学習価値** | ★★★★★ | プロトコル指向設計 |
| **実用性** | ★★★★★ | 人気エミュレータ |

---

## AltStore連携

### サイドロード配布

```swift
// AltStore ソース情報
struct AltStoreSource: Codable {
    let name: String
    let identifier: String
    let apps: [AltStoreApp]
}

struct AltStoreApp: Codable {
    let name: String
    let bundleIdentifier: String
    let version: String
    let downloadURL: URL
    let iconURL: URL
}
```

---

## Metal レンダリング

```swift
// 効率的なビデオ出力
class DeltaMetalView: MTKView {
    private var videoBuffer: MTLBuffer?
    private var renderPipeline: MTLRenderPipelineState?

    func updateFrame(_ buffer: UnsafeMutableRawPointer, bytesPerRow: Int) {
        // ゼロコピーでGPUバッファ更新
        videoBuffer?.contents().copyMemory(
            from: buffer,
            byteCount: bytesPerRow * Int(bounds.height)
        )
        setNeedsDisplay()
    }
}
```

---

## 開発者向け情報

### ビルド手順

```bash
git clone https://github.com/rileytestut/Delta
cd Delta
git submodule update --init --recursive
open Delta.xcworkspace
```

### 必要要件

- Xcode 15+
- DeltaCore フレームワーク
- 各システムのエミュレータコア

---

## まとめ

Deltaは、プロトコル指向設計とモダンSwiftの優れた実装例。エミュレータコアの抽象化、美しいUI、AltStore連携など、実践的なパターンを学べる。Riley Testut氏の設計哲学が反映された高品質なプロジェクト。

**推奨対象**:
- プロトコル指向設計を学びたい開発者
- バーチャルコントローラーを実装したい人
- AltStore連携アプリを開発したい人
- ゲームアプリのUI設計を参考にしたい人
