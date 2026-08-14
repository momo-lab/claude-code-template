# claude-code-template

Next.js (App Router) + Firebase App Hosting + Supabase構成で、Claude Codeと一緒に開発する
プロジェクト用の共通テンプレートです。新しいプロジェクトを始めるたびに、
このリポジトリの中身をプロジェクトルートにコピーして使います。

## これは何か

- `CLAUDE.md` … Claude Codeがセッション開始時に必ず読む、プロジェクト共通ルールの雛形
- `docs/` … アーキテクチャ・スコープ・テスト方針・Git運用など、詳細ドキュメントの雛形
- `.github/` … Issue/PRテンプレートと、CI・Claude Code起動用のGitHub Actions雛形

雛形として空欄・プレースホルダー（`{{PROJECT_NAME}}`など）を含んでいるため、
コピーしたあとにプロジェクトごとの内容を埋めて使います。

Git/PR運用やワークフローのトリガー設計（`@claude`のOWNER限定起動、レビュー必須に
しない方針など）は個人開発（一人開発）を前提にしています。詳細は
`docs/git-workflow.md`・`docs/config-templates.md`を参照してください。

## 含まれるもの

```
{project-root}/
├── CLAUDE.md                          ← リポジトリ直下に配置。PROJECT_NAME等を書き換える
├── docs/
│   ├── architecture.md                ← プロジェクトごとに内容を埋める
│   ├── testing-strategy.md            ← 汎用。プロジェクト固有の項目だけ追記
│   ├── scope.md                       ← プロジェクトごとに内容を埋める
│   ├── git-workflow.md                ← 汎用。ドメイン名などプロジェクト固有部分だけ調整
│   ├── config-templates.md            ← 汎用。.github/配下の設定ひな型の設計意図
│   └── docs-sync-policy.md            ← 汎用。docsと実ファイルの整合性ルール
└── .github/
    ├── ISSUE_TEMPLATE/                ← bug_report.yml / feature_task.yml / config.yml
    ├── pull_request_template.md
    └── workflows/
        ├── ci.yml.template            ← 利用開始時に`.yml`へリネームして有効化する
        └── claude-code.yml.template   ← 同上。Issue/PRで@claudeメンションした時に起動
```

## 注意: ワークフローが`.template`拡張子になっている理由

GitHub Actionsは、ワークフローファイルが**置かれているリポジトリ**で発火します。
`ci.yml` / `claude-code.yml`をそのままの拡張子でこのテンプレートリポジトリに
置いておくと、テンプレート自身を更新するPRやコメントでも動いてしまいます。
特に`ci.yml`は`package.json`やNext.js/Supabaseの実体を前提にしているため、
このリポジトリ自体を更新するPRでは`pnpm install`などが必ず失敗します。

これを避けるため、このリポジトリでは`ci.yml.template` / `claude-code.yml.template`
という名前で無効化した状態で管理しています。プロジェクトにコピーしたあと、
`.template`を外して`.yml`にリネームすることで初めて有効化されます
（下記の手順1）。

## 新しいプロジェクトを始める時の手順

1. このディレクトリの中身をコピーし、`CLAUDE.md`冒頭の`{{PROJECT_NAME}}`と
   プロダクト概要（1〜3行）を書き換える。あわせて`.github/workflows/`配下の
   `ci.yml.template` / `claude-code.yml.template`を、それぞれ`.template`を
   取った`ci.yml` / `claude-code.yml`にリネームして有効化する
2. `docs/architecture.md`のテンプレート項目（DB設計・権限モデル・スコープ）を埋める
   <!-- ここが一番重要。空のままだとClaudeが仕様を推測して実装してしまう -->
3. GitHubリポジトリ作成、コミット。**Default branchを`develop`に変更**
   （Settings > General > Default branch）
4. Settings > General で「Automatically delete head branches」を有効化
   （マージ後の作業ブランチを自動削除し、ブランチが溜まるのを防ぐ）
5. `main` / `develop` にブランチ保護ルールを設定する。**Privateリポジトリでは
   必須レビュー等の保護ルールはGitHub Pro以上が必要**なため、Freeプランの場合は
   GitHub側で強制せず「直接pushしない」を規約として守る運用にする（詳細は
   `docs/git-workflow.md`の「ブランチ保護」参照）。Pro以上で実際に設定する場合も、
   **最初のうちは必須チェックを設定しない**こと。`package.json`のスクリプトや
   `src/types/database.ts`が存在しないうちはCIが必ず失敗するため、最初の
   スキャフォールドPRを1回通してから必須チェックを有効にする
6. `package.json`に`lint` / `typecheck` / `build` / `test:unit` /
   `test:integration` / `test:e2e` のスクリプトを定義
   （Next.jsプロジェクト未初期化なら、Claude Codeに「CLAUDE.mdとdocs/を読んで
   雛形をセットアップして」と頼めばスクリプトも含めて提案してくれる）
7. `claude-code.yml`を使うので、リポジトリの Settings > Secrets に
   `ANTHROPIC_API_KEY` を登録
8. Firebaseプロジェクトを作成し、App Hostingでリポジトリと連携する。
   本番用/develop検証用で別々のbackendを作成し、それぞれのlive branchを
   `main`/`develop`に設定する（backendごとに固定ドメインが発行される）
9. Supabaseプロジェクトを作成し、`supabase init` → `supabase/migrations/`に
   `docs/architecture.md`のテーブル定義を反映。本番用とdevelop検証用でDBを分離

## Claude Codeへの最初の指示例

> CLAUDE.md と docs/ 以下を読んで、このプロジェクトのNext.js + Supabase
> プロジェクトの雛形をセットアップして。package.jsonのスクリプトは
> .github/workflows/ci.yml が要求しているコマンド名（lint, typecheck,
> build, test:unit, test:integration, test:e2e）に合わせて。

## スマホから使う場合

- 軽い修正・調査・PRレビュー依頼 → Claude Code on the web（claude.ai/code）
  またはClaudeモバイルアプリから直接
- ローカルのSupabase Docker環境が必要な作業の続きをスマホで見る/操作する
  → PC側で `claude` を起動した状態で Remote Control を有効化し、
  モバイルアプリの「Code」タブから接続
- PCを開いていない時でも指示だけ出したい → GitHubのissue/PRコメントで
  `@claude ◯◯して` とコメントすれば `claude-code.yml` 経由でクラウド上の
  Claude Codeが動く（GitHubモバイルアプリからでもOK）。開いたPRはdevelopが
  base branchになるので、developの固定URLで検証してからmainへ反映する

## 使い回す上での注意

- `CLAUDE.md`は200行程度を目安に収める。プロジェクト固有の詳細が増えてきたら
  `docs/`に新しいファイルを足し、CLAUDE.mdからは1行で参照するだけにする
- CLAUDE.mdにコーディング規約を書き足したくなったら、まずLinter/Formatterの
  ルールにできないか検討する（Claudeへの指示より機械的な強制の方が確実で安い）
