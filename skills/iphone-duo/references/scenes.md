# シーンとヒンジ

## ヒンジ

ヒンジの状態と角度をアプリから読めます。状態は**閉じた状態・部分的に開いた状態・完全に開いた状態**の3つで、あわせて角度が連続的に更新されます。

```swift
// SwiftUI
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

クロージャは変更前後のコンテキストを受け取ります。`nil` チェックを省くとヒンジのない端末で意図しない挙動になるため、必ず確認してください。

**用途の切り分けが重要です。** ライブのヒンジデータはインタラクションやエフェクト向けです。レイアウトの決定には arrangement と reserved regions の API を使ってください。ヒンジ角度から直接レイアウトを組むと、システムが提供する適応と二重になります。

## マルチタスキング

iPhone Duo では全アプリがマルチタスキングに参加します。2つのアプリを左右に並べる配置と、動画とアプリを上下に重ねる配置がありますが、**アプリ側からはどちらも同じ**です。与えられたサイズに追従するだけで済みます。

分割表示もピクチャ・イン・ピクチャの上部固定も、システムが提供する操作です。これらを実装するための API はありません。size class と scene のジオメトリで判断してください。

iPad や iPhone ミラーリングでのリサイズにすでに対応しているなら、その時点で有利です。

## 複数ウインドウ

iPhone Duo は、アプリの UI を複数インスタンス表示できる最初の iPhone です。iPad で対応しているアプリは iPhone Duo でも対応します。

**ただし新規ウインドウを作成できるのは内側ディスプレイだけ**で、外側ディスプレイでは作成できません。この可否が動的に変わるのが iPhone Duo 固有の挙動です。

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

- `CameraCaptureAccessory` — カメラ利用時に外側ディスプレイへ追加 UI を出す。条件は内側ディスプレイでの全画面表示とアクティブなカメラセッション。**カメラ UI と同じビューに登録する**と、カメラが表示されているときだけアクセサリが出る
- `ExternalNonInteractiveAccessory` — 外部ディスプレイへ操作を伴わないコンテンツを出す

UIKit では `UISceneAccessory` が対応し、利用可否は `UISceneAccessoryRegistration.isAvailable` で確認します。
