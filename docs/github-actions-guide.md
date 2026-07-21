# GitHub Actions yamlファイル解説資料

## 目的

このドキュメントは、本リポジトリの `.github/workflows/` に配置されている GitHub Actions の yaml ファイルについて、**初学者向けに**仕組みと記述内容を解説するものです。

対象ファイル：

- `.github/workflows/claude.yml`
- `.github/workflows/claude-code-review.yml`

## 1. GitHub Actionsとは

GitHub Actions は、GitHub 上で「何かが起きたら（イベント）」「自動で処理を実行する（ジョブ）」ための仕組みです。
たとえば以下のようなことができます。

- Issue や PR にコメントが付いたら自動で反応する
- PR が作成されたら自動でコードレビューする
- コードをpushしたら自動でテストを実行する

設定は `.github/workflows/` フォルダの中に yaml ファイル（`.yml` または `.yaml`）として置くだけで、GitHub が自動的に読み込んで実行してくれます。

## 2. YAMLファイルの基本

YAML（ヤムル）は設定を書くためのファイル形式です。JSONに似ていますが、より人間が読み書きしやすいように作られています。

### 基本ルール

- インデント（字下げ）で階層構造を表す（半角スペースを使う。タブは使えない）
- `キー: 値` の形式でデータを書く
- `-` はリスト（配列）の要素を表す
- `#` から始まる行はコメント（実行に影響しない説明文）

```yaml
name: サンプル       # キー: 値
on:                  # onの下に階層がある
  push:              # さらにネストしている
    branches:
      - main         # リストの要素
      - develop
```

## 3. GitHub Actionsの基本用語

workflowファイルを読むうえで、まず覚えておきたい用語です。

| 用語 | 説明 |
| --- | --- |
| **Workflow（ワークフロー）** | yamlファイル1つ全体のこと。「いつ」「何を」実行するかをまとめた自動化の単位 |
| **Event（イベント）** | ワークフローが起動するきっかけ。「Issueが作られた」「PRにコメントが付いた」など |
| **Job（ジョブ）** | ワークフローの中で実行される処理のまとまり。1つのワークフローに複数のジョブを置ける |
| **Step（ステップ）** | ジョブの中の1つ1つの作業単位。上から順番に実行される |
| **Runner（ランナー）** | 実際に処理を実行するマシン（サーバー）。`ubuntu-latest` はUbuntu OSの仮想マシンを使うという意味 |
| **Action（アクション）** | 再利用可能な処理の部品。`uses:` で呼び出す（例：チェックアウト処理をまとめた `actions/checkout`） |
| **Secrets（シークレット）** | APIキーなど、外部に漏らしたくない情報。GitHubのリポジトリ設定に登録しておき、`${{ secrets.XXX }}` の形式で読み出す |

## 4. `claude.yml` の解説

このワークフローは、Issue や PR のコメントに `@claude` と書かれたときに Claude が自動で反応するためのものです。

```yaml
name: Claude Code
```
- ワークフローの名前。GitHubの「Actions」タブに表示される名前です。

```yaml
on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  issues:
    types: [opened, assigned]
  pull_request_review:
    types: [submitted]
```
- `on:` は「どんなイベントが起きたときにこのワークフローを実行するか」を指定します。
- `issue_comment` / `pull_request_review_comment`：Issueやレビューコメントが**新規作成（created）**されたとき
- `issues`：Issueが**新規作成（opened）**または**担当者にアサイン（assigned）**されたとき
- `pull_request_review`：PRのレビューが**送信（submitted）**されたとき
- つまり「Issue・PR関連で何かコメントやレビューがあったら、まずこのワークフローが起動する」という設定です。

```yaml
jobs:
  claude:
    if: |
      (github.event_name == 'issue_comment' && contains(github.event.comment.body, '@claude')) ||
      ...
```
- `jobs:` の下に、実行するジョブ（今回は `claude` という名前の1つだけ）を定義します。
- `if:` は「本当にこのジョブを実行してよいか」の条件式です。
- `contains(github.event.comment.body, '@claude')` は「コメント本文に `@claude` という文字列が含まれているか」をチェックしています。
- つまり、イベントが起きても **コメントの中に `@claude` が含まれていない場合はジョブが実行されません**。これにより、無関係なコメントには反応しないようにしています。

```yaml
    runs-on: ubuntu-latest
```
- このジョブをどんな環境（マシン）で実行するかを指定します。`ubuntu-latest` は最新のUbuntu Linux環境です。

```yaml
    permissions:
      contents: read
      pull-requests: read
      issues: read
      id-token: write
      actions: read # Required for Claude to read CI results on PRs
```
- ワークフローがGitHub上で何をしてよいかの権限を指定します。
- `contents: read`：リポジトリのファイルを読み取ってよい
- `pull-requests: read` / `issues: read`：PRやIssueの情報を読み取ってよい
- `id-token: write`：認証用のトークンを発行してよい（Claude Code Actionの認証に必要）
- `actions: read`：CI（他のワークフロー）の実行結果を読み取ってよい
- 必要最小限の権限だけを与えるのがセキュリティ上のベストプラクティスです。

