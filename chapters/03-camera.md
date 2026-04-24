# 第3章：カメラの利用

> 執筆者：葉林野
> 最終更新：2026-04-24

## この章で学ぶこと

この章では、SwiftUIで写真を選択したり、カメラで撮影した画像を画面に表示したりする方法を学ぶ。
PhotosPickerを使ったフォトライブラリからの画像選択、UIImagePickerControllerを使ったカメラ撮影、そしてUIViewControllerRepresentableによるSwiftUIとUIKitの連携について理解する。

## 模範コードの全体像

```swift
// ============================================
// 第3章（基本）：写真を選択・撮影して表示するアプリ
// ============================================
// PhotosPickerを使ってフォトライブラリから写真を選択し、
// 画面に表示します。シミュレータでも動作します。
// ============================================

import SwiftUI
import PhotosUI

// MARK: - メインビュー

struct ContentView: View {
    @State private var selectedItem: PhotosPickerItem?
    @State private var selectedImage: Image?
    @State private var isShowingCamera = false
    @State private var capturedUIImage: UIImage?

    var body: some View {
        NavigationStack {
            VStack(spacing: 20) {
                // 画像表示エリア
                imageDisplayArea

                // ボタンエリア
                HStack(spacing: 20) {
                    // フォトライブラリから選択
                    PhotosPicker(selection: $selectedItem, matching: .images) {
                        Label("ライブラリ", systemImage: "photo.on.rectangle")
                    }
                    .buttonStyle(.bordered)

                    // カメラで撮影
                    Button {
                        isShowingCamera = true
                    } label: {
                        Label("カメラ", systemImage: "camera")
                    }
                    .buttonStyle(.borderedProminent)
                }
                .padding()
            }
            .navigationTitle("写真アプリ")
            .onChange(of: selectedItem) { _, newItem in
                Task {
                    await loadImage(from: newItem)
                }
            }
            .fullScreenCover(isPresented: $isShowingCamera) {
                CameraView(capturedImage: $capturedUIImage)
            }
            .onChange(of: capturedUIImage) { _, newImage in
                if let uiImage = newImage {
                    selectedImage = Image(uiImage: uiImage)
                }
            }
        }
    }

    // MARK: - 画像表示エリア

    @ViewBuilder
    private var imageDisplayArea: some View {
        if let image = selectedImage {
            image
                .resizable()
                .aspectRatio(contentMode: .fit)
                .frame(maxHeight: 400)
                .clipShape(RoundedRectangle(cornerRadius: 16))
                .shadow(radius: 4)
                .padding()
        } else {
            RoundedRectangle(cornerRadius: 16)
                .fill(.gray.opacity(0.1))
                .frame(height: 300)
                .overlay {
                    VStack(spacing: 8) {
                        Image(systemName: "photo")
                            .font(.system(size: 48))
                            .foregroundStyle(.gray)
                        Text("写真を選択または撮影してください")
                            .font(.caption)
                            .foregroundStyle(.secondary)
                    }
                }
                .padding()
        }
    }

    // MARK: - 画像の読み込み

    func loadImage(from item: PhotosPickerItem?) async {
        guard let item = item else { return }

        do {
            if let data = try await item.loadTransferable(type: Data.self),
               let uiImage = UIImage(data: data) {
                selectedImage = Image(uiImage: uiImage)
            }
        } catch {
            print("画像の読み込みに失敗: \(error.localizedDescription)")
        }
    }
}

// MARK: - カメラビュー（UIKit連携）

struct CameraView: UIViewControllerRepresentable {
    @Binding var capturedImage: UIImage?
    @Environment(\.dismiss) private var dismiss

    func makeUIViewController(context: Context) -> UIImagePickerController {
        let picker = UIImagePickerController()
        picker.sourceType = .camera
        picker.delegate = context.coordinator
        return picker
    }

    func updateUIViewController(_ uiViewController: UIImagePickerController, context: Context) {}

    func makeCoordinator() -> Coordinator {
        Coordinator(self)
    }

    class Coordinator: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {
        let parent: CameraView

        init(_ parent: CameraView) {
            self.parent = parent
        }

        func imagePickerController(
            _ picker: UIImagePickerController,
            didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey: Any]
        ) {
            if let image = info[.originalImage] as? UIImage {
                parent.capturedImage = image
            }
            parent.dismiss()
        }

        func imagePickerControllerDidCancel(_ picker: UIImagePickerController) {
            parent.dismiss()
        }
    }
}

#Preview {
    ContentView()
}
```

**このアプリは何をするものか：**

このアプリは、フォトライブラリから写真を選択したり、カメラで写真を撮影したりして、その画像を画面に表示するアプリである。
最初は灰色の画像表示エリアが表示され、「写真を選択または撮影してください」という案内が出る。ユーザーが「ライブラリ」ボタンを押すと写真を選択でき、「カメラ」ボタンを押すとカメラ画面が開き、撮影した写真を表示できる。

## コードの詳細解説

### PhotosPickerによる写真選択

