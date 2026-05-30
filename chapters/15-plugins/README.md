# 第15章: プラグイン化・マーケットプレイス

> Status: 完了
> 発展（探究）コース。作った道具を束ねて配布・再利用する

## この章で学ぶこと

- プラグインとは「skill / command / agent / hook / MCP を**1つに束ねて配布**する仕組み」
- ディレクトリ構造と `plugin.json`（**最頻出の失敗ポイントに注意**）
- インストール・有効化（`/plugin`）と、ローカルでのテスト
- 自作**マーケットプレイス**（GitHub リポジトリ）で配る
- コピペで動く最小プラグイン

## 前提

- [第4章（スキル）](../04-commands-skills/README.md)・[第5章（サブエージェント）](../05-subagents/README.md)・[第6章（フック）](../06-hooks/README.md)・[第7章（MCP）](../07-mcp/README.md)。これらを束ねる話です。
- 発展コースの各章は**独立した読み切り**です。

## 本文

### 1. なぜプラグイン化するのか

ここまでで、スキル・サブエージェント・フック・MCP を個別に作ってきました。でも、これらは**バラバラのファイル**として `.claude/` に散っています。別のプロジェクトで使い回したり、人に渡したりするには、手でコピーするしかありません。

**プラグイン**は、これらを<mark>**1つのパッケージにまとめて、配布・再利用できるようにする仕組み**</mark>です。MAGI（サブエージェント）・`/save`（スキル）・整形フック・自作 MCP を1つにまとめて、`/plugin install` 一発で他のプロジェクトにも入る ── そういう世界です。

| | 単体（`.claude/`） | プラグイン |
|---|---|---|
| 使い回し | 手でコピー | インストールで自動 |
| 配布 | 不可 | マーケットプレイス経由 |
| 呼び出し名 | `/hello` | `/プラグイン名:hello`（名前空間つき） |

### 2. 実体 — ディレクトリ構造と `plugin.json`

プラグインは、**`plugin.json`（身分証）を持つフォルダ**です。中に各部品を同梱します。

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json      ← 身分証（これだけが .claude-plugin/ の中）
├── skills/
│   └── hello/SKILL.md   ← スキル
├── agents/
│   └── reviewer.md      ← サブエージェント
├── hooks/
│   └── hooks.json       ← フック
└── .mcp.json            ← MCP 設定
```

> [!CAUTION]
> <mark>**いちばん多い失敗**</mark>がこれです。<mark>**`.claude-plugin/` の中に入れるのは `plugin.json` だけ**</mark>。`skills/` や `agents/` などは、`.claude-plugin/` の中ではなく**プラグインのルート直下**に置きます。ここを間違えると、プラグインが認識されません。

`plugin.json` の最小形:

```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "description": "私の道具一式",
  "author": { "name": "SHOHE001" }
}
```

- `name` は必須。これが呼び出しの名前空間になります（`/my-plugin:hello`）
- `version` は省略可（省くと git のコミットが版の代わりになる）

### 3. インストールとテスト

プラグインの管理は `/plugin` コマンド（管理 UI）で行います。

```
/plugin                              # 管理画面を開く（一覧・インストール）
/plugin install 名前@マーケット名      # インストール
/plugin marketplace add owner/repo   # マーケットプレイスを追加
```

自分で開発中のプラグインは、**インストールせずローカルで試せます**。

```powershell
# 開発中のプラグインフォルダを指定して起動
claude --plugin-dir .\my-plugin
```

起動したら、名前空間つきで呼びます。

```
/my-plugin:hello
```

### 4. 最小の動く例

PowerShell で、最小プラグインを作ってみます。

```powershell
# 1. フォルダを作る
mkdir my-plugin\.claude-plugin
mkdir my-plugin\skills\hello
```

`my-plugin\.claude-plugin\plugin.json`（UTF-8・BOM なしで保存）:

```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "description": "My first plugin",
  "author": { "name": "SHOHE001" }
}
```

`my-plugin\skills\hello\SKILL.md`:

```markdown
---
name: hello
description: あいさつする。テスト用。
---

