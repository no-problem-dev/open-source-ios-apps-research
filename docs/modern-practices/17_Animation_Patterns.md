# SwiftUI アニメーションパターン集

**参考プロジェクト**:
- Ice Cubes (スムーズなUI)
- 各種オープンソースアプリ

**実装参考度**: ⭐⭐⭐⭐⭐

---

## なぜ重要か

アニメーションは**UXの質を決定づける**要素。SwiftUIの宣言的アニメーションを活用することで、少ないコードで滑らかなUIを実現。

| 学習ポイント | 重要度 | 理由 |
|-------------|--------|------|
| **withAnimation** | ⭐⭐⭐⭐⭐ | 明示的アニメーション |
| **animation modifier** | ⭐⭐⭐⭐⭐ | 暗黙的アニメーション |
| **Transition** | ⭐⭐⭐⭐☆ | 表示/非表示アニメ |
| **matchedGeometryEffect** | ⭐⭐⭐⭐☆ | 共有要素トランジション |
| **PhaseAnimator** | ⭐⭐⭐⭐☆ | iOS 17の新機能 |

---

## 1. 基本アニメーション

### withAnimation（明示的）

```swift
struct CounterView: View {
    @State private var count = 0

    var body: some View {
        VStack {
            Text("\(count)")
                .font(.largeTitle)
                .scaleEffect(count > 0 ? 1.2 : 1.0)

            Button("増加") {
                withAnimation(.spring(response: 0.3, dampingFraction: 0.6)) {
                    count += 1
                }
            }
        }
    }
}
```

### animation modifier（暗黙的）

```swift
struct LoadingView: View {
    @State private var isLoading = false

    var body: some View {
        Circle()
            .trim(from: 0, to: 0.7)
            .stroke(Color.blue, lineWidth: 4)
            .frame(width: 50, height: 50)
            .rotationEffect(Angle(degrees: isLoading ? 360 : 0))
            .animation(.linear(duration: 1).repeatForever(autoreverses: false), value: isLoading)
            .onAppear {
                isLoading = true
            }
    }
}
```

---

## 2. アニメーションの種類

### 標準アニメーション

```swift
// デフォルト
.animation(.default, value: state)

// イージング
.animation(.easeIn(duration: 0.3), value: state)
.animation(.easeOut(duration: 0.3), value: state)
.animation(.easeInOut(duration: 0.3), value: state)
.animation(.linear(duration: 0.3), value: state)

// スプリング
.animation(.spring(), value: state)
.animation(.spring(response: 0.5, dampingFraction: 0.6, blendDuration: 0), value: state)

// バウンス（iOS 17+）
.animation(.bouncy, value: state)
.animation(.bouncy(duration: 0.5, extraBounce: 0.2), value: state)

// スムース（iOS 17+）
.animation(.smooth, value: state)
.animation(.smooth(duration: 0.5), value: state)

// スナッピー（iOS 17+）
.animation(.snappy, value: state)
```

### 繰り返し

```swift
// 永続的に繰り返し
.animation(.linear(duration: 2).repeatForever(autoreverses: false), value: state)

// 往復で繰り返し
.animation(.easeInOut(duration: 1).repeatForever(autoreverses: true), value: state)

// 指定回数繰り返し
.animation(.easeInOut.repeatCount(3, autoreverses: true), value: state)
```

---

## 3. Transition

### 基本トランジション

```swift
struct ToastView: View {
    @State private var showToast = false

    var body: some View {
        VStack {
            Button("Show Toast") {
                withAnimation {
                    showToast = true
                }
                DispatchQueue.main.asyncAfter(deadline: .now() + 2) {
                    withAnimation {
                        showToast = false
                    }
                }
            }

            Spacer()

            if showToast {
                Text("Toast Message")
                    .padding()
                    .background(Color.black.opacity(0.8))
                    .foregroundColor(.white)
                    .cornerRadius(10)
                    .transition(.move(edge: .bottom).combined(with: .opacity))
            }
        }
    }
}
```

### カスタムトランジション

```swift
extension AnyTransition {
    static var slideAndFade: AnyTransition {
        .asymmetric(
            insertion: .move(edge: .trailing).combined(with: .opacity),
            removal: .move(edge: .leading).combined(with: .opacity)
        )
    }

    static var scaleAndRotate: AnyTransition {
        .modifier(
            active: ScaleRotateModifier(scale: 0, rotation: 90),
            identity: ScaleRotateModifier(scale: 1, rotation: 0)
        )
    }
}

struct ScaleRotateModifier: ViewModifier {
    let scale: CGFloat
    let rotation: Double

    func body(content: Content) -> some View {
        content
            .scaleEffect(scale)
            .rotationEffect(.degrees(rotation))
    }
}

// 使用
if isVisible {
    CardView()
        .transition(.scaleAndRotate)
}
```

---

## 4. matchedGeometryEffect

### 共有要素トランジション

