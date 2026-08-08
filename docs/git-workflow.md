# Git / Issue / PR ワークフロー

**読むタイミング**: ブランチを切る時、PRを作る時、リリース（本番反映）する時。

個人開発前提の軽量フロー。ただし「全ての修正はIssue起点」「本番反映前にdevelopで人間が
検証する」の2点は必ず守る。

## ブランチ構成

- `main`: 本番。Vercel Production Branch。直接pushしない。PRのみで更新する
- `develop`: **リポジトリのデフォルトブランチ**。開発/検証用。Vercelの固定URL
  （開発系環境）に自動デプロイされる
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
4. PRで`.github/workflows/ci.yml`が走る。VercelがPreviewデプロイし、PRにURLが
   コメントされる（PR単体の動作確認用）
5. `develop`にマージ → Vercelの開発系環境（固定URL）に反映される。
   **ここで実際に触って手動検証する**
6. 検証OKなものがある程度溜まったら、`develop → main`のリリースPRを作成してマージ
   → 本番デプロイ

## Claude Codeに直接Issue対応を依頼する場合

1. Issueに `@claude ◯◯して` とコメントする（GitHub Web/モバイルアプリどちらでも可）
2. `.github/workflows/claude-code.yml`が起動し、デフォルトブランチ(`develop`)を
   起点にブランチを作成してPRを自動で開く
3. PRのVercel Preview、またはマージ後のdevelop固定URLで人間が検証する
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

## Vercel設定

- Production Branch = `main`
- `develop`ブランチに固定ドメイン（例: `dev.example.com`）を割り当てる
  - Hobbyプラン: Preview環境の中で`develop`ブランチに固定ドメインを割り当てる
    方式で代替する
  - Proプラン以上: Custom Environmentとして`staging`を作成し、`develop`への
    branch trackingを設定すると、専用の環境変数・専用ドメインを綺麗に分離できる
- 本番とdevelop検証環境ではSupabaseプロジェクト（または最低限DB/スキーマ）を
  分離し、手動検証で本番データを汚さないようにする
