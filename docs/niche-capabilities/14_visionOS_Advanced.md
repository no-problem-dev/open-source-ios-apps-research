# visionOS高度な技法

**驚きポイント**: 動的メッシュ生成、空間UIカスタマイズ

## 参考プロジェクト

| プロジェクト | Stars | 特徴 |
|-------------|-------|------|
| [LowLevelMesh](https://github.com/metal-by-example/metal-spatial-dynamic-mesh) | 101 | 動的メッシュ生成 |
| [SpatialDock](https://github.com/kjwamlex/SpatialDock) | 77 | 空間UI配置 |
| [VisionPro Vacuum](https://github.com/gonchar/VisionProVacuumDemo) | 73 | ARKit+RealityKit |

---

## 1. LowLevelMesh（動的メッシュ）

### 基本構造

```swift
import RealityKit
import Metal

class DynamicMeshGenerator {
    let device: MTLDevice
    var mesh: LowLevelMesh?

    init() {
        device = MTLCreateSystemDefaultDevice()!
    }

    func createMesh(vertexCount: Int, indexCount: Int) throws -> LowLevelMesh {
        var descriptor = LowLevelMesh.Descriptor()

        // 頂点属性
        descriptor.vertexAttributes = [
            .init(
                semantic: .position,
                format: .float3,
                offset: 0
            ),
            .init(
                semantic: .normal,
                format: .float3,
                offset: MemoryLayout<SIMD3<Float>>.stride
            ),
            .init(
                semantic: .uv0,
                format: .float2,
                offset: MemoryLayout<SIMD3<Float>>.stride * 2
            )
        ]

        descriptor.vertexLayouts = [
            .init(bufferIndex: 0, bufferStride: MemoryLayout<Vertex>.stride)
        ]

        descriptor.indexType = .uint32

        // 容量設定
        descriptor.vertexCapacity = vertexCount
        descriptor.indexCapacity = indexCount

        return try LowLevelMesh(descriptor: descriptor)
    }

    struct Vertex {
        var position: SIMD3<Float>
        var normal: SIMD3<Float>
        var uv: SIMD2<Float>
    }
}
```

### 頂点更新

```swift
extension DynamicMeshGenerator {
    func updateVertices(time: Float) {
        guard let mesh = mesh else { return }

        mesh.withUnsafeMutableBytes(bufferIndex: 0) { buffer in
            let vertices = buffer.bindMemory(to: Vertex.self)

            for i in 0..<vertices.count {
                // 波のアニメーション
                let x = vertices[i].position.x
                let z = vertices[i].position.z
                vertices[i].position.y = sin(x * 2 + time) * cos(z * 2 + time) * 0.2
            }
        }

        // バウンディングボックス更新
        mesh.parts.replaceAll([
            LowLevelMesh.Part(
                indexCount: mesh.indexCount,
                topology: .triangle,
                bounds: BoundingBox(min: [-1, -1, -1], max: [1, 1, 1])
            )
        ])
    }
}
```

### RealityKitで使用

```swift
import RealityKit

@MainActor
class WaveMeshEntity {
    let entity: Entity

    init() async throws {
        let generator = DynamicMeshGenerator()
        let lowLevelMesh = try generator.createMesh(
            vertexCount: 10000,
            indexCount: 60000
        )

        // MeshResourceに変換
        let meshResource = try await MeshResource(from: lowLevelMesh)

        // マテリアル
        var material = SimpleMaterial()
        material.color = .init(tint: .blue)

        // Entity作成
        entity = Entity()
        entity.components.set(ModelComponent(
            mesh: meshResource,
            materials: [material]
        ))
    }

    func update(time: Float) async throws {
        // メッシュ更新
    }
}
```

---

## 2. SpatialDock（空間UI配置）

### ユーザーの視線追従

```swift
import SwiftUI
import RealityKit

struct FollowGazeView: View {
    @State private var headPosition: SIMD3<Float> = .zero

    var body: some View {
        RealityView { content in
            // ARKitからヘッドトラッキング
        } update: { content in
            // ドックを視界の下部に配置
            let dockOffset = SIMD3<Float>(0, -0.5, -1)
            // 位置更新
        }
    }
}
```

### Ornament配置

```swift
struct WindowWithDock: View {
    var body: some View {
        NavigationStack {
            ContentView()
        }
        .ornament(
            visibility: .visible,
            attachmentAnchor: .scene(.bottom)
        ) {
            HStack(spacing: 20) {
                Button(action: {}) {
                    Image(systemName: "house")
                }
                Button(action: {}) {
                    Image(systemName: "gear")
                }
            }
            .padding()
            .glassBackgroundEffect()
        }
    }
}
```

---

## 3. Hand Tracking

```swift
import ARKit
import RealityKit

class HandTracker: ObservableObject {
    let session = ARKitSession()
    let handTracking = HandTrackingProvider()

    @Published var leftHandPosition: SIMD3<Float>?
    @Published var rightHandPosition: SIMD3<Float>?

    func start() async {
        guard HandTrackingProvider.isSupported else { return }

        try? await session.run([handTracking])

        for await update in handTracking.anchorUpdates {
            let anchor = update.anchor

            guard anchor.isTracked else { continue }

            let handSkeleton = anchor.handSkeleton

            // 人差し指の先端
            if let indexTip = handSkeleton?.joint(.indexFingerTip) {
                let position = anchor.originFromAnchorTransform * indexTip.anchorFromJointTransform

                await MainActor.run {
                    if anchor.chirality == .left {
                        leftHandPosition = position.columns.3.xyz
                    } else {
                        rightHandPosition = position.columns.3.xyz
                    }
                }
            }
        }
    }
}

extension simd_float4 {
    var xyz: SIMD3<Float> {
        SIMD3(x, y, z)
    }
}
```

---

## 4. Immersive Space内でのポータル

```swift
import RealityKit
import SwiftUI

struct PortalView: View {
    var body: some View {
        RealityView { content in
            // ポータルフレーム
            let portalFrame = ModelEntity(
                mesh: .generateBox(size: [1.5, 2, 0.1]),
                materials: [SimpleMaterial(color: .gray, isMetallic: true)]
            )

            // ポータル内のワールド
            let portalWorld = Entity()
            portalWorld.components.set(WorldComponent())

            // ポータル内にコンテンツ追加
            let skybox = try! Entity.load(named: "BeachSkybox")
            portalWorld.addChild(skybox)

            // ポータルコンポーネント設定
            portalFrame.components.set(PortalComponent(target: portalWorld))

            content.add(portalFrame)
            content.add(portalWorld)
        }
    }
}
```

---

## 5. Spatial Audio

```swift
import RealityKit
import AVFoundation

class SpatialAudioManager {
    func attachAudioToEntity(_ entity: Entity, audioFile: String) {
        guard let url = Bundle.main.url(forResource: audioFile, withExtension: "mp3") else {
            return
        }

        let audioSource = AudioFileResource.load(
            contentsOf: url,
            withName: audioFile,
            inputMode: .spatial,
            loadingStrategy: .stream,
            shouldLoop: true
        )

        // 空間オーディオコンポーネント
        let audioComponent = SpatialAudioComponent(resource: audioSource)
        entity.components.set(audioComponent)

        // 減衰設定
        var spatialAudio = SpatialAudioComponent()
        spatialAudio.gain = 0 // dB
        spatialAudio.distanceAttenuation = .rolloff(factor: 1.0)
        spatialAudio.directivity = .beam(focus: 0.5)

        entity.playAudio(audioSource)
    }
}
```

---

## 6. Collision & Physics

```swift
import RealityKit

class PhysicsWorld {
    func setupPhysicsEntity() -> Entity {
        let entity = Entity()

        // 衝突形状
        let shape = ShapeResource.generateBox(size: [0.5, 0.5, 0.5])
        entity.components.set(CollisionComponent(shapes: [shape]))

        // 物理ボディ
        var physics = PhysicsBodyComponent(
            shapes: [shape],
            mass: 1.0,
            mode: .dynamic
        )
        physics.material = .init(
            staticFriction: 0.5,
            dynamicFriction: 0.3,
            restitution: 0.8
        )
        entity.components.set(physics)

        return entity
    }

    func applyForce(to entity: Entity, direction: SIMD3<Float>) {
        guard var physics = entity.components[PhysicsBodyComponent.self] else { return }

        // 力を適用
        entity.applyImpulse(direction * 10, at: .zero, relativeTo: nil)
    }
}
```

---

## まとめ

| 技術 | 用途 | 難易度 |
|------|------|--------|
| LowLevelMesh | 動的ジオメトリ | 高 |
| Hand Tracking | ジェスチャー入力 | 中 |
| Portal | 空間内ポータル | 中 |
| Spatial Audio | 3Dサウンド | 低 |
| Physics | 物理シミュレーション | 中 |

### visionOS固有の考慮点

1. **視界の快適さ**: ユーザーの首が疲れない配置
2. **奥行き知覚**: 適切な距離感
3. **没入度調整**: Shared → Full Immersion
4. **パフォーマンス**: 90FPS維持
