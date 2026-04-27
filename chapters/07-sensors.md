# 第7章：センサーの活用

> 執筆者：葉林野
> 最終更新：2026-04-27

## この章で学ぶこと

この章では、iPhoneに搭載されているセンサーを使って、端末の傾きや歩数、移動速度などを取得する方法を学ぶ。
CoreMotionを使った水平器アプリと、CMPedometer・CoreLocationを使った活動トラッカーを通して、センサーデータを画面に表示する実装を理解する。

## 模範コードの全体像

```swift
// ============================================
// 第7章（応用）：歩数計・移動距離トラッカー
// ============================================
// CoreMotion（歩数計）とCoreLocation（移動距離）を
// 組み合わせて、今日の活動を記録するアプリです。
//
// 【注意】Info.plist に以下のキーを追加してください：
//   - NSMotionUsageDescription
//     値: "歩数を計測するためにモーションセンサーを使用します"
//   - NSLocationWhenInUseUsageDescription
//     値: "移動距離を計測するために位置情報を使用します"
// ============================================

import SwiftUI
import CoreMotion
import CoreLocation

// MARK: - 活動トラッカー

@Observable
class ActivityTracker: NSObject, CLLocationManagerDelegate {
    // 歩数関連
    private let pedometer = CMPedometer()
    var stepCount: Int = 0
    var distance: Double = 0     // メートル
    var isPedometerAvailable: Bool = false

    // 位置関連
    private let locationManager = CLLocationManager()
    var currentSpeed: Double = 0  // m/s
    var locations: [CLLocationCoordinate2D] = []

    // 状態
    var isTracking: Bool = false
    var startTime: Date?

    override init() {
        super.init()
        locationManager.delegate = self
        locationManager.desiredAccuracy = kCLLocationAccuracyBest
        locationManager.requestWhenInUseAuthorization()
        isPedometerAvailable = CMPedometer.isStepCountingAvailable()
    }

    func startTracking() {
        isTracking = true
        startTime = Date()
        stepCount = 0
        distance = 0
        locations = []

        // 歩数計の開始
        if isPedometerAvailable {
            pedometer.startUpdates(from: Date()) { [weak self] data, error in
                guard let self = self, let data = data else { return }

                DispatchQueue.main.async {
                    self.stepCount = data.numberOfSteps.intValue
                    if let dist = data.distance {
                        self.distance = dist.doubleValue
                    }
                }
            }
        }

        // 位置情報の開始
        locationManager.startUpdatingLocation()
    }

    func stopTracking() {
        isTracking = false
        pedometer.stopUpdates()
        locationManager.stopUpdatingLocation()
    }

    // MARK: - CLLocationManagerDelegate

    func locationManager(_ manager: CLLocationManager, didUpdateLocations newLocations: [CLLocation]) {
        guard let location = newLocations.last else { return }
        currentSpeed = max(0, location.speed)
        locations.append(location.coordinate)
    }

    // MARK: - 計算プロパティ

    var distanceInKm: Double {
        distance / 1000
    }

    var speedInKmh: Double {
        currentSpeed * 3.6
    }

    var caloriesBurned: Double {
        // 簡易計算：歩数 × 0.04 kcal（目安）
        Double(stepCount) * 0.04
    }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var tracker = ActivityTracker()
    @State private var timer = Timer.publish(every: 1, on: .main, in: .common).autoconnect()
    @State private var now: Date = .now

    // 経過時間（startTime と now の差分から算出）
    // 次の Timer tick まで now が古いままになるため、max(0, ...) でガード
    private var elapsedTime: TimeInterval {
        guard let start = tracker.startTime else { return 0 }
        return max(0, now.timeIntervalSince(start))
    }

    var body: some View {
        NavigationStack {
            ScrollView {
                VStack(spacing: 20) {
                    // タイマー表示
                    timerSection

                    // メイン統計
                    statsGrid

                    // スタート/ストップボタン
                    controlButton

                    // 速度メーター
                    if tracker.isTracking {
                        SpeedMeter(speed: tracker.speedInKmh)
                    }
                }
                .padding()
            }
            .navigationTitle("活動トラッカー")
            .onReceive(timer) { date in
                // 1秒ごとに now を更新することで、経過時間表示が再描画される
                now = date
            }
        }
    }

    // MARK: - タイマーセクション

    private var timerSection: some View {
        VStack(spacing: 4) {
            Text(formatTime(elapsedTime))
                .font(.system(size: 48, weight: .thin, design: .monospaced))

            if tracker.isTracking {
                Text("計測中")
                    .font(.caption)
                    .foregroundStyle(.green)
            }
        }
        .padding()
    }

    // MARK: - 統計グリッド

    private var statsGrid: some View {
        LazyVGrid(columns: [
            GridItem(.flexible()),
            GridItem(.flexible()),
        ], spacing: 16) {
            StatCard(
                icon: "figure.walk",
                value: "\(tracker.stepCount)",
                unit: "歩",
                color: .blue
            )
            StatCard(
                icon: "map",
                value: String(format: "%.2f", tracker.distanceInKm),
                unit: "km",
                color: .green
            )
            StatCard(
                icon: "flame",
                value: String(format: "%.0f", tracker.caloriesBurned),
                unit: "kcal",
                color: .orange
            )
            StatCard(
                icon: "speedometer",
                value: String(format: "%.1f", tracker.speedInKmh),
                unit: "km/h",
                color: .purple
            )
        }
    }

    // MARK: - コントロールボタン

    private var controlButton: some View {
        Button {
            if tracker.isTracking {
                tracker.stopTracking()
            } else {
                tracker.startTracking()
            }
        } label: {
            HStack {
                Image(systemName: tracker.isTracking ? "stop.fill" : "play.fill")
                Text(tracker.isTracking ? "ストップ" : "スタート")
            }
            .font(.title3)
            .frame(maxWidth: .infinity)
            .padding()
            .background(tracker.isTracking ? Color.red : Color.green)
            .foregroundStyle(.white)
            .clipShape(RoundedRectangle(cornerRadius: 16))
        }
    }

    // MARK: - 時間フォーマット

    func formatTime(_ interval: TimeInterval) -> String {
        let hours = Int(interval) / 3600
        let minutes = Int(interval) / 60 % 60
        let seconds = Int(interval) % 60
        return String(format: "%02d:%02d:%02d", hours, minutes, seconds)
    }
}

// MARK: - 統計カード

struct StatCard: View {
    let icon: String
    let value: String
    let unit: String
    let color: Color

    var body: some View {
        VStack(spacing: 8) {
            Image(systemName: icon)
                .font(.title2)
                .foregroundStyle(color)

            Text(value)
                .font(.title)
                .bold()

            Text(unit)
                .font(.caption)
                .foregroundStyle(.secondary)
        }
        .frame(maxWidth: .infinity)
        .padding()
        .background(
            RoundedRectangle(cornerRadius: 12)
                .fill(color.opacity(0.08))
        )
    }
}

// MARK: - 速度メーター

struct SpeedMeter: View {
    let speed: Double

    var body: some View {
        VStack(spacing: 8) {
            Text("現在の速度")
                .font(.caption)
                .foregroundStyle(.secondary)

            ZStack {
                Circle()
                    .trim(from: 0, to: 0.75)
                    .stroke(.gray.opacity(0.2), lineWidth: 8)
                    .rotationEffect(.degrees(135))

                Circle()
                    .trim(from: 0, to: min(speed / 15.0, 1.0) * 0.75)
                    .stroke(speedColor, style: StrokeStyle(lineWidth: 8, lineCap: .round))
                    .rotationEffect(.degrees(135))
                    .animation(.spring, value: speed)

                VStack {
                    Text(String(format: "%.1f", speed))
                        .font(.system(size: 32, weight: .bold, design: .monospaced))
                    Text("km/h")
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
            }
            .frame(width: 150, height: 150)
        }
        .padding()
    }

    var speedColor: Color {
        if speed < 4 { return .green }
        if speed < 8 { return .orange }
        return .red
    }
}

#Preview {
    ContentView()
}
```

