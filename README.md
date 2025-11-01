自分:

以下の記事でMCPについて詳しくなってください。

https://modelcontextprotocol.io/specification/2025-06-18/server/tools

https://modelcontextprotocol.io/specification/2025-06-18/server/resources

--------------------------------------------------------------------------------------------------

Gemini:
[回答は省略]

------------------------------------------------------------------------------------------------------------------------

自分:

MCPサーバーを自作することが私達の共通目標です。

私達のMCPサーバーを通して「できるようにしたいこと」は「MCPクライアント(LLM)が以下のような会話形式で返答できるようにする」ことです。

[中略]

さて、これから私達のMCPサーバーの設計やそれを実装するコードを具体的に考えていきましょう。 考えがまとまり次第、

1.大まかな設計の説明

2.実装にあたってのADR(Architecture Decision Record)

3.開発していくのに必要になる手順の説明

4.実装に向けた各ファイルごとの実際のコード(省略は一切しないように心掛けてください)

5.テストするときの手順の説明

の5つを提示してください。ゆっくりと段階的に考えていきましょうね。

※全文は[prompt.md]に

--------------------------------------------------------------------------------------------------------------------------------------------------

Gemini:

承知いたしました。これは非常にエキサイティングなプロジェクトです。MCP（Model Context Protocol）の「Tools」機能と「Resources」機能を組み合わせて、ローカルファイルシステム（和歌集データ）にアクセスし、指定された形式でクライアント（LLM）に応答を返すMCPサーバーをTypeScriptで構築していきましょう。

私達の共通目標であるMCPサーバーの実装に向けて、設計と具体的なコードを段階的に提示します。

-----

### 1\. 大まかな設計の説明

このMCPサーバーは、Node.jsの標準HTTPサーバーをベースに構築します。クライアント（Claude Desktop）からのJSON-RPCリクエストをPOSTで受け付けます。

**中心となる機能:**

1.  **`Tools` (ツール) 機能**:

      * クライアントが「和歌集のランキング情報を取得する」というタスクを実行できるよう、`wakashu/getRanking` という名前のツールを公開します。
      * このツールは、`wakashuName`（和歌集名、今回は使いませんが将来の拡張用）、`edition`（第何回）、`rankStart`（開始順位）、`rankEnd`（終了順位）をパラメータとして受け取ります。
      * `tools/call` でこのツールが呼び出されると、サーバーは指定されたルートディレクトリ（`/home/info-gather/wakashu_mcp_dir/wakashu_dir`）をスキャンします。
      * `fs` (File System) モジュールを使い、指定された回（例：`第1回`）と順位（例：`1-Laptop`, `2-Wakashu`）に一致するディレクトリを検索します。
      * 各順位のディレクトリから、JSONファイル（`TextData*.json`）と画像ファイル（`Picture*.png` や `.jpg` など）を読み込みます。

2.  **`Resources` (リソース) 機能**:

      * `tools/call` の応答 (Tool Result) として、MCPの **`Embedded Resources` (`"type": "resource"`)** 形式を利用します。
      * サーバーは、`tools/call` の処理中に画像ファイルを読み込み、**Base64エンコード**します。
      * MIMEタイプ（`image/png` など）と共に、Base64データ (`blob`) をレスポンスに直接埋め込みます。
      * これにより、クライアント（LLM）は追加の `resources/read` を呼び出す必要なく、即座に画像とテキストを組み合わせた応答を生成できます。
      * JSONから読み取ったテキストデータ（アカウント名、本文、URLなど）は、`"type": "text"` ブロックとして整形し、画像リソースブロックと交互に `content` 配列に配置します。

-----

### 2\. 実装にあたってのADR (Architecture Decision Record)

**ADR-001: ツール応答における画像データの転送方法**

  * **決定**: `tools/call` の応答 (Tool Result) にて、`Embedded Resources` (`"type": "resource"`) を使用し、画像データをBase64エンコードされた `blob` として含める。
  * **背景**:
      * 目標とする会話形式では、LLMがテキストと画像を即座に組み合わせて応答する必要がある。
      * `Resource Links` (`"type": "resource_link"`) を使う場合、クライアントは別途 `resources/read` を呼び出す必要があり、応答の即時性が失われるか、LLMが `file://` URIを正しく解釈できないリスクがある。
      * ユーザー要件の「リソース機能で参照」は、`Embedded Resources` 形式（リソース仕様に準拠）でも満たされる。
      * `Image Content` (`"type": "image"`) もBase64を使用するが、`Embedded Resources` は `uri` や `mimeType` を持つ正式なリソース表現であり、MCPの設計思想により適合する。
  * **結果**: サーバーは `tools/call` 処理中にファイルI/O（画像読み込みとBase64エンコード）を実行する。

