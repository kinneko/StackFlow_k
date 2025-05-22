# llm-kws (キーワードスポッティング)

`llm-kws`ユニットは、キーワードスポッティング（音声ウェイクアップ）サービスを提供します。オーディオストリームをリッスンし、事前に定義されたキーワードが検出されるとイベントをトリガーします。中国語や英語など、さまざまな言語の異なるモデルをサポートしています。

## API呼び出しの共通規約

`llm-kws`へのAPI呼び出しはJSONメッセージを介して行われます。

### リクエスト構造

`llm-kws` APIへの標準的なリクエストは以下の構造に従います：

```json
{
  "request_id": "クライアント生成のUUIDまたはカウンター",
  "work_id": "kwsまたはkws.インスタンスID",
  "action": "APIアクション名",
  "object": "オプションのオブジェクトタイプ文字列", // 該当する場合
  "data": { /* API固有のペイロード */ }     // または一部APIでは文字列
}
```

-   `request_id` (string, 必須): リクエストとレスポンスを関連付けるための、クライアントが生成する一意の識別子。
-   `work_id` (string, 必須):
    -   初期設定や一般的なクエリの場合: `"kws"`。
    -   特定のKWSタスクインスタンスに対する操作の場合: `"kws.XXXX"` (例: `"kws.1000"`)。`XXXX`は成功した`setup`呼び出しによって返されるインスタンスIDです。
-   `action` (string, 必須): 呼び出すAPIアクション (例: `"setup"`, `"pause"`)。
-   `object` (string, オプション): 送信するデータのタイプやコンテキストを指定します（該当する場合）。例：設定データの場合は`"kws.setup"`。
-   `data` (object または string, オプション): API呼び出しのペイロード。構造はAPIによって異なります。

### レスポンス構造

**成功レスポンス:**

API呼び出しが受け入れられ、正常に処理されたことを示します。

```json
{
  "request_id": "リクエストからミラーリング",
  "work_id": "kwsまたはkws.インスタンスID", // ミラーリングまたは更新 (例: setup後の "kws.1000")
  "action": "APIアクション名",      // リクエストからミラーリング
  "created": 1678886400,            // 整数: レスポンス生成のUnixタイムスタンプ
  "code": 0,                        // 整数: 0は成功を示す
  "message": "OK",                  // 文字列: 成功メッセージ
  "data": { /* API固有データ */ } // オプション: APIによって返されるペイロード
}
```
-   API呼び出しが新しいKWSタスクインスタンスの作成に成功した場合（例：`setup`）、レスポンスの`work_id`は新しいインスタンスID（例：`"kws.1000"`）になり、多くの場合`data`オブジェクトにも含まれます。

**エラーレスポンス:**

API呼び出し処理中の失敗を示します。

