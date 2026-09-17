# 垂直バー

## 何が起きるか

外側ディスプレイと、開いた状態の横向きでは、通常は上下にあるバーが**側面に移ります**。縦方向をコンテンツのために空け、コントロールを親指の届く位置に置くためです。開いた状態の縦向きは例外で、縦方向の空間が十分にあるため標準の水平バーを保ちます。

側面に移るのはアプリの要素だけではありません。Dynamic Island、ステータスバー、ツールバー（ナビゲーションボタンを含む）、タブバーが対象です。Split View では各アプリが自分の外側の端に配置します。バーはハードウェアに対して位置を保つため、右から左へ記述する言語でも同じ側に留まります。

新しい部品ではなく、**同じコンポーネントがレイアウトを変えているだけ**です。水平バーを90度回転させて縦のスタックにしたものを想像すると理解しやすくなります。

## 対応の前提

1. 最新 SDK で再ビルドする
2. ナビゲーションコンテナが提供するバーを使う

`UIToolbar` / `UINavigationBar` / `UITabBar` で独自にバーを組んでいる場合、その内容は新しい挙動の対象になりません。SwiftUI なら `NavigationStack` や `NavigationSplitView` に `toolbar` を組み合わせ、UIKit なら `UINavigationController` と `UITabBarController` に任せます。

### 独自のタブバー

`UITabBar` を生成してサブビューとして追加しているだけなら、垂直バーには移りません。`UITabBarController` か SwiftUI の `TabView` に置き換えれば追加作業なしで対応します。まず置き換えを提案してください。

完全に自作したタブバーを残す場合は、reserved regions などの低レベルな API で垂直バーの寸法に合わせる必要があり、作業量が大きくなります。次の点もすべて自分で扱うことになります。

- 垂直バーの位置は一定ではない。閉じた状態で回転すると常にカメラのある側に付くため左側にも来る。開いた状態の Split View では左側のアプリは左端に付く
- システムのタブバーは、押し下げてドラッグを始めた時点で各タブのラベルを表示する

## 配置の順序

垂直バーでは上から順に、主要ナビゲーション（戻る・閉じる）、主要アクション（完了など）を置きます。残りは元のグループを維持します。

```swift
// 主要アクション
// SwiftUI
ToolbarItem(placement: .topBarPinnedTrailing) { ... }
// UIKit
navigationItem.pinnedTrailingGroup = ...

// 独自の戻る・閉じる
// SwiftUI
ToolbarItem(placement: .cancellationAction) { ... }
// UIKit
navigationItem.leftItemsSupplementBackButton = false
```

ナビゲーションコントローラを使っていれば戻るボタンは自動で追加されます。姿勢によって垂直にならない場合もあるので、**配置の相対関係は姿勢をまたいで一貫させてください**。操作の位置を学び直させないためです。

## 縦に置ける項目、置けない項目

水平バーは項目の高さが固定で幅が可変、垂直バーは幅が固定で高さが可変です。このため垂直バーはシンボルだけの項目に向いています。

表示形式の選び方自体は水平バーと同じです。上下のバーではアイコンが優先され、なければテキストが出ます。オーバーフローではタイトルとアイコンの両方が出ます。垂直バーで加わったのは「その内容が垂直と水平のどちらに向くか」という判断だけです。アイコンを持つ項目は垂直へ、テキストだけの項目は水平に残ります。

**タイトルとアイコンの両方を渡してシステムに選ばせるのが基本です。**画像で表示する項目にもタイトルが要ります。オーバーフローや展開表示で使われるためです。

```swift
// SwiftUI
Label("共有", systemImage: "square.and.arrow.up")
// UIKit は UIBarButtonItem の title と image を両方設定する
```

既定の軸を変えたい場合は `axisBehavior` を使います。

```swift
// シンボルとテキストが切り替わる独自項目は水平に固定する
.axisBehavior(.horizontalOnly)      // SwiftUI
item.axisBehavior = .horizontalOnly // UIKit

// 垂直表示に対応したカスタムビューは垂直を優先させる
.axisBehavior(.verticalPreferred)
```

