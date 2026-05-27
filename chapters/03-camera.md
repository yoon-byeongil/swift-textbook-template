# 第3章：カメラの利用

> 執筆者：尹　ビョンイル
> 最終更新：2026-05-27

## この章で学ぶこと

この章では、SwiftUIのPhotosPickerを用いてデバイスのフォトライブラリから写真を選択する方法と、UIKitのUIImagePickerControllerを連携させてカメラで直接写真を撮影・表示する方法を学びます。

## 模範コードの全体像

```swift
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
デバイスのフォトライブラリから既存の写真を選択、またはカメラを起動して新しく撮影した写真を画面上に表示するアプリです。

## コードの詳細解説

### PhotosPickerによる写真選択

```swift
PhotosPicker(selection: $selectedItem, matching: .images) {
    Label("ライブラリ", systemImage: "photo.on.rectangle")
}
```

**何をしているか：**
iOS標準の写真選択UI（フォトライブラリ）を呼び出し、ユーザーが選択した写真のデータをselectedItemという変数に格納しています。

**なぜこう書くのか：**
PhotosUIフレームワークが提供するこの標準コンポーネントを使用すると、OS側で写真へのアクセス権限やプライバシーを安全に処理してくれるため、最小限のコードで写真選択機能を実装できるからです。

**もしこう書かなかったら：**
標準のPhotosPickerを使用せずに独自で実装しようとすると、写真へのアクセス権限（Info.plistの記述など）の管理や、写真一覧を表示するためのUI構築コードが膨大になり、バグも発生しやすくなります。

---

### 画像の非同期読み込み

```swift
.onChange(of: selectedItem) { _, newItem in
    Task {
        await loadImage(from: newItem)
    }
}
```

**何をしているか：**
写真が選択されたこと（selectedItemの変化）を検知し、別スレッド（Task）を作成して、重いデータ読み込み処理をバックグラウンドで実行しています。

**なぜこう書くのか：**
画像データ（Data型）の展開やUIImageへの変換は処理に時間がかかる場合があるため、Taskとawaitを使って非同期処理にし、アプリの画面（メインスレッド）がフリーズするのを防ぐためです。

**もしこう書かなかったら：**
メインスレッドで同期的に重い画像の読み込みを行うと、読み込みが終わるまでアプリの操作が完全に停止し、画面が固まったように見えてしまいます。

---

### UIViewControllerRepresentableによるカメラ連携

```swift
struct CameraView: UIViewControllerRepresentable {
    @Binding var capturedImage: UIImage?

    func makeUIViewController(context: Context) -> UIImagePickerController {
        let picker = UIImagePickerController()
        picker.sourceType = .camera
        picker.delegate = context.coordinator
        return picker
    }
    // ...
```

**何をしているか：**
SwiftUIの環境内で、UIKitのコンポーネントであるUIImagePickerController（カメラ機能）を呼び出して表示できるようにラップ（包み込み）しています。

**なぜこう書くのか：**
現在のSwiftUIにはカメラを直接起動するネイティブのビューが用意されていないため、従来のUIKitの機能を使用できる形に変換（ブリッジ）する必要があるためです。

**もしこう書かなかったら：**
SwiftUIだけで標準のカメラUIを表示することができず、カメラで直接撮影する機能をアプリに組み込むことができません。

---

### Coordinatorパターン

```swift
class Coordinator: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {
    let parent: CameraView

    init(_ parent: CameraView) { self.parent = parent }

