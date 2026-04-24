# 第1章：WebAPIの基本

> 執筆者：葉林野
> 最終更新：2026-04-24

## この章で学ぶこと

この章では、WebAPIを使ってインターネット上のデータを取得し、アプリ内に表示する方法を学ぶ。
具体的には、iTunes Search APIを使って音楽を検索し、取得したJSONデータをSwiftの構造体に変換して、リストや詳細画面に表示するアプリを作成する。
また、API通信中のローディング表示や、通信エラー・検索結果なしの場合のエラーハンドリングについても学ぶ。

例：この章では、インターネット上のサービス（API）からデータを取得して、アプリ内に表示する方法を学ぶ。具体的にはiTunes Search APIを使って音楽を検索し、その結果をリスト表示するアプリを題材にする。

## 模範コードの全体像

```swift
// ============================================
// 第1章（応用）：エラーハンドリングとMVVM構成
// ============================================
// 基本編のコードをMVVMパターンで書き直し、
// エラーハンドリングとローディング状態を
// より適切に管理するバージョンです。
// ============================================

import SwiftUI

// MARK: - データモデル

struct SearchResponse: Codable {
    let resultCount: Int
    let results: [Song]
}

struct Song: Codable, Identifiable {
    let trackId: Int
    let trackName: String
    let artistName: String
    let collectionName: String?
    let artworkUrl100: String
    let previewUrl: String?
    let trackPrice: Double?
    let currency: String?

    var id: Int { trackId }

    var priceText: String {
        guard let price = trackPrice, let currency = currency else {
            return "価格不明"
        }
        return "\(currency) \(String(format: "%.0f", price))"
    }
}

// MARK: - ViewModel

@Observable
class MusicSearchViewModel {
    var songs: [Song] = []
    var searchText: String = ""
    var isLoading: Bool = false
    var errorMessage: String?

    enum SearchError: LocalizedError {
        case invalidURL
        case networkError(Error)
        case decodingError(Error)
        case noResults

        var errorDescription: String? {
            switch self {
            case .invalidURL:
                return "検索URLの作成に失敗しました"
            case .networkError(let error):
                return "通信エラー: \(error.localizedDescription)"
            case .decodingError:
                return "データの読み取りに失敗しました"
            case .noResults:
                return "検索結果が見つかりませんでした"
            }
        }
    }

    func searchMusic() async {
        guard !searchText.trimmingCharacters(in: .whitespaces).isEmpty else { return }

        guard let encodedText = searchText.addingPercentEncoding(
            withAllowedCharacters: .urlQueryAllowed
        ) else {
            errorMessage = SearchError.invalidURL.errorDescription
            return
        }

        let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"

        guard let url = URL(string: urlString) else {
            errorMessage = SearchError.invalidURL.errorDescription
            return
        }

        isLoading = true
        errorMessage = nil

        do {
            let (data, _) = try await URLSession.shared.data(from: url)
            let response = try JSONDecoder().decode(SearchResponse.self, from: data)

            if response.results.isEmpty {
                errorMessage = SearchError.noResults.errorDescription
                songs = []
            } else {
                songs = response.results
            }
        } catch let error as DecodingError {
            errorMessage = SearchError.decodingError(error).errorDescription
            songs = []
        } catch {
            errorMessage = SearchError.networkError(error).errorDescription
            songs = []
        }

        isLoading = false
    }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var viewModel = MusicSearchViewModel()

    var body: some View {
        NavigationStack {
            VStack(spacing: 0) {
                searchBar

                if let errorMessage = viewModel.errorMessage {
                    ErrorBanner(message: errorMessage)
                }

                contentArea
            }
            .navigationTitle("Music Search")
        }
    }

    // MARK: - 検索バー

    private var searchBar: some View {
        HStack {
            TextField("アーティスト名を入力", text: $viewModel.searchText)
                .textFieldStyle(.roundedBorder)
                .onSubmit {
                    Task { await viewModel.searchMusic() }
                }

            Button("検索") {
                Task { await viewModel.searchMusic() }
            }
            .buttonStyle(.borderedProminent)
            .disabled(viewModel.searchText.isEmpty || viewModel.isLoading)
        }
        .padding()
    }

    // MARK: - コンテンツエリア

    @ViewBuilder
    private var contentArea: some View {
        if viewModel.isLoading {
            Spacer()
            ProgressView("検索中...")
            Spacer()
        } else if viewModel.songs.isEmpty {
            ContentUnavailableView(
                "曲を検索してみよう",
                systemImage: "music.note",
                description: Text("アーティスト名を入力して検索ボタンを押してください")
            )
        } else {
            List(viewModel.songs) { song in
                NavigationLink(destination: SongDetailView(song: song)) {
                    SongRow(song: song)
                }
            }
        }
    }
}

// MARK: - 曲の行ビュー

struct SongRow: View {
    let song: Song

    var body: some View {
        HStack(spacing: 12) {
            AsyncImage(url: URL(string: song.artworkUrl100)) { image in
                image.resizable().aspectRatio(contentMode: .fill)
            } placeholder: {
                RoundedRectangle(cornerRadius: 8)
                    .fill(.gray.opacity(0.2))
            }
            .frame(width: 60, height: 60)
            .clipShape(RoundedRectangle(cornerRadius: 8))

            VStack(alignment: .leading, spacing: 4) {
                Text(song.trackName)
                    .font(.headline)
                    .lineLimit(1)
                Text(song.artistName)
                    .font(.subheadline)
                    .foregroundStyle(.secondary)
            }

            Spacer()

            Text(song.priceText)
                .font(.caption)
                .foregroundStyle(.secondary)
        }
        .padding(.vertical, 4)
    }
}

// MARK: - 詳細ビュー

struct SongDetailView: View {
    let song: Song

    var body: some View {
        ScrollView {
            VStack(spacing: 20) {
                AsyncImage(url: URL(string: song.artworkUrl100)) { image in
                    image.resizable().aspectRatio(contentMode: .fit)
                } placeholder: {
                    ProgressView()
                }
                .frame(width: 200, height: 200)
                .clipShape(RoundedRectangle(cornerRadius: 16))
                .shadow(radius: 8)

                Text(song.trackName)
                    .font(.title2)
                    .bold()
                    .multilineTextAlignment(.center)

                Text(song.artistName)
                    .font(.title3)
                    .foregroundStyle(.secondary)

                if let albumName = song.collectionName {
                    Text(albumName)
                        .font(.subheadline)
                        .foregroundStyle(.tertiary)
                }

                Text(song.priceText)
                    .font(.headline)
                    .padding(.horizontal, 16)
                    .padding(.vertical, 8)
                    .background(.blue.opacity(0.1))
                    .clipShape(Capsule())
            }
            .padding()
        }
        .navigationTitle("曲の詳細")
        .navigationBarTitleDisplayMode(.inline)
    }
}

// MARK: - エラーバナー

struct ErrorBanner: View {
    let message: String

    var body: some View {
        HStack {
            Image(systemName: "exclamationmark.triangle.fill")
                .foregroundStyle(.yellow)
            Text(message)
                .font(.caption)
        }
        .padding(10)
        .frame(maxWidth: .infinity)
        .background(.red.opacity(0.1))
    }
}

#Preview {
    ContentView()
}
```

