# 第3章: 権限と settings.json

> Status: 完了

## この章で学ぶこと

- なぜ「自動化を増やす前に、暴走しない安全装置」を先に押さえるのか
- `settings.json` の**3階層**（ユーザー / プロジェクト / ローカル）と優先順位
- `permissions` の **allow / deny / ask** と、マッチ構文（`Bash(...)`・`Read(...)`・`Edit(...)`）
- 自動実行レベル（`defaultMode`）の考え方
- 実例: しょうへいさんの「送信系は事前確認」「CronCreate は allow に入れない」運用を、権限設計として読む
- Windows/PowerShell 固有の注意

## 前提

- [第1章: 土台](../01-basics/README.md)（自動承認モード）・[第2章: 記憶](../02-memory/README.md)（CLAUDE.md との合わせ技）。

## 本文

### 1. なぜ権限を「先に」やるのか

第4章以降で、コマンド・フック・MCP と**自動で動く仕組み**をどんどん増やしていきます。便利な反面、<mark>**自動化が増えるほど「勝手に危険なことをされる」リスク**</mark>も増えます。

だから順序として、**自動化のアクセルを踏む前に、ブレーキ（権限）を理解しておく**。これが実用上いちばん効きます。`settings.json` は、その<mark>**「どこまで確認なしで許すか」を決める安全装置**</mark>です。

### 2. settings.json の3階層

CLAUDE.md と同じく、settings.json も置き場所で効く範囲が変わります。

| 種類 | パス（Windows） | git 共有 | 用途 |
|---|---|---|---|
| **ユーザー** | `C:\Users\zereh\.claude\settings.json` | ✗ | 自分の全プロジェクト共通の設定 |
| **プロジェクト** | `<プロジェクト>\.claude\settings.json` | ✓ | チーム共有したいルール |
| **ローカル** | `<プロジェクト>\.claude\settings.local.json` | ✗（gitignore） | 自分だけの上書き |

**優先順位は、より具体的なものが勝ちます**: ローカル ＞ プロジェクト ＞ ユーザー。個人で1人なら、まず「ユーザー」か「プロジェクト」のどちらかに書けば十分です。

> [!CAUTION]
> `settings.local.json` は**個人用なので git に含めない**のが原則（自動で `.gitignore` 対象になる版が多い）。チーム共有したい設定は `settings.json`、自分だけの実験は `settings.local.json`、と分けます。

### 3. permissions — allow / deny / ask

権限の中心が `permissions` ブロックです。3つのリストで「許す・禁じる・聞く」を指定します。

| キー | 意味 | 使いどころ |
|---|---|---|
| **allow** | 確認なしで実行を許可 | 安全な読み取り・定型コマンド |
| **deny** | 禁止（最優先で効く） | 危険な操作・秘密ファイルの読み取り |
| **ask** | 実行のたびに確認する | 取り返しのつきにくい操作 |

書き方は **`ツール名(引数パターン)`**。たとえばこんな具合です。

```json
{
  "permissions": {
    "allow": [
      "Read(**)",
      "Bash(npm run test:*)",
      "Bash(git status)",
      "Bash(git diff:*)"
    ],
    "ask": [
      "Bash(git push:*)"
    ],
    "deny": [
      "Bash(rm -rf:*)",
      "Read(.env)",
      "Read(**/.env)"
    ]
  }
}
```

読み解くと：

- すべてのファイル読み取りと、`npm run test...`・`git status`・`git diff...` は**確認なしでOK**
- `git push...` は**毎回確認**（取り返しがつきにくいから）
- `rm -rf...` は**禁止**、`.env`（秘密が入りがち）の読み取りも**禁止**

ただし `Read(.env)` の禁止は万能ではありません。読み取りツールを塞いでも、`Bash(cat .env)` のようなシェル経由で中身を読める抜け道が残ります。**いちばん確実なのは「秘密をファイルに置かない」こと**（環境変数や専用の仕組みに逃がす）。deny は補助の壁と考えてください。

> [!CAUTION]
> <mark>**deny は最優先**</mark>です。allow と deny がぶつかったら deny が勝ちます。「基本は広めに allow、でも危険なものは deny で確実に止める」が安全な組み立て方。マッチ構文（`:*` やワイルドカード `**` の細かい挙動）は版で変わるので、最終的には公式の *Settings / Permissions* ページで確認してください。

### 4. defaultMode — 既定の動作モード

第1章で見た「通常 / plan / 自動承認」を、`settings.json` で**既定値として固定**することもできます。

```json
{
  "permissions": {
    "defaultMode": "default"
  }
}
```

- `default` … 普通に確認を求める（安全）
- 編集を自動で通すモードや、計画から入るモードなどを既定にもできる（値の名前は版で変わるため公式確認）

