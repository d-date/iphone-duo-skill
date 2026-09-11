---
name: iphone-duo
description: Apple の iPhone Duo（複数ディスプレイと折り目を持つ iPhone）にアプリを対応させるための実装ガイド。size class・safe area の非対称性・reserved regions・垂直バー・arrangement・ヒンジ・scene accessories・デュアル前面カメラを扱う。iPhone Duo、折りたたみ iPhone、内側/外側ディスプレイ、垂直ツールバー、ヒンジ角度、reserved region、ArrangementView、CameraCaptureAccessory、AVCaptureDeviceDirectionCoordinator のいずれかに言及があれば必ず使うこと。「iPhone Duo に対応させたい」「折りたたみでレイアウトが崩れる」「バーが縦になる」「開閉でカメラを切り替えたい」といった相談でも、iPhone Duo という語が出ていなくても iOS の複数ディスプレイ・折り目・姿勢変化の話題なら参照すること。
---

# iPhone Duo 対応

Apple の Tech Talks 6本と Human Interface Guidelines を一次情報として整理した実装ガイドです。

## 最初に押さえること

iPhone Duo は内側と外側の2つのディスプレイを持ち、中央のヒンジで開閉します。**アプリは iPhone アプリのまま**で、開閉によって発生するのはサイズ変更です。iPad のような別イディオムにはなりません。

対応の土台は「リサイズに耐えるレイアウト」です。すでに iPad や iPhone ミラーリングでのリサイズに対応しているなら、その資産がそのまま効きます。

### SDK による段階

| ビルド SDK | 挙動 |
|---|---|
| iOS 27 より前 | 動作する。閉じた状態でステータスバーとカメラの左側の領域を使う |
| iOS 27 | 内側ディスプレイでステータスバーの左側まで広がる |
| iOS 27.1 | 画面端まで広がり、標準のバーがステータスバー下に縦配置される |

検証は Xcode 27.1 の Device Hub にある iPhone Duo シミュレータで行います。画面下部のコントロールで開閉・回転・折り曲げを切り替えられます。

## どこを読むか

作業内容に応じて必要な参照だけを読んでください。

| 参照 | 扱う内容 |
|---|---|
| `references/layout.md` | size class、safe area の非対称性、画面の角、reserved regions、arrangement |
| `references/bars.md` | 垂直バー、項目の配置順、軸の制御、バッジ、オーバーフロー、無効化 |
| `references/scenes.md` | ヒンジ、マルチタスキング、複数ウインドウ、scene accessories |
| `references/camera.md` | デュアル前面カメラ、カメラの向き、プレビュー、回転 |
| `references/checklist.md` | 既存アプリの移行チェックリスト |

## 全体に効く原則

これらは領域を問わず効いてくるので、先に頭に入れておくと判断が速くなります。

**size class で判断する。** user interface idiom から画面サイズやデバイス機能を推定したり、interface orientation でレイアウトを分岐したりしないでください。内側ディスプレイはアプリが宣言した supported interface orientations に従って回転せず、代わりにスケーリングされます。向きを見ても意図した結果になりません。

**画面を直接参照しない。** `UIScreen.main` は2画面のデバイスでは曖昧になり、ドキュメント上すでに非推奨です。environment、trait collection、scene の bounds を使い、画面が必要なら `window?.windowScene?.screen` から取ります。画面スケールは `displayScale` / `traitCollection.displayScale` で足ります。

**safe area は非対称になる。** 向かい合う辺のインセットが等しいという前提を置いたコードは壊れます。`view.bounds.width - insets.left * 2` のような書き方をやめ、各辺を個別に扱ってください。layout margins も同様に非対称です。

**標準コンポーネントを使うと多くが自動で解決する。** 標準のナビゲーション、バー、シート、アラート、メニューは各姿勢と折り目に適応します。折り目を避ける動作もシステム側に組み込まれています。独自実装に置き換えるほど、自分で面倒を見る範囲が増えます。

## iOS 27.1 の API について

iPhone Duo 向けに追加される API の多くは、セッションの公式コードサンプルには登場するものの、**公開ドキュメントの索引にも iOS 27.0 SDK にも含まれていません**。arrangement、reserved regions、ヒンジ、垂直バーの制御、カメラの direction coordinator が該当します。

各参照ではサンプルで確認できた綴りと使い方を記載していますが、完全なシグネチャや値の一覧は正式なドキュメントの公開を待つ必要があります。**実装時は必ず Xcode の補完と実機/シミュレータで確認してください。**

## 回答の仕方

説明・提案・検証結果は日本語で書いてください。API 名・型名・識別子・出典のタイトルは原綴りのまま扱います。

未公開 API を扱う性質上、**確認できないシグネチャを推測で補わないでください。**Xcode の補完や実機・シミュレータで確かめられない場合は、その旨を明示したうえで提案してください。誤った綴りをそれらしく書くほうが、分からないと言うより害になります。

## 出典

- Tech Talks: Prepare your app for iPhone Duo / Raise the bar with iPhone Duo / Strike a pose with adaptive layouts on iPhone Duo / Leverage multiple displays and scenes on iPhone Duo / Build a great camera experience for iPhone Duo / Design for iPhone Duo
- Human Interface Guidelines: Designing for iPhone Duo
