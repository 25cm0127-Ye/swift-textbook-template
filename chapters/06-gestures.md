# 第6章：ジェスチャー操作

> 執筆者：葉林野
> 最終更新：2026-04-27

## この章で学ぶこと

この章では、SwiftUIでユーザーの指の動きを検出するジェスチャー操作について学んだ。
タップ、ロングプレス、ドラッグ、ピンチによる拡大縮小、回転などの基本操作を確認し、最後にTinder風のスワイプカードUIを作成した。

## 模範コードの全体像

```swift
// ============================================
// 第6章（応用）：Tinder風スワイプカードUI
// ============================================
// ドラッグジェスチャーとアニメーションを組み合わせて、
// カードを左右にスワイプして仕分けるUIを作ります。
// ============================================

import SwiftUI

// MARK: - データモデル

struct Animal: Identifiable {
    let id = UUID()
    let name: String
    let emoji: String
    let description: String
    let color: Color
}

extension Animal {
    static let sampleData: [Animal] = [
        Animal(name: "ネコ", emoji: "🐱", description: "自由気ままなマイペース派", color: .orange),
        Animal(name: "イヌ", emoji: "🐶", description: "忠実で人懐っこい", color: .brown),
        Animal(name: "ウサギ", emoji: "🐰", description: "おとなしくてかわいい", color: .pink),
        Animal(name: "ペンギン", emoji: "🐧", description: "南極のタキシード紳士", color: .cyan),
        Animal(name: "パンダ", emoji: "🐼", description: "笹が大好きなのんびり屋", color: .green),
        Animal(name: "フクロウ", emoji: "🦉", description: "夜型の知恵者", color: .purple),
    ]
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var animals: [Animal] = Animal.sampleData
    @State private var likedAnimals: [Animal] = []
    @State private var dislikedAnimals: [Animal] = []

    var body: some View {
        VStack(spacing: 20) {
            Text("好きな動物は？")
                .font(.title2)
                .bold()

            // スコア表示
            HStack(spacing: 40) {
                Label("\(dislikedAnimals.count)", systemImage: "hand.thumbsdown")
                    .foregroundStyle(.red)
                Label("\(likedAnimals.count)", systemImage: "hand.thumbsup")
                    .foregroundStyle(.green)
            }
            .font(.headline)

            // カードスタック
            ZStack {
                if animals.isEmpty {
                    VStack(spacing: 12) {
                        Text("完了！")
                            .font(.largeTitle)

                        Button("もう一度") {
                            animals = Animal.sampleData.shuffled()
                            likedAnimals = []
                            dislikedAnimals = []
                        }
                        .buttonStyle(.borderedProminent)
                    }
                } else {
                    ForEach(animals.reversed()) { animal in
                        SwipeCardView(animal: animal) { direction in
                            removeCard(animal: animal, direction: direction)
                        }
                    }
                }
            }
            .frame(height: 400)

            // 手動ボタン
            if !animals.isEmpty {
                HStack(spacing: 40) {
                    Button {
                        if let top = animals.last {
                            removeCard(animal: top, direction: .left)
                        }
                    } label: {
                        Image(systemName: "xmark.circle.fill")
                            .font(.system(size: 50))
                            .foregroundStyle(.red)
                    }

                    Button {
                        if let top = animals.last {
                            removeCard(animal: top, direction: .right)
                        }
                    } label: {
                        Image(systemName: "heart.circle.fill")
                            .font(.system(size: 50))
                            .foregroundStyle(.green)
                    }
                }
            }

            Spacer()
        }
        .padding()
    }

    func removeCard(animal: Animal, direction: SwipeDirection) {
        withAnimation(.spring(duration: 0.3)) {
            animals.removeAll { $0.id == animal.id }
        }

        switch direction {
        case .left:
            dislikedAnimals.append(animal)
        case .right:
            likedAnimals.append(animal)
        }
    }
}

// MARK: - スワイプ方向

enum SwipeDirection {
    case left, right
}

// MARK: - スワイプカードビュー

struct SwipeCardView: View {
    let animal: Animal
    let onSwipe: (SwipeDirection) -> Void

    @State private var offset: CGSize = .zero
    @State private var rotation: Double = 0

    private let swipeThreshold: CGFloat = 100

    private var swipeProgress: CGFloat {
        min(abs(offset.width) / swipeThreshold, 1.0)
    }

    var body: some View {
        ZStack {
            // カード背景
            RoundedRectangle(cornerRadius: 20)
                .fill(animal.color.opacity(0.15))
                .overlay(
                    RoundedRectangle(cornerRadius: 20)
                        .stroke(animal.color.opacity(0.3), lineWidth: 2)
                )

            // カード内容
            VStack(spacing: 16) {
                Text(animal.emoji)
                    .font(.system(size: 80))

                Text(animal.name)
                    .font(.title)
                    .bold()

                Text(animal.description)
                    .font(.body)
                    .foregroundStyle(.secondary)
            }

            // いいね / NG オーバーレイ
            if offset.width > 0 {
                Text("LIKE")
                    .font(.system(size: 40, weight: .bold))
                    .foregroundStyle(.green)
                    .opacity(swipeProgress)
                    .rotationEffect(.degrees(-20))
                    .position(x: 80, y: 60)
            } else if offset.width < 0 {
                Text("NOPE")
                    .font(.system(size: 40, weight: .bold))
                    .foregroundStyle(.red)
                    .opacity(swipeProgress)
                    .rotationEffect(.degrees(20))
                    .position(x: 240, y: 60)
            }
        }
        .frame(width: 300, height: 380)
        .shadow(color: .black.opacity(0.1), radius: 8)
        .offset(offset)
        .rotationEffect(.degrees(rotation))
        .gesture(
            DragGesture()
                .onChanged { value in
                    offset = value.translation
                    rotation = Double(value.translation.width / 20)
                }
                .onEnded { value in
                    if value.translation.width > swipeThreshold {
                        // 右スワイプ → LIKE
                        withAnimation(.easeOut(duration: 0.3)) {
                            offset = CGSize(width: 500, height: 0)
                        }
                        DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
                            onSwipe(.right)
                        }
                    } else if value.translation.width < -swipeThreshold {
                        // 左スワイプ → NOPE
                        withAnimation(.easeOut(duration: 0.3)) {
                            offset = CGSize(width: -500, height: 0)
                        }
                        DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
                            onSwipe(.left)
                        }
                    } else {
                        // 元に戻す
                        withAnimation(.spring) {
                            offset = .zero
                            rotation = 0
                        }
                    }
                }
        )
    }
}

#Preview {
    ContentView()
}
```

