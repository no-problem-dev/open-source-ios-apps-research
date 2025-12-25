# visionOS 基礎 - 空間コンピューティング入門

**参考プロジェクト**:
- Apple公式: Hello World, Destination Video
- SwiftUI for visionOS サンプル

**実装参考度**: ⭐⭐⭐⭐⭐

---

## なぜ重要か

visionOSはAppleの**空間コンピューティング**プラットフォーム。SwiftUIの知識を活かしながら、3D空間でのアプリ開発が可能。

| 学習ポイント | 重要度 | 理由 |
|-------------|--------|------|
| **Window** | ⭐⭐⭐⭐⭐ | 基本的な2D UI |
| **Volume** | ⭐⭐⭐⭐⭐ | 3Dコンテンツ表示 |
| **ImmersiveSpace** | ⭐⭐⭐⭐⭐ | 没入型体験 |
| **RealityKit** | ⭐⭐⭐⭐☆ | 3Dレンダリング |
| **Ornament** | ⭐⭐⭐⭐☆ | 補助UI要素 |

---

## 1. アプリ構造

### 基本的なApp定義

```swift
import SwiftUI

@main
struct MyVisionApp: App {
    var body: some Scene {
        // 2Dウィンドウ
        WindowGroup {
            ContentView()
        }

        // 3Dボリューム
        WindowGroup(id: "globe") {
            GlobeView()
        }
        .windowStyle(.volumetric)
        .defaultSize(width: 0.6, height: 0.6, depth: 0.6, in: .meters)

        // 没入空間
        ImmersiveSpace(id: "immersive") {
            ImmersiveView()
        }
        .immersionStyle(selection: .constant(.mixed), in: .mixed, .full)
    }
}
```

### Window（2D UI）

```swift
struct ContentView: View {
    @Environment(\.openWindow) private var openWindow
    @Environment(\.openImmersiveSpace) private var openImmersiveSpace
    @Environment(\.dismissImmersiveSpace) private var dismissImmersiveSpace

    var body: some View {
        NavigationStack {
            List {
                Button("Open Globe") {
                    openWindow(id: "globe")
                }

                Button("Enter Immersive") {
                    Task {
                        await openImmersiveSpace(id: "immersive")
                    }
                }
            }
            .navigationTitle("My Vision App")
        }
    }
}
```

---

## 2. Volume（3Dコンテンツ）

### 基本的なVolume

```swift
import SwiftUI
import RealityKit

struct GlobeView: View {
    var body: some View {
        RealityView { content in
            // 地球モデルを追加
            let earth = try? await ModelEntity(named: "Earth")
            if let earth = earth {
                earth.scale = [0.5, 0.5, 0.5]
                content.add(earth)
            }
        }
    }
}
```

### インタラクティブな3Dオブジェクト

```swift
struct InteractiveGlobeView: View {
    @State private var rotation: Angle = .zero
    @State private var scale: Float = 1.0

    var body: some View {
        RealityView { content in
            let earth = try? await ModelEntity(named: "Earth")
            if let earth = earth {
                // 入力コンポーネントを追加
                earth.components.set(InputTargetComponent())
                earth.generateCollisionShapes(recursive: true)

                content.add(earth)
            }
        } update: { content in
            if let earth = content.entities.first {
                earth.transform.rotation = simd_quatf(
                    angle: Float(rotation.radians),
                    axis: [0, 1, 0]
                )
                earth.transform.scale = [scale, scale, scale]
            }
        }
        .gesture(
            DragGesture()
                .onChanged { value in
                    rotation = Angle(degrees: value.translation.width)
                }
        )
        .gesture(
            MagnifyGesture()
                .onChanged { value in
                    scale = Float(value.magnification)
                }
        )
    }
}
```

---

## 3. ImmersiveSpace（没入空間）

### 没入スタイルの種類

```swift
// Mixed: 現実世界と仮想コンテンツが共存
ImmersiveSpace(id: "mixed") {
    MixedImmersiveView()
}
.immersionStyle(selection: .constant(.mixed), in: .mixed)

// Progressive: 部分的に没入（調整可能）
ImmersiveSpace(id: "progressive") {
    ProgressiveImmersiveView()
}
.immersionStyle(selection: .constant(.progressive), in: .progressive)

// Full: 完全没入
ImmersiveSpace(id: "full") {
    FullImmersiveView()
}
.immersionStyle(selection: .constant(.full), in: .full)
```

### 没入空間のView

```swift
struct ImmersiveView: View {
    var body: some View {
        RealityView { content in
            // 環境を構築
            let floor = ModelEntity(
                mesh: .generatePlane(width: 10, depth: 10),
                materials: [SimpleMaterial(color: .gray, isMetallic: false)]
            )
            floor.position = [0, 0, 0]
            content.add(floor)

            // 空間内にオブジェクトを配置
            for i in 0..<5 {
                let sphere = ModelEntity(
                    mesh: .generateSphere(radius: 0.2),
                    materials: [SimpleMaterial(color: .blue, isMetallic: true)]
                )
                sphere.position = [Float(i) - 2, 1.5, -2]
                sphere.components.set(InputTargetComponent())
                sphere.generateCollisionShapes(recursive: true)
                content.add(sphere)
            }
        }
    }
}
```

---

## 4. Ornament（補助UI）

### 基本的なOrnament

