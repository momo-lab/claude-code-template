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
3. 実装 → `develop`向けにPRを作成。本文に`Closes #<issue番号>`を必ず書く
   （マージ時にIssueが自動クローズされる）
4. PRで`.github/workflows/ci.yml`が走る。Firebase App HostingのGitHub連携で
   PRごとのプレビュー用ロールアウトが自動生成され、PRにURLがコメントされる
   （PR単体の動作確認用。Firebase Console側でPRプレビューの発行を有効にしておく）
5. `develop`にマージ → develop用backendのlive branchが更新され、develop固定URLに
   反映される。**ここで実際に触って手動検証する**
6. 検証OKなものがある程度溜まったら、`develop → main`のリリースPRを作成してマージ
   → 本番デプロイ

## Claude Codeに直接Issue対応を依頼する場合

1. Issueに `@claude ◯◯して` とコメントする（GitHub Web/モバイルアプリどちらでも可）
2. `.github/workflows/claude-code.yml`が起動し、デフォルトブランチ(`develop`)を
   起点にブランチを作成してPRを自動で開く
3. PRのプレビューロールアウト、またはマージ後のdevelop固定URLで人間が検証する
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

## PRの書き方

- タイトルは `feat: ...` / `fix: ...` / `chore: ...` 程度のprefixで揃える
  （厳密なConventional Commits運用はしない。可読性のためだけ）
- 本文には最低限 `Closes #<issue番号>` と、動作確認方法（develop環境のURLで
  何を確認したか）を書く
- レビュー必須のルールは設けない（個人開発のため）。ただしCIが通っていない
  PRはマージしない

## ブランチ保護（GitHubリポジトリ設定・手動）

- `main`: PR必須 / CIのstatic-checksとintegration-testsを必須チェックに設定 /
  直接push禁止
- `develop`: 同上（個人開発でも統一した方がミスが減る）
- リポジトリのDefault branchは`develop`に設定する

`.github/workflows/ci.yml`は`develop`向けPRと`main`向けPR（リリースPR）の両方で
発火する。E2Eだけは重いので`main`向けPR（リリースPR）の時だけ実行される。

## Firebase App Hostingセットアップ

- Firebaseプロジェクト内にApp Hostingのbackendを**本番用/develop検証用の2つ**
  作成し、それぞれのlive branchを`main`/`develop`に設定する（backendごとに
  `*.web.app`等の固定ドメインが発行される。独自ドメインを使う場合もbackendごとに
  個別に割り当てる）
- 環境変数・シークレットは`apphosting.yaml`（backendごとに
  `apphosting.<branch>.yaml`で分離可能）とSecret Manager経由で管理する。
  Vercelのようなダッシュボード上の平文環境変数ではなく、
  `firebase apphosting:secrets:set`でSecret Managerに登録し、`apphosting.yaml`の
  `env`欄で参照する
- 本番とdevelop検証環境ではSupabaseプロジェクト（または最低限DB/スキーマ）を
  分離し、手動検証で本番データを汚さないようにする
