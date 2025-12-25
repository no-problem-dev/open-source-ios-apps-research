# FileProvider Extension 実装ガイド

**参考プロジェクト**:
- Cryptomator: https://github.com/nickolashkraus/cryptomator-ios
- クラウドストレージアプリ

**実装参考度**: ⭐⭐⭐⭐☆

---

## なぜ重要か

FileProvider ExtensionはFiles.appとの統合を実現。クラウドストレージ、暗号化ボルト、カスタムファイルシステムを提供できる。

| 学習ポイント | 重要度 | 理由 |
|-------------|--------|------|
| **NSFileProviderExtension** | ⭐⭐⭐⭐⭐ | 基本クラス |
| **NSFileProviderItem** | ⭐⭐⭐⭐⭐ | ファイル/フォルダ表現 |
| **Enumeration** | ⭐⭐⭐⭐☆ | コンテンツ列挙 |
| **Downloads/Uploads** | ⭐⭐⭐⭐☆ | ファイル転送 |

---

## 1. Extension作成

### Xcodeで追加

```
File → New → Target → File Provider Extension
```

### Capability設定

```
メインアプリとExtensionの両方:
- App Groups
- File Provider (Extension のみ)
```

---

## 2. 基本構造

### NSFileProviderItem

```swift
import FileProvider

class FileProviderItem: NSObject, NSFileProviderItem {
    let identifier: NSFileProviderItemIdentifier
    let parentIdentifier: NSFileProviderItemIdentifier
    let filename: String
    let typeIdentifier: String
    let documentSize: NSNumber?
    let creationDate: Date?
    let contentModificationDate: Date?
    let isFolder: Bool

    // 必須プロパティ
    var itemIdentifier: NSFileProviderItemIdentifier {
        return identifier
    }

    var parentItemIdentifier: NSFileProviderItemIdentifier {
        return parentIdentifier
    }

    var capabilities: NSFileProviderItemCapabilities {
        if isFolder {
            return [.allowsReading, .allowsWriting, .allowsAddingSubItems, .allowsDeleting]
        } else {
            return [.allowsReading, .allowsWriting, .allowsDeleting, .allowsRenaming]
        }
    }

    init(
        identifier: NSFileProviderItemIdentifier,
        parentIdentifier: NSFileProviderItemIdentifier,
        filename: String,
        isFolder: Bool = false,
        documentSize: Int? = nil,
        creationDate: Date? = nil,
        contentModificationDate: Date? = nil
    ) {
        self.identifier = identifier
        self.parentIdentifier = parentIdentifier
        self.filename = filename
        self.isFolder = isFolder
        self.documentSize = documentSize.map { NSNumber(value: $0) }
        self.creationDate = creationDate
        self.contentModificationDate = contentModificationDate

        if isFolder {
            self.typeIdentifier = "public.folder"
        } else {
            self.typeIdentifier = UTType(filenameExtension: URL(fileURLWithPath: filename).pathExtension)?.identifier ?? "public.data"
        }
    }
}
```

### NSFileProviderExtension

