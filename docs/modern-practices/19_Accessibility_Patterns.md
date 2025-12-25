# アクセシビリティ実装パターン

**参考プロジェクト**:
- Apple公式ガイドライン
- 各種オープンソースアプリ

**実装参考度**: ⭐⭐⭐⭐⭐

---

## なぜ重要か

アクセシビリティは**すべてのユーザー**に使いやすいアプリを提供するため必須。App Store審査でも重視される。

| 学習ポイント | 重要度 | 理由 |
|-------------|--------|------|
| **VoiceOver** | ⭐⭐⭐⭐⭐ | 視覚障害者対応 |
| **Dynamic Type** | ⭐⭐⭐⭐⭐ | 文字サイズ対応 |
| **Color Contrast** | ⭐⭐⭐⭐☆ | 視認性 |
| **Reduce Motion** | ⭐⭐⭐⭐☆ | 動作酔い対策 |
| **Voice Control** | ⭐⭐⭐☆☆ | 音声操作 |

---

## 1. VoiceOver対応

### 基本的なラベル付け

```swift
struct ProductCard: View {
    let product: Product

    var body: some View {
        VStack {
            Image(product.imageName)
                .resizable()
                .frame(width: 100, height: 100)
                .accessibilityLabel("商品画像")
                .accessibilityHidden(true)  // 画像は隠して、全体で読み上げ

            Text(product.name)
            Text(product.price.formatted(.currency(code: "JPY")))
        }
        .accessibilityElement(children: .combine)
        .accessibilityLabel("\(product.name)、価格 \(product.price.formatted(.currency(code: "JPY")))")
    }
}
```

### カスタムアクション

```swift
struct TodoItem: View {
    @Binding var todo: Todo
    let onDelete: () -> Void

    var body: some View {
        HStack {
            Button {
                todo.isCompleted.toggle()
            } label: {
                Image(systemName: todo.isCompleted ? "checkmark.circle.fill" : "circle")
            }

            Text(todo.title)
        }
        .accessibilityElement(children: .combine)
        .accessibilityLabel(todo.title)
        .accessibilityValue(todo.isCompleted ? "完了" : "未完了")
        .accessibilityHint("ダブルタップで完了状態を切り替え")
        .accessibilityAction(named: "削除") {
            onDelete()
        }
        .accessibilityAction(named: todo.isCompleted ? "未完了にする" : "完了にする") {
            todo.isCompleted.toggle()
        }
    }
}
```

### ローター対応

```swift
struct ArticleView: View {
    let article: Article

    var body: some View {
        ScrollView {
            VStack(alignment: .leading, spacing: 16) {
                Text(article.title)
                    .font(.title)
                    .accessibilityAddTraits(.isHeader)

                Text(article.subtitle)
                    .font(.headline)
                    .accessibilityAddTraits(.isHeader)

                ForEach(article.sections) { section in
                    Text(section.heading)
                        .font(.title2)
                        .accessibilityAddTraits(.isHeader)

                    Text(section.content)
                }
            }
        }
        .accessibilityRotor("見出し") {
            AccessibilityRotorEntry(article.title, id: article.id)
            AccessibilityRotorEntry(article.subtitle, id: "subtitle")
            ForEach(article.sections) { section in
                AccessibilityRotorEntry(section.heading, id: section.id)
            }
        }
    }
}
```

---

## 2. Dynamic Type

### 基本対応

```swift
struct ScalableText: View {
    var body: some View {
        VStack(alignment: .leading, spacing: 8) {
            // システムフォントは自動対応
            Text("タイトル")
                .font(.headline)

            Text("本文テキスト")
                .font(.body)

            // カスタムフォントも対応可能
            Text("カスタム")
                .font(.custom("Helvetica", size: 16, relativeTo: .body))
        }
    }
}
```

### 最大サイズ制限

```swift
struct LimitedScaleText: View {
    @Environment(\.dynamicTypeSize) var dynamicTypeSize

    var body: some View {
        Text("制限付きテキスト")
            .font(.body)
            .dynamicTypeSize(...DynamicTypeSize.accessibility2)
    }
}
```

### レイアウト調整

```swift
struct AdaptiveLayout: View {
    @Environment(\.dynamicTypeSize) var dynamicTypeSize

    var body: some View {
        if dynamicTypeSize >= .accessibility1 {
            // アクセシビリティサイズでは縦積み
            VStack(alignment: .leading) {
                icon
                textContent
            }
        } else {
            // 通常サイズでは横並び
            HStack {
                icon
                textContent
            }
        }
    }

    private var icon: some View {
        Image(systemName: "star.fill")
            .font(.title)
    }

    private var textContent: some View {
        VStack(alignment: .leading) {
            Text("タイトル").font(.headline)
            Text("説明文").font(.subheadline)
        }
    }
}
```

