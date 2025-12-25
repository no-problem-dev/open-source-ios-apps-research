# Provenance

**スター**: 5,800+ ⭐
**リポジトリ**: https://github.com/Provenance-Emu/Provenance
**ライセンス**: BSD-3-Clause

---

## 概要

Provenanceは、iOS/tvOS向けのマルチシステムエミュレータフロントエンド。レトロゲームコンソール（NES、SNES、Genesis、PlayStation等）を統合し、美しいUIとコントローラーサポートを提供。

---

## 主要機能

| 機能 | 説明 |
|------|------|
| マルチシステム | 30以上のコンソール対応 |
| コントローラー | MFi、PS、Xboxコントローラー |
| セーブステート | クイックセーブ/ロード |
| クラウド同期 | iCloud セーブ同期 |
| スクリーンショット | ゲーム画面キャプチャ |
| tvOS | Apple TV対応 |

---

## 技術スタック

| 項目 | 技術 |
|------|------|
| 言語 | Swift, Objective-C, C |
| UI | UIKit |
| エミュレーション | 各種コア（C/C++） |
| グラフィック | Metal, OpenGL ES |
| オーディオ | AVAudioEngine |
| 入力 | GameController |

---

## アーキテクチャ

```
Provenance/
├── Provenance/              # メインアプリ
│   ├── UI/                  # UIコンポーネント
│   ├── Game Library/        # ゲームライブラリ
│   └── Settings/            # 設定
├── PVLibrary/               # ライブラリ管理
├── PVSupport/               # サポートフレームワーク
├── PVEmulatorCore/          # エミュレータ基盤
│   └── Cores/               # 各エミュレータコア
│       ├── PVNES/           # NES
│       ├── PVSNES/          # SNES
│       ├── PVGenesis/       # Genesis
│       └── ...
└── PVCoreAudio/             # オーディオエンジン
```

---

## 学習価値

### エミュレータ統合

| 分野 | 学習ポイント |
|------|-------------|
| **Metal** | 低レベルグラフィック |
| **GameController** | コントローラー統合 |
| **C/C++統合** | ネイティブライブラリ連携 |
| **オーディオ** | 低レイテンシ音声 |
| **tvOS** | Apple TV開発 |

### コードパターン

**エミュレータコアプロトコル**:
```swift
// エミュレータコア基底
@objc protocol PVEmulatorCore {
    var videoBufferSize: CGSize { get }
    var frameInterval: TimeInterval { get }
    var audioSampleRate: Double { get }

    func loadFile(atPath path: String) throws
    func startEmulation()
    func pauseEmulation()
    func stopEmulation()

    func executeFrame()

    // セーブステート
    func saveStateToFile(atPath path: String) throws
    func loadStateFromFile(atPath path: String) throws

    // 入力
    func didPush(_ button: Int, forPlayer player: Int)
    func didRelease(_ button: Int, forPlayer player: Int)
}
```

**Metal レンダリング**:
```swift
// Metal ベースのビデオ出力
class PVMetalView: MTKView {
    private var commandQueue: MTLCommandQueue?
    private var pipelineState: MTLRenderPipelineState?
    private var videoTexture: MTLTexture?

    func setupMetal() {
        guard let device = MTLCreateSystemDefaultDevice() else { return }
        self.device = device
        commandQueue = device.makeCommandQueue()

        let library = device.makeDefaultLibrary()
        let pipelineDescriptor = MTLRenderPipelineDescriptor()
        pipelineDescriptor.vertexFunction = library?.makeFunction(name: "vertexShader")
        pipelineDescriptor.fragmentFunction = library?.makeFunction(name: "fragmentShader")
        pipelineDescriptor.colorAttachments[0].pixelFormat = .bgra8Unorm

        pipelineState = try? device.makeRenderPipelineState(descriptor: pipelineDescriptor)
    }

    func updateVideoBuffer(_ buffer: UnsafeMutableRawPointer, size: CGSize) {
        let descriptor = MTLTextureDescriptor.texture2DDescriptor(
            pixelFormat: .bgra8Unorm,
            width: Int(size.width),
            height: Int(size.height),
            mipmapped: false
        )
        videoTexture = device?.makeTexture(descriptor: descriptor)
        videoTexture?.replace(
            region: MTLRegion(origin: .init(), size: MTLSize(width: Int(size.width), height: Int(size.height), depth: 1)),
            mipmapLevel: 0,
            withBytes: buffer,
            bytesPerRow: Int(size.width) * 4
        )
    }

    override func draw(_ rect: CGRect) {
        guard let drawable = currentDrawable,
              let commandBuffer = commandQueue?.makeCommandBuffer(),
              let renderPassDescriptor = currentRenderPassDescriptor else { return }

        let encoder = commandBuffer.makeRenderCommandEncoder(descriptor: renderPassDescriptor)
        encoder?.setRenderPipelineState(pipelineState!)
        encoder?.setFragmentTexture(videoTexture, index: 0)
        encoder?.drawPrimitives(type: .triangleStrip, vertexStart: 0, vertexCount: 4)
        encoder?.endEncoding()

        commandBuffer.present(drawable)
        commandBuffer.commit()
    }
}
```