```swift
@State private var selectedItem: PhotosPickerItem?

PhotosPicker(selection: $selectedItem, matching: .images) {
    Label("ライブラリ", systemImage: "photo.on.rectangle")
}
.buttonStyle(.bordered)
```

**何をしているか：**

PhotosPickerを使って、フォトライブラリから画像を選択できるようにしている。
ユーザーが写真を選ぶと、その情報がselectedItemに保存される。matching: .imagesを指定しているため、動画ではなく画像だけを選択対象にしている。

**なぜこう書くのか：**

SwiftUIでは、フォトライブラリから写真を選択する場合、PhotosPickerを使うと簡単に実装できる。
また、selection: $selectedItemのように@State変数とバインディングすることで、選択された写真の情報を画面側で管理できる。

**もしこう書かなかったら：**

PhotosPickerがないと、ユーザーはフォトライブラリから写真を選択できない。
また、selectionを指定しなければ、選択した写真の情報を受け取ることができないため、画像を画面に表示する処理につなげられない。

---

### 画像の非同期読み込み

```swift
.onChange(of: selectedItem) { _, newItem in
    Task {
        await loadImage(from: newItem)
    }
}

func loadImage(from item: PhotosPickerItem?) async {
    guard let item = item else { return }

    do {
        if let data = try await item.loadTransferable(type: Data.self),
           let uiImage = UIImage(data: data) {
            selectedImage = Image(uiImage: uiImage)
        }
    } catch {
        print("画像の読み込みに失敗: \(error.localizedDescription)")
    }
}
```

**何をしているか：**

ユーザーが写真を選択すると、selectedItemの値が変わる。その変化をonChangeで検知し、loadImage(from:)を実行して画像データを読み込んでいる。
読み込んだデータは一度UIImageに変換され、その後SwiftUIのImageとしてselectedImageに保存される。

**なぜこう書くのか：**

写真データの読み込みは時間がかかる可能性があるため、asyncとawaitを使って非同期で処理している。
これにより、画像の読み込み中でも画面が固まりにくくなる。また、guard letを使うことで、写真が選択されていない場合は安全に処理を終了できる。

**もしこう書かなかったら：**

onChangeがなければ、ユーザーが写真を選んでも読み込み処理が実行されない。
また、awaitを使わずに画像を読み込もうとすると、非同期処理に対応できず、正しく画像を取得できない可能性がある。

---

### UIViewControllerRepresentableによるカメラ連携

```swift
.fullScreenCover(isPresented: $isShowingCamera) {
    CameraView(capturedImage: $capturedUIImage)
}

struct CameraView: UIViewControllerRepresentable {
    @Binding var capturedImage: UIImage?
    @Environment(\.dismiss) private var dismiss

    func makeUIViewController(context: Context) -> UIImagePickerController {
        let picker = UIImagePickerController()
        picker.sourceType = .camera
        picker.delegate = context.coordinator
        return picker
    }

    func updateUIViewController(_ uiViewController: UIImagePickerController, context: Context) {}
}
```

**何をしているか：**

fullScreenCoverを使って、カメラ画面を全画面で表示している。
CameraViewでは、UIKitのUIImagePickerControllerを利用してカメラ機能を呼び出している。SwiftUIではUIKitの画面をそのまま使えないため、UIViewControllerRepresentableを使ってSwiftUIから利用できる形に変換している。

**なぜこう書くのか：**

SwiftUIだけでは、UIImagePickerControllerを直接表示できない。
そのため、UIViewControllerRepresentableを使ってUIKitのViewControllerをSwiftUIのViewとして扱えるようにしている。これにより、SwiftUIアプリの中でもカメラ撮影機能を利用できる。

**もしこう書かなかったら：**

UIImagePickerControllerをSwiftUIの画面に直接表示することはできない。
また、picker.sourceType = .cameraを指定しなければ、カメラを起動する設定にならないため、撮影機能として動作しない。

---

### Coordinatorパターン

```swift
func makeCoordinator() -> Coordinator {
    Coordinator(self)
}

class Coordinator: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {
    let parent: CameraView

    init(_ parent: CameraView) {
        self.parent = parent
    }

    func imagePickerController(
        _ picker: UIImagePickerController,
        didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey: Any]
    ) {
        if let image = info[.originalImage] as? UIImage {
            parent.capturedImage = image
        }
        parent.dismiss()
    }

    func imagePickerControllerDidCancel(_ picker: UIImagePickerController) {
        parent.dismiss()
    }
}
```

**何をしているか：**

Coordinatorは、UIKitのカメラ画面で起きたイベントをSwiftUI側に伝える役割をしている。
写真撮影が完了したときはdidFinishPickingMediaWithInfoが呼ばれ、撮影した画像をcapturedImageに保存する。キャンセルされた場合はimagePickerControllerDidCancelが呼ばれ、カメラ画面を閉じる。

**なぜこう書くのか：**

