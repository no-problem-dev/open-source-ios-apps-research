# 逆コンパイル・バイナリ解析

**驚きポイント**: FairPlay暗号化を解除してIPAを解析

## 参考プロジェクト

| プロジェクト | Stars | 特徴 |
|-------------|-------|------|
| [yacd](https://github.com/DerekSelander/yacd) | 691 | FairPlay復号（iOS 13.4.1以下） |

---

## 1. FairPlayとは

App Storeからダウンロードしたアプリはすべて**FairPlay DRM**で暗号化。

```
通常のIPA:
└── Payload/
    └── App.app/
        └── App (暗号化されたMach-O)

復号後:
└── Payload/
    └── App.app/
        └── App (平文Mach-O → 解析可能)
```

---

## 2. yacdの仕組み

### 暗号化セクション検出

```c
// Mach-Oヘッダー解析
struct encryption_info_command_64 {
    uint32_t cmd;          // LC_ENCRYPTION_INFO_64
    uint32_t cmdsize;
    uint32_t cryptoff;     // 暗号化開始オフセット
    uint32_t cryptsize;    // 暗号化サイズ
    uint32_t cryptid;      // 0=平文, 1=暗号化
    uint32_t pad;
};

// cryptid == 1 なら暗号化されている
```

### メモリダンプ方式

実行中のプロセスからメモリをダンプ。

```c
// 1. アプリを起動（OSが復号してメモリにロード）
// 2. メモリ上の復号済みセクションを読み取り
// 3. 元のMach-Oファイルの暗号化部分を置換
// 4. cryptid を 0 に変更

task_t task;
task_for_pid(mach_task_self(), pid, &task);

vm_size_t size;
vm_address_t data;
vm_read(task, address, cryptsize, &data, &size);

// 復号済みデータを書き戻し
```

---

## 3. 代替手法

### Frida（動的解析）

```javascript
// script.js
Interceptor.attach(Module.findExportByName(null, "CCCrypt"), {
    onEnter: function(args) {
        console.log("CCCrypt called");
        console.log("  operation: " + args[0]);
        console.log("  data: " + hexdump(args[4], { length: 32 }));
    }
});
```

```bash
frida -U -f com.example.app -l script.js
```

### class-dump

```bash
# ヘッダー抽出（復号後）
class-dump -H /path/to/App.app/App -o Headers/
```

---

## 4. Mach-O構造

```
Mach-Oファイル:
├── Mach Header (magic, cputype, filetype)
├── Load Commands
│   ├── LC_SEGMENT_64 (__TEXT)
│   ├── LC_SEGMENT_64 (__DATA)
│   ├── LC_ENCRYPTION_INFO_64
│   ├── LC_DYLD_INFO_ONLY
│   └── ...
├── Segment Data
│   ├── __TEXT (コード)
│   ├── __DATA (データ)
│   └── __LINKEDIT (シンボル)
└── Code Signature
```

### Swift解析

```bash
# Swift型メタデータ
nm /path/to/App | grep "_$s"

# swift-demangle
echo "_\$s7MyClass4nameSSvg" | swift-demangle
# → MyClass.name.getter : String
```

---

## 5. 法的・倫理的注意

**やってはいけないこと**:
- 著作権侵害目的の復号
- 海賊版作成
- プロプライエタリコードの再配布

**許容される用途**:
- セキュリティ研究
- マルウェア解析
- 自分のアプリのデバッグ
- 相互運用性の確保（DMCA例外）

---

## 6. 防御策（開発者向け）

### 難読化

```swift
// 文字列暗号化
let encrypted = "SGVsbG8gV29ybGQ=".data(using: .utf8)!
let decrypted = Data(base64Encoded: encrypted)!
let string = String(data: decrypted, encoding: .utf8)!
```

### 改ざん検出

```swift
func checkIntegrity() -> Bool {
    // バイナリハッシュ検証
    guard let bundlePath = Bundle.main.bundlePath else { return false }
    let executablePath = bundlePath + "/\(Bundle.main.infoDictionary!["CFBundleExecutable"]!)"

    let fileData = try? Data(contentsOf: URL(fileURLWithPath: executablePath))
    let hash = SHA256.hash(data: fileData ?? Data())

    // 期待値と比較
    return hash.description == expectedHash
}
```

### デバッガ検出

```swift
func isDebuggerAttached() -> Bool {
    var info = kinfo_proc()
    var size = MemoryLayout<kinfo_proc>.stride
    var mib: [Int32] = [CTL_KERN, KERN_PROC, KERN_PROC_PID, getpid()]

    sysctl(&mib, UInt32(mib.count), &info, &size, nil, 0)

    return (info.kp_proc.p_flag & P_TRACED) != 0
}
```

---

## 7. ツールチェーン

| ツール | 用途 |
|--------|------|
| yacd | FairPlay復号 |
| Hopper | 逆アセンブラ（有料） |
| Ghidra | 逆アセンブラ（無料） |
| class-dump | Obj-Cヘッダー抽出 |
| Frida | 動的解析 |
| lldb | デバッガ |
| otool | Mach-O解析 |
| nm | シンボル表示 |

---

## まとめ

| 手法 | 対象 | 難易度 |
|------|------|--------|
| メモリダンプ | 暗号化解除 | 高 |
| class-dump | クラス構造 | 低 |
| Frida | 動的フック | 中 |
| 逆アセンブル | 実装解析 | 高 |

### 学習価値

- Mach-O構造の理解
- OSローダーの動作
- セキュリティ対策の設計
