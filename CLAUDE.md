# {{PROJECT_NAME}}

<!-- 1〜3行で「何のプロダクトか」「誰のためか」「コアコンセプトは何か」を書く。
     これがClaudeにとっての最重要コンテキストになる。 -->

## 技術スタック

- フロントエンド/インフラ: Next.js (App Router) / Firebase App Hosting
- DB/バックエンド: Supabase (PostgreSQL + RLS + Storage)
- 認証: Supabase Auth
- テスト: Vitest（unit）/ 統合テスト（ローカルSupabaseに対して実行）/ Playwright（E2E）
<!-- 決済・通知など、プロジェクト固有のサービスがあればここに追記 -->

## コマンド

```bash
pnpm dev                # 開発サーバー
pnpm lint                # Lint
pnpm typecheck           # tsc --noEmit
pnpm build                # 本番ビルド確認
supabase start                # ローカルSupabase起動（Docker）
supabase db reset             # ローカルDBをmigrationから再構築 + seed投入
pnpm test:unit              # ユニットテスト
pnpm test:integration       # 統合テスト（ローカルSupabase必須）
pnpm test:e2e                # Playwright E2E
supabase gen types typescript --local > src/types/database.ts  # DB型生成
```

## ディレクトリ構成（要点のみ）

```
src/app/            # Next.js App Router
src/lib/             # ドメインロジック（DBに依存しない純粋関数はここ）
supabase/migrations/ # DBスキーマ変更履歴（必ずmigrationファイル経由で変更する）
supabase/seed.sql    # テスト/開発用シードデータ
tests/integration/   # Server Action / API Route単位の統合テスト
tests/e2e/            # Playwright E2E（主要フローのみ）
docs/                 # 詳細ドキュメント（下記参照）
```

## 参照ドキュメント（必要な時だけ読む）

- `docs/architecture.md` — DBスキーマ・権限モデル・認証フローなどの詳細設計
  <!-- プロジェクトごとに書く。テーブル定義、RLSの方針、権限モデルなど -->
- `docs/testing-strategy.md` — テスト方針の詳細（IT中心の理由と書き方）
- `docs/scope.md` — 今のフェーズで実装する/しない機能の線引き
  <!-- MVPスコープなど、プロジェクトごとの意思決定を書く -->
- `docs/git-workflow.md` — ブランチ運用・Issue/PRの起こし方・リリースの手順
- `docs/config-templates.md` — `.github/`配下の設定ひな型がなぜ今の形なのか
- `docs/docs-sync-policy.md` — docsと実ファイルの整合性ルール（下記参照）

## 開発時の重要な決め事（プロジェクト共通のデフォルト方針）

- コミットメッセージ・PR/Issueの本文・コメントなど、コミュニケーションはすべて日本語で行う
- DBのView/RPC/triggerは極力使わない。JOINの共通化はアプリ側のクエリ関数で行う
  （Claudeにとって追いやすく、テストもしやすいため。強い理由があれば例外可）
- RLSは「テナント単位のデータ分離」など粗い境界の担保に使い、role別の細かい操作可否は
  アプリ層で実装する（RLSポリシーが複雑化すると可読性・デバッグ性が落ちるため）
- スキーマ変更は必ず`supabase/migrations`に新しいmigrationファイルを追加する形で行う。
  既存migrationの書き換え禁止
- すべての変更はIssue起点。作業ブランチはリポジトリのデフォルトブランチ(develop)から切り、
  developへPRを出す（mainへの直接変更はしない）。詳細は`docs/git-workflow.md`
- **`docs/`配下の記述は、実装済みの内容について実ファイルと必ず一致させる。** 未実装の
  先行設計は実装状況マーカーで明示する。設定・スキーマ・テスト方針など実ファイルを
  変更したら、対応する`docs/`も同じPRで更新する（マーカーの解消含む）。理由なく
  食い違っていることに気づいたら黙って進めず、その場で直すか人間に確認する。
  詳細は`docs/docs-sync-policy.md`
- コーディング規約・フォーマットはLinter/Formatterに任せる。CLAUDE.mdに書き足さない
  （Lintで機械的に強制できることをプロンプトで指示するのは非効率）

<!--
  このファイルは200行程度に収める。詳細はdocs/に逃がし、Claudeが必要な時だけ読みに行く
  構成（progressive disclosure）にする。プロジェクト固有の情報を書き足す際は、
  「毎セッション絶対に必要か」を基準に、ここに書くかdocs/に書くか判断すること。
-->