    func imagePickerController(_ picker: UIImagePickerController, didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey: Any]) {
        if let image = info[.originalImage] as? UIImage {
            parent.capturedImage = image
        }
        parent.dismiss()
    }
    // ...
```

**何をしているか：**
UIKit側（カメラ）のイベント（写真を撮り終わった、キャンセルした等）を受け取り、その結果である画像データをSwiftUI側（parent.capturedImage）に伝達しています。

**なぜこう書くのか：**
UIKitの「デリゲート（Delegate）」というイベント通知の仕組みは、SwiftUIと直接通信することができないため、両者の間を取り持つ「仲介役（Coordinator）」を意図的に定義してデータを渡す必要があるからです。

**もしこう書かなかったら：**
ユーザーがカメラで写真を撮影して「写真を使用」ボタンを押しても、その画像データをSwiftUIの変数に渡すことができず、画面に表示されません。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`PhotosPicker` | フォトライブラリから画像を選択するコンポーネント | `PhotosPicker(selection: $selectedItem, matching: .images)` |
| 例：`UIImagePickerController` | カメラまたはフォトライブラリにアクセスするUIKitコンポーネント | `picker.sourceType = .camera` |
| `UIViewControllerRepresentable` | UIKitのビューコントローラをSwiftUIで使えるようにするプロトコル | `struct CameraView: UIViewControllerRepresentable { ... }`　|
|　`Task`/`await` | メインの画面描画を止めずに非同期で重い処理を実行するための構文 | `Task { await loadImage(from: newItem) }` |
| `onChange(of:)` | 特定の値（状態）が変化したことを監視し、変化時に処理を実行するモディファイア | `.onChange(of: selectedItem) { _, newItem in ... }` |

## 自分の実験メモ

**実験1：**
- やったこと：PhotosPickerの引数にあるmatching: .imagesをmatching: .videosに変更して実行してみた。
- 結果：フォトライブラリの選択画面が開いた際、写真（静止画）は表示されず、動画ファイルのみが選択できるようになっていた。
- わかったこと：matchingプロパティを変えるだけで、ユーザーに選択させるメディアの種類（画像、動画、Live Photosなど）を簡単に制限できる。

**実験2：**
- やったこと：imageDisplayArea内の画像表示モディファイア .clipShape(RoundedRectangle(cornerRadius: 16)) を .clipShape(Circle()) に変更してみた。
- 結果：選択した写真の角が丸くなるのではなく、完全な円形（丸型）に切り抜かれて表示された。
- わかったこと：.clipShapeを使えば、ユーザーのアイコン画像などでよく見るような様々な形（円、カプセルなど）へのトリミングが簡単に実装できる。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：** @ViewBuilderとは何ですか？
   **得られた理解：** 関数や計算プロパティの中で、if文などの条件分岐を使って複数のUI部品を構築し、最終的に「ひとつのビュー」としてまとめることができる機能だと分かりました。これを使うとbodyの中身をスッキリ整理できます。

2. **質問：** Coordinator（コーディネーター）とはどのような役割ですか？
   **得られた理解：** UIKit（カメラ）側の動きやユーザー操作（撮影完了、キャンセルなど）を、SwiftUI側に伝達するための「橋渡し役」であることが理解できました。これがないと、カメラで撮影を終えてもSwiftUI側に写真データが渡りません。

3. **質問：** loadTransferableでtype: Data.selfを指定しているのはなぜですか？
   **得られた理解：** 選択した写真ファイルをいきなりImageとして扱うのではなく、一度iOSの汎用的なデータ(バイトの塊であるData型)として読み込んでから、iOS標準の画像型であるUIImageに変換し、最後にSwiftUIのImageにするという安全な手順を踏むためだと分かりました。

## この章のまとめ
この章では、SwiftUIの機能（PhotosPicker）とUIKitの機能（UIImagePickerController）を組み合わせてひとつのアプリを作る方法を学びました。新しいフレームワークであるSwiftUIだけでは補いきれない部分を、UIViewControllerRepresentableやCoordinatorを使って従来のUIKitと連携させる技術は少し複雑でしたが、これを使えるようになると実装の幅が大きく広がることが分かりました。また、画像の読み込み時にアプリをフリーズさせないための非同期処理（Taskとawait）の重要性も、ユーザー体験（UX）を向上させる上で欠かせない実践的な学びになりました。
