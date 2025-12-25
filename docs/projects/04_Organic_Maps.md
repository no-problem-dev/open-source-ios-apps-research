# Organic Maps

**スター**: 12,352 ⭐
**リポジトリ**: https://github.com/organicmaps/organicmaps
**ライセンス**: Apache-2.0

---

## 概要

Organic Mapsは、プライバシー重視の完全オフラインマップアプリ。MAPS.MEのフォークとして開始され、広告なし、トラッキングなし、寄付ベースで運営。ハイキング、サイクリング、ドライブ向けナビゲーション機能を提供。

---

## 主要機能

| 機能 | 説明 |
|------|------|
| オフラインマップ | 事前ダウンロード対応 |
| ナビゲーション | ターンバイターン音声案内 |
| 検索 | POI・住所検索（オフライン） |
| ルート | 徒歩・自転車・車対応 |
| ブックマーク | 場所の保存・整理 |
| 地形図 | 等高線表示 |

---

## 技術スタック

| 項目 | 技術 |
|------|------|
| コア | C++ |
| iOS UI | Swift, Objective-C++ |
| マップデータ | OpenStreetMap |
| レンダリング | OpenGL ES / Metal |
| ビルド | CMake, Xcode |

---

## アーキテクチャ

```
organicmaps/
├── iphone/              # iOS アプリ
│   ├── Maps/            # メインアプリ
│   ├── CoreApi/         # C++ブリッジ
│   └── Widget/          # ウィジェット
├── android/             # Android アプリ
├── map/                 # マップレンダリング
├── routing/             # ルート計算
├── search/              # 検索エンジン
├── storage/             # データストレージ
└── drape/               # 描画エンジン
```

---

## 学習価値

### クロスプラットフォーム設計

| 分野 | 学習ポイント |
|------|-------------|
| **C++/Swift連携** | Objective-C++ブリッジ |
| **OpenGL/Metal** | カスタム描画 |
| **オフラインファースト** | データ永続化戦略 |
| **マップタイル** | 地図データ管理 |
| **ルーティング** | 経路計算アルゴリズム |

### コードパターン

**C++ブリッジ**:
```objc
// Objective-C++ でのブリッジ
@interface MWMMapView : UIView
- (void)setCenter:(CLLocationCoordinate2D)center zoom:(float)zoom;
- (void)searchQuery:(NSString *)query;
@end

@implementation MWMMapView
- (void)setCenter:(CLLocationCoordinate2D)center zoom:(float)zoom {
    // C++ Framework 呼び出し
    GetFramework().SetViewportCenter(
        MercatorBounds::FromLatLon(center.latitude, center.longitude),
        zoom
    );
}
@end
```

**オフラインデータ管理**:
```swift
class MapDownloader {
    func downloadRegion(_ region: MapRegion) async throws {
        let url = region.downloadURL
        let (data, _) = try await URLSession.shared.data(from: url)
        try saveToLocalStorage(data, for: region)
    }
}
```

---

## プロジェクト評価

| 観点 | 評価 | 理由 |
|------|------|------|
| **コード品質** | ★★★★☆ | 大規模C++プロジェクト |
| **ドキュメント** | ★★★★☆ | ビルドガイド充実 |
| **アクティビティ** | ★★★★★ | 活発なコミュニティ |
| **学習価値** | ★★★★★ | オフラインマップの参考 |
| **実用性** | ★★★★★ | 実用的なマップアプリ |

---

## プライバシー特徴

| 項目 | 対応 |
|------|------|
| トラッキング | なし |
| 広告 | なし |
| アナリティクス | なし |
| 位置情報送信 | なし（完全ローカル） |
| アカウント | 不要 |

---

## 開発者向け情報

### ビルド要件

- Xcode 14+
- CMake 3.21+
- Python 3

### ビルド手順

```bash
# サブモジュール取得
git submodule update --init --recursive

# ビルド
./configure.sh
xcodebuild -workspace xcode/omim.xcworkspace -scheme Maps
```

---

## まとめ

Organic Mapsは、プライバシーファーストのアプリ設計、オフラインファースト戦略、C++とSwiftの連携を学ぶための最良の教材。大規模クロスプラットフォームプロジェクトの実例として価値が高い。

**推奨対象**:
- オフラインファーストアプリ開発者
- C++/Swift連携を学びたい人
- マップ/位置情報アプリ開発者
- プライバシー重視のアプリ設計者

