# 第1章：WebAPIの基本

> 執筆者：ユン　ビョンイル
> 最終更新：2026-04-22

## この章で学ぶこと

Web API（今回はiTunes Search API）を使って、インターネットからJSONというデータをもらい、それをSwiftの形（構造体）に変えて、画面にリストとして表示する一連の流れを学ぶ。
また、通信にかかる時間でアプリが止まらないようにする「非同期通信」という少し難しそうな概念も、実際にコードを動かして理解する。

## 模範コードの全体像

```swift
// ============================================
// 第1章（基本）：iTunes Search APIで音楽を検索するアプリ
// ============================================
// このアプリは、iTunes Search APIを使って
// 音楽（曲）を検索し、結果をリスト表示します。
// APIキーは不要で、すぐに動かすことができます。
// ============================================

import SwiftUI

// MARK: - データモデル

struct SearchResponse: Codable {
    let results: [Song]
}

struct Song: Codable, Identifiable {
    let trackId: Int
    let trackName: String
    let artistName: String
    let artworkUrl100: String
    let previewUrl: String?

    var id: Int { trackId }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var songs: [Song] = []
    @State private var searchText: String = ""
    @State private var isLoading: Bool = false

    var body: some View {
        NavigationStack {
            VStack {
                // 検索バー
                HStack {
                    TextField("アーティスト名を入力", text: $searchText)
                        .textFieldStyle(.roundedBorder)

                    Button("検索") {
                        Task {
                            await searchMusic()
                        }
                    }
                    .buttonStyle(.borderedProminent)
                    .disabled(searchText.isEmpty)
                }
                .padding(.horizontal)

                // 検索結果リスト
                if isLoading {
                    ProgressView("検索中...")
                        .padding()
                    Spacer()
                } else if songs.isEmpty {
                    ContentUnavailableView(
                        "曲を検索してみよう",
                        systemImage: "music.note",
                        description: Text("アーティスト名を入力して検索ボタンを押してください")
                    )
                } else {
                    List(songs) { song in
                        SongRow(song: song)
                    }
                }
            }
            .navigationTitle("Music Search")
        }
    }

    // MARK: - API通信

    func searchMusic() async {
        guard let encodedText = searchText.addingPercentEncoding(
            withAllowedCharacters: .urlQueryAllowed
        ) else { return }

        let urlString = "[https://itunes.apple.com/search?term=](https://itunes.apple.com/search?term=)\(encodedText)&media=music&country=jp&limit=25"

        guard let url = URL(string: urlString) else { return }

        isLoading = true

        do {
            let (data, _) = try await URLSession.shared.data(from: url)
            let response = try JSONDecoder().decode(SearchResponse.self, from: data)
            songs = response.results
        } catch {
            print("エラー: \(error.localizedDescription)")
            songs = []
        }

        isLoading = false
    }
}

// MARK: - 曲の行ビュー

struct SongRow: View {
    let song: Song

    var body: some View {
        HStack(spacing: 12) {
            AsyncImage(url: URL(string: song.artworkUrl100)) { image in
                image
                    .resizable()
                    .aspectRatio(contentMode: .fill)
            } placeholder: {
                Color.gray.opacity(0.3)
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
        }
        .padding(.vertical, 4)
    }
}

#Preview {
    ContentView()
}
```

**このアプリは何をするものか：**

検索窓に好きなアーティストの名前を入れて「検索」を押すと、アプリが裏でAppleのサーバーにお願いして、曲の名前やジャケットの画像をもらってきて、リストで綺麗に表示してくれるアプリ！

## コードの詳細解説

### データモデル（Codable構造体）

```swift
struct SearchResponse: Codable {
    let results: [Song]
}

struct Song: Codable, Identifiable {
    let trackId: Int
    let trackName: String
    // ...
    var id: Int { trackId }
}
```

**何をしているか：**
iTunes APIから返ってくるJSONデータ（ただの文字の集まり）を、Swiftアプリが使いやすいように「箱（構造体）」の設計図を作っている。

**なぜこう書くのか：**
`Codable`プロトコルに準拠させると、JSONのキー名とSwiftの変数名を合わせるだけで、面倒な変換処理をSwiftが自動でやってくれるから。すごく便利！

**もしこう書かなかったら：**
JSONの中から「ここは曲名、ここはアーティスト名…」と手作業で1つずつ取り出すコードを長く書かないといけなくて、バグが出やすくなる（アプリが落ちる原因になる）。

---

### API通信の処理

```swift
func searchMusic() async {
    // ...URLの生成処理...
    isLoading = true
    do {
        let (data, _) = try await URLSession.shared.data(from: url)
        let response = try JSONDecoder().decode(SearchResponse.self, from: data)
        songs = response.results
    } catch {
        print("エラー: \(error.localizedDescription)")
    }
    isLoading = false
}
```

