---
name: iphone-duo
description: Apple の iPhone Duo（複数ディスプレイと折り目を持つ iPhone）にアプリを対応させるための実装ガイド。size class・safe area の非対称性・reserved regions・垂直バー・arrangement・ヒンジ・scene accessories・デュアル前面カメラを扱う。iPhone Duo、折りたたみ iPhone、内側/外側ディスプレイ、垂直ツールバー、ヒンジ角度、reserved region、ArrangementView、CameraCaptureAccessory、AVCaptureDeviceDirectionCoordinator のいずれかに言及があれば必ず使うこと。「iPhone Duo に対応させたい」「折りたたみでレイアウトが崩れる」「バーが縦になる」「開閉でカメラを切り替えたい」といった相談でも、iPhone Duo という語が出ていなくても iOS の複数ディスプレイ・折り目・姿勢変化の話題なら参照すること。
---

# iPhone Duo 対応

Apple の Tech Talks 6本、Human Interface Guidelines、Apple Developer Documentation、iPhone Duo Group Lab（2026-09-16 と 2026-09-17 の2回）を一次情報として整理した実装ガイドです。

## 最初に押さえること

iPhone Duo は 2026-10-23 に iOS 27.1 搭載で発売されます。iPhone Duo 対応の SDK と Device Hub は Xcode 27.1（2026年9月中に提供）に含まれます。

iPhone Duo は内側と外側の2つのディスプレイを持ち、中央のヒンジで開閉します。**アプリは iPhone アプリのまま**で、開閉によって発生するのはサイズ変更です。専用の user interface idiom はなく phone を返します。内側ディスプレイの水平・垂直ともに regular という組み合わせが iPhone では初めて、という点だけが新しい要素です。

内側ディスプレイで全画面表示していたアプリは、閉じると外側ディスプレイへ移り、そのまま動き続けます。バックグラウンドへは回りません。例外は Split View で、2つのアプリが並んでいる場合は直前に操作していた方が外側へ上がり、もう一方がバックグラウンドへ回ります。閉じてすぐ開けば両方戻り、時間が空くと外側にいたアプリが内側全体を占めます。

対応の土台は「リサイズに耐えるレイアウト」です。すでに iPad や iPhone ミラーリングでのリサイズに対応しているなら、その資産がそのまま効きます。

### SDK による段階

| ビルド SDK | 挙動 |
|---|---|
| iOS 27 より前（Xcode 26） | 動作するがレターボックス表示。外側は iPhone mini に近い比率で垂直バー側に黒帯、内側は同じ比率で中央に表示され、姿勢の変化に反応しない |
| iOS 27 | リサイズ対応が有効になる（オプトアウト不可）。内側ディスプレイのほぼ全体を使うが、ステータスバー下の側面に黒帯が残る |
| iOS 27.1 | 画面端まで広がり、標準のバーがステータスバー下に縦配置される |

最初からすべての姿勢を作り込む必要はありません。発売日にはリサイズ対応とベストプラクティスに沿った状態を出し、その後に改善する進め方が Apple から勧められています。

検証は Xcode 27.1 の Device Hub にある iPhone Duo シミュレータで行います。ヒンジの角度を変えられるほか、閉じた状態・開いた状態・book・laptop・tent の姿勢を選べます。姿勢そのものだけでなく、姿勢から姿勢へ移る途中の表示も確認してください。シミュレータで再現できないのは内側と外側のディスプレイを同時に点灯させる状態だけです（カメラが必要なため）。なお iOS 27.1 のシミュレータは、カメラを使うアプリを起動できるようになります。映像は映らず「利用できるカメラがない」と判定されますが、それ以外の UI は確認できます。

Xcode 27.1 がない段階では、iPad で実行してウインドウをリサイズするか、iPhone のみのアプリなら Mac の iPhone ミラーリングでリサイズして確認できます。iPhone ミラーリングは内側ディスプレイに近い比率になり、interface idiom も iPhone のままです。多くの不具合はこれで再現します。垂直バーによる safe area やマージンの非対称に起因する問題は、iPhone Duo シミュレータでないと見つかりにくい点に注意してください。

## どこを読むか

作業内容に応じて必要な参照だけを読んでください。

| 参照 | 扱う内容 |
|---|---|
| `references/layout.md` | size class、safe area の非対称性、画面の角、reserved regions、arrangement |
| `references/bars.md` | 垂直バー、項目の配置順、軸の制御、バッジ、オーバーフロー、シート、無効化 |
| `references/scenes.md` | ヒンジ、マルチタスキング、複数ウインドウ、scene accessories、カメラアクセサリ |
| `references/camera.md` | デュアル前面カメラ、カメラの向き、プレビュー、回転 |
| `references/checklist.md` | 既存アプリの移行チェックリスト |

## 全体に効く原則

これらは領域を問わず効いてくるので、先に頭に入れておくと判断が速くなります。

