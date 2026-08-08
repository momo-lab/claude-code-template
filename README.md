# claude-code-template

Next.js (App Router) + Vercel + Supabase構成で、Claude Codeと一緒に開発する
プロジェクト用の共通テンプレートです。新しいプロジェクトを始めるたびに、
このリポジトリの中身をプロジェクトルートにコピーして使います。

## これは何か

- `CLAUDE.md` … Claude Codeがセッション開始時に必ず読む、プロジェクト共通ルールの雛形
- `docs/` … アーキテクチャ・スコープ・テスト方針・Git運用など、詳細ドキュメントの雛形
- `.github/` … Issue/PRテンプレートと、CI・Claude Code起動用のGitHub Actions雛形

雛形として空欄・プレースホルダー（`{{PROJECT_NAME}}`など）を含んでいるため、
コピーしたあとにプロジェクトごとの内容を埋めて使います。

## 含まれるもの

```
{project-root}/
├── CLAUDE.md                          ← リポジトリ直下に配置。PROJECT_NAME等を書き換える
├── docs/
│   ├── architecture.md                ← プロジェクトごとに内容を埋める
│   ├── scope.md                       ← プロジェクトごとに内容を埋める
│   ├── testing-strategy.md            ← 汎用。プロジェクト固有の項目だけ追記
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

## クイックスタート

1. このリポジトリの中身を新しいプロジェクトのルートにコピーする
2. `.github/workflows/`配下の`ci.yml.template` / `claude-code.yml.template`を、
   `.template`を外した`ci.yml` / `claude-code.yml`にリネームして有効化する
3. `CLAUDE.md`冒頭の`{{PROJECT_NAME}}`とプロダクト概要を書き換える
4. `docs/architecture.md`・`docs/scope.md`などプロジェクト固有の項目を埋める

GitHubリポジトリ作成やブランチ保護、Vercel/Supabaseの設定など、その先の詳しい
手順は **[SETUP-README.md](./SETUP-README.md)** にまとめています。

## 注意: ワークフローが`.template`拡張子になっている理由

GitHub Actionsは、ワークフローファイルが**置かれているリポジトリ**で発火します。
`ci.yml` / `claude-code.yml`をそのままの拡張子でこのテンプレートリポジトリに
置いておくと、テンプレート自身を更新するPRやコメントでも動いてしまいます。
特に`ci.yml`は`package.json`やNext.js/Supabaseの実体を前提にしているため、
このリポジトリ自体を更新するPRでは`npm ci`などが必ず失敗します。

これを避けるため、このリポジトリでは`ci.yml.template` / `claude-code.yml.template`
という名前で無効化した状態で管理しています。プロジェクトにコピーしたあと、
`.template`を外して`.yml`にリネームすることで初めて有効化されます
（クイックスタートの手順2）。