UIImagePickerControllerはUIKitの機能なので、撮影完了やキャンセルなどのイベントをdelegateで受け取る必要がある。
SwiftUIではdelegateを直接扱いにくいため、Coordinatorを作ってUIKitとSwiftUIの橋渡しをしている。

**もしこう書かなかったら：**

撮影した画像をSwiftUI側に渡すことができない。
また、キャンセルボタンを押したときや撮影後にカメラ画面を閉じる処理もできなくなる可能性がある。

---

## 新しく学んだSwiftの文法・API

| 項目                              | 説明                                       | 使用例                                                         |
| ------------------------------- | ---------------------------------------- | ----------------------------------------------------------- |
| `PhotosPicker`                  | フォトライブラリから画像を選択するコンポーネント                 | `PhotosPicker(selection: $selectedItem, matching: .images)` |
| `PhotosPickerItem`              | 選択された写真の情報を保持する型                         | `@State private var selectedItem: PhotosPickerItem?`        |
| `loadTransferable`              | 選択した写真のデータを非同期で読み込むメソッド                  | `try await item.loadTransferable(type: Data.self)`          |
| `UIImage`                       | UIKitで使われる画像型                            | `UIImage(data: data)`                                       |
| `Image`                         | SwiftUIで画面に表示する画像型                       | `Image(uiImage: uiImage)`                                   |
| `Task`                          | 非同期処理を開始するために使う                          | `Task { await loadImage(from: newItem) }`                   |
| `onChange`                      | 値が変化したときに処理を実行する                         | `.onChange(of: selectedItem) { _, newItem in ... }`         |
| `fullScreenCover`               | 画面を全画面で表示する                              | `.fullScreenCover(isPresented: $isShowingCamera)`           |
| `UIViewControllerRepresentable` | UIKitのViewControllerをSwiftUIで使えるようにする仕組み | `struct CameraView: UIViewControllerRepresentable`          |
| `UIImagePickerController`       | カメラやフォトライブラリを扱うUIKitのViewController      | `picker.sourceType = .camera`                               |
| `@Binding`                      | 親Viewと子Viewの状態を共有する                      | `@Binding var capturedImage: UIImage?`                      |
| `@Environment(\.dismiss)`       | 現在の画面を閉じるために使う                           | `parent.dismiss()`                                          |
| `Coordinator`                   | UIKitのdelegate処理をSwiftUIに橋渡しする           | `func makeCoordinator() -> Coordinator`                     |


## 自分の実験メモ

**実験1：**
- やったこと：フォトライブラリから写真を選択して、画面に表示されるか確認した。
- 結果：「ライブラリ」ボタンを押すと写真を選択でき、選択した写真が画面中央に表示された。
- わかったこと：PhotosPickerで選択した写真は、そのまま表示されるのではなく、loadTransferableでデータを読み込み、UIImageからImageに変換する必要がある。

**実験2：**
- やったこと：「カメラ」ボタンを押して、カメラ画面が開くか確認した。
- 結果：実機ではカメラ画面が開き、撮影した写真を画面に表示できた。シミュレータではカメラが使えない場合がある。
- わかったこと：カメラ機能はUIKitのUIImagePickerControllerを使う必要があり、SwiftUIで使うためにはUIViewControllerRepresentableが必要である。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
   
   PhotosPickerItemとImageは何が違うのか。
   
   **得られた理解：**

   PhotosPickerItemは選択された写真の情報を持つだけで、画面に直接表示できる画像ではない。実際に表示するには、データを読み込んでUIImageに変換し、さらにSwiftUIのImageに変換する必要がある。

2. **質問：**

   なぜTaskとawaitを使う必要があるのか。
   
   **得られた理解：**

   写真データの読み込みは時間がかかる可能性があるため、非同期で処理する必要がある。Taskとawaitを使うことで、画面の動作を止めずに画像を読み込むことができる。

3. **質問：**

   なぜカメラ機能でUIViewControllerRepresentableとCoordinatorが必要なのか。
   
   **得られた理解：**

   カメラ機能はUIKitのUIImagePickerControllerを使うため、そのままではSwiftUIで扱えない。UIViewControllerRepresentableでUIKitの画面をSwiftUIに組み込み、Coordinatorで撮影完了やキャンセルなどのイベントを受け取る必要がある。

## この章のまとめ

この章では、SwiftUIで写真を選択したり、カメラで撮影した画像を表示したりする方法を学んだ。
フォトライブラリから画像を選ぶ場合はPhotosPickerを使い、選択されたPhotosPickerItemから非同期でデータを読み込む必要がある。読み込んだ画像はUIImageからSwiftUIのImageに変換して表示する。
また、カメラ機能はSwiftUIだけでは直接扱えないため、UIKitのUIImagePickerControllerをUIViewControllerRepresentableでSwiftUIに組み込む必要がある。さらに、撮影完了やキャンセルなどのイベントを受け取るためにCoordinatorパターンを使うことも理解できた。
この章で最も重要だと感じたのは、SwiftUIとUIKitを連携させる考え方である。SwiftUIだけでできない機能でも、UIKitを組み合わせることで実現できることがわかった。
