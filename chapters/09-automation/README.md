# 第9章: 自動化 — ヘッドレス実行・CI・スケジュール

> Status: 完了
> 発展（探究）コース。Claude Code を「対話で使う」から「対話なしで回す・仕組みに組み込む」へ

## この章で学ぶこと

- 対話なしで Claude Code を動かす **ヘッドレス実行**（`claude -p`）
- 出力を JSON で受けて、**スクリプトの部品**として使う
- 会話の続きを自動でつなぐ（`--continue` / `--resume`）
- **対話がない＝確認プロンプトが出せない**環境での権限と安全（最重要）
- **GitHub Actions** に組み込んで PR を自動レビュー
- **スケジュール実行**（`/schedule`・OS のタスクスケジューラ・`/loop`）で定期自動化

## 前提

- 基礎コース（第0〜8章）。特に [第1章（モード）](../01-basics/README.md)・[第3章（権限）](../03-permissions/README.md)・[第5章（サブエージェント）](../05-subagents/README.md)・[第6章（フック）](../06-hooks/README.md)。
- 自動化は<mark>**「権限を絞る」のが生命線**</mark>。第3章が頭に入っていることが必須です。

## 本文

### 1. 自動化の4段階

自動化は、いきなり CI に飛ばず、段階で登るのが安全です。

| 段階 | やること | この章の節 |
|---|---|---|
| ① 対話で使う | 普通に `claude` で会話（基礎コース） | — |
| ② `-p` で1回叩く | 対話せず、1コマンドで結果を得る | 第2節 |
| ③ スクリプトに組む | 出力を受けて、自分の処理の部品にする | 第3〜5節 |
| ④ CI・スケジュールに乗せる | PR で自動実行・定期実行する | 第6〜7節 |

<mark>**下の段ほど「人が見ていない」状態で動く**</mark>。だから上に行くほど、第3章の権限設計が効いてきます。

### 2. ヘッドレス実行 — `claude -p`

`-p`（`--print`）を付けると、Claude Code は**対話に入らず、1回答えて終了**します。プロンプトは引数か標準入力で渡します。

```powershell
# 引数で渡す
claude -p "このプロジェクトの構成を3行で要約して"

# 標準入力（パイプ）で渡す
Get-Content .\build-error.txt | claude -p "このエラーの原因は？"
```

これが自動化の最小単位です。**「Claude を関数のように1回呼ぶ」**イメージ。

#### 出力フォーマット（`--output-format`）

スクリプトで使うなら、結果を機械が読める形でもらえます。

| 値 | 中身 | 用途 |
|---|---|---|
| `text`（既定） | 平文テキスト | そのまま表示・ログ |
| `json` | `result` や使用量などを含む JSON | スクリプト処理・コスト記録 |
| `stream-json` | 逐次の JSON | 長いタスクの進捗表示 |

```powershell
# JSON で受けて、回答とコストを取り出す
$out = claude -p "src を要約して" --output-format json | ConvertFrom-Json
Write-Host $out.result
Write-Host "コスト: $($out.total_cost_usd) USD"
```

なお、フラグ名・出力 JSON の項目・標準入力のサイズ上限などは**版で変わります**。`total_cost_usd` のような項目名や `stream-json` の詳細形式は、公式の *CLI reference* / *Headless* ページで確認してください。本書は「`-p` で叩き、`--output-format json` で受ける」という確実な骨格を主軸にします。

### 3. スクリプトの部品にする

ヘッドレス実行の本領は、**他のコマンドとつなぐ**ことです。第5章のサブエージェント（読み取り専用の調査役）を、今度はシェルから呼ぶ感覚です。

```powershell
# git の差分をレビューさせ、結果をファイルに保存
git diff main | claude -p "セキュリティと可読性の観点でレビューして" `
  --output-format json | `
  ConvertFrom-Json | Select-Object -ExpandProperty result | `
  Set-Content -Encoding utf8 review.txt
```

```bash
# Git Bash なら jq で受ける
git diff origin/main | claude -p "脆弱性をチェック" --output-format json \
  | jq -r '.result' > review.txt
```

**「人がコピペして Claude に貼る」を、コマンド1本に畳む**。これが部品化です。

### 4. 会話の続きを自動でつなぐ

1回で終わらず、前の文脈を引き継ぎたいときは継続フラグを使います。

| フラグ | 何をする |
|---|---|
| `--continue` / `-c` | そのフォルダの**最新セッション**を続行 |
| `--resume <session-id>` | **特定のセッション**を ID 指定で再開 |

```powershell
# 1回目：セッションIDを控える
$first = claude -p "コードレビューを開始して" --output-format json | ConvertFrom-Json
$sid = $first.session_id