```swift
struct ContentView: View {
    @State private var showInfo = false

    var body: some View {
        VStack {
            Text("Main Content")
                .font(.largeTitle)
        }
        .ornament(
            visibility: .visible,
            attachmentAnchor: .scene(.bottom),
            contentAlignment: .center
        ) {
            HStack {
                Button("Info") {
                    showInfo.toggle()
                }
                .buttonStyle(.bordered)

                Button("Settings") {
                    // Settings action
                }
                .buttonStyle(.bordered)
            }
            .padding()
            .glassBackgroundEffect()
        }
    }
}
```

### 複数のOrnament

```swift
struct VideoPlayerView: View {
    @State private var isPlaying = false
    @State private var volume: Float = 0.5

    var body: some View {
        VideoView()
            // 下部にコントロール
            .ornament(attachmentAnchor: .scene(.bottom)) {
                HStack {
                    Button(action: { isPlaying.toggle() }) {
                        Image(systemName: isPlaying ? "pause.fill" : "play.fill")
                    }

                    Slider(value: $volume)
                        .frame(width: 200)
                }
                .padding()
                .glassBackgroundEffect()
            }
            // 右側に情報
            .ornament(attachmentAnchor: .scene(.trailing)) {
                VStack {
                    Text("Now Playing")
                    Text("Video Title")
                        .font(.headline)
                }
                .padding()
                .glassBackgroundEffect()
            }
    }
}
```

---

## 5. ジェスチャーと入力

### visionOS固有のジェスチャー

```swift
struct GestureView: View {
    @State private var selectedEntity: Entity?

    var body: some View {
        RealityView { content in
            // 3Dコンテンツをセットアップ
        }
        // タップ（視線 + ピンチ）
        .gesture(
            TapGesture()
                .targetedToAnyEntity()
                .onEnded { value in
                    print("Tapped: \(value.entity.name)")
                }
        )
        // ドラッグ
        .gesture(
            DragGesture()
                .targetedToAnyEntity()
                .onChanged { value in
                    value.entity.position = value.convert(
                        value.location3D,
                        from: .local,
                        to: value.entity.parent!
                    )
                }
        )
        // 長押し
        .gesture(
            LongPressGesture()
                .targetedToAnyEntity()
                .onEnded { value in
                    selectedEntity = value.entity
                }
        )
    }
}
```

### 空間タップジェスチャー

```swift
struct SpatialTapView: View {
    var body: some View {
        RealityView { content in
            // コンテンツ
        }
        .gesture(
            SpatialTapGesture()
                .onEnded { value in
                    // 3D空間での位置を取得
                    let location = value.location3D
                    print("Tapped at: \(location)")
                }
        )
    }
}
```

---

## 6. Glass Effect

### glassBackgroundEffect

```swift
struct GlassEffectView: View {
    var body: some View {
        VStack(spacing: 20) {
            Text("Standard Glass")
                .padding()
                .glassBackgroundEffect()

            Text("Tinted Glass")
                .padding()
                .glassBackgroundEffect(
                    in: RoundedRectangle(cornerRadius: 16)
                )

            // カードスタイル
            VStack(alignment: .leading) {
                Text("Title")
                    .font(.headline)
                Text("Description text goes here")
                    .font(.body)
            }
            .padding()
            .frame(width: 300)
            .glassBackgroundEffect()
        }
    }
}
```

---

## 7. システム統合

### SharePlayとの連携

```swift
import GroupActivities

struct MyActivity: GroupActivity {
    var metadata: GroupActivityMetadata {
        var metadata = GroupActivityMetadata()
        metadata.title = "Watch Together"
        metadata.type = .watchTogether
        return metadata
    }
}

struct SharePlayView: View {
    @StateObject private var groupStateObserver = GroupStateObserver()

    var body: some View {
        VStack {
            if groupStateObserver.isEligibleForGroupSession {
                Button("Start SharePlay") {
                    Task {
                        try await MyActivity().activate()
                    }
                }
            }
        }
    }
}
```

### Persona表示

```swift
struct PersonaView: View {
    var body: some View {
        // Personaを表示するエリアを確保
        ZStack {
            // 他の参加者のPersonaが表示される空間
        }
        .frame(width: 300, height: 300)
    }
}
```

---

## まとめ：visionOS開発

| 概念 | 用途 | 優先度 |
|------|------|--------|
| **Window** | 既存SwiftUIアプリの移植 | 🔴 必須 |
| **Volume** | 3Dコンテンツ表示 | 🔴 必須 |
| **ImmersiveSpace** | 没入体験 | 🟡 推奨 |
| **Ornament** | 補助UI | 🟡 推奨 |
| **RealityKit** | 3D/AR機能 | 🟡 推奨 |

### iOSアプリからの移行ステップ

1. **そのまま動作**: 既存SwiftUIアプリはWindowとして動作
2. **Ornament追加**: コントロールをOrnamentに移動
3. **Volume追加**: 3Dコンテンツがある場合
4. **ImmersiveSpace**: 没入体験が必要な場合

### 設計原則

- **グレアを避ける**: 明るすぎる背景は避ける
- **適切な距離**: コンテンツは0.5m〜3mの距離に配置
- **視野角を考慮**: 重要な情報は中央30度以内に
- **休憩を促す**: 長時間の没入は避ける設計