**ADR-002: ファイルシステムの探索ロジック**

  * **決定**: 順位（`rank`）に基づき、`fs.readdir` を使用して正規表現で一致するディレクトリとファイルを検索する。
  * **背景**:
      * ディレクトリ名（`1-Laptop`）とファイル名（`Picture1.png`, `TextData1.json`）は、前提条件により順位番号（`1-`）と一致することが保証されている。
      * 画像形式は多様（png, jpg, jpeg, heic, webp）であるため、正規表現による検索が必須。
  * **結果**: 以下の正規表現パターンを使用する。
      * ディレクトリ検索: `new RegExp(\`^${rank}-\`)\`
      * JSONファイル検索: `new RegExp(\`^TextData${rank}\\.json$\`, 'i')\`
      * 画像ファイル検索: `new RegExp(\`^Picture${rank}\\.(png|jpg|jpeg|heic|webp)$\`, 'i')\`

**ADR-003: 依存ライブラリの選定**

  * **決定**: MIMEタイプ特定のために `mime-types` ライブラリを使用する。
  * **背景**: `Embedded Resources` は正確な `mimeType` の指定が推奨される。ファイル拡張子からMIMEタイプを安全に導出するために、実績のあるライブラリを利用する。
  * **結果**: `npm install mime-types @types/mime-types` を実行する。

-----

### 3\. 開発していくのに必要になる手順の説明

1.  **環境構築**:

      * Node.js (v18以降推奨) と npm がインストールされていることを確認します。
      * 作業ディレクトリを作成し、`npm init -y` を実行します。
      * TypeScript環境をセットアップします: `npm install -D typescript ts-node @types/node`
      * `tsconfig.json` ファイルを作成します: `npx tsc --init --rootDir src --outDir dist --module commonjs --target es2020 --esModuleInterop true --strict true`
      * `src` ディレクトリを作成します。

2.  **依存関係のインストール**:

      * `npm install mime-types`
      * `npm install -D @types/mime-types`

3.  **ソースコードの実装**:

      * `src/` ディレクトリ内に、以下の5つのファイルを作成します。
        1.  `config.ts` (設定ファイル)
        2.  `mcp-types.ts` (MCP関連の型定義)
        3.  `file-service.ts` (ファイルシステム操作ロジック)
        4.  `mcp-handler.ts` (MCPリクエスト処理ロジック)
        5.  `index.ts` (サーバーエントリーポイント)

4.  **`package.json` の編集**:

      * `scripts` に `start` コマンドと `build` コマンドを追加します。

5.  **ビルドと実行**:

      * `npm run build` (TypeScriptをJavaScriptにコンパイル)
      * `npm start` (サーバーを起動)

-----

### 4\. 実装に向けた各ファイルごとの実際のコード

以下に、`src/` ディレクトリ内に作成する各ファイルの完全なコードを示します。

#### `src/config.ts`

(和歌集データのルートディレクトリパスを定義します)

[コードは省略]

#### `src/mcp-types.ts`

(MCPのJSON-RPCリクエスト/レスポンス、ツール、リソースに関する型定義)

[コードは省略]

#### `src/file-service.ts`

(ファイルシステム操作ロジック)

[コードは省略]

#### `src/mcp-handler.ts`

(MCPリクエスト処理ロジック。ルーティングと各メソッドの実装)

[コードは省略]

#### `src/index.ts`

(HTTPサーバーを起動し、リクエストをハンドラに渡すエントリーポイント)

[コードは省略]

#### `package.json` の編集

（`npm init -y` で生成された `package.json` を以下のように編集します）

[コードは省略]

-----

### 5\. テストするときの手順の説明

サーバーが正しく動作するか確認するために、`curl` コマンド（またはPostmanのようなAPIクライアント）を使用して、MCPサーバーのエンドポイント（`http://localhost:8080/`）にJSON-RPCリクエストを手動で送信します。

**前提**:

1.  上記の手順でサーバーをセットアップし、`npm run build` を実行します。
2.  `npm start` でサーバーを起動します。
3.  `/home/info-gather/wakashu_mcp_dir/wakashu_dir` に、指定されたファイル構造（`第1回/1-Laptop/...` など）が実在することを確認します。

#### テスト1: `capabilities/list` (機能の確認)

```bash
curl -X POST -H "Content-Type: application/json" \
-d '{"jsonrpc": "2.0", "id": 1, "method": "capabilities/list", "params": {}}' \
http://localhost:8080/
```

  * **期待される応答 (例)**:
    ```json
    {
      "jsonrpc": "2.0",
      "id": 1,
      "result": {
        "capabilities": {
          "tools": { "listChanged": false },
          "resources": { "subscribe": false, "listChanged": false }
        }
      }
    }
    ```

#### テスト2: `tools/list` (ツールの確認)

```bash
curl -X POST -H "Content-Type: application/json" \
-d '{"jsonrpc": "2.0", "id": 2, "method": "tools/list", "params": {}}' \
http://localhost:8080/
```

  * **期待される応答 (例)**:
    ```json
    {
      "jsonrpc": "2.0",
      "id": 2,
      "result": {
        "tools": [
          {
            "name": "wakashu/getRanking",
            "title": "和歌集ランキング情報取得",
            "description": "指定された和歌集の、指定された回の、指定された順位範囲のランキング情報を取得します。",
            "inputSchema": {
              "type": "object",
              "properties": {
                "wakashuName": { "type": "string", "description": "..." },
                "edition": { "type": "integer", "description": "..." },
                "rankStart": { "type": "integer", "description": "..." },
                "rankEnd": { "type": "integer", "description": "..." }
              },
              "required": ["wakashuName", "edition", "rankStart", "rankEnd"]
            }
          }
        ]
      }
    }
    ```

#### テスト3: `tools/call` (和歌集ランキングの取得 - **最重要**)

これが、ユーザーのプロンプト「'wakashu\_mcp\_server'で「遠慮したラップトップ和歌集」のランキング情報を取得してください。取得するのは第1回の和歌集のランキング1位から3位の投稿です。」をLLMが解釈して実行するリクエストに相当します。（ここでは1位から2位までを取得します）

```bash
curl -X POST -H "Content-Type: application/json" \
-d '{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": {
    "name": "wakashu/getRanking",
    "arguments": {
      "wakashuName": "遠慮したラップトップ和歌集",
      "edition": 1,
      "rankStart": 1,
      "rankEnd": 2
    }
  }
}' \
http://localhost:8080/
```

  * **期待される応答 (例)**:
    （`blob` の値は非常に長くなります）
    ```json
    {
      "jsonrpc": "2.0",
      "id": 3,
      "result": {
        "content": [
          {
            "type": "text",
            "text": "1位：@Laptop"
          },
          {
            "type": "resource",
            "resource": {
              "uri": "file:///home/info-gather/wakashu_mcp_dir/wakashu_dir/第1回/1-Laptop/Picture1.png",
              "name": "Picture1.png",
              "mimeType": "image/png",
              "blob": "iVBORw0KGgoAAAANSUhEUgA... (ここにPicture1.pngのBase64データが続く) ...AAAElFTkSuQmCC"
            }
          },
          {
            "type": "text",
            "text": "\"正々堂々\"\nhttps://www.youtube.com/watch?v=8OZDgBmehbA\nhttps://www.amazon.co.jp/gp/video/detail/B0CH9YSQ55/ref=atv_dp_share_cu_r\nジャケット生成に使ったプロンプト\n「横を向いたカマキリのイラストを描いてください！」"
          },
          {
            "type": "text",
            "text": "2位：@Wakashu"
          },
          {
            "type": "resource",
            "resource": {
              "uri": "file:///home/info-gather/wakashu_mcp_dir/wakashu_dir/第1回/2-Wakashu/Picture2.png",
              "name": "Picture2.png",
              "mimeType": "image/png",
              "blob": "iVBORw0KGgoAAAANSUhEUgA... (ここにPicture2.pngのBase64データが続く) ...AAAElFTkSuQmCC"
            }
          },
          {
            "type": "text",
            "text": "\"'Winny'(2023)の作中、47氏は「ご飯3原則」を語っている。いわく「その１、熱くない。その２、辛くない。その３、食べにくくない。」あくまで推測の域を出ないけれど、これは「ロボット３原則」をオマージュした発言なのではないかと思う。工学出身の47氏だからあまり違和感はないし、47氏らしい感性だなと思う。\"\nhttps://www.amazon.co.jp/gp/video/detail/B0CH9YSQ55/ref=atv_dp_share_cu_r\nジャケット生成に使ったプロンプト\n「デフォルメされたロボットらしいロボットの座っているイラストを描いてくれる？」"
          }
        ],
        "isError": false
      }
    }
    ```

この`tools/call`の応答（`result.content`）を受け取ったClaude Desktop (LLM) は、この配列を上から順に解釈し、`type: "text"` をテキストとして表示し、`type: "resource"` の `blob` データ（Base64）をデコードして画像としてインライン表示することで、目標とする会話形式の応答を生成することが期待されます。
