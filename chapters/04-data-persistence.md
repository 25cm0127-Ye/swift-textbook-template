# 第4章：データの永続化

> 執筆者：葉林野
> 最終更新：2026-04-24

## この章で学ぶこと

この章では、アプリのデータを端末に保存する方法を学ぶ。
具体的には、AppStorage を使ってユーザー設定を保存し、SwiftData を使ってメモのタイトル・内容・作成日時・お気に入り状態を保存するメモアプリを作成する。

## 模範コードの全体像

```swift
// ============================================
// 第4章：データの永続化（AppStorage + SwiftData）
// ============================================
// シンプルなメモアプリで、2つの永続化方法を学びます。
// - AppStorage：アプリ設定の保存
// - SwiftData：構造化データの保存
// ============================================

import SwiftUI
import SwiftData

// MARK: - SwiftDataモデル

@Model
class Memo {
    var title: String
    var content: String
    var createdAt: Date
    var isFavorite: Bool

    init(title: String, content: String, createdAt: Date = .now, isFavorite: Bool = false) {
        self.title = title
        self.content = content
        self.createdAt = createdAt
        self.isFavorite = isFavorite
    }
}

// MARK: - メインビュー

struct ContentView: View {
    @Environment(\.modelContext) private var modelContext
    @Query(sort: \Memo.createdAt, order: .reverse) private var memos: [Memo]
    @AppStorage("sortByFavorite") private var sortByFavorite: Bool = false
    @AppStorage("userName") private var userName: String = ""
    @State private var isShowingAddSheet = false
    @State private var isShowingSettings = false

    var displayedMemos: [Memo] {
        if sortByFavorite {
            return memos.sorted { $0.isFavorite && !$1.isFavorite }
        }
        return memos
    }

    var body: some View {
        NavigationStack {
            Group {
                if memos.isEmpty {
                    ContentUnavailableView(
                        "メモがありません",
                        systemImage: "note.text",
                        description: Text("右上の＋ボタンからメモを追加してください")
                    )
                } else {
                    List {
                        ForEach(displayedMemos) { memo in
                            NavigationLink(destination: MemoEditView(memo: memo)) {
                                MemoRow(memo: memo)
                            }
                        }
                        .onDelete(perform: deleteMemos)
                    }
                }
            }
            .navigationTitle(userName.isEmpty ? "メモ帳" : "\(userName)のメモ帳")
            .toolbar {
                ToolbarItem(placement: .topBarLeading) {
                    Button {
                        isShowingSettings = true
                    } label: {
                        Image(systemName: "gear")
                    }
                }
                ToolbarItem(placement: .topBarTrailing) {
                    Button {
                        isShowingAddSheet = true
                    } label: {
                        Image(systemName: "plus")
                    }
                }
            }
            .sheet(isPresented: $isShowingAddSheet) {
                MemoAddView()
            }
            .sheet(isPresented: $isShowingSettings) {
                SettingsView(userName: $userName, sortByFavorite: $sortByFavorite)
            }
        }
    }

    func deleteMemos(at offsets: IndexSet) {
        for index in offsets {
            let memo = displayedMemos[index]
            modelContext.delete(memo)
        }
    }
}

// MARK: - メモの行

struct MemoRow: View {
    let memo: Memo

    var body: some View {
        HStack {
            VStack(alignment: .leading, spacing: 4) {
                Text(memo.title)
                    .font(.headline)

                Text(memo.content)
                    .font(.caption)
                    .foregroundStyle(.secondary)
                    .lineLimit(2)

                Text(memo.createdAt, style: .date)
                    .font(.caption2)
                    .foregroundStyle(.tertiary)
            }

            Spacer()

            if memo.isFavorite {
                Image(systemName: "star.fill")
                    .foregroundStyle(.yellow)
            }
        }
        .padding(.vertical, 2)
    }
}

// MARK: - メモ追加画面

struct MemoAddView: View {
    @Environment(\.modelContext) private var modelContext
    @Environment(\.dismiss) private var dismiss
    @State private var title = ""
    @State private var content = ""

    var body: some View {
        NavigationStack {
            Form {
                Section("タイトル") {
                    TextField("メモのタイトル", text: $title)
                }
                Section("内容") {
                    TextEditor(text: $content)
                        .frame(minHeight: 200)
                }
            }
            .navigationTitle("新しいメモ")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("キャンセル") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("保存") {
                        let memo = Memo(title: title, content: content)
                        modelContext.insert(memo)
                        dismiss()
                    }
                    .disabled(title.isEmpty)
                }
            }
        }
    }
}

// MARK: - メモ編集画面

struct MemoEditView: View {
    @Bindable var memo: Memo

    var body: some View {
        Form {
            Section("タイトル") {
                TextField("タイトル", text: $memo.title)
            }
            Section("内容") {
                TextEditor(text: $memo.content)
                    .frame(minHeight: 200)
            }
            Section {
                Toggle("お気に入り", isOn: $memo.isFavorite)
            }
        }
        .navigationTitle("メモを編集")
        .navigationBarTitleDisplayMode(.inline)
    }
}

// MARK: - 設定画面

struct SettingsView: View {
    @Binding var userName: String
    @Binding var sortByFavorite: Bool
    @Environment(\.dismiss) private var dismiss

    var body: some View {
        NavigationStack {
            Form {
                Section("ユーザー設定") {
                    TextField("あなたの名前", text: $userName)
                }
                Section("表示設定") {
                    Toggle("お気に入りを上に表示", isOn: $sortByFavorite)
                }
                Section {
                    Text("設定はアプリを閉じても保存されます")
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
            }
            .navigationTitle("設定")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .confirmationAction) {
                    Button("完了") { dismiss() }
                }
            }
        }
    }
}
```

