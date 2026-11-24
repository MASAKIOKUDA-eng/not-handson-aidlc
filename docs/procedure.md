# 検証 A の手順（AI-DLC なし）

## 0. 事前準備

1. このリポジトリを手元に clone し、作業用のブランチを作る

   ```bash
   git clone https://github.com/MASAKIOKUDA-eng/not-handson-aidlc.git
   cd not-handson-aidlc
   git switch -c run/a-YYYYMMDD
   ```

2. このフォルダに AI-DLC の設定（`.claude/` 配下の aidlc 関連ファイル、`aidlc/`）が **ない** ことを確認する
3. Claude Code のバージョンと、使うモデル・effort を [evaluation-sheet.md](evaluation-sheet.md) の「1. 実施条件」に書く
4. 開始時刻を書く（上限は 60 分）

## 1. 依頼する

1. このフォルダで Claude Code を起動する
2. [prompts/initial-request.md](../prompts/initial-request.md) の依頼文を **一字一句そのまま** 入力する
3. 続けて、成果物の置き場所だけを伝える（A・B の差が出ないよう、内容に関する追加の指示はしない）

   ```text
   成果物は scenario/ フォルダに Markdown で保存してください。
   ```

## 2. やり取りする

- 2 回目以降は、ふだん生成 AI を使うときと同じように、自由に確認・修正依頼をしてかまいません
- ただし **参考資料（既存のリポジトリ、URL、社内資料など）は渡さない**
- 「直してほしい」と依頼したときは、その回数を数えておく（評価シートの「差し戻しの回数」）

## 3. 終える

1. 60 分に達したら、途中でも終える（時間内に終わった場合は、その時刻を書く）
2. Claude Code で `/export` を実行し、会話のログを `records/conversation-a.md` として保存する
3. ログと成果物に、アカウント ID、メールアドレス、端末のユーザー名やパスなどがないか確認し、あれば伏せ字にする
4. コミットして push する

   ```bash
   git add scenario/ records/ docs/evaluation-sheet.md
   git commit -m "run: 検証 A を実施"
   git push -u origin run/a-YYYYMMDD
   ```

## 4. 評価する

[evaluation-sheet.md](evaluation-sheet.md) は、A・B の両方が終わってから記入します（片方だけ先に評価すると、もう片方の進め方に影響するため）。
