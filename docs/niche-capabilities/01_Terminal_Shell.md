# iOSでターミナル/シェル実装

**驚きポイント**: iOSのサンドボックス内でLinuxシェルやSSHクライアントが動作する

## 参考プロジェクト

| プロジェクト | Stars | 特徴 |
|-------------|-------|------|
| [iSH](https://github.com/ish-app/ish) | 18,958 | x86エミュレーションでLinux |
| [a-shell](https://github.com/holzschu/a-shell) | 3,396 | ネイティブUnixコマンド |
| [SwiftTermApp](https://github.com/migueldeicaza/SwiftTermApp) | 345 | SwiftUIでSSH |
| [MobileTerminal](https://github.com/steventroughtonsmith/MobileTerminal) | 197 | サンドボックス内PoC |

---

## 1. iSH - Linuxエミュレーション

x86命令をユーザー空間でエミュレートしてAlpine Linuxを実行。

### アーキテクチャ

```
┌─────────────────────────────────┐
│         Alpine Linux            │
├─────────────────────────────────┤
│      x86 User Emulation         │
├─────────────────────────────────┤
│    iOS App (Objective-C/C)      │
└─────────────────────────────────┘
```

### 学べる技術

- **CPU Emulation**: x86命令のソフトウェア実装
- **System Call Translation**: Linux syscall → iOS/Darwin変換
- **File System Abstraction**: iOS Documents → Linux FS マッピング
- **PTY (Pseudo Terminal)**: ターミナル入出力の仮想化

### なぜApp Storeで許可されるか

- JITコンパイルではなくインタープリタ方式
- 任意コード実行ではなく限定的なLinux環境
- ユーザーが明示的にインストールしたパッケージのみ

---

## 2. a-shell - ネイティブUnixツール

iOS/iPadOS向けにコンパイルされたUnixコマンド群。

### 含まれるツール

```bash
# テキスト処理
vim, python, lua, tex

# ファイル操作
ls, cat, grep, awk, sed

# ネットワーク
curl, ssh, sftp

# 開発
clang, wasm, javascript
```

### iOS固有の工夫

```swift
// Files.appとの統合
let documentPicker = UIDocumentPickerViewController(
    forOpeningContentTypes: [.folder]
)

// ショートカット連携
class IntentHandler: INExtension {
    override func handler(for intent: INIntent) -> Any {
        return ShellCommandIntentHandler()
    }
}
```

### 学べる技術

- **Cross-compilation**: Unix toolsのARM64ビルド
- **WebAssembly**: ブラウザ技術のネイティブ活用
- **Files.app統合**: UIDocumentPickerでファイルアクセス

---

## 3. SwiftTermApp - SwiftUIでSSH

### ターミナルエミュレータの基本

```swift
import SwiftTerm

class TerminalViewController: UIViewController {
    var terminalView: TerminalView!

    override func viewDidLoad() {
        super.viewDidLoad()

        terminalView = TerminalView(frame: view.bounds)
        view.addSubview(terminalView)

        // ターミナルサイズ
        terminalView.getTerminal().resize(
            cols: 80,
            rows: 24
        )
    }

    // 入力処理
    func send(_ text: String) {
        terminalView.send(txt: text)
    }

    // ANSIエスケープシーケンス処理
    func processOutput(_ data: Data) {
        terminalView.feed(byteArray: [UInt8](data))
    }
}
```

### SSH接続

```swift
import NMSSH

class SSHConnection {
    private var session: NMSSHSession?

    func connect(host: String, port: Int, user: String) async throws {
        session = NMSSHSession(host: host, port: port, andUsername: user)
        session?.connect()

        // 認証方式
        if session?.authenticateBy(
            inMemoryPublicKey: publicKey,
            privateKey: privateKey,
            andPassword: nil
        ) == true {
            // 接続成功
        }
    }

    func executeCommand(_ command: String) async throws -> String {
        guard let channel = session?.channel else { throw SSHError.noChannel }
        return try channel.execute(command)
    }
}
```

### 学べる技術

- **VT100/ANSI**: ターミナルエスケープシーケンス
- **SSH Protocol**: libssh2のiOS移植
- **Keychain統合**: SSH鍵の安全な保存

---

## 4. MobileTerminal - サンドボックス内PoC

### dlopen活用

```c
// 動的ライブラリロード（制限付き）
void* handle = dlopen("/usr/lib/libSystem.B.dylib", RTLD_NOW);
if (handle) {
    // システム関数へのアクセス
    typedef int (*system_fn)(const char*);
    system_fn sys = (system_fn)dlsym(handle, "system");
}
```

### Pseudo Terminal (PTY)

```c
#include <util.h>

int master_fd, slave_fd;
char slave_name[PATH_MAX];

// PTYペア作成
if (openpty(&master_fd, &slave_fd, slave_name, NULL, NULL) == -1) {
    perror("openpty");
    return -1;
}

// 子プロセス（サンドボックス内で制限）
pid_t pid = fork();
if (pid == 0) {
    // 子プロセス: slave側をstdin/stdout/stderrに
    close(master_fd);
    dup2(slave_fd, STDIN_FILENO);
    dup2(slave_fd, STDOUT_FILENO);
    dup2(slave_fd, STDERR_FILENO);

    execl("/bin/sh", "sh", NULL);
}
```

### 学べる技術

- **PTY**: Unix疑似端末の仕組み
- **Process Control**: fork/exec（制限あり）
- **Dynamic Loading**: dlopen/dlsym

---

## 実装の制約と回避策

### App Store制約

| 制約 | 回避策 |
|-----|-------|
| JIT禁止 | インタープリタ方式 |
| fork制限 | PTYエミュレーション |
| /bin/sh不可 | 組み込みシェル実装 |
| root権限なし | ユーザー空間完結 |

### Entitlements

```xml
<!-- SSH接続に必要 -->
<key>com.apple.security.network.client</key>
<true/>

<!-- Files.appアクセス -->
<key>com.apple.security.files.user-selected.read-write</key>
<true/>

<!-- バックグラウンド実行（長時間処理） -->
<key>UIBackgroundModes</key>
<array>
    <string>processing</string>
</array>
```

---

## まとめ

| プロジェクト | 学習価値 | 難易度 |
|-------------|---------|--------|
| iSH | CPU/OS内部理解 | 高 |
| a-shell | クロスコンパイル | 中 |
| SwiftTermApp | SSH/ターミナルUI | 中 |
| MobileTerminal | Unix低レベルAPI | 高 |

### 実装のヒント

1. **SwiftTerm**: ターミナルエミュレータライブラリ
2. **NMSSH**: SSH/SFTPライブラリ
3. **Files.app統合**: Document Picker活用
4. **Keychain**: SSH鍵の安全な保存