```swift
import FileProvider
import UniformTypeIdentifiers

class FileProviderExtension: NSFileProviderExtension {
    // ドメインマネージャー
    var domain: NSFileProviderDomain?

    override init() {
        super.init()
    }

    // MARK: - Item Management

    override func item(for identifier: NSFileProviderItemIdentifier) throws -> NSFileProviderItem {
        // 識別子からアイテムを取得
        guard let item = fetchItem(for: identifier) else {
            throw NSFileProviderError(.noSuchItem)
        }
        return item
    }

    override func urlForItem(withPersistentIdentifier identifier: NSFileProviderItemIdentifier) -> URL? {
        // アイテムのローカルURL
        guard let item = try? item(for: identifier) else {
            return nil
        }

        let manager = NSFileProviderManager.default
        let perItemDirectory = manager.documentStorageURL
            .appendingPathComponent(identifier.rawValue, isDirectory: true)

        return perItemDirectory.appendingPathComponent(item.filename)
    }

    override func persistentIdentifierForItem(at url: URL) -> NSFileProviderItemIdentifier? {
        // URLから識別子を取得
        let pathComponents = url.pathComponents
        guard pathComponents.count >= 2 else { return nil }
        return NSFileProviderItemIdentifier(pathComponents[pathComponents.count - 2])
    }

    // MARK: - Providing Items

    override func providePlaceholder(at url: URL, completionHandler: @escaping (Error?) -> Void) {
        guard let identifier = persistentIdentifierForItem(at: url),
              let item = try? item(for: identifier) as? FileProviderItem else {
            completionHandler(NSFileProviderError(.noSuchItem))
            return
        }

        do {
            try FileManager.default.createDirectory(
                at: url.deletingLastPathComponent(),
                withIntermediateDirectories: true
            )

            try NSFileProviderManager.writePlaceholder(
                at: url,
                withMetadata: item
            )
            completionHandler(nil)
        } catch {
            completionHandler(error)
        }
    }

    override func startProvidingItem(
        at url: URL,
        completionHandler: @escaping (Error?) -> Void
    ) {
        guard let identifier = persistentIdentifierForItem(at: url) else {
            completionHandler(NSFileProviderError(.noSuchItem))
            return
        }

        // ファイルをダウンロード
        downloadFile(identifier: identifier) { result in
            switch result {
            case .success(let data):
                do {
                    try FileManager.default.createDirectory(
                        at: url.deletingLastPathComponent(),
                        withIntermediateDirectories: true
                    )
                    try data.write(to: url)
                    completionHandler(nil)
                } catch {
                    completionHandler(error)
                }
            case .failure(let error):
                completionHandler(error)
            }
        }
    }

    override func stopProvidingItem(at url: URL) {
        // クリーンアップ（オプション）
        try? FileManager.default.removeItem(at: url)
        providePlaceholder(at: url) { _ in }
    }

    // MARK: - Enumeration

    override func enumerator(
        for containerItemIdentifier: NSFileProviderItemIdentifier
    ) throws -> NSFileProviderEnumerator {
        return FileProviderEnumerator(
            identifier: containerItemIdentifier,
            extension: self
        )
    }

    // MARK: - Actions

    override func createDirectory(
        withName directoryName: String,
        inParentItemIdentifier parentItemIdentifier: NSFileProviderItemIdentifier,
        completionHandler: @escaping (NSFileProviderItem?, Error?) -> Void
    ) {
        // ディレクトリ作成処理
        createDirectoryOnServer(name: directoryName, parent: parentItemIdentifier) { result in
            switch result {
            case .success(let item):
                completionHandler(item, nil)
            case .failure(let error):
                completionHandler(nil, error)
            }
        }
    }

    override func deleteItem(
        withIdentifier itemIdentifier: NSFileProviderItemIdentifier,
        completionHandler: @escaping (Error?) -> Void
    ) {
        // 削除処理
        deleteItemOnServer(identifier: itemIdentifier) { error in
            completionHandler(error)
        }
    }

    override func renameItem(
        withIdentifier itemIdentifier: NSFileProviderItemIdentifier,
        toName itemName: String,
        completionHandler: @escaping (NSFileProviderItem?, Error?) -> Void
    ) {
        // リネーム処理
        renameItemOnServer(identifier: itemIdentifier, newName: itemName) { result in
            switch result {
            case .success(let item):
                completionHandler(item, nil)
            case .failure(let error):
                completionHandler(nil, error)
            }
        }
    }

    // MARK: - Private Methods

    private func fetchItem(for identifier: NSFileProviderItemIdentifier) -> FileProviderItem? {
        // データベースまたはAPIからアイテムを取得
        return nil
    }

    private func downloadFile(
        identifier: NSFileProviderItemIdentifier,
        completion: @escaping (Result<Data, Error>) -> Void
    ) {
        // サーバーからファイルをダウンロード
    }

    private func createDirectoryOnServer(
        name: String,
        parent: NSFileProviderItemIdentifier,
        completion: @escaping (Result<FileProviderItem, Error>) -> Void
    ) {
        // サーバーにディレクトリを作成
    }

    private func deleteItemOnServer(
        identifier: NSFileProviderItemIdentifier,
        completion: @escaping (Error?) -> Void
    ) {
        // サーバーからアイテムを削除
    }

    private func renameItemOnServer(
        identifier: NSFileProviderItemIdentifier,
        newName: String,
        completion: @escaping (Result<FileProviderItem, Error>) -> Void
    ) {
        // サーバーでアイテムをリネーム
    }
}
```

---

## 3. Enumerator実装