**このアプリは何をするものか：**

このアプリは、iTunes Search APIを使って音楽を検索するアプリである。
ユーザーがアーティスト名を入力して「検索」ボタンを押すと、APIから曲の情報を取得し、曲名・アーティスト名・ジャケット画像・価格をリストで表示する。
また、リストの曲をタップすると詳細画面へ移動し、大きな画像、曲名、アーティスト名、アルバム名、価格を確認できる。通信中はローディング表示を行い、エラーや検索結果がない場合はエラーメッセージを表示する。

## コードの詳細解説

### データモデル（Codable構造体）

```swift
struct SearchResponse: Codable {
    let resultCount: Int
    let results: [Song]
}

struct Song: Codable, Identifiable {
    let trackId: Int
    let trackName: String
    let artistName: String
    let collectionName: String?
    let artworkUrl100: String
    let previewUrl: String?
    let trackPrice: Double?
    let currency: String?

    var id: Int { trackId }

    var priceText: String {
        guard let price = trackPrice, let currency = currency else {
            return "価格不明"
        }
        return "\(currency) \(String(format: "%.0f", price))"
    }
}
```

**何をしているか：**

この部分では、iTunes Search APIから返ってくるJSONデータをSwiftで扱いやすい形にするための構造体を定義している。
SearchResponseはAPI全体のレスポンスを表し、resultCountには検索結果の件数、resultsには曲の配列が入る。
Songは1曲分のデータを表しており、曲ID、曲名、アーティスト名、アルバム名、画像URL、試聴URL、価格、通貨などを持っている。

**なぜこう書くのか：**

