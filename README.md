# オープンソースiOSアプリ調査

**調査日**: 2025年12月25日

## ディレクトリ構造

```
open-source-ios-apps-research/
├── README.md                    # このファイル
│
├── repo/                        # クローンしたリポジトリ
│   ├── contents.json            # アプリデータ (1,639件)
│   ├── README.md                # オリジナルREADME
│   ├── APPSTORE.md              # App Store公開アプリ
│   ├── ARCHIVE.md               # アーカイブ済みアプリ
│   └── ...
│
└── docs/                        # 調査ドキュメント
    ├── README.md                # ドキュメント索引
    │
    ├── categories/              # カテゴリ別分析レポート (22件)
    │   ├── 00_レポート索引.md
    │   ├── 01_概要統計.md
    │   ├── ...
    │   └── 21_ニュース＆RSSアプリ.md
    │
    └── projects/                # プロジェクト別詳細ドキュメント (29件)
        ├── 00_プロジェクト索引.md
        ├── 01_UTM.md
        ├── ...
        └── 28_Jitsi_Meet.md
```

## 概要

[dkhamsing/open-source-ios-apps](https://github.com/dkhamsing/open-source-ios-apps) リポジトリを分析し、2025年12月時点でのiOS開発トレンド、ベストプラクティス、Swiftパッケージ機会を調査。

## 主要発見

| 項目 | 数値 |
|------|------|
| 総プロジェクト数 | 1,639 |
| Swift製アプリ | 1,001 (61%) |
| SwiftUI使用 | 207 |
| visionOS対応 | 17 |
| カテゴリレポート | 22 |
| プロジェクトドキュメント | 29 |

---

## ドキュメント

### カテゴリ別レポート

[**→ docs/categories/**](docs/categories/)

1,639件のアプリを分野別に分析。

| レポート | 内容 |
|---------|------|
| [概要統計](docs/categories/01_概要統計.md) | 全体統計と傾向 |
| [SwiftUI分析](docs/categories/02_SwiftUIアプリ分析.md) | SwiftUIアプリ207件 |
| [アーキテクチャ](docs/categories/03_アーキテクチャパターン.md) | 設計パターン分析 |
| [パッケージアイデア](docs/categories/05_Swiftパッケージアイデア.md) | SPM開発アイデア |
| [visionOS](docs/categories/06_visionOSアプリ分析.md) | visionOS対応17件 |
| [最終まとめ](docs/categories/12_最終まとめと提言.md) | 総括と推奨事項 |

全22レポート → [索引](docs/categories/00_レポート索引.md)

### プロジェクト別ドキュメント

[**→ docs/projects/**](docs/projects/)

人気上位28プロジェクトを個別に詳細分析。

| プロジェクト | スター | カテゴリ | 主要技術 |
|------------|--------|---------|---------|
| [UTM](docs/projects/01_UTM.md) | 31,921 | 仮想化 | Metal, C/Swift |
| [AltStore](docs/projects/02_AltStore.md) | 13,277 | 配布 | Code Signing |
| [Signal](docs/projects/05_Signal.md) | 11,736 | メッセージング | Signal Protocol |
| [Ice Cubes](docs/projects/10_Ice_Cubes.md) | 6,748 | Mastodon | @Observable, SwiftData |
| [Kickstarter](docs/projects/08_Kickstarter.md) | 8,601 | クラウドファンディング | MVVM, ReactiveSwift |

全28プロジェクト → [索引](docs/projects/00_プロジェクト索引.md)

---

## 技術別ハイライト

### SwiftUI学習

| プロジェクト | 特徴 |
|------------|------|
| [Ice Cubes](docs/projects/10_Ice_Cubes.md) | @Observable, SwiftData, SPM |
| [MovieSwiftUI](docs/projects/14_MovieSwiftUI.md) | Flux/Redux, Combine |
| [Clean Architecture](docs/projects/15_CleanArchitectureSwiftUI.md) | Clean Architecture実装 |

### セキュリティ実装

| プロジェクト | 特徴 |
|------------|------|
| [Signal](docs/projects/05_Signal.md) | Signal Protocol, E2E |
| [Bitwarden](docs/projects/19_Bitwarden.md) | Keychain, AutoFill |
| [SimpleX Chat](docs/projects/06_SimpleX_Chat.md) | User-IDなし設計 |

### システム統合

| プロジェクト | 特徴 |
|------------|------|
| [UTM](docs/projects/01_UTM.md) | Metal, 仮想化 |
| [Mullvad VPN](docs/projects/13_Mullvad_VPN.md) | NetworkExtension, Rust FFI |
| [VLC iOS](docs/projects/16_VLC_iOS.md) | CarPlay, メディア処理 |

---

## 元リポジトリ

- https://github.com/dkhamsing/open-source-ios-apps
