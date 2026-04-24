# 第2章：地図アプリの基本

> 執筆者：葉林野
> 最終更新：2026-04-24

## この章で学ぶこと

この章では、SwiftUIとMapKitを使って、アプリ内に地図を表示する方法を学ぶ。
また、東京の観光スポットをデータとして用意し、地図上にマーカーを表示したり、カテゴリごとに表示・非表示を切り替えたりする方法を学ぶ。

## 模範コードの全体像

```swift
// ============================================
// 第2章（基本）：MapKitで地図を表示するアプリ
// ============================================
// 東京の観光スポットを地図上にマーカーで表示します。
// マーカーをタップすると詳細情報が表示されます。
// ============================================

import SwiftUI
import MapKit

// MARK: - データモデル

struct Landmark: Identifiable {
    let id = UUID()
    let name: String
    let description: String
    let coordinate: CLLocationCoordinate2D
    let category: Category

    enum Category: String, CaseIterable {
        case temple = "寺社"
        case tower = "タワー"
        case park = "公園"

        var iconName: String {
            switch self {
            case .temple: return "building.columns"
            case .tower: return "antenna.radiowaves.left.and.right"
            case .park: return "leaf"
            }
        }

        var color: Color {
            switch self {
            case .temple: return .red
            case .tower: return .blue
            case .park: return .green
            }
        }
    }
}

// MARK: - サンプルデータ

extension Landmark {
    static let sampleData: [Landmark] = [
        Landmark(
            name: "浅草寺",
            description: "東京都内最古の寺院。雷門が有名。",
            coordinate: CLLocationCoordinate2D(latitude: 35.7148, longitude: 139.7967),
            category: .temple
        ),
        Landmark(
            name: "東京タワー",
            description: "1958年に完成した高さ333mの電波塔。",
            coordinate: CLLocationCoordinate2D(latitude: 35.6586, longitude: 139.7454),
            category: .tower
        ),
        Landmark(
            name: "東京スカイツリー",
            description: "高さ634mの世界一高い自立式電波塔。",
            coordinate: CLLocationCoordinate2D(latitude: 35.7101, longitude: 139.8107),
            category: .tower
        ),
        Landmark(
            name: "明治神宮",
            description: "明治天皇と昭憲皇太后を祀る神社。",
            coordinate: CLLocationCoordinate2D(latitude: 35.6764, longitude: 139.6993),
            category: .temple
        ),
        Landmark(
            name: "上野恩賜公園",
            description: "美術館や動物園がある広大な公園。",
            coordinate: CLLocationCoordinate2D(latitude: 35.7146, longitude: 139.7732),
            category: .park
        ),
        Landmark(
            name: "新宿御苑",
            description: "都心にある広さ58.3ヘクタールの庭園。",
            coordinate: CLLocationCoordinate2D(latitude: 35.6852, longitude: 139.7100),
            category: .park
        ),
    ]
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var cameraPosition: MapCameraPosition = .region(
        MKCoordinateRegion(
            center: CLLocationCoordinate2D(latitude: 35.6812, longitude: 139.7671),
            span: MKCoordinateSpan(latitudeDelta: 0.08, longitudeDelta: 0.08)
        )
    )
    @State private var selectedLandmark: Landmark?
    @State private var selectedCategories: Set<Landmark.Category> = Set(Landmark.Category.allCases)

    var filteredLandmarks: [Landmark] {
        Landmark.sampleData.filter { selectedCategories.contains($0.category) }
    }

    var body: some View {
        ZStack(alignment: .bottom) {
            Map(position: $cameraPosition) {
                ForEach(filteredLandmarks) { landmark in
                    Marker(
                        landmark.name,
                        systemImage: landmark.category.iconName,
                        coordinate: landmark.coordinate
                    )
                    .tint(landmark.category.color)
                }
            }
            .mapStyle(.standard(elevation: .realistic))

            VStack(spacing: 8) {
                if let landmark = selectedLandmark {
                    LandmarkCard(landmark: landmark)
                        .transition(.move(edge: .bottom))
                }

                CategoryFilter(selectedCategories: $selectedCategories)
            }
            .padding()
        }
        .onMapCameraChange { context in
        }
    }
}

// MARK: - カテゴリフィルター

struct CategoryFilter: View {
    @Binding var selectedCategories: Set<Landmark.Category>

    var body: some View {
        HStack(spacing: 8) {
            ForEach(Landmark.Category.allCases, id: \.self) { category in
                Button {
                    if selectedCategories.contains(category) {
                        selectedCategories.remove(category)
                    } else {
                        selectedCategories.insert(category)
                    }
                } label: {
                    HStack(spacing: 4) {
                        Image(systemName: category.iconName)
                        Text(category.rawValue)
                    }
                    .font(.caption)
                    .padding(.horizontal, 10)
                    .padding(.vertical, 6)
                    .background(
                        selectedCategories.contains(category)
                            ? category.color.opacity(0.2)
                            : Color.gray.opacity(0.1)
                    )
                    .foregroundStyle(
                        selectedCategories.contains(category)
                            ? category.color
                            : .gray
                    )
                    .clipShape(Capsule())
                }
            }
        }
        .padding(8)
        .background(.ultraThinMaterial)
        .clipShape(RoundedRectangle(cornerRadius: 16))
    }
}

// MARK: - ランドマーク詳細カード

struct LandmarkCard: View {
    let landmark: Landmark

    var body: some View {
        VStack(alignment: .leading, spacing: 6) {
            HStack {
                Image(systemName: landmark.category.iconName)
                    .foregroundStyle(landmark.category.color)
                Text(landmark.name)
                    .font(.headline)
                Spacer()
            }
            Text(landmark.description)
                .font(.caption)
                .foregroundStyle(.secondary)
        }
        .padding()
        .background(.ultraThinMaterial)
        .clipShape(RoundedRectangle(cornerRadius: 12))
    }
}

#Preview {
    ContentView()
}
```

