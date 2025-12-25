# Bitwarden iOS

**スター**: 6,000+ ⭐（メインリポジトリ）
**リポジトリ**: https://github.com/bitwarden/ios
**ライセンス**: GPL-3.0

---

## 概要

Bitwardenは、オープンソースのパスワードマネージャー。iOS版はネイティブSwiftで実装され、セキュリティとユーザビリティを両立。AutoFill、生体認証、セキュアなクラウド同期を提供。

---

## 主要機能

| 機能 | 説明 |
|------|------|
| パスワード管理 | 保存、生成、自動入力 |
| AutoFill | iOS AutoFill拡張 |
| 生体認証 | Face ID / Touch ID |
| 同期 | クラウド同期 |
| 二要素認証 | TOTP対応 |
| セキュアノート | 暗号化メモ |

---

## 技術スタック

| 項目 | 技術 |
|------|------|
| 言語 | Swift |
| UI | UIKit + SwiftUI |
| アーキテクチャ | MVVM + Coordinator |
| 暗号化 | CryptoKit, CommonCrypto |
| ストレージ | Core Data, Keychain |
| AutoFill | Credential Provider |

---

## アーキテクチャ

```
bitwarden-ios/
├── Bitwarden/               # メインアプリ
│   ├── Application/         # アプリ起動
│   ├── UI/                  # UIコンポーネント
│   │   ├── Vault/           # ボールト画面
│   │   ├── Auth/            # 認証画面
│   │   └── Settings/        # 設定画面
│   ├── Core/                # ビジネスロジック
│   │   ├── Services/        # サービス層
│   │   └── Repositories/    # データアクセス
│   └── Crypto/              # 暗号化
├── AutofillExtension/       # AutoFill拡張
└── ShareExtension/          # 共有拡張
```

---

## 学習価値

### セキュリティ実装

| 分野 | 学習ポイント |
|------|-------------|
| **暗号化** | AES-256, PBKDF2 |
| **Keychain** | セキュアストレージ |
| **生体認証** | LocalAuthentication |
| **AutoFill** | Credential Provider |
| **セキュアコーディング** | ゼロ知識設計 |

### コードパターン

**マスターパスワード導出**:
```swift
// PBKDF2 でマスターキー導出
class CryptoService {
    func makeMasterKey(
        password: String,
        salt: String,
        iterations: Int = 600000
    ) throws -> SymmetricKey {
        let passwordData = password.data(using: .utf8)!
        let saltData = salt.data(using: .utf8)!

        var derivedKey = [UInt8](repeating: 0, count: 32)

        let status = CCKeyDerivationPBKDF(
            CCPBKDFAlgorithm(kCCPBKDF2),
            password,
            passwordData.count,
            [UInt8](saltData),
            saltData.count,
            CCPseudoRandomAlgorithm(kCCPRFHmacAlgSHA256),
            UInt32(iterations),
            &derivedKey,
            derivedKey.count
        )

        guard status == kCCSuccess else {
            throw CryptoError.keyDerivationFailed
        }

        return SymmetricKey(data: Data(derivedKey))
    }
}
```

**AES暗号化**:
```swift
// AES-256-CBC 暗号化
class EncryptionService {
    func encrypt(data: Data, key: SymmetricKey) throws -> EncryptedData {
        let iv = generateIV()

        let sealed = try AES.GCM.seal(
            data,
            using: key,
            nonce: AES.GCM.Nonce(data: iv)
        )

        return EncryptedData(
            iv: iv,
            ciphertext: sealed.ciphertext,
            tag: sealed.tag
        )
    }

    func decrypt(encrypted: EncryptedData, key: SymmetricKey) throws -> Data {
        let box = try AES.GCM.SealedBox(
            nonce: AES.GCM.Nonce(data: encrypted.iv),
            ciphertext: encrypted.ciphertext,
            tag: encrypted.tag
        )

        return try AES.GCM.open(box, using: key)
    }
}
```

