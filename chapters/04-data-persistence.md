# 第4章：データの永続化

> 執筆者：ユン　ビョンイル
> 最終更新：2026-07-08

## この章で学ぶこと
この章では、AppStorageとSwiftDataを使ってアプリのデータを端末に永続的に保存する方法を学ぶ。具体的には、@Modelを使ったデータ構造の定義、modelContextを使ったデータの追加・削除、@Queryを使ったリスト表示、そして@AppStorageを使ったユーザー設定（名前や表示順）の保存を実装する。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

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

// MARK: - アプリのエントリポイント
// ※ @main のあるAppファイルに以下を記述してください：
//
// @main
// struct MemoApp: App {
//     var body: some Scene {
//         WindowGroup {
//             ContentView()
//         }
//         .modelContainer(for: Memo.self)
//     }
// }

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

// MARK: - 設定画面（AppStorageの活用）

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

#Preview {
    ContentView()
        .modelContainer(for: Memo.self, inMemory: true)
}
```

**このアプリは何をするものか：**
メモの追加、編集、削除ができるシンプルなメモアプリ。特徴として、設定画面からユーザーの名前を登録でき、お気に入りにしたメモを一番上に表示するかどうかを選択できる。アプリをタスクキルして開き直しても、作成したメモや設定した名前が消えずにそのまま残る。

## コードの詳細解説

### SwiftDataモデル（@Model）

```swift
@Model
class Memo {
    var title: String
    var content: String
    var createdAt: Date
    var isFavorite: Bool
    // (init省略)
}
```

**何をしているか：**
メモのデータ構造（タイトル、内容、作成日時、お気に入りかどうか）を定義している。

**なぜこう書くのか：**
クラスの前に @Model と書くだけで、SwiftDataが自動的にこのクラスをデータベースのテーブルとして認識し、保存できるようにしてくれるから。

**もしこう書かなかったら：**
ただのクラスになってしまうため、アプリを閉じるとメモリから消えてしまい、作成したメモが保存されなくなる。

---

### データの追加・削除（modelContext）

```swift
// 追加部分
let memo = Memo(title: title, content: content)
modelContext.insert(memo)

// 削除部分
func deleteMemos(at offsets: IndexSet) {
    for index in offsets {
        let memo = displayedMemos[index]
        modelContext.delete(memo)
    }
}
```

**何をしているか：**
新しいメモをデータベースに登録（insert）したり、不要になったメモをデータベースから消去（delete）している。

**なぜこう書くのか：**
modelContext はデータベースを操作するための窓口（管理者）のような役割を持っているため。データの変更はこの窓口にお願いして行う。

**もしこう書かなかったら：**
配列に要素を追加・削除するだけのコードだと画面上は変わるかもしれないが、データベースには反映されず、次にアプリを開いたときに元の状態に戻ってしまう。

---

### @Queryによるデータ取得

```swift
@Query(sort: \Memo.createdAt, order: .reverse) private var memos: [Memo]
```

**何をしているか：**
保存されているメモの一覧をデータベースから取得し、作成日時の新しい順（降順）に並べ替えている。

**なぜこう書くのか：**
@Query を使うと、データベースの中身が変更されたとき（追加や削除など）に自動で検知して、画面（View）を再描画してくれるから。

**もしこう書かなかったら：**
メモを追加した後に、手動でリストを更新する処理をわざわざ書かなければ画面に反映されなくなってしまう。

---

### @AppStorageによる設定保存

```swift
@AppStorage("sortByFavorite") private var sortByFavorite: Bool = false
@AppStorage("userName") private var userName: String = ""
```

**何をしているか：**
ユーザーが入力した名前や、お気に入り優先表示のON/OFF状態を端末（UserDefaults）に保存している。

**なぜこう書くのか：**
ユーザー名のような簡単な設定データなら、大がかりなSwiftDataを使わずに、もっと手軽な @AppStorage を使う方がシンプルに書けるから。

**もしこう書かなかったら：**
@State だけで管理すると、アプリを起動するたびに変数が初期値（空文字やfalse）に戻ってしまい、毎回設定し直す必要が出てくる。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`@Model` | SwiftDataでオブジェクトを永続化するためのマクロ | `@Model final class Memo { ... }` |
| 例：`@Query` | データベースからデータを取得し、変更を自動で反映するプロパティラッパー | `@Query var memos: [Memo]` |
| `modelContext` | データの追加(insert)や削除(delete)を実行するための窓口オブジェクト | `modelContext.insert(memo)` |
| `@AppStorage` | UserDefaultsに値を保存・読み込みし、画面と連動させるラッパー | `@AppStorage("userName") var name = ""` |
| `@Bindable` | SwiftDataのモデルデータを、画面から直接書き換え（バインディング）したい時に使う | `@Bindable var memo: Memo` |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：@Query の order: .reverse を .forward に変更してビルドしてみた。
- 結果：一番古いメモが一番上に表示されるようになった。
- わかったこと：ソート順（昇順・降順）はここで簡単に制御できる。

**実験2：**
- やったこと：設定画面の @AppStorage のキー名（"userName"）をわざと別の名前にして読み込んでみた。
- 結果：今まで保存していた名前が表示されず、空文字になった。
- わかったこと：保存する時と読み込む時でキーの文字列が完全に一致していないと、データを取り出せない。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：** SwiftData と AppStorage はどう使い分けるのが正解ですか？
   **得られた理解：** AppStorage は設定やフラグなどの「軽くて単純なデータ」向き。SwiftData はメモや日記などの「数がどんどん増えて、検索や並べ替えが必要なデータ」に向いている。

2. **質問：** @Environment(\.modelContext) って一体何者ですか？
   **得られた理解：**　アプリ全体に用意された「データベースの管理人」を呼び出すためのコード。この管理人に「追加して」「削除して」と命令することでデータが操作される。

3. **質問：** 編集画面で @Binding ではなく @Bindable を使っているのはなぜですか？
   **得られた理解：** @Binding は普通の値（StringやBoolなど）を渡す時に使うが、@Modelで作ったSwiftDataのクラスの中身を直接画面から編集させたい場合は @Bindable を使うのがSwiftDataのルールだから。

## この章のまとめ
データを永続化（保存）するには、用途に合わせてツールを選ぶことが重要。

設定などの軽いデータ は手軽な @AppStorage を使う。
リストや複雑なデータ は SwiftData を使う。

SwiftDataを使う時は、「@Modelで設計図を作り、modelContextで追加・削除を行い、@Queryで画面に表示する」という3点セットの流れを覚えておくこと。