**このアプリは何をするものか：**

このアプリは、iPhoneのセンサーを使って、端末の状態やユーザーの活動を記録するアプリである。
水平器アプリでは、端末の前後・左右の傾きを取得し、画面上の丸いバブルを動かして水平かどうかを表示する。
活動トラッカーでは、歩数、移動距離、消費カロリー、現在の速度、経過時間を表示する。

## コードの詳細解説

### CoreMotionの基本（CMMotionManager）

```swift
import CoreMotion

@Observable
class MotionManager {
    private let motionManager = CMMotionManager()

    var pitch: Double = 0
    var roll: Double = 0
    var yaw: Double = 0
    var isAvailable: Bool

    init() {
        isAvailable = motionManager.isDeviceMotionAvailable
    }

    func startUpdates() {
        guard isAvailable else { return }

        motionManager.deviceMotionUpdateInterval = 1.0 / 60.0

        motionManager.startDeviceMotionUpdates(to: .main) { [weak self] motion, error in
            guard let self = self, let motion = motion else { return }

            self.pitch = motion.attitude.pitch
            self.roll = motion.attitude.roll
            self.yaw = motion.attitude.yaw
        }
    }

    func stopUpdates() {
        motionManager.stopDeviceMotionUpdates()
    }
}
```

**何をしているか：**

