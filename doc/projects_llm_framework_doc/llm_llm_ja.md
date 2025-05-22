# llm-llm (大規模言語モデル)

`llm-llm`ユニットは、大規模言語モデルの推論サービスを提供し、プロンプトに応じたテキスト生成を可能にします。様々なモデルを使用するように設定でき、直接のAPI呼び出し、ASRユニットの書き起こし、KWSユニットのトリガーなど、異なる入力ソースをサポートしています。出力はストリーミングまたは完全なテキストとして配信できます。

## API呼び出しの共通規約

`llm-llm`へのAPI呼び出しはJSONメッセージを介して行われます。

### リクエスト構造

`llm-llm` APIへの標準的なリクエストは以下の構造に従います：

```json
{
  "request_id": "クライアント生成のUUIDまたはカウンター",
  "work_id": "llmまたはllm.インスタンスID",
  "action": "APIアクション名",
  "object": "オプションのオブジェクトタイプ文字列", // 該当する場合
  "data": { /* API固有のペイロード */ }     // または一部APIでは文字列/オブジェクト
}
```

-   `request_id` (string, 必須): リクエストとレスポンスを関連付けるための、クライアントが生成する一意の識別子。
-   `work_id` (string, 必須):
    -   初期設定や一般的なクエリの場合: `"llm"`。
    -   特定のLLMタスクインスタンスに対する操作の場合: `"llm.XXXX"` (例: `"llm.1002"`)。`XXXX`は成功した`setup`呼び出しによって返されるインスタンスIDです。
-   `action` (string, 必須): 呼び出すAPIアクション (例: `"setup"`, `"inference"`)。
-   `object` (string, オプション): 送信するデータのタイプやコンテキストを指定します（該当する場合）。例：設定データの場合は`"llm.setup"`、プロンプトデータの場合は`"llm.utf-8"`。
-   `data` (object または string, オプション): API呼び出しのペイロード。

### レスポンス構造

**成功レスポンス (ほとんどのAPI呼び出し):**

API呼び出しが受け入れられ、正常に処理されたことを示します。

```json
{
  "request_id": "リクエストからミラーリング",
  "work_id": "llmまたはllm.インスタンスID", // ミラーリングまたは更新 (例: setup後の "llm.1002")
  "action": "APIアクション名",       // リクエストからミラーリング
  "created": 1678886400,             // 整数: レスポンス生成のUnixタイムスタンプ
  "code": 0,                         // 整数: 0は成功を示す
  "message": "OK",                   // 文字列: 成功メッセージ
  "data": { /* API固有データ */ } // オプション: APIによって返されるペイロード
}
```
-   `inference` APIの場合、この同期レスポンスはリクエストの受領を確認するだけです。実際のLLM出力は非同期に送信されます（データ入出力セクション参照）。
-   API呼び出しが新しいタスクインスタンスの作成に成功した場合（例：`setup`）、レスポンスの`work_id`および`data.work_id`は新しいインスタンスIDになります。

**エラーレスポンス:**

API呼び出し処理中の失敗を示します。

```json
{
  "request_id": "リクエストからミラーリング",
  "work_id": "llmまたはllm.インスタンスID",
  "action": "APIアクション名",        // リクエストからミラーリング
  "created": 1678886400,              // 整数: レスポンス生成のUnixタイムスタンプ
  "error": {
    "code": -1,                       // 整数: 0以外のエラーコード
    "message": "エラー詳細"    // 文字列: 詳細なエラーメッセージ
  }
}
```

**一般的なエラーコード:**

-   `-2`: リクエスト`data`内のJSONフォーマットが無効です。
-   `-3`: LLMエンジンの初期化に失敗しました（`LLM::Init`内部エラー）。
-   `-5`: モデルの読み込みに失敗しました（例：モデルファイルが見つからない、トークナイザの問題、モデル設定が無効）。
-   `-6`: 指定されたLLMタスクインスタンス（例：`"llm.1002"`）が存在しません。
-   `-11`: 一般的なモデル推論中または内部処理中のエラー。
-   `-20`: 入力ソースのリンクまたはデータ購読に関するエラー。
-   `-21`: タスク制限に達しました（例：LLMインスタンスの`MAX_TASK_NUM`、現在2）。
-   `-23`: トークン化エラーまたは無効なプロンプト構造。
-   `-25`: 推論エラー（例：`lLaMa_->Run`呼び出し中の問題）。