# 2回目：同じ流れで続ける
claude -p "次は性能面を見て" --resume $sid
```

### 5.【最重要】対話なし＝確認が出せない、の権限設計

ここが自動化のいちばんの落とし穴です。対話なら「これ実行していい？」と聞けますが、<mark>**ヘッドレスでは聞く相手がいません**</mark>。だから「何を確認なしで許すか」を**起動時に決めておく**必要があります。

第3章の `allow` / `deny` が、ヘッドレスでは**フラグ**として効きます。

```powershell
# 使えるツールを明示して、その範囲だけ自動実行
claude -p "lint エラーを直して" `
  --allowedTools "Read,Edit,Bash(git *)" `
  --permission-mode acceptEdits
```

- `--allowedTools` … 確認なしで使ってよいツールを列挙（第3章の `allow` のコマンド版）
- `--permission-mode` … 既定の動作モード（`default` / `acceptEdits` / `plan` など）

> [!CAUTION]
> <mark>**`--dangerously-skip-permissions`（全確認スキップ）を、普段の Windows 環境で使わないこと**</mark>。名前のとおり危険で、`rm -rf` のような破壊的操作も**止める人がいないまま実行**されます。使うのは**使い捨てのコンテナ／隔離環境の中だけ**。普段の自動化は、`--allowedTools` で**必要なツールだけ許可**するのが正解です。第6章で学んだ「フックを唯一の安全柵にしない」と同じ発想で、<mark>**自動実行ほど「できることを最小限に絞る」**</mark>。

なお、権限モードには新しい値（自動判定する `auto` など）も登場していますが、研究プレビュー段階や版依存のものは挙動が変わり得ます。安全側の基本は「`--allowedTools` で許可を絞る」。新しいモードを使うときは公式の *Permission modes* ページで現在の意味を確認してください。

しょうへいさんの「送信系は事前確認」運用は、自動化では特に効きます。<mark>**人が見ていない自動実行に、外へ通知が飛ぶ操作（Slack 送信・予定作成）を混ぜない**</mark>。読み取り・整形・レビューのような「取り返しがつく」操作に留めるのが、安全な自動化の鉄則です。

### 6. GitHub Actions に組み込む

PR が来たら自動でレビューさせる、といった CI 連携には**公式アクション** `anthropics/claude-code-action` を使います。

最小の形は「PR や Issue で **`@claude` とメンションすると動く**」設定です。

```yaml
# .github/workflows/claude.yml
name: Claude Code
on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
jobs:
  claude:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
```

PR を開いたら毎回レビューさせる「自動モード」もできます。

```yaml
# PR ごとに /code-review を走らせる例
on:
  pull_request:
    types: [opened, synchronize]
jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          prompt: "/code-review"
          claude_args: "--max-turns 5 --model claude-sonnet-4-6"