APIから返ってくるデータはJSON形式なので、そのままではSwiftUIの画面に表示しにくい。
そのため、Codableを使ってJSONをSwiftの構造体に変換できるようにしている。
また、SongにIdentifiableを付けることで、Listで曲を表示するときに、SwiftUIがそれぞれの曲を区別できるようになる。
priceTextを作ることで、価格がある場合は「JPY 255」のように表示し、価格がない場合は「価格不明」と表示できる。

**もしこう書かなかったら：**

Codableがないと、JSONDecoderでJSONデータを簡単に構造体へ変換できない。
また、Identifiableがないと、List(viewModel.songs)のように簡単にリスト表示できなくなる。
さらに、trackPriceやcollectionNameなどをOptionalにしなかった場合、APIからその項目が返ってこないとデコードに失敗する可能性がある。

---

### API通信の処理

```swift
func searchMusic() async {
    guard !searchText.trimmingCharacters(in: .whitespaces).isEmpty else { return }

    guard let encodedText = searchText.addingPercentEncoding(
        withAllowedCharacters: .urlQueryAllowed
    ) else {
        errorMessage = SearchError.invalidURL.errorDescription
        return
    }

    let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"

    guard let url = URL(string: urlString) else {
        errorMessage = SearchError.invalidURL.errorDescription
        return
    }

    isLoading = true
    errorMessage = nil

    do {
        let (data, _) = try await URLSession.shared.data(from: url)
        let response = try JSONDecoder().decode(SearchResponse.self, from: data)

        if response.results.isEmpty {
            errorMessage = SearchError.noResults.errorDescription
            songs = []
        } else {
            songs = response.results
        }
    } catch let error as DecodingError {
        errorMessage = SearchError.decodingError(error).errorDescription
        songs = []
    } catch {
        errorMessage = SearchError.networkError(error).errorDescription
        songs = []
    }

    isLoading = false
}
```

**何をしているか：**

この部分では、ユーザーが入力した検索キーワードを使ってiTunes Search APIにアクセスしている。
まず、入力欄が空でないか確認し、検索文字列をURLで使える形に変換する。
その後、APIのURLを作成し、URLSession.shared.data(from:)でデータを取得する。
取得したJSONデータはJSONDecoderでSearchResponseに変換され、検索結果があればsongsに保存される。
検索結果がない場合や、通信エラー、JSON解析エラーが起きた場合は、errorMessageにエラー内容を保存する。

**なぜこう書くのか：**

API通信は時間がかかる処理なので、async/awaitを使って非同期で実行している。
また、ユーザーの入力には空白や日本語、スペースが含まれる可能性があるため、addingPercentEncodingでURL用に変換する必要がある。
do-catchを使うことで、通信失敗やデコード失敗などのエラーを安全に処理できる。
isLoadingを使うことで、検索中に画面へ「検索中...」を表示できる。

**もしこう書かなかったら：**

URLエンコードをしないと、空白や日本語を含む検索語で正しく検索できない可能性がある。
try awaitやdo-catchを書かないと、通信エラーが起きたときにアプリが正しく処理できない。
また、isLoadingを管理しないと、ユーザーは検索中かどうか分からず、何度も検索ボタンを押してしまう可能性がある。

---

### ビューの構成

```swift
struct ContentView: View {
    @State private var viewModel = MusicSearchViewModel()

    var body: some View {
        NavigationStack {
            VStack(spacing: 0) {
                searchBar

                if let errorMessage = viewModel.errorMessage {
                    ErrorBanner(message: errorMessage)
                }

                contentArea
            }
            .navigationTitle("Music Search")
        }
    }
}
```

**何をしているか：**

この部分では、アプリのメイン画面を構成している。
NavigationStackを使って画面遷移ができるようにし、VStackの中に検索バー、エラーバナー、コンテンツエリアを配置している。
viewModel.errorMessageに値がある場合は、ErrorBannerを表示してエラーをユーザーに知らせる。

**なぜこう書くのか：**

画面の構成をsearchBar、contentArea、ErrorBannerのように分けることで、コードが読みやすくなる。
また、MusicSearchViewModelを使うことで、画面側は主に表示を担当し、検索処理や状態管理はViewModelに任せることができる。
これはMVVMパターンに近い構成であり、処理の役割が分かりやすくなる。

**もしこう書かなかったら：**

すべての処理をContentViewの中に直接書くと、画面表示とAPI通信の処理が混ざってしまい、コードが長くなって読みにくくなる。
また、エラーメッセージの表示を分けておかないと、通信に失敗したときにユーザーが何が起きたか分かりにくくなる。

