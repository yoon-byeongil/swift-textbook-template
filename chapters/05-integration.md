# 第5章：機能統合の実践

> 執筆者：ユン　ビョンイル
> 最終更新：2026-07-08

## この章で学ぶこと
この章では、これまでに学んだ技術（地図、写真選択、データ保存）をすべて組み合わせて、「フォトマップ」アプリを作る方法を学ぶ。具体的には、位置情報(GPS)を取得して写真・メモと一緒にSwiftDataへ保存し、タブ画面を使って「地図上のピン」と「リスト」の2つの形式でデータを表示する実践的なアーキテクチャ設計を理解する。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
// ============================================
// 第5章：カメラ + 地図 + データ保存の統合アプリ
// ============================================
// 写真を撮影し、撮影場所を地図上に記録する
// 「フォトマップ」アプリです。
// 第2〜4章で学んだ技術を組み合わせて使います。
//
// 【注意】Info.plist に以下のキーを追加してください：
//   - NSLocationWhenInUseUsageDescription
//   - NSPhotoLibraryAddUsageDescription
//   - NSCameraUsageDescription（実機の場合）
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
フォトライブラリから写真を選び、タイトルやメモと一緒に「今の場所(GPS情報)」を記録できるアプリ。保存したデータは、マップ上に丸く切り抜かれた写真アイコンとしてピン留めされ、一覧画面からはリスト形式で確認・削除ができる。詳細画面では、記録した内容と撮影場所のミニマップが表示される。

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
    // ...
    var coordinate: CLLocationCoordinate2D {
        CLLocationCoordinate2D(latitude: latitude, longitude: longitude)
    }
}
```

**何をしているか：**
記録として保存する項目をSwiftDataの @Model で定義しつつ、地図へピンを立てるために必要な CLLocationCoordinate2D を生成する便利なプロパティ(coordinate)を持たせている。

**なぜこう書くのか：**
SwiftDataはデータベースなので、標準的な型（StringやDouble、Dataなど）しか保存できない。そのため、緯度・経度を別々の Double 型で保存しておき、アプリ内で使うときだけMap用の型に変換するという工夫をしている。

**もしこう書かなかったら：**
保存機能と地図機能でデータの型が合わず、地図上に保存したピンを表示することができなくなってしまう。

---

### タブ構成の設計

```swift
TabView {
    MapTab()
        .tabItem { Label("マップ", systemImage: "map") }

    ListTab()
        .tabItem { Label("一覧", systemImage: "list.bullet") }
}
```

**何をしているか：**
画面の下部にタブバーを表示し、「地図画面（MapTab）」と「リスト画面（ListTab）」をタップで切り替えられるようにしている。

**なぜこう書くのか：**
地図とリストという2つの異なる見せ方を、ユーザーが迷わず並行して使えるようにするため。

**もしこう書かなかったら：**
ボタンなどで別画面に遷移(NavigationLink)させなければならず、画面の行き来が面倒で使いにくいアプリになってしまう。

---

### カメラと位置情報の連携

```swift
// 位置情報取得部分
@Observable
class LocationManager: NSObject, CLLocationManagerDelegate {
    private let manager = CLLocationManager()
    // ...
    manager.startUpdatingLocation()
}
```

**何をしているか：**
UIKit時代のフレームワークであるCore Locationの CLLocationManager を使い、現在地の緯度・経度をリアルタイムで取得・更新し続けている。

**なぜこう書くのか：**
SwiftUI単体では継続的なGPS取得が難しいため、従来のAPIを NSObject や CLLocationManagerDelegate を使ってラップ（包んで使いやすく）している。

**もしこう書かなかったら：**
「どこで記録したか」というGPS情報が得られず、フォトマップの最も重要な機能が失われてしまう。

---

### SwiftDataでの画像保存

```swift
// 保存時の処理
let record = PhotoRecord(
    // ...
    imageData: selectedImageData
)
modelContext.insert(record)
```

**何をしているか：**
PhotosPickerから取得した画像データを Data 型として保存している。

**なぜこう書くのか：**
SwiftDataは UIImage や Image のままでは直接データベースに保存できないため、一度バイナリデータ（Data）に変換してから保存する必要があるから。

**もしこう書かなかったら：**
画像をデータベースに記録できず、アプリを再起動したときに写真が全て消えてしまう。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`TabView` | 複数のビューをタブで切り替えるコンポーネント | `TabView { ... }.tabViewStyle(.page)` |
| 例：`CLLocationManager` | GPS位置情報を取得するAPIManager | `let location = manager.location?.coordinate` |
| `PhotosPicker` | フォトライブラリを開き、写真を選択させるビュー | `PhotosPicker(selection: $selectedItem, matching: .images)` |
| `Annotation` | マップ上に任意のビュー（画像など）をピンとして配置できる機能 | `Annotation("タイトル", coordinate: record.coordinate) { ... }` |
| | | |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：マップ上のピンの見た目を変えるため、MapTab の clipShape(Circle()) を clipShape(RoundedRectangle(cornerRadius: 8)) に変更してみた。
- 結果：地図上の写真アイコンが丸ではなく、角丸の四角形になった。
- わかったこと：地図上の Annotation の中身は普通のSwiftUIと同じようにモディファイアで自由にデザインできる。

**実験2：**
- やったこと：AddRecordView の保存ボタンの条件から locationManager.currentLocation == nil のガードを外してビルドしてみた。
- 結果：位置情報の取得が間に合っていない状態で保存ボタンを押すと、予期せぬエラーが起きる可能性が出てきた。
- わかったこと：必須データ（この場合はGPS座標）が揃うまでは保存ボタンを .disabled(true) にして押させないようにする設計がアプリの安定化にはとても重要。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：** NSObject とか CLLocationManagerDelegate って何ですか？急に書き方が古く見えます。
   **得られた理解：** AppleのCore Location（位置情報）などはObjective-Cという古い言語から使われている仕組みを引き継いでいるため、SwiftUIの新しい書き方だけでなく、昔のルールに従ってデリゲート（通知係）を設定する必要がある。

2. **質問：** Info.plistにキーを追加し忘れるとどうなりますか？
   **得られた理解：** アプリがユーザーの許可なしに位置情報やカメラを使おうとしたと判定され、起動後または機能を使おうとした瞬間にクラッシュ（強制終了）する。iOSのプライバシー保護ルールはとても厳しい。

3. **質問：** @Observable って何ですか？
   **得られた理解：** クラスのデータが変更されたときに、自動でSwiftUIの画面に「値が変わったよ！」と知らせて画面を更新させるための最新の仕組み。（以前は ObservableObject や @Published と書いていたもの）。

## この章のまとめ
地図(MapKit)、写真選択(PhotosUI)、データベース(SwiftData)という別々の機能を、ひとつのアプリとして破綻なく連携させる「アーキテクチャの作り方」を学んだ。

特に大事なのは「データの持ち方（モデル設計）」。データベースに保存しやすい型（DataやDouble）で保存し、使うときに画面用の型（UIImageやCLLocationCoordinate2D）に変換する、という役割分担をModelクラスの中にしっかり書いておくことが、複雑なアプリを作るコツだと分かった。
