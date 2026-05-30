# 第16章: MCPサーバーを自作する

> Status: 完了
> 発展（探究）コース。MCP を「使う側」から「作る側」へ

## この章で学ぶこと

- MCP サーバーを自作する＝独自のツールを Claude の手足にすること
- 最小サーバーの骨格（Python の FastMCP / TypeScript の SDK）
- 自作 stdio サーバーを `claude mcp add` でつなぐ
- **最重要の落とし穴**: 標準出力はプロトコル専用（ログは標準エラーへ）
- コピペで動く最小例（「2つの数を足す」ツール）

## 前提

- [第7章: MCP・外部連携](../07-mcp/README.md)（使う側）。本章はその「作る側」。
- 簡単な Python か Node.js が動かせること。発展コースの各章は**独立した読み切り**です。

## 本文

### 1. なぜ自作するのか

第7章では、Notion や Slack など**既にある MCP サーバー**をつなぎました。でも、「自分のローカルの処理」や「公開されていない独自ツール」を Claude に使わせたいときは、<mark>**自分で MCP サーバーを作る**</mark>必要があります。

| 状況 | 判断 |
|---|---|
| 既存の MCP がぴったりある | 使う（第7章） |
| 既存がない・独自のローカル処理を使わせたい | **自作する**（本章） |

身近に言えば、第7章が「市販の道具を Claude に持たせる」なら、本章は<mark>**「自分で道具を削り出して Claude に持たせる」**</mark>。一度作れると、できることが一気に広がります。

### 2. 最小サーバーの骨格

いちばん簡単なのは **Python の FastMCP** です。型ヒントと docstring から、ツールの仕様を自動で組み立ててくれます。

```python
# ※ import 文・パッケージ名の正確な書き方は、公式 Python SDK ドキュメントで確認してください
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("MyTools")

@mcp.tool()
def add(a: int, b: int) -> int:
    """2つの数を足す"""
    return a + b

if __name__ == "__main__":
    mcp.run()   # stdio（標準入出力）で起動
```

`@mcp.tool()` を付けた関数が、そのまま Claude のツールになります。TypeScript（`@modelcontextprotocol/sdk`）でも同じことができますが、まずは Python が学びやすいです。

> [!CAUTION]
> パッケージ名・`import` 文・API（`FastMCP` の置き場所など）は版で変わります。本書はコードの**骨格**を示します。<mark>**正確な書き方は公式 SDK ドキュメント（Python / TypeScript）で確認**</mark>してください。`pip install` するパッケージ名も含め、最新を一次情報で。

### 3.【最重要】標準出力を汚さない

ここが、MCP サーバー自作で<mark>**いちばん多く・いちばんハマる落とし穴**</mark>です。

stdio サーバーは、**標準出力（stdout）を Claude との通信専用**に使います。だから、

> [!CAUTION]
> <mark>**サーバーのコードで `print()` してはいけません**</mark>。`print()` は標準出力に文字を出し、それが通信に混ざってプロトコルを壊します（「接続できない」「Malformed message」の典型原因）。デバッグの表示やログは、必ず<mark>**標準エラー（stderr）へ**</mark>出してください。

```python
import sys
# ログは stderr へ（stdout は通信専用なので絶対に使わない）
print("debug message", file=sys.stderr)
```

FastMCP はログを自動で stderr に回してくれますが、**自分で `print()` を足さない**ことが鉄則です。

### 4. Claude Code につなぐ

作ったサーバーは、第7章と同じ `claude mcp add` で登録します。種別は `stdio`（手元でプロセスを起動）です。

```powershell
# Windows：python と server.py を絶対パスで
claude mcp add --transport stdio my-tools -- python "C:\path\to\server.py"
```

つないだら、`/mcp` で接続を確認し、普通に「5 と 3 を足して」と頼めば、自作の `add` ツールが呼ばれます。

> [!CAUTION]
> Windows でのつまずき。①**python が見つからない**: `where python` で確認し、仮想環境なら `.venv\Scripts\python.exe` の**絶対パス**を指定。②**npx 系が起動しない**: `cmd /c "npx -y ..."` のようにラップすると安定。③**起動が遅くてタイムアウト**: `MCP_TIMEOUT`（ミリ秒）を伸ばす手があります（値・名称は公式で確認）。

### 5. 最小の動く例（通し）

1. 上の `add` サーバーを `server.py` として保存（UTF-8・BOM なし、`print()` は入れない）
2. 登録:
   ```powershell
   claude mcp add --transport stdio my-tools -- python "C:\full\path\to\server.py"
   ```
3. Claude Code を起動して `/mcp` で接続確認
4. 「7 と 8 を足して」と頼む → `add` ツールが呼ばれて 15 が返る

これが「自分で作った道具を Claude が使う」最小の成功体験です。

### 6. 作るのを手伝ってもらう

ゼロから書かなくても、**Claude 自身に手伝わせる**手があります。コミュニティには `mcp-builder` のような、MCP サーバーの設計・実装を支援する skill があります（第4章のスキル）。<mark>**「このローカル処理を MCP にして」と頼んで骨組みを作らせ、自分は中身に集中する**</mark>。第0章で見た「頭＋手足」を、道具づくりにも使うわけです。

デバッグには公式の **MCP Inspector**（ローカルでツールを試せる UI）も使えます。

## つまずきポイント

> [!CAUTION]
> MCP 自作でハマりがちな点。

- **`print()` で stdout を汚す**（最頻出）: ログは必ず stderr へ。stdout は通信専用
- **python のパス**: Windows は絶対パス・仮想環境の `python.exe` を明示
- **npx が動かない**: `cmd /c` でラップ
- **起動タイムアウト**: 初回ダウンロードや重い初期化で失敗。タイムアウトを伸ばす
- **パッケージ名・API を推測で書く**: 版で変わる。公式 SDK ドキュメントで確認
- **環境変数が渡らない**: `.mcp.json` の `env` や `claude mcp add --env` で明示

## 公式ドキュメント

- **Model Context Protocol 公式**: <https://modelcontextprotocol.io>（サーバーの作り方・SDK）
- **Claude Code MCP**: <https://code.claude.com/docs/en/mcp>（`claude mcp add`・スコープ）
- 参考: Python SDK / TypeScript SDK の各公式ドキュメント、`modelcontextprotocol/servers`（公式リファレンス実装）
- パッケージ名・API・コマンドは版で変わるため、必ず上記で最新を確認

## 次の章へ

「Claude Code をツールとして使う」の究極形へ。**[第17章: Agent SDK](../17-agent-sdk/README.md)** で、Claude のエージェント機能を自分のプログラムに組み込む方法に進みます。

## 重要な単語まとめ

| 用語 | 一言で言うと | 身近な実例 |
|---|---|---|
| **MCP サーバー自作** | 独自ツールを Claude の手足にする | 自分のローカル処理をツール化 |
| **FastMCP** | Python で MCP を手軽に作る仕組み | `@mcp.tool()` で関数をツール化 |
| **stdio トランスポート** | 標準入出力で通信する方式 | 手元でサーバーを起動 |
| **stdout を汚さない** | ログは stderr へ（最重要の鉄則） | `print()` 厳禁 |
| **mcp-builder** | MCP 作成を手伝う skill | 骨組みを生成させる |

---

◀ [第15章: プラグイン化・マーケットプレイス](../15-plugins/README.md) ｜ [📖 目次](../../README.md) ｜ [第17章: Agent SDK](../17-agent-sdk/README.md) ▶