```swift
class FileProviderEnumerator: NSObject, NSFileProviderEnumerator {
    private let identifier: NSFileProviderItemIdentifier
    private weak var fileProviderExtension: FileProviderExtension?

    init(identifier: NSFileProviderItemIdentifier, extension: FileProviderExtension) {
        self.identifier = identifier
        self.fileProviderExtension = `extension`
    }

    func invalidate() {
        // クリーンアップ
    }

    func enumerateItems(
        for observer: NSFileProviderEnumerationObserver,
        startingAt page: NSFileProviderPage
    ) {
        // アイテムを列挙
        fetchItems(in: identifier) { items in
            observer.didEnumerate(items)
            observer.finishEnumerating(upTo: nil)
        }
    }

    func enumerateChanges(
        for observer: NSFileProviderChangeObserver,
        from anchor: NSFileProviderSyncAnchor
    ) {
        // 変更を列挙
        fetchChanges(since: anchor) { deletedItems, updatedItems, newAnchor in
            observer.didDeleteItems(withIdentifiers: deletedItems)
            observer.didUpdate(updatedItems)
            observer.finishEnumeratingChanges(upTo: newAnchor, moreComing: false)
        }
    }

    func currentSyncAnchor(completionHandler: @escaping (NSFileProviderSyncAnchor?) -> Void) {
        // 現在の同期アンカーを返す
        let anchor = NSFileProviderSyncAnchor(Data())
        completionHandler(anchor)
    }

    private func fetchItems(
        in identifier: NSFileProviderItemIdentifier,
        completion: @escaping ([NSFileProviderItem]) -> Void
    ) {
        // サーバーまたはキャッシュからアイテムを取得
        completion([])
    }

    private func fetchChanges(
        since anchor: NSFileProviderSyncAnchor,
        completion: @escaping ([NSFileProviderItemIdentifier], [NSFileProviderItem], NSFileProviderSyncAnchor) -> Void
    ) {
        // 変更を取得
        let newAnchor = NSFileProviderSyncAnchor(Data())
        completion([], [], newAnchor)
    }
}
```

---

## 4. ドメイン登録

### メインアプリでの登録

```swift
import FileProvider

class FileProviderManager {
    static let shared = FileProviderManager()

    private let domainIdentifier = NSFileProviderDomainIdentifier("com.example.app.fileprovider")

    func registerDomain() {
        let domain = NSFileProviderDomain(
            identifier: domainIdentifier,
            displayName: "My Cloud Storage"
        )

        NSFileProviderManager.add(domain) { error in
            if let error = error {
                print("Failed to add domain: \(error)")
            } else {
                print("Domain added successfully")
            }
        }
    }

    func removeDomain() {
        NSFileProviderManager.remove(domain: NSFileProviderDomain(
            identifier: domainIdentifier,
            displayName: ""
        )) { error in
            if let error = error {
                print("Failed to remove domain: \(error)")
            }
        }
    }

    func signalEnumerator(for identifier: NSFileProviderItemIdentifier) {
        NSFileProviderManager.default.signalEnumerator(for: identifier) { error in
            if let error = error {
                print("Failed to signal enumerator: \(error)")
            }
        }
    }
}
```

---

## 5. サムネイル提供

```swift
extension FileProviderExtension {
    override func fetchThumbnails(
        for itemIdentifiers: [NSFileProviderItemIdentifier],
        requestedSize size: CGSize,
        perThumbnailCompletionHandler: @escaping (
            NSFileProviderItemIdentifier,
            Data?,
            Error?
        ) -> Void,
        completionHandler: @escaping (Error?) -> Void
    ) -> Progress {
        let progress = Progress(totalUnitCount: Int64(itemIdentifiers.count))

        for identifier in itemIdentifiers {
            fetchThumbnail(for: identifier, size: size) { data, error in
                perThumbnailCompletionHandler(identifier, data, error)
                progress.completedUnitCount += 1
            }
        }

        completionHandler(nil)
        return progress
    }

    private func fetchThumbnail(
        for identifier: NSFileProviderItemIdentifier,
        size: CGSize,
        completion: @escaping (Data?, Error?) -> Void
    ) {
        // サムネイルを生成または取得
        completion(nil, nil)
    }
}
```

---

## まとめ：FileProvider Extension

| 機能 | 用途 | 優先度 |
|------|------|--------|
| **NSFileProviderItem** | ファイル/フォルダ表現 | 🔴 必須 |
| **Enumeration** | コンテンツ列挙 | 🔴 必須 |
| **Downloads** | ファイル提供 | 🔴 必須 |
| **Actions** | CRUD操作 | 🟡 推奨 |
| **Thumbnails** | サムネイル表示 | 🟢 参考 |

### 適したユースケース

- クラウドストレージ（Dropbox、Google Drive風）
- 暗号化ボルト（Cryptomator風）
- 特殊ファイルシステム
- リモートファイルアクセス

### 注意点

1. **オフライン対応**: ローカルキャッシュ戦略が重要
2. **同期**: 変更の双方向同期を考慮
3. **パフォーマンス**: 大量ファイルの効率的な処理
4. **エラーハンドリング**: ネットワークエラー対応