> [!CAUTION]
> 「すべてを確認なしで通す」ような**全許可モード**（いわゆる危険スキップ）は強力ですが、<mark>**事故が即・不可逆になりがち**</mark>。使うとしても `deny` で危険操作を必ず塞いだ上で、信頼できる作業に限定します。

### 5. 実例：しょうへいさんの運用は「権限設計」そのもの

しょうへいさんのグローバル CLAUDE.md には、実は**権限設計の考え方**が詰まっています。settings.json の視点で読み直すと腑に落ちます。

**(a) 送信系は allow に入っていても実行前に確認**

> Slack・Gmail・Calendar・Notion など外部チャネルへ飛ぶ操作は、allow に含まれていても実行前に内容を提示し承認を得る。

これは<mark>**「settings.json の allow（機械的な許可）」と「CLAUDE.md の運用ルール（人の確認）」の二重の安全装置**</mark>です。許可はしてあるが、送信のような外向きの操作は、もう一段「人の目」を通す。allow と ask の中間を、運用ルールで作っている good な例です。

**(b) CronCreate は allow に入れない**

> `CronCreate` は allow に含めない → 都度プロンプト承認とする。理由：リモートで永続実行され、課金が読めないため。

<mark>**「リスクが読めないものは、あえて自動許可しない」**</mark>。一方で `CronList` / `CronDelete`（確認・削除＝安全寄り）は許可済み。**操作の危険度で allow / ask を仕分ける**、まさに権限設計の実践です。

**(c) 読み取り系は許可、書き込み・送信は慎重に**

`slack_read_*` / `slack_search_*` のような読み取りは許可し、`slack_send_*` は確認。<mark>**「読むのは自由、変える・送るのは慎重に」**</mark>という、最も基本的で効く線引きです。

### 6. 最小の動く例：安全な初期 settings.json

プロジェクトの `.claude\settings.json` にこれだけ置けば、「読むのは自由・テストはOK・破壊と秘密は禁止・push は確認」という安全な土台になります。

```json
{
  "permissions": {
    "allow": [
      "Read(**)",
      "Bash(npm run test:*)",
      "Bash(npm run lint)"
    ],
    "ask": [
      "Bash(git push:*)"
    ],
    "deny": [
      "Bash(rm -rf:*)",
      "Read(.env)",
      "Read(**/.env*)"
    ]
  }
}
```

> [!CAUTION]
> settings.json は **JSON** です。末尾カンマやコメント（`//`）でエラーになります。Windows で作るときは **UTF-8（BOM なし）** で保存（`Out-File` は `-Encoding utf8` 必須）。保存後に Claude Code を再起動して読み直させるのが確実です。

## つまずきポイント

> [!CAUTION]
> 権限でハマりがちな点。

- **allow を広げすぎる**: `Bash(*)` のような全許可は危険。具体的なコマンドに絞り、危険なものは `deny` で塞ぐ
- **deny の強さを忘れる**: deny は allow より優先。「許可したのに動かない」ときは deny に引っかかっていないか見る
- **`settings.local.json` を git に入れる**: 個人用設定が共有されてしまう。ローカルは gitignore 対象に
- **JSON 構文ミス**: カンマ・括弧の閉じ忘れで丸ごと読み込まれない。保存後にエラーが出ていないか確認
- **マッチ構文の思い込み**: `Bash(git push)` と `Bash(git push:*)` は別物（後者は引数付きにマッチ）。挙動が怪しいときは公式の構文を確認

## 公式ドキュメント

- **Claude Code 公式ドキュメント**: <https://code.claude.com/docs>
- 関連ページ: *Settings*（settings.json の階層・キー）/ *Permissions / IAM*（allow・deny・ask のマッチ構文・`defaultMode`）
- マッチ構文・モード名は版で変わるため、上記で最新を確認

## 次の章へ

次は **[第4章: スラッシュコマンド & スキル](../04-commands-skills/README.md)**。いよいよ「決まった手順をワンコマンドにする」自動化に入ります。ここから先は自動で動くものが増えるので、本章の**「allow / deny / ask の線引き」**が効いてきます。

## 重要な単語まとめ

| 用語 | 一言で言うと | 身近な実例 |
|---|---|---|
| **settings.json** | 権限などを決める設定ファイル | `.claude\settings.json` |
| **allow / deny / ask** | 許可・禁止・確認の3リスト | テストは allow、push は ask、rm は deny |
| **deny 最優先** | allow と衝突したら deny が勝つ | 「広く許可、危険だけ確実に禁止」 |
| **settings.local.json** | 自分だけの上書き設定（git 非共有） | 個人の実験用 |
| **defaultMode** | 既定の動作モード | default / 自動承認 など |
| **二重の安全装置** | 機械的な allow＋人の確認ルール | 送信系は許可済みでも実行前に提示 |
