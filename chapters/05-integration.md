# 第5章：機能統合の実践

> 執筆者：葉林野
> 最終更新：2026-04-27

## この章で学ぶこと

この章では、これまでに学んだ写真選択、地図表示、位置情報取得、データ保存の機能を組み合わせて、「フォトマップ」アプリを作る方法を学ぶ。  
写真を選んだときの現在地を一緒に保存し、その記録を地図上や一覧画面で確認できるようにする。  
複数の機能を一つのアプリとしてまとめる流れを理解することが大切である。

## 模範コードの全体像

```swift
// ============================================
// 第5章：写真 + 地図 + データ保存の統合アプリ
// ============================================
// 写真を選択し、選択時の現在地を地図上に記録する
// 「フォトマップ」アプリです。
// 第2〜4章で学んだ技術を組み合わせて使います。
//
// 【注意】Info.plist に以下のキーを追加してください：
//   - NSLocationWhenInUseUsageDescription
//   - NSPhotoLibraryAddUsageDescription
// ============================================

import SwiftUI
import SwiftData
import MapKit
import PhotosUI

// MARK: - データモデル

@Model
class PhotoRecord {
    var title: String
    var memo: String
    var latitude: Double
    var longitude: Double
    var imageData: Data?
    var createdAt: Date

    init(title: String, memo: String = "", latitude: Double, longitude: Double, imageData: Data? = nil) {
        self.title = title
        self.memo = memo
        self.latitude = latitude
        self.longitude = longitude
        self.imageData = imageData
        self.createdAt = .now
    }

    var coordinate: CLLocationCoordinate2D {
        CLLocationCoordinate2D(latitude: latitude, longitude: longitude)
    }

    var uiImage: UIImage? {
        guard let data = imageData else { return nil }
        return UIImage(data: data)
    }
}

// MARK: - 位置情報マネージャー

@Observable
class LocationManager: NSObject, CLLocationManagerDelegate {
    private let manager = CLLocationManager()
    var currentLocation: CLLocationCoordinate2D?

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
        manager.requestWhenInUseAuthorization()
        manager.startUpdatingLocation()
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        currentLocation = locations.last?.coordinate
    }
}

// MARK: - アプリエントリポイント
// ※ App ファイルに以下を記述：
//
// @main
// struct PhotoMapApp: App {
//     var body: some Scene {
//         WindowGroup {
//             ContentView()
//         }
//         .modelContainer(for: PhotoRecord.self)
//     }
// }

// MARK: - メインビュー（タブ構成）

struct ContentView: View {
    var body: some View {
        TabView {
            MapTab()
                .tabItem {
                    Label("マップ", systemImage: "map")
                }

            ListTab()
                .tabItem {
                    Label("一覧", systemImage: "list.bullet")
                }
        }
    }
}

// MARK: - マップタブ

struct MapTab: View {
    @Environment(\.modelContext) private var modelContext
    @Query private var records: [PhotoRecord]
    @State private var locationManager = LocationManager()
    @State private var cameraPosition: MapCameraPosition = .automatic
    @State private var isShowingAddSheet = false
    @State private var selectedRecord: PhotoRecord?

    var body: some View {
        NavigationStack {
            ZStack(alignment: .bottomTrailing) {
                Map(position: $cameraPosition) {
                    UserAnnotation()

                    ForEach(records) { record in
                        Annotation(record.title, coordinate: record.coordinate) {
                            Button {
                                selectedRecord = record
                            } label: {
                                if let uiImage = record.uiImage {
                                    Image(uiImage: uiImage)
                                        .resizable()
                                        .aspectRatio(contentMode: .fill)
                                        .frame(width: 40, height: 40)
                                        .clipShape(Circle())
                                        .overlay(Circle().stroke(.white, lineWidth: 2))
                                        .shadow(radius: 2)
                                } else {
                                    Image(systemName: "photo.circle.fill")
                                        .font(.title)
                                        .foregroundStyle(.blue)
                                }
                            }
                        }
                    }
                }
                .mapControls {
                    MapUserLocationButton()
                }

                // 追加ボタン
                Button {
                    isShowingAddSheet = true
                } label: {
                    Image(systemName: "plus.circle.fill")
                        .font(.system(size: 56))
                        .foregroundStyle(.blue)
                        .background(Circle().fill(.white))
                        .shadow(radius: 4)
                }
                .padding(24)
            }
            .navigationTitle("フォトマップ")
            .sheet(isPresented: $isShowingAddSheet) {
                AddRecordView(locationManager: locationManager)
            }
            .sheet(item: $selectedRecord) { record in
                RecordDetailView(record: record)
            }
        }
    }
}

// MARK: - 一覧タブ

struct ListTab: View {
    @Environment(\.modelContext) private var modelContext
    @Query(sort: \PhotoRecord.createdAt, order: .reverse) private var records: [PhotoRecord]

    var body: some View {
        NavigationStack {
            List {
                ForEach(records) { record in
                    HStack(spacing: 12) {
                        if let uiImage = record.uiImage {
                            Image(uiImage: uiImage)
                                .resizable()
                                .aspectRatio(contentMode: .fill)
                                .frame(width: 50, height: 50)
                                .clipShape(RoundedRectangle(cornerRadius: 8))
                        }

                        VStack(alignment: .leading, spacing: 4) {
                            Text(record.title)
                                .font(.headline)
                            Text(record.createdAt, style: .date)
                                .font(.caption)
                                .foregroundStyle(.secondary)
                        }
                    }
                }
                .onDelete { offsets in
                    for index in offsets {
                        modelContext.delete(records[index])
                    }
                }
            }
            .navigationTitle("記録一覧")
        }
    }
}

// MARK: - 記録追加画面

struct AddRecordView: View {
    @Environment(\.modelContext) private var modelContext
    @Environment(\.dismiss) private var dismiss
    let locationManager: LocationManager

    @State private var title = ""
    @State private var memo = ""
    @State private var selectedItem: PhotosPickerItem?
    @State private var selectedImageData: Data?
    @State private var previewImage: Image?

    var body: some View {
        NavigationStack {
            Form {
                Section("写真") {
                    if let image = previewImage {
                        image
                            .resizable()
                            .aspectRatio(contentMode: .fit)
                            .frame(maxHeight: 200)
                            .clipShape(RoundedRectangle(cornerRadius: 8))
                    }

                    PhotosPicker(selection: $selectedItem, matching: .images) {
                        Label("写真を選択", systemImage: "photo")
                    }
                }

                Section("情報") {
                    TextField("タイトル", text: $title)
                    TextField("メモ（任意）", text: $memo, axis: .vertical)
                        .lineLimit(3...6)
                }

                Section("位置情報") {
                    if let location = locationManager.currentLocation {
                        Text("緯度: \(location.latitude, specifier: "%.4f")")
                        Text("経度: \(location.longitude, specifier: "%.4f")")
                    } else {
                        Text("位置情報を取得中...")
                            .foregroundStyle(.secondary)
                    }
                }
            }
            .navigationTitle("新しい記録")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("キャンセル") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("保存") {
                        saveRecord()
                    }
                    .disabled(title.isEmpty || locationManager.currentLocation == nil)
                }
            }
            .onChange(of: selectedItem) { _, newItem in
                Task {
                    if let data = try? await newItem?.loadTransferable(type: Data.self) {
                        selectedImageData = data
                        if let uiImage = UIImage(data: data) {
                            previewImage = Image(uiImage: uiImage)
                        }
                    }
                }
            }
        }
    }

    func saveRecord() {
        guard let location = locationManager.currentLocation else { return }

        let record = PhotoRecord(
            title: title,
            memo: memo,
            latitude: location.latitude,
            longitude: location.longitude,
            imageData: selectedImageData
        )
        modelContext.insert(record)
        dismiss()
    }
}

// MARK: - 記録詳細画面

struct RecordDetailView: View {
    let record: PhotoRecord

    var body: some View {
        ScrollView {
            VStack(spacing: 16) {
                if let uiImage = record.uiImage {
                    Image(uiImage: uiImage)
                        .resizable()
                        .aspectRatio(contentMode: .fit)
                        .clipShape(RoundedRectangle(cornerRadius: 12))
                }

                VStack(alignment: .leading, spacing: 8) {
                    Text(record.title)
                        .font(.title2)
                        .bold()

                    if !record.memo.isEmpty {
                        Text(record.memo)
                            .foregroundStyle(.secondary)
                    }

                    Text(record.createdAt, style: .date)
                        .font(.caption)
                        .foregroundStyle(.tertiary)
                }
                .frame(maxWidth: .infinity, alignment: .leading)

                // ミニマップ
                Map {
                    Marker(record.title, coordinate: record.coordinate)
                }
                .frame(height: 200)
                .clipShape(RoundedRectangle(cornerRadius: 12))
            }
            .padding()
        }
    }
}

#Preview {
    ContentView()
        .modelContainer(for: PhotoRecord.self, inMemory: true)
}
```

