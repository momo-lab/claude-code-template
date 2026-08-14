# アーキテクチャ詳細

**読むタイミング**: DBスキーマ変更、権限周り、認証フローを実装・修正する時。

このファイルはプロジェクトごとに内容を埋めて使う。以下は書くべき項目の雛形。

## 認証方針

<!--
例:
- どの認証方式を使うか（メール+パスワード、OAuth、匿名認証など）
- ロールや権限の持たせ方
- 招待/オンボーディングフローがあれば、その分岐条件
-->

## DB設計（主要テーブル）

<!--
テーブルごとに以下を書く:
- テーブル名と役割
- 重要なカラムとその意味（NULL許容の理由など、コードだけでは読み取れない設計判断）
- 他テーブルとの関係

例:
- `users`: 認証のみ。表示情報は持たない
- `organizations`: 契約・課金単位
-->

## 権限モデル

<!--
- ロールの一覧と、それぞれ何ができるか
- RLSでどこまで担保し、アプリ層で何を担保するかの切り分け
  （CLAUDE.mdの「開発時の重要な決め事」と矛盾しないように）
-->

### Supabaseのデフォルト権限に関する注意

<!-- プロジェクトで最初のmigrationを書く際に必ず確認すること -->

**service_roleへのアクセス権**: 新しめのSupabase CLIでは、`public`スキーマに
新規テーブルを作成しても`service_role`を含むData APIロールへのアクセス権が
自動付与されない（`auto_expose_new_tables`の挙動変更）。Server Actionから
`service_role`キーでテーブルにアクセスする設計の場合、最初のmigrationで
明示的にGRANTしないと`permission denied for table ...`で失敗する。

```sql
-- 例: publicスキーマの全テーブルへservice_roleのフルアクセスを許可する
grant usage on schema public to service_role;
grant all on all tables in schema public to service_role;
alter default privileges in schema public grant all on tables to service_role;
```

**anon/authenticatedへの不要なGRANT**: 「RLSを有効化してポリシーを定義しない
＝拒否」という設計は、GRANTレベルで`anon`/`authenticated`ロールに標準権限が
残っていると実質的に不完全になる。ローカルDocker環境だけで開発していると
気づきにくく、`supabase db diff --linked`で実プロジェクトとの差分を取って
初めて発覚しやすい落とし穴。RLS deny-by-defaultを採用する場合は、以下のように
明示的にREVOKEしておく（`alter default privileges`まで設定しておくと、以後
追加するテーブルにも自動的に適用される）。

```sql
-- 例: anon/authenticatedからpublicスキーマの標準権限を剥奪する
revoke all on all tables in schema public from anon, authenticated;
alter default privileges in schema public
  revoke all on tables from anon, authenticated;
```

## スコープ

<!--
権限まわりの現フェーズのスコープは docs/scope.md に委譲し、ここで重複させなくてよい。
-->