```yaml
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 1
```
- `steps:` の下に、実行する作業を順番に並べます。
- 1つ目のステップは「リポジトリのコードをランナー上にダウンロード（チェックアウト）する」処理です。
- `uses: actions/checkout@v4`：GitHub公式が提供している再利用可能なアクション（`actions/checkout` のバージョン4）を使うという意味です。
- `with:` はそのアクションに渡す引数（パラメータ）です。`fetch-depth: 1` は「直近のコミット1つ分だけ取得する（履歴全部はダウンロードしない）」という指定で、実行を高速化します。

```yaml
      - name: Run Claude Code
        id: claude
        uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}

          additional_permissions: |
            actions: read
```
- 2つ目のステップで、実際にClaude Codeを動かします。
- `uses: anthropics/claude-code-action@v1`：Anthropic社が提供しているClaude Code用のアクションを使います。
- `id: claude`：このステップに `claude` という名前を付けています（後続のステップから結果を参照する際に使えます）。
- `claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}`：認証トークンを、リポジトリに事前登録された **Secrets**（暗号化された秘密情報）から読み込んでいます。yamlファイルに直接パスワードやトークンを書かないのが鉄則です。
- `additional_permissions:` はClaude Codeに追加で与える権限の指定です。

```yaml
          # prompt: 'Update the pull request description to include a summary of changes.'
          # claude_args: '--allowed-tools Bash(gh pr *)'
```
- `#` から始まる行はすべてコメント（説明用のメモ）で、実行には影響しません。
- コメントアウトされているのは「使いたい場合はコメントを外して設定してください」というオプション項目の例です。

## 5. `claude-code-review.yml` の解説

こちらは、Pull Requestが作成・更新されたときに、Claudeが自動でコードレビューを行うためのワークフローです。

```yaml
name: Claude Code Review

on:
  pull_request:
    types: [opened, synchronize, ready_for_review, reopened]
```
- `pull_request` イベントをトリガーにしています。
- `opened`：PRが新規作成されたとき
- `synchronize`：PRに新しいコミットがpushされたとき
- `ready_for_review`：ドラフトPRが「レビュー可能」状態に変更されたとき
- `reopened`：閉じたPRが再オープンされたとき
- これらのタイミングで自動的にレビューが走ります。

```yaml
jobs:
  claude-review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: read
      issues: read
      id-token: write
```
- `claude.yml` と同様に、実行環境と必要な権限を指定しています。

```yaml
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 1

      - name: Run Claude Code Review
        id: claude-review
        uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          plugin_marketplaces: 'https://github.com/anthropics/claude-code.git'
          plugins: 'code-review@claude-code-plugins'
          prompt: '/code-review:code-review ${{ github.repository }}/pull/${{ github.event.pull_request.number }}'
```
- 1つ目のステップはこちらもリポジトリのチェックアウトです。
- 2つ目のステップでClaude Codeを実行しますが、今回は `prompt:` に `/code-review:code-review ...` という**スラッシュコマンド**を指定しています。これは「このPRに対してコードレビューを実行して」という指示をあらかじめ固定で与えている、という意味です。
- `plugin_marketplaces` / `plugins`：Claude Codeの「コードレビュー用プラグイン」を読み込むための設定です。
- `${{ github.repository }}` や `${{ github.event.pull_request.number }}` のような `${{ }}` の書き方は、GitHub Actionsが自動的に用意している**変数（コンテキスト）**を埋め込む記法です。実行時に「対象のリポジトリ名」や「対象のPR番号」に自動で置き換わります。

## 6. 覚えておくと便利な記法まとめ

| 記法 | 意味 |
| --- | --- |
| `on:` | ワークフローの起動条件（イベント） |
| `jobs:` | 実行するジョブの一覧 |
| `runs-on:` | ジョブを実行する環境 |
| `steps:` | ジョブ内の作業手順 |
| `uses:` | 既存のアクション（再利用可能な処理）を呼び出す |
| `with:` | `uses:` で呼び出したアクションへの引数 |
| `run:` | シェルコマンドを直接実行する（今回のファイルでは未使用） |
| `if:` | 条件を満たすときだけ実行する |
| `${{ ... }}` | 式（変数やSecrets、関数呼び出しなど）を埋め込む記法 |
| `secrets.XXX` | リポジトリに登録した秘密情報を参照する |
| `#` | コメント行 |

## 7. まとめ

- `claude.yml` は「Issue/PRのコメントに `@claude` が含まれていたら起動し、Claudeが応答する」ワークフローです。
- `claude-code-review.yml` は「PRが作成・更新されたら自動でClaudeがコードレビューを行う」ワークフローです。
- どちらも `actions/checkout` でコードを取得したあと、`anthropics/claude-code-action` を使ってClaude Codeを実行する、という共通の流れになっています。
- `on:` でイベントを、`if:` で追加条件を、`permissions:` で権限を絞り込むことで、意図しないタイミングでの実行や、過剰な権限付与を防いでいます。
