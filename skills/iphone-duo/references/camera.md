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

**まず「Virtual Front Camera で足りるか」を決めてください。**足りるなら切り替えを自分で書く必要がありません。

## カメラの向き

固定の `position`（前面・背面・不定）だけでは、カメラが利用者の方を向いているか判断できません。端末を開いたり裏返したりすると関係が変わるためです。

`AVCaptureDeviceDirectionCoordinator`（AVKit）が、**アプリのビューから見た**カメラの向きを報告します。

```swift
directionCoordinator = AVCaptureDeviceDirectionCoordinator(
    view: view,
    deviceTypes: [.builtInOuterUltraWideCamera, .builtInInnerUltraWideCamera, .builtInDualWideCamera],
    changeHandler: { [weak self] map in
        self?.updateCameraSession(map)
    }
)
```

生成にはビュー、監視するデバイスタイプ、変更ハンドラーの3つが要ります。たとえば端末を開いてビューが内側ディスプレイへ移ると、外側の前面カメラと背面カメラはどちらも backward-facing、内側の前面カメラが forward-facing として報告されます。

両ディスプレイを同時に使う場合は**ビューごとに coordinator を作ります**。同じ背面カメラでも、外側ディスプレイ側からは forward-facing、内側ディスプレイ側からは backward-facing になります。それぞれのビューを基準に報告されるためです。

### 変更ハンドラーの書き方

coordinator はビューに紐づくため main actor に隔離されます。ハンドラーに渡されるのは `AVCaptureDevice` ではなく `AVCaptureDeviceDescriptor` です。これは main actor で安全に扱える sendable な表現で、`AVCaptureDevice` を生成するのに必要な情報を含みます。

**ハンドラーの中で AVFoundation の API を直接呼ばないでください。**descriptor をカメラ用 actor に渡し、そこで操作します。

向きが変わったときにやることは3つですが、**実行する場所が違います**。

| やること | どこで |
|---|---|
| descriptor を受け取ってカメラ用 actor へ渡す | 変更ハンドラー |
| `AVCaptureSession` を再構成し、利用者側を向くカメラからの配信を維持する | カメラ用 actor |
| プレビューのミラーリングと UI の更新 | main actor |

背面カメラが利用者側を向く場合はミラーリングすると自然なセルフィー体験になります。

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

採用後は、性能のためセンサーの向き補正を無効にします。この補正は iPhone Duo の全前面カメラで有効になっています。

```swift
photoOutput.isCameraSensorOrientationCompensationEnabled = false
```
