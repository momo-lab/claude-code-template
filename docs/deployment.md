# デプロイ先・インフラのセットアップ

**読むタイミング**: デプロイ先やSupabaseプロジェクトを新規構築・移行する時。

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

## Supabaseプロジェクトのセットアップ

本番用/develop検証用のSupabaseプロジェクトを新規作成・作り直しする際の手順。

1. ダッシュボードでプロジェクトを作成する。**DBパスワードは記号を含まない
   英数字にする**（`supabase db push --db-url`に渡す接続文字列に記号入りだと
   URLパースエラーになりやすいため）
2. migrationを適用する
   - 通常は`SUPABASE_DB_URL_DEVELOP`/`SUPABASE_DB_URL_PROD`をリポジトリ
     secretsに設定しておけば、`develop`/`main`へのマージ（push）時に
     `.github/workflows/db-migrate.yml`が自動適用する（下記
     「マイグレーションの自動デプロイ」参照）
   - 手元で先に確認したい場合は
     `supabase db push --dry-run --db-url <Session poolerの接続文字列>`
     （副作用無しで内容を確認できる。`--dry-run`を外せば実際に適用される）
3. Firebase App Hostingの対応する環境（develop/production backend）に接続情報を
   設定する。値はSupabaseダッシュボードの Project Settings → API から取得し、
   Secret Manager経由で`apphosting.yaml`に反映する（具体的にどの値が必要かは
   採用する権限モデルによる。例: `SUPABASE_URL`、`SUPABASE_ANON_KEY`、
   `SUPABASE_SERVICE_ROLE_KEY`など。`docs/architecture.md`の「権限モデル」参照）
4. Supabase無料プランは一定期間アクセスが無いとプロジェクトがpauseされる。
   `.github/workflows/keep-alive.yml`を有効化し、本番/develop両方のURLを
   登録しておく（設計意図は`docs/config-templates.md`参照）

## マイグレーションの自動デプロイ

`.github/workflows/db-migrate.yml`が、`develop`/`main`へのpush（＝PRマージ）を
トリガーに`supabase db push --db-url`でmigrationを自動適用する。手動での
push忘れによる「コードとDBスキーマの乖離」を防ぐための仕組み（設計判断の詳細は
`docs/config-templates.md`参照）。