**このアプリは何をするものか：**

このアプリは、東京の観光スポットを地図上に表示する地図アプリである。
浅草寺、東京タワー、東京スカイツリー、明治神宮、上野恩賜公園、新宿御苑などの場所をマーカーで表示する。
また、画面下部のカテゴリボタンを押すことで、「寺社」「タワー」「公園」の表示を切り替えることができる。

## コードの詳細解説

### データモデル（ランドマーク構造体）

```swift
struct Landmark: Identifiable {
    let id = UUID()
    let name: String
    let description: String
    let coordinate: CLLocationCoordinate2D
    let category: Category
}
```

**何をしているか：**

観光スポット1つ分の情報をまとめるための構造体である。
名前、説明、座標、カテゴリを1つのデータとして扱えるようにしている。

**なぜこう書くのか：**

地図上に複数の観光スポットを表示するためには、それぞれの場所の名前や座標をまとめて管理する必要がある。
また、Identifiable を使うことで、ForEach でデータを繰り返し表示するときに、それぞれのデータを区別できる。

**もしこう書かなかったら：**

id や Identifiable がないと、ForEach でランドマークを表示するときにエラーが出る可能性がある。
また、名前や座標を別々に管理すると、データが分かりにくくなり、修正もしにくくなる。

---

### 地図の表示とカメラ制御

```swift
@State private var cameraPosition: MapCameraPosition = .region(
    MKCoordinateRegion(
        center: CLLocationCoordinate2D(latitude: 35.6812, longitude: 139.7671),
        span: MKCoordinateSpan(latitudeDelta: 0.08, longitudeDelta: 0.08)
    )
)
```

**何をしているか：**

地図を最初にどの位置、どの大きさで表示するかを設定している。
中心座標は東京駅付近で、span によって地図の表示範囲を決めている。

**なぜこう書くのか：**

アプリを開いたときに、どこを表示するかを決めておかないと、ユーザーが見たい場所がすぐに表示されない。
@State を使うことで、地図の表示位置が変わったときにSwiftUIが画面を更新できる。

**もしこう書かなかったら：**

地図の初期位置を自分で指定できず、意図した場所が表示されない可能性がある。
また、東京の観光スポットを表示するアプリなのに、最初から東京周辺が見えない場合がある。

---

### マーカーの表示

```swift
Map(position: $cameraPosition) {
    ForEach(filteredLandmarks) { landmark in
        Marker(
            landmark.name,
            systemImage: landmark.category.iconName,
            coordinate: landmark.coordinate
        )
        .tint(landmark.category.color)
    }
}
```

**何をしているか：**

地図を表示し、その上に観光スポットのマーカーを表示している。
ForEach を使って、複数のランドマークを1つずつ取り出し、それぞれの座標にマーカーを置いている。

**なぜこう書くのか：**

マーカーを1つずつ手で書くと、データが増えたときにコードが長くなる。
ForEach を使うことで、配列のデータから自動的に複数のマーカーを作ることができる。

**もしこう書かなかったら：**

地図だけが表示され、観光スポットの位置が分からない。
また、マーカーを手作業で追加する必要があり、管理が大変になる。

---

### フィルター機能

```swift
@State private var selectedCategories: Set<Landmark.Category> = Set(Landmark.Category.allCases)

var filteredLandmarks: [Landmark] {
    Landmark.sampleData.filter { selectedCategories.contains($0.category) }
}
```

