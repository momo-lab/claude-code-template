# 設定ひな型の設計方針

**読むタイミング**: `.github/`配下の設定ファイルを変更・追加する時、CIやIssue/PR運用
そのものを見直す時。

このリポジトリには「開発を回すための設定ひな型」が複数含まれている。それぞれ何のために
存在し、どういう考えで今の形になっているかをここに書く。**設定ファイルを変更する時は、
まずここを読んで設計意図を理解してから変更し、意図自体が変わるならこのファイルも
同じPRで更新すること。**

## CLAUDE.md

Claude Codeがセッション開始時に必ず読む唯一のファイル。「常に必要な最小限の地図」と
「詳細はdocs/を参照」という2層構造にすることで、コンテキストを圧迫せずに済ませている。
CLAUDE.mdにプロジェクト固有の詳細（DBスキーマ全部、テスト方針の全文など）を書き足す
運用はしない。書き足したくなったら`docs/`に新しいファイルを作り、CLAUDE.mdからは
1行で参照するだけにする。

## `.github/workflows/ci.yml`

**目的**: 「壊れたコードをdevelop/mainに入れない」ための最低限の自動チェック。

このテンプレートリポジトリ内では`ci.yml.template`という名前で無効化した状態で
管理している。GitHub Actionsはワークフローが置かれているリポジトリで発火するため、
`.yml`のまま置いておくとテンプレート自身を更新するPRでも起動してしまい、
`package.json`もNext.js/Supabaseの実体も無いため必ず失敗する。テンプレート採用時に
`.yml`へリネームして初めて有効化される。

設計判断:
- 静的検証（lint/typecheck/build）と統合テストは、**develop向けPRと main向けPR
  （リリースPR）の両方**で走らせる。日常の開発フローの主戦場はdevelopなので、
  ここでCIが走らないと事故に気づけない
- E2E（Playwright）は実行コストが高いため、**main向けPR（リリースPR）の時だけ**
  走らせる。本番反映直前の最終確認という位置づけ
- DB型ドリフト検出は独立ジョブにせず、`integration-tests`ジョブの1ステップとして
  統合している。別ジョブにすると`supabase start`（Docker起動）が重複してActionsの
  消費時間が増えるため、Supabaseを起動するジョブは極力まとめている
- ローカルSupabaseの起動・migration適用・接続情報のエクスポートは
  `.github/actions/setup-local-supabase`という複合アクション（composite action）
  に切り出し、`integration-tests`と`e2e-tests`の両方から呼び出している。
  以前は同じ内容を2ジョブにコピーしていたが、一方だけ更新して他方が古いままに
  なる（drift）事故が起きた。特に`e2e-tests`はmain向けPR（リリースPR）でしか
  発火しないため、setup手順の不整合はリリース時までCIで一度も検知されない
  ブラインドスポットになりやすい。共通化はこの再発防止策
- 接続情報（API_URL/ANON_KEY/SERVICE_ROLE_KEY/JWT_SECRET等）は固定の既知値を
  ワークフローにハードコードせず、`supabase status -o env`から都度取得している。
  Supabase CLIのバージョンによってはこれらの値が`supabase start`のたびに
  動的発行されるため、固定値の決め打ちはCLIの更新でいつか壊れる
- ブランチ運用そのもの（develop/mainの役割）は`docs/git-workflow.md`が正。
  このファイルはその運用を「機械的に強制する手段」という位置づけ

## `.github/workflows/claude-code.yml`

**目的**: Issue/PRのコメントで`@claude`とメンションするだけで、Claude Codeに実装や
調査を依頼できるようにする（PCを開いていなくても、GitHubのモバイルアプリからでも
依頼できる）。

`ci.yml`と同様、このテンプレートリポジトリ内では`claude-code.yml.template`として
無効化した状態で管理し、テンプレート採用時に`.yml`へリネームして有効化する。

設計判断:
- ベースブランチを明示的に指定していない。リポジトリのデフォルトブランチ
  （`develop`）が自動的に使われる前提で、**mainへの直接変更が起きないようにしている**
  （`docs/git-workflow.md`の運用と対応）
- 全PR/全pushで自動起動するのではなく、`@claude`と明示メンションされた時だけ
  起動する。API費用を無駄にかけないため
- `author_association == 'OWNER'`のガードを入れている。これがないとpublic repoで
  第三者のコメントでも`contents: write`権限のワークフローが起動してしまう
  （プロンプトインジェクション・APIコスト濫用のリスク）。個人開発なので
  所有者本人のコメントのみに限定している。共同開発者を増やす場合は
  `COLLABORATOR`等を許可リストに追加する

## `.github/ISSUE_TEMPLATE/`

**目的**: 「すべての変更はIssue起点」という運用（`docs/git-workflow.md`）を、
書きやすいテンプレートで後押しする。

設計判断:
- `bug_report.yml`と`feature_task.yml`の2種類のみ。個人開発でテンプレートの種類を
  増やしても運用コストが見合わないため、最小限に絞っている
- `feature_task.yml`に「Claudeへの直接依頼」チェックボックスがあるのは、
  「これは`@claude`に投げてよい簡単なタスクか」を書き手自身に一度考えさせるため
  （実際にチェックを入れなくても`@claude`とコメントすれば動く。あくまで目印）
- `config.yml`でblank issueを無効化しているのは、テンプレートを経由しない
  Issueが増えると「何を書けばいいかバラバラ」になりやすいため

## `.github/pull_request_template.md`

**目的**: `Closes #`の書き忘れ（Issueが自動クローズされない）と、「developの固定URLで
実際に動作確認したか」の確認漏れを防ぐ。

設計判断: 個人開発なのでレビュー必須のような重いチェックリストにはしていない。
「Issue番号」と「実際に動かして確認したか」の2点だけに絞っている。