**このアプリは何をするものか：**

このアプリは、写真と位置情報を一緒に保存する「フォトマップ」アプリである。
ユーザーは写真を選択し、タイトルやメモを入力して保存する。保存すると、その時の現在地の緯度と経度も一緒に記録される。
保存した記録は地図上に写真付きのピンとして表示され、一覧画面でも確認できる。地図上の写真をタップすると、写真・タイトル・メモ・日付・ミニマップを表示する詳細画面を見ることができる。

## コードの詳細解説

### データモデルの設計

```swift
@Model
class PhotoRecord {
    var title: String
    var memo: String
    var latitude: Double
    var longitude: Double
    var imageData: Data?
    var createdAt: Date

    init(title: String, memo: String = "", latitude: Double, longitude: Double, imageData: Data? = nil) {
        self.title = title
        self.memo = memo
        self.latitude = latitude
        self.longitude = longitude
        self.imageData = imageData
        self.createdAt = .now
    }

    var coordinate: CLLocationCoordinate2D {
        CLLocationCoordinate2D(latitude: latitude, longitude: longitude)
    }

    var uiImage: UIImage? {
        guard let data = imageData else { return nil }
        return UIImage(data: data)
    }
}
```

**何をしているか：**

この部分では、写真記録を保存するためのデータモデルを定義している。
title はタイトル、memo はメモ、latitude と longitude は位置情報、imageData は写真データ、createdAt は作成日時を表している。
また、coordinate では緯度と経度を地図で使える座標に変換している。uiImage では保存された画像データを画面に表示できる画像に変換している。

