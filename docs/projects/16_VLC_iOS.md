# VLC for iOS

**スター**: 6,369 ⭐
**リポジトリ**: https://github.com/videolan/vlc-ios
**ライセンス**: GPL-2.0

---

## 概要

VLC for iOSは、VideoLANプロジェクトによるクロスプラットフォームメディアプレーヤーのiOS版。ほぼすべての動画・音声フォーマットに対応し、ネットワークストリーミング、字幕、再生速度調整など豊富な機能を持つ。

---

## 主要機能

| 機能 | 説明 |
|------|------|
| 幅広いフォーマット | MKV, AVI, MP4, FLV等 |
| ネットワーク再生 | SMB, FTP, NFS, DLNA |
| 字幕 | SRT, SSA, 埋め込み字幕 |
| 再生制御 | 速度、イコライザー |
| Chromecast | キャスト対応 |
| CarPlay | 車載連携 |

---

## 技術スタック

| 項目 | 技術 |
|------|------|
| 言語 | Swift, Objective-C, C |
| UI | UIKit |
| メディア | VLCKit (libVLC) |
| ネットワーク | VLCKit内蔵 |
| ストレージ | Core Data |
| 同期 | iCloud, Dropbox |

---

## アーキテクチャ

```
vlc-ios/
├── Sources/                 # Swiftソースコード
│   ├── App/                 # アプリ起動
│   ├── Playback/            # 再生機能
│   ├── Library/             # メディアライブラリ
│   ├── Network/             # ネットワーク機能
│   └── Settings/            # 設定
├── Resources/               # リソース
├── VLCKit/                  # メディアエンジン
│   └── libVLC/              # C ライブラリ
└── Extensions/              # App Extensions
    ├── Widget/              # ウィジェット
    └── Share/               # 共有拡張
```

---

## 学習価値

### 大規模メディアアプリ

| 分野 | 学習ポイント |
|------|-------------|
| **メディア再生** | VLCKit統合 |
| **ネットワーク** | 複数プロトコル対応 |
| **CarPlay** | 車載アプリ開発 |
| **レガシー統合** | ObjC/C統合 |
| **App Extensions** | ウィジェット等 |

### コードパターン

**VLCKit メディア再生**:
```swift
// VLCMediaPlayer を使用した再生
class VideoPlayerController {
    private var mediaPlayer: VLCMediaPlayer
    private var media: VLCMedia?

    init() {
        mediaPlayer = VLCMediaPlayer()
        mediaPlayer.delegate = self
        mediaPlayer.drawable = videoView
    }

    func play(url: URL) {
        media = VLCMedia(url: url)
        media?.addOptions([
            "network-caching": 3000,
            "file-caching": 1500
        ])
        mediaPlayer.media = media
        mediaPlayer.play()
    }

    func setPlaybackRate(_ rate: Float) {
        mediaPlayer.rate = rate
    }

    func setSubtitle(track: Int) {
        mediaPlayer.currentVideoSubTitleIndex = Int32(track)
    }
}

extension VideoPlayerController: VLCMediaPlayerDelegate {
    func mediaPlayerStateChanged(_ notification: Notification) {
        switch mediaPlayer.state {
        case .playing: updateNowPlayingInfo()
        case .paused: updatePlaybackState(.paused)
        case .ended: handlePlaybackEnded()
        default: break
        }
    }
}
```

**ネットワークブラウジング**:
```swift
// SMB/FTPブラウジング
class NetworkBrowser: ObservableObject {
    @Published var servers: [NetworkServer] = []

    private var discovery: VLCMediaDiscoverer?

    func startDiscovery() {
        discovery = VLCMediaDiscoverer(name: "upnp")
        discovery?.start()

        discovery?.mediaList.delegate = self
    }

    func browse(server: NetworkServer) async throws -> [NetworkFile] {
        let media = VLCMedia(url: server.url)
        let list = VLCMediaList()
        list.media = media

        return list.map { item in
            NetworkFile(
                name: item.metaData.title,
                url: item.url,
                isDirectory: item.mediaType == .directory
            )
        }
    }
}
```

**CarPlay統合**:
```swift
// CarPlay 対応
class CarPlaySceneDelegate: UIResponder, CPTemplateApplicationSceneDelegate {
    var interfaceController: CPInterfaceController?

    func templateApplicationScene(
        _ scene: CPTemplateApplicationScene,
        didConnect interfaceController: CPInterfaceController
    ) {
        self.interfaceController = interfaceController

        let listTemplate = CPListTemplate(
            title: "VLC",
            sections: [createMediaSection()]
        )
        interfaceController.setRootTemplate(listTemplate, animated: true)
    }

    private func createMediaSection() -> CPListSection {
        let items = mediaLibrary.recentMedia.map { media in
            CPListItem(
                text: media.title,
                detailText: media.artist,
                image: media.artwork
            ) { [weak self] item, completion in
                self?.playMedia(media)
                completion()
            }
        }
        return CPListSection(items: items)
    }
}
```

---

## プロジェクト評価

| 観点 | 評価 | 理由 |
|------|------|------|
| **コード品質** | ★★★★☆ | レガシーコード含む |
| **ドキュメント** | ★★★★☆ | ビルドガイド充実 |
| **アクティビティ** | ★★★★★ | 継続的な開発 |
| **学習価値** | ★★★★★ | メディアアプリの教科書 |
| **実用性** | ★★★★★ | 世界的人気アプリ |

---

## VLCKit統合

### セットアップ

```swift
// Podfile
pod 'MobileVLCKit', '~> 3.6.0'

// Package.swift
.package(url: "https://github.com/nicko88/VLCKitSPM", from: "3.6.0")
```

### メディアオプション

```swift
// 詳細設定
let options = [
    "--network-caching=3000",
    "--file-caching=1500",
    "--avcodec-hw=any",
    "--audio-time-stretch"
]

VLCLibrary.shared().setVerbose(true, options: options)
```

---

## App Extensions

### ウィジェット

```swift
// 再生中ウィジェット
struct NowPlayingWidget: Widget {
    var body: some WidgetConfiguration {
        StaticConfiguration(
            kind: "NowPlaying",
            provider: NowPlayingProvider()
        ) { entry in
            NowPlayingView(entry: entry)
        }
        .configurationDisplayName("Now Playing")
        .supportedFamilies([.systemSmall, .systemMedium])
    }
}
```

---

## 開発者向け情報

### ビルド手順

```bash
# VLCKit ビルドが必要
git clone https://github.com/videolan/vlc-ios
cd vlc-ios
./buildMobileVLCKit.sh -a arm64

open VLC.xcodeproj
```

### 必要要件

- Xcode 15+
- CocoaPods
- VLCKit ビルド環境

---

## まとめ

VLC for iOSは、プロダクション品質のメディアアプリ開発を学ぶ最良の教材。VLCKit統合、ネットワークストリーミング、CarPlay対応など、高度なメディア機能を網羅。大規模オープンソースプロジェクトの運営も参考になる。

**推奨対象**:
- メディアプレーヤーを開発したい人
- CarPlay対応アプリを作りたい開発者
- C/ObjCライブラリ統合を学びたい人
- ネットワークストリーミングを実装したい人
