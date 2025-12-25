# Keychain セキュアストレージ実装

**参考プロジェクト**:
- Bitwarden: https://github.com/nickolashkraus/bitwarden-mobile
- ProtonMail: セキュリティ重視の実装

**実装参考度**: ⭐⭐⭐⭐⭐

---

## なぜ重要か

Keychainは**機密データの安全な保存場所**。パスワード、トークン、暗号鍵などを保護する。

| 学習ポイント | 重要度 | 理由 |
|-------------|--------|------|
| **基本操作** | ⭐⭐⭐⭐⭐ | CRUD操作 |
| **アクセス制御** | ⭐⭐⭐⭐⭐ | セキュリティレベル |
| **iCloud同期** | ⭐⭐⭐⭐☆ | デバイス間共有 |
| **App Group** | ⭐⭐⭐⭐☆ | アプリ間共有 |

---

## 1. 基本的なKeychain操作

### Keychainヘルパー

```swift
import Security
import Foundation

enum KeychainError: Error {
    case duplicateItem
    case itemNotFound
    case unexpectedStatus(OSStatus)
    case invalidData
}

class KeychainManager {
    static let shared = KeychainManager()

    private let service: String

    init(service: String = Bundle.main.bundleIdentifier ?? "com.app.keychain") {
        self.service = service
    }

    // MARK: - Save

    func save(_ data: Data, for key: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: key,
            kSecValueData as String: data
        ]

        let status = SecItemAdd(query as CFDictionary, nil)

        switch status {
        case errSecSuccess:
            return
        case errSecDuplicateItem:
            // 既存のアイテムを更新
            try update(data, for: key)
        default:
            throw KeychainError.unexpectedStatus(status)
        }
    }

    func save(_ string: String, for key: String) throws {
        guard let data = string.data(using: .utf8) else {
            throw KeychainError.invalidData
        }
        try save(data, for: key)
    }

    // MARK: - Read

    func read(key: String) throws -> Data {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: key,
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne
        ]

        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)

        switch status {
        case errSecSuccess:
            guard let data = result as? Data else {
                throw KeychainError.invalidData
            }
            return data
        case errSecItemNotFound:
            throw KeychainError.itemNotFound
        default:
            throw KeychainError.unexpectedStatus(status)
        }
    }

    func readString(key: String) throws -> String {
        let data = try read(key: key)
        guard let string = String(data: data, encoding: .utf8) else {
            throw KeychainError.invalidData
        }
        return string
    }

    // MARK: - Update

    func update(_ data: Data, for key: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: key
        ]

        let attributes: [String: Any] = [
            kSecValueData as String: data
        ]

        let status = SecItemUpdate(query as CFDictionary, attributes as CFDictionary)

        guard status == errSecSuccess else {
            throw KeychainError.unexpectedStatus(status)
        }
    }

    // MARK: - Delete

    func delete(key: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: key
        ]

        let status = SecItemDelete(query as CFDictionary)

        guard status == errSecSuccess || status == errSecItemNotFound else {
            throw KeychainError.unexpectedStatus(status)
        }
    }

    func deleteAll() throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service
        ]

        let status = SecItemDelete(query as CFDictionary)

        guard status == errSecSuccess || status == errSecItemNotFound else {
            throw KeychainError.unexpectedStatus(status)
        }
    }
}
```

---

## 2. アクセス制御

### アクセシビリティ設定

```swift
extension KeychainManager {
    func saveWithAccessibility(
        _ data: Data,
        for key: String,
        accessibility: CFString
    ) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: key,
            kSecValueData as String: data,
            kSecAttrAccessible as String: accessibility
        ]

        let status = SecItemAdd(query as CFDictionary, nil)

        guard status == errSecSuccess else {
            throw KeychainError.unexpectedStatus(status)
        }
    }
}

// アクセシビリティオプション
enum KeychainAccessibility {
    // デバイスがロック解除されている間のみアクセス可能
    static let whenUnlocked = kSecAttrAccessibleWhenUnlocked

    // デバイスがロック解除されている間のみ（iCloud同期なし）
    static let whenUnlockedThisDeviceOnly = kSecAttrAccessibleWhenUnlockedThisDeviceOnly

    // 最初のロック解除後はいつでもアクセス可能
    static let afterFirstUnlock = kSecAttrAccessibleAfterFirstUnlock

    // 最初のロック解除後（iCloud同期なし）
    static let afterFirstUnlockThisDeviceOnly = kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly

    // パスコード設定時のみ（パスコード削除でデータも削除）
    static let whenPasscodeSet = kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly
}

// 使用例
try keychainManager.saveWithAccessibility(
    tokenData,
    for: "auth_token",
    accessibility: KeychainAccessibility.afterFirstUnlockThisDeviceOnly
)
```