**何をしているか：**

現在選択されているカテゴリだけを表示するために、ランドマークを絞り込んでいる。
selectedCategories に含まれているカテゴリのランドマークだけが、filteredLandmarks に入る。

**なぜこう書くのか：**

ユーザーが「寺社だけ見たい」「公園だけ非表示にしたい」などの操作をできるようにするためである。
Set を使うことで、カテゴリが選択されているかどうかを簡単に確認できる。

**もしこう書かなかったら：**

すべてのマーカーが常に表示され、カテゴリごとの切り替えができない。
表示する場所が多くなると、地図が見にくくなる。

---

## 新しく学んだSwiftの文法・API

| 項目                       | 説明                          | 使用例                                                              |
| ------------------------ | --------------------------- | ---------------------------------------------------------------- |
| `Map`                    | SwiftUIで地図を表示するビュー          | `Map(position: $cameraPosition)`                                 |
| `Marker`                 | 地図上にマーカーを表示するコンポーネント        | `Marker("東京タワー", coordinate: coordinate)`                        |
| `MapCameraPosition`      | 地図の表示位置を管理する型               | `.region(MKCoordinateRegion(...))`                               |
| `MKCoordinateRegion`     | 地図の中心座標と表示範囲を指定する型          | `MKCoordinateRegion(center: coordinate, span: span)`             |
| `CLLocationCoordinate2D` | 緯度と経度を表す型                   | `CLLocationCoordinate2D(latitude: 35.6812, longitude: 139.7671)` |
| `Identifiable`           | データを一意に識別できるようにするプロトコル      | `struct Landmark: Identifiable`                                  |
| `UUID()`                 | 一意のIDを作成する                  | `let id = UUID()`                                                |
| `enum`                   | 決まった種類の値を定義する               | `enum Category { case temple }`                                  |
| `CaseIterable`           | enumの全ケースを取得できるようにする        | `Landmark.Category.allCases`                                     |
| `@State`                 | SwiftUIで状態を管理するためのプロパティラッパー | `@State private var selectedCategories`                          |
| `@Binding`               | 親ビューの状態を子ビューから変更するための仕組み    | `@Binding var selectedCategories`                                |
| `ForEach`                | 配列の要素を繰り返し表示する              | `ForEach(filteredLandmarks) { landmark in }`                     |
| `filter`                 | 条件に合うデータだけを取り出す             | `sampleData.filter { ... }`                                      |
| `Set`                    | 重複しない値の集合を扱う型               | `Set(Landmark.Category.allCases)`                                |


## 自分の実験メモ

**実験1：**
- やったこと：selectedCategories から .park を外して、最初に公園を表示しないようにした。
- 結果：上野恩賜公園と新宿御苑のマーカーが最初から表示されなくなった。
- わかったこと：selectedCategories に入っているカテゴリだけが地図に表示されることが分かった。

**実験2：**
- やったこと：MKCoordinateSpan の latitudeDelta と longitudeDelta を 0.08 から 0.03 に変更した。
- 結果：地図がより拡大されて表示された。
- わかったこと：span の数値を小さくすると地図が拡大され、大きくすると広い範囲が表示されることが分かった。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
   
   Landmark に Identifiable を付ける理由は何ですか？

   **得られた理解：**
   
   ForEach で複数のランドマークを表示するとき、それぞれのデータを区別するために必要である。id があることで、SwiftUIがどのデータを表示・更新すればよいか判断できる。

2. **質問：**

   @State と @Binding の違いは何ですか？
   
   **得られた理解：**

   @State はそのビュー自身が持つ状態を管理するために使う。
@Binding は親ビューの状態を子ビューから変更したいときに使う。今回のコードでは、カテゴリ選択の状態を ContentView で持ち、CategoryFilter から変更している。

3. **質問：**

   filter は何をしているのですか？
    
   **得られた理解：**

   配列の中から、条件に合うデータだけを取り出すために使う。今回のコードでは、選択されているカテゴリに含まれるランドマークだけを取り出して、地図に表示している。

## この章のまとめ

この章では、SwiftUIとMapKitを使って地図を表示し、特定の座標にマーカーを置く方法を学んだ。
また、ランドマークの情報を構造体でまとめることで、複数の観光スポットを分かりやすく管理できることが分かった。
さらに、enum、Set、filter、@State、@Binding を使うことで、カテゴリごとにマーカーの表示を切り替える機能を実装できた。
今回のコードで特に重要だと思った点は、データと画面表示を分けて考えることである。
Landmark にデータをまとめ、ContentView で地図を表示し、CategoryFilter で操作部分を作ることで、コード全体の役割が分かりやすくなっている。
