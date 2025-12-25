# Amethyst

**スター**: 5,732 ⭐
**リポジトリ**: https://github.com/ianyh/Amethyst
**ライセンス**: MIT

---

## 概要

Amethystは、macOS向けのタイリングウィンドウマネージャー。xmonadにインスパイアされ、キーボードショートカットでウィンドウを自動配置。プログラマー向けの生産性ツール。

---

## 主要機能

| 機能 | 説明 |
|------|------|
| タイリングレイアウト | 自動ウィンドウ配置 |
| 複数レイアウト | tall, wide, fullscreen等 |
| マルチモニター | 複数ディスプレイ対応 |
| スペース対応 | macOSスペース連携 |
| カスタマイズ | 設定ファイル |
| ホットキー | キーボード操作 |

---

## 技術スタック

| 項目 | 技術 |
|------|------|
| 言語 | Swift |
| UI | AppKit |
| ウィンドウ管理 | Accessibility API |
| 設定 | YAML |
| ホットキー | Carbon Events |
| IPC | Distributed Notifications |

---

## アーキテクチャ

```
Amethyst/
├── Amethyst/                # メインアプリ
│   ├── Layout/              # レイアウトアルゴリズム
│   │   ├── TallLayout.swift
│   │   ├── WideLayout.swift
│   │   └── FullscreenLayout.swift
│   ├── Managers/            # マネージャー
│   │   ├── WindowManager.swift
│   │   └── ScreenManager.swift
│   ├── Model/               # データモデル
│   └── Preferences/         # 設定
└── AmethystTests/           # テスト
```

---

## 学習価値

### macOS システム統合

| 分野 | 学習ポイント |
|------|-------------|
| **Accessibility API** | ウィンドウ制御 |
| **ホットキー** | グローバルショートカット |
| **マルチモニター** | 画面管理 |
| **システム統合** | macOS深層API |
| **設定管理** | YAML設定ファイル |

### コードパターン

**Accessibility API ウィンドウ操作**:
```swift
// ウィンドウ制御
class AccessibilityElement {
    private let element: AXUIElement

    init?(window: AXUIElement) {
        self.element = window
    }

    var frame: CGRect? {
        get {
            var value: CFTypeRef?
            let result = AXUIElementCopyAttributeValue(
                element,
                kAXPositionAttribute as CFString,
                &value
            )
            guard result == .success else { return nil }

            var position = CGPoint.zero
            AXValueGetValue(value as! AXValue, .cgPoint, &position)

            // サイズも取得
            AXUIElementCopyAttributeValue(
                element,
                kAXSizeAttribute as CFString,
                &value
            )
            var size = CGSize.zero
            AXValueGetValue(value as! AXValue, .cgSize, &size)

            return CGRect(origin: position, size: size)
        }
        set {
            guard let frame = newValue else { return }

            var position = frame.origin
            let positionValue = AXValueCreate(.cgPoint, &position)!
            AXUIElementSetAttributeValue(
                element,
                kAXPositionAttribute as CFString,
                positionValue
            )

            var size = frame.size
            let sizeValue = AXValueCreate(.cgSize, &size)!
            AXUIElementSetAttributeValue(
                element,
                kAXSizeAttribute as CFString,
                sizeValue
            )
        }
    }
}
```

**レイアウトアルゴリズム**:
```swift
// Tallレイアウト（左にメイン、右にスタック）
class TallLayout: Layout {
    var mainPaneRatio: CGFloat = 0.5
    var mainPaneCount: Int = 1

    func frameAssignments(
        _ windows: [SIWindow],
        on screen: NSScreen
    ) -> [FrameAssignment] {
        let screenFrame = screen.adjustedFrame()
        let mainCount = min(mainPaneCount, windows.count)
        let secondaryCount = windows.count - mainCount

        var assignments: [FrameAssignment] = []

        // メインペイン
        let mainWidth = secondaryCount > 0
            ? screenFrame.width * mainPaneRatio
            : screenFrame.width

        for (index, window) in windows.prefix(mainCount).enumerated() {
            let height = screenFrame.height / CGFloat(mainCount)
            let frame = CGRect(
                x: screenFrame.minX,
                y: screenFrame.minY + height * CGFloat(index),
                width: mainWidth,
                height: height
            )
            assignments.append(FrameAssignment(frame: frame, window: window))
        }

        // セカンダリペイン
        if secondaryCount > 0 {
            let secondaryWidth = screenFrame.width - mainWidth
            for (index, window) in windows.dropFirst(mainCount).enumerated() {
                let height = screenFrame.height / CGFloat(secondaryCount)
                let frame = CGRect(
                    x: screenFrame.minX + mainWidth,
                    y: screenFrame.minY + height * CGFloat(index),
                    width: secondaryWidth,
                    height: height
                )
                assignments.append(FrameAssignment(frame: frame, window: window))
            }
        }

        return assignments
    }
}
```

**グローバルホットキー**:
```swift
// ホットキー登録
class HotKeyManager {
    private var hotKeys: [UInt32: HotKey] = [:]

    func register(
        keyCode: UInt32,
        modifiers: UInt32,
        handler: @escaping () -> Void
    ) {
        var hotKeyRef: EventHotKeyRef?
        let hotKeyID = EventHotKeyID(
            signature: OSType("AMTH".fourCharCode),
            id: keyCode
        )

        let status = RegisterEventHotKey(
            keyCode,
            modifiers,
            hotKeyID,
            GetEventDispatcherTarget(),
            0,
            &hotKeyRef
        )

        if status == noErr {
            hotKeys[keyCode] = HotKey(ref: hotKeyRef!, handler: handler)
        }
    }

    func handleHotKey(id: UInt32) {
        hotKeys[id]?.handler()
    }
}
```

---

## プロジェクト評価

| 観点 | 評価 | 理由 |
|------|------|------|
| **コード品質** | ★★★★★ | クリーンなSwift |
| **ドキュメント** | ★★★★☆ | ユーザーガイド充実 |
| **アクティビティ** | ★★★★☆ | 継続的なメンテナンス |
| **学習価値** | ★★★★★ | macOS深層API学習 |
| **実用性** | ★★★★★ | 生産性向上ツール |

---

## レイアウト種類

| レイアウト | 説明 |
|-----------|------|
| tall | 左メイン+右スタック |
| wide | 上メイン+下スタック |
| fullscreen | 全画面 |
| column | 均等列分割 |
| row | 均等行分割 |
| floating | フローティング |
| bsp | Binary Space Partition |

---

## 設定

### YAML設定ファイル

```yaml
# ~/.amethyst.yml
layouts:
  - tall
  - wide
  - fullscreen

mod1:
  - option
  - shift

mod2:
  - option
  - shift
  - control

float:
  - com.apple.systempreferences
  - com.apple.finder

screen-padding:
  top: 0
  bottom: 0
  left: 0
  right: 0

window-margins: true
window-margin-size: 5
```

---

## 開発者向け情報

### ビルド手順

```bash
git clone https://github.com/ianyh/Amethyst
cd Amethyst
xcodebuild -scheme Amethyst
```

### Accessibility権限

アプリ実行にはシステム環境設定でAccessibility権限が必要。

---

## まとめ

Amethystは、macOSのシステムレベルAPIを学ぶ最良の教材。Accessibility API、グローバルホットキー、マルチモニター管理など、深いシステム統合を網羅。タイリングウィンドウマネージャーのアルゴリズムも参考になる。

**推奨対象**:
- macOSシステムAPIを学びたい開発者
- 生産性ツールを開発したい人
- Accessibility APIを理解したい人
- ウィンドウ管理アルゴリズムに興味がある人
