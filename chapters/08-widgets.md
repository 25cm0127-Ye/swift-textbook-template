# 第8章：ウィジェット

> 執筆者：葉林野
> 最終更新：2026-04-28

## この章で学ぶこと

この章では、WidgetKitを使って、iPhoneのホーム画面に表示できるウィジェットを作る方法を学んだ。
今回は「今日の名言」を表示するアプリを題材にして、メインアプリとウィジェットで同じ名言データを使う方法、TimelineProviderによる更新の仕組み、ウィジェットサイズごとのレイアウトの作り方を学んだ。

## 模範コードの全体像

```swift
// ============================================
// 第8章：ウィジェットを作る
// ============================================
// 今日の名言をホーム画面に表示するウィジェットです。
// メインアプリとウィジェットの両方のコードを含みます。
//
// 【セットアップ手順】
// 1. Xcodeで File → New → Target → Widget Extension を選択
// 2. 「Include Configuration App Intent」のチェックを外す
// 3. Widget Extensionの名前を「QuoteWidget」にする
// 4. QuoteStore（名言データ）を別ファイル QuoteStore.swift に切り出し、
//    そのファイルを「メインアプリ」と「QuoteWidget Extension」の
//    両方の Target Membership にチェックを入れる
//    （ファイル右側のインスペクタ → Target Membership）
//
// ※ App Group の設定は不要です（QuoteStore は静的データのため、
//   UserDefaults や共有ファイルでのデータ受け渡しを行いません）
// ============================================

// ============================================
// ■ メインアプリ側のコード（ContentView.swift）
// ============================================

import SwiftUI

// MARK: - 名言データ（アプリとウィジェットで共有）

struct Quote: Identifiable, Codable {
    let id: Int
    let text: String
    let author: String
}

struct QuoteStore {
    static let quotes: [Quote] = [
        Quote(id: 1, text: "為せば成る、為さねば成らぬ何事も", author: "上杉鷹山"),
        Quote(id: 2, text: "千里の道も一歩から", author: "老子"),
        Quote(id: 3, text: "継続は力なり", author: "ことわざ"),
        Quote(id: 4, text: "失敗は成功のもと", author: "ことわざ"),
        Quote(id: 5, text: "知ることは愛することの始まりである", author: "ことわざ"),
        Quote(id: 6, text: "学びて思わざれば則ち罔し", author: "孔子"),
        Quote(id: 7, text: "過ちて改めざる、是を過ちと謂う", author: "孔子"),
    ]

    static func todaysQuote() -> Quote {
        let dayOfYear = Calendar.current.ordinality(of: .day, in: .year, for: Date()) ?? 0
        let index = dayOfYear % quotes.count
        return quotes[index]
    }
}

// MARK: - メインアプリのContentView

struct ContentView: View {
    let todaysQuote = QuoteStore.todaysQuote()
    @State private var allQuotes = QuoteStore.quotes

    var body: some View {
        NavigationStack {
            VStack(spacing: 24) {
                // 今日の名言（ハイライト）
                VStack(spacing: 16) {
                    Text("今日の名言")
                        .font(.caption)
                        .foregroundStyle(.secondary)

                    Text("「\(todaysQuote.text)」")
                        .font(.title2)
                        .bold()
                        .multilineTextAlignment(.center)

                    Text("— \(todaysQuote.author)")
                        .font(.subheadline)
                        .foregroundStyle(.secondary)
                }
                .padding(24)
                .frame(maxWidth: .infinity)
                .background(
                    RoundedRectangle(cornerRadius: 16)
                        .fill(.blue.opacity(0.08))
                )
                .padding(.horizontal)

                // 全名言リスト
                List(allQuotes) { quote in
                    VStack(alignment: .leading, spacing: 4) {
                        Text(quote.text)
                            .font(.body)
                        Text("— \(quote.author)")
                            .font(.caption)
                            .foregroundStyle(.secondary)
                    }
                    .padding(.vertical, 4)
                }
            }
            .navigationTitle("名言集")
        }
    }
}

#Preview {
    ContentView()
}


// ============================================
// ■ ウィジェット側のコード（QuoteWidget.swift）
// ============================================
// ※ Widget Extension ターゲット内のファイルに記述します。
// ※ QuoteStore は共有ファイルとして両ターゲットに追加するか、
//    同じコードをウィジェット側にもコピーしてください。
// ============================================

/*
import WidgetKit
import SwiftUI

// MARK: - タイムラインエントリ

struct QuoteEntry: TimelineEntry {
    let date: Date
    let quote: Quote
}

// MARK: - タイムラインプロバイダ

struct QuoteProvider: TimelineProvider {
    // プレースホルダー（読み込み中の仮表示）
    func placeholder(in context: Context) -> QuoteEntry {
        QuoteEntry(
            date: Date(),
            quote: Quote(id: 0, text: "読み込み中...", author: "")
        )
    }

    // スナップショット（ウィジェットギャラリーでのプレビュー）
    func getSnapshot(in context: Context, completion: @escaping (QuoteEntry) -> Void) {
        let entry = QuoteEntry(
            date: Date(),
            quote: QuoteStore.todaysQuote()
        )
        completion(entry)
    }

    // タイムライン（実際のウィジェット更新スケジュール）
    func getTimeline(in context: Context, completion: @escaping (Timeline<QuoteEntry>) -> Void) {
        let currentDate = Date()
        let quote = QuoteStore.todaysQuote()
        let entry = QuoteEntry(date: currentDate, quote: quote)

        // 次の日の0時にウィジェットを更新
        let tomorrow = Calendar.current.startOfDay(
            for: Calendar.current.date(byAdding: .day, value: 1, to: currentDate)!
        )

        let timeline = Timeline(entries: [entry], policy: .after(tomorrow))
        completion(timeline)
    }
}

// MARK: - ウィジェットのビュー

struct QuoteWidgetEntryView: View {
    var entry: QuoteProvider.Entry
    @Environment(\.widgetFamily) var family

    var body: some View {
        switch family {
        case .systemSmall:
            smallWidget
        case .systemMedium:
            mediumWidget
        default:
            mediumWidget
        }
    }

    // 小サイズ
    var smallWidget: some View {
        VStack(spacing: 4) {
            Image(systemName: "quote.opening")
                .font(.caption)
                .foregroundStyle(.blue)

            Text(entry.quote.text)
                .font(.caption)
                .bold()
                .multilineTextAlignment(.center)
                .lineLimit(3)

            Text(entry.quote.author)
                .font(.caption2)
                .foregroundStyle(.secondary)
        }
        .padding(12)
    }

    // 中サイズ
    var mediumWidget: some View {
        HStack(spacing: 16) {
            Image(systemName: "quote.opening")
                .font(.title)
                .foregroundStyle(.blue)

            VStack(alignment: .leading, spacing: 4) {
                Text("今日の名言")
                    .font(.caption2)
                    .foregroundStyle(.secondary)

                Text(entry.quote.text)
                    .font(.subheadline)
                    .bold()

                Text("— \(entry.quote.author)")
                    .font(.caption)
                    .foregroundStyle(.secondary)
            }

            Spacer()
        }
        .padding()
    }
}

// MARK: - ウィジェット定義

@main
struct QuoteWidget: Widget {
    let kind: String = "QuoteWidget"

    var body: some WidgetConfiguration {
        StaticConfiguration(kind: kind, provider: QuoteProvider()) { entry in
            QuoteWidgetEntryView(entry: entry)
                .containerBackground(.fill.tertiary, for: .widget)
        }
        .configurationDisplayName("今日の名言")
        .description("日替わりで名言を表示します")
        .supportedFamilies([.systemSmall, .systemMedium])
    }
}

// MARK: - プレビュー

#Preview(as: .systemMedium) {
    QuoteWidget()
} timeline: {
    QuoteEntry(date: .now, quote: QuoteStore.todaysQuote())
}
*/
```

