# FSNotes

**スター**: 7,108 ⭐
**リポジトリ**: https://github.com/glushchenko/fsnotes
**ライセンス**: MIT

---

## 概要

FSNotesは、macOS/iOS向けのプレーンテキストノートアプリ。Notational Velocityにインスパイアされ、高速検索、Markdown対応、Git同期などを特徴とする。プライバシー重視でローカルファースト。

---

## 主要機能

| 機能 | 説明 |
|------|------|
| 高速検索 | インクリメンタル検索 |
| Markdown | プレビュー＆エディット |
| タグ | ノート分類 |
| 暗号化 | AES-256暗号化 |
| iCloud同期 | Appleエコシステム連携 |
| Git同期 | バージョン管理 |

---

## 技術スタック

| 項目 | 技術 |
|------|------|
| 言語 | Swift |
| UI (macOS) | AppKit |
| UI (iOS) | UIKit |
| ストレージ | ファイルシステム |
| 同期 | iCloud Drive, Git |
| 暗号化 | CryptoKit |

---

## アーキテクチャ

```
FSNotes/
├── FSNotes/                 # macOSアプリ
│   ├── View/                # AppKit ビュー
│   ├── Model/               # データモデル
│   └── Business/            # ビジネスロジック
├── FSNotes iOS/             # iOSアプリ
│   └── View/                # UIKit ビュー
└── FSNotesCore/             # 共有コード
    ├── Core/                # コアロジック
    ├── Extensions/          # 拡張
    └── Helpers/             # ユーティリティ
```

---

## 学習価値

### ローカルファースト設計

| 分野 | 学習ポイント |
|------|-------------|
| **ファイルベース** | プレーンテキスト管理 |
| **検索最適化** | 高速インデックス |
| **マルチプラットフォーム** | macOS/iOS共通コード |
| **暗号化** | ローカル暗号化実装 |
| **同期戦略** | iCloud/Git連携 |

### コードパターン

**高速検索**:
```swift
// インクリメンタル検索
class SearchEngine {
    private var index: [String: Set<Note>] = [:]

    func buildIndex(notes: [Note]) {
        for note in notes {
            let words = tokenize(note.content)
            for word in words {
                index[word.lowercased(), default: []].insert(note)
            }
        }
    }

    func search(query: String) -> [Note] {
        let terms = tokenize(query)
        guard let firstTerm = terms.first else { return [] }

        var results = index[firstTerm.lowercased()] ?? []
        for term in terms.dropFirst() {
            results = results.intersection(index[term.lowercased()] ?? [])
        }
        return Array(results).sorted { $0.modifiedAt > $1.modifiedAt }
    }
}
```

**ファイル監視**:
```swift
// ファイルシステム変更監視
class FileWatcher {
    private var source: DispatchSourceFileSystemObject?

    func watch(directory: URL, handler: @escaping () -> Void) {
        let descriptor = open(directory.path, O_EVTONLY)
        source = DispatchSource.makeFileSystemObjectSource(
            fileDescriptor: descriptor,
            eventMask: [.write, .delete, .rename],
            queue: .main
        )

        source?.setEventHandler { handler() }
        source?.setCancelHandler { close(descriptor) }
        source?.resume()
    }
}
```

**Markdownレンダリング**:
```swift
// Markdownプレビュー
class MarkdownRenderer {
    func render(_ markdown: String) -> NSAttributedString {
        let parser = MarkdownParser()
        let document = parser.parse(markdown)

        let renderer = AttributedStringRenderer(theme: .default)
        return renderer.render(document)
    }
}
```

---

## プロジェクト評価

| 観点 | 評価 | 理由 |
|------|------|------|
| **コード品質** | ★★★★☆ | クリーンなSwift |
| **ドキュメント** | ★★★★☆ | ユーザーガイド充実 |
| **アクティビティ** | ★★★★★ | 継続的な開発 |
| **学習価値** | ★★★★★ | ローカルファースト設計 |
| **実用性** | ★★★★★ | 高機能ノートアプリ |

---

## ファイルベース設計

### ノート形式

```
notes/
├── inbox/
│   └── 新規メモ.md
├── projects/
│   ├── プロジェクトA.md
│   └── プロジェクトB.md
└── archive/
    └── 完了タスク.md
```

### メタデータ

```yaml
---
title: ノートタイトル
tags: [swift, ios, development]
created: 2024-01-15
modified: 2024-01-20
encrypted: false
---
```

---

## 暗号化実装

```swift
// AES-256暗号化
class NoteEncryption {
    func encrypt(content: String, password: String) throws -> Data {
        let key = deriveKey(from: password)
        let iv = generateIV()

        let sealed = try AES.GCM.seal(
            content.data(using: .utf8)!,
            using: key,
            nonce: AES.GCM.Nonce(data: iv)
        )

        return iv + sealed.ciphertext + sealed.tag
    }

    func decrypt(data: Data, password: String) throws -> String {
        let key = deriveKey(from: password)
        let iv = data.prefix(12)
        let ciphertext = data.dropFirst(12).dropLast(16)
        let tag = data.suffix(16)

        let box = try AES.GCM.SealedBox(
            nonce: AES.GCM.Nonce(data: iv),
            ciphertext: ciphertext,
            tag: tag
        )

        let decrypted = try AES.GCM.open(box, using: key)
        return String(data: decrypted, encoding: .utf8)!
    }
}
```

---

## 開発者向け情報

### ビルド手順

```bash
git clone https://github.com/glushchenko/fsnotes
cd fsnotes
open FSNotes.xcodeproj
```

### 対応プラットフォーム

- macOS 10.14+
- iOS 13+

---

## まとめ

FSNotesは、ローカルファースト・プライバシー重視のノートアプリの模範。プレーンテキスト、ファイルベース設計、暗号化、マルチプラットフォーム対応など、モダンなノートアプリの要素を網羅。

**推奨対象**:
- ローカルファーストアプリを学びたい開発者
- ファイルベースのデータ管理を実装したい人
- macOS/iOS共通コード設計を参考にしたい人
- プライバシー重視のアプリ設計を学びたい人