**このアプリは何をするものか：**

このアプリは、SwiftUIのジェスチャー操作を実際に試しながら学ぶためのアプリである。
基本編では、画面上の図形やアイコンをタップしたり、長押ししたり、ドラッグ、拡大縮小、回転させたりできる。
応用編では、動物のカードを右にスワイプすると「LIKE」、左にスワイプすると「NOPE」として分類され、カードが画面外へ移動する。

## コードの詳細解説

### 基本ジェスチャー（タップ、ロングプレス）

```swift
RoundedRectangle(cornerRadius: 16)
    .fill(backgroundColor)
    .frame(width: 200, height: 200)
    .overlay {
        Text("タップしてね")
            .foregroundStyle(.white)
            .font(.headline)
    }
    .onTapGesture {
        tapCount += 1
        backgroundColor = Color(
            hue: Double.random(in: 0...1),
            saturation: 0.7,
            brightness: 0.9
        )
    }

Circle()
    .fill(isPressed ? .green : .orange)
    .frame(width: 120, height: 120)
    .scaleEffect(isPressed ? 1.3 : 1.0)
    .overlay {
        Text(isPressed ? "成功!" : "長押し")
            .foregroundStyle(.white)
            .font(.headline)
    }
    .animation(.spring(duration: 0.3), value: isPressed)
    .onLongPressGesture(minimumDuration: 1.0) {
        isPressed = true
        DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
            isPressed = false
        }
    }
```

**何をしているか：**

この部分では、タップとロングプレスのジェスチャーを実装している。
四角形をタップすると、タップ回数が増え、背景色がランダムに変わる。
円を1秒間長押しすると、色がオレンジから緑に変わり、少し大きくなって「成功!」と表示される。

**なぜこう書くのか：**

onTapGesture を使うことで、Viewがタップされたときの処理を簡単に書くことができる。
また、onLongPressGesture(minimumDuration: 1.0) を使うことで、1秒以上押し続けた場合だけ処理を実行できる。
@State の値を変えることで、画面の表示も自動的に更新される。

**もしこう書かなかったら：**

onTapGesture がなければ、タップしても回数や色は変わらない。
onLongPressGesture がなければ、長押ししても反応しない。
また、@State を使わないと、値が変わってもSwiftUIの画面が正しく更新されない。