**なぜこう書くのか：**

SwiftDataでデータを保存するために、クラスに @Model を付けている。
画像はそのままではSwiftDataに保存しにくいため、Data 型として保存している。地図で表示するときは CLLocationCoordinate2D が必要なので、latitude と longitude から座標を作る計算プロパティを用意している。

**もしこう書かなかったら：**

@Model を付けなければ、SwiftDataでこのデータを保存できない。
また、coordinate がなければ、地図上に記録を表示するときに毎回 CLLocationCoordinate2D を作る必要があり、コードが読みにくくなる。
uiImage がなければ、保存した画像データを画面に表示する処理を別の場所に書く必要がある。

---

### タブ構成の設計

```swift
struct ContentView: View {
    var body: some View {
        TabView {
            MapTab()
                .tabItem {
                    Label("マップ", systemImage: "map")
                }

            ListTab()
                .tabItem {
                    Label("一覧", systemImage: "list.bullet")
                }
        }
    }
}
```

**何をしているか：**

この部分では、アプリの画面をタブで分けている。
MapTab は地図画面、ListTab は保存した記録の一覧画面である。
ユーザーは画面下のタブを押すことで、地図表示と一覧表示を切り替えることができる。

**なぜこう書くのか：**

地図を見る機能と一覧を見る機能は役割が違うため、タブで分けると使いやすい。
TabView を使うことで、複数の画面を簡単に切り替えられる。
また、Label を使うことで、文字とアイコンを一緒に表示できる。

**もしこう書かなかったら：**

タブ構成にしない場合、地図画面と一覧画面を一つの画面にまとめる必要があり、画面が複雑になる。
また、ユーザーが保存した記録を確認しにくくなる可能性がある。

---

### カメラと位置情報の連携

```swift
@Observable
class LocationManager: NSObject, CLLocationManagerDelegate {
    private let manager = CLLocationManager()
    var currentLocation: CLLocationCoordinate2D?

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
        manager.requestWhenInUseAuthorization()
        manager.startUpdatingLocation()
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        currentLocation = locations.last?.coordinate
    }
}

PhotosPicker(selection: $selectedItem, matching: .images) {
    Label("写真を選択", systemImage: "photo")
}
```

**何をしているか：**

LocationManager は現在地を取得するためのクラスである。
アプリが起動すると、位置情報の使用許可を求め、現在地の取得を開始する。
PhotosPicker は写真ライブラリから画像を選択するために使っている。
保存時には、選択した写真と現在地の緯度・経度を一緒に記録する。

**なぜこう書くのか：**

写真だけを保存しても、どこで記録した写真なのか分からない。
位置情報と一緒に保存することで、地図上に写真の場所を表示できる。
また、CLLocationManagerDelegate を使うことで、位置情報が更新されたタイミングで現在地を受け取ることができる。

**もしこう書かなかったら：**

位置情報を取得しなければ、写真を地図上に表示できない。
また、位置情報の使用許可を求めなければ、iPhoneでは現在地を取得できない。
PhotosPicker がなければ、ユーザーが写真を選択する機能を実装できない。

---

### SwiftDataでの画像保存

```swift
@Environment(\.modelContext) private var modelContext

@State private var selectedImageData: Data?

.onChange(of: selectedItem) { _, newItem in
    Task {
        if let data = try? await newItem?.loadTransferable(type: Data.self) {
            selectedImageData = data
            if let uiImage = UIImage(data: data) {
                previewImage = Image(uiImage: uiImage)
            }
        }
    }
}

func saveRecord() {
    guard let location = locationManager.currentLocation else { return }

    let record = PhotoRecord(
        title: title,
        memo: memo,
        latitude: location.latitude,
        longitude: location.longitude,
        imageData: selectedImageData
    )
    modelContext.insert(record)
    dismiss()
}
```