```

ポイント:

- **`ANTHROPIC_API_KEY` を GitHub のシークレット**に登録する（コードに直書きしない＝第7章の秘密情報の扱い）
- **コストを抑える**: `--max-turns` で往復回数を制限、`--model` で軽いモデルを指定（上の例の `claude-sonnet-4-6` などモデル名は版で変わるので公式で確認）。`--output-format json` の `total_cost_usd` をログに残すと使用料を追える
- セットアップは Claude Code 内の `/install-github-app` で進めると楽

なお、公式アクションは版（`@v1` など）で引数名や挙動が変わってきた経緯があります。`prompt` や `claude_args` の名前・YAML の書き方は、必ずリポジトリ `anthropics/claude-code-action` の README と公式 *GitHub Actions* ページで最新を確認してください。本書は「公式アクションを使い、API キーはシークレットに置く」という骨格だけ示します。

### 7. スケジュールで定期実行

「毎朝 PR をレビュー」「毎週、依存関係をチェック」のような定期自動化には、いくつか道があります。

| 方法 | どこで動く | 向いている場面 |
|---|---|---|
| **`/schedule`（ルーチン）** | クラウド側 | PC を閉じても動かしたい定期実行 |
| **OS のタスクスケジューラ ＋ `claude -p`** | 自分の PC | 手元の環境・ファイルで動かしたい |
| **`/loop`** | 起動中のセッション内 | その場で「N分ごとに確認」したいとき |

しょうへいさんが既に使っている **`/schedule`（リモートのルーチン）** が、まさにこのクラウド定期実行です。OS 側でやるなら、Windows のタスクスケジューラで `claude -p "..."` を定時起動する形になります。

```powershell
# /loop の例：5分ごとにテスト状況を確認（セッションを開いている間だけ動く）
claude /loop "5m" "テストが通るか確認して、失敗があれば要約して"
```

> [!CAUTION]
> 定期実行は<mark>**「気づかないうちに回数とコストが積み上がる」**</mark>のが最大の注意点。しょうへいさんが CLAUDE.md で「`CronCreate` は課金が読めないので自動許可しない」と決めているのは、まさにこれ。定期自動化を組むときは、**実行頻度・権限・コスト上限**を先に決めること。ルーチン（`/schedule`）の細かい仕様（トリガー種別・API 連携など）は研究プレビュー段階で変わり得るので、公式の *Routines* ページで確認してください。

## つまずきポイント

> [!CAUTION]
> 自動化でハマりがちな点。Windows 環境は特に注意。

- **API キーの環境変数**: ヘッドレスや CI では `ANTHROPIC_API_KEY` が要ることがある。Windows では `[Environment]::SetEnvironmentVariable("ANTHROPIC_API_KEY", $key, "User")` で永続設定。**キーをスクリプトに直書きして git に上げない**
- **出力のエンコーディング**: 結果をファイルに保存するとき、PowerShell の `>` や `Out-File` は既定が UTF-16。`Set-Content -Encoding utf8` で UTF-8（BOM なし）に
- **CI で権限プロンプト待ち→タイムアウト**: 対話モードのまま CI で動かすと、確認待ちで止まり、やがて kill される。CI では必ず `--allowedTools` 等で**事前に許可を決める**
- **コストの暴走**: 自動・定期実行は回数が見えにくい。`--max-turns` で上限、`--output-format json` の `total_cost_usd` でコスト監視
- **危険フラグの常用**: `--dangerously-skip-permissions` を普段使いにしない。隔離環境限定
- **巨大入力**: 巨大なログを丸ごとパイプすると失敗することがある。必要部分に絞るか、ファイルを読ませる形に

## 公式ドキュメント

- **Claude Code 公式ドキュメント**: <https://code.claude.com/docs>
- 関連ページ: *Headless*（`-p`・`--output-format`）/ *CLI reference*（全フラグ）/ *Permission modes*（`--permission-mode`）/ *GitHub Actions*（`claude-code-action`）/ *Routines*（スケジュール・定期実行）
- フラグ名・アクションの引数・ルーチン仕様は版で変わる（研究プレビューを含む）ため、必ず上記で最新を確認

## 次の章へ

この章は**発展コースの探究編**（作る側・自動化を極める）の入口です。自動化は、基礎コースで学んだ拡張点（権限・サブエージェント・フック）を「人が見ていない場所」で安全に動かす応用でした。

探究編は続きます ── **[第15章: プラグイン化・マーケットプレイス](../15-plugins/README.md)**・**[第16章: MCPサーバーを自作する](../16-mcp-server/README.md)**・**[第17章: Agent SDK](../17-agent-sdk/README.md)**・**[第18章: 演出とカスタマイズ](../18-customization/README.md)**。応用編（[第10章](../10-multi-agent/README.md)〜[第14章](../14-context-cost/README.md)）と合わせ、発展コースは興味のある章からどうぞ。まずは**いちばん不便を感じている繰り返しを、`claude -p` 1本で叩くところから**始めてみてください。

## 重要な単語まとめ

| 用語 | 一言で言うと | 身近な実例 |
|---|---|---|
| **ヘッドレス実行（`claude -p`）** | 対話せず1回で結果を返す呼び方 | `git diff \| claude -p "レビューして"` |
| **`--output-format json`** | 結果を機械が読める JSON で受ける | コストや回答をスクリプトで処理 |
| **`--continue` / `--resume`** | 前のセッションを自動で続ける | 多段の自動レビュー |
| **`--allowedTools`** | 確認なしで使えるツールを絞る（権限の自動版） | 第3章の allow のコマンド版 |
| **`--dangerously-skip-permissions`** | 全確認をスキップ（危険・隔離環境限定） | 普段使いは厳禁 |
| **GitHub Actions 連携** | PR/Issue で Claude を自動実行 | `@claude` メンションで自動レビュー |
| **`/schedule` / `/loop`** | 定期実行・繰り返し実行 | 毎朝の PR レビュー |
