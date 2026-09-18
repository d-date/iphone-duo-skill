# カメラ

## 2つの前面カメラ

iPhone Duo は正方形センサーと超広角の画角を持つ前面カメラを2つ持ちます。外側ディスプレイ側と、内側のディスプレイ下埋め込み型です。

選択肢は2つあります。

| | 解像度・フレームレート | 深度 | 開閉時の切り替え |
|---|---|---|---|
| Virtual Front Camera | 最大 1080p・60fps（両カメラ共通の機能のみ） | 非対応 | 自動 |
| 個別のカメラ | 内側 1080p・最大60fps / 外側 最大 4K・120fps | 対応 | アプリが担当 |

前面位置と Wide または Ultra Wide のデバイスタイプで探索すると Virtual Front Camera が見つかります。

```swift
let session = AVCaptureDevice.DiscoverySession(
    deviceTypes: [.builtInWideAngleCamera, .builtInUltraWideCamera],
    mediaType: .video,
    position: .front
)
```

個別のカメラは `.builtInOuterUltraWideCamera` と `.builtInInnerUltraWideCamera` で指定します。全機能を使いたい場合や深度が必要な場合はこちらです。

**まず「Virtual Front Camera で足りるか」を決めてください。**足りるなら切り替えを自分で書く必要がありません。既存アプリは Virtual Front Camera から始め、撮影が主目的のアプリで個別のカメラを使うのが Apple の示す使い分けです。

Virtual Front Camera は仮想デバイスで、`isVirtualDevice` が `true` です。実際に配信している物理カメラは `activePrimaryConstituent` で分かります（セッションが動くまでは `nil`）。

## カメラの向き

縦向きに固定して UI 部品を個別に回す疑似的な自動回転を実装している場合は、rotation coordinator と direction coordinator の2つに置き換えてください。前者が撮影画面をどれだけ回せばよいかを、後者が使っているカメラがどちらを向いているかを教えます。前面カメラが2つあるため、前面カメラが利用者の方を向いているという前提は成り立ちません。自前で位置関係を計算しないでください。

固定の `position`（前面・背面・不定）だけでは、カメラが利用者の方を向いているか判断できません。端末を開いたり裏返したりすると関係が変わるためです。たとえば開いて裏返すと、背面カメラと外側ディスプレイのカメラがどちらも利用者側を向きます（Apple の記事「Choosing a camera by the direction it faces」に、閉じた状態・開いた状態・開いて裏返した状態の図があります）。

`AVCaptureDeviceDirectionCoordinator`（AVKit）が、**アプリのビューから見た**カメラの向きを報告します。

```swift
// init(view:deviceTypes:changeHandler:)（iOS 27.1+ Beta）。main actor で生成し、強参照で保持する
directionCoordinator = AVCaptureDeviceDirectionCoordinator(
    view: previewView,
    deviceTypes: [.builtInOuterUltraWideCamera, .builtInInnerUltraWideCamera, .builtInDualWideCamera]
) { [weak self] map in
    self?.cameraDirectionsDidChange(map)
}
```

生成にはビュー（向きの基準。描画先ではない）、監視するデバイスタイプ、変更ハンドラーを渡します。

`deviceTypes` の注意:

- **背面カメラも含める。** iPhone Duo では背面カメラが利用者側を向くことがある
- **Virtual Front Camera は報告されない。** 代わりに `.builtInOuterUltraWideCamera` と `.builtInInnerUltraWideCamera` を指定する
- 外部カメラ、連係カメラ、デスクビューのカメラは指定しても無視される

たとえば端末を開いてビューが内側ディスプレイへ移ると、外側の前面カメラと背面カメラはどちらも backward-facing、内側の前面カメラが forward-facing として報告されます。

両ディスプレイを同時に使う場合は**ビューごとに coordinator を作ります**。同じ背面カメラでも、外側ディスプレイ側からは forward-facing、内側ディスプレイ側からは backward-facing になります。それぞれのビューを基準に報告されるためです。両ディスプレイを同時に使う仕組みそのものは `references/scenes.md` の「カメラアクセサリの登録」を参照してください。

### 変更ハンドラーの書き方

ハンドラーは生成直後に一度（初期状態）、以降は変化のたびに main actor で呼ばれます。最初の呼び出しまで `deviceDirections` は空です。

渡される `AVCaptureDeviceDirectionMap`（iOS 27.1+ Beta）は `forwardFacingDeviceDescriptors` と `backwardFacingDeviceDescriptors` を持ちます。**forward-facing は「ビューと同じ向き」であって、`position == .front` ではありません。**向きを `position` やデバイスタイプから推定しないでください。画面が1つの iPhone でも同じコードで動きます（前面は forward、背面は backward、不定はどちらにも入らず、ハンドラーは1回だけ呼ばれる）。

使用中のカメラが forward-facing に残っているなら何もせず、なくなったときだけ代わりを選びます。選んだ descriptor はカメラ用 actor に渡し、**切り替えに成功してから**使用中のカメラとして記録してください。先に記録すると、デバイスを作れなかったときに実際のカメラと記録が食い違い、次の通知で切り替えが省略されます。

coordinator はビューに紐づくため main actor に隔離されます。ハンドラーに渡されるのは `AVCaptureDevice` ではなく `AVCaptureDeviceDescriptor` です。これは main actor で安全に扱える sendable な表現で、`AVCaptureDevice` を生成するのに必要な情報を含みます。

**ハンドラーの中で AVFoundation の API を直接呼ばないでください。**descriptor をカメラ用 actor に渡し、そこで操作します。

向きが変わったときにやることは3つですが、**実行する場所が違います**。

