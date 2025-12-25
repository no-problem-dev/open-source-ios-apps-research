# 生体認証 (Face ID / Touch ID) 実装ガイド

**参考プロジェクト**:
- Bitwarden
- 各種パスワード管理アプリ

**実装参考度**: ⭐⭐⭐⭐⭐

---

## なぜ重要か

生体認証はユーザー体験とセキュリティの両立を実現。iOS標準機能として広くサポートされている。

| 学習ポイント | 重要度 | 理由 |
|-------------|--------|------|
| **LAContext** | ⭐⭐⭐⭐⭐ | 生体認証の基本 |
| **ポリシー設定** | ⭐⭐⭐⭐⭐ | 認証方式の制御 |
| **エラーハンドリング** | ⭐⭐⭐⭐☆ | UX向上 |
| **Keychain連携** | ⭐⭐⭐⭐☆ | セキュアデータ保護 |

---

## 1. 基本的な生体認証

### LAContextの使用

```swift
import LocalAuthentication

class BiometricAuthManager {
    enum BiometricType {
        case none
        case faceID
        case touchID
    }

    enum AuthError: Error {
        case biometryNotAvailable
        case biometryNotEnrolled
        case biometryLockout
        case userCancel
        case userFallback
        case systemCancel
        case passcodeNotSet
        case authenticationFailed
        case unknown(Error)
    }

    // 利用可能な生体認証の種類
    var biometricType: BiometricType {
        let context = LAContext()
        _ = context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: nil)

        switch context.biometryType {
        case .faceID:
            return .faceID
        case .touchID:
            return .touchID
        case .opticID:
            return .faceID  // visionOS
        case .none:
            return .none
        @unknown default:
            return .none
        }
    }

    // 生体認証が利用可能かチェック
    func canUseBiometrics() -> Result<Bool, AuthError> {
        let context = LAContext()
        var error: NSError?

        let canEvaluate = context.canEvaluatePolicy(
            .deviceOwnerAuthenticationWithBiometrics,
            error: &error
        )

        if canEvaluate {
            return .success(true)
        }

        guard let laError = error as? LAError else {
            return .failure(.unknown(error ?? NSError()))
        }

        switch laError.code {
        case .biometryNotAvailable:
            return .failure(.biometryNotAvailable)
        case .biometryNotEnrolled:
            return .failure(.biometryNotEnrolled)
        case .biometryLockout:
            return .failure(.biometryLockout)
        case .passcodeNotSet:
            return .failure(.passcodeNotSet)
        default:
            return .failure(.unknown(laError))
        }
    }

    // 生体認証を実行
    func authenticate(reason: String) async -> Result<Bool, AuthError> {
        let context = LAContext()
        context.localizedFallbackTitle = "パスコードを使用"
        context.localizedCancelTitle = "キャンセル"

        do {
            let success = try await context.evaluatePolicy(
                .deviceOwnerAuthenticationWithBiometrics,
                localizedReason: reason
            )
            return .success(success)
        } catch let error as LAError {
            return .failure(mapError(error))
        } catch {
            return .failure(.unknown(error))
        }
    }

    private func mapError(_ error: LAError) -> AuthError {
        switch error.code {
        case .userCancel:
            return .userCancel
        case .userFallback:
            return .userFallback
        case .systemCancel:
            return .systemCancel
        case .biometryLockout:
            return .biometryLockout
        case .authenticationFailed:
            return .authenticationFailed
        default:
            return .unknown(error)
        }
    }
}
```

---

## 2. 認証ポリシー

### ポリシーの種類

```swift
// 生体認証のみ
context.evaluatePolicy(
    .deviceOwnerAuthenticationWithBiometrics,
    localizedReason: reason
)

// 生体認証 or パスコード
context.evaluatePolicy(
    .deviceOwnerAuthentication,
    localizedReason: reason
)

// Apple Watch（watchOS）
context.evaluatePolicy(
    .deviceOwnerAuthenticationWithWatch,
    localizedReason: reason
)

// 生体認証 or Apple Watch
context.evaluatePolicy(
    .deviceOwnerAuthenticationWithBiometricsOrWatch,
    localizedReason: reason
)
```

### 認証の再利用

```swift
class BiometricSession {
    private var context = LAContext()

    // 認証の有効期間を設定
    func authenticate(
        reason: String,
        reuseInterval: TimeInterval = 0
    ) async throws -> Bool {
        // 前回の認証から指定時間内なら再認証不要
        context.touchIDAuthenticationAllowableReuseDuration = reuseInterval

        return try await context.evaluatePolicy(
            .deviceOwnerAuthenticationWithBiometrics,
            localizedReason: reason
        )
    }

    // セッションをリセット
    func invalidate() {
        context.invalidate()
        context = LAContext()
    }
}

// 使用例：5分間は再認証不要
let session = BiometricSession()
try await session.authenticate(
    reason: "アプリのロックを解除",
    reuseInterval: 300
)
```

---

## 3. SwiftUI統合