### 生体認証付きアクセス

```swift
import LocalAuthentication

extension KeychainManager {
    func saveWithBiometrics(
        _ data: Data,
        for key: String,
        reason: String
    ) throws {
        var error: Unmanaged<CFError>?

        let accessControl = SecAccessControlCreateWithFlags(
            nil,
            kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
            .biometryCurrentSet,  // 現在の生体認証のみ
            &error
        )

        if let error = error?.takeRetainedValue() {
            throw error as Error
        }

        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: key,
            kSecValueData as String: data,
            kSecAttrAccessControl as String: accessControl as Any
        ]

        let status = SecItemAdd(query as CFDictionary, nil)

        guard status == errSecSuccess else {
            throw KeychainError.unexpectedStatus(status)
        }
    }

    func readWithBiometrics(key: String, reason: String) throws -> Data {
        let context = LAContext()
        context.localizedReason = reason

        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: key,
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne,
            kSecUseAuthenticationContext as String: context
        ]

        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)

        switch status {
        case errSecSuccess:
            guard let data = result as? Data else {
                throw KeychainError.invalidData
            }
            return data
        case errSecItemNotFound:
            throw KeychainError.itemNotFound
        case errSecUserCanceled:
            throw KeychainError.unexpectedStatus(status)
        default:
            throw KeychainError.unexpectedStatus(status)
        }
    }
}
```

---

## 3. iCloud Keychain同期

```swift
extension KeychainManager {
    func saveToiCloud(_ data: Data, for key: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: key,
            kSecValueData as String: data,
            kSecAttrSynchronizable as String: true,  // iCloud同期を有効化
            kSecAttrAccessible as String: kSecAttrAccessibleAfterFirstUnlock
        ]

        let status = SecItemAdd(query as CFDictionary, nil)

        guard status == errSecSuccess else {
            throw KeychainError.unexpectedStatus(status)
        }
    }
}
```

---

## 4. App Group共有

```swift
class SharedKeychainManager {
    private let accessGroup: String

    init(accessGroup: String) {
        // 形式: TEAM_ID.group.com.example.shared
        self.accessGroup = accessGroup
    }

    func save(_ data: Data, for key: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecValueData as String: data,
            kSecAttrAccessGroup as String: accessGroup
        ]

        let status = SecItemAdd(query as CFDictionary, nil)

        guard status == errSecSuccess else {
            throw KeychainError.unexpectedStatus(status)
        }
    }

    func read(key: String) throws -> Data {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne,
            kSecAttrAccessGroup as String: accessGroup
        ]

        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)

        guard status == errSecSuccess, let data = result as? Data else {
            throw KeychainError.itemNotFound
        }

        return data
    }
}
```

---

## 5. Codable対応

```swift
extension KeychainManager {
    func save<T: Encodable>(_ item: T, for key: String) throws {
        let data = try JSONEncoder().encode(item)
        try save(data, for: key)
    }

    func read<T: Decodable>(key: String, as type: T.Type) throws -> T {
        let data = try read(key: key)
        return try JSONDecoder().decode(type, from: data)
    }
}

// 使用例
struct Credentials: Codable {
    let username: String
    let password: String
    let token: String
}

// 保存
let credentials = Credentials(
    username: "user@example.com",
    password: "secret",
    token: "jwt_token"
)
try keychainManager.save(credentials, for: "user_credentials")

// 読み込み
let savedCredentials = try keychainManager.read(
    key: "user_credentials",
    as: Credentials.self
)
```

---

## まとめ：Keychain実装

| 機能 | 用途 | 優先度 |
|------|------|--------|
| **基本CRUD** | トークン、パスワード保存 | 🔴 必須 |
| **アクセシビリティ** | セキュリティレベル設定 | 🔴 必須 |
| **生体認証** | 高セキュリティアイテム | 🟡 推奨 |
| **iCloud同期** | マルチデバイス対応 | 🟢 参考 |
| **App Group** | アプリ間共有 | 🟢 参考 |

### ベストプラクティス

1. **UserDefaults vs Keychain**
   - UserDefaults: 設定、プリファレンス（非機密）
   - Keychain: パスワード、トークン、秘密鍵

2. **アクセシビリティ選択**
   - 認証トークン → `afterFirstUnlockThisDeviceOnly`
   - 生体認証必須データ → `whenPasscodeSet` + 生体認証

3. **エラーハンドリング**
   - `errSecItemNotFound`: 初回アクセス時の正常ケース
   - `errSecDuplicateItem`: 更新で対応
