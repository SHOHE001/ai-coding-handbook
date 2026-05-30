# 第6章: フック（hooks）

> Status: 完了

## この章で学ぶこと

- フックとは「**特定のイベントが起きたら自動でスクリプトを実行する**」仕組み
- よく使う主要イベント（`PreToolUse` / `PostToolUse` / `UserPromptSubmit` / `Stop` / `SessionStart`）
- `settings.json` の `hooks` への書き方（イベント → matcher → コマンドの3段）
- フックに渡る入力と、終了コードで「続行／ブロック」を返す挙動
- 実用パターン: 保存後に自動整形・危険コマンドの事前ブロック
- **Windows/PowerShell 固有の注意**

## 前提

- [第3章: 権限と settings.json](../03-permissions/README.md)（フックは settings.json に書く）・[第5章: サブエージェント](../05-subagents/README.md)。

## 本文

### 1. フックは「○○したら必ず△△する」の自動化

コマンドやスキルは「**呼んだとき**」に動きました。フックは違います。<mark>**特定のイベントが起きたら、こちらが何も言わなくても自動で動く**</mark>。

たとえば：

- ファイルを編集したら → **自動で整形（フォーマッタ）をかける**
- `rm -rf` のような危険コマンドを実行しようとしたら → **その前に止める**
- セッションを始めたら → **いまの git ブランチや状況を読み込む**

「毎回手でやっていた段取り」を、イベントに紐づけて自動化するのがフックです。第0章の地図で「常駐（黙っていても効く）」に分類したのは、この自動発火の性質ゆえです。

### 2. よく使う主要イベント

フックは「いつ発火するか」をイベントで選びます。まず押さえるべきはこの5つです。

| イベント | いつ発火するか | 代表的な用途 |
|---|---|---|
| **PreToolUse** | ツールを使う**直前** | 危険な操作を止める・確認する |
| **PostToolUse** | ツールを使った**直後** | 編集後の自動整形・テスト実行 |
| **UserPromptSubmit** | あなたがメッセージを送った直後 | 前処理・文脈の注入 |
| **Stop** | Claude が応答を終えたとき | 完了通知・ログ記録 |
| **SessionStart** | セッション開始時 | 現状（ブランチ等）の読み込み |

> [!CAUTION]
> フックのイベントは他にもあり、**版で増減します**。本書は「確実に実在し、よく使う5つ」に絞っています。`PreToolUse` と `PostToolUse` の2つを押さえれば、実用の大半はカバーできます。全イベントは公式の *Hooks* ページで確認してください（出どころ不明の「全◯◯個」という一覧は鵜呑みにしない）。

### 3. 書き方 — 3段のネスト

フックは `settings.json` の `hooks` に、**「イベント → matcher（対象を絞る） → 実行するコマンド」**の3段で書きます。

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "echo hook-fired"
          }
        ]
      }
    ]
  }
}
```

読み解くと：

- **`PostToolUse`**: ツール実行の直後に
- **`matcher": "Edit|Write"`**: Edit か Write（ファイル編集系）のときだけ
- **`type": "command"` / `command`**: このコマンドを実行する

`matcher` は対象ツール名を絞る指定で、`"Edit|Write"` のように `|` で複数指定できます。省略すると全ツールが対象です。

#### 最小の動く例：編集を検知してメッセージを出す

上の JSON をそのまま `settings.json` に入れれば、ファイルを編集するたびに `hook-fired` と表示されます。<mark>**まずこの「発火する」感覚をつかむ**</mark>のが第一歩。動いたら、`command` を本物の処理（整形・テスト）に差し替えます。

### 4. フックに渡る入力と、終了コードの意味

フックのスクリプトには、**標準入力（stdin）に JSON** が渡されます。「どのツールが・どんな入力で呼ばれたか」などが入っています（例: `tool_name`、`tool_input`）。スクリプト側でこれを読めば、中身を見て判断できます。

そして**終了コード**で結果を返します。

| 終了コード | 意味 |
|---|---|
| **0** | 正常。そのまま続行 |
| **2** | **ブロック**。標準エラー出力（stderr）の内容が Claude にフィードバックされる |
| その他 | 非ブロックのエラー（処理は続く） |

<mark>**「危険なら exit 2 で止める」**</mark>。これがフックでガードを作る基本パターンです。

> [!CAUTION]
> 終了コードだけでなく、stdout に JSON を返して細かく制御する高度なやり方もありますが、その形式は版で変わります。まずは<mark>**「0 で続行・2 でブロック」**</mark>のシンプルな形を覚え、応用は公式で確認してください。

### 5. 実用パターン

#### (a) 編集後に自動整形（PostToolUse）

ファイルを編集するたびにフォーマッタをかけ、コードを常にきれいに保ちます。

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "npx prettier --write ."
          }
        ]
      }
    ]
  }
}
```

「整形し忘れ」が構造的に起きなくなります。

#### (b) 危険コマンドを事前ブロック（PreToolUse）

`rm -rf` のような操作を、実行の**直前に止める**ガードです。スクリプトで stdin の JSON を見て、危険なら `exit 2` で止めます（Git Bash 前提の例）。

`<プロジェクト>\.claude\hooks\block-rm.sh`:

```bash
#!/bin/bash
input=$(cat)
cmd=$(echo "$input" | grep -o '"command"[^,]*')

