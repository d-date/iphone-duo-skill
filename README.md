# iPhone Duo Skill

Apple の iPhone Duo にアプリを対応させるための [Claude Code](https://claude.com/claude-code) skill です。

Tech Talks 6本、Human Interface Guidelines、iPhone Duo Group Lab（2026-09-16）を一次情報として整理しています。API の綴りと使い方は、各セッションページに掲載された公式コードサンプルで確認しています。

## 収録内容

| ファイル | 扱う内容 |
|---|---|
| `skills/iphone-duo/SKILL.md` | 全体の原則と、どの参照を読むかの案内 |
| `references/layout.md` | size class、safe area の非対称性、画面の角、reserved regions、arrangement |
| `references/bars.md` | 垂直バー、項目の配置順、軸の制御、バッジ、オーバーフロー、無効化 |
| `references/scenes.md` | ヒンジ、マルチタスキング、複数ウインドウ、scene accessories |
| `references/camera.md` | デュアル前面カメラ、カメラの向き、プレビュー、回転 |
| `references/checklist.md` | 既存アプリの移行チェックリスト |

## インストール

### npx を使う

```bash
npx skills add d-date/iphone-duo-skill
```

対話的に、グローバル（`~/.claude/skills/`）とプロジェクト（`.claude/skills/`）のどちらへ入れるか選べます。更新は `npx skills update`、削除は `npx skills remove` です。

### 手動で置く

```bash
git clone https://github.com/d-date/iphone-duo-skill.git
mkdir -p ~/.claude/skills
cp -r iphone-duo-skill/skills/iphone-duo ~/.claude/skills/
```

プロジェクト単位で使う場合は `.claude/skills/` 配下に置いてください。

## 注意

iPhone Duo 向けの API の多くは iOS 27.1+ Beta としてドキュメントが公開されています。ベータ版のため、正式リリースまでに変わる可能性があります。

SwiftUI の `onHingeChange`、`toolbarVerticalBehavior(_:)`、`toolbarVerticalCompressionBehavior(_:)` は、2026-09-17 時点でドキュメントに見当たらず、セッションの公式コードサンプルでのみ確認しています。実装時は Xcode の補完と実機・シミュレータで確認してください。

## 出典

- Tech Talks: Prepare your app for iPhone Duo / Raise the bar with iPhone Duo / Strike a pose with adaptive layouts on iPhone Duo / Leverage multiple displays and scenes on iPhone Duo / Build a great camera experience for iPhone Duo / Design for iPhone Duo
- Human Interface Guidelines: Designing for iPhone Duo
- Meet with Apple: [iPhone Duo Group Lab](https://www.youtube.com/watch?v=0zp4gAgC6TI)（2026-09-16）
- Apple Developer Documentation: Choosing a camera by the direction it faces / Supporting device rotation in your camera app / iOS 27.1 Beta の API リファレンス

本 skill の文章は上記を読んで整理したもので、Apple の文章やコードサンプルをそのまま収録したものではありません。

## 更新履歴

- 2026-09-17: Apple のカメラ関連ドキュメント2本と iOS 27.1 Beta の API リファレンスを反映
- 2026-09-17: iPhone Duo Group Lab（2026-09-16）の回答を反映
- 2026-09-11: 初版

## ライセンス

MIT License