## データ入出力 (推論結果)

### 入力データ (プロンプト)

`llm-llm`ユニットは、いくつかの方法で推論用のプロンプトを受信できます：

1.  **特定の`llm.XXXX`インスタンスへの`inference` API経由:**
    *   **非ストリーミングプロンプト (`object: "llm.utf-8"`)**:
        -   `data`フィールドは、完全なプロンプトを含む単一の文字列です。
    *   **ストリーミングプロンプト (`object: "llm.utf-8.stream"`)**:
        -   `data`フィールドはJSONオブジェクトです: `{"index": <整数>, "delta": "<プロンプトのチャンク>", "finish": <真偽値>}`。
        -   `index`: チャンクのシーケンス番号、0から開始。
        -   `delta`: プロンプトのセグメント。
        -   `finish`: これがプロンプトの最終チャンクである場合は`true`、それ以外は`false`。これにより、長いプロンプトをセグメントで送信できます。

2.  **リンクされたユニット経由 (`setup` APIの`input`パラメータで設定):**
    *   **`"llm.utf-8"` または `"llm.utf-8.stream"`**: インスタンスは、上記のように`inference` APIを使用して自身の`work_id`に送信されたプロンプトをリッスンします。
    *   **`"asr.XXXX"` (例: `"asr.1001"`)**: LLMインスタンスは、指定されたASR（自動音声認識）ユニットの出力を購読します。ASRからの書き起こされたテキスト（完全なテキスト、または`finish`がtrueの場合のストリームからの`delta`）がLLMのプロンプトとして使用されます。
    *   **`"kws.XXXX"` (例: `"kws.1000"`)**: LLMインスタンスは、指定されたKWS（キーワードスポッティング）ユニットからのイベントを購読します。キーワードが検出されると、KWSユニットは通常、ブール値またはキーワード自体を送信します。`llm-llm`ユニットの`kws_awake`関数がトリガーされ、`lLaMa_->Stop()`を呼び出します。これは、進行中のLLM生成を中断するために使用され、事実上停止信号または新しい（おそらく事前に定義された）対話フローのトリガーとして機能します（KWSデータで直接プロンプトを出すためではありません）。

### 出力データ (生成されたテキスト)

`setup`呼び出しで`enoutput`が`true`（デフォルト）の場合、`llm-llm`ユニットは`inference`リクエストが受け付けられた後、生成されたテキストを非同期にプッシュします。フォーマットは`setup` APIの`response_format`によって決定されます：

1.  **ストリーミング出力 (`response_format: "llm.utf-8.stream"`)**
    -   ユニットはJSONオブジェクトのストリームを送信し、各オブジェクトは生成されたテキストのチャンクを表します。
    -   **チャンクごとのJSON構造:**
        ```json
        {
          "request_id": "<元のinferenceリクエストID>",
          "work_id": "llm.XXXX",                     // 特定のLLMタスクインスタンスID
          "action": "llm_event",                     // LLM生成イベント/データを示す
          "object": "llm.utf-8.stream",              // レスポンスフォーマット
          "created": 1678886410,                     // 整数: このチャンク生成のUnixタイムスタンプ
          "code": 0,                                 // 整数: 成功したチャンクの場合は0
          "message": "OK",                           // 文字列: "OK"
          "data": {
            "index": 0,                              // 整数: このチャンクのシーケンス番号
            "delta": "<生成されたテキストチャンク>",    // 文字列: 生成されたテキストのセグメント
            "finish": false                          // 真偽値: これがレスポンスの最終チャンクである場合は`true`、それ以外は`false`
          }
        }
        ```
    -   これらの非同期メッセージの`request_id`は、生成を開始した`inference`呼び出しの`request_id`と一致します。

