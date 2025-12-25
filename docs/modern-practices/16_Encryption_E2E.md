# 暗号化 & E2E暗号化 実装ガイド

**参考プロジェクト**:
- Signal: https://github.com/nickolashkraus/Signal-iOS
- ProtonMail
- Element (Matrix)

**実装参考度**: ⭐⭐⭐⭐⭐

---

## なぜ重要か

**CryptoKit**はApple純正の暗号化フレームワーク。対称暗号、非対称暗号、ハッシュ、署名を提供。

| 学習ポイント | 重要度 | 理由 |
|-------------|--------|------|
| **AES-GCM** | ⭐⭐⭐⭐⭐ | 対称暗号の標準 |
| **ECDH** | ⭐⭐⭐⭐⭐ | 鍵交換 |
| **HKDF** | ⭐⭐⭐⭐☆ | 鍵導出 |
| **SHA256** | ⭐⭐⭐⭐☆ | ハッシュ |
| **P256署名** | ⭐⭐⭐☆☆ | デジタル署名 |

---

## 1. 対称暗号（AES-GCM）

### 基本的な暗号化/復号

```swift
import CryptoKit
import Foundation

class SymmetricEncryption {
    // ランダムな鍵を生成
    static func generateKey() -> SymmetricKey {
        SymmetricKey(size: .bits256)
    }

    // 暗号化
    static func encrypt(data: Data, key: SymmetricKey) throws -> Data {
        let sealedBox = try AES.GCM.seal(data, using: key)
        guard let combined = sealedBox.combined else {
            throw EncryptionError.encryptionFailed
        }
        return combined
    }

    // 復号
    static func decrypt(data: Data, key: SymmetricKey) throws -> Data {
        let sealedBox = try AES.GCM.SealedBox(combined: data)
        return try AES.GCM.open(sealedBox, using: key)
    }

    // 文字列を暗号化
    static func encryptString(_ string: String, key: SymmetricKey) throws -> Data {
        guard let data = string.data(using: .utf8) else {
            throw EncryptionError.invalidInput
        }
        return try encrypt(data: data, key: key)
    }

    // 暗号化データを文字列に復号
    static func decryptToString(data: Data, key: SymmetricKey) throws -> String {
        let decryptedData = try decrypt(data: data, key: key)
        guard let string = String(data: decryptedData, encoding: .utf8) else {
            throw EncryptionError.invalidOutput
        }
        return string
    }
}

enum EncryptionError: Error {
    case encryptionFailed
    case decryptionFailed
    case invalidInput
    case invalidOutput
    case keyDerivationFailed
}
```

### パスワードベースの暗号化

```swift
class PasswordBasedEncryption {
    // パスワードから鍵を導出
    static func deriveKey(
        password: String,
        salt: Data
    ) throws -> SymmetricKey {
        guard let passwordData = password.data(using: .utf8) else {
            throw EncryptionError.invalidInput
        }

        // HKDF で鍵を導出
        let derivedKey = HKDF<SHA256>.deriveKey(
            inputKeyMaterial: SymmetricKey(data: passwordData),
            salt: salt,
            info: "encryption".data(using: .utf8)!,
            outputByteCount: 32
        )

        return derivedKey
    }

    // ソルトを生成
    static func generateSalt() -> Data {
        var salt = Data(count: 32)
        _ = salt.withUnsafeMutableBytes { ptr in
            SecRandomCopyBytes(kSecRandomDefault, 32, ptr.baseAddress!)
        }
        return salt
    }

    // パスワードで暗号化
    static func encrypt(
        data: Data,
        password: String
    ) throws -> (encryptedData: Data, salt: Data) {
        let salt = generateSalt()
        let key = try deriveKey(password: password, salt: salt)
        let encryptedData = try SymmetricEncryption.encrypt(data: data, key: key)
        return (encryptedData, salt)
    }

    // パスワードで復号
    static func decrypt(
        data: Data,
        password: String,
        salt: Data
    ) throws -> Data {
        let key = try deriveKey(password: password, salt: salt)
        return try SymmetricEncryption.decrypt(data: data, key: key)
    }
}
```

---

## 2. 非対称暗号（ECDH鍵交換）

### 鍵ペアの生成と交換