システムの編集ボタンは自動で水平に残ります。UIKit のカスタムビューや複雑なビューも既定では水平です。

### 縦に置ける内容を増やす

テキストが必要かどうかは、**そのテキストがシンボルを補強しているだけか、それ自体で独立した情報を持つか**で判断します。補助的なだけならシンボルだけで足ります。件数のような情報はバッジに逃がせます。

```swift
.badge(unreadCount)          // SwiftUI
item.badge = .count(7)       // UIKit（iOS 26.0+）
```

金額を表示するカートボタンのように、テキスト自体が意味を持つ場合は水平バーに残します。

垂直バーがあるかどうかは、SwiftUI が `@Environment(\.toolbarVerticalEdge)`、UIKit が `traitCollection.verticalBarEdge` で読めます。カスタムビュー側で表示を変えたいときに使います。カスタムビューはバーの固定幅に収まるか、垂直用のレイアウトを持たせてください。

## オーバーフロー

外側ディスプレイの横向きや、キーボードなどが出た場面では項目がオーバーフローに移ります。

**最初の判断はツールバーとタブバーのどちらを長く残すか**です。ナビゲーション中心の画面なら既定どおりツールバーを先に圧縮し、作業中心の画面ならタブバーを先に圧縮してアクションを残します。

```swift
// SwiftUI
.toolbarVerticalCompressionBehavior(.prefersToolbarItems)
// UIKit
navigationItem.verticalBarCompressionBehavior = .prefersBarItems
// UIVerticalBarCompressionBehavior は .automatic / .prefersBarItems / .prefersTabBar
```

項目は既定で下から上の順にオーバーフローします。順序は可視性優先度で調整します。まずグループ単位で決め、必要に応じて個別項目にも設定します。頻繁に使う操作や重要な状態を示す項目は長く残します。

```swift
.visibilityPriority(.high)        // SwiftUI（ToolbarItemVisibilityPriority）
item.visibilityPriority = .high   // UIKit（UIBarButtonItemVisibilityPriority）
// カスタム値は init(higherThan:) / init(lowerThan:)
```

独自のオーバーフローがある場合は、システム管理のメニューに統合します。SwiftUI は `ToolbarOverflowMenu`、UIKit は `UINavigationItem.additionalOverflowItems` です。他プラットフォームの記号を持ち込まず、省略記号はオーバーフロー専用にしてください。

## 垂直バーを無効にする

すべてのアプリに向くわけではありません。電卓のように下部に内容が集中する単一ページのアプリや、閉じるボタンしかないシートでは、無効にしたほうが領域を活かせます。

```swift
// SwiftUI
NavigationStack {
    ContentView()
        .toolbarVerticalBehavior(.disabled)
}

// UIKit
override var preferredVerticalBarBehavior: UIVerticalBarBehavior { .disabled }
```

外側ディスプレイのシートで無効にすると、前面カメラの手前までを使う表示になり、ステータスバーも再配置されます。

AR アプリのような没入型の全画面アプリは、ステータスバーを隠せば実現できます。内容に合わなければ垂直のコントロールを使う必要はありません。

## その他

- 分割ビューでは detail 列だけが垂直バーの対象。展開したインスペクタには独立した垂直バーを付けない（detail 列が既に持つため）
- キーボードのアクセサリバーは縦軸へ移さず、キーボードに付随させたままにする
- バーの向きにかかわらず、アプリ側で追加の間隔を作らない
- 垂直バーは水平バーと同様に既定でスクロール端の効果を持たない。透明度を下げるアクセシビリティ設定では、垂直バーの背後に不透明な矩形が描かれる。この矩形は垂直バーの safe area より少し狭いので、設定を有効にして表示を確認する
- 可変スペーサーは縦軸では既定でサイズがゼロ、固定スペーサーは最小サイズを保つ