2.  **非ストリーミング出力 (`response_format: "llm.utf-8"`)**
    -   ユニットは、推論が完了すると、生成されたテキスト全体を含む単一のJSONメッセージを送信します。
    -   **JSON構造:**
        ```json
        {
          "request_id": "<元のinferenceリクエストID>",
          "work_id": "llm.XXXX",
          "action": "llm_event",
          "object": "llm.utf-8",
          "created": 1678886415,
          "code": 0,
          "message": "OK",
          "data": "<生成された完全なテキスト文字列>" // 文字列: 生成された完全なテキスト
        }
        ```

## APIリファレンス

### **setup**

新しい大規模言語モデル（LLM）タスクインスタンスを初期化し、設定します。

-   リクエストの`work_id`: `"llm"`
-   レスポンスの`work_id` (成功時): `"llm.XXXX"` (例: `"llm.1002"`)

**リクエストパラメータ:**

| パラメータ             | 型            | 必須 | デフォルト (モデルJSONより該当する場合) | 説明                                                                                                                                                                                                                                                             |
|------------------------|---------------|------|-----------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `object`               | string        | はい | N/A                                     | `llm.setup`である必要があります。                                                                                                                                                                                                                                          |
| `data.model`           | string        | はい | N/A                                     | LLM設定ファイルの名前（`.json`拡張子なし）。例: `"qwen2.5-0.5B-prefill-20e"`, `"qwen1.5-0.5B-Chat"`。モデルは`projects/llm_framework/main_llm/models/`に配置されます。                                                                                  |
| `data.response_format` | string        | はい | N/A                                     | `enoutput`がtrueの場合の生成テキストの出力フォーマット。サポート: `"llm.utf-8.stream"` (テキストチャンクのストリーム), `"llm.utf-8"` (単一テキスト文字列)。                                                                                                               |
| `data.input`           | string/array  | はい | N/A                                     | プロンプトの入力ソース。文字列 (例: `"llm.utf-8"`) または配列 (例: `["llm.utf-8", "asr.1001", "kws.1000"]`)。異なるソースがどのように処理されるかの詳細は「入力データ」セクションを参照してください。                                                              |
| `data.enoutput`        | boolean       | いいえ | `true`                                  | 生成されたテキスト結果の非同期プッシュを有効 (`true`) または無効 (`false`) にします。                                                                                                                                                                                     |
| `data.max_token_len`   | integer       | いいえ | (モデルJSONより)                        | レスポンスで生成する最大トークン数。指定された場合、モデルJSONの`max_token_len`を上書きします。モデルの絶対最大シーケンス長によって制限されます。                                                                                                                               |
| `data.prompt`          | string        | いいえ | (モデルJSONより、または空)              | LLMインスタンスに設定するシステムプロンプトまたは初期コンテキスト。指定された場合、モデルJSONの`system_prompt`を上書きします。                                                                                                                                         |
| `data.mode_param`      | object        | いいえ | {}                                      | モデルのJSONファイルで定義された特定の推論パラメータ（例：`temperature`, `top_p`）を上書きできます。**注意:** 現在の`main.cpp`に基づくと、これは全ての`LLMAttrType`フィールドに対する汎用的な上書きメカニズムではありません。これらのための特定の解析は`setup`の`load_model`には実装されていません。推論パラメータは主に選択されたモデルのJSONファイルによって制御されます。 |

**リクエスト例:**
```json
{
  "request_id": "setup_llm_001",
  "work_id": "llm",
  "action": "setup",
  "object": "llm.setup",
  "data": {
    "model": "qwen2.5-0.5B-prefill-20e",
    "response_format": "llm.utf-8.stream",
    "input": ["llm.utf-8", "asr.1001"],
    "enoutput": true,
    "max_token_len": 512,
    "prompt": "あなたは役立つAIアシスタントです。"
  }
}
```

**レスポンス (成功):**
```json
{
  "request_id": "setup_llm_001",
  "work_id": "llm.1002", // 新しいLLMタスクインスタンスID
  "action": "setup",
  "created": 1678886400,
  "code": 0,
  "message": "OK",
  "data": {
    "work_id": "llm.1002" // 作成されたインスタンスIDを確認
  }
}
```

