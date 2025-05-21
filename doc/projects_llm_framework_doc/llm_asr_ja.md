# llm-asr (自動音声認識)

`llm-asr`ユニットは、StackFlowフレームワーク内で音声テキスト変換サービスを提供します。様々な音声認識モデル（例：中国語、英語）をロードし、音声入力をテキストに書き起こします。システムマイク入力や他の連携ユニット（キーワードスポッティング - KWSなど）からの音声受信をサポートしています。

## API呼び出しの共通規約

`llm-asr`へのAPI呼び出しはJSONメッセージを介して行われます。

### リクエスト構造

`llm-asr` APIへの標準的なリクエストは以下の構造に従います：

```json
{
  "request_id": "クライアント生成のUUIDまたはカウンター",
  "work_id": "asrまたはasr.インスタンスID",
  "action": "APIアクション名",
  "object": "オプションのオブジェクトタイプ文字列", // 該当する場合
  "data": { /* API固有のペイロード */ }     // またはlink/unlinkのような一部APIでは文字列
}
```

-   `request_id` (string, 必須): リクエストとレスポンスを関連付けるための、クライアントが生成する一意の識別子。
-   `work_id` (string, 必須):
    -   初期設定や一般的なクエリの場合: `"asr"`。
    -   特定のASRタスクインスタンスに対する操作の場合: `"asr.XXXX"` (例: `"asr.1001"`)。`XXXX`は成功した`setup`呼び出しによって返されるインスタンスIDです。
-   `action` (string, 必須): 呼び出すAPIアクション (例: `"setup"`, `"link"`)。
-   `object` (string, オプション): 送信するデータのタイプやコンテキストを指定します（該当する場合）。例：設定データの場合は`"asr.setup"`、リンク時は`"work_id"`。
-   `data` (object または string, オプション): API呼び出しのペイロード。構造はAPIによって異なります。

### レスポンス構造

**成功レスポンス:**

API呼び出しが受け入れられ、正常に処理されたことを示します。

```json
{
  "request_id": "リクエストからミラーリング",
  "work_id": "asrまたはasr.インスタンスID", // ミラーリングまたは更新 (例: setup後の "asr.1001")
  "action": "APIアクション名",        // リクエストからミラーリング
  "created": 1678886400,              // 整数: レスポンス生成のUnixタイムスタンプ
  "code": 0,                          // 整数: 0は成功を示す
  "message": "OK",                    // 文字列: 成功メッセージ
  "data": { /* API固有データ */ }    // オプション: APIによって返されるペイロード (例: タスク情報、新しいwork_id)
}
```
-   API呼び出しが新しいASRタスクインスタンスの作成に成功した場合（例：`setup`）、レスポンスの`work_id`は新しいインスタンスID（例：`"asr.1001"`）になります。
-   APIが特定の情報を返す場合、`data`フィールドが存在します。

**エラーレスポンス:**

API呼び出し処理中の失敗を示します。

