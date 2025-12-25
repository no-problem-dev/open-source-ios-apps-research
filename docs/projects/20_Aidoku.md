# Aidoku

**スター**: 4,000+ ⭐
**リポジトリ**: https://github.com/Aidoku/Aidoku
**ライセンス**: GPL-3.0

---

## 概要

Aidokuは、iOS向けのオープンソース漫画リーダー。拡張機能システムにより多数のソースに対応し、オフライン読書、カスタマイズ可能なリーダーを提供。SwiftUIで構築されたモダンなアプリ。

---

## 主要機能

| 機能 | 説明 |
|------|------|
| 拡張機能 | 多数のソース対応 |
| ライブラリ | 漫画管理 |
| リーダー | カスタマイズ可能 |
| ダウンロード | オフライン読書 |
| トラッキング | 進捗管理 |
| iCloud同期 | デバイス間同期 |

---

## 技術スタック

| 項目 | 技術 |
|------|------|
| 言語 | Swift |
| UI | SwiftUI |
| アーキテクチャ | MVVM |
| 拡張システム | WasmInterpreter |
| ストレージ | Core Data |
| 画像 | Nuke |

---

## アーキテクチャ

```
Aidoku/
├── Aidoku/                  # メインアプリ
│   ├── Views/               # SwiftUI ビュー
│   │   ├── Library/         # ライブラリ画面
│   │   ├── Browse/          # ブラウズ画面
│   │   ├── Reader/          # リーダー
│   │   └── Settings/        # 設定
│   ├── Models/              # データモデル
│   ├── Sources/             # ソース管理
│   └── Managers/            # マネージャー
├── AidokuCore/              # コアフレームワーク
│   ├── Source/              # ソースプロトコル
│   └── Wasm/                # WASM実行環境
└── Extensions/              # 拡張機能
```

---

## 学習価値

### 拡張機能システム

| 分野 | 学習ポイント |
|------|-------------|
| **WASM統合** | WebAssembly実行 |
| **プラグインアーキテクチャ** | 拡張可能設計 |
| **SwiftUI** | 複雑なUI実装 |
| **画像処理** | 効率的な画像読込 |
| **オフライン** | ダウンロード管理 |

### コードパターン

**WASM拡張実行**:
```swift
// WebAssembly ソース実行
class WasmSource: Source {
    private let runtime: WasmRuntime
    private var instance: WasmInstance?

    init(wasmData: Data) throws {
        runtime = WasmRuntime()
        instance = try runtime.instantiate(wasmData)
    }

    func getMangaList(page: Int) async throws -> [Manga] {
        guard let instance = instance else {
            throw SourceError.notInitialized
        }

        // WASM関数呼び出し
        let result = try instance.call(
            "get_manga_list",
            args: [.i32(Int32(page))]
        )

        // 結果をパース
        return try parseMangaList(from: result)
    }

    func getChapters(mangaId: String) async throws -> [Chapter] {
        let result = try instance?.call(
            "get_chapters",
            args: [.string(mangaId)]
        )
        return try parseChapters(from: result)
    }
}
```

**リーダービュー**:
```swift
// ページリーダー
struct ReaderView: View {
    @StateObject private var viewModel: ReaderViewModel
    @State private var currentPage: Int = 0
    @State private var showControls: Bool = false

    var body: some View {
        GeometryReader { geometry in
            TabView(selection: $currentPage) {
                ForEach(viewModel.pages.indices, id: \.self) { index in
                    PageView(page: viewModel.pages[index])
                        .tag(index)
                }
            }
            .tabViewStyle(.page(indexDisplayMode: .never))
            .gesture(tapGesture)
            .overlay(controlsOverlay)
        }
        .ignoresSafeArea()
        .onAppear { viewModel.loadPages() }
    }

    private var tapGesture: some Gesture {
        TapGesture()
            .onEnded { _ in
                withAnimation { showControls.toggle() }
            }
    }

    @ViewBuilder
    private var controlsOverlay: some View {
        if showControls {
            ReaderControlsView(
                currentPage: currentPage,
                totalPages: viewModel.pages.count,
                onPageChange: { currentPage = $0 }
            )
            .transition(.opacity)
        }
    }
}
```