---

### ドラッグジェスチャーとオフセット管理

```swift
@State private var offset: CGSize = .zero
@State private var lastOffset: CGSize = .zero

RoundedRectangle(cornerRadius: 20)
    .frame(width: 200, height: 280)
    .offset(offset)
    .gesture(
        DragGesture()
            .onChanged { value in
                offset = CGSize(
                    width: lastOffset.width + value.translation.width,
                    height: lastOffset.height + value.translation.height
                )
            }
            .onEnded { _ in
                lastOffset = offset
            }
    )
```

**何をしているか：**

この部分では、カードを指でドラッグして動かす処理を実装している。
offset は現在の位置のずれを表し、lastOffset は前回ドラッグが終わった時点の位置を保存している。
ドラッグ中は value.translation を使って、指の移動距離に合わせてカードを動かしている。

**なぜこう書くのか：**

DragGesture を使うことで、ユーザーがViewをドラッグしたときの移動量を取得できる。
lastOffset を用意することで、カードを一度動かしたあとも、その位置から続けてドラッグできる。
もし lastOffset を使わないと、毎回ドラッグ開始位置を基準にしてしまい、自然な動きにならない。

**もしこう書かなかったら：**

offset を使わなければ、ドラッグしても画面上のカードは移動しない。
lastOffset を保存しなければ、ドラッグを終えた後の位置が正しく残らず、次にドラッグすると動きが不自然になる。

---

### 拡大縮小と回転

```swift
@State private var scale: CGFloat = 1.0
@State private var lastScale: CGFloat = 1.0

Image(systemName: "star.fill")
    .font(.system(size: 100))
    .scaleEffect(scale)
    .gesture(
        MagnifyGesture()
            .onChanged { value in
                scale = lastScale * value.magnification
            }
            .onEnded { _ in
                lastScale = scale
            }
    )

@State private var angle: Angle = .zero
@State private var lastAngle: Angle = .zero

Image(systemName: "arrow.up")
    .font(.system(size: 80))
    .rotationEffect(angle)
    .gesture(
        RotateGesture()
            .onChanged { value in
                angle = lastAngle + value.rotation
            }
            .onEnded { _ in
                lastAngle = angle
            }
    )
```

**何をしているか：**

この部分では、ピンチによる拡大縮小と、2本指による回転を実装している。
MagnifyGesture では、指を広げると画像が大きくなり、指を近づけると小さくなる。
RotateGesture では、2本指で回転させると、画像も同じように回転する。

**なぜこう書くのか：**

scaleEffect(scale) はViewの大きさを変えるために使う。
rotationEffect(angle) はViewの角度を変えるために使う。
lastScale や lastAngle を使うことで、前回の操作結果を残したまま、次の操作を続けられる。

**もしこう書かなかったら：**

scaleEffect がなければ、ピンチしても画像の大きさは変わらない。
rotationEffect がなければ、回転操作をしても画像は回転しない。
また、lastScale や lastAngle を使わないと、前回の操作結果がうまく引き継がれず、操作感が悪くなる。

---

### ジェスチャーの組み合わせとアニメーション

```swift
Image(systemName: "photo.artframe")
    .font(.system(size: 120))
    .scaleEffect(scale)
    .rotationEffect(angle)
    .offset(offset)
    .gesture(
        DragGesture()
            .onChanged { value in
                offset = CGSize(
                    width: lastOffset.width + value.translation.width,
                    height: lastOffset.height + value.translation.height
                )
            }
            .onEnded { _ in
                lastOffset = offset
            }
    )
    .simultaneousGesture(
        MagnifyGesture()
            .onChanged { value in
                scale = lastScale * value.magnification
            }
            .onEnded { _ in
                lastScale = scale
            }
    )
    .simultaneousGesture(
        RotateGesture()
            .onChanged { value in
                angle = lastAngle + value.rotation
            }
            .onEnded { _ in
                lastAngle = angle
            }
    )
```

**何をしているか：**

この部分では、ドラッグ、拡大縮小、回転の3つのジェスチャーを同時に使えるようにしている。
画像を移動させながら、同時に拡大縮小したり回転したりできる。

**なぜこう書くのか：**

複数のジェスチャーを同時に有効にするために、.simultaneousGesture を使っている。
.gesture を何回も重ねるだけだと、後から書いたジェスチャーが優先され、他のジェスチャーがうまく動かない場合がある。
そのため、同時に使いたいジェスチャーには .simultaneousGesture を使う。