### ViewModel

```swift
@MainActor
class AuthViewModel: ObservableObject {
    @Published var isAuthenticated = false
    @Published var showError = false
    @Published var errorMessage = ""

    private let biometricManager = BiometricAuthManager()

    var biometricButtonTitle: String {
        switch biometricManager.biometricType {
        case .faceID:
            return "Face IDでロック解除"
        case .touchID:
            return "Touch IDでロック解除"
        case .none:
            return "パスコードでロック解除"
        }
    }

    var biometricIconName: String {
        switch biometricManager.biometricType {
        case .faceID:
            return "faceid"
        case .touchID:
            return "touchid"
        case .none:
            return "lock.fill"
        }
    }

    var canUseBiometrics: Bool {
        if case .success(true) = biometricManager.canUseBiometrics() {
            return true
        }
        return false
    }

    func authenticate() async {
        let result = await biometricManager.authenticate(
            reason: "アプリのロックを解除するには認証が必要です"
        )

        switch result {
        case .success(true):
            isAuthenticated = true
        case .success(false):
            errorMessage = "認証に失敗しました"
            showError = true
        case .failure(let error):
            handleError(error)
        }
    }

    private func handleError(_ error: BiometricAuthManager.AuthError) {
        switch error {
        case .userCancel:
            break  // ユーザーキャンセルは無視
        case .biometryLockout:
            errorMessage = "生体認証がロックされています。パスコードを使用してください。"
            showError = true
        case .biometryNotEnrolled:
            errorMessage = "生体認証が設定されていません。設定アプリから設定してください。"
            showError = true
        case .biometryNotAvailable:
            errorMessage = "このデバイスでは生体認証を利用できません。"
            showError = true
        default:
            errorMessage = "認証エラーが発生しました。"
            showError = true
        }
    }
}
```

### View

```swift
struct LockScreenView: View {
    @StateObject private var viewModel = AuthViewModel()

    var body: some View {
        VStack(spacing: 32) {
            Image(systemName: "lock.fill")
                .font(.system(size: 64))
                .foregroundStyle(.secondary)

            Text("アプリがロックされています")
                .font(.title2)

            if viewModel.canUseBiometrics {
                Button {
                    Task {
                        await viewModel.authenticate()
                    }
                } label: {
                    Label(
                        viewModel.biometricButtonTitle,
                        systemImage: viewModel.biometricIconName
                    )
                }
                .buttonStyle(.borderedProminent)
            }

            Button("パスコードを入力") {
                // パスコード入力画面へ
            }
            .buttonStyle(.bordered)
        }
        .alert("エラー", isPresented: $viewModel.showError) {
            Button("OK") {}
        } message: {
            Text(viewModel.errorMessage)
        }
        .fullScreenCover(isPresented: $viewModel.isAuthenticated) {
            MainAppView()
        }
    }
}
```

---

## 4. Info.plist設定

### 必須の説明文

```xml
<!-- Face ID -->
<key>NSFaceIDUsageDescription</key>
<string>アプリのロックを解除するためにFace IDを使用します</string>
```

---

## 5. Keychain連携

### 生体認証付きKeychain

```swift
class SecureBiometricKeychain {
    func saveWithBiometric(_ data: Data, for key: String) throws {
        var error: Unmanaged<CFError>?

        guard let accessControl = SecAccessControlCreateWithFlags(
            nil,
            kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
            [.biometryCurrentSet, .or, .devicePasscode],
            &error
        ) else {
            throw error!.takeRetainedValue() as Error
        }

        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecValueData as String: data,
            kSecAttrAccessControl as String: accessControl
        ]

        SecItemDelete(query as CFDictionary)
        let status = SecItemAdd(query as CFDictionary, nil)

        guard status == errSecSuccess else {
            throw NSError(domain: NSOSStatusErrorDomain, code: Int(status))
        }
    }

    func readWithBiometric(key: String, reason: String) async throws -> Data {
        let context = LAContext()
        context.localizedReason = reason

        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecReturnData as String: true,
            kSecUseAuthenticationContext as String: context
        ]

        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)

        guard status == errSecSuccess, let data = result as? Data else {
            throw NSError(domain: NSOSStatusErrorDomain, code: Int(status))
        }

        return data
    }
}
```

---

## まとめ：生体認証実装

| 機能 | 用途 | 優先度 |
|------|------|--------|
| **LAContext** | 基本的な生体認証 | 🔴 必須 |
| **ポリシー選択** | 認証方式の制御 | 🔴 必須 |
| **エラーハンドリング** | UX向上 | 🟡 推奨 |
| **Keychain連携** | セキュアデータ保護 | 🟡 推奨 |
| **セッション管理** | 再認証間隔制御 | 🟢 参考 |

### チェックリスト

1. Info.plistに`NSFaceIDUsageDescription`を追加
2. 生体認証の可否をチェック
3. 適切なエラーメッセージを表示
4. フォールバック（パスコード）を用意
5. ロックアウト時の対応を実装