if echo "$cmd" | grep -qE 'rm +-rf'; then
  echo "Blocked: 'rm -rf' is not allowed by hook." >&2
  exit 2
fi
exit 0
```

`settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "bash ./.claude/hooks/block-rm.sh"
          }
        ]
      }
    ]
  }
}
```

> [!CAUTION]
> これは第3章の `deny` と**役割が重なります**。単純な「このコマンド禁止」は `permissions.deny` のほうが簡単で確実。フックの強みは、<mark>**stdin の中身を見て条件分岐できる**</mark>こと（「このディレクトリ配下の rm だけ止める」など）。**まず deny で済むか考え、複雑な判定が要るときフックにする**、が使い分けです。
>
> さらに重要: 上の `grep` 例は**簡易デモであって、完全な防御ではありません**。`rm -fr`・タブや連続スペース・`rm --recursive --force`・`find ... -delete` などで容易にすり抜けます。<mark>**フックを唯一の安全柵にしない**</mark> ── 本気で止めたい操作は `permissions.deny`（第3章）と併用し、フックはあくまで補助と考えてください。

### 6. Windows / PowerShell の注意（重要）

フックは**スクリプトを実行する**ので、Windows ではここが事故りやすいです。しょうへいさんが繰り返しハマってきた点と直結します。

- <mark>**スクリプトは UTF-8（BOM なし）で保存**</mark>。PowerShell の `Out-File`/`Set-Content` は既定が UTF-16。`bash` が BOM を変数と誤解して落ちます。`-Encoding utf8` 必須
- <mark>**実行スクリプトに日本語インラインコメントを入れない**</mark>。コードページの文字化けで関数定義が壊れた実績あり
- **`jq` は Windows に標準で無い**。bash 例で `jq` を使うときは、無ければ `grep`/`sed` で代用するか、PowerShell の `ConvertFrom-Json` を使う
- **どのシェルで実行されるかを意識する**。`command` はシェル経由で動く。Git Bash 前提なら `bash ./script.sh`、PowerShell 前提なら `powershell -File ./script.ps1` のように明示すると安定
- **パス区切り**。`C:\...` のバックスラッシュは bash でエスケープ扱いされる。フォワードスラッシュ（`./.claude/hooks/...`）か、引数渡しで安全に

#### PowerShell でフックを書くなら（最小形）

stdin の JSON は `$input` で受け取れます。

```powershell
$json = $input | Out-String
if ($json -match 'rm\s+-rf') {
  [Console]::Error.WriteLine("Blocked by hook")
  exit 2
}
exit 0
```

`settings.json` 側は `"command": "powershell -NoProfile -File ./.claude/hooks/block-rm.ps1"` のように呼びます。

## つまずきポイント

> [!CAUTION]
> フックでハマりがちな点。

- **エンコーディング**: UTF-8（BOM なし）以外だと bash が落ちる。Windows 最大の罠
- **`exit 2` を返し忘れる**: ブロックしたいのに `exit 0` だと素通り。止めたいなら必ず 2
- **deny で済むことをフックにする**: 単純な禁止は `permissions.deny`。フックは条件分岐が要るときだけ
- **matcher の取り違え**: 編集を捕まえたいのに `Bash` を指定、など。対象ツール名を確認
- **重いフックで遅くなる**: 毎回テスト全実行のような重い処理を `PostToolUse` に置くと、編集のたびに待たされる。軽く保つ
- **イベント名を推測で書く**: 実在しないイベント名は無視されて動かない。確実な5つから始める

## 公式ドキュメント

- **Claude Code 公式ドキュメント**: <https://code.claude.com/docs>
- 関連ページ: *Hooks*（イベント一覧・matcher・入出力・終了コード）/ *Hooks guide*（実用例）
- イベント名・入出力スキーマは版で変わるため、上記で最新を確認

## 次の章へ

次は **[第7章: MCP・外部連携](../07-mcp/README.md)**。ここまでは Claude Code の内側を拡張してきました。次は<mark>**外のサービス（Notion・Slack・Gmail）へ手足を伸ばす**</mark> MCP に入ります。送信系の操作は、第3章の権限とセットで設計します。

## 重要な単語まとめ

| 用語 | 一言で言うと | 身近な実例 |
|---|---|---|
| **フック（hooks）** | イベントが起きたら自動でスクリプト実行 | 保存したら自動整形 |
| **PreToolUse / PostToolUse** | ツール実行の直前／直後に発火 | 直前に危険コマンドを止める／直後に整形 |
| **matcher** | どのツールを対象にするか絞る指定 | `"Edit|Write"` |
| **exit 2** | フックでブロックを返す終了コード | `rm -rf` を止める |
| **stdin の JSON** | フックに渡る「何が呼ばれたか」の情報 | `tool_name`・`tool_input` を見て判定 |
