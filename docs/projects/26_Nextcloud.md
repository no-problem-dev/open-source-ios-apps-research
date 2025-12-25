# Nextcloud iOS

**スター**: 2,000+ ⭐
**リポジトリ**: https://github.com/nextcloud/ios
**ライセンス**: GPL-3.0

---

## 概要

Nextcloud iOSは、オープンソースのクラウドストレージNextcloudの公式iOSクライアント。ファイル同期、共有、オフラインアクセス、自動アップロードなど、フルクラウドストレージ機能を提供。

---

## 主要機能

| 機能 | 説明 |
|------|------|
| ファイル管理 | アップロード、ダウンロード、共有 |
| 自動アップロード | 写真/動画自動バックアップ |
| オフライン | ファイルのオフライン保存 |
| E2E暗号化 | エンドツーエンド暗号化 |
| ファイルプロバイダー | Files.app統合 |
| ウィジェット | ホーム画面ウィジェット |

---

## 技術スタック

| 項目 | 技術 |
|------|------|
| 言語 | Swift, Objective-C |
| UI | UIKit + SwiftUI |
| 同期 | バックグラウンドタスク |
| ストレージ | Core Data, Realm |
| 暗号化 | OpenSSL |
| Files.app | FileProvider |

---

## アーキテクチャ

```
nextcloud-ios/
├── iOSClient/               # メインアプリ
│   ├── Main/                # メイン画面
│   ├── Files/               # ファイル管理
│   ├── Media/               # メディアビューア
│   ├── Share/               # 共有機能
│   └── Settings/            # 設定
├── Share Extension/         # 共有拡張
├── File Provider Extension/ # Files.app統合
├── Widget/                  # ウィジェット
└── NCCommunication/         # API ライブラリ
```

---

## 学習価値

### クラウドストレージ実装

| 分野 | 学習ポイント |
|------|-------------|
| **FileProvider** | Files.app統合 |
| **バックグラウンド同期** | BGTaskScheduler |
| **E2E暗号化** | クライアント暗号化 |
| **大規模アプリ** | 企業級設計 |
| **App Extensions** | 複数拡張機能 |

### コードパターン

**FileProvider拡張**:
```swift
// Files.app統合
class FileProviderExtension: NSFileProviderExtension {
    override func item(for identifier: NSFileProviderItemIdentifier) throws -> NSFileProviderItem {
        if identifier == .rootContainer {
            return RootContainerItem()
        }

        guard let metadata = NCManageDatabase.shared.getMetadata(ocId: identifier.rawValue) else {
            throw NSFileProviderError(.noSuchItem)
        }

        return FileProviderItem(metadata: metadata)
    }

    override func urlForItem(withPersistentIdentifier identifier: NSFileProviderItemIdentifier) -> URL? {
        guard let item = try? item(for: identifier) else { return nil }
        let url = NSFileProviderManager.default.documentStorageURL
            .appendingPathComponent(identifier.rawValue)
            .appendingPathComponent(item.filename)
        return url
    }

    override func providePlaceholder(at url: URL) async throws {
        let identifier = try persistentIdentifierForItem(at: url)
        let item = try item(for: identifier)
        try NSFileProviderManager.writePlaceholder(
            at: url,
            withMetadata: item
        )
    }

    override func startProvidingItem(at url: URL) async throws {
        let identifier = try persistentIdentifierForItem(at: url)
        let metadata = NCManageDatabase.shared.getMetadata(ocId: identifier.rawValue)!

        // ダウンロード
        try await NCCommunication.shared.download(
            serverUrlFileName: metadata.serverUrl + "/" + metadata.fileName,
            fileNameLocalPath: url.path
        )
    }
}
```

**自動アップロード**:
```swift
// 写真自動バックアップ
class AutoUploadManager {
    func startAutoUpload() {
        PHPhotoLibrary.requestAuthorization { status in
            guard status == .authorized else { return }
            self.fetchNewAssets()
        }
    }

    private func fetchNewAssets() {
        let lastUploadDate = UserDefaults.standard.object(forKey: "lastAutoUploadDate") as? Date ?? .distantPast

        let options = PHFetchOptions()
        options.predicate = NSPredicate(format: "creationDate > %@", lastUploadDate as NSDate)
        options.sortDescriptors = [NSSortDescriptor(key: "creationDate", ascending: true)]

        let assets = PHAsset.fetchAssets(with: options)

        assets.enumerateObjects { asset, _, _ in
            self.uploadAsset(asset)
        }
    }

    private func uploadAsset(_ asset: PHAsset) {
        let options = PHImageRequestOptions()
        options.isNetworkAccessAllowed = true
        options.deliveryMode = .highQualityFormat

        PHImageManager.default().requestImageDataAndOrientation(for: asset, options: options) { data, _, _, _ in
            guard let data = data else { return }

            Task {
                try await NCCommunication.shared.upload(
                    serverUrlFileName: self.autoUploadPath + "/" + asset.localIdentifier,
                    data: data
                )
            }
        }
    }
}
```

