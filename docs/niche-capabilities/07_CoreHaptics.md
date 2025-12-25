# Core Haptics触覚フィードバック

**驚きポイント**: 振動パターンを細かく設計してUX向上

## 参考プロジェクト

| プロジェクト | Stars | 特徴 |
|-------------|-------|------|
| [Haptic Haven](https://github.com/davejacobsen/HapticHaven) | 43 | 全パターン実装例 |

---

## 1. 基本的なHaptic

### UIFeedbackGenerator（シンプル）

```swift
import UIKit

class SimpleHaptics {
    // 衝撃フィードバック
    func impact(_ style: UIImpactFeedbackGenerator.FeedbackStyle) {
        let generator = UIImpactFeedbackGenerator(style: style)
        generator.prepare()
        generator.impactOccurred()
    }

    // 通知フィードバック
    func notification(_ type: UINotificationFeedbackGenerator.FeedbackType) {
        let generator = UINotificationFeedbackGenerator()
        generator.prepare()
        generator.notificationOccurred(type)
    }

    // 選択フィードバック
    func selection() {
        let generator = UISelectionFeedbackGenerator()
        generator.prepare()
        generator.selectionChanged()
    }
}

// 使用例
let haptics = SimpleHaptics()
haptics.impact(.heavy)           // 重い衝撃
haptics.notification(.success)   // 成功
haptics.selection()              // 選択変更
```

---

## 2. Core Haptics（高度）

### エンジン初期化

```swift
import CoreHaptics

class AdvancedHaptics {
    private var engine: CHHapticEngine?

    init() {
        guard CHHapticEngine.capabilitiesForHardware().supportsHaptics else { return }

        do {
            engine = try CHHapticEngine()
            try engine?.start()

            // エンジン停止時の再開
            engine?.stoppedHandler = { [weak self] reason in
                try? self?.engine?.start()
            }

            engine?.resetHandler = { [weak self] in
                try? self?.engine?.start()
            }
        } catch {
            print("Haptic engine error: \(error)")
        }
    }
}
```

### カスタムパターン

```swift
extension AdvancedHaptics {
    // 単発パルス
    func pulse(intensity: Float, sharpness: Float) {
        guard let engine = engine else { return }

        let event = CHHapticEvent(
            eventType: .hapticTransient,
            parameters: [
                CHHapticEventParameter(parameterID: .hapticIntensity, value: intensity),
                CHHapticEventParameter(parameterID: .hapticSharpness, value: sharpness)
            ],
            relativeTime: 0
        )

        do {
            let pattern = try CHHapticPattern(events: [event], parameters: [])
            let player = try engine.makePlayer(with: pattern)
            try player.start(atTime: CHHapticTimeImmediate)
        } catch {
            print("Haptic error: \(error)")
        }
    }

    // 連続振動
    func continuous(duration: TimeInterval, intensity: Float, sharpness: Float) {
        guard let engine = engine else { return }

        let event = CHHapticEvent(
            eventType: .hapticContinuous,
            parameters: [
                CHHapticEventParameter(parameterID: .hapticIntensity, value: intensity),
                CHHapticEventParameter(parameterID: .hapticSharpness, value: sharpness)
            ],
            relativeTime: 0,
            duration: duration
        )

        do {
            let pattern = try CHHapticPattern(events: [event], parameters: [])
            let player = try engine.makePlayer(with: pattern)
            try player.start(atTime: CHHapticTimeImmediate)
        } catch {
            print("Haptic error: \(error)")
        }
    }

    // 心臓の鼓動
    func heartbeat() {
        guard let engine = engine else { return }

        let events: [CHHapticEvent] = [
            // 最初のビート（強）
            CHHapticEvent(
                eventType: .hapticTransient,
                parameters: [
                    CHHapticEventParameter(parameterID: .hapticIntensity, value: 1.0),
                    CHHapticEventParameter(parameterID: .hapticSharpness, value: 0.5)
                ],
                relativeTime: 0
            ),
            // 2つ目のビート（弱）
            CHHapticEvent(
                eventType: .hapticTransient,
                parameters: [
                    CHHapticEventParameter(parameterID: .hapticIntensity, value: 0.5),
                    CHHapticEventParameter(parameterID: .hapticSharpness, value: 0.3)
                ],
                relativeTime: 0.15
            )
        ]

        do {
            let pattern = try CHHapticPattern(events: events, parameters: [])
            let player = try engine.makePlayer(with: pattern)
            try player.start(atTime: CHHapticTimeImmediate)
        } catch {
            print("Haptic error: \(error)")
        }
    }
}
```

---

## 3. 動的パラメータ制御

```swift
extension AdvancedHaptics {
    private var continuousPlayer: CHHapticAdvancedPatternPlayer?

    func startDynamicHaptic() throws {
        guard let engine = engine else { return }

        let event = CHHapticEvent(
            eventType: .hapticContinuous,
            parameters: [
                CHHapticEventParameter(parameterID: .hapticIntensity, value: 0.5),
                CHHapticEventParameter(parameterID: .hapticSharpness, value: 0.5)
            ],
            relativeTime: 0,
            duration: 100  // 長時間
        )

        let pattern = try CHHapticPattern(events: [event], parameters: [])
        continuousPlayer = try engine.makeAdvancedPlayer(with: pattern)
        try continuousPlayer?.start(atTime: CHHapticTimeImmediate)
    }

    // リアルタイムで強度変更
    func updateIntensity(_ value: Float) throws {
        let parameter = CHHapticDynamicParameter(
            parameterID: .hapticIntensityControl,
            value: value,
            relativeTime: 0
        )
        try continuousPlayer?.sendParameters([parameter], atTime: CHHapticTimeImmediate)
    }

    func stopDynamicHaptic() throws {
        try continuousPlayer?.stop(atTime: CHHapticTimeImmediate)
    }
}
```

---

## 4. AHAPファイル（パターン定義）

```json
{
  "Version": 1.0,
  "Metadata": {
    "Project": "MyApp",
    "Created": "2024-01-01"
  },
  "Pattern": [
    {
      "Event": {
        "Time": 0.0,
        "EventType": "HapticTransient",
        "EventParameters": [
          { "ParameterID": "HapticIntensity", "ParameterValue": 1.0 },
          { "ParameterID": "HapticSharpness", "ParameterValue": 0.5 }
        ]
      }
    },
    {
      "Event": {
        "Time": 0.1,
        "EventType": "HapticContinuous",
        "EventDuration": 0.3,
        "EventParameters": [
          { "ParameterID": "HapticIntensity", "ParameterValue": 0.8 },
          { "ParameterID": "HapticSharpness", "ParameterValue": 0.2 }
        ]
      }
    }
  ]
}
```

### AHAPファイル読み込み

```swift
extension AdvancedHaptics {
    func playFromAHAP(named name: String) {
        guard let engine = engine,
              let url = Bundle.main.url(forResource: name, withExtension: "ahap") else {
            return
        }

        do {
            try engine.playPattern(from: url)
        } catch {
            print("AHAP error: \(error)")
        }
    }
}
```

---

## 5. SwiftUI統合

```swift
import SwiftUI

struct HapticButton: View {
    let haptics = AdvancedHaptics()

    var body: some View {
        Button("Tap Me") {
            haptics.pulse(intensity: 1.0, sharpness: 0.5)
        }
        .sensoryFeedback(.impact, trigger: tapCount)  // iOS 17+
    }

    @State private var tapCount = 0
}

// iOS 17+ sensoryFeedback
struct ModernHapticView: View {
    @State private var isSuccess = false

    var body: some View {
        Button("Complete") {
            isSuccess = true
        }
        .sensoryFeedback(.success, trigger: isSuccess)
    }
}
```

---

## まとめ

| 方式 | 用途 | カスタマイズ |
|------|------|-------------|
| UIFeedbackGenerator | シンプルなフィードバック | 低 |
| Core Haptics | カスタムパターン | 高 |
| AHAP | パターン外部定義 | 高 |
| sensoryFeedback | SwiftUI統合 | 中 |

### パラメータ

| パラメータ | 範囲 | 効果 |
|-----------|------|------|
| Intensity | 0-1 | 振動の強さ |
| Sharpness | 0-1 | 鋭さ（0=鈍い、1=鋭い） |