**もしこう書かなかったら：**

.simultaneousGesture を使わないと、ドラッグ、ピンチ、回転を同時に操作できない可能性がある。
また、offset、scale、angle の状態を分けて管理しないと、それぞれの操作結果を正しく画面に反映できない。

---

## 新しく学んだSwiftの文法・API

| 項目                    | 説明                                       | 使用例                                                     |
| --------------------- | ---------------------------------------- | ------------------------------------------------------- |
| `@State`              | Viewの状態を保存するためのプロパティラッパー。値が変わると画面も更新される。 | `@State private var offset: CGSize = .zero`             |
| `onTapGesture`        | Viewがタップされたときの処理を書く。                     | `.onTapGesture { tapCount += 1 }`                       |
| `onLongPressGesture`  | Viewが長押しされたときの処理を書く。                     | `.onLongPressGesture(minimumDuration: 1.0) { ... }`     |
| `DragGesture`         | ドラッグ操作を認識するジェスチャー。                       | `.gesture(DragGesture().onChanged { value in ... })`    |
| `MagnifyGesture`      | ピンチ操作による拡大・縮小を認識するジェスチャー。                | `.gesture(MagnifyGesture().onChanged { value in ... })` |
| `RotateGesture`       | 2本指による回転操作を認識するジェスチャー。                   | `.gesture(RotateGesture().onChanged { value in ... })`  |
| `offset`              | Viewの表示位置をずらす。                           | `.offset(offset)`                                       |
| `scaleEffect`         | Viewの大きさを変更する。                           | `.scaleEffect(scale)`                                   |
| `rotationEffect`      | Viewを指定した角度だけ回転させる。                      | `.rotationEffect(angle)`                                |
| `withAnimation`       | 状態変化にアニメーションを付ける。                        | `withAnimation(.spring) { offset = .zero }`             |
| `simultaneousGesture` | 複数のジェスチャーを同時に有効にする。                      | `.simultaneousGesture(MagnifyGesture())`                |
| `Identifiable`        | データを一意に識別できるようにするプロトコル。                  | `struct Animal: Identifiable`                           |


## 自分の実験メモ

**実験1：**
- やったこと：ドラッグジェスチャーの swipeThreshold を100から50に変更した。
- 結果：少し横に動かしただけで、カードがLIKEまたはNOPEとして分類されるようになった。
- わかったこと：swipeThreshold の値が小さいほど反応しやすくなり、大きいほどしっかりスワイプしないと反応しないことがわかった。

**実験2：**
- やったこと：カードの回転角度を決める rotation = Double(value.translation.width / 20) の20を10に変更した。
- 結果：カードがドラッグしたときに、より大きく傾くようになった。
- わかったこと：割る数を小さくすると回転が大きくなり、割る数を大きくすると回転が小さくなることがわかった。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
   
   offset と lastOffset はなぜ両方必要なのか。
   
   **得られた理解：**
   
   offset は現在の位置を表し、lastOffset は前回ドラッグが終わった位置を保存するために必要である。これにより、カードを何度も自然に動かすことができる。

2. **質問：**

   MagnifyGesture と RotateGesture では、なぜ lastScale や lastAngle を使うのか。
   
   **得られた理解：**

   前回の拡大率や回転角度を保存しておくことで、次の操作を前回の状態から続けられる。これがないと、操作のたびに値がリセットされたように見えてしまう。

3. **質問：**

   Tinder風カードUIで、なぜ DispatchQueue.main.asyncAfter を使うのか。
 
   **得られた理解：**

    カードが画面外へ移動するアニメーションが終わってから、配列からカードを削除するためである。すぐに削除すると、カードが飛んでいく動きが見えず、不自然なUIになる。

## この章のまとめ

この章では、SwiftUIでジェスチャー操作を実装する方法を学んだ。
タップや長押しのような単純な操作だけでなく、ドラッグ、ピンチ、回転など、ユーザーの指の動きに合わせてViewを変化させる方法を理解できた。
特に重要だと感じたのは、ジェスチャー操作では @State を使って現在の状態を保存し、その値を offset、scaleEffect、rotationEffect などに反映させることで、画面が動いているように見えるという点である。
また、Tinder風スワイプカードUIでは、ドラッグ距離によってLIKE・NOPEを判定し、アニメーションを付けてカードを画面外へ移動させることで、実際のアプリに近い自然な操作感を作れることがわかった。