**レスポンス (エラー - モデル読み込み失敗):**
```json
{
  "request_id": "setup_llm_002",
  "work_id": "llm",
  "action": "setup",
  "created": 1678886401,
  "error": {
    "code": -5,
    "message": "モデルの読み込みに失敗しました。"
  }
}
```

### **inference**

特定のLLMタスクインスタンスにテキスト生成のためのプロンプトを送信します。実際の生成テキストは非同期に送信されます。

-   リクエストの`work_id`: `"llm.XXXX"` (特定のLLMタスクインスタンスID)

**リクエストパラメータ:**

| パラメータ | 型            | 必須 | デフォルト | 説明                                                                                                                               |
|-----------|---------------|------|------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| `object`  | string        | はい | N/A        | プロンプトデータのフォーマットを指定します: `"llm.utf-8"` (単一の完全なプロンプトの場合) または `"llm.utf-8.stream"` (チャンク化されたプロンプトの場合)。      |
| `data`    | string/object | はい | N/A        | `object`が`"llm.utf-8"`の場合、`data`は完全なプロンプトを含む文字列です。`object`が`"llm.utf-8.stream"`の場合、`data`はオブジェクトです: `{"index": <整数>, "delta": "<プロンプトのチャンク>", "finish": <真偽値>}`。 |

**リクエスト例 (非ストリーミング):**
```json
{
  "request_id": "infer_llm_003",
  "work_id": "llm.1002",
  "action": "inference",
  "object": "llm.utf-8",
  "data": "フランスの首都はどこですか？"
}
```

**リクエスト例 (ストリーミングプロンプト - 最終チャンク):**
```json
{
  "request_id": "infer_llm_004",
  "work_id": "llm.1002",
  "action": "inference",
  "object": "llm.utf-8.stream",
  "data": {
    "index": 0,
    "delta": "勇敢なロボットについての短い物語を教えてください。",
    "finish": true
  }
}
```

**レスポンス (成功 - リクエスト受付):**
このレスポンスはプロンプトが受信されたことを確認するだけです。生成されたテキストは非同期メッセージで続きます。
```json
{
  "request_id": "infer_llm_003",
  "work_id": "llm.1002",
  "action": "inference",
  "created": 1678886402,
  "code": 0,
  "message": "OK" // または "推論プロセスが開始されました" のようなメッセージ
}
```

**レスポンス (エラー - タスクが見つからない):**
```json
{
  "request_id": "infer_llm_005",
  "work_id": "llm.9999", // 存在しないタスク
  "action": "inference",
  "created": 1678886403,
  "error": {
    "code": -6,
    "message": "ユニットが存在しません"
  }
}
```

### **link**

セットアップ後、LLMタスクインスタンスを入力ソース（例：ASRまたはKWSユニット）に動的にリンクします。

-   リクエストの`work_id`: `"llm.XXXX"`

**リクエストパラメータ:**

| パラメータ | 型     | 必須 | デフォルト | 説明                                                                       |
|-----------|--------|------|------------|----------------------------------------------------------------------------|
| `object`  | string | はい | N/A        | 通常は`"work_id"`で、dataフィールドにリンクするユニットIDが含まれることを示します。 |
| `data`    | string | はい | N/A        | 入力ソースとしてリンクするユニットの`work_id` (例: `"asr.1001"`)。             |

**リクエスト例:**
```json
{
  "request_id": "link_llm_006",
  "work_id": "llm.1002",
  "action": "link",
  "object": "work_id",
  "data": "asr.1001"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "link_llm_006",
  "work_id": "llm.1002",
  "action": "link",
  "created": 1678886404,
  "code": 0,
  "message": "OK"
}
```

### **unlink**

以前にリンクされた入力ソースをLLMタスクインスタンスから解除します。

-   リクエストの`work_id`: `"llm.XXXX"`

**リクエストパラメータ:**

| パラメータ | 型     | 必須 | デフォルト | 説明                                                                           |
|-----------|--------|------|------------|--------------------------------------------------------------------------------|
| `object`  | string | はい | N/A        | 通常は`"work_id"`で、dataフィールドに解除するユニットIDが含まれることを示します。     |
| `data`    | string | はい | N/A        | 解除する入力ソースユニットの`work_id` (例: `"asr.1001"`)。                 |