**AutoFill Credential Provider**:
```swift
// AutoFill拡張
class CredentialProviderViewController: ASCredentialProviderViewController {
    override func prepareCredentialList(for serviceIdentifiers: [ASCredentialServiceIdentifier]) {
        // サービス識別子でフィルタリング
        let matchingCredentials = credentialStore.credentials(for: serviceIdentifiers)

        if matchingCredentials.isEmpty {
            // 認証情報がない場合
            extensionContext.cancelRequest(withError: NSError(
                domain: ASExtensionErrorDomain,
                code: ASExtensionError.credentialIdentityNotFound.rawValue
            ))
        } else {
            // 認証情報リスト表示
            showCredentialList(matchingCredentials)
        }
    }

    func selectCredential(_ credential: Credential) {
        let passwordCredential = ASPasswordCredential(
            user: credential.username,
            password: credential.password
        )
        extensionContext.completeRequest(
            withSelectedCredential: passwordCredential
        )
    }
}
```

**生体認証**:
```swift
// Face ID / Touch ID
class BiometricService {
    private let context = LAContext()

    var biometryType: LABiometryType {
        context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: nil)
        return context.biometryType
    }

    func authenticate(reason: String) async throws -> Bool {
        return try await withCheckedThrowingContinuation { continuation in
            context.evaluatePolicy(
                .deviceOwnerAuthenticationWithBiometrics,
                localizedReason: reason
            ) { success, error in
                if let error = error {
                    continuation.resume(throwing: error)
                } else {
                    continuation.resume(returning: success)
                }
            }
        }
    }
}
```

---

## プロジェクト評価

| 観点 | 評価 | 理由 |
|------|------|------|
| **コード品質** | ★★★★★ | セキュリティ重視 |
| **ドキュメント** | ★★★★☆ | セキュリティ監査公開 |
| **アクティビティ** | ★★★★★ | 継続的な開発 |
| **学習価値** | ★★★★★ | セキュリティの教科書 |
| **実用性** | ★★★★★ | 商用パスワードマネージャー |

---

## Keychain統合

```swift
// セキュアストレージ
class KeychainService {
    func save(key: String, data: Data) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecValueData as String: data,
            kSecAttrAccessible as String: kSecAttrAccessibleWhenUnlockedThisDeviceOnly
        ]

        let status = SecItemAdd(query as CFDictionary, nil)
        guard status == errSecSuccess else {
            throw KeychainError.saveFailed(status)
        }
    }

    func load(key: String) throws -> Data? {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne
        ]

        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)

        guard status == errSecSuccess else {
            if status == errSecItemNotFound { return nil }
            throw KeychainError.loadFailed(status)
        }

        return result as? Data
    }
}
```

---

## ゼロ知識設計

### 原則

| 原則 | 実装 |
|------|------|
| クライアント暗号化 | データはデバイスで暗号化 |
| サーバー不可読 | サーバーはデータを読めない |
| マスターパスワード | ユーザーのみ知る |
| 鍵導出 | PBKDF2で鍵を導出 |

---

## 開発者向け情報

### ビルド手順

```bash
git clone https://github.com/bitwarden/ios
cd ios
# 依存関係インストール
./bootstrap.sh
open Bitwarden.xcodeproj
```

### セキュリティ監査

Bitwardenは定期的に第三者セキュリティ監査を受け、結果を公開。

---

## まとめ

Bitwarden iOSは、セキュリティ重視のアプリ開発を学ぶ最良の教材。暗号化、Keychain、生体認証、AutoFill拡張など、セキュリティ機能を網羅。ゼロ知識設計の実践例としても価値が高い。

**推奨対象**:
- セキュリティ実装を学びたい開発者
- 暗号化・Keychain を実装したい人
- AutoFill拡張を開発したい人
- パスワードマネージャーを参考にしたい人