---

## 新しく学んだSwiftの文法・API

| 項目                       | 説明                                   | 使用例                                                              |
| ------------------------ | ------------------------------------ | ---------------------------------------------------------------- |
| `Codable`                | JSONデータとSwiftの構造体を相互変換するためのプロトコル     | `struct Song: Codable { ... }`                                   |
| `Identifiable`           | SwiftUIの`List`などで各データを区別するためのプロトコル   | `struct Song: Identifiable { var id: Int { trackId } }`          |
| `async/await`            | 非同期処理を分かりやすく書くための構文                  | `try await URLSession.shared.data(from: url)`                    |
| `URLSession`             | WebAPIなどにアクセスしてデータを取得するためのAPI        | `URLSession.shared.data(from: url)`                              |
| `JSONDecoder`            | JSONデータをSwiftの構造体に変換するためのクラス         | `JSONDecoder().decode(SearchResponse.self, from: data)`          |
| `guard`                  | 条件を満たさない場合に早めに処理を終了する構文              | `guard let url = URL(string: urlString) else { return }`         |
| `Optional`               | 値がある場合とない場合を表す型                      | `let collectionName: String?`                                    |
| `if let`                 | Optionalの値がある場合だけ取り出して使う構文           | `if let albumName = song.collectionName { ... }`                 |
| `do-catch`               | エラーが発生する可能性のある処理を安全に扱う構文             | `do { ... } catch { ... }`                                       |
| `@Observable`            | ViewModelの状態変化をSwiftUIの画面に反映するための仕組み | `@Observable class MusicSearchViewModel { ... }`                 |
| `@State`                 | SwiftUIのView内で状態を保持するためのプロパティラッパー    | `@State private var viewModel = MusicSearchViewModel()`          |
| `AsyncImage`             | URLから画像を非同期で読み込んで表示するView            | `AsyncImage(url: URL(string: song.artworkUrl100))`               |
| `NavigationStack`        | 画面遷移を管理するためのSwiftUIのView             | `NavigationStack { ... }`                                        |
| `NavigationLink`         | タップしたときに別の画面へ移動するためのView             | `NavigationLink(destination: SongDetailView(song: song))`        |
| `ProgressView`           | 読み込み中の状態を表示するView                    | `ProgressView("検索中...")`                                         |
| `ContentUnavailableView` | データがないときの案内画面を表示するView               | `ContentUnavailableView("曲を検索してみよう", systemImage: "music.note")` |


## 自分の実験メモ

**実験1：**
- やったこと：limit=25をlimit=10に変更して、検索結果の件数を減らしてみた。
- 結果：表示される曲の数が最大10件になった。
- わかったこと：APIのURLに付けるパラメータを変更することで、取得するデータの量を調整できる。

**実験2：**
- やったこと：検索キーワードに存在しないアーティスト名を入力して検索してみた。
- 結果：「検索結果が見つかりませんでした」というエラーメッセージが表示された。
- わかったこと：response.results.isEmptyを使うことで、検索結果がない場合を判定できる。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**

   Codableは何のために使うのか。
   
   **得られた理解：**

   APIから返ってくるJSONデータを、Swiftの構造体に簡単に変換するために使うことが分かった。JSONDecoderと組み合わせることで、手作業で一つずつ値を取り出さなくてもよくなる。

2. **質問：**

   なぜsearchMusic()にasyncを付けるのか。
   
   **得られた理解：**

   API通信はすぐに終わる処理ではなく、サーバーからの応答を待つ必要があるため、非同期処理として書く必要があると分かった。async/awaitを使うと、非同期処理でも読みやすいコードになる。

4. **質問：**

   MVVM構成にすると何が良いのか。
   
   **得られた理解：**

   画面表示を担当するViewと、データ取得や状態管理を担当するViewModelを分けることで、コードの役割が分かりやすくなる。特に、API通信やエラー処理をContentViewから分離できるので、コードが整理しやすくなる。

## この章のまとめ

この章では、iTunes Search APIを使って、WebAPIからデータを取得し、SwiftUIアプリの画面に表示する方法を学んだ。
特に重要だと感じたのは、APIから返ってくるJSONデータをCodableとJSONDecoderでSwiftの構造体に変換する流れである。
また、API通信ではasync/awaitやdo-catchを使って、通信中の状態やエラーを適切に処理する必要があると分かった。
さらに、MVVM構成にすることで、画面表示、データ管理、API通信の役割を分けられ、コードが読みやすく保守しやすくなることを学んだ。