**このアプリは何をするものか：**

このアプリは、毎日違う名言を表示する名言アプリである。
メインアプリを開くと、上部に「今日の名言」がカードのように表示され、その下にすべての名言リストが表示される。
また、ウィジェットをホーム画面に追加すると、アプリを開かなくても今日の名言を確認できる。

## コードの詳細解説

### TimelineProviderの仕組み

```swift
struct QuoteProvider: TimelineProvider {
    func placeholder(in context: Context) -> QuoteEntry {
        QuoteEntry(
            date: Date(),
            quote: Quote(id: 0, text: "読み込み中...", author: "")
        )
    }

    func getSnapshot(in context: Context, completion: @escaping (QuoteEntry) -> Void) {
        let entry = QuoteEntry(
            date: Date(),
            quote: QuoteStore.todaysQuote()
        )
        completion(entry)
    }

    func getTimeline(in context: Context, completion: @escaping (Timeline<QuoteEntry>) -> Void) {
        let currentDate = Date()
        let quote = QuoteStore.todaysQuote()
        let entry = QuoteEntry(date: currentDate, quote: quote)

        let tomorrow = Calendar.current.startOfDay(
            for: Calendar.current.date(byAdding: .day, value: 1, to: currentDate)!
        )

        let timeline = Timeline(entries: [entry], policy: .after(tomorrow))
        completion(timeline)
    }
}
```

