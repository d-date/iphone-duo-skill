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

固定幅、ブレークポイント、特定の画面に結び付いた寸法は避けてください。外側ディスプレイで縦向きと横向きを区別したい場合も、2軸の size class が異なるので判断できます。ただしまず「区別する必要が本当にあるか」を疑ってください。

### 分岐で状態を失わない

size class で `if` を切り、分岐ごとに別のコンテナを使っている SwiftUI のコードは、分岐が切り替わった時点で片方のビュー階層が捨てられます。状態を上位に持ち上げていなければ失われます。

```swift
// 避ける: 分岐ごとに別のコンテナ
if horizontalSizeClass == .regular {
    NavigationSplitView { ... } detail: { ... }
} else {
    NavigationStack { ... }
}
```

内側と外側のディスプレイを行き来してもアプリは作り直されません。破棄と再生成ではなくリサイズなので、標準のナビゲーションコンテナを使っていれば状態はそのまま引き継がれます。分岐そのものをやめられないか（`NavigationSplitView` 単体や `ArrangementView` で吸収できないか）を先に検討してください。

内側ディスプレイで縦横のレイアウトを変えたい場合、現在の向きは environment と trait から取れますが、判断は利用できる幅で行ってください。iPad では横向きのまま幅の狭いウインドウを作れるためです。分割ビューなどのコンテナに列の判断を任せ、グリッドは実際の幅に合わせます（例: 横向きで2列、縦向きで1列）。

### 3列レイアウト

iPad で3列のアプリは、内側ディスプレイでは実質2列を同時に表示します。縦向きでは詳細だけを表示し、左上のボタンでサイドバーをオーバーレイ表示します。SwiftUI は `NavigationSplitView`、UIKit は `UISplitViewController` を使ってください。本のように部分的に折ると 50/50 に自動調整されます。情報量の多いアプリでは、タブをサイドバーとして表示する選択肢もあります。

### シートとタブ

地図の上にシートを重ね、その中にタブバーを置く構成（Find My など）で、広い画面ではタブバーとシートを分けるべきか迷う場合: タブがシートの中身を切り替えているなら一緒にしておきます。タブごとに別のシートを持つ構成なら分ける選択肢があります。

## safe area

非対称になることが前提です。垂直バーが片側に寄るため、左右のインセットが揃いません。Split View では自分のアプリが左右どちらにも来ます。

```swift
// 避ける: 向かい合う辺が等しいという前提
let width = view.bounds.width - view.safeAreaInsets.left * 2

// 各辺を個別に扱う
let width = view.bounds.inset(by: view.safeAreaInsets).width
```

コンテンツのインセットも非対称です。片側の値を使って左右をそろえるのではなく、システムが返す実際の値を使ってください。縦向き固定だったアプリは上下の safe area しか考慮していないことが多く、左右の前提が残りやすい箇所です。

配置の原則は「前景は safe area の内側、背景はその外側まで」です。SwiftUI は既定でコンテンツが safe area 内に入るため、意識するのは背景を広げるときだけです。

```swift
// SwiftUI: 背景だけ広げる
Color.accentColor.ignoresSafeArea()

// UIKit
foreground.frame = view.bounds.inset(by: view.safeAreaInsets)
backgroundView.frame = view.bounds
```

縦向き固定のアプリは固定を外し、横向きで leading と trailing の safe area を確認してください。`safeAreaInsets.left * 2` のように片側を2倍している箇所が典型的な壊れどころです。サイドバーを持つ iPad アプリは leading 側をすでに扱っているぶん有利です。

インセットの取得は SwiftUI が `GeometryProxy.safeAreaInsets`、UIKit が `UIView.safeAreaInsets` です。UIKit には領域ごとのガイドを返す `UIView.LayoutRegion`（iOS 26.0+）もあり、`layoutGuide(for:)` / `edgeInsets(for:)` で safe area・margins・readable content を角への追従つきで取得できます。

バーを持たないアプリで、safe area 全体ではなくカメラとステータスバーのある領域だけを避けたい場合は、`UIView.LayoutRegion` の corner adaptation でその領域だけを尊重し、残りは端まで描けます（全画面のゲームなど）。

## 画面の角

iOS 26 の Concentricity API が iPhone Duo の画面形状に対応するよう更新されています。

外側ディスプレイの4隅は半径がそろっていません（ヒンジから遠い側のほうが丸い）。またこれまで角を扱う必要がなかった位置に角が現れます。iPad 向けに角の対応を済ませていても、扱う場面が増えていないか確認してください。

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

