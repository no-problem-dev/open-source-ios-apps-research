# CoreMotionとセンサー活用

**驚きポイント**: 加速度計・ジャイロスコープで「ノック検出」や「空中描画」

## 参考プロジェクト

| プロジェクト | Stars | 特徴 |
|-------------|-------|------|
| [Science Journal](https://github.com/googlearchive/science-journal-ios) | 536 | 全センサー可視化 |
| [SwiftSpace](https://github.com/FlexMonkey/SwiftSpace) | 152 | ジャイロで3D描画 |
| [Knock](https://github.com/MatheusCavalca/Knock) | 25 | 加速度計ノック検出 |

---

## 1. センサー一覧

| センサー | クラス | 用途 |
|---------|--------|------|
| 加速度計 | CMAccelerometerData | 動き、傾き |
| ジャイロスコープ | CMGyroData | 回転 |
| 磁力計 | CMMagnetometerData | 方角 |
| 気圧計 | CMAltimeter | 高度変化 |
| 歩数計 | CMPedometer | 歩数、距離 |
| デバイスモーション | CMDeviceMotion | 融合データ |

---

## 2. 加速度計でノック検出

```swift
import CoreMotion

class KnockDetector: ObservableObject {
    private let motionManager = CMMotionManager()
    private var lastAcceleration: Double = 0
    private var knockCount = 0
    private var lastKnockTime: Date?

    @Published var isKnockDetected = false

    func startDetection() {
        guard motionManager.isAccelerometerAvailable else { return }

        motionManager.accelerometerUpdateInterval = 0.01  // 100Hz

        motionManager.startAccelerometerUpdates(to: .main) { [weak self] data, _ in
            guard let self = self, let data = data else { return }

            let acceleration = sqrt(
                pow(data.acceleration.x, 2) +
                pow(data.acceleration.y, 2) +
                pow(data.acceleration.z, 2)
            )

            // 急激な加速度変化を検出
            let delta = abs(acceleration - self.lastAcceleration)
            self.lastAcceleration = acceleration

            if delta > 2.5 {  // しきい値
                self.handlePotentialKnock()
            }
        }
    }

    private func handlePotentialKnock() {
        let now = Date()

        // 連続ノック判定（0.5秒以内）
        if let lastTime = lastKnockTime,
           now.timeIntervalSince(lastTime) < 0.5 {
            knockCount += 1
        } else {
            knockCount = 1
        }
        lastKnockTime = now

        // ダブルノック検出
        if knockCount >= 2 {
            isKnockDetected = true
            knockCount = 0

            // 0.5秒後にリセット
            DispatchQueue.main.asyncAfter(deadline: .now() + 0.5) {
                self.isKnockDetected = false
            }
        }
    }

    func stopDetection() {
        motionManager.stopAccelerometerUpdates()
    }
}
```

---

## 3. ジャイロスコープで3D描画

```swift
import CoreMotion
import SceneKit

class GyroDrawing: ObservableObject {
    private let motionManager = CMMotionManager()

    @Published var currentRotation = SCNVector3Zero
    @Published var drawingPath: [SCNVector3] = []

    private var isDrawing = false
    private var currentPosition = SCNVector3Zero

    func startTracking() {
        guard motionManager.isDeviceMotionAvailable else { return }

        motionManager.deviceMotionUpdateInterval = 0.02  // 50Hz

        motionManager.startDeviceMotionUpdates(to: .main) { [weak self] motion, _ in
            guard let self = self, let motion = motion else { return }

            // 回転データ
            let attitude = motion.attitude
            self.currentRotation = SCNVector3(
                Float(attitude.pitch),
                Float(attitude.yaw),
                Float(attitude.roll)
            )

            if self.isDrawing {
                // 加速度から移動量を計算
                let userAccel = motion.userAcceleration
                self.currentPosition.x += Float(userAccel.x * 0.01)
                self.currentPosition.y += Float(userAccel.y * 0.01)
                self.currentPosition.z += Float(userAccel.z * 0.01)

                self.drawingPath.append(self.currentPosition)
            }
        }
    }

    func startDrawing() {
        isDrawing = true
        currentPosition = SCNVector3Zero
        drawingPath = []
    }

    func stopDrawing() {
        isDrawing = false
    }
}
```

### SceneKitで可視化

```swift
import SceneKit
import SwiftUI

struct Drawing3DView: UIViewRepresentable {
    let path: [SCNVector3]

    func makeUIView(context: Context) -> SCNView {
        let sceneView = SCNView()
        sceneView.scene = SCNScene()
        sceneView.allowsCameraControl = true
        sceneView.backgroundColor = .black
        return sceneView
    }

    func updateUIView(_ sceneView: SCNView, context: Context) {
        // 既存のパスを削除
        sceneView.scene?.rootNode.childNodes
            .filter { $0.name == "path" }
            .forEach { $0.removeFromParentNode() }

        guard path.count > 1 else { return }

        // パスを描画
        for i in 1..<path.count {
            let start = path[i - 1]
            let end = path[i]

            let line = lineBetween(start: start, end: end)
            line.name = "path"
            sceneView.scene?.rootNode.addChildNode(line)
        }
    }

    private func lineBetween(start: SCNVector3, end: SCNVector3) -> SCNNode {
        let distance = sqrt(
            pow(end.x - start.x, 2) +
            pow(end.y - start.y, 2) +
            pow(end.z - start.z, 2)
        )

        let cylinder = SCNCylinder(radius: 0.002, height: CGFloat(distance))
        cylinder.firstMaterial?.diffuse.contents = UIColor.cyan

        let node = SCNNode(geometry: cylinder)
        node.position = SCNVector3(
            (start.x + end.x) / 2,
            (start.y + end.y) / 2,
            (start.z + end.z) / 2
        )

        node.look(at: end, up: SCNVector3(0, 1, 0), localFront: SCNVector3(0, 1, 0))

        return node
    }
}
```

---

## 4. 気圧計で高度変化

```swift
import CoreMotion

class AltitudeTracker: ObservableObject {
    private let altimeter = CMAltimeter()

    @Published var relativeAltitude: Double = 0  // メートル
    @Published var pressure: Double = 0  // キロパスカル

    func startTracking() {
        guard CMAltimeter.isRelativeAltitudeAvailable() else { return }

        altimeter.startRelativeAltitudeUpdates(to: .main) { [weak self] data, error in
            guard let self = self, let data = data else { return }

            self.relativeAltitude = data.relativeAltitude.doubleValue
            self.pressure = data.pressure.doubleValue
        }
    }

    func stopTracking() {
        altimeter.stopRelativeAltitudeUpdates()
    }
}
```

---

## 5. 歩数計

```swift
import CoreMotion

class StepCounter: ObservableObject {
    private let pedometer = CMPedometer()

    @Published var steps: Int = 0
    @Published var distance: Double = 0  // メートル
    @Published var floorsAscended: Int = 0
    @Published var floorsDescended: Int = 0

    func startCounting() {
        guard CMPedometer.isStepCountingAvailable() else { return }

        let startOfDay = Calendar.current.startOfDay(for: Date())

        // 今日のデータを取得
        pedometer.queryPedometerData(from: startOfDay, to: Date()) { [weak self] data, _ in
            DispatchQueue.main.async {
                self?.updateData(data)
            }
        }

        // リアルタイム更新
        pedometer.startUpdates(from: startOfDay) { [weak self] data, _ in
            DispatchQueue.main.async {
                self?.updateData(data)
            }
        }
    }

    private func updateData(_ data: CMPedometerData?) {
        guard let data = data else { return }

        steps = data.numberOfSteps.intValue
        distance = data.distance?.doubleValue ?? 0
        floorsAscended = data.floorsAscended?.intValue ?? 0
        floorsDescended = data.floorsDescended?.intValue ?? 0
    }
}
```

---

## 6. バックグラウンドでのセンサー取得

### Info.plist

```xml
<key>UIBackgroundModes</key>
<array>
    <string>location</string>  <!-- 継続的な位置情報 -->
</array>

<key>NSMotionUsageDescription</key>
<string>モーションデータを使用してアクティビティを追跡します</string>
```

### バックグラウンドタスク

```swift
import BackgroundTasks

class BackgroundMotionManager {
    static let taskIdentifier = "com.app.motion-tracking"

    func registerBackgroundTask() {
        BGTaskScheduler.shared.register(
            forTaskWithIdentifier: Self.taskIdentifier,
            using: nil
        ) { task in
            self.handleBackgroundTask(task as! BGProcessingTask)
        }
    }

    func scheduleBackgroundTask() {
        let request = BGProcessingTaskRequest(identifier: Self.taskIdentifier)
        request.requiresNetworkConnectivity = false
        request.requiresExternalPower = false

        try? BGTaskScheduler.shared.submit(request)
    }

    private func handleBackgroundTask(_ task: BGProcessingTask) {
        // バックグラウンドでセンサーデータ処理
        task.expirationHandler = {
            // クリーンアップ
        }

        // 処理完了
        task.setTaskCompleted(success: true)
    }
}
```

---

## 7. センサーフュージョン

```swift
class SensorFusion: ObservableObject {
    private let motionManager = CMMotionManager()

    @Published var heading: Double = 0  // 方角
    @Published var tilt: (pitch: Double, roll: Double) = (0, 0)
    @Published var isMoving: Bool = false

    func startFusedTracking() {
        guard motionManager.isDeviceMotionAvailable else { return }

        motionManager.deviceMotionUpdateInterval = 0.02

        // 磁力計を含むフュージョン
        motionManager.startDeviceMotionUpdates(
            using: .xMagneticNorthZVertical,
            to: .main
        ) { [weak self] motion, _ in
            guard let self = self, let motion = motion else { return }

            // 方角（磁北基準）
            self.heading = motion.heading

            // 傾き
            self.tilt = (
                pitch: motion.attitude.pitch * 180 / .pi,
                roll: motion.attitude.roll * 180 / .pi
            )

            // 動き検出
            let acceleration = motion.userAcceleration
            let magnitude = sqrt(
                acceleration.x * acceleration.x +
                acceleration.y * acceleration.y +
                acceleration.z * acceleration.z
            )
            self.isMoving = magnitude > 0.1
        }
    }
}
```

---

## まとめ

| センサー | 用途例 | 精度 |
|---------|--------|------|
| 加速度計 | ノック検出、シェイク | 高 |
| ジャイロ | 3D描画、ゲーム操作 | 高 |
| 磁力計 | コンパス、AR方位 | 中 |
| 気圧計 | 階段検出、高度変化 | 高 |
| 歩数計 | ヘルスケア連携 | 高 |

### 実装のヒント

1. **更新頻度**: 用途に応じて調整（バッテリー消費）
2. **ノイズ除去**: ローパスフィルタ適用
3. **キャリブレーション**: 磁力計は環境影響大
4. **バックグラウンド**: 制限あり、BGTaskScheduler活用