このプロジェクトの README を見て、1〜2文で何のプロジェクトか説明してください。
```

テスト:

```powershell
claude --plugin-dir .\my-plugin
# 起動後： /my-plugin:hello
```

> [!CAUTION]
> Windows での2大つまずき。①**エンコーディング**: `plugin.json` は UTF-8（BOM なし）。PowerShell の `Out-File` は `-Encoding utf8` を付ける。②**MCP を同梱する場合の PATH**: `.mcp.json` で `python` などを起動するなら、`where python` で確認した**絶対パス**を書くと確実です。

### 5. 自作マーケットプレイスで配る

マーケットプレイスは、**プラグインのカタログ**です。GitHub リポジトリに `.claude-plugin/marketplace.json` を置けば、<mark>**自分の配布所になります**</mark>。

```
（利用する人の手元で）
/plugin marketplace add SHOHE001/my-marketplace
/plugin install my-plugin@my-marketplace
```

> [!CAUTION]
> 配布は<mark>**Git ホスト（`owner/repo` 形式）を推奨**</mark>。`marketplace.json` の中で相対パス（`./plugins/...`）を使うと、生 JSON を URL 直指定した場合に解決できないことがあります。GitHub リポジトリ越しなら相対パスが正しく解けます。`marketplace.json` の正確なスキーマは版で変わるので、公式の *Plugin marketplaces* ページで最新を確認してください。

### 6. 段階的に進める

1. **既存プラグインを入れて慣れる**: `/plugin` で公式・コミュニティのプラグインを試し、名前空間（`/名前:skill`）に慣れる
2. **自作スキルをプラグイン化**: 既存の `.claude/skills/` のスキルを、最小の `plugin.json` で包む
3. **部品を足す**: agent・hook・MCP を同梱して、`claude --plugin-dir` でテスト
4. **GitHub で配布**: `marketplace.json` を置き、チームに `/plugin marketplace add` してもらう

## つまずきポイント

> [!CAUTION]
> プラグインでハマりがちな点。

- **`.claude-plugin/` に部品を入れる**: 入れるのは `plugin.json` だけ。skills/agents 等はルート直下（最頻出）
- **エンコーディング**: `plugin.json` などは UTF-8（BOM なし）
- **MCP 同梱時の PATH**: `python`/`node` は絶対パスで。`/plugin` の Errors タブで「実行ファイルが見つからない」を確認できる
- **配布時の相対パス**: 生 JSON の URL 直指定だと相対パスが解けない。Git ホスト推奨
- **変更が反映されない**: セッション中に直したら再読み込み（`/reload-plugins` 等、版による）
- **秘密情報の同梱**: プラグインに API キーやトークンを含めない（配布するとそのまま流出する）。秘密は利用者側の環境変数・設定で渡す

## 公式ドキュメント

- **Claude Code 公式ドキュメント**: <https://code.claude.com/docs>
- 関連ページ: *Plugins*（作成）/ *Plugins reference*（`plugin.json` の項目）/ *Plugin marketplaces*（配布）/ *Discover plugins*（インストール）
- 参考リポジトリ: `anthropics/claude-code` の `plugins/`（デモ集）。マニフェストの項目・コマンドは版で変わるため公式で確認

## 次の章へ

外部サービスを「使う」だけでなく「作る」側へ。**[第16章: MCPサーバーを自作する](../16-mcp-server/README.md)** で、自分のツールを Claude の手足にする方法に進みます。

## 重要な単語まとめ

| 用語 | 一言で言うと | 身近な実例 |
|---|---|---|
| **プラグイン** | skill/agent/hook/MCP を束ねたパッケージ | MAGI＋/save＋フックを1つに |
| **plugin.json** | プラグインの身分証（`.claude-plugin/` 内） | name・version・description |
| **名前空間** | `/プラグイン名:skill` の呼び出し形 | `/my-plugin:hello` |
| **マーケットプレイス** | プラグインのカタログ（GitHub） | `/plugin marketplace add owner/repo` |
| **`--plugin-dir`** | 開発中プラグインをローカルで試す | `claude --plugin-dir .\my-plugin` |

---

◀ [第9章: 自動化](../09-automation/README.md) ｜ [📖 目次](../../README.md) ｜ [第16章: MCPサーバーを自作する](../16-mcp-server/README.md) ▶
