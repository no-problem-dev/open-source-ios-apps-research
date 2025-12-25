# Live Activity & Dynamic Island 実装ガイド

**参考プロジェクト**:
- Apple公式サンプル: Food Truck
- DoorDash: https://github.com/nickolashkraus/doordash-ios
- Uber風アプリ実装例

**実装参考度**: ⭐⭐⭐⭐⭐

---

## なぜ重要か

Live Activityは**リアルタイム進行状況**をロック画面とDynamic Islandに表示する機能。配送、スポーツ、タイマーなど、時間経過で変化するコンテンツに最適。

| 学習ポイント | 重要度 | 理由 |
|-------------|--------|------|
| **ActivityKit** | ⭐⭐⭐⭐⭐ | Live Activity の基盤 |
| **Dynamic Island** | ⭐⭐⭐⭐⭐ | iPhone 14 Pro+ 専用UI |
| **Push通知更新** | ⭐⭐⭐⭐☆ | サーバーからのリアルタイム更新 |
| **ContentState** | ⭐⭐⭐⭐☆ | 動的コンテンツ管理 |

---

## 1. 基本的なLive Activity

### ActivityAttributes 定義

```swift
import ActivityKit

struct DeliveryActivityAttributes: ActivityAttributes {
    // 静的コンテンツ（開始時に決まり、変更されない）
    let orderID: String
    let restaurantName: String
    let totalItems: Int

    // 動的コンテンツ（更新される）
    struct ContentState: Codable, Hashable {
        var status: DeliveryStatus
        var driverName: String?
        var estimatedDeliveryTime: Date
        var currentStep: Int // 1: 準備中, 2: 配達中, 3: 到着
    }
}

enum DeliveryStatus: String, Codable {
    case preparing = "preparing"
    case pickedUp = "picked_up"
    case delivering = "delivering"
    case arrived = "arrived"

    var displayText: String {
        switch self {
        case .preparing: "準備中"
        case .pickedUp: "ピックアップ完了"
        case .delivering: "配達中"
        case .arrived: "到着"
        }
    }
}
```

### Live Activity 開始

```swift
import ActivityKit

class DeliveryManager {
    private var currentActivity: Activity<DeliveryActivityAttributes>?

    func startLiveActivity(order: Order) async throws {
        // iOS 16.2+でLive Activityが利用可能かチェック
        guard ActivityAuthorizationInfo().areActivitiesEnabled else {
            throw DeliveryError.liveActivityNotSupported
        }

        let attributes = DeliveryActivityAttributes(
            orderID: order.id,
            restaurantName: order.restaurant.name,
            totalItems: order.items.count
        )

        let initialState = DeliveryActivityAttributes.ContentState(
            status: .preparing,
            driverName: nil,
            estimatedDeliveryTime: order.estimatedDelivery,
            currentStep: 1
        )

        do {
            let activity = try Activity.request(
                attributes: attributes,
                content: .init(state: initialState, staleDate: nil),
                pushType: .token // Push通知で更新する場合
            )

            currentActivity = activity

            // Push tokenをサーバーに送信
            for await pushToken in activity.pushTokenUpdates {
                let tokenString = pushToken.map { String(format: "%02x", $0) }.joined()
                await sendTokenToServer(tokenString, orderID: order.id)
            }
        } catch {
            throw DeliveryError.failedToStartActivity(error)
        }
    }
}
```

### Live Activity 更新

```swift
extension DeliveryManager {
    func updateActivity(status: DeliveryStatus, driverName: String?, eta: Date) async {
        guard let activity = currentActivity else { return }

        let updatedState = DeliveryActivityAttributes.ContentState(
            status: status,
            driverName: driverName,
            estimatedDeliveryTime: eta,
            currentStep: status.step
        )

        await activity.update(
            ActivityContent(
                state: updatedState,
                staleDate: Calendar.current.date(byAdding: .minute, value: 30, to: Date())
            )
        )
    }

    func endActivity(finalStatus: DeliveryStatus) async {
        guard let activity = currentActivity else { return }

        let finalState = DeliveryActivityAttributes.ContentState(
            status: finalStatus,
            driverName: activity.content.state.driverName,
            estimatedDeliveryTime: Date(),
            currentStep: 4
        )

        await activity.end(
            ActivityContent(state: finalState, staleDate: nil),
            dismissalPolicy: .after(.now + 300) // 5分後に自動消去
        )

        currentActivity = nil
    }
}
```

