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

### 選択肢: Capability URL方式（アカウント登録不要）

<!-- 「共有リンクを知っている＝アクセス権」で完結する小規模な共同作業ツール
（日程調整・匿名アンケート・共有チェックリストなど）の場合、Supabase Authに
よるアカウントベースの認証ではなくこちらを選ぶ余地がある -->

- 推測困難なID（十分なエントロピーを持つランダムID）をURLに含め、それ自体を
  アクセス権とする。アカウント登録・ログインは導入しない
- 閲覧専用など権限を分けたい場合は、別途`view_token`のような専用カラムを
  用意する（同じIDをクエリパラメータで使い分けるより、権限ごとに別カラムに
  した方がミスなく扱える）
- RLSは有効化するがポリシーは一切定義しない（ハード拒否）。アプリは
  `service_role`、または`role`claimを含む自己署名JWTをサーバー側で発行して
  PostgREST（Supabase Data API）にリクエストし、IDによるスコープ絞り込みは
  アプリ層（`src/server/`）で行う
- **これはUIレベルのアクセス制御であり、セキュリティ境界そのものではない**
  ことを明示的に認識する。リンクが漏洩すれば、そのリンクを持つ誰でも
  読み書きできる。取り扱う情報の機微さに応じて採用可否を判断する

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

### 選択肢: 複数アプリの同居（スキーマ+専用ロールでの分離）

<!-- Supabase無料プランは「1アカウントで同時にアクティブにできる無料
プロジェクトが2つまで」という制約がある。複数の小規模アプリを1つの
Supabaseプロジェクトに同居させたい場合の選択肢。通常は1プロジェクト1アプリ
（publicスキーマ）がデフォルトでよく、この制約に当たらない限り不要 -->

- アプリのテーブルは`public`ではなく、アプリ専用のスキーマ（例: `myapp`）に
  作成する。DBへは専用ロール（例: `myapp_role`）のみがアクセスでき、他の
  スキーマ（同一プロジェクトに将来同居する別アプリのスキーマなど）へは
  `permission denied for schema`で拒否される。境界はPostgresのGRANT/REVOKE
  で強制され、アプリ側のコードミスに依存しない
- アプリはServer Action内で、専用ロールを`role`claimに含む自己署名JWTを
  発行してPostgREST（Supabase Data API）にリクエストする
  （`grant <専用ロール> to authenticator`が必須。PostgRESTがJWTの`role`claim
  を見て`SET ROLE`できるようにするため）
- **ハマりどころ**: リモートのSupabaseプロジェクトでは、Data APIが専用
  スキーマを公開する設定（Exposed schemas）が別途必要になる。
  `supabase/config.toml`の`[api] schemas`はローカルDocker専用で`db push`
  では反映されないため、`authenticator`ロールの`pgrst.db_schemas`を直接
  設定するmigrationを用意し、ダッシュボードでの手動設定に頼らないようにする

  ```sql
  alter role authenticator set pgrst.db_schemas = 'myapp';
  notify pgrst; -- 設定変更 + スキーマキャッシュの両方をリロードする
  ```

  （`notify pgrst, 'reload config'`は設定のみのリロードで、テーブル定義の
  スキーマキャッシュはリロードされない点に注意。引数無しの`notify pgrst`で
  両方リロードされる）

## スコープ

<!--
権限まわりの現フェーズのスコープは docs/scope.md に委譲し、ここで重複させなくてよい。
-->