```swift
class ECDHKeyExchange {
    // 鍵ペアを生成
    static func generateKeyPair() -> P256.KeyAgreement.PrivateKey {
        P256.KeyAgreement.PrivateKey()
    }

    // 公開鍵を取得
    static func getPublicKey(
        from privateKey: P256.KeyAgreement.PrivateKey
    ) -> P256.KeyAgreement.PublicKey {
        privateKey.publicKey
    }

    // 共有秘密を導出
    static func deriveSharedSecret(
        myPrivateKey: P256.KeyAgreement.PrivateKey,
        theirPublicKey: P256.KeyAgreement.PublicKey
    ) throws -> SharedSecret {
        try myPrivateKey.sharedSecretFromKeyAgreement(with: theirPublicKey)
    }

    // 共有秘密から暗号化鍵を導出
    static func deriveSymmetricKey(
        sharedSecret: SharedSecret,
        salt: Data
    ) -> SymmetricKey {
        sharedSecret.hkdfDerivedSymmetricKey(
            using: SHA256.self,
            salt: salt,
            sharedInfo: "encryption".data(using: .utf8)!,
            outputByteCount: 32
        )
    }
}

// 使用例
class E2EEncryption {
    let myPrivateKey: P256.KeyAgreement.PrivateKey
    let myPublicKey: P256.KeyAgreement.PublicKey

    init() {
        self.myPrivateKey = ECDHKeyExchange.generateKeyPair()
        self.myPublicKey = ECDHKeyExchange.getPublicKey(from: myPrivateKey)
    }

    // 相手の公開鍵を受け取って暗号化
    func encrypt(
        message: Data,
        recipientPublicKey: P256.KeyAgreement.PublicKey
    ) throws -> (encryptedData: Data, salt: Data) {
        let sharedSecret = try ECDHKeyExchange.deriveSharedSecret(
            myPrivateKey: myPrivateKey,
            theirPublicKey: recipientPublicKey
        )

        let salt = PasswordBasedEncryption.generateSalt()
        let key = ECDHKeyExchange.deriveSymmetricKey(
            sharedSecret: sharedSecret,
            salt: salt
        )

        let encryptedData = try SymmetricEncryption.encrypt(data: message, key: key)
        return (encryptedData, salt)
    }

    // 暗号化されたメッセージを復号
    func decrypt(
        encryptedData: Data,
        salt: Data,
        senderPublicKey: P256.KeyAgreement.PublicKey
    ) throws -> Data {
        let sharedSecret = try ECDHKeyExchange.deriveSharedSecret(
            myPrivateKey: myPrivateKey,
            theirPublicKey: senderPublicKey
        )

        let key = ECDHKeyExchange.deriveSymmetricKey(
            sharedSecret: sharedSecret,
            salt: salt
        )

        return try SymmetricEncryption.decrypt(data: encryptedData, key: key)
    }
}
```

---

## 3. ハッシュと署名

### ハッシュ

```swift
import CryptoKit

class HashingService {
    // SHA256ハッシュ
    static func sha256(_ data: Data) -> Data {
        Data(SHA256.hash(data: data))
    }

    static func sha256(_ string: String) -> String {
        guard let data = string.data(using: .utf8) else { return "" }
        let hash = SHA256.hash(data: data)
        return hash.compactMap { String(format: "%02x", $0) }.joined()
    }

    // HMAC
    static func hmac(
        data: Data,
        key: SymmetricKey
    ) -> Data {
        let authCode = HMAC<SHA256>.authenticationCode(for: data, using: key)
        return Data(authCode)
    }

    // HMAC検証
    static func verifyHMAC(
        data: Data,
        authCode: Data,
        key: SymmetricKey
    ) -> Bool {
        HMAC<SHA256>.isValidAuthenticationCode(authCode, authenticating: data, using: key)
    }
}
```

### デジタル署名

```swift
class DigitalSignature {
    // 署名鍵ペアを生成
    static func generateSigningKeyPair() -> P256.Signing.PrivateKey {
        P256.Signing.PrivateKey()
    }

    // 署名
    static func sign(data: Data, privateKey: P256.Signing.PrivateKey) throws -> Data {
        let signature = try privateKey.signature(for: data)
        return signature.rawRepresentation
    }

    // 署名検証
    static func verify(
        data: Data,
        signature: Data,
        publicKey: P256.Signing.PublicKey
    ) -> Bool {
        guard let ecdsaSignature = try? P256.Signing.ECDSASignature(rawRepresentation: signature) else {
            return false
        }
        return publicKey.isValidSignature(ecdsaSignature, for: data)
    }
}
```

---

## 4. 鍵の永続化

### Keychainへの保存