**このアプリは何をするものか：**

このアプリは、メモを作成・表示・編集・削除できるシンプルなメモ帳アプリである。
メモのタイトル、内容、作成日時、お気に入り状態は SwiftData に保存される。また、ユーザー名や「お気に入りを上に表示する」設定は AppStorage に保存され、アプリを閉じても設定が残る。

## コードの詳細解説

### SwiftDataモデル（@Model）

```swift
@Model
class Memo {
    var title: String
    var content: String
    var createdAt: Date
    var isFavorite: Bool

    init(title: String, content: String, createdAt: Date = .now, isFavorite: Bool = false) {
        self.title = title
        self.content = content
        self.createdAt = createdAt
        self.isFavorite = isFavorite
    }
}
```

**何をしているか：**

Memo クラスは、メモ1件分のデータを表している。
タイトル、内容、作成日時、お気に入り状態を持っており、@Model を付けることで SwiftData に保存できるデータモデルになる。

**なぜこう書くのか：**

SwiftData では、保存したいデータを @Model で定義する必要がある。
このようにモデルを作ることで、modelContext.insert() で追加したり、@Query で一覧を取得したりできる。

**もしこう書かなかったら：**

@Model を付けなかった場合、SwiftData の保存対象として認識されない。
そのため、メモを追加してもデータベースに保存したり、@Query で取得したりできなくなる。

---

### データの追加・削除（modelContext）

```swift
@Environment(\.modelContext) private var modelContext

Button("保存") {
    let memo = Memo(title: title, content: content)
    modelContext.insert(memo)
    dismiss()
}
.disabled(title.isEmpty)
```

**何をしているか：**

modelContext は SwiftData のデータを操作するためのもの。
新しいメモを保存するときは modelContext.insert(memo) を使い、削除するときは modelContext.delete(memo) を使っている。

**なぜこう書くのか：**

SwiftData では、データの追加や削除を modelContext を通して行う。
画面側では直接データベースを操作するのではなく、modelContext を使うことで SwiftData が自動的に変更を管理してくれる。

**もしこう書かなかったら：**

modelContext.insert(memo) を書かなければ、新しいメモを作っても保存されない。
また、modelContext.delete(memo) を書かなければ、画面上で削除操作をしてもデータは消えない。

---

### @Queryによるデータ取得

```swift
@Query(sort: \Memo.createdAt, order: .reverse) private var memos: [Memo]

var displayedMemos: [Memo] {
    if sortByFavorite {
        return memos.sorted { $0.isFavorite && !$1.isFavorite }
    }
    return memos
}

ForEach(displayedMemos) { memo in
    NavigationLink(destination: MemoEditView(memo: memo)) {
        MemoRow(memo: memo)
    }
}
```

**何をしているか：**

@Query を使って、SwiftData に保存されている Memo の一覧を取得している。
sort: \Memo.createdAt, order: .reverse によって、新しいメモが上に表示されるようにしている。

**なぜこう書くのか：**