**リクエスト例:**
```json
{
  "request_id": "unlink_llm_007",
  "work_id": "llm.1002",
  "action": "unlink",
  "object": "work_id",
  "data": "asr.1001"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "unlink_llm_007",
  "work_id": "llm.1002",
  "action": "unlink",
  "created": 1678886405,
  "code": 0,
  "message": "OK"
}
```

### **pause**

LLMタスクインスタンスを一時停止し、進行中のテキスト生成を停止させます。

-   リクエストの`work_id`: `"llm.XXXX"`

**リクエストパラメータ**: 標準の`request_id`, `work_id`, `action`以外はありません。

**リクエスト例:**
```json
{
  "request_id": "pause_llm_008",
  "work_id": "llm.1002",
  "action": "pause"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "pause_llm_008",
  "work_id": "llm.1002",
  "action": "pause",
  "created": 1678886406,
  "code": 0,
  "message": "OK"
}
```

### **work**

一時停止したLLMタスクインスタンスを再開します。注意: これは完了した推論を再開するものではなく、一時停止した生成（基盤となるモデルがサポートしている場合）を継続したり、新しい推論を受け付けたりすることを可能にします。テキストを生成する主な方法は`inference` API経由です。

-   リクエストの`work_id`: `"llm.XXXX"`

**リクエストパラメータ**: 標準の`request_id`, `work_id`, `action`以外はありません。

**リクエスト例:**
```json
{
  "request_id": "work_llm_009",
  "work_id": "llm.1002",
  "action": "work"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "work_llm_009",
  "work_id": "llm.1002",
  "action": "work",
  "created": 1678886407,
  "code": 0,
  "message": "OK"
}
```

### **exit**

LLMタスクインスタンスを停止し、削除してリソースを解放します。

-   リクエストの`work_id`: `"llm.XXXX"`

**リクエストパラメータ**: 標準の`request_id`, `work_id`, `action`以外はありません。

**リクエスト例:**
```json
{
  "request_id": "exit_llm_010",
  "work_id": "llm.1002",
  "action": "exit"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "exit_llm_010",
  "work_id": "llm.1002", // 終了したタスクのwork_id
  "action": "exit",
  "created": 1678886408,
  "code": 0,
  "message": "OK"
}
```

### **taskinfo**

アクティブなLLMタスクに関する情報を取得します。

**ケース1: 一般的なLLMユニット情報**

-   リクエストの`work_id`: `"llm"`
-   **説明**: 全てのアクティブなLLMタスクインスタンスIDのリストを返します。

**リクエスト例:**
```json
{
  "request_id": "taskinfo_llm_general_011",
  "work_id": "llm",
  "action": "taskinfo"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "taskinfo_llm_general_011",
  "work_id": "llm",
  "action": "taskinfo",
  "created": 1678886409,
  "code": 0,
  "message": "OK",
  "object": "llm.tasklist",
  "data": [
    "llm.1002" // アクティブなインスタンスIDのリスト
  ]
}
```

**ケース2: 特定のLLMタスクインスタンス情報**

-   リクエストの`work_id`: `"llm.XXXX"` (例: `"llm.1002"`)
-   **説明**: 指定されたLLMタスクインスタンスのランタイムパラメータと設定を返します。

**リクエスト例:**
```json
{
  "request_id": "taskinfo_llm_specific_012",
  "work_id": "llm.1002",
  "action": "taskinfo"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "taskinfo_llm_specific_012",
  "work_id": "llm.1002",
  "action": "taskinfo",
  "created": 1678886410,
  "code": 0,
  "message": "OK",
  "object": "llm.taskinfo",
  "data": {
    "model": "qwen2.5-0.5B-prefill-20e",
    "response_format": "llm.utf-8.stream",
    "enoutput": true,
    "inputs": ["llm.utf-8", "asr.1001"], // 設定された入力ソースの配列
    "max_token_len": 512,
    "prompt": "あなたは役立つAIアシスタントです。"
    // temperatureやtop_pのような他の関連パラメータはモデルの内部設定の一部であり、
    // main.cppで明示的に追加されない限り、通常taskinfoにはリストされません。
  }
}
```

