# Picture in Picture実装

**驚きポイント**: 任意のHTML5動画をPiPで再生

## 参考プロジェクト

| プロジェクト | Stars | 特徴 |
|-------------|-------|------|
| [PiPifier](https://github.com/arnoappenzeller/PiPifier) | 793 | Safari Extension でPiP |

---

## 1. AVPlayerでPiP

### 基本実装

```swift
import AVKit

class PiPVideoPlayer: NSObject, ObservableObject {
    let player = AVPlayer()
    private var pipController: AVPictureInPictureController?

    @Published var isPiPActive = false
    @Published var isPiPPossible = false

    func setupPiP(with playerLayer: AVPlayerLayer) {
        guard AVPictureInPictureController.isPictureInPictureSupported() else { return }

        pipController = AVPictureInPictureController(playerLayer: playerLayer)
        pipController?.delegate = self

        // PiP可能状態の監視
        pipController?.publisher(for: \.isPictureInPicturePossible)
            .receive(on: DispatchQueue.main)
            .assign(to: &$isPiPPossible)
    }

    func togglePiP() {
        if pipController?.isPictureInPictureActive == true {
            pipController?.stopPictureInPicture()
        } else {
            pipController?.startPictureInPicture()
        }
    }

    func play(url: URL) {
        let item = AVPlayerItem(url: url)
        player.replaceCurrentItem(with: item)
        player.play()
    }
}

extension PiPVideoPlayer: AVPictureInPictureControllerDelegate {
    func pictureInPictureControllerDidStartPictureInPicture(
        _ pictureInPictureController: AVPictureInPictureController
    ) {
        isPiPActive = true
    }

    func pictureInPictureControllerDidStopPictureInPicture(
        _ pictureInPictureController: AVPictureInPictureController
    ) {
        isPiPActive = false
    }

    // アプリに戻る時の復元
    func pictureInPictureController(
        _ pictureInPictureController: AVPictureInPictureController,
        restoreUserInterfaceForPictureInPictureStopWithCompletionHandler completionHandler: @escaping (Bool) -> Void
    ) {
        // UIを復元
        completionHandler(true)
    }
}
```

### SwiftUI統合

```swift
import SwiftUI
import AVKit

struct PiPVideoView: UIViewRepresentable {
    let player: AVPlayer
    let onPiPSetup: (AVPlayerLayer) -> Void

    func makeUIView(context: Context) -> UIView {
        let view = PlayerView()
        view.player = player
        onPiPSetup(view.playerLayer)
        return view
    }

    func updateUIView(_ uiView: UIView, context: Context) {}
}

class PlayerView: UIView {
    override class var layerClass: AnyClass { AVPlayerLayer.self }

    var playerLayer: AVPlayerLayer { layer as! AVPlayerLayer }

    var player: AVPlayer? {
        get { playerLayer.player }
        set { playerLayer.player = newValue }
    }
}

// 使用例
struct ContentView: View {
    @StateObject private var videoPlayer = PiPVideoPlayer()

    var body: some View {
        VStack {
            PiPVideoView(player: videoPlayer.player) { layer in
                videoPlayer.setupPiP(with: layer)
            }
            .aspectRatio(16/9, contentMode: .fit)

            HStack {
                Button("Play") {
                    videoPlayer.play(url: URL(string: "https://example.com/video.mp4")!)
                }

                Button(videoPlayer.isPiPActive ? "Exit PiP" : "PiP") {
                    videoPlayer.togglePiP()
                }
                .disabled(!videoPlayer.isPiPPossible)
            }
        }
    }
}
```

---

## 2. バックグラウンドオーディオ

PiPを維持するにはバックグラウンドオーディオが必要。

```swift
import AVFoundation

class AudioSessionManager {
    static func configure() {
        do {
            try AVAudioSession.sharedInstance().setCategory(
                .playback,
                mode: .moviePlayback,
                options: []
            )
            try AVAudioSession.sharedInstance().setActive(true)
        } catch {
            print("Audio session error: \(error)")
        }
    }
}
```

### Info.plist

```xml
<key>UIBackgroundModes</key>
<array>
    <string>audio</string>
</array>
```

---

## 3. Safari Extension でPiP（PiPifier方式）

### Extension構造

```
PiPifier/
├── Extension/
│   ├── SafariWebExtensionHandler.swift
│   ├── Resources/
│   │   ├── manifest.json
│   │   ├── content.js
│   │   └── popup.html
```

### manifest.json

```json
{
  "manifest_version": 3,
  "name": "PiPifier",
  "version": "1.0",
  "permissions": ["activeTab"],
  "action": {
    "default_popup": "popup.html"
  },
  "content_scripts": [{
    "matches": ["<all_urls>"],
    "js": ["content.js"]
  }]
}
```

### content.js

```javascript
// ページ内の動画を検出してPiPを有効化
function enablePiP() {
    const videos = document.querySelectorAll('video');

    videos.forEach(video => {
        if (document.pictureInPictureEnabled && !video.disablePictureInPicture) {
            video.requestPictureInPicture().catch(e => {
                console.log('PiP failed:', e);
            });
        }
    });
}

// メッセージを受信
browser.runtime.onMessage.addListener((message) => {
    if (message.action === 'togglePiP') {
        enablePiP();
    }
});
```

---

## 4. AVPictureInPictureSampleBufferPlaybackDelegate

カスタムレンダリング用（ゲーム画面など）。

```swift
import AVKit

class CustomPiPController: NSObject {
    private var pipController: AVPictureInPictureController?
    private let sampleBufferDisplayLayer = AVSampleBufferDisplayLayer()

    func setup() {
        guard AVPictureInPictureController.isPictureInPictureSupported() else { return }

        let contentSource = AVPictureInPictureController.ContentSource(
            sampleBufferDisplayLayer: sampleBufferDisplayLayer,
            playbackDelegate: self
        )

        pipController = AVPictureInPictureController(contentSource: contentSource)
        pipController?.delegate = self
    }

    func enqueueSampleBuffer(_ sampleBuffer: CMSampleBuffer) {
        sampleBufferDisplayLayer.enqueue(sampleBuffer)
    }
}

extension CustomPiPController: AVPictureInPictureSampleBufferPlaybackDelegate {
    func pictureInPictureController(
        _ pictureInPictureController: AVPictureInPictureController,
        setPlaying playing: Bool
    ) {
        // 再生/一時停止
    }

    func pictureInPictureControllerTimeRangeForPlayback(
        _ pictureInPictureController: AVPictureInPictureController
    ) -> CMTimeRange {
        CMTimeRange(start: .zero, duration: CMTime(seconds: 3600, preferredTimescale: 600))
    }

    func pictureInPictureControllerIsPlaybackPaused(
        _ pictureInPictureController: AVPictureInPictureController
    ) -> Bool {
        false
    }

    func pictureInPictureController(
        _ pictureInPictureController: AVPictureInPictureController,
        didTransitionToRenderSize newRenderSize: CMVideoDimensions
    ) {
        // サイズ変更時
    }

    func pictureInPictureController(
        _ pictureInPictureController: AVPictureInPictureController,
        skipByInterval skipInterval: CMTime
    ) async {
        // スキップ
    }
}
```

---

## まとめ

| 方式 | 用途 | 難易度 |
|------|------|--------|
| AVPlayer + PiP | 標準動画再生 | 低 |
| Safari Extension | Web動画 | 中 |
| SampleBuffer | カスタムレンダリング | 高 |

### 要件

- バックグラウンドオーディオ有効
- AVAudioSession.category = .playback
- 動画再生中のみPiP可能
