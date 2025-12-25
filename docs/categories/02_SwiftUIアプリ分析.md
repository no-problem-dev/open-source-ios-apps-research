# SwiftUIアプリ詳細分析

**調査日**: 2025年12月25日
**対象**: 非アーカイブSwiftUIアプリ 207件

---

## Tier 1: 本番レベル（5,000★以上）

### Ice Cubes（6,748★）
**URL**: https://github.com/Dimillian/IceCubesApp
**種別**: Mastodonクライアント

**特徴**:
- マルチアカウント対応
- タイムラインカスタマイズ
- プッシュ通知
- async/await活用

**学習ポイント**:
- 大規模SwiftUIアプリの設計
- ネットワーク層の実装
- 状態管理パターン

---

### MovieSwiftUI（6,524★）
**URL**: https://github.com/Dimillian/MovieSwiftUI
**種別**: 映画ブラウザ

**特徴**:
- TheMovieDB API連携
- Combine活用
- iOS/iPadOS/macOS対応
- Redux風状態管理

**学習ポイント**:
- マルチプラットフォーム設計
- Combineの実践的使用
- 単方向データフロー

---

### Clean Architecture SwiftUI（6,436★）
**URL**: https://github.com/nalexn/clean-architecture-swiftui
**種別**: アーキテクチャサンプル

**特徴**:
- MVVM + Clean Architecture
- 完全なテストカバレッジ
- DI実装例

**学習ポイント**:
- 本格的なアーキテクチャ
- テスト戦略
- 依存性注入

---

### SwiftUI Examples（5,572★）
**URL**: https://github.com/ivanvorobei/SwiftUI
**種別**: コンポーネント集

**学習ポイント**:
- レイアウト例
- アニメーション技法
- ジェスチャー処理

---

## Tier 2: 注目アプリ（2,000-5,000★）

| スター | アプリ | 特徴 |
|--------|--------|------|
| 3,700 | EhPanda | Combine, 画像キャッシュ |
| 3,412 | OldOS | iOS 4再現、創造的UI |
| 3,270 | PeopleInSpace | KMM連携 |
| 2,942 | isowords | TCA、ゲーム開発 |
| 2,499 | SwiftUI-Kit | コンポーネント展示 |
| 2,355 | Swift Charts Examples | グラフ描画 |
| 2,179 | fullmoon | ローカルLLM、MLX |

---

## Tier 3: 新興アプリ（1,000-2,000★）

| スター | アプリ | 焦点 |
|--------|--------|------|
| 1,892 | pISSStream | visionOS + watchOS |
| 1,874 | SwiftUI Experiments | UI実験 |
| 1,832 | Food Truck | Apple公式サンプル |
| 1,724 | AC Helper | どうぶつの森 |
| 1,287 | reddit-swiftui | クロスプラットフォーム |
| 1,276 | Dime | 家計簿、Appleガイドライン準拠 |
| 1,223 | SwiftTerm | ターミナル |
| 1,045 | Find | 画像内テキスト検索 |

---

## アーキテクチャパターン

### 1. Redux風（単方向データフロー）
**採用例**: MovieSwiftUI, isowords

```
ユーザー操作 → Action → Store → Reducer → State → View
```

### 2. MVVM + Combine
**採用例**: Clean Architecture, SwiftHub

```
View ←→ ViewModel ←→ Repository
         ↓
      Combine
```

### 3. The Composable Architecture (TCA)
**採用例**: isowords

```
State + Action + Environment → Reducer → Effect
```

### 4. Environment DI
**採用例**: Ice Cubes

```swift
@Environment(\.apiClient) var apiClient
@EnvironmentObject var appState: AppState
```

---

## よく使われるライブラリ

| ライブラリ | 用途 | 使用例 |
|-----------|------|--------|
| Combine | リアクティブ | MovieSwiftUI |
| Kingfisher | 画像ロード | 複数 |
| Realm | ローカルDB | Find |
| Firebase | バックエンド | MakeItSo |
| SwiftSoup | HTMLパース | NYTimes-iOS |

---

## SwiftUIベストプラクティス

### ビュー構成
- 小さく焦点を絞ったビュー
- サブビューの抽出
- ViewModifierで共通スタイル

### 状態管理
- `@State`: ローカルUI状態
- `@StateObject`: ビュー所有オブジェクト
- `@EnvironmentObject`: アプリ全体状態
- `@ObservedObject`: 注入された依存

### パフォーマンス
- LazyVStack/LazyHStackの使用
- .taskで非同期処理
- .id()の適切な使用

---

## 主要開発者

| 開発者 | 代表作 | GitHub |
|--------|--------|--------|
| Thomas Ricouard | Ice Cubes, MovieSwiftUI | @Dimillian |
| Alexey Naumov | Clean Architecture | @nalexn |
| Ivan Vorobei | SwiftUI Examples | @ivanvorobei |
| Jordan Singer | SwiftUI-Kit | @jordansinger |
| Point-Free | isowords | @pointfreeco |
| Paul Hudson | Unwrap | @twostraws |
