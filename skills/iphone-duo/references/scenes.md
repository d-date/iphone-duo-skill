# シーンとヒンジ

## ヒンジ

ヒンジの状態と角度をアプリから読めます。状態は**閉じた状態・部分的に開いた状態・完全に開いた状態**の3つで、あわせて角度が連続的に更新されます。

```swift
// SwiftUI: onHingeChange(isEnabled:_:)（iOS 27.1+ Beta）
// 宣言は (DeviceHingeContext, DeviceHingeContext) -> Void。isEnabled には既定値があるので省略できる
someView
    .onHingeChange { _, context in
        // context の hinge が nil ならヒンジのないデバイス
        if let hinge = context.hinge, hinge.status == .partiallyOpen {
            value = compute(from: hinge.angle)   // angle は Angle 型
        } else {
            value = 0
        }
    }
```

UIKit は `UIHingeInteraction` が対応します。

クロージャは変更前後の `DeviceHingeContext` を受け取ります。`DeviceHingeContext` が持つのは `hinge`（`DeviceHinge?`）だけです。`DeviceHinge` は `angle` と `status` を持ち、`DeviceHinge.Status` は `closed` / `partiallyOpen` / `fullyOpen` の3つです。`nil` チェックを省くとヒンジのない端末で意図しない挙動になるため、必ず確認してください。

**用途の切り分けが重要です。** ライブのヒンジデータはインタラクションやエフェクト向けです。レイアウトの決定には arrangement と reserved regions の API を使ってください。ヒンジ角度から直接レイアウトを組むと、システムが提供する適応と二重になります。

## マルチタスキング

iPhone Duo では全アプリがマルチタスキングに参加します。2つのアプリを左右に並べる配置と、動画とアプリを上下に重ねる配置がありますが、**アプリ側からはどちらも同じ**です。与えられたサイズに追従するだけで済みます。

分割表示もピクチャ・イン・ピクチャの上部固定も、システムが提供する操作です。これらを実装するための API はありません。size class と scene のジオメトリで判断してください。

iPad や iPhone ミラーリングでのリサイズにすでに対応しているなら、その時点で有利です。

外部ディスプレイの扱いは通常の iPhone と同じだろう、というのが Group Lab での見立てです（接続して動くこと自体は確認されていますが、挙動全体の検証結果ではありません）。iPad の Stage Manager のようなマルチタスキングには対応しない見込みです。

## 複数ウインドウ

iPhone Duo は、アプリの UI を複数インスタンス表示できる最初の iPhone です。iPad で対応しているアプリは iPhone Duo でも対応します。仕組みは iPad と同じで、同じアプリのインスタンス同士も、他のアプリとも並べられます。`@AppStorage` のような保存先を参照している場合、状態はインスタンス間で共有されます。

**ただし新規ウインドウを作成できるのは内側ディスプレイだけ**で、外側ディスプレイでは作成できません。この可否が動的に変わるのが iPhone Duo 固有の挙動です。

音声はシーンごとに分かれないという見立てが Group Lab で示されました（確答ではありません）。音声の制御は UI とは別の層にあり、コントロールセンターに音量スライダーが2つ出ることはありません。分けたいなら自前でミキシングすることになります。iPad でも同じアプリの2つのシーンを開けるので、そこで挙動を確かめてください。

```swift
// 作成できない状況では自動的に隠れる
UIWindowScene.ActivationAction   // UIKit

// SwiftUI は openWindow と supportsMultipleWindows で可否を判断する
@Environment(\.supportsMultipleWindows) private var supportsMultipleWindows
```

要求が失敗する場合に備えてエラーを処理します。UIKit では `UIApplication.activateSceneSession(for:errorHandler:)`（iOS 17.0+）で要求し、`UISceneError.Code` で理由を判別します。`.requestDenied` は外側ディスプレイなどで作成できない状況、`.multipleScenesNotSupported` はアプリが複数シーンに対応していない場合です。

## scene accessories

メイン UI に付随するコンテンツを別のディスプレイへ同時に表示する仕組みです。iPhone Duo 専用ではなく iPhone と iPad の機能で、外部ディスプレイにゲームを出して iPhone をコントローラーにする、といった用途があります。

利用可否はシステムが動的に管理します。既定では有効ですが随時切り替わるため、**変化に追従する前提で組んでください**。

```swift
// SwiftUI（iOS 27.0+）
CameraView(model: model)
    .sceneAccessory {
        CameraCaptureAccessory(isEnabled: $model.isEnabled) {
            TeleprompterView(model: model)
        }
        .onAvailabilityChange { newValue in
            model.isAvailable = newValue
        }
    }
```

`sceneAccessory(content:)` に渡す内容は `SceneAccessoryContent` に準拠している必要があります。