**size class で判断する。** user interface idiom から画面サイズやデバイス機能を推定したり、interface orientation でレイアウトを分岐したりしないでください。内側ディスプレイはアプリが宣言した supported interface orientations に従って回転せず、代わりにスケーリングされます。向きを見ても意図した結果になりません。縦向き専用のアプリでも、内側ディスプレイでは size class が regular/regular になります。現在の向きは environment や trait から読めますが、判断は利用できる幅で行ってください。`UIRequiresFullScreen` のような向きやサイズを固定する設定からは離れることが勧められています。

**分岐でビュー階層を作り替えない。** size class で `if` を切り、分岐ごとに別のコンテナを使っている SwiftUI のコードは、切り替わった時点で片方のビュー階層が捨てられ、状態を上位に持ち上げていなければ失われます。iPad のリサイズでも起きていた問題ですが、iPhone Duo では内側ディスプレイから閉じた状態へ移るたびに表面化します。分岐そのものをやめられないか（`ArrangementView` などで置き換えられないか）を先に検討してください。

**画面を直接参照しない。** `UIScreen.main` は2画面のデバイスでは曖昧になり、ドキュメント上すでに非推奨です。environment、trait collection、scene の bounds を使い、画面が必要なら `window?.windowScene?.screen` から取ります。画面スケールは `displayScale` / `traitCollection.displayScale` で足ります。

**safe area は非対称になる。** 向かい合う辺のインセットが等しいという前提を置いたコードは壊れます。`view.bounds.width - insets.left * 2` のような書き方をやめ、各辺を個別に扱ってください。layout margins やコンテンツのインセットも同様に非対称です。片側の値を反対側に流用せず、システムが返す実際の値を使ってください。

**標準コンポーネントを使うと多くが自動で解決する。** 標準のナビゲーション、バー、シート、アラート、メニューは各姿勢と折り目に適応します。折り目を避ける動作もシステム側に組み込まれています。独自実装に置き換えるほど、自分で面倒を見る範囲が増えます。

**抽象度の高い API から選ぶ。** 折り目に関する API は「ヒンジ角度 → reserved regions → arrangement view → システムコンポーネント」の層になっています。ヒンジ角度から自前で計算する前に、上位の層で足りないか検討してください。

**これは電話である。** iPad 版があるならそのデザインは良い出発点ですが、そのまま持ち込まないでください。iPad にしか属さない要素は取り除くか idiom で分けます。逆に、iPhone Duo だけで動くアプリや、1つの姿勢だけを想定したアプリは作らないでください。利用者は製品ラインの1機種として扱います。iPad アプリを載せる第一歩は、ユニバーサル化して対応プラットフォームに iPhone を加えることです（Apple Vision Pro のような互換モードの有無については、明確な回答が出ていません）。

**1つの端末として扱う。** 開く操作は、アプリのウインドウを横に広げるリサイズとして設計します。利用者はさまざまな持ち方・置き方をするため、特定の姿勢だけを前提にしないでください。卓上に立てる姿勢はカメラ側を下にしても外側ディスプレイを下にしても同じ体験になるべきです。姿勢ごとに UI を変える場合は、その間の遷移を滑らかにアニメーションできるかを考えます。

## iOS 27.1 の API について

arrangement、reserved regions、ヒンジ（UIKit）、垂直バーの制御、カメラの direction coordinator などは、2026-09-18 時点で **iOS 27.1+ Beta** としてドキュメントの公開を確認しています。ベータ版なので正式リリースまでに変わる可能性があります。

2026-09-18 時点で、このガイドが扱う API はすべてドキュメントで宣言を確認できています。

iOS 16 など古いバージョンへの対応を続ける場合は、iPhone Duo 固有の部分を availability check で囲みます。範囲はアプリ次第で、新しい UI を別に作って古い UI を凍結する進め方も、条件を細かく切って同じ機能を全機種へ出し続ける進め方もあります。

**実装時は Xcode の補完と実機/シミュレータで確認してください。**

## 回答の仕方

説明・提案・検証結果は日本語で書いてください。API 名・型名・識別子・出典のタイトルは原綴りのまま扱います。

未公開 API を扱う性質上、**確認できないシグネチャを推測で補わないでください。**Xcode の補完や実機・シミュレータで確かめられない場合は、その旨を明示したうえで提案してください。誤った綴りをそれらしく書くほうが、分からないと言うより害になります。

## 出典

- Tech Talks: Prepare your app for iPhone Duo / Raise the bar with iPhone Duo / Strike a pose with adaptive layouts on iPhone Duo / Leverage multiple displays and scenes on iPhone Duo / Build a great camera experience for iPhone Duo / Design for iPhone Duo
- Human Interface Guidelines: Designing for iPhone Duo
- Meet with Apple: iPhone Duo Group Lab（2026-09-16 / 2026-09-17）
- Apple Developer Documentation: Preparing your app for iPhone Duo / Registering a camera capture accessory on iPhone Duo / Choosing a camera by the direction it faces / Supporting device rotation in your camera app / iOS 27.1 Beta API リファレンス