```json
{
  "request_id": "リクエストからミラーリング",
  "work_id": "asrまたはasr.インスタンスID",
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
-   `-5`: モデルの読み込みに失敗しました（例：モデルファイルが見つからない、モデル設定が無効）。
-   `-6`: 指定されたASRタスクインスタンス（例：`"asr.1001"`）が存在しないか、アクティブではありません。
-   `-11`: 一般的なモデルランタイムエラーまたは処理中の内部エラー。
-   `-20`: 指定された入力ソースユニットへのリンクに失敗しました（例：KWSユニットIDが見つからないか無効）。
-   `-21`: 同時ASRタスクの最大数に達しました。
-   `-23`: Base64音声データのデコードエラー。
-   `-25`: ストリーム音声データのエラー（例：インデックスの不一致、解析エラー）。

## ASRデータ出力 (認識結果)

`llm-asr`ユニットは、`setup`呼び出し時に指定された`response_format`パラメータに基づいて、認識結果をクライアント（または接続されたZMQサブスクライバ）に送信します。結果は直接のAPIレスポンス経由ではなく、非同期にプッシュされます。

### 1. 完全な発話結果 (`response_format: "asr.utf-8"`)

`response_format`が`"asr.utf-8"`に設定されている場合、ASRユニットは音声アクティビティ検出が発話の終了を判断した後、認識された完全なテキストを単一のUTF-8文字列として送信します。

**出力フォーマット (直接文字列):**
例: `"これは認識された音声です。"`
(注意: ASRサービスは、最終的に認識された文字列に句点などの句読点を付加する場合があります。)

### 2. ストリーム形式の部分結果 (`response_format: "asr.utf-8.stream"`)

`response_format`が`"asr.utf-8.stream"`に設定されている場合、ASRユニットは部分的（中間）および最終的な認識結果を表すJSONオブジェクトのストリームを送信します。

**出力フォーマット (JSONオブジェクトストリーム):**
ストリーム内の各JSONオブジェクトは以下の構造を持ちます：

```json
{
  "index": 0,                     // 整数: セグメントのシーケンス番号、0から開始。新しい発話ごとにリセット。
  "delta": "認識されたセグメント",  // 文字列: 現在認識されているテキストセグメント。
  "finish": false                 // 真偽値: これが発話の最終セグメントである場合は`true`、それ以外は`false`。
}
```

-   **部分結果の例:**
    ```json
    {
      "index": 0,
      "delta": "これは",
      "finish": false
    }
    ```
    ```json
    {
      "index": 1,
      "delta": "これはテストです",
      "finish": false
    }
    ```
-   **最終結果の例 (ストリームの一部):**
    ```json
    {
      "index": 2,
      "delta": "これはテストです。", // 最終セグメントには句読点が含まれることが多い
      "finish": true
    }
    ```
    `finish: true`のメッセージの後、次の発話のために`index`は`0`にリセットされます。

## APIリファレンス

### **setup**

特定のモデルと操作パラメータで新しいASRタスクインスタンスを初期化し、設定します。

-   リクエストの`work_id`: `"asr"`
-   レスポンスの`work_id` (成功時): `"asr.XXXX"` (例: `"asr.1001"`)

**リクエストパラメータ:**

| パラメータ                                    | 型            | 必須 | デフォルト (モデルJSONより)          | 説明                                                                                                                                                                |
|-----------------------------------------------|---------------|------|------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `object`                                      | string        | はい | N/A                                | `asr.setup`である必要があります。                                                                                                                                      |
| `data.model`                                  | string        | はい | N/A                                | ASRモデル設定ファイルの名前（`.json`拡張子なし）。例: `"sherpa-ncnn-streaming-zipformer-20M-2023-02-17"`。                                                              |
| `data.response_format`                        | string        | はい | N/A                                | 認識結果の配信方法: `"asr.utf-8"` (単一文字列) または `"asr.utf-8.stream"` (JSONオブジェクトストリーム)。                                                                        |
| `data.input`                                  | string/array  | はい | N/A                                | 音声入力ソース。単一文字列 (例: `"sys.pcm"`) または文字列の配列 (例: `["sys.pcm", "kws.1001"]`)。`sys.pcm` は一般的なシステム音声入力を指します。                               |
| `data.enoutput`                               | boolean       | いいえ | `true`                             | ASR結果の送信を有効 (`true`) または無効 (`false`) にします。                                                                                                                   |
| `data.awake_delay`                            | integer       | いいえ | 50 (ms)                            | KWSウェイクアップイベント後、ASR処理を開始するまでの遅延時間（ミリ秒）。KWSユニットにリンクされている場合に関連します。                                                                   |
| `data.endpoint_config.rule1.min_trailing_silence` | float         | いいえ | 2.4 (s)                            | ウェイクアップ後、発話がない場合にエンドポイントをトリガーする無音時間。                                                                                                              |
| `data.endpoint_config.rule2.min_trailing_silence` | float         | いいえ | 1.2 (s)                            | 音声認識後にエンドポイントをトリガーする無音時間。                                                                                                                  |
| `data.endpoint_config.rule3.min_utterance_length` | float         | いいえ | 30.0 (s)                           | エンドポイントを強制する前の単一発話の最大長。                                                                                                       |
| `data.endpoint_config.rule[1-3].must_contain_nonsilence` | boolean    | いいえ | モデルJSONによる                | `true`の場合、それぞれのルールは無音でない音声が検出された場合にのみ適用されます。                                                                                                  |
| `data.hotwords`                               | string        | いいえ | "" (空)                            | 特定の単語の認識を強化するためのホットワード文字列。フォーマットはモデルに依存します。                                                                                                |
| `data.feat_config.sampling_rate`              | integer       | いいえ | 16000                              | 想定される音声サンプルレート。モデルのデフォルトと異なる場合、リサンプリングが必要になるか、エラーが発生する可能性があります。                                                                      |
| ... (その他のモデルパラメータ)                  | various       | いいえ | (モデルJSONより)                   | その他のモデル固有パラメータ（例：`num_threads`）は、ASRエンジンが設定構造経由でサポートしていれば上書きできる場合があります。利用可能なオプションについてはモデルJSONを参照してください。                  |

**リクエスト例:**
```json
{
  "request_id": "setup_asr_001",
  "work_id": "asr",
  "action": "setup",
  "object": "asr.setup",
  "data": {
    "model": "sherpa-ncnn-streaming-zipformer-20M-2023-02-17",
    "response_format": "asr.utf-8.stream",
    "input": ["sys.pcm", "kws.1001"],
    "enoutput": true,
    "awake_delay": 100,
    "endpoint_config.rule2.min_trailing_silence": 1.5
  }
}
```

**レスポンス (成功):**
```json
{
  "request_id": "setup_asr_001",
  "work_id": "asr.1001", // 新しいASRタスクインスタンスID
  "action": "setup",
  "created": 1678886400,
  "code": 0,
  "message": "OK",
  "data": {
    "work_id": "asr.1001" // 作成されたインスタンスIDを確認
  }
}
```

**レスポンス (エラー - モデル読み込み失敗):**
```json
{
  "request_id": "setup_asr_002",
  "work_id": "asr",
  "action": "setup",
  "created": 1678886401,
  "error": {
    "code": -5,
    "message": "モデルの読み込みに失敗しました。"
  }
}
```

### **link**

ASRタスクインスタンスを音声ソース（音声データを生成する別のユニット、例：KWS）にリンクします。これは、`setup`時に音声ソースを指定する代わりの方法、または動的に音声ソースを追加する方法です。

-   リクエストの`work_id`: `"asr.XXXX"` (特定のASRタスクインスタンス)

**リクエストパラメータ:**

| パラメータ | 型     | 必須 | デフォルト | 説明                                                                       |
|-----------|--------|------|------------|----------------------------------------------------------------------------|
| `object`  | string | はい | N/A        | 通常は`"work_id"`で、dataフィールドにリンクするユニットIDが含まれることを示します。 |
| `data`    | string | はい | N/A        | 入力ソースとしてリンクするユニットの`work_id` (例: `"kws.1000"`)。             |

**リクエスト例:**
```json
{
  "request_id": "link_asr_003",
  "work_id": "asr.1001",
  "action": "link",
  "object": "work_id",
  "data": "kws.1000"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "link_asr_003",
  "work_id": "asr.1001",
  "action": "link",
  "created": 1678886402,
  "code": 0,
  "message": "OK"
}
```

**レスポンス (エラー - タスクが見つからない):**
```json
{
  "request_id": "link_asr_004",
  "work_id": "asr.9999", // 存在しないタスク
  "action": "link",
  "created": 1678886403,
  "error": {
    "code": -6,
    "message": "ユニットが存在しません"
  }
}
```

### **unlink**

以前にリンクされた音声ソースをASRタスクインスタンスから解除します。

-   リクエストの`work_id`: `"asr.XXXX"`

**リクエストパラメータ:**

| パラメータ | 型     | 必須 | デフォルト | 説明                                                                           |
|-----------|--------|------|------------|--------------------------------------------------------------------------------|
| `object`  | string | はい | N/A        | 通常は`"work_id"`で、dataフィールドに解除するユニットIDが含まれることを示します。     |
| `data`    | string | はい | N/A        | 解除する入力ソースユニットの`work_id` (例: `"kws.1000"`)。                 |

**リクエスト例:**
```json
{
  "request_id": "unlink_asr_005",
  "work_id": "asr.1001",
  "action": "unlink",
  "object": "work_id",
  "data": "kws.1000"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "unlink_asr_005",
  "work_id": "asr.1001",
  "action": "unlink",
  "created": 1678886404,
  "code": 0,
  "message": "OK"
}
```

### **pause**

ASRタスクインスタンスを一時停止します。一時停止中は、音声を処理したり認識結果を生成したりしません。KWSにリンクされたタスクの場合、ASRタスクの`ensleep_`フラグがtrue（KWSへのリンク時に内部的に設定）であれば、通常、発話後に自動的にこれが発生します。

-   リクエストの`work_id`: `"asr.XXXX"`

**リクエストパラメータ**: 標準の`request_id`, `work_id`, `action`以外はありません。

**リクエスト例:**
```json
{
  "request_id": "pause_asr_006",
  "work_id": "asr.1001",
  "action": "pause"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "pause_asr_006",
  "work_id": "asr.1001",
  "action": "pause",
  "created": 1678886405,
  "code": 0,
  "message": "OK"
}
```

### **work**

一時停止したASRタスクインスタンスを再開します。KWSにリンクされている場合、これは通常、KWSウェイクアップイベント（`awake_delay`後）によってトリガーされます。手動で`work`を呼び出すことで、手動で一時停止されたタスクや発話後にスリープ状態になったタスクを再開できます。

-   リクエストの`work_id`: `"asr.XXXX"`

**リクエストパラメータ**: 標準の`request_id`, `work_id`, `action`以外はありません。

**リクエスト例:**
```json
{
  "request_id": "work_asr_007",
  "work_id": "asr.1001",
  "action": "work"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "work_asr_007",
  "work_id": "asr.1001",
  "action": "work",
  "created": 1678886406,
  "code": 0,
  "message": "OK"
}
```

### **exit**

ASRタスクインスタンスを停止し、削除してリソースを解放します（他のタスクが使用していない場合はロードされたモデルも含む）。

-   リクエストの`work_id`: `"asr.XXXX"`

**リクエストパラメータ**: 標準の`request_id`, `work_id`, `action`以外はありません。

**リクエスト例:**
```json
{
  "request_id": "exit_asr_008",
  "work_id": "asr.1001",
  "action": "exit"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "exit_asr_008",
  "work_id": "asr.1001", // 終了したタスクのwork_id
  "action": "exit",
  "created": 1678886407,
  "code": 0,
  "message": "OK"
}
```

### **taskinfo**

アクティブなASRタスクに関する情報を取得します。動作はリクエストの`work_id`に依存します。

**ケース1: 一般的なASRユニット情報**

-   リクエストの`work_id`: `"asr"`
-   **説明**: 全てのアクティブなASRタスクインスタンスIDのリストを返します。

**リクエスト例:**
```json
{
  "request_id": "taskinfo_asr_general_009",
  "work_id": "asr",
  "action": "taskinfo"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "taskinfo_asr_general_009",
  "work_id": "asr",
  "action": "taskinfo",
  "created": 1678886408,
  "code": 0,
  "message": "OK",
  "object": "asr.tasklist",
  "data": [
    "asr.1001",
    "asr.1002"
  ]
}
```

**ケース2: 特定のASRタスクインスタンス情報**

-   リクエストの`work_id`: `"asr.XXXX"` (例: `"asr.1001"`)
-   **説明**: 指定されたASRタスクインスタンスのランタイムパラメータと設定を返します。

**リクエスト例:**
```json
{
  "request_id": "taskinfo_asr_specific_010",
  "work_id": "asr.1001",
  "action": "taskinfo"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "taskinfo_asr_specific_010",
  "work_id": "asr.1001",
  "action": "taskinfo",
  "created": 1678886409,
  "code": 0,
  "message": "OK",
  "object": "asr.taskinfo",
  "data": {
    "model": "sherpa-ncnn-streaming-zipformer-20M-2023-02-17",
    "response_format": "asr.utf-8.stream",
    "enoutput": true,
    "inputs": [ "sys.pcm", "kws.1001" ] // リンクされた入力ソースの配列
  }
}
```

**レスポンス (エラー - タスクが見つからない):**
```json
{
  "request_id": "taskinfo_asr_specific_011",
  "work_id": "asr.9999", // 存在しないタスク
  "action": "taskinfo",
  "created": 1678886410,
  "error": {
    "code": -6,
    "message": "ユニットが存在しません"
  }
}
```

> **タスクインスタンスの`work_id`に関する注意**: ASRタスクインスタンスの`work_id`の数値部分（例：`asr.1001`の`1001`）は、`setup` APIによる作成時にシステムによって動的に割り当てられます。これは固定インデックスではなく、新しいタスクごとに増分します。

## モデル設定ファイルとの関連性

`llm-asr`ユニットは、コアとなる音声認識エンジンのパラメータについて、モデル固有のJSON設定ファイルに依存しています。これらのファイルは通常、ベースモデルパス（例：`llm-sys`で使用される`/opt/m5stack/data/models/`）およびベースモデル設定パス（例：`/opt/m5stack/etc/`）の下に構造化されたディレクトリに配置されます。システムはこれらのパスを検索して指定されたモデルファイルを見つけます。

`setup` APIでモデルを指定すると（例：`"sherpa-ncnn-streaming-zipformer-20M-2023-02-17"`）、システムは対応するJSONファイル（例：`mode_sherpa-ncnn-streaming-zipformer-20M-2023-02-17.json`）を探します。

これらのモデル設定ファイルの主な側面：
-   **`mode_param`オブジェクト**: モデルJSON内のこのオブジェクトには、Sherpa-NCNN ASRエンジンの主要なパラメータが含まれています。例：
    -   `model_config.*`: エンコーダ、デコーダ、ジョイナモデルのバイナリおよびトークンファイルへのパス（モデルのディレクトリからの相対パス）。
    -   `feat_config.*`: `sampling_rate`（例：16000）や`feature_dim`（例：80）などの音声特徴設定。
    -   `endpoint_config.*`: `min_trailing_silence`、`min_utterance_length`、`must_contain_nonsilence`などの音声アクティビティ検出（VAD）ルールのパラメータ。
    -   `decoder_config.*`: デコード方法（例：`"greedy_search"`）、`num_active_paths`。
    -   `model_config.encoder_opt.num_threads`, `model_config.decoder_opt.num_threads`, `model_config.joiner_opt.num_threads`: モデルの異なる部分のスレッド数。
    -   `hotwords_file`, `hotwords_score`: カスタムホットワード強調用。
    -   `awake_delay`: KWSアクティベーション後のASR処理開始までのデフォルト遅延時間（ミリ秒）。
-   **デフォルト値**: `mode_param`内の多くのパラメータは、`setup` APIを使用してASRタスクが作成される際のデフォルト値として機能します。
-   **デフォルト値の上書き**: これらのデフォルトの一部は、`setup` API呼び出しの`data`オブジェクトに対応するパラメータを提供することで上書きできます。例えば、`endpoint_config.ruleX`パラメータ、`awake_delay`、`feat_config.sampling_rate`は、`setup`リクエストに含まれていればASRタスクインスタンスごとにカスタマイズできます。
-   **モデルファイルの場所**: 実際のニューラルネットワークモデルファイル（`.ncnn.param`、`.ncnn.bin`）およびトークンファイル（`tokens.txt`）は、通常、モデルのJSON設定ファイルと同じ場所、多くの場合モデル名のサブディレクトリ（例：`sherpa-ncnn-streaming-zipformer-20M-2023-02-17/`）に配置されます。

ユーザーは通常、事前に設定されたモデルを名前で選択します。上級ユーザーは、ASRのパフォーマンスを調整したり新しいモデルを統合したりするために、これらのモデルJSONファイルをカスタマイズしたり新規作成したりすることがありますが、これはAPIの使用範囲外です。