**何をしているか：**

この部分では、ウィジェットに表示するデータと、ウィジェットを更新するタイミングを決めている。
placeholderは読み込み中の仮表示、getSnapshotはウィジェット追加画面などで表示するプレビュー、getTimelineは実際にホーム画面で表示する内容と更新タイミングを設定する。

**なぜこう書くのか：**

ウィジェットは普通のアプリ画面とは違い、常にリアルタイムで動くものではない。そのため、WidgetKitでは「いつ、どの内容を表示するか」をTimelineとして先に渡す必要がある。
このコードでは、今日の名言を表示し、次の日の0時になったら更新されるようにしている。

**もしこう書かなかったら：**

getTimelineを書かなければ、ウィジェットが実際にどのデータを表示すればよいかわからなくなる。
また、更新タイミングを指定しなければ、毎日違う名言を表示する仕組みがうまく動かない可能性がある。

---

### TimelineEntryとウィジェットビュー

```swift
struct QuoteEntry: TimelineEntry {
    let date: Date
    let quote: Quote
}

struct QuoteWidgetEntryView: View {
    var entry: QuoteProvider.Entry
    @Environment(\.widgetFamily) var family

    var body: some View {
        switch family {
        case .systemSmall:
            smallWidget
        case .systemMedium:
            mediumWidget
        default:
            mediumWidget
        }
    }
}
```

**何をしているか：**

QuoteEntryは、ウィジェットがある時点で表示するデータを表している。
このコードでは、表示する日付であるdateと、表示する名言であるquoteを持っている。

QuoteWidgetEntryViewは、実際にウィジェットに表示する画面を作っている。
entry.quote.textやentry.quote.authorを使って、名言の本文と作者を表示する。

**なぜこう書くのか：**

WidgetKitでは、表示するデータをTimelineEntryとして用意し、そのデータをViewに渡して画面を作る。
こうすることで、「データを作る部分」と「画面を表示する部分」を分けて考えることができる。

**もしこう書かなかったら：**

TimelineEntryがないと、ウィジェットに渡すデータの形を決められない。
また、Viewで直接データを取得しようとすると、ウィジェットの更新管理がわかりにくくなり、WidgetKitの仕組みに合わなくなる。

---

### ウィジェットサイズごとのレイアウト