```swift
struct HeroAnimationView: View {
    @State private var isExpanded = false
    @Namespace private var animation

    var body: some View {
        VStack {
            if !isExpanded {
                // サムネイル状態
                HStack {
                    Image("photo")
                        .resizable()
                        .matchedGeometryEffect(id: "image", in: animation)
                        .frame(width: 80, height: 80)
                        .clipShape(RoundedRectangle(cornerRadius: 8))

                    VStack(alignment: .leading) {
                        Text("Title")
                            .matchedGeometryEffect(id: "title", in: animation)
                    }
                }
                .onTapGesture {
                    withAnimation(.spring(response: 0.4, dampingFraction: 0.8)) {
                        isExpanded = true
                    }
                }
            } else {
                // 展開状態
                VStack {
                    Image("photo")
                        .resizable()
                        .matchedGeometryEffect(id: "image", in: animation)
                        .frame(height: 300)
                        .clipShape(RoundedRectangle(cornerRadius: 16))

                    Text("Title")
                        .font(.largeTitle)
                        .matchedGeometryEffect(id: "title", in: animation)

                    Text("詳細コンテンツ...")
                }
                .onTapGesture {
                    withAnimation(.spring(response: 0.4, dampingFraction: 0.8)) {
                        isExpanded = false
                    }
                }
            }
        }
    }
}
```

---

## 5. PhaseAnimator（iOS 17+）

### 自動フェーズアニメーション

```swift
struct PulsingView: View {
    var body: some View {
        PhaseAnimator([false, true]) { phase in
            Circle()
                .fill(.blue)
                .frame(width: 100, height: 100)
                .scaleEffect(phase ? 1.2 : 1.0)
                .opacity(phase ? 0.7 : 1.0)
        } animation: { phase in
            .easeInOut(duration: 0.8)
        }
    }
}
```

### 複数フェーズ

```swift
enum AnimationPhase: CaseIterable {
    case initial, up, down, final

    var scale: Double {
        switch self {
        case .initial: 1.0
        case .up: 1.2
        case .down: 0.8
        case .final: 1.0
        }
    }

    var rotation: Double {
        switch self {
        case .initial: 0
        case .up: 10
        case .down: -10
        case .final: 0
        }
    }
}

struct MultiPhaseAnimationView: View {
    @State private var trigger = false

    var body: some View {
        PhaseAnimator(
            AnimationPhase.allCases,
            trigger: trigger
        ) { phase in
            RoundedRectangle(cornerRadius: 20)
                .fill(.blue)
                .frame(width: 100, height: 100)
                .scaleEffect(phase.scale)
                .rotationEffect(.degrees(phase.rotation))
        } animation: { phase in
            switch phase {
            case .initial: .smooth
            case .up: .bouncy
            case .down: .bouncy
            case .final: .smooth
            }
        }
        .onTapGesture {
            trigger.toggle()
        }
    }
}
```

---

## 6. KeyframeAnimator（iOS 17+）

### キーフレームアニメーション

```swift
struct KeyframeAnimationView: View {
    @State private var trigger = false

    var body: some View {
        VStack {
            Image(systemName: "bell.fill")
                .font(.system(size: 50))
                .keyframeAnimator(
                    initialValue: AnimationValues(),
                    trigger: trigger
                ) { content, value in
                    content
                        .rotationEffect(value.rotation)
                        .scaleEffect(value.scale)
                } keyframes: { _ in
                    KeyframeTrack(\.rotation) {
                        LinearKeyframe(.degrees(0), duration: 0.1)
                        SpringKeyframe(.degrees(-20), duration: 0.1)
                        SpringKeyframe(.degrees(20), duration: 0.1)
                        SpringKeyframe(.degrees(-20), duration: 0.1)
                        SpringKeyframe(.degrees(0), duration: 0.2)
                    }
                    KeyframeTrack(\.scale) {
                        LinearKeyframe(1.0, duration: 0.1)
                        SpringKeyframe(1.2, duration: 0.2)
                        SpringKeyframe(1.0, duration: 0.2)
                    }
                }

            Button("Ring") {
                trigger.toggle()
            }
        }
    }
}

struct AnimationValues {
    var rotation: Angle = .degrees(0)
    var scale: Double = 1.0
}
```

---

## 7. インタラクティブアニメーション

### ジェスチャー連動

```swift
struct DraggableCard: View {
    @State private var offset: CGSize = .zero
    @State private var isDragging = false

    var body: some View {
        RoundedRectangle(cornerRadius: 20)
            .fill(.blue)
            .frame(width: 200, height: 300)
            .offset(offset)
            .scaleEffect(isDragging ? 1.05 : 1.0)
            .shadow(radius: isDragging ? 20 : 5)
            .gesture(
                DragGesture()
                    .onChanged { value in
                        withAnimation(.interactiveSpring()) {
                            offset = value.translation
                            isDragging = true
                        }
                    }
                    .onEnded { _ in
                        withAnimation(.spring(response: 0.5, dampingFraction: 0.6)) {
                            offset = .zero
                            isDragging = false
                        }
                    }
            )
    }
}
```

---

## まとめ：アニメーションパターン

| パターン | 用途 | 優先度 |
|---------|------|--------|
| **withAnimation** | 状態変化のアニメーション | 🔴 必須 |
| **Spring** | 自然な動き | 🔴 必須 |
| **Transition** | 表示/非表示 | 🟡 推奨 |
| **matchedGeometryEffect** | Hero Animation | 🟡 推奨 |
| **PhaseAnimator** | 自動アニメーション（iOS 17+） | 🟢 参考 |
| **KeyframeAnimator** | 複雑なアニメーション（iOS 17+） | 🟢 参考 |

### パフォーマンスTips

1. **アニメーション対象を限定**: 必要な要素のみアニメーション
2. **drawingGroup()**: 複雑なViewをフラット化
3. **適切なduration**: 0.2-0.5秒が一般的
4. **reducedMotion対応**: アクセシビリティ考慮