---

## 3. カラーコントラスト

### 高コントラスト対応

```swift
struct HighContrastView: View {
    @Environment(\.colorSchemeContrast) var contrast

    var body: some View {
        Text("テキスト")
            .foregroundColor(contrast == .increased ? .primary : .secondary)
            .background(contrast == .increased ? Color.white : Color.gray.opacity(0.1))
    }
}
```

### セマンティックカラー

```swift
struct SemanticColorView: View {
    var body: some View {
        VStack {
            // システムカラーを使用（自動で高コントラスト対応）
            Text("プライマリ")
                .foregroundStyle(.primary)

            Text("セカンダリ")
                .foregroundStyle(.secondary)

            Button("アクション") {}
                .foregroundStyle(.tint)

            Text("エラー")
                .foregroundStyle(.red)  // システムレッドは最適化済み
        }
    }
}
```

---

## 4. Reduce Motion

### アニメーション無効化

```swift
struct MotionSensitiveView: View {
    @Environment(\.accessibilityReduceMotion) var reduceMotion
    @State private var isAnimating = false

    var body: some View {
        Circle()
            .scaleEffect(isAnimating ? 1.2 : 1.0)
            .animation(reduceMotion ? nil : .spring(), value: isAnimating)
            .onAppear {
                if !reduceMotion {
                    isAnimating = true
                }
            }
    }
}
```

### トランジション置換

```swift
struct SafeTransitionView: View {
    @Environment(\.accessibilityReduceMotion) var reduceMotion
    @State private var isVisible = false

    var body: some View {
        VStack {
            Button("Toggle") {
                withAnimation {
                    isVisible.toggle()
                }
            }

            if isVisible {
                Text("コンテンツ")
                    .transition(reduceMotion ? .opacity : .slide)
            }
        }
    }
}
```

---

## 5. その他のアクセシビリティ

### Reduce Transparency

```swift
struct TransparencyAwareView: View {
    @Environment(\.accessibilityReduceTransparency) var reduceTransparency

    var body: some View {
        Rectangle()
            .fill(reduceTransparency
                ? Color.gray
                : Color.gray.opacity(0.5))
    }
}
```

### Invert Colors

```swift
struct InvertAwareImage: View {
    @Environment(\.accessibilityInvertColors) var invertColors

    var body: some View {
        Image("photo")
            .accessibilityIgnoresInvertColors(true)  // 写真は反転しない
    }
}
```

### Bold Text

```swift
struct BoldTextAwareView: View {
    @Environment(\.legibilityWeight) var legibilityWeight

    var body: some View {
        Text("テキスト")
            .fontWeight(legibilityWeight == .bold ? .bold : .regular)
    }
}
```

---

## 6. テスト

### Accessibility Inspector

```swift
// Xcode → Open Developer Tool → Accessibility Inspector

// SwiftUIプレビューでのテスト
#Preview {
    ContentView()
        .environment(\.dynamicTypeSize, .accessibility5)
}

#Preview {
    ContentView()
        .environment(\.accessibilityReduceMotion, true)
}
```

### Unit Test

```swift
import XCTest

final class AccessibilityTests: XCTestCase {
    func testVoiceOverLabels() {
        let app = XCUIApplication()
        app.launch()

        // アクセシビリティラベルでの要素検索
        let button = app.buttons["送信ボタン"]
        XCTAssertTrue(button.exists)

        // アクセシビリティ値のテスト
        let slider = app.sliders["音量"]
        XCTAssertEqual(slider.value as? String, "50%")
    }
}
```

---

## まとめ：アクセシビリティ対応

| 機能 | 対応方法 | 優先度 |
|------|---------|--------|
| **VoiceOver** | accessibilityLabel, accessibilityValue | 🔴 必須 |
| **Dynamic Type** | システムフォント使用 | 🔴 必須 |
| **Color Contrast** | セマンティックカラー | 🔴 必須 |
| **Reduce Motion** | animation条件分岐 | 🟡 推奨 |
| **カスタムアクション** | accessibilityAction | 🟡 推奨 |
| **ローター** | accessibilityRotor | 🟢 参考 |

### チェックリスト

1. すべてのインタラクティブ要素にラベルがあるか
2. Dynamic Typeで崩れないか
3. 高コントラストモードで視認できるか
4. Reduce Motionでアニメーションが止まるか
5. VoiceOverでナビゲートできるか
