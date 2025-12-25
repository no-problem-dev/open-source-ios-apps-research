# Apple公式サンプルコード分析

**調査日**: 2025年12月25日
**対象**: 18件のApple公式サンプル

---

## 概要

Apple Developer Documentationおよび公式GitHubリポジトリから提供されるサンプルコードは、ベストプラクティスの参照実装として非常に価値が高い。

---

## GitHub公開サンプル

### Food Truck（1,832★）

**URL**: https://github.com/apple/sample-food-truck

**特徴**:
- マルチプラットフォーム（iOS, iPadOS, macOS）
- 単一コードベース
- SwiftUI全面採用
- 実践的なビジネスアプリ

**学習ポイント**:
- プラットフォーム適応レイアウト
- ナビゲーション構造
- データモデル設計
- アクセシビリティ

```swift
// マルチプラットフォーム対応例
struct ContentView: View {
    var body: some View {
        #if os(iOS)
        iOSLayout()
        #elseif os(macOS)
        macOSLayout()
        #endif
    }
}
```

---

### Backyard Birds（601★）

**URL**: https://github.com/apple/sample-backyard-birds

**特徴**:
- SwiftData使用
- ウィジェット実装
- App内課金
- iOS 17新機能

**学習ポイント**:
- SwiftData永続化
- ウィジェットタイムライン
- StoreKit 2
- インタラクティブウィジェット

```swift
// SwiftData使用例
@Model
class Bird {
    var name: String
    var species: String
    var lastSeen: Date

    init(name: String, species: String) {
        self.name = name
        self.species = species
        self.lastSeen = Date()
    }
}
```

---

## Developer Documentation サンプル

### Landmarks

**URL**: https://developer.apple.com/tutorials/swiftui/creating-and-combining-views

**特徴**:
- SwiftUI入門
- ビュー構成
- データバインディング
- リスト表示

**学習順序**:
1. ビュー作成と結合
2. リストとナビゲーション
3. ユーザー入力処理
4. 描画とアニメーション

---

### Scrumdinger

**URL**: https://developer.apple.com/tutorials/app-dev-training/

**特徴**:
- 実践的なアプリ開発
- データ永続化
- 音声機能
- ライフサイクル管理

**学習ポイント**:
- @State, @Binding, @ObservableObject
- Codableでのデータ保存
- AVFoundation使用
- エラーハンドリング

---

### Fruta

**URL**: https://developer.apple.com/documentation/appclip/fruta-building-a-feature-rich-app-with-swiftui

**特徴**:
- App Clip実装
- ウィジェット
- マルチプラットフォーム
- HealthKit連携

**学習ポイント**:
- App Clipターゲット設定
- 共有コードの構造化
- ウィジェットとApp Clip連携

---

### UIKit Catalog

**URL**: https://developer.apple.com/documentation/uikit/uikit-catalog-creating-and-customizing-views-and-controls

**特徴**:
- 全UIKitコンポーネント
- カスタマイズ例
- アクセシビリティ

**学習ポイント**:
- 標準コンポーネント使用法
- カスタム外観設定
- アクセシビリティ対応

---

## visionOS サンプル

### Hello World

**URL**: https://developer.apple.com/documentation/visionOS/World

**特徴**:
- ウィンドウ、ボリューム、イマーシブスペース
- 地球学習アプリ
- 基本的なvisionOS概念

**学習ポイント**:
- WindowGroup, WindowStyle
- RealityView
- ImmersiveSpace

---

### BOT-anist

**URL**: https://developer.apple.com/documentation/visionos/bot-anist

**特徴**:
- マルチプラットフォームvisionOSアプリ
- ウィンドウとボリューム
- アニメーション

**学習ポイント**:
- Xcode 16対応
- Swift 6機能
- 3Dアニメーション

---

### Destination Video

**URL**: https://developer.apple.com/documentation/visionos/destination-video

**特徴**:
- イマーシブメディア体験
- 動画再生
- 空間オーディオ

**学習ポイント**:
- visionOSでの動画
- イマーシブ環境
- AVPlayerとの統合