@Query を使うと、データベースの中身が変わったときに画面も自動で更新される。
メモを追加・削除・編集したときに、自分でリストを更新する処理を書かなくてもよい。

**もしこう書かなかったら：**

@Query がないと、保存されているメモを一覧として取得できない。
また、データを追加しても画面が自動で更新されず、リストに表示されない可能性がある。

---

### @AppStorageによる設定保存

```swift
@AppStorage("sortByFavorite") private var sortByFavorite: Bool = false
@AppStorage("userName") private var userName: String = ""

.navigationTitle(userName.isEmpty ? "メモ帳" : "\(userName)のメモ帳")

SettingsView(userName: $userName, sortByFavorite: $sortByFavorite)

TextField("あなたの名前", text: $userName)
Toggle("お気に入りを上に表示", isOn: $sortByFavorite)
```

**何をしているか：**

@AppStorage を使って、ユーザー名と表示設定を保存している。
ユーザー名を入力すると、ナビゲーションタイトルが「〇〇のメモ帳」に変わる。
また、「お気に入りを上に表示」をオンにすると、お気に入りのメモが上に表示される。

**なぜこう書くのか：**

ユーザー名や表示設定のような小さなデータは、SwiftData よりも AppStorage の方が簡単に保存できる。
@AppStorage は UserDefaults と連動しているため、アプリを閉じても設定が残る。

**もしこう書かなかったら：**

普通の @State だけで保存すると、アプリを閉じたときに設定が消えてしまう。
そのため、ユーザー名や表示設定を毎回入力し直す必要がある。

---

## 新しく学んだSwiftの文法・API

| 項目                       | 説明                               | 使用例                                                            |
| ------------------------ | -------------------------------- | -------------------------------------------------------------- |
| `@Model`                 | SwiftDataで保存するデータモデルを定義するためのマクロ  | `@Model class Memo { ... }`                                    |
| `@Query`                 | SwiftDataからデータを取得し、変更を自動で画面に反映する | `@Query var memos: [Memo]`                                     |
| `modelContext`           | SwiftDataのデータを追加・削除するために使う       | `modelContext.insert(memo)`                                    |
| `@AppStorage`            | アプリ設定などの小さなデータを保存する              | `@AppStorage("userName") var userName`                         |
| `@Bindable`              | SwiftDataのモデルを画面上で編集可能にする        | `@Bindable var memo: Memo`                                     |
| `.sheet`                 | 別画面をモーダル表示する                     | `.sheet(isPresented: $isShowingAddSheet)`                      |
| `NavigationStack`        | 画面遷移を管理する                        | `NavigationStack { ... }`                                      |
| `ContentUnavailableView` | データがないときの空画面を表示する                | `ContentUnavailableView("メモがありません", systemImage: "note.text")` |


## 自分の実験メモ

**実験1：**
- やったこと：SettingsView でユーザー名を入力して、メイン画面に戻った。
- 結果：画面タイトルが「メモ帳」から「入力した名前のメモ帳」に変わった。
- わかったこと：@AppStorage を使うと、設定画面で変更した内容がすぐに別の画面にも反映される。

**実験2：**
- やったこと：メモを追加してから、アプリを一度閉じてもう一度開いた。
- 結果：追加したメモが消えずに残っていた。
- わかったこと：SwiftData を使うと、アプリを閉じてもデータが保存される。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**

   @Model は何のために使うのか。
   
   **得られた理解：**

   @Model を付けることで、そのクラスを SwiftData に保存できるデータとして扱えることがわかった。

2. **質問：**

   メモを追加したのに一覧に表示されない原因は何か。
   
   **得られた理解：**

   Appファイルに .modelContainer(for: Memo.self) を設定しないと、SwiftData が正しく動かないことがわかった。

3. **質問：**

   @AppStorage と SwiftData の違いは何か。
   
   **得られた理解：**

   @AppStorage はユーザー名や設定のような小さいデータに向いていて、SwiftData はメモのような複数件の構造化データを保存するのに向いていることがわかった。

## この章のまとめ

この章では、SwiftUIアプリでデータを永続化する方法を学んだ。
メモのような複数のデータは SwiftData を使って保存し、ユーザー名や表示設定のような小さな設定は AppStorage を使って保存する。
また、@Query を使うことで保存されたデータを自動的に取得でき、modelContext を使って追加や削除ができることがわかった。
今後アプリを作るときは、保存したいデータの種類によって SwiftData と AppStorage を使い分けることが大切だと思った。