**何をしているか：**
検索ワードを入れたURLを作って、iTunesのサーバーにデータのリクエストを送り、受け取ったデータをさっき作った `SearchResponse` という箱に入れて、 `songs` 配列に保存している。

**なぜこう書くのか：**
インターネット通信は時間がかかるので、`async/await` を使って「裏側」で待機させるため。これを使うと、上から下へ普通のコードのように直感的に読みやすく書ける。

**もしこう書かなかったら：**
コールバック関数を使った昔の書き方になり、コードが複雑になる。また、画面を更新する処理をメインスレッドに戻すのを忘れて、アプリがエラーになるリスクが高まるらしい。

---

### ビューの構成

```swift
if isLoading {
    ProgressView("検索中...")
} else if songs.isEmpty {
    ContentUnavailableView("曲を検索してみよう", ...)
} else {
    List(songs) { song in
        SongRow(song: song)
    }
}
```

**何をしているか：**
「検索中...」なのか、「まだ何も検索してない」のか、「結果が出た」のか、今のアプリの状態（State）に合わせて画面の表示を切り替えている。

**なぜこう書くのか：**
ユーザーから見て「今アプリが何をしているか」が分からないと、壊れたと思ってボタンを何度も押してしまうかもしれないから。UX（ユーザー体験）を良くするため。

**もしこう書かなかったら：**
通信中も画面がずっと止まったままになり、ユーザーを不安にさせてしまう。

---

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| `Codable` | JSONデータとSwiftの構造体をくっつける（相互変換する）ための便利なプロトコル | `struct Song: Codable { ... }` |
| `async/await` | 通信など時間がかかる処理を、アプリを止めずに裏で待たせるための書き方 | `let data = try await URLSession.shared.data(from: url)` |
| `URLSession` | インターネット通信（HTTP通信）をするためのSwiftの標準機能 | `URLSession.shared.data(from: url)` |
| `AsyncImage` | URLを渡すだけで、インターネット上の画像を裏でダウンロードして表示してくれるUI部品 | `AsyncImage(url: URL(string: song.artworkUrl100))` |

## 自分の実験メモ

**実験1：**
- やったこと：APIリクエストURLの末尾にある `limit=25` を `limit=5` に変更してみた。
- 結果：検索結果のリストが、本当に5件までしか表示されなくなった！
- わかったこと：URLの末尾（クエリパラメータ）を変えるだけで、サーバーからもらうデータの数やルールを直接コントロールできること。

**実験2：**
- やったこと：`SongRow`にある`AsyncImage`の`placeholder`（画像読み込み中に出る仮のUI）を、グレーの四角形から `ProgressView()` に変えてみた。
- 結果：画像が読み込まれるまでの間、くるくると回るローディングインジケーターが表示されるようになった。
- わかったこと：プレースホルダーには自分の好きなViewを配置できるため、少し工夫するだけでユーザーに優しい画面が作れる！

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：** Swiftの `guard let` と `if let` の違いを教えて。
   **得られた理解：** `if let`は「値があればその中で使う」条件分岐のような書き方。一方`guard let`は「値がなければここで処理を終わる（return）」という書き方で、成功したらその後ずっとその値を使える。「if let → あれば使う」「guard let → なければ終わる」と整理できた。関数の最初でチェックする時によく使うとわかった。

2. **質問：** このコードが全体的にどのように動作するのか、初心者に合わせて教えて。
   **得られた理解：** ユーザーが検索 → アプリが裏でAppleに注文（通信） → 受け取ったJSONを「箱（Codable）」に入れる → 画面のリストに並べる、という一連の流れ。JSONデータはアプリが理解できる「箱」に入れるイメージがはっきりした。

3. **質問：** 44行目の `await searchMusic()` は、なぜただ関数を呼び出さずに `await` を使うの？
   **得られた理解：** インターネットからデータを取得するのは宅配便のようにいつ届くかわからないから、「荷物が届くまでアプリをフリーズさせずに裏で大人しく待ってて！」と指示する必須の信号機のようなものだとわかった。（この例えが一番しっくりきた！）

## この章のまとめ

Web APIを使うアプリの開発は、大きく分けて３つのステップが基本になる。
「①データの受け皿を作る（Codable）」「②非同期で通信する（async/awaitとURLSession）」「③状態に応じて画面を出し分ける（StateとView）」。
特に、通信などの時間がかかる処理は、アプリをフリーズさせないために裏側で実行（非同期処理）することが、スマホアプリを作る上で一番大事なルールだと学んだ。少しずつSwiftの全体像が見えてきて面白い。
