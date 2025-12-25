# UI/UXパターンとベストプラクティス

**調査日**: 2025年12月25日

---

## フレームワーク分布

| フレームワーク | 使用数 | トレンド |
|---------------|--------|----------|
| SwiftUI | 207 | 急成長 |
| UIKit | 600+ | 依然主流 |
| 混合 | 100+ | 移行期 |

---

## デザインシステム

### Appleガイドライン準拠型

**Dime**（1,276★）- 家計簿
- HIG完全準拠
- システムカラー使用
- SF Symbols統合
- ダークモード対応

**Clendar**（692★）- カレンダー
- ミニマルデザイン
- ウィジェット重視
- システムカレンダー連携

### カスタムデザインシステム型

**Ice Cubes**（6,748★）- Mastodon
- カスタムテーマエンジン
- アクセントカラー変更可能
- タイポグラフィバリエーション

**MovieSwiftUI**（6,524★）
- ポスター中心デザイン
- アニメーション豊富
- パララックス効果

---

## ナビゲーションパターン

### タブベース
ソーシャル・コンテンツアプリに最適

```
┌─────────────────────────────┐
│          コンテンツ          │
├────┬────┬────┬────┬────┤
│ホーム│検索│投稿│通知│設定│
└────┴────┴────┴────┴────┘
```

**採用例**: Ice Cubes, SNSクローン

### サイドバー
iPadOS/macOS最適化

```
┌─────┬───────────────────┐
│ Nav │                   │
│     │     コンテンツ     │
│List │                   │
└─────┴───────────────────┘
```

**採用例**: NetNewsWire, Element

### モーダル重視
タスク・生産性アプリ

**採用例**: Todoアプリ、メモアプリ

---

## リスト・コレクション

### 無限スクロール

```swift
List {
    ForEach(items) { item in
        ItemRow(item: item)
            .onAppear {
                if item == items.last {
                    loadMore()
                }
            }
    }
}
```

**ベストプラクティス**:
- 事前フェッチ
- 下部ローディング表示
- エラー状態対応
- プルリフレッシュ

### グリッドレイアウト

**写真アプリ**（PixPic, Find）
- 適応的カラム数
- ピンチズーム
- 選択モード

**コンテンツアプリ**（MovieSwiftUI）
- ポスターグリッド
- 横スクロールセクション

---

## アニメーション

### マイクロインタラクション

**Purposeful Animations**（869★）より

- ボタン押下フィードバック
- ローディング遷移
- 成功/エラー表示
- プルリフレッシュ

### 画面遷移

```swift
// Hero Animation
NavigationLink {
    DetailView(item: item)
} label: {
    ItemCard(item: item)
        .matchedGeometryEffect(id: item.id, in: namespace)
}
```

### スケルトンローディング

- プレースホルダー形状
- シマーエフェクト
- 段階的ロード
- スムーズな遷移

---

## フォーム・入力

### リアルタイムバリデーション

```swift
TextField("メール", text: $email)
    .textContentType(.emailAddress)
    .overlay(alignment: .trailing) {
        if !email.isEmpty {
            Image(systemName: isValid ? "checkmark.circle" : "xmark.circle")
                .foregroundColor(isValid ? .green : .red)
        }
    }
```

### マルチステップフォーム
- 進捗インジケーター
- 戻る/次へナビゲーション
- スキップオプション
- ステップごとのバリデーション

### 検索インターフェース
- キャンセル付き検索バー
- 最近の検索履歴
- サジェスト/オートコンプリート
- スコープフィルタリング

---

## ダークモード実装

### システムカラー使用

```swift
Color(.systemBackground)
Color(.label)
Color(.secondaryLabel)
```

### カスタムカラーアセット

```swift
// Color.xcassetsでAny/Dark設定
Color("CardBackground")
Color("AccentPrimary")
```

### 動的対応
- 画像のダークモード版
- シャドウ調整
- ブラー効果適応

---

## アクセシビリティ

### VoiceOver対応

```swift
Image("profile")
    .accessibilityLabel("プロフィール写真")
    .accessibilityHint("ダブルタップでプロフィールを表示")
```

**優秀例**: Ice Cubes, Clendar, Find

### Dynamic Type

```swift
Text("コンテンツ")
    .font(.body)  // Dynamic Type対応
    .lineLimit(nil)
```

### モーション軽減

```swift
@Environment(\.accessibilityReduceMotion) var reduceMotion

.animation(reduceMotion ? nil : .spring(), value: isExpanded)
```

---

## ウィジェットデザイン

### 情報密度

**Clendar**（692★）
- カレンダー一覧
- イベントプレビュー
- 次のイベントカウントダウン
- 複数サイズ対応

### ウィジェットファミリー

| サイズ | 最適な用途 |
|--------|-----------|
| Small | 単一データ、ステータス |
| Medium | リストプレビュー、クイックアクション |
| Large | 詳細情報、複数項目 |
| Extra Large | ダッシュボード表示 |

---

## プラットフォーム別パターン

### iPadOS対応

**採用例**: MovieSwiftUI, Ice Cubes

- サイドバーナビゲーション
- マルチカラムレイアウト
- ポインターホバー効果
- キーボードショートカット
- Split View対応

### watchOS

**TermiWatch**（2,213★）
- 最小限の情報表示
- グランス可能なデータ
- コンプリケーション対応
- Digital Crown連携

### visionOS

- 空間レイアウト
- ウィンドウ管理
- イマーシブ空間
- ハンドトラッキングUI

---

## よくあるUI問題

| 問題 | 原因 | 解決策 |
|------|------|--------|
| パフォーマンス低下 | 重い画像、同期ロード | キャッシュ、非同期化 |
| アクセシビリティ不足 | VoiceOverラベル欠如 | 全要素にラベル付与 |
| 状態管理の不整合 | ローディング状態未定義 | 全状態を明示的に管理 |
| ナビゲーション混乱 | 深い階層 | フラット化、タブ使用 |

---

## UIコンポーネントのパッケージ化候補

### リストコンポーネント
- スワイプアクション（取り消し付き）
- ドラッグ並べ替え
- バッチ選択モード
- 展開可能セクション

### ローディング状態
- スケルトンビュー
- プログレスインジケーター
- リトライ機構
- オフラインプレースホルダー

### メディアコンポーネント
- 画像ギャラリー（ズーム付き）
- 動画プレイヤーコントロール
- オーディオ波形表示
- ドキュメントプレビュー

### 入力コンポーネント
- PIN/OTPエントリー
- リッチテキストエディタ
- 日付範囲ピッカー
- マルチセレクトチップ

### フィードバック
- トースト通知
- スナックバー
- アプリ内評価プロンプト
- お祝いアニメーション

---

## デザインインスピレーション

| アプリ | スター | デザイン強み |
|--------|--------|-------------|
| Ice Cubes | 6,748 | SNS UX |
| Dime | 1,276 | 金融アプリの明瞭さ |
| Clendar | 692 | カレンダーのシンプルさ |
| OldOS | 3,412 | スキューモーフィズム |
| SwiftUI-Kit | 2,499 | コンポーネント展示 |
| buttoncraft | 445 | ボタンスタイリング |