**E2E暗号化**:
```swift
// エンドツーエンド暗号化
class NCEndToEndEncryption {
    private let privateKey: SecKey
    private let publicKey: SecKey

    func encryptFile(at url: URL, metadata: tableMetadata) throws -> Data {
        // ファイルキー生成
        let fileKey = generateRandomKey(length: 32)
        let iv = generateRandomKey(length: 16)

        // AES暗号化
        let encryptedData = try aesEncrypt(
            data: try Data(contentsOf: url),
            key: fileKey,
            iv: iv
        )

        // ファイルキーを公開鍵で暗号化
        let encryptedKey = try rsaEncrypt(data: fileKey, publicKey: publicKey)

        // メタデータ保存
        let e2eMetadata = E2EMetadata(
            encryptedKey: encryptedKey,
            iv: iv,
            authenticationTag: encryptedData.tag
        )
        try saveE2EMetadata(e2eMetadata, for: metadata)

        return encryptedData.ciphertext
    }

    func decryptFile(data: Data, metadata: tableMetadata) throws -> Data {
        let e2eMetadata = try loadE2EMetadata(for: metadata)

        // ファイルキーを復号
        let fileKey = try rsaDecrypt(data: e2eMetadata.encryptedKey, privateKey: privateKey)

        // AES復号
        return try aesDecrypt(
            data: data,
            key: fileKey,
            iv: e2eMetadata.iv,
            tag: e2eMetadata.authenticationTag
        )
    }
}
```

**バックグラウンドタスク**:
```swift
// バックグラウンド同期
class BackgroundSyncManager {
    func registerBackgroundTasks() {
        BGTaskScheduler.shared.register(
            forTaskWithIdentifier: "com.nextcloud.sync",
            using: nil
        ) { task in
            self.handleSyncTask(task as! BGProcessingTask)
        }
    }

    func scheduleSync() {
        let request = BGProcessingTaskRequest(identifier: "com.nextcloud.sync")
        request.requiresNetworkConnectivity = true
        request.requiresExternalPower = false

        try? BGTaskScheduler.shared.submit(request)
    }

    private func handleSyncTask(_ task: BGProcessingTask) {
        let syncOperation = SyncOperation()

        task.expirationHandler = {
            syncOperation.cancel()
        }

        syncOperation.completionBlock = {
            task.setTaskCompleted(success: !syncOperation.isCancelled)
            self.scheduleSync() // 次回スケジュール
        }

        OperationQueue.main.addOperation(syncOperation)
    }
}
```

---

## プロジェクト評価

| 観点 | 評価 | 理由 |
|------|------|------|
| **コード品質** | ★★★★☆ | 大規模だが整理されている |
| **ドキュメント** | ★★★★☆ | 貢献ガイド充実 |
| **アクティビティ** | ★★★★★ | 継続的な開発 |
| **学習価値** | ★★★★★ | クラウドストレージの教科書 |
| **実用性** | ★★★★★ | 本番クライアント |

---

## App Extensions

### 含まれる拡張機能

| 拡張 | 機能 |
|-----|------|
| Share Extension | 他アプリからファイル共有 |
| File Provider | Files.app統合 |
| Widget | ホーム画面ウィジェット |
| Notification Service | リッチ通知 |

---

## 開発者向け情報

### ビルド手順

```bash
git clone https://github.com/nextcloud/ios
cd ios
pod install
open Nextcloud.xcworkspace
```

### 必要要件

- Xcode 15+
- CocoaPods
- Nextcloudサーバー（テスト用）

---

## まとめ

Nextcloud iOSは、クラウドストレージアプリ開発の包括的な参考資料。FileProvider、バックグラウンド同期、E2E暗号化など、高度な機能を網羅。大規模オープンソースプロジェクトの運営も参考になる。

**推奨対象**:
- クラウドストレージアプリを開発したい人
- FileProviderを実装したい開発者
- バックグラウンド同期を学びたい人
- E2E暗号化を実装したい人
