# Git / Issue / PR ワークフロー

**読むタイミング**: ブランチを切る時、PRを作る時、リリース（本番反映）する時。

個人開発前提の軽量フロー。ただし「全ての修正はIssue起点」「本番反映前にdevelopで人間が
検証する」の2点は必ず守る。

## ブランチ構成

- `main`: 本番。Firebase App Hostingの本番用backend（live branch = `main`）。
  直接pushしない。PRのみで更新する
- `develop`: **リポジトリのデフォルトブランチ**。開発/検証用。Firebase App Hosting
  のdevelop用backend（live branch = `develop`）に自動デプロイされる
- `feature/{issue番号}-slug` / `fix/{issue番号}-slug`: 通常の作業ブランチ。
  `develop`から切り、`develop`へPRを出す
- `hotfix/{issue番号}-slug`: 本番で今すぐ直したい緊急修正のみ、例外的に`main`から切る

デフォルトブランチを`develop`にしているのは、GitHubの「Issueからブランチ作成」も
Claude Code GitHub Actionが開くPRも、何も指定しなければデフォルトブランチ起点に
なるため。これにより「うっかりmainに直接影響が及ぶ」事故を構造的に防いでいる。

## 通常フロー

1. Issue作成（`.github/ISSUE_TEMPLATE/`のテンプレートを使う）
2. Issue画面から「Create a branch」で`develop`ベースの作業ブランチを作成
3. 実装 → `develop`向けにPRを作成。本文に`Refs #<issue番号>`のようにIssue番号を
   明記する（`Closes`/`Fixes`等のクローズキーワードは使わない。マージ時点では
   まだdevelopでの手動検証が済んでいないため、Issueは自動クローズさせない）
4. PRで`.github/workflows/ci.yml`が走る。Firebase App HostingのGitHub連携で
   PRごとのプレビュー用ロールアウトが自動生成され、PRにURLがコメントされる
   （PR単体の動作確認用。Firebase Console側でPRプレビューの発行を有効にしておく）
5. `develop`にマージ → develop用backendのlive branchが更新され、develop固定URLに
   反映される。**ここで実際に触って手動検証し、問題なければ対応するIssueを
   手動でクローズする**
6. 検証OKなものがある程度溜まったら、`develop → main`のリリースPRを作成してマージ
   → 本番デプロイ

## Claude Codeに直接Issue対応を依頼する場合

1. Issueに `@claude ◯◯して` とコメントする（GitHub Web/モバイルアプリどちらでも可）
2. `.github/workflows/claude-code.yml`が起動し、デフォルトブランチ(`develop`)を
   起点にブランチを作成してPRを自動で開く
3. PRのプレビューロールアウト、またはマージ後のdevelop固定URLで人間が検証する。
   問題なければ対応するIssueを手動でクローズする（Claude Codeが開くPRの本文も
   クローズキーワードを使わない書き方になるよう、Issueコメントで依頼する際に
   伝える。「PRの書き方」参照）
4. 問題なければ通常フロー同様、develop→mainのリリースPRでまとめて本番反映する

Claudeが開いたPRであっても、mainへの直接マージは行わない。必ずdevelopを経由させる。

## hotfixの例外パス

本番で今すぐ直す必要がある場合のみ:

1. `main`から`hotfix/{issue番号}-slug`を切る
2. `main`へ直接PR、マージして本番反映
3. マージ後、`main`を`develop`に取り込むPRを作成してマージする
   （`develop`も直接push禁止のため、backmergeもPRを経由する。
   `git checkout -b sync/main-to-develop main && git push` してから
   `develop`向けPRを作る）

hotfixは`develop`での検証待ちという中間状態を挟まず、マージ＝即本番反映
であるため、通常フローと異なり`Closes #<issue番号>`を使ってよい（マージと
同時にIssueをクローズして問題ない）。

## PRの書き方

- タイトルは `feat: ...` / `fix: ...` / `chore: ...` 程度のprefixで揃える
  （厳密なConventional Commits運用はしない。可読性のためだけ）
- 本文には最低限 `Refs #<issue番号>`（`Closes`/`Fixes`等のクローズキーワードは
  使わない。理由は「通常フロー」参照）と、動作確認方法（develop環境のURLで
  何を確認したか）を書く
- レビュー必須のルールは設けない（個人開発のため）。ただしCIが通っていない
  PRはマージしない

## ブランチ保護（GitHubリポジトリ設定・手動）

**注意**: 必須レビュー・必須ステータスチェックなどのブランチ保護ルールは、
Privateリポジトリでは**GitHub Pro以上のプランが必要**な機能。個人開発で
Privateリポジトリ+Freeプランを使っている場合、GitHub側でこれらを強制することは
できない（設定しようとすると "Please ensure the billing plan supports the
required reviewers protection rule." のようなエラーで拒否される）。

- **Free/Privateの場合（デフォルト）**: GitHub側の強制設定はせず、以下を
  **規約として守る**運用にする
  - `main`/`develop`へは直接pushしない。必ずPR経由で更新する
  - CIが通っていないPRはマージしない（下記「PRの書き方」参照）
  - リポジトリのDefault branchは`develop`に設定する
- **Pro以上のプランを使っている場合**: 上記に加えて実際にGitHub側でも強制できる
  - `main`: PR必須 / CIのstatic-checksとintegration-testsを必須チェックに設定 /
    直接push禁止
  - `develop`: 同上（個人開発でも統一した方がミスが減る）
  - **ただし最初のうちは必須チェックを設定しない**こと。`package.json`の
    スクリプトや`src/types/database.ts`が存在しないうちはCIが必ず失敗するため、
    最初のスキャフォールドPRを1回通してから必須チェックを有効にする

`.github/workflows/ci.yml`は`develop`向けPRと`main`向けPR（リリースPR）の両方で
発火する。E2Eだけは重いので`main`向けPR（リリースPR）の時だけ実行される。

## デプロイ先・インフラ関連

Firebase App Hostingのセットアップ、Supabaseプロジェクトのセットアップ、
マイグレーションの自動デプロイについては`docs/deployment.md`を参照。

## リリース手順

`develop → main`のリリースPRをマージしたあとの手順。

1. `main`へのマージ後、gitタグを作成する（`v1.0.0`のようなセマンティック
   バージョニング形式）
2. `gh release create <タグ> --generate-notes`でGitHub Releaseを作成する。
   リリースノートはこのコマンドの自動生成（マージ済みPRの一覧から生成）に
   任せ、CHANGELOG.mdは手動運用しない
3. ロールバックが必要になった場合は、Firebase App Hostingのロールアウト履歴
   から直前の正常なロールアウトに戻す。gitタグを戻す・リバートコミットを積む
   といった手順は基本的に組まず、ホスティング側のロールバック機能に任せる

**バージョンアップ（`package.json`の`version`を上げる）PR自体は、「すべての
変更はIssue起点」ルールの例外として扱ってよい。** リリースという行為そのもの
であり、機能追加・修正のような通常の変更ではないため。
