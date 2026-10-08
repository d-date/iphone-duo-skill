# 移行チェックリスト

既存アプリを iPhone Duo に対応させる手順です。上から順に進めると手戻りが少なくなります。

## 1. ビルドと検証環境

- [ ] iOS 27.1 SDK でビルドする
- [ ] Xcode 27.1 の Device Hub で iPhone Duo シミュレータを選ぶ
- [ ] ヒンジの角度と姿勢（閉じた状態・開いた状態・book・laptop・tent）の各状態を確認する
- [ ] 姿勢から姿勢へ移る途中の表示も確認する
- [ ] 折り目やカメラを避ける自前のレイアウトは、シミュレータで折り曲げて reserved regions を描き、重なっていないか確認する（`references/layout.md` の「シミュレータでの検証」参照）
- [ ] Xcode のコーディングアシスタントに「get my app ready for iPhone Duo」と頼み、App Resizability スキルを試す。ほかのエージェントでは `xcrun agent skills export` で書き出す（すべての問題を検出できるわけではない）
- [ ] カメラを使うアプリは実機でプレビューを確認する
- [ ] Device Hub の iOS resizable simulator、macOS 27 の iPhone ミラーリング、iPad のウインドウのリサイズでも表示を確認する。ミラーリングでは両方向に極端なサイズまでリサイズする。ネイティブの体験全体は iPhone Duo シミュレータで確認する
- [ ] iOS 27.1 SDK でビルドしたアプリを出す前に、バーが縦になってもコンテンツが適応することを確認する

## 2. レイアウト判定の見直し

ここが最も壊れやすい箇所です。

- [ ] レイアウトの大小判定を size class に置き換える
- [ ] user interface idiom から画面サイズやデバイス機能を推定している箇所を除去する
- [ ] `UIDevice.current.orientation`・`statusBarOrientation`・`interfaceOrientation`（`windowScene.effectiveGeometry.interfaceOrientation` を含む）に依存するレイアウトを除去し、縦長か横長かは bounds の幅と高さで判断する
- [ ] 非推奨の `UIScreen.main` への参照を除去し、レイアウト判断には environment・trait collection・scene の bounds を使う。画面が必要なら `UIScreen` を保持せず window scene から動的に取得する
- [ ] `UIWindow(frame: UIScreen.main.bounds)` を `UIWindow(windowScene:)` に置き換える
- [ ] サイズを起動時や `viewIsAppearing` で一度読むだけにせず、`layoutSubviews` / `viewDidLayoutSubviews` で読み直す。サイズ変更時の処理は `viewWillTransition(to:with:)` に置く
- [ ] 固定幅、ブレークポイント、`bounds.height == 844` のような特定の画面に結び付いた寸法・比較を除去する
- [ ] `UIRequiresFullScreen` など、向きやサイズを固定する設定から離れる
- [ ] size class で分岐して別々のコンテナを使っている箇所を洗い出し、状態を上位に持ち上げる

## 3. safe area

- [ ] 向かい合う辺のインセットが等しいという前提を除去し、各辺を個別に処理する（コンテンツのインセットも同様）
- [ ] 操作可能な要素と前景コンテンツを safe area の内側に配置する
- [ ] 背景を safe area の外側やバーの背後まで広げる
- [ ] Split View でアプリを左右両側に配置して検証する（垂直バーがどちら側にも来る）
- [ ] 縦向き固定を外し、横向きで leading・trailing の safe area を確認する
- [ ] 幅から片側のインセットを2倍して引いている箇所を探す

## 4. ナビゲーションとバー