```swift
@Environment(\.widgetFamily) var family

var body: some View {
    switch family {
    case .systemSmall:
        smallWidget
    case .systemMedium:
        mediumWidget
    default:
        mediumWidget
    }
}

var smallWidget: some View {
    VStack(spacing: 4) {
        Image(systemName: "quote.opening")
            .font(.caption)
            .foregroundStyle(.blue)

        Text(entry.quote.text)
            .font(.caption)
            .bold()
            .multilineTextAlignment(.center)
            .lineLimit(3)

        Text(entry.quote.author)
            .font(.caption2)
            .foregroundStyle(.secondary)
    }
    .padding(12)
}

var mediumWidget: some View {
    HStack(spacing: 16) {
        Image(systemName: "quote.opening")
            .font(.title)
            .foregroundStyle(.blue)

        VStack(alignment: .leading, spacing: 4) {
            Text("今日の名言")
                .font(.caption2)
                .foregroundStyle(.secondary)

            Text(entry.quote.text)
                .font(.subheadline)
                .bold()

            Text("— \(entry.quote.author)")
                .font(.caption)
                .foregroundStyle(.secondary)
        }

        Spacer()
    }
    .padding()
}
```

**何をしているか：**

この部分では、ウィジェットのサイズに合わせて表示レイアウトを変えている。
小さいサイズでは縦方向にコンパクトに表示し、中サイズでは横方向にアイコンと文章を並べて表示している。

**なぜこう書くのか：**

ウィジェットはサイズによって使える画面スペースが違う。
小サイズでは表示できる文字数が少ないため、lineLimit(3)を使って最大3行までにしている。
中サイズでは横幅が広いため、HStackを使って見やすく配置している。

**もしこう書かなかったら：**

すべてのサイズで同じレイアウトを使うと、小サイズでは文字が入りきらなかったり、中サイズでは余白が多すぎて見た目が悪くなったりする。
サイズごとにレイアウトを分けることで、どのサイズでも見やすいウィジェットになる。

---

### メインアプリとの連携

```swift
struct Quote: Identifiable, Codable {
    let id: Int
    let text: String
    let author: String
}

struct QuoteStore {
    static let quotes: [Quote] = [
        Quote(id: 1, text: "為せば成る、為さねば成らぬ何事も", author: "上杉鷹山"),
        Quote(id: 2, text: "千里の道も一歩から", author: "老子"),
        Quote(id: 3, text: "継続は力なり", author: "ことわざ"),
        Quote(id: 4, text: "失敗は成功のもと", author: "ことわざ"),
        Quote(id: 5, text: "知ることは愛することの始まりである", author: "ことわざ"),
        Quote(id: 6, text: "学びて思わざれば則ち罔し", author: "孔子"),
        Quote(id: 7, text: "過ちて改めざる、是を過ちと謂う", author: "孔子"),
    ]

    static func todaysQuote() -> Quote {
        let dayOfYear = Calendar.current.ordinality(of: .day, in: .year, for: Date()) ?? 0
        let index = dayOfYear % quotes.count
        return quotes[index]
    }
}
```

**何をしているか：**

Quoteは名言1件分のデータ構造で、QuoteStoreは名言データをまとめて管理している。
todaysQuote()では、今日が1年の中で何日目かを計算し、その数字を使って今日表示する名言を選んでいる。

**なぜこう書くのか：**

メインアプリとウィジェットの両方で同じ名言データを使うため、QuoteStoreを共通のファイルにしておくと便利である。
今回の名言データは固定データなので、UserDefaultsやApp Groupを使わなくても共有できる。

**もしこう書かなかったら：**

メインアプリ側とウィジェット側で別々に名言データを書く必要があり、修正するときに二重管理になってしまう。
また、片方だけ内容を変更してしまうと、アプリとウィジェットで表示内容が違ってしまう可能性がある。

---

## 新しく学んだSwiftの文法・API