---

### Particles

**URL**: https://developer.apple.com/documentation/realitykit/simulating-particles-in-your-visionos-app

**特徴**:
- RealityKitパーティクル
- 視覚エフェクト

---

### Physics

**URL**: https://developer.apple.com/documentation/realitykit/simulating-physics-with-collisions-in-your-visionos-app

**特徴**:
- 物理シミュレーション
- 衝突検出
- RealityKit物理エンジン

---

## その他の重要サンプル

### SwiftUI Navigation Cookbook

**URL**: https://developer.apple.com/documentation/swiftui/bringing_robust_navigation_structure_to_your_swiftui_app

**学習ポイント**:
- NavigationStack
- NavigationPath
- プログラマティックナビゲーション
- ディープリンク

---

### Tab Navigation Enhancement

**URL**: https://developer.apple.com/documentation/swiftui/enhancing-your-app-content-with-tab-navigation

**学習ポイント**:
- TabView
- タブカスタマイズ
- iPad サイドバー対応

---

### RoomPlan

**URL**: https://developer.apple.com/documentation/roomplan/create-a-3d-model-of-an-interior-room-by-guiding-the-user-through-an-ar-experience

**学習ポイント**:
- RoomPlan API
- 部屋のスキャン
- 3Dモデル生成
- ARKit統合

---

### Physics Joints

**URL**: https://developer.apple.com/documentation/realitykit/simulating-physics-joints-in-your-realitykit-app

**学習ポイント**:
- RealityKit物理ジョイント
- 物理制約
- 関節シミュレーション

---

## MLX Examples（2,345★）

**URL**: https://github.com/ml-explore/mlx-swift-examples

**特徴**:
- Apple MLXフレームワーク
- ローカルML推論
- Swift統合

**学習ポイント**:
- MLXのSwift使用
- モデルロード
- 推論実行

---

## 学習推奨順序

### SwiftUI初心者

1. **Landmarks** - 基礎概念
2. **Scrumdinger** - 実践的アプリ
3. **Food Truck** - マルチプラットフォーム
4. **Backyard Birds** - 最新機能

### 中級者

1. **Navigation Cookbook** - ナビゲーション深堀り
2. **Fruta** - App Clip + ウィジェット
3. **UIKit Catalog** - UIKit知識

### visionOS

1. **Hello World** - 基礎
2. **BOT-anist** - 中級
3. **Destination Video** - メディア
4. **Particles/Physics** - RealityKit

---

## サンプルから学ぶベストプラクティス

### コード構造

```
App/
├── Views/
│   ├── ContentView.swift
│   └── Components/
├── Models/
│   └── DataModel.swift
├── ViewModels/
│   └── AppViewModel.swift
├── Services/
│   └── DataService.swift
└── Resources/
    └── Assets.xcassets
```

### 命名規則

| 種類 | 規則 | 例 |
|------|------|-----|
| View | 名詞 + View | ContentView |
| Model | 名詞 | Bird, Order |
| ViewModel | 名詞 + ViewModel | OrderViewModel |
| Protocol | 形容詞的 | Identifiable |

### アクセシビリティ

```swift
// Appleサンプルで共通
.accessibilityLabel("説明")
.accessibilityHint("操作のヒント")
.accessibilityAddTraits(.isButton)
```

---

## サンプル活用のコツ

### 1. 段階的学習
- 小さいサンプルから開始
- 一度に1機能に集中
- 改造して実験

### 2. コードリーディング
- ファイル構造を理解
- データフローを追跡
- コメントを読む

### 3. 応用
- 自分のプロジェクトに適用
- パターンを抽出
- カスタマイズ

---

## まとめ

Apple公式サンプルは：

1. **ベストプラクティスの宝庫** - Apple推奨の書き方
2. **最新機能の参照実装** - 新API使用例
3. **実践的な設計パターン** - 再利用可能な構造
4. **アクセシビリティの手本** - 正しい実装例

定期的にApple Developer Documentationをチェックし、新しいサンプルを活用することを推奨。