この部分では、CMMotionManagerを使ってiPhoneの姿勢情報を取得している。
startDeviceMotionUpdatesを使うことで、端末の傾きが変わるたびにpitch、roll、yawの値が更新される。
また、isDeviceMotionAvailableで、端末がモーションセンサーを利用できるかどうかを確認している。

**なぜこう書くのか：**

センサーはすべての環境で使えるわけではないため、最初にisAvailableを確認する必要がある。
また、画面の更新はメインスレッドで行う必要があるため、startDeviceMotionUpdates(to: .main)と書いている。
deviceMotionUpdateInterval = 1.0 / 60.0にすることで、1秒間に約60回データを更新し、バブルの動きがなめらかになる。

**もしこう書かなかったら：**

isAvailableを確認しないと、センサーが使えない環境で正しく動作しない可能性がある。
また、stopUpdates()を書かないと、画面を閉じたあともセンサーが動き続け、バッテリーを消費してしまう。

---

### デバイスの姿勢データ（pitch/roll/yaw）

```swift
self.pitch = motion.attitude.pitch
self.roll = motion.attitude.roll
self.yaw = motion.attitude.yaw

private var xOffset: CGFloat {
    CGFloat(roll) * maxOffset
}

private var yOffset: CGFloat {
    CGFloat(pitch) * maxOffset
}

private var isLevel: Bool {
    abs(pitch) < 0.03 && abs(roll) < 0.03
}
```

**何をしているか：**

この部分では、端末の姿勢データを取得している。
pitchは前後の傾き、rollは左右の傾き、yawは水平方向の回転を表している。
水平器アプリでは、主にpitchとrollを使って、画面上のバブルを動かしている。

**なぜこう書くのか：**

水平器では、端末がどちらに傾いているかを視覚的に表示する必要がある。
そのため、左右の傾きであるrollをX方向の移動に使い、前後の傾きであるpitchをY方向の移動に使っている。
このコードでは、pitchとrollがどちらも0に近い場合、端末がほぼ水平であると判断している。

**もしこう書かなかったら：**

pitchやrollを使わなければ、端末の傾きを画面に反映できない。
また、isLevelのような判定がないと、水平になったときに色を変えたり、「水平!」と表示したりすることができない。

---

### 歩数計（CMPedometer）

```swift
import CoreMotion

private let pedometer = CMPedometer()
var stepCount: Int = 0
var distance: Double = 0
var isPedometerAvailable: Bool = false

isPedometerAvailable = CMPedometer.isStepCountingAvailable()

if isPedometerAvailable {
    pedometer.startUpdates(from: Date()) { [weak self] data, error in
        guard let self = self, let data = data else { return }

        DispatchQueue.main.async {
            self.stepCount = data.numberOfSteps.intValue
            if let dist = data.distance {
                self.distance = dist.doubleValue
            }
        }
    }
}
```

**何をしているか：**

この部分では、CMPedometerを使って歩数や移動距離を取得する準備をしている。
stepCountには歩数、distanceには移動距離が入る。
isPedometerAvailableは、端末が歩数計機能を使えるかどうかを表している。
このコードで、歩数計が利用可能かどうかを確認している。

**なぜこう書くのか：**

歩数計も端末によって利用できない場合があるため、最初にisStepCountingAvailable()で確認している。
また、startUpdates(from: Date())を使うことで、スタートボタンを押した時点から歩数の計測を始められる。
取得したデータを画面に反映するために、DispatchQueue.main.asyncでメインスレッドに戻している。

**もしこう書かなかったら：**

歩数計が使えない端末で処理を実行してしまう可能性がある。
また、メインスレッドで画面更新をしないと、表示が正しく更新されないことがある。

---

### CoreLocationとの連携

```swift
import CoreLocation

@Observable
class ActivityTracker: NSObject, CLLocationManagerDelegate {
    private let locationManager = CLLocationManager()
    var currentSpeed: Double = 0
    var locations: [CLLocationCoordinate2D] = []

    override init() {
        super.init()
        locationManager.delegate = self
        locationManager.desiredAccuracy = kCLLocationAccuracyBest
        locationManager.requestWhenInUseAuthorization()
    }
}

locationManager.startUpdatingLocation()

func locationManager(_ manager: CLLocationManager, didUpdateLocations newLocations: [CLLocation]) {
    guard let location = newLocations.last else { return }
    currentSpeed = max(0, location.speed)
    locations.append(location.coordinate)
}
```

**何をしているか：**

この部分では、CoreLocationを使って位置情報を取得する準備をしている。
locationManager.delegate = selfとすることで、位置情報が更新されたときに、このクラスのメソッドが呼ばれるようになる。
requestWhenInUseAuthorization()では、アプリ使用中に位置情報を使う許可をユーザーに求めている。
このコードで位置情報の取得を開始する。
この部分では、最新の位置情報を取得し、現在の速度と座標を保存している。