---

## 2. Widget Extension 実装

### Live Activity View

```swift
import WidgetKit
import SwiftUI

struct DeliveryLiveActivity: Widget {
    var body: some WidgetConfiguration {
        ActivityConfiguration(for: DeliveryActivityAttributes.self) { context in
            // ロック画面用UI
            LockScreenView(context: context)
        } dynamicIsland: { context in
            // Dynamic Island UI
            DynamicIsland {
                // Expanded (長押し時)
                DynamicIslandExpandedRegion(.leading) {
                    ExpandedLeadingView(context: context)
                }
                DynamicIslandExpandedRegion(.trailing) {
                    ExpandedTrailingView(context: context)
                }
                DynamicIslandExpandedRegion(.center) {
                    ExpandedCenterView(context: context)
                }
                DynamicIslandExpandedRegion(.bottom) {
                    ExpandedBottomView(context: context)
                }
            } compactLeading: {
                // Compact Left
                Image(systemName: "bicycle")
                    .foregroundStyle(.green)
            } compactTrailing: {
                // Compact Right
                Text(context.state.estimatedDeliveryTime, style: .timer)
                    .font(.caption2)
                    .monospacedDigit()
            } minimal: {
                // Minimal (他のLive Activityがある時)
                Image(systemName: "bicycle.circle.fill")
                    .foregroundStyle(.green)
            }
        }
    }
}
```

### ロック画面View

```swift
struct LockScreenView: View {
    let context: ActivityViewContext<DeliveryActivityAttributes>

    var body: some View {
        VStack(alignment: .leading, spacing: 12) {
            // ヘッダー
            HStack {
                Image(systemName: "bag.fill")
                    .foregroundStyle(.green)
                Text(context.attributes.restaurantName)
                    .font(.headline)
                Spacer()
                Text("注文 #\(context.attributes.orderID.suffix(4))")
                    .font(.caption)
                    .foregroundStyle(.secondary)
            }

            // プログレス
            DeliveryProgressView(currentStep: context.state.currentStep)

            // ステータス
            HStack {
                VStack(alignment: .leading) {
                    Text(context.state.status.displayText)
                        .font(.subheadline)
                        .fontWeight(.semibold)

                    if let driver = context.state.driverName {
                        Text("配達員: \(driver)")
                            .font(.caption)
                            .foregroundStyle(.secondary)
                    }
                }

                Spacer()

                VStack(alignment: .trailing) {
                    Text("到着予定")
                        .font(.caption)
                        .foregroundStyle(.secondary)
                    Text(context.state.estimatedDeliveryTime, style: .time)
                        .font(.title2)
                        .fontWeight(.bold)
                }
            }
        }
        .padding()
    }
}

struct DeliveryProgressView: View {
    let currentStep: Int

    var body: some View {
        HStack(spacing: 4) {
            ForEach(1...4, id: \.self) { step in
                RoundedRectangle(cornerRadius: 2)
                    .fill(step <= currentStep ? Color.green : Color.gray.opacity(0.3))
                    .frame(height: 4)
            }
        }
    }
}
```

### Dynamic Island Expanded View

```swift
struct ExpandedLeadingView: View {
    let context: ActivityViewContext<DeliveryActivityAttributes>

    var body: some View {
        VStack(alignment: .leading) {
            Image(systemName: "bag.fill")
                .font(.title2)
                .foregroundStyle(.green)
            Text(context.attributes.restaurantName)
                .font(.caption)
                .lineLimit(1)
        }
    }
}

struct ExpandedTrailingView: View {
    let context: ActivityViewContext<DeliveryActivityAttributes>

    var body: some View {
        VStack(alignment: .trailing) {
            Text(context.state.estimatedDeliveryTime, style: .time)
                .font(.title2)
                .fontWeight(.bold)
            Text("到着予定")
                .font(.caption2)
                .foregroundStyle(.secondary)
        }
    }
}

struct ExpandedCenterView: View {
    let context: ActivityViewContext<DeliveryActivityAttributes>

    var body: some View {
        Text(context.state.status.displayText)
            .font(.headline)
    }
}

struct ExpandedBottomView: View {
    let context: ActivityViewContext<DeliveryActivityAttributes>

    var body: some View {
        HStack {
            DeliveryProgressView(currentStep: context.state.currentStep)
        }
        .padding(.horizontal)
    }
}
```

