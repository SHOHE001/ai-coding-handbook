# 第17章: Agent SDK — Claude を自分のプログラムに組み込む

> Status: 完了
> 発展（探究）コース。Claude のエージェント機能を、ライブラリとしてコードから使う

## この章で学ぶこと

- Agent SDK とは「Claude Code を**ライブラリ**として使う」もの
- CLI / ヘッドレス（`claude -p`）/ SDK の違いと使い分け
- パッケージと最小例（Python / TypeScript）
- コードから制御できること（ツール・権限・サブエージェント・MCP）
- API キー・コスト・Windows の注意

## 前提

- [第9章: 自動化（ヘッドレス実行）](../09-automation/README.md)。`claude -p` を一度触っていると違いが分かりやすい。
- 簡単な Python か Node.js が書けること。発展コースの各章は**独立した読み切り**です。

## 本文

### 1. CLI・ヘッドレス・SDK の違い

ここまで Claude Code を「ターミナルで対話」「`claude -p` で1回叩く」と使ってきました。**Agent SDK** は、その先です。<mark>**Claude Code の頭脳（エージェントの考えて動く仕組み）を、自分のプログラムの中から呼ぶ**</mark>。

| 使い方 | どう動かす | 向いている場面 |
|---|---|---|
| **CLI**（対話） | ターミナルで会話 | 日常の開発・探索 |
| **ヘッドレス**（`claude -p`） | コマンドで1回実行 | CI・スクリプトの部品（第9章） |
| **Agent SDK** | **コードから呼ぶ** | 自分のアプリ・本番システムに組み込む |

<mark>**「自分のアプリに Claude のエージェントを埋め込みたい」ときが SDK の出番**</mark>です。たとえば、しょうへいさんの Discord Bot に「コードを読んで答える機能」を足す、といった統合がコードで書けます。

### 2. パッケージと最小例

| 言語 | パッケージ | 必要環境 |
|---|---|---|
| **Python** | `claude-agent-sdk`（`pip install claude-agent-sdk`） | Python 3.10+ |
| **TypeScript** | `@anthropic-ai/claude-agent-sdk`（`npm install ...`） | Node.js 18+ |

Python の最小例（1回問い合わせて結果を受け取る）:

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions

async def main():
    async for message in query(
        prompt="このフォルダの構成を3行で要約して",
        options=ClaudeAgentOptions(allowed_tools=["Read", "Bash"]),
    ):
        if hasattr(message, "result"):
            print(message.result)

asyncio.run(main())
```

TypeScript もほぼ同じ形で `query({ prompt, options })` を呼びます。

> [!CAUTION]
> パッケージ名・関数名・オプション名（`query`・`ClaudeAgentOptions` など）は版で変わります。本書は骨格を示すので、<mark>**正確な API は公式 Agent SDK ドキュメントで確認**</mark>してください。

### 3. API キーの渡し方

SDK は API を直接使うので、**API キー**が必要です。第7章の秘密情報の扱いがそのまま効きます。

```powershell
# 環境変数で渡す（コードに直書きしない）
$env:ANTHROPIC_API_KEY = "sk-ant-..."
```

> [!CAUTION]
> <mark>**API キーをコードに直書きして git に上げない**</mark>。環境変数か `.env`（`.gitignore` 対象）で渡す。キーの流出は、そのまま課金リスクです（第3章・第7章）。

### 4. コードから制御できること

SDK の強みは、対話ではできない<mark>**「細かい制御をコードで書ける」**</mark>ことです。ここまで学んだ拡張点が、ほぼコードから操れます。

- **ツール**: `allowed_tools` で使えるツールを限定（第3章の権限のコード版）
- **権限モード**: 自動承認の度合いをコードで指定
- **サブエージェント**: 専門エージェントを定義して委譲（第5・10章）
- **MCP 接続**: 外部サービスをコードからつなぐ（第7章）
- **フック**: ライフサイクルにコードを挟む（第6章）
- **セッション**: 過去の文脈を復元して継続

つまり、<mark>**これまで「ファイルで設定してきたこと」を、SDK では「コードで動的に組み立てられる」**</mark>。本番アプリに組み込むなら、この柔軟さが効いてきます。

### 5. コストの注意

SDK は便利ですが、コストの考え方が対話とは別になり得ます。

> [!CAUTION]
> Agent SDK や `claude -p` の使用は、<mark>**対話の利用枠とは別のコスト体系になる場合があります**</mark>（料金体系は時期・プランで変わります）。本番でループを回す前に、**最新の料金・利用枠を公式で確認**し、第14章のコスト戦略（モデル使い分け・キャッシュ・上限設定）を併用してください。

### 6. 段階的に進める

1. **セットアップ**: SDK を入れ、API キーを環境変数に設定、「このフォルダに何がある？」で動作確認
2. **ツールと権限**: `allowed_tools` や権限モードを変えて挙動を体験
3. **セッション継続**: 前の文脈を復元して多段の処理を書く
4. **MCP・サブエージェント**: 外部連携や役割分担をコードから組む

## つまずきポイント

> [!CAUTION]
> Agent SDK でハマりがちな点。

- **API キー未設定**: `ANTHROPIC_API_KEY` がないと動かない。環境変数を確認
- **キーの直書き**: コードに書かない。流出＝課金リスク
- **コスト体系の誤認**: 対話の枠とは別になり得る。事前に公式で確認
- **Windows の実行ポリシー**: 仮想環境の `Activate.ps1` でエラーが出たら `Set-ExecutionPolicy -Scope Process RemoteSigned`
- **API を推測で書く**: パッケージ名・関数名は版で変わる。公式ドキュメントで確認

## 公式ドキュメント

- **Claude Code 公式ドキュメント**: <https://code.claude.com/docs>
- 関連ページ: *Agent SDK overview* / *Agent SDK quickstart* / *Configure permissions*（SDK の権限）
- パッケージ情報: PyPI `claude-agent-sdk` / npm `@anthropic-ai/claude-agent-sdk`
- パッケージ名・API・料金は版で変わるため、必ず上記で最新を確認

## 次の章へ

最後は楽しい仕上げ。**[第18章: 演出とカスタマイズ](../18-customization/README.md)** で、見た目・出力スタイルを自分好みにして、使い続けたくなる環境を作ります。

## 重要な単語まとめ

| 用語 | 一言で言うと | 身近な実例 |
|---|---|---|
| **Agent SDK** | Claude Code をライブラリとしてコードから使う | 自分のアプリに組み込む |
| **CLI / ヘッドレス / SDK** | 対話 / 1回実行 / コードから制御 | 用途で使い分ける |
| **`query()`** | SDK で問い合わせる基本の呼び出し | プロンプトとオプションを渡す |
| **コードで制御** | ツール・権限・MCP をコードで組む | 設定ファイルの動的版 |
| **別のコスト体系** | SDK は対話と料金枠が別になり得る | 事前に公式で確認 |
