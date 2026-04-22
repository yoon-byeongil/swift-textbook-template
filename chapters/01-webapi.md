# 第1章：WebAPIの基本

> 執筆者：ユン　ビョンイル
> 最終更新：2000-10-27

## この章で学ぶこと

この章では、インターネット上のサービス（iTunes Search API）から非同期通信でJSONデータを取得し、Swiftの構造体に変換（デコード）して、アプリのリスト画面に表示する一連の流れを学ぶ。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

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

        let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"

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

ユーザーが検索窓にアーティスト名を入力して「検索」ボタンを押すと、裏側でiTunesのサーバーに通信を行い、関連する楽曲データを取得して、曲名、アーティスト名、ジャケット画像をリスト状に表示するアプリ。

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
iTunes APIから返ってくるJSONデータを、Swiftアプリが理解できる「箱（型）」に変換するための設計図を定義している。

**なぜこう書くのか：**
Codableプロトコルに準拠させることで、複雑なJSONの解析処理をJSONDecoderで自動化できるから。
JSONのキー名とプロパティ名を一致させるだけで、自動的にデータがマッピングされる。

**もしこう書かなかったら：**
JSONのデータを手動で1つずつ取り出す処理を何行も書く必要があり、コードが非常に複雑になる。また、型が保証されないためアプリがクラッシュする原因にもなる。

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
検索ワードを組み込んだURLを使ってiTunesサーバーにデータのリクエストを送り、取得したデータをSearchResponse型に変換してsongs配列に格納している。

**なぜこう書くのか：**
async/awaitを使うことで、通信という時間がかかる処理を「裏側」で実行しながら、コード自体は上から下へ直感的に書けるから。

**もしこう書かなかったら：**
コールバック関数（クロージャ）を使った古い書き方になり、処理のネストが深くなってコードが読みにくくなる。
また、UIを更新するためにメインスレッドへ切り替える処理を書き忘れてエラーになるリスクが高まる。

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
現在のアプリの状態に応じて、画面の表示を切り替えている。

**なぜこう書くのか：**
ユーザーに「今アプリが何をしているのか」を視覚的に伝えることで、優れたユーザー体験（UX）を提供するため。

**もしこう書かなかったら：**
通信中も画面が何も変化しないため、ユーザーが「フリーズしたのかな？」と不安になり、何度も検索ボタンを連打してしまう可能性がある。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`Codable` | JSONデータとSwiftの構造体を相互変換するプロトコル | `struct Song: Codable { ... }` |
| 例：`async/await` | 非同期処理を同期的に書ける構文 | `let data = try await URLSession.shared.data(from: url)` |
| `URLSession` | HTTP通信を行うための標準的なAPI | `URLSession.shared.data(from: url)` |
| | | |
| `AsyncImage` | 指定したURLから画像を非同期でダウンロードし、表示するUI部品 | `AsyncImage(url: URL(string: song.artworkUrl100))` |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：APIリクエストURLの末尾にある limit=25 を limit=5 に変更してみた。
- 結果：検索結果のリストが、最大5件までしか表示されなくなった。
- わかったこと：URLのパラメータ（クエリ）を変更することで、サーバーから取得するデータの量や条件を制御できること。

**実験2：**
- やったこと：SongRowにあるAsyncImageのplaceholder（画像読み込み中に出るUI）を、グレーの四角形から ProgressView() に変えてみた。
- 結果：画像が読み込まれるまでの間、くるくると回るローディングインジケーターが表示されるようになった。
- わかったこと：プレースホルダーには任意のViewを配置できるため、ユーザーにより親切なUIを簡単に作れること。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：** Swiftの guard let と if let の違いを教えて。
   **得られた理解：** if letは「値があればその中で使う」条件分岐的な書き方。一方guard letは「値がなければここで処理を終わる」という書き方で、成功したらその後ずっとその値を使える。自分の理解では「if let → あれば使う」「guard let → なければ終わる」と整理できた。

2. **質問：** このコードが全体的にどのように動作するのか、初心者に合わせて教えて。
   **得られた理解：** ユーザーが検索語を入力すると、アプリが裏でAppleのサーバーに尋ねて結果を受け取り、あらかじめ綺麗に作っておいたリスト画面に次々と当てはめて表示する、という一連の流れを理解でた。JSONデータはアプリが理解できる「箱」に入れるイメージ。

3. **質問：** 44行目の await searchMusic() は、なぜただ関数を呼び出さずに await を使うの？
   **得られた理解：** インターネットからデータを取得するのは宅配便のように時間がかかることだから、「荷物が届くまでアプリを止めずに裏で大人しく待ってて！」と指示する必須の信号機のようなものだとわかった。

## この章のまとめ

Web APIを利用するアプリ開発では、「①データの受け皿を作る（Codable）」「②非同期で通信する（async/awaitとURLSession）」「③状態に応じて画面を出し分ける（StateとView）」という3つのステップが基本となる。特に、通信などの時間のかかる処理は、アプリをフリーズさせないために裏側で実行（非同期処理）することが不可欠であることを学んだ。