SwiftUI の宣言は `reservedRegions(kind:options:layoutDirectionBehavior:)` で、`options` の既定値は空、`layoutDirectionBehavior` の既定値は `.mirrors` です。`ReservedRegion` は `frame` のほかに `isActive`、`kind`、`margins` を持ちます（iOS 27.1+ Beta）。

グリッド状のレイアウトでは列数を偶数にしておくと、折り目で分かれたときにきれいに割れます。折り目の状態によらず偶数を保ちたい場合に `.includeInactive` が効きます。

**Duo 固有の寸法をハードコードしないでください。**アラートのようなシステムコンポーネントは、本のように折ったときも卓上に立てたときも読みやすく押しやすい位置へ自動で動きます。動かないのはコンテンツ領域に置いた自前の部品です。購入・カートへ追加・無料トライアル開始のような収益に直結するボタンが折り目に重なる構成なら、reserved regions で折り目がアクティブかどうかを見て置き場所を選び直してください。単独の要素を動かしたい場合がこの API の出番です。

### シミュレータでの検証

Xcode 27.1 beta の iPhone Duo シミュレータでは、`reservedRegions` は閉じた状態・平らに開いた状態・折り曲げた状態のいずれでも、`.division` と `.occlusion` の両方で0件を返します。`.includeInactive` を付けても0件です。ヒンジの状態はシミュレートされていますが、システムがアプリのシーンに領域を1つも渡していません（SwiftUI/UIKit からの問い合わせ方の問題ではありません）。正式版の Xcode 27.1 やシミュレータの更新で変わる可能性はありますが、この beta ではシミュレータで折り目回避のコードを動かして確かめることはできません。次の形で組んでください。

- **折り目の位置を注入できるようにする。** レイアウトは `reservedRegions` を直接読まず、折り目の範囲（`[ReservedRegion]` から取った `frame` など）を environment 値や引数で受け取る。プレビューとユニットテストでは任意の位置・幅の折り目を与え、折り目あり・なしの両方のレイアウトを確かめる
- **`reservedRegions` を読むアダプタは薄く分ける。** `GeometryReader` などで `reservedRegions(kind:)` を読み、注入用の値へ変換するだけの層にする。この層は実機で確認する
- **0件なら折り目なしとして振る舞う。** 領域が返らない場合のフォールバックを必ず持つ。この beta のシミュレータでは常に0件になるため、0件を想定していないレイアウトはシミュレータ上で崩れる

折り目の位置を environment 値で注入してレイアウトとテストを組み、`reservedRegions` を読むアダプタを後から分けて足す進め方は、公開されている SwiftUICalendar の対応（maniramezan/SwiftUICalendar PR #21）と同じです。

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

arrangement view を使うと、姿勢が変わったときに2つのビューが滑るように分かれるアニメーションが得られます（TV アプリで部分的に折ると動画と操作部が分かれる動き）。姿勢ごとに独自に UI を切り替えるより、遷移が自然になります。

**iPhone Duo 専用ではありません。**折りたたまない端末でも、指定した条件が満たされていれば同じように働きます。2列を並べられる幅があれば2列、なければ単一のビューになり、これは内側と外側のディスプレイを行き来するときの挙動と同じです。`HStack` / `VStack` / `ZStack` を `if` で切り替える書き方から離れる手段として使えます。

## 姿勢ごとの作り込み

**姿勢を判定して分岐する前に、arrangement view と reserved regions で足りないか検討してください。**laptop か book かを直接見に行くのではなく、ビュー同士の関係を宣言して配置をシステムに任せるほうが素直です（Apple の音楽アプリは arrangement view を使い、分割の位置を知るために reserved region を参照しています）。

部分的に折った状態が主要な使い方になるのか、姿勢を変える途中の一瞬にすぎないのかは Apple 内でも結論が出ていません。発売直後から作り込みすぎないでください。

ベストプラクティスに従っていれば、各姿勢で問題なく表示されます。すべての姿勢に専用の体験を用意しようとするのは、Apple 自身が失敗例として挙げている進め方です。アプリの用途に合う姿勢（例: 卓上に置いたときに操作部を下側へ移す動画・ポッドキャストプレーヤー）があれば、そこだけ作り込んでください。

## アクセシビリティ

- 内側ディスプレイは Dynamic Type の大きなサイズで効果が大きい。readable content guide などを使い、文字サイズの変更とリサイズの両方に耐えるようにする
- VoiceOver は2つのディスプレイで同時に動作する
- 「透明度を下げる」を有効にした状態で垂直バーまわりを確認する（`references/bars.md` 参照）