**GameController統合**:
```swift
// コントローラー入力処理
class GameControllerManager: ObservableObject {
    @Published var controllers: [GCController] = []

    private var core: PVEmulatorCore?

    init() {
        setupNotifications()
    }

    private func setupNotifications() {
        NotificationCenter.default.addObserver(
            self,
            selector: #selector(controllerConnected),
            name: .GCControllerDidConnect,
            object: nil
        )
    }

    @objc private func controllerConnected(_ notification: Notification) {
        guard let controller = notification.object as? GCController else { return }
        setupController(controller)
        controllers.append(controller)
    }

    private func setupController(_ controller: GCController) {
        controller.extendedGamepad?.valueChangedHandler = { [weak self] gamepad, element in
            self?.handleInput(gamepad: gamepad, element: element)
        }
    }

    private func handleInput(gamepad: GCExtendedGamepad, element: GCControllerElement) {
        // ボタンマッピング
        if gamepad.buttonA.isPressed {
            core?.didPush(PVButton.A.rawValue, forPlayer: 0)
        } else {
            core?.didRelease(PVButton.A.rawValue, forPlayer: 0)
        }
        // ... 他のボタン
    }
}
```

---

## プロジェクト評価

| 観点 | 評価 | 理由 |
|------|------|------|
| **コード品質** | ★★★★☆ | レガシーコード含む |
| **ドキュメント** | ★★★★☆ | ビルドガイド充実 |
| **アクティビティ** | ★★★★☆ | 継続的なメンテナンス |
| **学習価値** | ★★★★★ | Metal/GameController学習 |
| **実用性** | ★★★★★ | 人気エミュレータ |

---

## 対応システム

### 主要コンソール

| システム | エンジン |
|---------|---------|
| NES | FCEUX |
| SNES | SNES9x |
| Genesis | Genesis Plus GX |
| Game Boy | Gambatte |
| PlayStation | Mednafen |
| N64 | Mupen64Plus |

---

## tvOS対応

```swift
// tvOS フォーカス対応
class GameCollectionViewController: UIViewController {
    #if os(tvOS)
    override var preferredFocusEnvironments: [UIFocusEnvironment] {
        return [collectionView]
    }

    func collectionView(
        _ collectionView: UICollectionView,
        didUpdateFocusIn context: UICollectionViewFocusUpdateContext,
        with coordinator: UIFocusAnimationCoordinator
    ) {
        coordinator.addCoordinatedAnimations {
            context.nextFocusedView?.transform = CGAffineTransform(scaleX: 1.1, y: 1.1)
            context.previouslyFocusedView?.transform = .identity
        }
    }
    #endif
}
```

---

## 開発者向け情報

### ビルド手順

```bash
git clone https://github.com/Provenance-Emu/Provenance
cd Provenance
git submodule update --init --recursive
open Provenance.xcworkspace
```

### 必要要件

- Xcode 15+
- CocoaPods
- 各エミュレータコアの依存関係

---

## まとめ

Provenanceは、Metal、GameController、C/C++統合など、高度なiOS開発技術を学べるプロジェクト。エミュレータの統合パターンは、低レベルAPIの活用方法として参考になる。tvOS対応も含め、マルチプラットフォーム開発の実例。

**推奨対象**:
- Metal/OpenGLを学びたい開発者
- GameController統合を実装したい人
- C/C++ライブラリ統合を学びたい人
- tvOS開発に興味がある人