> **`work_id`に関する注意**: タスクインスタンスの`work_id`の数値部分（例：`llm.1002`の`1002`）は動的に割り当てられます。

## 設定ファイルとスクリプトとの関連性

`llm-llm`ユニットの動作は、モデル固有のJSON設定ファイルと、一部のモデルではPythonスクリプトによって実行される外部トークナイザサービスによって大きく定義されます。

### モデル設定JSON (例: `models/mode_qwen2.5-0.5B-prefill-20e.json`)

これらのファイルは、`projects/llm_framework/main_llm/models/`ディレクトリ（またはシステム全体のモデルパス）にあり、LLMエンジンのパラメータを含んでいます：

-   **`mode`**: `setup` APIで使用されるモデル名（例：`"qwen2.5-0.5B-prefill-20e"`）。
-   **`type`**: `"llm"`。
-   **`capabilities`**: 例: `"text_generation"`, `"chat"`。
-   **`mode_param`オブジェクト**: 主要なモデルおよび推論設定が含まれています：
    -   `tokenizer_type` (integer): トークナイザアルゴリズムを指定します（例：LLaMa、Qwen）。
    -   `filename_tokenizer_model` (string): トークナイザモデルファイルへのパス（例：`"qwen.tiktoken"`）、またはトークナイザサーバーが使用される場合はHTTP URL（例：`"http://localhost:PORT"`）。
    -   `filename_tokens_embed` (string): トークン埋め込みファイルへのパス。
    -   `filename_post_axmodel` (string): 後処理モデルファイルへのパス。
    -   `template_filename_axmodel` (string): レイヤー固有のモデルファイルのテンプレート（例：`"qwen2_p128_l%d_together.axmodel"`）。
    -   `axmodel_num` (integer): モデルレイヤーの数。
    -   `tokens_embed_num`, `tokens_embed_size`: トークン埋め込みの次元。
    -   `max_token_len` (integer): 生成するトークンのデフォルト最大数。`setup` APIで上書き可能。
    -   `temperature`, `top_p`, `top_k`, `repetition_penalty`: ランダム性や創造性を制御する標準的なLLM推論パラメータ。これらは`LLM::Init`によってロードされ、現在の`main.cpp`ロジックによれば、通常`setup` APIの`data.mode_param`フィールドで上書きすることはできません。
    -   `system_prompt` (string, チャットモデルでよく見られる): デフォルトのシステムプロンプト。`setup` APIの`prompt`フィールドで上書き可能。
    -   その他のモデル固有のパスとフラグ（例：`b_use_topk`, `b_bos`, `b_eos`）。

### トークナイザサーバースクリプト (例: `scripts/*_tokenizer.py`)

-   **目的**: 一部のモデル、特に`tiktoken`のようなトークナイザを使用するモデル（例：Qwenモデル）は、外部のPythonベースのトークナイザサーバーを利用する場合があります。
-   **操作**:
    1.  モデルのJSON設定内の`filename_tokenizer_model`がHTTP URL（例：`"http://localhost:DYNAMIC_PORT"`）である場合、`llm-llm`ユニットの`main.cpp`はPythonスクリプトをフォークしようとします。
    2.  スクリプト（例：`scripts/qwen2.5-0.5B-prefill-20e_tokenizer.py`または汎用スクリプト）は、動的に割り当てられたポート（8080から開始）でローカルHTTPサーバーを起動します。
    3.  `LLM::Init`は次に、トークン化タスク（エンコード/デコード）のためにこのPythonサーバーと通信するようにC++トークナイザクライアントを設定します。
    4.  `setup` APIで提供された`prompt`は、初期化中にこのサーバーに渡される場合があります。
-   このメカニズムにより、純粋なC++で実装または維持するのが難しい複雑なPythonベースのトークナイザを使用できます。

これらの設定を理解することは、モデルの選択、適切なパラメータの設定、およびモデルの読み込みや推論動作に関連する問題の診断に不可欠です。
```

Now, I will create the Japanese documentation file.
