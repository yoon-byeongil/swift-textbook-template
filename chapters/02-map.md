# 第2章：地図アプリの基本

> 執筆者：ユン　ビョンイル
> 最終更新：2026-05-13

## この章で学ぶこと

SwiftUIとMapKitを使用して、アプリ内に地図を表示し、特定の観光スポットにマーカーを配置する方法を学びます。また、構造体を用いてデータを管理し、ユーザーの操作に応じて表示するマーカーをカテゴリ別にフィルタリングする機能の実装方法を習得します。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

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
            // 地図
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

            // カテゴリフィルター
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
            // 地図の操作に応じた処理を追加できる
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
東京の代表的な観光スポットを地図上にピン（マーカー）として表示するアプリです。画面下部のボタンをタップすると「寺社」「タワー」「公園」といったカテゴリごとに表示・非表示を切り替えることができ、地図上のマーカーをタップするとその場所の詳細情報がカード形式でポップアップ表示されます。

## コードの詳細解説

### データモデル（ランドマーク構造体）

```swift
struct Landmark: Identifiable, Hashable {
    let id = UUID()
    let name: String
    let description: String
    let coordinate: CLLocationCoordinate2D
    let category: Category

    static func == (lhs: Landmark, rhs: Landmark) -> Bool {
        lhs.id == rhs.id
    }
    
    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
    }
```

**何をしているか：**
地図に表示する各スポットのデータ（名前、説明、座標、カテゴリ）をまとめた設計図（モデル）を定義しています。

**なぜこう書くのか：**
複数の関連するデータを一つのまとまりとして扱うためです。Identifiableでリスト表示やループ処理時の識別のためのIDを持たせ、Hashableで地図上の選択（タップ）時にどのマーカーが選ばれたかを一意に判定できるようにしています。

**もしこう書かなかったら：**
Hashableプロトコルへの準拠を省略すると、マーカーをタップしてもシステムがどのデータが選択されたか識別できず、詳細情報の表示（状態の更新）が機能しなくなります。

---

### 地図の表示とカメラ制御

```swift
@State private var cameraPosition: MapCameraPosition = .region( ... )
@State private var selectedLandmark: Landmark?

// ...

Map(position: $cameraPosition, selection: $selectedLandmark) { ... }
```

**何をしているか：**
地図の初期表示位置（東京の中心）とズームレベルを設定し、ユーザーの操作や選択状態（タップされたマーカー）を監視して地図を描画しています。

**なぜこう書くのか：**
@Stateを使って状態を管理し、$をつけてBindingとして渡すことで、ユーザーが地図を動かしたりピンをタップしたりした際に、SwiftUIが自動的に画面を再描画（更新）してくれるからです。

**もしこう書かなかったら：**
@Stateを使わずにただの定数や変数にすると、値が変化しても画面に反映されず、ユーザーの操作を受け付けない静止画のような地図になってしまいます。

---

### マーカーの表示

```swift
ForEach(filteredLandmarks) { landmark in
    Marker(
        landmark.name,
        systemImage: landmark.category.iconName,
        coordinate: landmark.coordinate
    )
    .tint(landmark.category.color)
    .tag(landmark)
}
```

**何をしているか：**
フィルタリングされたデータの数だけ、地図上の指定された座標にアイコンと色を指定したマーカーを配置しています。

**なぜこう書くのか：**
ForEachループと組み合わせることで、配列内のデータが増減しても自動的にすべてのマーカーを描画できるためです。また、.tag(landmark)でデータ本体をビューに紐付けることで、タップによる選択機能を実現しています。

**もしこう書かなかったら：**
.tag(landmark)を付与しない場合、ユーザーがマーカーをタップしてもselectedLandmarkにデータが渡らず、結果として詳細情報カードが表示されなくなります。

---

### フィルター機能

```swift
var filteredLandmarks: [Landmark] {
    Landmark.sampleData.filter { selectedCategories.contains($0.category) }
}
```

**何をしているか：**
すべてのランドマークデータ（sampleData）の中から、現在ユーザーが選択しているカテゴリ（selectedCategories）に含まれるものだけを抽出して、表示用の新しい配列を作っています。

**なぜこう書くのか：**
データを直接削除・変更するのではなく、「表示用の計算プロパティ」を用意することで、元のデータを破壊せずに安全かつ高速に表示を切り替えられるためです。

**もしこう書かなかったら：**
元の配列自体を変更・削除する処理にしてしまうと、一度非表示にしたカテゴリのデータを再度表示させる際に、データを再取得または再生成する余計な手間がかかってしまいます。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`Map` | SwiftUIで地図を表示するビューコンポーネント | `Map(position: .constant(.region(region)))` |
| 例：`Marker` | 地図上に位置をマーキングするコンポーネント | `Marker("名前", coordinate: coordinate)` |
| `CLLocationCoordinate2D` | 緯度(latitude)と経度(longitude)を保持する構造体 | `CLLocationCoordinate2D(latitude: 35.6812, longitude: 139.7671)` |
| `Set` | 重複を許さず、順序を持たないコレクション（カテゴリ管理などに使用） | `Set(Landmark.Category.allCases)` |
| | | |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：
- 結果：
- わかったこと：

**実験2：**
- やったこと：
- 結果：
- わかったこと：

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**   SetはArray（配列）とどう違うのですか？   
   **得られた理解：** Setは要素の重複を許さず、順序も持ちません。今回のカテゴリ選択のように、「特定のカテゴリが含まれているか(contains)」を頻繁にチェックする処理においては、配列よりも検索が高速で適していることが理解できました。

2. **質問：** @Stateと通常の変数は何が違いますか？
   **得られた理解：** SwiftUIにおいてビューは一度描画されると固定されますが、@Stateを付けた変数は「状態」として監視され、値が変更されるとSwiftUIがそれを検知して自動的に画面（ビュー）を再描画してくれるトリガーになることがわかりました。

3. **質問：**
   **得られた理解：**

## この章のまとめ

（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）