```json
{
  "request_id": "リクエストからミラーリング",
  "work_id": "kwsまたはkws.インスタンスID",
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
-   `-3`: KWSエンジンの初期化に失敗しました（Sherpa-ONNXの内部エラー）。
-   `-5`: モデルの読み込みに失敗しました（例：モデルファイルが見つからない、モデル設定が無効）。
-   `-6`: 指定されたKWSタスクインスタンス（例：`"kws.1000"`）が存在しません。
-   `-11`: キーワードスポッティング中または内部処理中の一般的なエラー。
-   `-21`: タスク制限に達しました（通常、モデルタイプごと、または全体で1つのKWSインスタンスのみサポートされます）。
-   `-23`: キーワードファイルの生成中にエラーが発生しました（例：`llm-kws_text2token.py`スクリプトの失敗、無効なキーワードフォーマット、または`tokens.txt`や`bpe.model`の問題）。
-   `-25`: ウェイクアップ音声の再生に失敗しました（`enwake_audio`がtrueの場合）。`llm_audio`ユニットまたは指定された`wake_wav_file`の問題可能性があります。

## ウェイクアップイベント通知

設定されたキーワードが検出され、`enoutput`が`true`（デフォルト）の場合、`llm-kws`ユニットはイベント通知をプッシュします。このユニットの主要な`response_format`は`"kws.bool"`であり、検出時にブール型のイベントを示します。

-   **メカニズム**: クライアントに（またはユニットの出力に接続されたZMQサブスクライバに）非同期JSONメッセージがプッシュされます。
-   **トリガー**: 設定されたキーワードのいずれかが十分な信頼度で検出された場合。

**イベントJSON構造 (`response_format: "kws.bool"`):**

```json
{
  "request_id": "<元のsetupリクエストID>", // このインスタンスを作成したsetup呼び出しのrequest_id
  "work_id": "kws.XXXX",                     // 特定のKWSタスクインスタンスID
  "action": "kws_event",                     // キーワードスポッティングイベントを示すアクション
  "object": "kws.bool",                      // 指定されたレスポンスフォーマット
  "created": 1678886410,                     // 整数: イベント生成のUnixタイムスタンプ
  "code": 0,
  "message": "キーワードが検出されました",     // または同様の情報メッセージ（実際のメッセージは異なる場合があります）
  "data": true                               // 真偽値: trueはキーワードが検出されたことを示す
}
```
-   イベントの`action`フィールド（例：`"kws_event"`）は明確性のための規約です。主要な情報は`object`と`data`に含まれます。`main.cpp`の`send_obj_true`によってプッシュされる実際のメッセージには、イベント自体の"action"フィールドが含まれていない場合がありますが、`object`と`data`は含まれます。

## APIリファレンス

### **setup**

新しいキーワードスポッティング（KWS）タスクインスタンスを初期化し、設定します。

-   リクエストの`work_id`: `"kws"`
-   レスポンスの`work_id` (成功時): `"kws.XXXX"` (例: `"kws.1000"`)

**リクエストパラメータ:**

| パラメータ        | 型             | 必須 | デフォルト (モデルJSONより該当する場合) | 説明                                                                                                                                                                                                                                                           |
|-------------------|----------------|------|-----------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `object`          | string         | はい | N/A                                     | `"kws.setup"`である必要があります。                                                                                                                                                                                                                                  |
| `data.model`      | string         | はい | N/A                                     | KWSモデル設定ファイルの名前（`.json`拡張子なし）。例: `"sherpa-onnx-kws-zipformer-gigaspeech-3.3M-2024-01-01"`。                                                                                                                                      |
| `data.response_format`| string    | はい | `"kws.bool"`                            | キーワード検出イベントの出力フォーマット。現在、`"kws.bool"`が主要なサポートフォーマットであり、検出時にブール値`true`のイベントを示します。                                                                                                                     |
| `data.input`      | string/array   | はい | N/A                                     | 音声入力ソース。通常、システムマイク入力の場合は`"sys.pcm"`、または他のユニット（例：音声処理ユニットID）からのオーディオストリーム。                                                                                                                |
| `data.enoutput`   | boolean        | いいえ | `true`                                  | キーワード検出イベントのプッシュを有効 (`true`) または無効 (`false`) にします。                                                                                                                                                                                             |
| `data.kws`        | string / array | はい | (モデルJSONのデフォルト`keywords_file`を使用) | スポッティングするキーワード。単一の文字列または文字列の配列。これらは`llm-kws_text2token.py`によって処理され、モデル固有のキーワードファイルが生成されます。<br>英語の例: `"HEY SIRI"`, `["HELLO WORLD", "OK GOOGLE"]`<br>中国語の例: `"你好小爱"`, `["小爱同学", "你好问问"]` |
| `data.enwake_audio`| boolean       | いいえ | `true` (例より。ただし`main.cpp`のデフォルトはキーが存在すれば`true`、なければ`false`のようです) | `true`の場合、キーワード検出時にウェイクアップ音を再生します。音声ファイルはモデルのJSON設定内の`wake_wav_file`で指定されます。これは内部的に`llm_audio`ユニットの`play_raw` APIを使用します。                                                              |

**リクエスト例 (英語キーワード):**
```json
{
  "request_id": "setup_kws_en_001",
  "work_id": "kws",
  "action": "setup",
  "object": "kws.setup",
  "data": {
    "model": "sherpa-onnx-kws-zipformer-gigaspeech-3.3M-2024-01-01",
    "response_format": "kws.bool",
    "input": "sys.pcm",
    "enoutput": true,
    "kws": ["HELLO M5", "HEY STACK"],
    "enwake_audio": true
  }
}
```

**リクエスト例 (中国語キーワード - 中国語モデルが指定されていると仮定):**
```json
{
  "request_id": "setup_kws_cn_002",
  "work_id": "kws",
  "action": "setup",
  "object": "kws.setup",
  "data": {
    "model": "sherpa-onnx-kws-zipformer-wenetspeech-3.3M-2024-01-01", // 中国語KWSモデル名の例
    "response_format": "kws.bool",
    "input": "sys.pcm",
    "kws": "你好小爱" 
  }
}
```

**レスポンス (成功):**
```json
{
  "request_id": "setup_kws_en_001",
  "work_id": "kws.1000", // 新しいKWSタスクインスタンスID
  "action": "setup",
  "created": 1678886400,
  "code": 0,
  "message": "OK",
  "data": {
    "work_id": "kws.1000" // 作成されたインスタンスIDを確認
  }
}
```

**レスポンス (エラー - モデル読み込み失敗):**
```json
{
  "request_id": "setup_kws_003",
  "work_id": "kws",
  "action": "setup",
  "created": 1678886401,
  "error": {
    "code": -5,
    "message": "モデルの読み込みに失敗しました。"
  }
}
```

**レスポンス (エラー - キーワード処理失敗):**
```json
{
  "request_id": "setup_kws_004",
  "work_id": "kws",
  "action": "setup",
  "created": 1678886402,
  "error": {
    "code": -23,
    "message": "キーワードの処理またはキーワードファイルの生成に失敗しました。"
  }
}
```

### **pause**

KWSタスクインスタンスを一時停止します。一時停止中は、キーワードをリッスンしたり検出したりしません。

-   リクエストの`work_id`: `"kws.XXXX"` (特定のKWSタスクインスタンスID)

**リクエストパラメータ**: 標準の`request_id`, `work_id`, `action`以外はありません。

**リクエスト例:**
```json
{
  "request_id": "pause_kws_005",
  "work_id": "kws.1000",
  "action": "pause"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "pause_kws_005",
  "work_id": "kws.1000",
  "action": "pause",
  "created": 1678886403,
  "code": 0,
  "message": "OK"
}
```

### **work**

一時停止したKWSタスクインスタンスを再開します。

-   リクエストの`work_id`: `"kws.XXXX"`

**リクエストパラメータ**: 標準の`request_id`, `work_id`, `action`以外はありません。

**リクエスト例:**
```json
{
  "request_id": "work_kws_006",
  "work_id": "kws.1000",
  "action": "work"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "work_kws_006",
  "work_id": "kws.1000",
  "action": "work",
  "created": 1678886404,
  "code": 0,
  "message": "OK"
}
```

### **exit**

KWSタスクインスタンスを停止し、削除してリソースを解放します。

-   リクエストの`work_id`: `"kws.XXXX"`

**リクエストパラメータ**: 標準の`request_id`, `work_id`, `action`以外はありません。

**リクエスト例:**
```json
{
  "request_id": "exit_kws_007",
  "work_id": "kws.1000",
  "action": "exit"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "exit_kws_007",
  "work_id": "kws.1000", // 終了したタスクのwork_id
  "action": "exit",
  "created": 1678886405,
  "code": 0,
  "message": "OK"
}
```

### **taskinfo**

アクティブなKWSタスクに関する情報を取得します。動作はリクエストの`work_id`に依存します。

**ケース1: 一般的なKWSユニット情報**

-   リクエストの`work_id`: `"kws"`
-   **説明**: 全てのアクティブなKWSタスクインスタンスIDのリストを返します。

**リクエスト例:**
```json
{
  "request_id": "taskinfo_kws_general_008",
  "work_id": "kws",
  "action": "taskinfo"
}
```

**レスポンス (成功 - 1タスクアクティブ時):**
```json
{
  "request_id": "taskinfo_kws_general_008",
  "work_id": "kws",
  "action": "taskinfo",
  "created": 1678886406,
  "code": 0,
  "message": "OK",
  "object": "kws.tasklist",
  "data": [
    "kws.1000" // アクティブなインスタンスIDのリスト
  ]
}
```

**ケース2: 特定のKWSタスクインスタンス情報**

-   リクエストの`work_id`: `"kws.XXXX"` (例: `"kws.1000"`)
-   **説明**: 指定されたKWSタスクインスタンスのランタイムパラメータと設定を返します。

**リクエスト例:**
```json
{
  "request_id": "taskinfo_kws_specific_009",
  "work_id": "kws.1000",
  "action": "taskinfo"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "taskinfo_kws_specific_009",
  "work_id": "kws.1000",
  "action": "taskinfo",
  "created": 1678886407,
  "code": 0,
  "message": "OK",
  "object": "kws.taskinfo",
  "data": {
    "model": "sherpa-onnx-kws-zipformer-gigaspeech-3.3M-2024-01-01",
    "response_format": "kws.bool",
    "enoutput": true,
    "inputs": [ "sys.pcm" ], // リンクされた入力ソースの配列
    "kws": "HELLO M5,HEY STACK", // 使用中の有効なキーワード（生成されたファイルまたはデフォルトから）
    "enwake_audio": true
  }
}
```

> **タスクインスタンスの`work_id`に関する注意**: タスクインスタンスの`work_id`の数値部分（例：`kws.1000`の`1000`）は、`setup` APIによる作成時にシステムによって動的に割り当てられます。

## モデル設定ファイルとスクリプトとの関連性

`llm-kws`ユニットは、モデル固有のJSON設定ファイルとキーワード処理用のPythonスクリプトに依存しています。

### モデル設定JSON (例: `mode_sherpa-onnx-kws-zipformer-gigaspeech-3.3M-2024-01-01.json`)

-   **`mode`**: モデル設定の名前。
-   **`type`**: ユニットタイプ、例: `"kws"`。
-   **`capabilities`**: 機能の説明、例: `"Keyword_spotting"`, `"English"`。
-   **`mode_param`オブジェクト**: Sherpa-ONNX KWSエンジンの重要なパラメータが含まれています：
    -   `model_config.transducer.*`, `model_config.tokens`: ニューラルネットワークモデルコンポーネント（エンコーダ、デコーダ、ジョイナ）およびトークンリストへのパス。これらのパスはモデルのディレクトリからの相対パスです (例: `sherpa-onnx-kws-zipformer-gigaspeech-3.3M-2024-01-01/`)。
    -   `keywords_file`: `setup` APIで`kws`が指定されない場合に使用されるデフォルトのキーワードリストファイル（例: `"keywords.txt"`）。このファイルは`llm-kws_text2token.py`スクリプトの出力ターゲットでもあります。
    -   `feat_config.sample_rate`, `feat_config.feature_dim`: オーディオ特徴設定。
    -   `model_config.num_threads`: 計算用スレッド数。
    -   `keywords_score`, `keywords_threshold`: キーワード検出感度の調整パラメータ。
    -   `text2token-bpe-model`, `text2token-tokens-type`: キーワード処理のために`llm-kws_text2token.py`スクリプトに渡されるパラメータで、トークン化戦略（例: `cjkchar+bpe`, `ppinyin`）を定義します。
    -   `wake_wav_file`: `enwake_audio`が`true`でキーワードが検出されたときに再生される音声ファイルへのパス (例: `"/opt/m5stack/data/audio/wakeup_en_us.wav"`)。

### キーワード処理スクリプト (`llm-kws_text2token.py`)

-   **目的**: このPythonスクリプトは、`setup` APIの`kws`パラメータ（またはテキストファイル）で提供される人間が読めるキーワードを、Sherpa-ONNX KWSエンジンが理解できる形式（トークンID）に変換するために不可欠です。
-   **操作**:
    1.  `kws`パラメータを指定して`setup` APIが呼び出されると、`llm-kws`ユニットはこれらのキーワードを一時ファイル（例：`/tmp/kws_awake.txt.tmp`）に保存します。
    2.  次に、モデルのJSON設定で定義されたパスとパラメータ（例：Pythonインタプリタパス、スクリプトパス、`tokens.txt`へのパス、使用されていれば`bpe.model`へのパス、および処理された`keywords_file`の出力パス）を使用して`llm-kws_text2token.py`を呼び出します。
    3.  スクリプトは一時キーワードファイルを読み取り、モデルのトークン化タイプ（`text2token-tokens-type`）に基づいてキーワードを処理し、モデルのJSONで指定された`keywords_file`（例：`sherpa-onnx-kws-zipformer-gigaspeech-3.3M-2024-01-01/keywords.txt`）に出力を書き込みます。
    4.  KWSエンジンはこの生成された`keywords_file`をロードします。
-   **呼び出し例 (`text2token.txt`および`main.cpp`ロジックから派生)**:
    ```bash
    /usr/bin/python3 /opt/m5stack/scripts/llm-kws_text2token.py \
      --text /tmp/kws_awake.txt.tmp \
      --tokens <モデル固有のtokens.txtへのパス> \
      --tokens-type <モデルJSONからのタイプ> \
      --bpe-model <モデル固有のbpe.modelへのパス> \ # BPEを使用する場合
      --output <モデル固有のkeywords.txtへのパス>
    ```
    これは、特定のモデルがそのトークンリストと潜在的にBPEモデルを使用してキーワードを処理する方法を示しています。正確なパスは、ロードされたモデルの設定に基づいて動的に構築されます。

これらの関係を理解することは、カスタムキーワードの設定やKWSの動作のトラブルシューティングの鍵となります。`llm-kws`ユニットはこの複雑さの多くを抽象化しますが、基盤となるファイルが機能とデフォルトの動作を定義します。
```

Now, I will use `overwrite_file_with_block` to save this translated content, as `create_file_with_block` indicated the file already exists.