---

## 3. Push通知による更新

### サーバー側実装（Node.js例）

```javascript
const apn = require('apn');

async function updateLiveActivity(pushToken, orderID, newStatus) {
    const provider = new apn.Provider({
        token: {
            key: 'path/to/AuthKey.p8',
            keyId: 'KEY_ID',
            teamId: 'TEAM_ID'
        },
        production: true
    });

    const notification = new apn.Notification();
    notification.topic = 'com.example.app.push-type.liveactivity';
    notification.pushType = 'liveactivity';
    notification.relevanceScore = 100;
    notification.timestamp = Math.floor(Date.now() / 1000);

    notification.payload = {
        aps: {
            timestamp: Math.floor(Date.now() / 1000),
            event: 'update', // 'update' or 'end'
            'content-state': {
                status: newStatus,
                driverName: 'Taro',
                estimatedDeliveryTime: new Date().toISOString(),
                currentStep: 2
            },
            alert: {
                title: 'Delivery Update',
                body: 'Your order is on its way!'
            }
        }
    };

    await provider.send(notification, pushToken);
}
```

### Push payload 形式

```json
{
    "aps": {
        "timestamp": 1699123456,
        "event": "update",
        "content-state": {
            "status": "delivering",
            "driverName": "田中太郎",
            "estimatedDeliveryTime": "2024-01-15T18:30:00Z",
            "currentStep": 3
        },
        "dismissal-date": 1699127056,
        "alert": {
            "title": "配達状況",
            "body": "配達員が向かっています"
        }
    }
}
```

---

## 4. タイマー型Live Activity

### カウントダウンタイマー

```swift
struct TimerActivityAttributes: ActivityAttributes {
    let timerName: String
    let totalDuration: TimeInterval

    struct ContentState: Codable, Hashable {
        var endTime: Date
        var isPaused: Bool
    }
}

struct TimerLiveActivityView: View {
    let context: ActivityViewContext<TimerActivityAttributes>

    var body: some View {
        VStack {
            Text(context.attributes.timerName)
                .font(.headline)

            if context.state.isPaused {
                Text("一時停止中")
                    .font(.title)
            } else {
                // 自動更新されるタイマー表示
                Text(context.state.endTime, style: .timer)
                    .font(.system(size: 48, weight: .bold, design: .monospaced))
                    .monospacedDigit()
            }
        }
    }
}
```

---

## 5. 注意点とベストプラクティス

### 制限事項

```swift
// 同時に実行できるLive Activityの最大数
// - アプリごとに1つ
// - システム全体で5つまで

// 更新頻度の制限
// - 1時間あたり約10-15回のPush更新
// - アプリ内更新は制限なし（ただしバッテリー考慮）

// データサイズ制限
// - ActivityAttributes + ContentState で約4KB以内
```

### チェックリスト

```swift
// 1. Capability追加
// - Push Notifications
// - Background Modes > Remote notifications

// 2. Info.plist
// - NSSupportsLiveActivities = YES
// - NSSupportsLiveActivitiesFrequentUpdates = YES (頻繁な更新が必要な場合)

// 3. Entitlements
// - com.apple.developer.live-activity = true
```

---

## まとめ：Live Activity実装

| 機能 | 用途 | 優先度 |
|------|------|--------|
| **ActivityKit** | Live Activity の開始・更新・終了 | 🔴 必須 |
| **ロック画面UI** | メインの情報表示 | 🔴 必須 |
| **Dynamic Island** | iPhone 14 Pro+ 専用UI | 🟡 推奨 |
| **Push更新** | サーバーからのリアルタイム更新 | 🟡 推奨 |
| **タイマー表示** | 自動更新カウントダウン | 🟢 参考 |

### 適したユースケース

- 配送・配達追跡
- スポーツのライブスコア
- タイマー・ストップウォッチ
- 音楽・ポッドキャスト再生
- 乗車アプリ（Uber風）
- フライト情報