**なぜこう書くのか：**

活動トラッカーでは、歩数だけでなく、現在の速度や移動経路も取得したい。
そのため、CMPedometerだけでなくCoreLocationも組み合わせて使っている。
また、location.speedは負の値になる場合があるため、max(0, location.speed)として、速度が0未満にならないようにしている。

**もしこう書かなかったら：**

位置情報の許可を求めなければ、速度や位置を取得できない。
また、delegateを設定しないと、位置情報が更新されてもdidUpdateLocationsが呼ばれない。
その結果、速度メーターが正しく表示されない。

---

## 新しく学んだSwiftの文法・API

| 項目                          | 説明                           | 使用例                                                                                                 |
| --------------------------- | ---------------------------- | --------------------------------------------------------------------------------------------------- |
| `CMMotionManager`           | 端末の動きや姿勢のデータを取得するためのクラス      | `motionManager.startDeviceMotionUpdates(to: .main) { ... }`                                         |
| `motion.attitude.pitch`     | 端末の前後方向の傾きを取得する              | `self.pitch = motion.attitude.pitch`                                                                |
| `motion.attitude.roll`      | 端末の左右方向の傾きを取得する              | `self.roll = motion.attitude.roll`                                                                  |
| `motion.attitude.yaw`       | 端末の水平方向の回転を取得する              | `self.yaw = motion.attitude.yaw`                                                                    |
| `CMPedometer`               | 歩数や移動距離を取得するためのクラス           | `pedometer.startUpdates(from: Date()) { ... }`                                                      |
| `CLLocationManager`         | 位置情報や速度を取得するためのクラス           | `locationManager.startUpdatingLocation()`                                                           |
| `CLLocationManagerDelegate` | 位置情報が更新されたときの処理を書くための仕組み     | `func locationManager(_ manager: CLLocationManager, didUpdateLocations newLocations: [CLLocation])` |
| `Timer.publish`             | 一定時間ごとに処理を実行する               | `Timer.publish(every: 1, on: .main, in: .common)`                                                   |
| `@Observable`               | データが変わったときにSwiftUIの画面を自動更新する | `@Observable class ActivityTracker`                                                                 |
| `DispatchQueue.main.async`  | メインスレッドで画面更新を行う              | `DispatchQueue.main.async { self.stepCount = ... }`                                                 |


## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：水平器アプリで、iPhoneを左右に傾けて、画面上のバブルの動きを確認した。
- 結果：端末を右や左に傾けると、バブルもそれに合わせて移動した。水平に近づくと、バブルが緑色になり、「水平!」と表示された。
- わかったこと：rollとpitchの値を使うことで、端末の傾きを画面上の位置として表現できることが分かった。

**実験2：**
- やったこと：活動トラッカーでスタートボタンを押して、歩数と時間の変化を確認した。
- 結果：計測を開始すると、経過時間が1秒ごとに増え、歩くと歩数も増えた。ストップボタンを押すと、歩数や位置情報の更新が止まった。
- わかったこと：CMPedometerを使うと歩数を取得でき、Timerを使うと経過時間をリアルタイムで表示できることが分かった。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**

   pitch、roll、yawはそれぞれ何を表していますか？
   
   **得られた理解：**

   pitchは前後の傾き、rollは左右の傾き、yawは水平方向の回転を表している。水平器では主にpitchとrollを使ってバブルの位置を決めている。

2. **質問：**

   なぜstartDeviceMotionUpdates(to: .main)のように.mainを指定するのですか？
   
   **得られた理解：**

   SwiftUIの画面更新はメインスレッドで行う必要があるため、センサーデータを取得したあと画面に反映する処理を安全に行うために.mainを指定している。

3. **質問：**

   CMPedometerとCoreLocationは何が違いますか？
   
   **得られた理解：**

   CMPedometerは歩数や歩行距離を取得するために使い、CoreLocationはGPSなどを使って位置情報や速度を取得するために使う。活動トラッカーでは、この2つを組み合わせることで、歩数・距離・速度を表示している。

## この章のまとめ

この章では、iPhoneに搭載されているセンサーを使って、端末の傾きやユーザーの活動情報を取得する方法を学んだ。
CoreMotionを使うと、pitch、roll、yawのような姿勢データを取得でき、水平器のようなアプリを作ることができる。
また、CMPedometerを使うと歩数や移動距離を取得でき、CoreLocationを使うと位置情報や速度を取得できる。
特に重要だと思ったことは、センサーを使うときには、利用可能かどうかの確認、権限設定、開始と停止の処理が必要だということである。
センサーは便利だが、バッテリーを使うため、必要なときだけ開始し、不要になったら停止することが大切だと分かった。
