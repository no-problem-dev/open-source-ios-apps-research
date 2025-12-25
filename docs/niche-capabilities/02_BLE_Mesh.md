# BLE Meshネットワーク通信

**驚きポイント**: サーバー不要で端末同士が直接通信するP2Pメッシュネットワーク

## 参考プロジェクト

| プロジェクト | Stars | 特徴 |
|-------------|-------|------|
| [BLEMeshChat](https://github.com/chrisballinger/BLEMeshChat) | 500 | BLE 4.0メッシュチャット |

---

## 1. BLE Meshの概念

### 従来のクライアント/サーバー

```
Device A  ←→  Server  ←→  Device B
              ↕
           Device C
```

### BLE Mesh

```
Device A  ←→  Device B
    ↕           ↕
Device C  ←→  Device D
```

各端末がPeripheral（広告）とCentral（スキャン）を同時に担う。

---

## 2. CoreBluetooth基礎

### Peripheral（発信側）

```swift
import CoreBluetooth

class BLEPeripheralManager: NSObject, CBPeripheralManagerDelegate {
    private var peripheralManager: CBPeripheralManager!
    private var messageCharacteristic: CBMutableCharacteristic!

    // カスタムUUID
    private let serviceUUID = CBUUID(string: "your-service-uuid")
    private let characteristicUUID = CBUUID(string: "your-char-uuid")

    func startAdvertising() {
        peripheralManager = CBPeripheralManager(delegate: self, queue: nil)
    }

    func peripheralManagerDidUpdateState(_ peripheral: CBPeripheralManager) {
        guard peripheral.state == .poweredOn else { return }

        // Characteristicを作成
        messageCharacteristic = CBMutableCharacteristic(
            type: characteristicUUID,
            properties: [.read, .write, .notify],
            value: nil,
            permissions: [.readable, .writeable]
        )

        // Serviceを作成
        let service = CBMutableService(type: serviceUUID, primary: true)
        service.characteristics = [messageCharacteristic]
        peripheralManager.add(service)

        // 広告開始
        peripheralManager.startAdvertising([
            CBAdvertisementDataServiceUUIDsKey: [serviceUUID],
            CBAdvertisementDataLocalNameKey: "MeshNode"
        ])
    }

    // 書き込みリクエスト受信
    func peripheralManager(
        _ peripheral: CBPeripheralManager,
        didReceiveWrite requests: [CBATTRequest]
    ) {
        for request in requests {
            if let value = request.value {
                // メッセージ受信
                let message = String(data: value, encoding: .utf8)
                handleReceivedMessage(message)

                // 他のノードにリレー
                relayMessage(value)
            }
            peripheral.respond(to: request, withResult: .success)
        }
    }
}
```

### Central（受信側）

```swift
class BLECentralManager: NSObject, CBCentralManagerDelegate, CBPeripheralDelegate {
    private var centralManager: CBCentralManager!
    private var discoveredPeripherals: [CBPeripheral] = []

    private let serviceUUID = CBUUID(string: "your-service-uuid")
    private let characteristicUUID = CBUUID(string: "your-char-uuid")

    func startScanning() {
        centralManager = CBCentralManager(delegate: self, queue: nil)
    }

    func centralManagerDidUpdateState(_ central: CBCentralManager) {
        guard central.state == .poweredOn else { return }

        // スキャン開始
        centralManager.scanForPeripherals(
            withServices: [serviceUUID],
            options: [CBCentralManagerScanOptionAllowDuplicatesKey: true]
        )
    }

    func centralManager(
        _ central: CBCentralManager,
        didDiscover peripheral: CBPeripheral,
        advertisementData: [String: Any],
        rssi RSSI: NSNumber
    ) {
        // 新しいノード発見
        if !discoveredPeripherals.contains(peripheral) {
            discoveredPeripherals.append(peripheral)
            peripheral.delegate = self
            centralManager.connect(peripheral)
        }
    }

    func centralManager(
        _ central: CBCentralManager,
        didConnect peripheral: CBPeripheral
    ) {
        peripheral.discoverServices([serviceUUID])
    }

    func peripheral(
        _ peripheral: CBPeripheral,
        didDiscoverServices error: Error?
    ) {
        peripheral.services?.forEach { service in
            peripheral.discoverCharacteristics(
                [characteristicUUID],
                for: service
            )
        }
    }

    // メッセージ送信
    func sendMessage(_ message: String, to peripheral: CBPeripheral) {
        guard let characteristic = findCharacteristic(for: peripheral),
              let data = message.data(using: .utf8) else { return }

        peripheral.writeValue(
            data,
            for: characteristic,
            type: .withResponse
        )
    }
}
```

---

## 3. Mesh Routing

### メッセージ構造

```swift
struct MeshMessage: Codable {
    let id: UUID           // 重複排除用
    let source: String     // 送信元ノードID
    let destination: String // 宛先（"*"でブロードキャスト）
    let ttl: Int           // Time To Live（ホップ数制限）
    let payload: Data
    let timestamp: Date
}
```

### フラッディング方式

```swift
class MeshRouter {
    private var seenMessages: Set<UUID> = []
    private let maxTTL = 5

    func handleMessage(_ message: MeshMessage) {
        // 重複チェック
        guard !seenMessages.contains(message.id) else { return }
        seenMessages.insert(message.id)

        // 自分宛てなら処理
        if message.destination == myNodeID || message.destination == "*" {
            processMessage(message)
        }

        // TTLチェックしてリレー
        if message.ttl > 0 && message.destination != myNodeID {
            var relayMessage = message
            relayMessage.ttl -= 1
            broadcastToAllPeers(relayMessage)
        }
    }

    func sendMessage(_ payload: Data, to destination: String) {
        let message = MeshMessage(
            id: UUID(),
            source: myNodeID,
            destination: destination,
            ttl: maxTTL,
            payload: payload,
            timestamp: Date()
        )
        broadcastToAllPeers(message)
    }
}
```

---

## 4. バックグラウンド動作

### Info.plist設定

```xml
<key>UIBackgroundModes</key>
<array>
    <string>bluetooth-central</string>
    <string>bluetooth-peripheral</string>
</array>

<key>NSBluetoothAlwaysUsageDescription</key>
<string>メッシュネットワークでメッセージを送受信します</string>

<key>NSBluetoothPeripheralUsageDescription</key>
<string>他のデバイスからの接続を受け付けます</string>
```

### バックグラウンド制約

```swift
// バックグラウンドでは広告データが制限される
// LocalNameは含められない
peripheralManager.startAdvertising([
    CBAdvertisementDataServiceUUIDsKey: [serviceUUID]
    // CBAdvertisementDataLocalNameKey は無視される
])

// スキャンも制限される
// allowDuplicates: false が強制
centralManager.scanForPeripherals(
    withServices: [serviceUUID],  // サービスUUID必須
    options: nil
)
```

---

## 5. 実用的な最適化

### 接続管理

```swift
class ConnectionManager {
    private var connections: [String: CBPeripheral] = [:]
    private let maxConnections = 7  // BLE制限

    func shouldConnect(to peripheral: CBPeripheral) -> Bool {
        // 接続数制限
        guard connections.count < maxConnections else { return false }

        // 既に接続済みはスキップ
        guard connections[peripheral.identifier.uuidString] == nil else {
            return false
        }

        return true
    }

    func pruneStaleConnections() {
        // 長時間応答がないノードを切断
        for (id, peripheral) in connections {
            if peripheral.state == .disconnected {
                connections.removeValue(forKey: id)
            }
        }
    }
}
```

### RSSI活用

```swift
// 信号強度で近いノードを優先
func prioritizeNearbyNodes(
    _ peripherals: [CBPeripheral],
    rssiValues: [UUID: Int]
) -> [CBPeripheral] {
    peripherals.sorted { a, b in
        let rssiA = rssiValues[a.identifier] ?? -100
        let rssiB = rssiValues[b.identifier] ?? -100
        return rssiA > rssiB  // 強い順
    }
}
```

---

## 6. MultipeerConnectivityとの比較

| 項目 | CoreBluetooth | MultipeerConnectivity |
|------|---------------|----------------------|
| 通信距離 | 〜10m | 〜100m (Wi-Fi) |
| バッテリー | 省電力 | 消費大 |
| バックグラウンド | 制限付きで可 | 制限大 |
| データサイズ | 小（〜512byte） | 大 |
| カスタマイズ | 高 | 低 |
| 実装難易度 | 高 | 低 |

### MultipeerConnectivity例

```swift
import MultipeerConnectivity

class MultipeerManager: NSObject {
    private let serviceType = "mesh-chat"
    private let peerID = MCPeerID(displayName: UIDevice.current.name)
    private var session: MCSession!
    private var advertiser: MCNearbyServiceAdvertiser!
    private var browser: MCNearbyServiceBrowser!

    func start() {
        session = MCSession(peer: peerID, securityIdentity: nil, encryptionPreference: .required)
        session.delegate = self

        // 広告
        advertiser = MCNearbyServiceAdvertiser(peer: peerID, discoveryInfo: nil, serviceType: serviceType)
        advertiser.delegate = self
        advertiser.startAdvertisingPeer()

        // 検索
        browser = MCNearbyServiceBrowser(peer: peerID, serviceType: serviceType)
        browser.delegate = self
        browser.startBrowsingForPeers()
    }
}
```

---

## まとめ

| 技術 | 用途 | 難易度 |
|------|------|--------|
| BLE Peripheral/Central | P2P通信基盤 | 中 |
| Mesh Routing | マルチホップ通信 | 高 |
| バックグラウンドBLE | 常時接続 | 高 |
| MultipeerConnectivity | 簡易P2P | 低 |

### 適用シナリオ

- 災害時オフライン通信
- イベント会場でのP2Pチャット
- IoTセンサーネットワーク
- 近接ファイル共有
