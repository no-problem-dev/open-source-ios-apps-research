# RealityKit 3D 実装ガイド

**参考プロジェクト**:
- Apple公式: Hello World, Destination Video
- AR Quick Look サンプル

**実装参考度**: ⭐⭐⭐⭐⭐

---

## なぜ重要か

RealityKitは**AR/VR 3Dコンテンツ**を作成するフレームワーク。visionOSとiOS ARKit両方で使用可能。

| 学習ポイント | 重要度 | 理由 |
|-------------|--------|------|
| **Entity** | ⭐⭐⭐⭐⭐ | 3Dオブジェクトの基本 |
| **Component** | ⭐⭐⭐⭐⭐ | ECSアーキテクチャ |
| **Material** | ⭐⭐⭐⭐☆ | 見た目のカスタマイズ |
| **Animation** | ⭐⭐⭐⭐☆ | 3Dアニメーション |
| **Physics** | ⭐⭐⭐☆☆ | 物理シミュレーション |

---

## 1. Entity-Component-System (ECS)

### 基本構造

```swift
import RealityKit

// Entity: 3D空間内のオブジェクト
let entity = Entity()

// Component: Entityの振る舞いや属性を定義
entity.components.set(ModelComponent(
    mesh: .generateSphere(radius: 0.1),
    materials: [SimpleMaterial(color: .blue, isMetallic: true)]
))

// Transform: 位置・回転・スケール
entity.transform = Transform(
    scale: [1, 1, 1],
    rotation: simd_quatf(angle: 0, axis: [0, 1, 0]),
    translation: [0, 1, -2]
)
```

### カスタムComponent

```swift
// カスタムコンポーネント定義
struct SpinComponent: Component {
    var speed: Float = 1.0
    var axis: SIMD3<Float> = [0, 1, 0]
}

// システムで処理
class SpinSystem: System {
    static let query = EntityQuery(where: .has(SpinComponent.self))

    required init(scene: RealityKit.Scene) {}

    func update(context: SceneUpdateContext) {
        for entity in context.entities(matching: Self.query, updatingSystemWhen: .rendering) {
            guard let spin = entity.components[SpinComponent.self] else { continue }

            let rotation = simd_quatf(
                angle: spin.speed * Float(context.deltaTime),
                axis: spin.axis
            )
            entity.transform.rotation *= rotation
        }
    }
}

// 登録
SpinComponent.registerComponent()
SpinSystem.registerSystem()

// 使用
entity.components.set(SpinComponent(speed: 2.0))
```

---

## 2. モデルの読み込み

### USDZファイル

```swift
struct ModelLoaderView: View {
    var body: some View {
        RealityView { content in
            // バンドルからモデルを読み込み
            if let model = try? await ModelEntity(named: "robot") {
                model.scale = [0.5, 0.5, 0.5]
                model.position = [0, 0, -2]
                content.add(model)
            }

            // URLから読み込み
            let url = URL(string: "https://example.com/model.usdz")!
            if let remoteModel = try? await ModelEntity(contentsOf: url) {
                content.add(remoteModel)
            }
        }
    }
}
```

### プロシージャルメッシュ

```swift
func createProceduralContent() -> Entity {
    let parent = Entity()

    // 球体
    let sphere = ModelEntity(
        mesh: .generateSphere(radius: 0.2),
        materials: [SimpleMaterial(color: .red, isMetallic: false)]
    )
    sphere.position = [-0.5, 0, 0]
    parent.addChild(sphere)

    // ボックス
    let box = ModelEntity(
        mesh: .generateBox(size: [0.3, 0.3, 0.3]),
        materials: [SimpleMaterial(color: .blue, isMetallic: true)]
    )
    box.position = [0, 0, 0]
    parent.addChild(box)

    // シリンダー
    let cylinder = ModelEntity(
        mesh: .generateCylinder(height: 0.4, radius: 0.1),
        materials: [SimpleMaterial(color: .green, roughness: 0.5)]
    )
    cylinder.position = [0.5, 0, 0]
    parent.addChild(cylinder)

    // 平面
    let plane = ModelEntity(
        mesh: .generatePlane(width: 2, depth: 2),
        materials: [SimpleMaterial(color: .gray, roughness: 0.8)]
    )
    plane.position = [0, -0.5, 0]
    parent.addChild(plane)

    return parent
}
```

---

## 3. Material（マテリアル）

### SimpleMaterial

```swift
// 基本的なマテリアル
let metalMaterial = SimpleMaterial(
    color: .yellow,
    roughness: 0.1,
    isMetallic: true
)

let matteMaterial = SimpleMaterial(
    color: .red,
    roughness: 0.9,
    isMetallic: false
)
```

### PhysicallyBasedMaterial (PBR)

```swift
// フォトリアルなマテリアル
var pbr = PhysicallyBasedMaterial()

// ベースカラー
pbr.baseColor = .init(tint: .white, texture: .init(try! .load(named: "albedo")))

// 法線マップ
pbr.normal = .init(texture: .init(try! .load(named: "normal")))

// 粗さ
pbr.roughness = .init(floatLiteral: 0.3)

// 金属度
pbr.metallic = .init(floatLiteral: 0.8)

// エミッシブ（発光）
pbr.emissiveColor = .init(color: .orange)
pbr.emissiveIntensity = 0.5
```

### UnlitMaterial

```swift
// ライティングの影響を受けないマテリアル
let unlitMaterial = UnlitMaterial(color: .cyan)

// テクスチャ付き
var texturedUnlit = UnlitMaterial()
texturedUnlit.color = .init(
    tint: .white,
    texture: .init(try! .load(named: "texture"))
)
```

