# HomeKit/IoT連携

**驚きポイント**: iPhoneからスマートホーム機器を直接制御

## 参考プロジェクト

| プロジェクト | Stars | 特徴 |
|-------------|-------|------|
| [HomeKit-Demo](https://github.com/KhaosT/HomeKit-Demo) | 314 | HomeKit基礎実装 |
| [openHAB](https://github.com/openhab/openhab-ios) | 205 | オープンソースホームオートメーション |
| [Locative](https://github.com/LocativeHQ/Locative-iOS) | 212 | ジオフェンシング + IoT |

---

## 1. HomeKit基礎

### セットアップ

```swift
import HomeKit

class HomeManager: NSObject, ObservableObject {
    private let homeManager = HMHomeManager()

    @Published var homes: [HMHome] = []
    @Published var accessories: [HMAccessory] = []

    override init() {
        super.init()
        homeManager.delegate = self
    }
}

extension HomeManager: HMHomeManagerDelegate {
    func homeManagerDidUpdateHomes(_ manager: HMHomeManager) {
        homes = manager.homes
        accessories = homes.flatMap { $0.accessories }
    }
}
```

### アクセサリ検出・追加

```swift
extension HomeManager {
    func addAccessory(to home: HMHome) {
        home.addAndSetupAccessories { error in
            if let error = error {
                print("Error adding accessory: \(error)")
            }
        }
    }

    // QRコードでペアリング
    func addAccessory(with setupCode: String, to home: HMHome) async throws {
        let payload = try HMAccessorySetupPayload(url: URL(string: "X-HM://\(setupCode)")!)

        try await withCheckedThrowingContinuation { continuation in
            home.addAndSetupAccessories(with: payload) { error in
                if let error = error {
                    continuation.resume(throwing: error)
                } else {
                    continuation.resume()
                }
            }
        }
    }
}
```

---

## 2. 機器制御

### 照明制御

```swift
class LightController {
    func toggleLight(_ accessory: HMAccessory) async throws {
        guard let lightbulb = accessory.services.first(where: {
            $0.serviceType == HMServiceTypeLightbulb
        }) else { return }

        guard let powerState = lightbulb.characteristics.first(where: {
            $0.characteristicType == HMCharacteristicTypePowerState
        }) else { return }

        let currentState = powerState.value as? Bool ?? false

        try await powerState.writeValue(!currentState)
    }

    func setBrightness(_ accessory: HMAccessory, level: Int) async throws {
        guard let lightbulb = accessory.services.first(where: {
            $0.serviceType == HMServiceTypeLightbulb
        }) else { return }

        guard let brightness = lightbulb.characteristics.first(where: {
            $0.characteristicType == HMCharacteristicTypeBrightness
        }) else { return }

        try await brightness.writeValue(level)  // 0-100
    }

    func setColor(_ accessory: HMAccessory, hue: Float, saturation: Float) async throws {
        guard let lightbulb = accessory.services.first(where: {
            $0.serviceType == HMServiceTypeLightbulb
        }) else { return }

        if let hueChar = lightbulb.characteristics.first(where: {
            $0.characteristicType == HMCharacteristicTypeHue
        }) {
            try await hueChar.writeValue(hue)  // 0-360
        }

        if let satChar = lightbulb.characteristics.first(where: {
            $0.characteristicType == HMCharacteristicTypeSaturation
        }) {
            try await satChar.writeValue(saturation)  // 0-100
        }
    }
}
```

### サーモスタット制御

```swift
class ThermostatController {
    func setTemperature(_ accessory: HMAccessory, celsius: Float) async throws {
        guard let thermostat = accessory.services.first(where: {
            $0.serviceType == HMServiceTypeThermostat
        }) else { return }

        guard let targetTemp = thermostat.characteristics.first(where: {
            $0.characteristicType == HMCharacteristicTypeTargetTemperature
        }) else { return }

        try await targetTemp.writeValue(celsius)
    }

    func setMode(_ accessory: HMAccessory, mode: HMCharacteristicValueHeatingCooling) async throws {
        guard let thermostat = accessory.services.first(where: {
            $0.serviceType == HMServiceTypeThermostat
        }) else { return }

        guard let heatingMode = thermostat.characteristics.first(where: {
            $0.characteristicType == HMCharacteristicTypeTargetHeatingCooling
        }) else { return }

        try await heatingMode.writeValue(mode.rawValue)
    }
}
```

---

## 3. シーン・オートメーション

### シーン作成

```swift
extension HomeManager {
    func createScene(
        name: String,
        actions: [(HMCharacteristic, Any)],
        in home: HMHome
    ) async throws {
        let actionSet = try await withCheckedThrowingContinuation { continuation in
            home.addActionSet(withName: name) { actionSet, error in
                if let error = error {
                    continuation.resume(throwing: error)
                } else if let actionSet = actionSet {
                    continuation.resume(returning: actionSet)
                }
            }
        }

        for (characteristic, value) in actions {
            let action = HMCharacteristicWriteAction(
                characteristic: characteristic,
                targetValue: value as! NSCopying
            )
            try await actionSet.addAction(action)
        }
    }

    func executeScene(_ actionSet: HMActionSet) async throws {
        try await withCheckedThrowingContinuation { continuation in
            actionSet.home?.executeActionSet(actionSet) { error in
                if let error = error {
                    continuation.resume(throwing: error)
                } else {
                    continuation.resume()
                }
            }
        }
    }
}
```

### トリガー設定

```swift
extension HomeManager {
    // 時間トリガー
    func createTimeTrigger(
        name: String,
        at time: DateComponents,
        actionSet: HMActionSet,
        in home: HMHome
    ) async throws {
        let trigger = HMTimerTrigger(
            name: name,
            fireDate: Calendar.current.date(from: time)!,
            timeZone: .current,
            recurrence: nil,  // 繰り返しなし
            recurrenceCalendar: nil
        )

        try await home.addTrigger(trigger)
        try await trigger.addActionSet(actionSet)
        try await trigger.enable(true)
    }

    // 位置トリガー（ジオフェンス）
    func createLocationTrigger(
        name: String,
        region: CLCircularRegion,
        event: HMEventTriggerActivationState,
        actionSet: HMActionSet,
        in home: HMHome
    ) async throws {
        let locationEvent = HMLocationEvent(region: region)

        let trigger = HMEventTrigger(
            name: name,
            events: [locationEvent],
            predicate: nil
        )

        try await home.addTrigger(trigger)
        try await trigger.addActionSet(actionSet)
        try await trigger.enable(true)
    }
}
```

---

## 4. センサー値の監視

```swift
class SensorMonitor: NSObject {
    private var observedCharacteristics: [HMCharacteristic] = []

    func startMonitoring(_ accessory: HMAccessory) {
        guard let sensor = accessory.services.first(where: {
            $0.serviceType == HMServiceTypeTemperatureSensor
        }) else { return }

        guard let temperature = sensor.characteristics.first(where: {
            $0.characteristicType == HMCharacteristicTypeCurrentTemperature
        }) else { return }

        temperature.enableNotification(true) { error in
            if error == nil {
                self.observedCharacteristics.append(temperature)
            }
        }
    }
}

extension SensorMonitor: HMAccessoryDelegate {
    func accessory(
        _ accessory: HMAccessory,
        service: HMService,
        didUpdateValueFor characteristic: HMCharacteristic
    ) {
        if characteristic.characteristicType == HMCharacteristicTypeCurrentTemperature {
            if let temp = characteristic.value as? Float {
                print("Temperature: \(temp)°C")
            }
        }
    }
}
```

---

## 5. Entitlements

```xml
<!-- HomeKit.entitlements -->
<key>com.apple.developer.homekit</key>
<true/>
```

### Info.plist

```xml
<key>NSHomeKitUsageDescription</key>
<string>スマートホーム機器を制御します</string>
```

---

## まとめ

| 機能 | クラス | 用途 |
|------|--------|------|
| ホーム管理 | HMHomeManager | ホーム一覧 |
| 機器制御 | HMCharacteristic | ON/OFF、値設定 |
| シーン | HMActionSet | 複数操作の一括実行 |
| オートメーション | HMTrigger | 自動実行 |
| 監視 | enableNotification | リアルタイム通知 |

### 対応機器タイプ

- 照明（HMServiceTypeLightbulb）
- スイッチ（HMServiceTypeSwitch）
- サーモスタット（HMServiceTypeThermostat）
- センサー（温度、湿度、モーション）
- ドアロック（HMServiceTypeLockMechanism）
- カメラ（HMServiceTypeCameraControl）
