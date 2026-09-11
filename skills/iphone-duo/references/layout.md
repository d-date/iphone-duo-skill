# レイアウト

## size class

| 状態 | 水平 | 垂直 |
|---|---|---|
| 外側ディスプレイ・縦向き | compact | regular |
| 外側ディスプレイ・横向き | compact | compact |
| 内側ディスプレイ | regular | regular |

設計上は「外側 = compact width、内側 = regular width」の2つを対象にすれば、すべての姿勢の土台になります。姿勢ごとに個別のレイアウトを作る必要はありません。

```swift
// SwiftUI
@Environment(\.horizontalSizeClass) private var horizontalSizeClass

// UIKit
traitCollection.horizontalSizeClass
// 変化への追従は registerForTraitChanges(_:action:)（iOS 17.0+）
```

固定幅、ブレークポイント、特定の画面に結び付いた寸法は避けてください。

## safe area

非対称になることが前提です。垂直バーが片側に寄るため、左右のインセットが揃いません。Split View では自分のアプリが左右どちらにも来ます。

```swift
// 避ける: 向かい合う辺が等しいという前提
let width = view.bounds.width - view.safeAreaInsets.left * 2

// 各辺を個別に扱う
let width = view.bounds.inset(by: view.safeAreaInsets).width
```

配置の原則は「前景は safe area の内側、背景はその外側まで」です。SwiftUI は既定でコンテンツが safe area 内に入るため、意識するのは背景を広げるときだけです。

```swift
// SwiftUI: 背景だけ広げる
Color.accentColor.ignoresSafeArea()

// UIKit
foreground.frame = view.bounds.inset(by: view.safeAreaInsets)
backgroundView.frame = view.bounds
```

インセットの取得は SwiftUI が `GeometryProxy.safeAreaInsets`、UIKit が `UIView.safeAreaInsets` です。UIKit には領域ごとのガイドを返す `UIView.LayoutRegion`（iOS 26.0+）もあり、`layoutGuide(for:)` / `edgeInsets(for:)` で safe area・margins・readable content を角への追従つきで取得できます。

## 画面の角

iOS 26 の Concentricity API が iPhone Duo の画面形状に対応するよう更新されています。

```swift
// SwiftUI
ConcentricRectangle()

// UIKit
view.cornerConfiguration = .uniformCorners(radius: .containerConcentric(minimum: 0))
```

## reserved regions

ヒンジと内外のカメラが占める領域です。3種類あります。

| 領域 | 存在する条件 |
|---|---|
| 外側の前面カメラ | 常に存在。Live Activities では Dynamic Island へ広がる |
| 内側の前面カメラ | カメラが作動しているときだけ |
| 折り目 | 部分的に開いているとき。内側ディスプレイを複数の領域に分割する |

iPadOS のウインドウコントロールと同じように、レイアウトが以前から適応してきた領域として扱えば十分です。標準コンポーネントは自動で避けます。独自コンポーネントで避ける必要がある場合に API を使います。

```swift
// SwiftUI
GeometryReader { proxy in
    let regions = proxy.reservedRegions(kind: .division)   // 折り目
    let frames = regions.map(\.frame)
}

// UIKit
let regions = view.reservedRegions(kind: .division)

// 非アクティブな領域も含める
proxy.reservedRegions(kind: .division, options: .includeInactive)

// カメラは .occlusion
proxy.reservedRegions(kind: .occlusion)
```

グリッド状のレイアウトでは列数を偶数にしておくと、折り目で分かれたときにきれいに割れます。折り目の状態によらず偶数を保ちたい場合に `.includeInactive` が効きます。

## displacement（要素の移動）

利用可能な空間に合わせて既存要素のフレームを調整する**設計パターン**です。`displacement` という API はありません。自分で動かす場合は reserved regions で領域を問い合わせ、結果を自分のレイアウトに反映します。

指針:

- 独立して適応できる要素は単独で、連携する要素は関係を保って一緒に移動する
- 移動元との視覚的な関係を弱める過度な移動は避ける
- 記事・フィード・文書・リストなどの連続スクロールコンテンツは移動させない（スクロールで適応済みのため、移動は連続性を妨げる）
- 移動先は要素の目的と端末の使用状態に合わせる

## arrangement（2ビューの配置）

primary と secondary の2つのビューを持つレイアウトコンテナです。ディスプレイのサイズ、向き、reserved regions に応じて配置を決めます。iOS 27.1 で利用可能になります。

```swift
// SwiftUI
NavigationStack {
    ArrangementView {
        PlayerView()
    } secondary: {
        UpNextView()
    }
    .arrangementViewStyle(.split)
}

// UIKit
let vc = UIArrangementViewController()
vc.setViewController(playerVC, for: .primary)
vc.setViewController(upNextVC, for: .secondary)
```

2つのスタイルがあります。

- **split**: 横長なら水平、縦長なら垂直に分割。`.split.axes(.horizontal)` で軸を制限でき、その軸が主軸に対応しない場合は単一ビューになる
- **overlay**: 通常は重ね合わせ、折ると横並びを優先。重なり順は SwiftUI が `@Environment(\.overlayArrangementZIndex)`、UIKit が `state(for:)` の `zIndex` で取れる

移行の目安は、`HStack` / `VStack` による分割は split、`ZStack` による重ね合わせは overlay です。既存パターンに当てはまらない場合、前景と背景の関係が明確なら overlay、主内容と詳細で双方を隠したくないなら split を検討します。

**入れ子の制約は向きが逆なので注意してください。**

- `ArrangementView` の**中に** `NavigationSplitView` などのナビゲーションコンテナを置かない
- `List` や `ScrollView` の**中に** `ArrangementView` を置かない

ナビゲーションは `ArrangementView` の外側に置きます。