### VideoMaterial

```swift
import AVFoundation

// ビデオを3Dサーフェスに表示
let url = Bundle.main.url(forResource: "video", withExtension: "mp4")!
let player = AVPlayer(url: url)

var videoMaterial = VideoMaterial(avPlayer: player)

let screen = ModelEntity(
    mesh: .generatePlane(width: 1.6, height: 0.9),
    materials: [videoMaterial]
)

player.play()
```

---

## 4. アニメーション

### Transform アニメーション

```swift
func animateEntity(_ entity: Entity) {
    // 移動アニメーション
    var transform = entity.transform
    transform.translation = [0, 2, -2]

    entity.move(
        to: transform,
        relativeTo: entity.parent,
        duration: 2.0,
        timingFunction: .easeInOut
    )
}
```

### FromToByAnimation

```swift
// 回転アニメーション
let rotation = FromToByAnimation<Transform>(
    from: Transform(rotation: simd_quatf(angle: 0, axis: [0, 1, 0])),
    to: Transform(rotation: simd_quatf(angle: .pi * 2, axis: [0, 1, 0])),
    duration: 3.0,
    timing: .linear,
    isAdditive: false,
    bindTarget: .transform,
    repeatMode: .repeat
)

let animationResource = try! AnimationResource.generate(with: rotation)
entity.playAnimation(animationResource)
```

### USDZアニメーション

```swift
RealityView { content in
    if let animatedModel = try? await Entity(named: "character") {
        content.add(animatedModel)

        // モデル内のアニメーションを再生
        if let animation = animatedModel.availableAnimations.first {
            animatedModel.playAnimation(animation.repeat())
        }
    }
}
```

---

## 5. 物理シミュレーション

### コリジョン

```swift
// コリジョン形状を生成
entity.generateCollisionShapes(recursive: true)

// 手動でコリジョン設定
entity.components.set(CollisionComponent(
    shapes: [.generateSphere(radius: 0.2)],
    mode: .trigger,  // .default, .trigger, .colliding
    filter: .default
))
```

### PhysicsBodyComponent

```swift
// 動的な物理ボディ
let dynamicBody = PhysicsBodyComponent(
    massProperties: .init(mass: 1.0),
    material: .generate(friction: 0.5, restitution: 0.3),
    mode: .dynamic
)
entity.components.set(dynamicBody)

// 静的な物理ボディ（動かない）
let staticBody = PhysicsBodyComponent(
    massProperties: .default,
    material: .default,
    mode: .static
)
floor.components.set(staticBody)

// キネマティック（アニメーションで動かす）
let kinematicBody = PhysicsBodyComponent(
    massProperties: .default,
    material: .default,
    mode: .kinematic
)
```

### 力を加える

```swift
func applyForce(to entity: Entity) {
    guard var physics = entity.components[PhysicsBodyComponent.self] else { return }

    // 力を加える
    entity.addForce([0, 10, 0], relativeTo: nil)

    // トルクを加える
    entity.addTorque([0, 5, 0], relativeTo: nil)

    // インパルス（瞬間的な力）
    entity.applyLinearImpulse([0, 5, 0], relativeTo: nil)
}
```

---

## 6. インタラクション

### InputTargetComponent

```swift
RealityView { content in
    let sphere = ModelEntity(
        mesh: .generateSphere(radius: 0.2),
        materials: [SimpleMaterial(color: .blue, isMetallic: true)]
    )

    // インタラクション可能にする
    sphere.components.set(InputTargetComponent())
    sphere.generateCollisionShapes(recursive: true)

    content.add(sphere)
}
.gesture(
    TapGesture()
        .targetedToAnyEntity()
        .onEnded { value in
            // タップされたエンティティ
            let tappedEntity = value.entity
            tappedEntity.components[ModelComponent.self]?.materials = [
                SimpleMaterial(color: .red, isMetallic: true)
            ]
        }
)
```

### HoverEffectComponent

```swift
// ホバー時のエフェクト（visionOS）
entity.components.set(HoverEffectComponent())
```

---

## 7. ライティング

### ImageBasedLight

```swift
RealityView { content in
    // 環境ライティング
    let resource = try? await EnvironmentResource(named: "studio")
    if let resource = resource {
        let light = Entity()
        light.components.set(ImageBasedLightComponent(
            source: .single(resource),
            intensityExponent: 10
        ))
        light.components.set(ImageBasedLightReceiverComponent(imageBasedLight: light))
        content.add(light)
    }
}
```

---

## まとめ：RealityKit実装

| 機能 | 用途 | 優先度 |
|------|------|--------|
| **Entity/Component** | 3Dオブジェクト管理 | 🔴 必須 |
| **モデル読み込み** | USDZファイル | 🔴 必須 |
| **Material** | 見た目のカスタマイズ | 🟡 推奨 |
| **Animation** | 動きの追加 | 🟡 推奨 |
| **Physics** | 物理シミュレーション | 🟢 参考 |

### Reality Composer Pro

- Xcodeに統合された3Dシーンエディタ
- ノードベースのマテリアル作成
- アニメーションのタイムライン編集
- コードなしでシーン構築可能

### パフォーマンス最適化

1. **LOD (Level of Detail)**: 距離に応じてメッシュ品質を変更
2. **オクルージョンカリング**: 見えないオブジェクトを描画しない
3. **テクスチャ圧縮**: ASTC形式を使用
4. **バッチング**: 同一マテリアルのオブジェクトをまとめる