| 項目                             | 説明                                      | 使用例                                                             |
| ------------------------------ | --------------------------------------- | --------------------------------------------------------------- |
| `WidgetKit`                    | iOSのホーム画面やロック画面に表示するウィジェットを作るためのフレームワーク | `import WidgetKit`                                              |
| `TimelineProvider`             | ウィジェットの表示内容と更新タイミングを管理する仕組み             | `struct QuoteProvider: TimelineProvider { ... }`                |
| `TimelineEntry`                | ウィジェットがある時点で表示するデータを表す型                 | `struct QuoteEntry: TimelineEntry { let date: Date }`           |
| `Timeline`                     | ウィジェットに表示するデータの予定表                      | `Timeline(entries: [entry], policy: .after(tomorrow))`          |
| `@Environment(\.widgetFamily)` | ウィジェットのサイズを取得するための環境値                   | `@Environment(\.widgetFamily) var family`                       |
| `StaticConfiguration`          | ユーザーによる設定項目がない静的なウィジェットを作る              | `StaticConfiguration(kind: kind, provider: QuoteProvider())`    |
| `@main`                        | アプリやウィジェットの起動入口を示す                      | `@main struct QuoteWidget: Widget { ... }`                      |
| `supportedFamilies`            | 対応するウィジェットサイズを指定する                      | `.supportedFamilies([.systemSmall, .systemMedium])`             |
| `Calendar.current.ordinality`  | 今日が1年の何日目かを取得する                         | `Calendar.current.ordinality(of: .day, in: .year, for: Date())` |
| `Identifiable`                 | `List`でデータを一意に識別できるようにする                | `struct Quote: Identifiable`                                    |


## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：QuoteStoreの名言データを増やして、表示される内容が変わるか確認した。
- 結果：メインアプリのリストに追加した名言が表示された。
- わかったこと：QuoteStore.quotesにデータを追加すると、アプリ側の一覧表示にも反映されることがわかった。

**実験2：**
- やったこと：smallWidgetのlineLimit(3)を削除してみた。
- 結果：長い名言の場合、小さいウィジェットでは文字が多くなり、見た目が窮屈になった。
- わかったこと：小さいウィジェットでは、表示できる範囲が限られているため、lineLimitで行数を制限することが大切だとわかった。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**

   TimelineProviderは何をするものですか？
   
   **得られた理解：**

   ウィジェットの表示内容と更新タイミングを決めるための仕組みだと理解した。普通のアプリ画面のように常に動くのではなく、Timelineを使って更新予定をシステムに渡す必要がある。

2. **質問：**

   TimelineEntryはなぜ必要ですか？
   
   **得られた理解：**

   ウィジェットが「いつ」「どのデータ」を表示するかを表すために必要だとわかった。今回の場合は、日付と名言データを持つQuoteEntryを作っている。

3. **質問：**

   メインアプリとウィジェットで同じデータを使うにはどうすればよいですか？
   
   **得られた理解：**

   QuoteStore.swiftのように共通ファイルを作り、メインアプリとWidget Extensionの両方のTarget Membershipにチェックを入れればよいとわかった。今回のような固定データではApp Groupは不要である。

## この章のまとめ

この章では、WidgetKitを使ってホーム画面に表示できるウィジェットを作る方法を学んだ。
特に重要だと思ったのは、ウィジェットは普通のアプリ画面とは違い、TimelineProviderを使って表示内容と更新タイミングをシステムに渡す必要があるという点である。
また、TimelineEntryで表示するデータを定義し、QuoteWidgetEntryViewでそのデータを画面に表示するという流れも理解できた。
ウィジェットサイズによって使える画面の広さが違うため、.systemSmallと.systemMediumでレイアウトを分けることも大切だと感じた。
今回の名言アプリでは、QuoteStoreをメインアプリとウィジェットで共有することで、同じ名言データを両方で使うことができた。
今後、自分でウィジェットを作るときも、「データ」「更新タイミング」「サイズごとの画面設計」を分けて考えることが重要だと思った。