| やること | どこで |
|---|---|
| descriptor を受け取ってカメラ用 actor へ渡す | 変更ハンドラー |
| `AVCaptureSession` を再構成し、利用者側を向くカメラからの配信を維持する | カメラ用 actor |
| プレビューのミラーリングと UI の更新 | main actor |

### カメラ用 actor での再構成

- `AVCaptureDeviceDescriptor`（iOS 27.1+ Beta）は `deviceType` / `mediaTypes` / `position` / `uniqueID` / `localizedName` を持つ `Sendable` な値。デバイスの確保はしない
- actor 側で `AVCaptureDevice(uniqueID:)` から作る。**dispatch の間にカメラ構成が変わるので `nil` を処理する（強制アンラップしない）**
- マルチカメラセッションで2台つなぐより、1つのビデオ入力を差し替える
- `beginConfiguration()` / `commitConfiguration()` の中で入力を入れ替え、`canAddInput` が通らなければ元の入力に戻す

### プレビューとミラーリング

- 切り替え中は古いカメラの映像が出るので、ハンドラーが呼ばれたらプレビューを隠し、新しいカメラの映像が届いたら戻す
- ミラーリングは `position` ではなく map から決める。接続は `position == .front` を自動でミラーリングするため、**背面カメラが forward なら自分でミラーリングし、前面カメラが backward ならミラーリングを外す**
- `isVideoMirrored` の前に `automaticallyAdjustsVideoMirroring = false` にする（有効なまま設定すると例外）。`isVideoMirroringSupported` が `false` の接続に設定しても例外
- 入力を差し替えるとプレビューの接続が作り直され設定が消える。map 受信時と新デバイス接続後の両方で設定し直す
- 手動で設定したあと、開閉で向きと `position` が再び一致することがある。そのときは `automaticallyAdjustsVideoMirroring` を `true` に戻すか、map から求めた値を毎回設定し、古い設定を残さない

実装例は Apple の記事「Choosing a camera by the direction it faces」（https://developer.apple.com/documentation/avkit/choosing-a-camera-by-the-direction-it-faces）にあります。

## プレビュー

内側ディスプレイで背面カメラの全画角を出すと余白ができます。プレビューを寄せて残りに操作部品をまとめる構成と、画面全体を埋める構成のどちらも選べます。

```swift
previewLayer.videoGravity = .resizeAspectFill
```

超広角の前面カメラでは正方形センサーを活かして横長の比率を選べます。

```swift
let current = device.dynamicAspectRatio   // 読み取り専用

// 変更は専用メソッド。lockForConfiguration が必要で、
// activeFormat の supportedDynamicAspectRatios にある値だけ指定できる
try device.lockForConfiguration()
device.setDynamicAspectRatio(ratio) { syncTime, error in }
device.unlockForConfiguration()
```

## 回転

`AVCaptureDevice.RotationCoordinator`（iOS 17.0+）を採用すると、プレビューと写真が常に正立します。iPhone Duo ではディスプレイ間を移動したときにも更新されます。

```swift
let coordinator = AVCaptureDevice.RotationCoordinator(device: device, previewLayer: previewLayer)
previewLayer.connection?.videoRotationAngle = coordinator.videoRotationAngleForHorizonLevelPreview
```

角度のプロパティは key-value observing に対応しているため、変化を監視して反映します。

Apple の AVFoundation サンプルコード「Supporting device rotation in your camera app」（iOS 27.0+、Xcode 27.0+、https://developer.apple.com/documentation/avfoundation/supporting-device-rotation-in-your-camera-app）が、AVCam を題材にプレビューと撮影結果の両方へ角度を適用する方法を扱っています。アプリ全体の構成は「AVCam: Building a camera app」（https://developer.apple.com/documentation/avfoundation/avcam-building-a-camera-app）にあります。

実装上の注意:

- **coordinator はデバイスに固定。カメラを切り替えるたびに作り直す**
- `previewLayer` に `nil` を渡して作った coordinator は、あとでレイヤーができてもプレビュー角度を報告しない。レイヤーが揃ったら作り直す
- レイヤーはウインドウに入るまで位置を測れない。`didMoveToWindow()` のあとにも渡し直す
- KVO は以後の変化しか通知しない。**監視を始める前に現在の値を読んで適用する**（しないと最初の回転までプレビューが横倒し）
- 撮影結果には `videoRotationAngleForHorizonLevelCapture` を各出力の接続の `videoRotationAngle` に設定する。プレビュー用の値とは異なることがある
- 出力を追加すると既定の角度で接続されるので、追加のたびに現在の角度を適用し直す

```swift
photoOutput.connection(with: .video)?.videoRotationAngle = coordinator.videoRotationAngleForHorizonLevelCapture
```

採用後は、性能のためセンサーの向き補正を無効にします（`isCameraSensorOrientationCompensationEnabled`、iOS 26.0+。対応は `isCameraSensorOrientationCompensationSupported` で確認）。この補正は iPhone Duo の全前面カメラで有効になっています。

```swift
photoOutput.isCameraSensorOrientationCompensationEnabled = false
```

## ビデオ通話アプリ

内側の前面カメラは端末の右寄りという独特な位置にあります。利用者の視線がそこへ向くよう、**UI の重心をカメラのある側へ寄せてください**（Apple の FaceTime はそうしています）。

自分の映像を映す小さなビューは、内側カメラの領域に重ならないようにします。内側カメラの occlusion はカメラが作動している間だけ現れ、切ると消えるので、移動させる側も出し入れに追従する必要があります。reserved regions の `.occlusion` で領域を取ります（`references/layout.md` 参照）。