**ダウンロード管理**:
```swift
// チャプターダウンロード
class DownloadManager: ObservableObject {
    @Published var downloads: [Download] = []

    private let queue = OperationQueue()

    func download(chapter: Chapter, manga: Manga) {
        let download = Download(chapter: chapter, manga: manga)
        downloads.append(download)

        let operation = DownloadOperation(download: download)
        operation.progressHandler = { [weak self] progress in
            DispatchQueue.main.async {
                self?.updateProgress(download.id, progress: progress)
            }
        }
        operation.completionHandler = { [weak self] result in
            DispatchQueue.main.async {
                self?.handleCompletion(download.id, result: result)
            }
        }

        queue.addOperation(operation)
    }

    func cancelDownload(_ id: UUID) {
        queue.operations
            .compactMap { $0 as? DownloadOperation }
            .first { $0.download.id == id }?
            .cancel()
    }
}
```

**ライブラリ管理**:
```swift
// ライブラリ + Core Data
class LibraryManager: ObservableObject {
    @Published var manga: [LibraryManga] = []

    private let context: NSManagedObjectContext

    func addToLibrary(_ manga: Manga, source: Source) {
        let libraryManga = LibraryManga(context: context)
        libraryManga.id = manga.id
        libraryManga.title = manga.title
        libraryManga.coverURL = manga.coverURL
        libraryManga.sourceId = source.id
        libraryManga.addedDate = Date()

        try? context.save()
        fetchLibrary()
    }

    func updateReadProgress(manga: LibraryManga, chapter: Chapter, page: Int) {
        manga.lastReadChapter = chapter.id
        manga.lastReadPage = Int32(page)
        manga.lastReadDate = Date()

        try? context.save()
    }
}
```

---

## プロジェクト評価

| 観点 | 評価 | 理由 |
|------|------|------|
| **コード品質** | ★★★★☆ | モダンSwiftUI |
| **ドキュメント** | ★★★★☆ | 拡張開発ガイド |
| **アクティビティ** | ★★★★★ | 活発な開発 |
| **学習価値** | ★★★★★ | プラグインアーキテクチャ |
| **実用性** | ★★★★★ | 人気漫画リーダー |

---

## プラグインシステム

### 拡張機能構造

```swift
// ソースプロトコル
protocol Source {
    var id: String { get }
    var name: String { get }
    var language: String { get }

    func getMangaList(page: Int) async throws -> [Manga]
    func getMangaDetails(id: String) async throws -> MangaDetails
    func getChapters(mangaId: String) async throws -> [Chapter]
    func getPages(chapterId: String) async throws -> [Page]
}
```

### WASM拡張仕様

```rust
// Rust で拡張機能を実装
#[no_mangle]
pub extern "C" fn get_manga_list(page: i32) -> *const c_char {
    let url = format!("https://example.com/manga?page={}", page);
    let response = fetch(&url);
    let manga = parse_manga_list(&response);
    return_json(&manga)
}
```

---

## 開発者向け情報

### ビルド手順

```bash
git clone https://github.com/Aidoku/Aidoku
cd Aidoku
open Aidoku.xcodeproj
```

### 拡張機能開発

拡張機能はRustで実装し、WASMにコンパイル。

---

## まとめ

Aidokuは、プラグインアーキテクチャとSwiftUIを組み合わせた優れた実装例。WASM統合、効率的な画像処理、オフライン機能など、実践的なパターンを学べる。拡張可能なアプリ設計の参考になる。

**推奨対象**:
- プラグインアーキテクチャを学びたい開発者
- SwiftUIで複雑なUIを実装したい人
- WASM統合に興味がある人
- メディアリーダーを開発したい人