```swift
class SecureKeyStorage {
    private let keychain = KeychainManager.shared

    // 秘密鍵を保存
    func savePrivateKey(
        _ privateKey: P256.KeyAgreement.PrivateKey,
        identifier: String
    ) throws {
        let keyData = privateKey.rawRepresentation
        try keychain.save(keyData, for: "privateKey_\(identifier)")
    }

    // 秘密鍵を読み込み
    func loadPrivateKey(
        identifier: String
    ) throws -> P256.KeyAgreement.PrivateKey {
        let keyData = try keychain.read(key: "privateKey_\(identifier)")
        return try P256.KeyAgreement.PrivateKey(rawRepresentation: keyData)
    }

    // 公開鍵をData形式で取得（共有用）
    func exportPublicKey(
        _ publicKey: P256.KeyAgreement.PublicKey
    ) -> Data {
        publicKey.rawRepresentation
    }

    // Data形式から公開鍵を復元
    func importPublicKey(
        _ data: Data
    ) throws -> P256.KeyAgreement.PublicKey {
        try P256.KeyAgreement.PublicKey(rawRepresentation: data)
    }
}
```

---

## 5. 実践的なE2Eメッセージング

### メッセージ暗号化サービス

```swift
struct EncryptedMessage: Codable {
    let ciphertext: Data
    let salt: Data
    let senderPublicKey: Data
}

class SecureMessagingService {
    private let keyStorage = SecureKeyStorage()
    private var myPrivateKey: P256.KeyAgreement.PrivateKey!
    private var myPublicKey: P256.KeyAgreement.PublicKey!

    init() throws {
        // 鍵ペアをロードまたは生成
        do {
            myPrivateKey = try keyStorage.loadPrivateKey(identifier: "messaging")
        } catch {
            myPrivateKey = ECDHKeyExchange.generateKeyPair()
            try keyStorage.savePrivateKey(myPrivateKey, identifier: "messaging")
        }
        myPublicKey = myPrivateKey.publicKey
    }

    // 自分の公開鍵を取得（相手に送信）
    func getMyPublicKeyData() -> Data {
        keyStorage.exportPublicKey(myPublicKey)
    }

    // メッセージを暗号化
    func encryptMessage(
        _ message: String,
        recipientPublicKeyData: Data
    ) throws -> EncryptedMessage {
        let recipientPublicKey = try keyStorage.importPublicKey(recipientPublicKeyData)

        let sharedSecret = try myPrivateKey.sharedSecretFromKeyAgreement(
            with: recipientPublicKey
        )

        let salt = PasswordBasedEncryption.generateSalt()
        let key = sharedSecret.hkdfDerivedSymmetricKey(
            using: SHA256.self,
            salt: salt,
            sharedInfo: Data(),
            outputByteCount: 32
        )

        let messageData = message.data(using: .utf8)!
        let ciphertext = try SymmetricEncryption.encrypt(data: messageData, key: key)

        return EncryptedMessage(
            ciphertext: ciphertext,
            salt: salt,
            senderPublicKey: keyStorage.exportPublicKey(myPublicKey)
        )
    }

    // メッセージを復号
    func decryptMessage(_ encryptedMessage: EncryptedMessage) throws -> String {
        let senderPublicKey = try keyStorage.importPublicKey(encryptedMessage.senderPublicKey)

        let sharedSecret = try myPrivateKey.sharedSecretFromKeyAgreement(
            with: senderPublicKey
        )

        let key = sharedSecret.hkdfDerivedSymmetricKey(
            using: SHA256.self,
            salt: encryptedMessage.salt,
            sharedInfo: Data(),
            outputByteCount: 32
        )

        let decryptedData = try SymmetricEncryption.decrypt(
            data: encryptedMessage.ciphertext,
            key: key
        )

        return String(data: decryptedData, encoding: .utf8)!
    }
}
```

---

## まとめ：暗号化実装

| 機能 | 用途 | 優先度 |
|------|------|--------|
| **AES-GCM** | ローカルデータ暗号化 | 🔴 必須 |
| **ECDH** | E2E暗号化の鍵交換 | 🔴 必須（E2E時） |
| **HKDF** | 鍵導出 | 🟡 推奨 |
| **SHA256** | データ整合性検証 | 🟡 推奨 |
| **デジタル署名** | 認証・改ざん検知 | 🟢 参考 |

### セキュリティベストプラクティス

1. **鍵管理**: 秘密鍵はKeychainに保存
2. **乱数生成**: SecRandomCopyBytesを使用
3. **メモリ管理**: 機密データは使用後にゼロクリア
4. **プロトコル**: 既存の暗号化プロトコル（Signal Protocol等）を参考に
