# ドキュメント

オープンソースiOSアプリ調査の分析ドキュメント集。

---

## ディレクトリ構造

```
docs/
├── README.md              # このファイル
├── categories/            # カテゴリ別分析レポート
│   ├── 00_レポート索引.md
│   ├── 01_概要統計.md
│   ├── ...
│   └── 21_ニュース＆RSSアプリ.md
└── projects/              # プロジェクト別詳細ドキュメント
    ├── 00_プロジェクト索引.md
    ├── 01_UTM.md
    ├── ...
    └── 28_Jitsi_Meet.md
```

---

## カテゴリ別レポート

[**→ categories/**](categories/)

1,639件のオープンソースiOSアプリを分野別に分析したレポート集。

| # | レポート | 内容 |
|---|---------|------|
| 00 | [索引](categories/00_レポート索引.md) | 全レポートの索引 |
| 01 | [概要統計](categories/01_概要統計.md) | 全体統計と傾向 |
| 02 | [SwiftUI分析](categories/02_SwiftUIアプリ分析.md) | SwiftUIアプリ207件 |
| 03 | [アーキテクチャ](categories/03_アーキテクチャパターン.md) | 設計パターン分析 |
| 04 | [UI/UXパターン](categories/04_UIUXパターン.md) | UI実装パターン |
| 05 | [パッケージアイデア](categories/05_Swiftパッケージアイデア.md) | SPM開発アイデア |
| 06 | [visionOS](categories/06_visionOSアプリ分析.md) | visionOS対応17件 |
| 07 | [通信アプリ](categories/07_コミュニケーションアプリ.md) | メッセージング分析 |
| 08 | [メディア](categories/08_メディアアプリ分析.md) | メディアアプリ分析 |
| 09 | [開発者ツール](categories/09_開発者ツール分析.md) | 開発支援ツール |
| 10 | [ヘルス](categories/10_ヘルス＆フィットネス分析.md) | 健康・フィットネス |
| 11 | [拡張機能](categories/11_ウィジェット＆拡張機能.md) | ウィジェット・App Extensions |
| 12 | [まとめ](categories/12_最終まとめと提言.md) | 総括と推奨事項 |
| 13 | [金融](categories/13_金融＆暗号通貨アプリ.md) | 金融・暗号通貨 |
| 14 | [Apple公式](categories/14_Apple公式サンプル.md) | Apple公式サンプル |
| 15 | [ゲーム](categories/15_ゲームアプリ分析.md) | ゲームアプリ103件 |
| 16 | [地図](categories/16_位置情報＆マップアプリ.md) | 位置情報・マップ82件 |
| 17 | [クローン](categories/17_クローンアプリ学習ガイド.md) | 学習用クローン58件 |
| 18 | [教育](categories/18_教育＆学習アプリ.md) | 教育アプリ60件 |
| 19 | [ブラウザ](categories/19_ブラウザ＆Webアプリ.md) | ブラウザ・Web31件 |
| 20 | [セキュリティ](categories/20_セキュリティ＆プライバシー.md) | セキュリティ87件 |
| 21 | [ニュース](categories/21_ニュース＆RSSアプリ.md) | ニュース・RSS87件 |

---

## プロジェクト別ドキュメント

[**→ projects/**](projects/)

人気上位プロジェクトを個別に詳細分析したドキュメント集。

### 超人気（10,000+ ⭐）

| # | プロジェクト | スター | カテゴリ |
|---|------------|--------|---------|
| 01 | [UTM](projects/01_UTM.md) | 31,921 | 仮想化 |
| 02 | [AltStore](projects/02_AltStore.md) | 13,277 | 配布 |
| 03 | [Firefox iOS](projects/03_Firefox_iOS.md) | 12,796 | ブラウザ |
| 04 | [Organic Maps](projects/04_Organic_Maps.md) | 12,352 | 地図 |
| 05 | [Signal](projects/05_Signal.md) | 11,736 | メッセージング |
| 06 | [SimpleX Chat](projects/06_SimpleX_Chat.md) | 10,062 | プライバシー |

### 人気（5,000-10,000 ⭐）

| # | プロジェクト | スター | カテゴリ |
|---|------------|--------|---------|
| 07 | [NetNewsWire](projects/07_NetNewsWire.md) | 9,399 | RSS |
| 08 | [Kickstarter](projects/08_Kickstarter.md) | 8,601 | クラウドファンディング |
| 09 | [Telegram](projects/09_Telegram.md) | 7,791 | メッセージング |
| 10 | [Ice Cubes](projects/10_Ice_Cubes.md) | 6,748 | Mastodon |
| 11 | [Bark](projects/11_Bark.md) | 7,209 | 通知 |
| 12 | [FSNotes](projects/12_FSNotes.md) | 7,108 | ノート |
| 13 | [Mullvad VPN](projects/13_Mullvad_VPN.md) | 6,537 | VPN |
| 14 | [MovieSwiftUI](projects/14_MovieSwiftUI.md) | 6,524 | デモ |
| 15 | [Clean Architecture](projects/15_CleanArchitectureSwiftUI.md) | 6,436 | テンプレート |
| 16 | [VLC iOS](projects/16_VLC_iOS.md) | 6,369 | メディア |
| 17 | [Amethyst](projects/17_Amethyst.md) | 5,732 | ウィンドウ管理 |
| 21 | [Provenance](projects/21_Provenance.md) | 5,800+ | エミュレータ |

### 注目（2,000-5,000 ⭐）

| # | プロジェクト | スター | カテゴリ |
|---|------------|--------|---------|
| 18 | [Wikipedia iOS](projects/18_Wikipedia_iOS.md) | 3,314 | 百科事典 |
| 19 | [Bitwarden](projects/19_Bitwarden.md) | 6,000+ | パスワード管理 |
| 20 | [Aidoku](projects/20_Aidoku.md) | 4,000+ | 漫画 |
| 22 | [Delta](projects/22_Delta.md) | 4,300+ | エミュレータ |
| 23 | [Passepartout](projects/23_Passepartout.md) | 3,500+ | VPN |
| 24 | [Mastonaut](projects/24_Mastonaut.md) | 1,400+ | Mastodon (macOS) |
| 25 | [Mlem](projects/25_Mlem.md) | 1,100+ | Lemmy |
| 26 | [Nextcloud](projects/26_Nextcloud.md) | 2,000+ | クラウド |
| 27 | [Element](projects/27_Element.md) | 1,900+ | Matrix |
| 28 | [Jitsi Meet](projects/28_Jitsi_Meet.md) | 2,500+ | ビデオ会議 |

---

## 技術別クイックリファレンス

### SwiftUI
- [Ice Cubes](projects/10_Ice_Cubes.md) - @Observable, SwiftData
- [MovieSwiftUI](projects/14_MovieSwiftUI.md) - Flux/Redux
- [Clean Architecture](projects/15_CleanArchitectureSwiftUI.md) - Clean Architecture

### セキュリティ
- [Signal](projects/05_Signal.md) - Signal Protocol
- [Bitwarden](projects/19_Bitwarden.md) - Keychain, AutoFill
- [Mullvad VPN](projects/13_Mullvad_VPN.md) - WireGuard

### メディア
- [VLC iOS](projects/16_VLC_iOS.md) - VLCKit, CarPlay
- [Provenance](projects/21_Provenance.md) - Metal, GameController

### ネットワーク
- [Element](projects/27_Element.md) - Matrix, WebRTC
- [Jitsi Meet](projects/28_Jitsi_Meet.md) - WebRTC, CallKit