- [ ] 標準のナビゲーションコンテナが提供するバーを使う（独自構成のバーは対象外）
- [ ] サブビューとして追加した `UITabBar` を `UITabBarController` / `TabView` に置き換える
- [ ] ツールバー構成を主要ナビゲーション、主要アクションの順に見直す
- [ ] 画像で表示する項目にもタイトルを提供する
- [ ] タイトルだけの項目や、テキストと画像を併記するカスタムビューを減らす（ただし金額のようにテキスト自体が意味を持つものは水平バーに残す。`references/bars.md` 参照）
- [ ] 件数表示をバッジに置き換える
- [ ] 独自のオーバーフローを単一のシステム管理メニューへ統合する
- [ ] グループ単位で可視性優先度を決める
- [ ] キーボードのアクセサリバーをキーボードに付随させたままにする
- [ ] 「透明度を下げる」を有効にした状態で垂直バーまわりの表示を確認する
- [ ] タブ項目にアイコンを設定する
- [ ] 回転・ピクチャ・イン・ピクチャ・Split View で垂直バーの高さを変え、圧縮とオーバーフローの見え方を確認する
- [ ] 背景画像を持つビューは `backgroundExtensionEffect()` / `UIBackgroundExtensionView` で垂直バーの背後まで広げる
- [ ] 地図のように背後を広く見せたいシートは `presentationPlacement(_:)` / `preferredPlacement` で配置を指定する
- [ ] 垂直バーが向かない画面（下部に内容が集中する単一ページ、ボタン1つのシート）では無効化を検討する

## 5. レイアウトの適応

- [ ] `.scaleAspectFill` / `.aspectRatio(contentMode: .fill)` の全面メディアが横長で欠けないか確認し、size class やアスペクト比で fill と fit を切り替えるか焦点を指定する
- [ ] 中央配置のレイアウトを点検し、2列化や displacement を検討する
- [ ] 連続スクロールコンテンツを領域間で移動させていないか確認する
- [ ] 複雑なレイアウトや、独自 UI を safe area 外に配置する場合は reserved regions を検討する
- [ ] 折り目回避が必要な箇所で、対応するシステムコンポーネントを使う
- [ ] 折り目に重なる自前のボタン（購入・カートへ追加など）を reserved regions で動かす
- [ ] 平らな状態で division が非アクティブになり、アクティブな領域が0件のときに折り目なしとして振る舞うフォールバックを持たせる
- [ ] 姿勢ごとに UI を変える場合、姿勢の間の遷移を滑らかにアニメーションできるか確認する
- [ ] Dynamic Type の大きなサイズで内側ディスプレイの表示を確認する
- [ ] グリッド状のレイアウトは列数を偶数にする（折り目で分かれたときにきれいに割れるため。`references/layout.md` 参照）

## 6. 複数ディスプレイ

- [ ] Split View multitasking に対応する
- [ ] 新規シーン要求時のエラーを処理し、`UIWindowScene.ActivationAction` を使う
- [ ] 別ディスプレイへ出す付随コンテンツがあれば scene accessories を検討する
- [ ] scene accessory の利用可否の変化に追従する
- [ ] カメラアプリは `CameraCaptureAccessory` を撮影画面のビューに登録し、利用可否に応じて操作を無効にする
- [ ] 複数インスタンスで共有される状態（`@AppStorage` など）の更新が各インスタンスに反映されるか確認する
- [ ] ヒンジを使うインタラクションやエフェクトを検討する（レイアウトには使わない）

## 7. カメラ

- [ ] 疑似的な自動回転（縦向き固定 + UI 部品の個別回転）をやめ、rotation coordinator と direction coordinator に置き換える
- [ ] 前面カメラの方針を決める（Virtual Front Camera か個別のカメラか）
- [ ] 個別のカメラを使う場合、開閉時の切り替えを実装する
- [ ] 個別のカメラを扱う場合は direction coordinator を採用する。両ディスプレイ同時使用ならビューごとに作る
- [ ] 変更ハンドラーから AVFoundation を直接呼ばず、カメラ用 actor 経由にする
- [ ] 向きの変更時にプレビューのミラーリングを検討する
- [ ] rotation coordinator を採用し、その後センサーの向き補正を無効にする
- [ ] ビデオ通話アプリは、自分の映像を映すビューが内側カメラの occlusion を避けるようにする

## 8. App Store

- [ ] iPhone Duo に最適化したアプリを App Store Connect で提出する（2026-10-05 から受付）
- [ ] 2027年4月以降の提出では、必須の iPhone Duo のスクリーンショットを用意する
- [ ] 各向きでの表示が分かるスクリーンショットと app preview を用意する
- [ ] App Store Connect のプレビューツールで、iPhone Duo での見え方を確認する
- [ ] フィーチャーのノミネートを出し、Helpful Details に iPhone Duo への最適化とすべての姿勢への対応を書く