- `CameraCaptureAccessory`（iOS 27.1+ Beta） — カメラ利用時に外側ディスプレイへ追加 UI を出す。次節を参照
- `ExternalNonInteractiveAccessory` — 外部ディスプレイへ操作を伴わないコンテンツを出す

UIKit では `UISceneAccessory` が対応し、利用可否は `UISceneAccessoryRegistration.isAvailable` で確認します。`sceneAccessory(content:)`、`UISceneAccessory`、`UISceneAccessoryRegistration`、`registerSceneAccessory(_:)`、`onAvailabilityChange(perform:)` はいずれも iOS 27.0 です。

置ける内容に制限はありません。ウィジェットとは違い、渡されるのは完全な UIScene なので、アプリの他の画面と同じように作れます。アニメーションや更新頻度についても、Group Lab の回答者は把握している制限はないと述べています。

**内側と外側のディスプレイを同時に点灯させられるのはカメラアプリだけです。**テント姿勢で内側ディスプレイを光らせる時計のような表現は AlarmKit が提供する機能です。アラーム以外で同じことはできないだろう、というのが Group Lab での回答でした。1日目の Group Lab ではこれにシステムの entitlement が必要だと述べられましたが、名前や申請方法は公開されておらず、Apple のドキュメント側にも entitlement への言及はありません。**推測で書かないでください。**

## カメラアクセサリの登録

撮影される側に何かを見せるための仕組みです。背面カメラと外側ディスプレイが同じ方向を向くため、台本・カウントダウン・カメラに写っている範囲などをカメラの前に立つ人へ出せます。

**設計の前提:**

- 表示先はシステムが決める。アプリが渡すのはコンテンツの種類だけで、ディスプレイもキャプチャセッションもカメラも指定しない
- 外側ディスプレイはタッチを受け付けるが、プレビューをタップしてフォーカスを合わせる程度の単一の操作に留める。2つ目の操作画面にはしない
- **欠かせない操作は撮影画面の側に置く。**システムはいつでもコンテンツを引っ込められる。外側ディスプレイがない端末でも、システムが何も出さないときでも、撮影画面だけで完結するよう設計する

**登録先は撮影画面を表示しているビューです。**そのビューが画面にある間だけコンテンツが出て、別の画面へ移ると止まります。

```swift
// UIKit
let configuration = UISceneConfiguration()
configuration.delegateClass = ScriptSceneDelegate.self

let accessory = UISceneAccessory.cameraCapture(sceneConfiguration: configuration,
                                               userInfo: model)
registration = registerSceneAccessory(accessory)   // 戻り値を強参照で保持する
```

- `cameraCapture(sceneConfiguration:userInfo:)` と `UISceneSession.Role.windowCameraCaptureAccessory` は iOS 27.1+ Beta
- 戻り値の `UISceneAccessoryRegistration` は強参照で保持する。提供自体をやめるときだけ `unregisterSceneAccessory(_:)` を呼ぶ
- session role はシステムが割り当てる。シーンマニフェストに項目を書いても効果はない。1つのシーンデリゲートで複数種類のシーンを扱うなら `windowCameraCaptureAccessory` と比較して判別する

**状態は送り合わず、同じオブジェクトを両方から読ませます。**SwiftUI ではコンテンツのクロージャが周囲の状態を捉えるので observable なモデルをそのまま渡せます。UIKit では `userInfo` に渡し、シーン接続時に `UIScene.ConnectionOptions.sceneAccessoryUserInfo` から取り出します。渡したオブジェクトはアプリ側で強参照を保ってください（アクセサリは置き場所ではありません）。

**利用可否とオン・オフは別物です。**

| | 決めるのは | 読み書き |
|---|---|---|
| 利用可否 | システムのみ | SwiftUI `onAvailabilityChange(perform:)` / UIKit `isAvailable` |
| オン・オフ | アプリ | SwiftUI `CameraCaptureAccessory(isEnabled:)` / UIKit `isEnabled` |

`isAvailable` は observation に対応しているので、`updateProperties()` の中で読めば通知なしで追随します。

利用可否が変わる条件:

- キャプチャが止まった、アプリが前面から外れた、端末を閉じた
- 開いた状態の Split View では使えない
- 同じ種類の登録は最も手前のものだけが表示される。自前のコンテンツを登録する画面へ移ると前の登録は利用不可になり、戻ると復帰する
- 種類の違うアクセサリ同士は競合しない（外部ディスプレイへスライドを出しつつ外側ディスプレイへ撮影用コンテンツを出せる）

表示先がないときは登録が非アクティブのまま `isAvailable` が false を返し続けるので、外側ディスプレイのない端末も含めて1つの経路で書けます。**コンテンツを止めたいときは登録を解除せず、撮影画面にオン・オフの操作を置いてください。**既定は有効です。

検証はコンテンツを通常のビューで作ればプレビューやシミュレータでレイアウトを確認できますが、シミュレータにカメラがないため、カメラに依存する部分は必ず実機で確認します。