**何をしているか：**

写真を選択すると、その画像を Data 型として読み込んでいる。
その後、タイトル、メモ、緯度、経度、画像データを使って PhotoRecord を作成し、modelContext.insert(record) でSwiftDataに保存している。

**なぜこう書くのか：**

SwiftDataには画像そのものではなく、画像を Data として保存する方法が使いやすい。
また、modelContext を使うことで、SwiftDataのデータ追加や削除ができる。
dismiss() を使うことで、保存後に追加画面を閉じることができる。

**もしこう書かなかったら：**

画像データを Data として保存しなければ、アプリを閉じた後に写真を再表示できない。
また、modelContext.insert(record) を書かなければ、新しい記録はSwiftDataに保存されない。
その場合、画面を閉じたりアプリを再起動したりすると、記録が残らない。

---

## 新しく学んだSwiftの文法・API

| 項目                      | 説明                         | 使用例                                                               |
| ----------------------- | -------------------------- | ----------------------------------------------------------------- |
| `@Model`                | SwiftDataで保存するデータモデルを定義する  | `@Model class PhotoRecord { ... }`                                |
| `@Query`                | SwiftDataに保存されたデータを自動で取得する | `@Query private var records: [PhotoRecord]`                       |
| `modelContext.insert()` | SwiftDataに新しいデータを保存する      | `modelContext.insert(record)`                                     |
| `modelContext.delete()` | SwiftDataからデータを削除する        | `modelContext.delete(records[index])`                             |
| `TabView`               | 複数の画面をタブで切り替える             | `TabView { MapTab(); ListTab() }`                                 |
| `Map`                   | 地図を表示するSwiftUIのビュー         | `Map(position: $cameraPosition) { ... }`                          |
| `Annotation`            | 地図上の指定した場所に表示するマーク         | `Annotation(record.title, coordinate: record.coordinate) { ... }` |
| `UserAnnotation`        | 地図上にユーザーの現在地を表示する          | `UserAnnotation()`                                                |
| `CLLocationManager`     | GPS位置情報を取得するAPI            | `manager.startUpdatingLocation()`                                 |
| `PhotosPicker`          | 写真ライブラリから写真を選択する           | `PhotosPicker(selection: $selectedItem, matching: .images)`       |
| `Data`                  | 画像などのバイナリデータを扱う型           | `var imageData: Data?`                                            |
| `sheet`                 | 別画面を下から表示する                | `.sheet(isPresented: $isShowingAddSheet) { ... }`                 |
| `dismiss()`             | 表示中の画面を閉じる                 | `dismiss()`                                                       |


## 自分の実験メモ

**実験1：**
- やったこと：写真を選択せずに、タイトルだけ入力して保存できるか試した。
- 結果：写真がなくても保存はできた。地図上では写真の代わりにシステムアイコンが表示された。
- わかったこと：imageData は Data? なので、写真がない場合は nil として扱える。画像がなくても記録自体は保存できる。

**実験2：**
- やったこと：タイトルを空のまま保存ボタンを押せるか試した。
- 結果：保存ボタンが無効になり、押せなかった。
- わかったこと：.disabled(title.isEmpty || locationManager.currentLocation == nil) によって、タイトルが空の場合や位置情報が取得できていない場合は保存できないようになっている。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**

   @Model は何のために使うのか。
   
   **得られた理解：**

   @Model を付けることで、そのクラスをSwiftDataで保存できるデータとして扱えることが分かった。普通のクラスではなく、データベースに保存するためのモデルになる。

2. **質問：**

   なぜ画像を UIImage ではなく Data として保存するのか。
   
   **得られた理解：**

   SwiftDataでは画像そのものよりも、画像をバイナリデータである Data として保存する方が扱いやすいことが分かった。表示するときに UIImage(data:) で画像に戻している。

3. **質問：**

   @Query と modelContext の違いは何か。
   
   **得られた理解：**

   @Query は保存されたデータを読み込むために使い、modelContext はデータを追加・削除するために使うことが分かった。つまり、読む処理と変更する処理で役割が違う。

## この章のまとめ

この章では、写真選択、位置情報、地図表示、SwiftDataによる保存を組み合わせて、一つのアプリを作る方法を学んだ。
特に大切だと思ったのは、各機能を別々に作るだけでなく、それらをどのように連携させるかである。
写真を選ぶ、現在地を取得する、データとして保存する、地図や一覧に表示するという流れを理解することで、実用的なアプリの作り方が少し分かった。
また、SwiftDataの @Model、@Query、modelContext.insert()、modelContext.delete() の役割を整理できた。
今後、自分でアプリを作るときも、データモデルを先に考えてから画面や機能を作ることが重要だと感じた。
