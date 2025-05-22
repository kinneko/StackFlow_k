# llm-melotts (MeloTTS テキスト音声合成)

`llm-melotts`ユニットは、一部モデルでNPUアクセラレーションを活用したテキスト音声合成（TTS）サービスを提供します。入力されたテキストを可聴音声に変換し、英語、日本語、中国語など、さまざまな言語のモデルをサポートしています。

## API呼び出しの共通規約

`llm-melotts`へのAPI呼び出しはJSONメッセージを介して行われます。

### リクエスト構造

`llm-melotts` APIへの標準的なリクエストは以下の構造に従います：

```json
{
  "request_id": "クライアント生成のUUIDまたはカウンター",
  "work_id": "melottsまたはmelotts.インスタンスID",
  "action": "APIアクション名",
  "object": "オプションのオブジェクトタイプ文字列", // 該当する場合
  "data": { /* API固有のペイロード */ }     // または一部APIでは文字列
}
```

-   `request_id` (string, 必須): リクエストとレスポンスを関連付けるための、クライアントが生成する一意の識別子。
-   `work_id` (string, 必須):
    -   初期設定や一般的なクエリの場合: `"melotts"`。
    -   特定のTTSタスクインスタンスに対する操作の場合: `"melotts.XXXX"` (例: `"melotts.1003"`)。`XXXX`は成功した`setup`呼び出しによって返されるインスタンスIDです。
-   `action` (string, 必須): 呼び出すAPIアクション (例: `"setup"`, `"inference"`)。
-   `object` (string, オプション): 送信するデータのタイプやコンテキストを指定します（該当する場合）。例：設定データの場合は`"melotts.setup"`、テキストデータの場合は`"melotts.utf-8"`。
-   `data` (object または string, オプション): API呼び出しのペイロード。

### レスポンス構造

**成功レスポンス (ほとんどのAPI呼び出し):**

API呼び出しが受け入れられ、正常に処理されたことを示します。

```json
{
  "request_id": "リクエストからミラーリング",
  "work_id": "melottsまたはmelotts.インスタンスID", // ミラーリングまたは更新
  "action": "APIアクション名",                // リクエストからミラーリング
  "created": 1678886400,                      // 整数: レスポンス生成のUnixタイムスタンプ
  "code": 0,                                  // 整数: 0は成功を示す
  "message": "OK",                            // 文字列: 成功メッセージ
  "data": { /* API固有データ */ }         // オプション: APIによって返されるペイロード
}
```
-   `inference` APIの場合、この同期レスポンスはリクエストの受領を確認するだけです。実際の合成音声は非同期に送信されるか再生されます。
-   API呼び出しが新しいタスクインスタンスの作成に成功した場合（例：`setup`）、レスポンスの`work_id`および`data.work_id`は新しいインスタンスIDになります。

**エラーレスポンス:**

API呼び出し処理中の失敗を示します。

```json
{
  "request_id": "リクエストからミラーリング",
  "work_id": "melottsまたはmelotts.インスタンスID",
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
-   `-3`: TTSエンジンまたはNPUの初期化に失敗しました。
-   `-5`: モデルの読み込みに失敗しました（例：モデルファイル、レキシコン、または音素データが見つからない）。
-   `-6`: 指定されたTTSタスクインスタンス（例：`"melotts.1003"`）が存在しません。
-   `-11`: 音声合成中または内部処理中の一般的なエラー。
-   `-20`: 入力ソースのリンクまたはデータ購読に関するエラー。
-   `-21`: タスク制限に達しました（通常、MeloTTSインスタンスは1つのみサポートされます）。
-   `-23`: 入力テキストの処理エラー（例：テキストが長すぎる、トークン化/音素化の問題）。
-   `-25`: `llm_audio`ユニット経由での音声再生エラー（`enaudio`がtrueの場合）。

## データ入出力 (テキストと合成音声)

### 入力データ (合成用テキスト)

`llm-melotts`ユニットは、いくつかの方法で合成用のテキストを受信できます：

1.  **特定の`melotts.XXXX`インスタンスへの`inference` API経由:**
    *   **非ストリーミングテキスト (`object: "melotts.utf-8"`)**:
        -   `data`フィールドは、合成する完全なテキストを含む単一の文字列です。
    *   **ストリーミングテキスト (`object: "melotts.utf-8.stream"`)**:
        -   `data`フィールドはJSONオブジェクトです: `{"index": <整数>, "delta": "<テキストチャンク>", "finish": <真偽値>}`。
        -   テキストは内部的に蓄積されます。音声セグメントの合成は通常、蓄積されたテキスト内で句読点（コンマ、ピリオド、感嘆符、疑問符など）が検出されたとき、または`finish`が`true`のときに行われます。

2.  **リンクされたユニット経由 (`setup` APIの`input`パラメータで設定):**
    *   **`"tts.utf-8"` または `"tts.utf-8.stream"`**: インスタンスは、上記のように`inference` APIを使用して自身の`work_id`に送信されたテキストをリッスンします。(注意: 現在のドキュメントでは直接入力に`tts.utf-8`を使用していますが、これは`melotts`ユニット自体を対象としていると理解してください。)
    *   **`"llm.XXXX"` または `"vlm.XXXX"`**: MeloTTSインスタンスは、指定されたLLMまたはVLMユニットの出力を購読します。これらのユニットによって生成されたテキストは、句読点を尊重してセグメント化され、合成のための入力として処理されます。
    *   **`"kws.XXXX"`**: MeloTTSインスタンスは、KWSユニットからのイベントを購読します。キーワードが検出されると、`main.cpp`の`kws_awake`関数がトリガーされます。この関数は、以前にリンクされたLLM/VLM (`superior_id_`) からの進行中のTTS再生を停止し、その後そのLLM/VLMに再購読します。これは、KWSイベントデータ自体を合成するのではなく、割り込みと再開のフローを意味します。

### 出力データ (合成音声)

合成された音声は、主に2つの方法で処理できます：

1.  **`llm_audio`経由の直接再生 (`response_format`に`"sys.pcm"`または`"sys"`が含まれ、かつ `enaudio: true`の場合)**
    -   `response_format`が`"sys.pcm"`（または"sys"を含む任意のフォーマット文字列）に設定され、かつ`enaudio`が`true`（リクエストで指定されていないかtrueの場合、`main.cpp`ではデフォルトでtrue）の場合、生成されたPCMオーディオデータはセグメントごとに`llm_audio`ユニットに送信され、即時再生されます（`audio->queue_play`を使用）。
    -   これは、直接的な聴覚フィードバックのための主要なモードです。

2.  **オーディオデータのAPIプッシュ (`enoutput: true`の場合)**
    -   `enoutput`が`true`の場合、合成されたオーディオデータ（PCM）はBase64エンコードされ、タスクを開始したクライアントに非同期にプッシュされます。
    -   **ストリーミング出力 (`response_format: "tts.pcm.base64.stream"`)**:
        -   音声は合成されるとチャンク単位で送信されます。
        -   チャンクごとのJSON構造:
            ```json
            {
              "request_id": "<元のinferenceリクエストID>",
              "work_id": "melotts.XXXX",
              "action": "tts_audio_chunk", // 慣例的なアクション名の例
              "object": "tts.pcm.base64.stream",
              "created": 1678886410,
              "code": 0,
              "message": "OK",
              "data": {
                "index": 0, // このオーディオチャンクのシーケンス番号
                "delta": "<Base64エンコードされたPCMオーディオチャンク>",
                "finish": false // 合成タスク全体の最後のチャンクの場合はtrue
              }
            }
            ```
    -   **非ストリーミング出力 (`response_format: "tts.pcm.base64"`)**:
        -   完全な入力テキスト（または`finish: true`の最終チャンク）が処理された後、合成された音声全体が単一のBase64エンコード文字列として送信されます。
        -   JSON構造:
            ```json
            {
              "request_id": "<元のinferenceリクエストID>",
              "work_id": "melotts.XXXX",
              "action": "tts_audio_full", // 慣例的なアクション名の例
              "object": "tts.pcm.base64",
              "created": 1678886415,
              "code": 0,
              "message": "OK",
              "data": "<完全なBase64エンコードされたPCMオーディオ>"
            }
            ```
    -   これらの非同期メッセージの`request_id`は、テキストを提供した`inference` API呼び出しの`request_id`に対応します。

## APIリファレンス

### **setup**

新しいMeloTTSタスクインスタンスを初期化し、設定します。

-   リクエストの`work_id`: `"melotts"`
-   レスポンスの`work_id` (成功時): `"melotts.XXXX"` (例: `"melotts.1003"`)

**リクエストパラメータ:**

| パラメータ             | 型            | 必須 | デフォルト (モデルJSONより該当する場合) | 説明                                                                                                                                                                                                                                                           |
|------------------------|---------------|------|-----------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `object`               | string        | はい | N/A                                     | `melotts.setup`である必要があります。                                                                                                                                                                                                                  |
| `data.model`           | string        | はい | N/A                                     | MeloTTSモデル設定ファイルの名前（`.json`拡張子なし）。例: `"melotts-en-us"`, `"melotts-ja-jp"`, `"melotts-zh-cn"`。モデルは`projects/llm_framework/main_melotts/models/`に配置されます。             |
| `data.response_format` | string        | はい | N/A                                     | 音声出力の処理方法を定義します。`"sys.pcm"`（または"sys"を含む任意のフォーマット）は、`enaudio`がtrueの場合に`llm_audio`経由の直接再生を有効にします。`"tts.pcm.base64"`や`"tts.pcm.base64.stream"`のようなフォーマットは、`enoutput`もtrueの場合にオーディオデータのAPIプッシュを有効にします。 |
| `data.input`           | string/array  | はい | N/A                                     | テキストの入力ソース。例: `"tts.utf-8"`（このインスタンスへの直接API入力用）、`"llm.1002"`（LLMの出力へリンクするため）、`"kws.1000"`。「入力データ」セクションを参照してください。                                  |
| `data.enoutput`        | boolean       | いいえ | `false`                                 | `true`の場合、`response_format`に従ってBase64エンコードされたPCMオーディオデータのクライアントへの非同期プッシュを有効にします。                                                                                                                            |
| `data.enaudio`         | boolean       | いいえ | `true`                                  | `true`で、かつ`response_format`がシステムオーディオ（例："sys.pcm"）を示す場合、`llm_audio`経由で合成音声を再生します。`false`の場合、`response_format`が"sys.pcm"であっても音声は再生されません。                                           |
| `data.mode_param`      | object        | いいえ | {}                                      | モデルのJSONから特定のパラメータ（例：`{"spacker_speed": 1.2}`）を上書きできます。サポートされる上書きは`main.cpp`の実装に依存します（現在、`spacker_speed`、`audio_rate`は主にエンジン初期化時にモデルJSONからロードされます）。 |

**リクエスト例:**
```json
{
  "request_id": "setup_melotts_001",
  "work_id": "melotts",
  "action": "setup",
  "object": "melotts.setup",
  "data": {
    "model": "melotts-en-us",
    "response_format": "sys.pcm", // 音声を直接再生
    "input": "tts.utf-8",
    "enoutput": true, // API呼び出し元にもPCMデータを送信 (response_formatがtts.pcm.*の場合)
    "enaudio": true,   // 再生を有効化
    "mode_param": {
      "spacker_speed": 1.1 // モデルパラメータ上書きの試行例
    }
  }
}
```

**レスポンス (成功):**
```json
{
  "request_id": "setup_melotts_001",
  "work_id": "melotts.1003", // 新しいTTSタスクインスタンスID
  "action": "setup",
  "created": 1678886400,
  "code": 0,
  "message": "OK",
  "data": {
    "work_id": "melotts.1003" // 作成されたインスタンスIDを確認
  }
}
```

### **inference**

特定のMeloTTSタスクインスタンスに音声合成のためのテキストを送信します。音声出力は`setup`設定に基づいて非同期に処理されます。

-   リクエストの`work_id`: `"melotts.XXXX"` (特定のMeloTTSタスクインスタンスID)

**リクエストパラメータ:**

| パラメータ | 型            | 必須 | デフォルト | 説明                                                                                                                               |
|-----------|---------------|------|------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| `object`  | string        | はい | N/A        | 入力テキストのフォーマットを指定します: `"melotts.utf-8"` (単一の完全なテキストの場合) または `"melotts.utf-8.stream"` (チャンク化されたテキストの場合)。 |
| `data`    | string/object | はい | N/A        | `object`が`"melotts.utf-8"`の場合、`data`は文字列です。`object`が`"melotts.utf-8.stream"`の場合、`data`はオブジェクトです: `{"index": <整数>, "delta": "<テキストチャンク>", "finish": <真偽値>}`。 |

**リクエスト例 (非ストリーミング):**
```json
{
  "request_id": "infer_melotts_002",
  "work_id": "melotts.1003",
  "action": "inference",
  "object": "melotts.utf-8",
  "data": "こんにちは、これはテストです。"
}
```

**レスポンス (成功 - リクエスト受付):**
このレスポンスは、テキストが合成のために受信されたことを確認するだけです。
```json
{
  "request_id": "infer_melotts_002",
  "work_id": "melotts.1003",
  "action": "inference",
  "created": 1678886402,
  "code": 0,
  "message": "OK"
}
```

### **link**

MeloTTSタスクインスタンスを入力ソースに動的にリンクします。

-   リクエストの`work_id`: `"melotts.XXXX"`

**リクエストパラメータ:**

| パラメータ | 型     | 必須 | デフォルト | 説明                                                                       |
|-----------|--------|------|------------|----------------------------------------------------------------------------|
| `object`  | string | はい | N/A        | 通常は`"work_id"`で、dataフィールドにリンクするユニットIDが含まれることを示します。 |
| `data`    | string | はい | N/A        | 入力ソースとしてリンクするユニットの`work_id` (例: `"llm.1002"`)。             |

**リクエスト例:**
```json
{
  "request_id": "link_melotts_003",
  "work_id": "melotts.1003",
  "action": "link",
  "object": "work_id",
  "data": "llm.1002"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "link_melotts_003",
  "work_id": "melotts.1003",
  "action": "link",
  "created": 1678886404,
  "code": 0,
  "message": "OK"
}
```

### **unlink**

以前にリンクされた入力ソースをMeloTTSタスクインスタンスから解除します。

-   リクエストの`work_id`: `"melotts.XXXX"`

**リクエストパラメータ:**

| パラメータ | 型     | 必須 | デフォルト | 説明                                                                           |
|-----------|--------|------|------------|--------------------------------------------------------------------------------|
| `object`  | string | はい | N/A        | 通常は`"work_id"`で、dataフィールドに解除するユニットIDが含まれることを示します。     |
| `data`    | string | はい | N/A        | 解除する入力ソースユニットの`work_id` (例: `"llm.1002"`)。                 |

**リクエスト例:**
```json
{
  "request_id": "unlink_melotts_004",
  "work_id": "melotts.1003",
  "action": "unlink",
  "object": "work_id",
  "data": "llm.1002"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "unlink_melotts_004",
  "work_id": "melotts.1003",
  "action": "unlink",
  "created": 1678886405,
  "code": 0,
  "message": "OK"
}
```

### **pause**

MeloTTSタスクインスタンスを一時停止します。これは通常、さらなる音声生成と再生を停止します。

-   リクエストの`work_id`: `"melotts.XXXX"`

**リクエスト例:**
```json
{
  "request_id": "pause_melotts_005",
  "work_id": "melotts.1003",
  "action": "pause"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "pause_melotts_005",
  "work_id": "melotts.1003",
  "action": "pause",
  "created": 1678886406,
  "code": 0,
  "message": "OK"
}
```

### **work**

一時停止したMeloTTSタスクインスタンスを再開します。

-   リクエストの`work_id`: `"melotts.XXXX"`

**リクエスト例:**
```json
{
  "request_id": "work_melotts_006",
  "work_id": "melotts.1003",
  "action": "work"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "work_melotts_006",
  "work_id": "melotts.1003",
  "action": "work",
  "created": 1678886407,
  "code": 0,
  "message": "OK"
}
```

### **exit**

MeloTTSタスクインスタンスを停止し、削除してリソースを解放します。

-   リクエストの`work_id`: `"melotts.XXXX"`

**リクエスト例:**
```json
{
  "request_id": "exit_melotts_007",
  "work_id": "melotts.1003",
  "action": "exit"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "exit_melotts_007",
  "work_id": "melotts.1003",
  "action": "exit",
  "created": 1678886408,
  "code": 0,
  "message": "OK"
}
```

### **taskinfo**

アクティブなMeloTTSタスクに関する情報を取得します。

**ケース1: 一般的なユニット情報**

-   リクエストの`work_id`: `"melotts"`
-   **説明**: 全てのアクティブなタスクインスタンスIDのリストを返します。

**リクエスト例:**
```json
{
  "request_id": "taskinfo_melotts_general_008",
  "work_id": "melotts",
  "action": "taskinfo"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "taskinfo_melotts_general_008",
  "work_id": "melotts",
  "action": "taskinfo",
  "created": 1678886409,
  "code": 0,
  "message": "OK",
  "object": "melotts.tasklist",
  "data": [
    "melotts.1003"
  ]
}
```

**ケース2: 特定のタスクインスタンス情報**

-   リクエストの`work_id`: `"melotts.XXXX"`
-   **説明**: 指定されたタスクインスタンスのランタイムパラメータを返します。

**リクエスト例:**
```json
{
  "request_id": "taskinfo_melotts_specific_009",
  "work_id": "melotts.1003",
  "action": "taskinfo"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "taskinfo_melotts_specific_009",
  "work_id": "melotts.1003",
  "action": "taskinfo",
  "created": 1678886410,
  "code": 0,
  "message": "OK",
  "object": "melotts.taskinfo",
  "data": {
    "model": "melotts-en-us",
    "response_format": "sys.pcm",
    "enoutput": false, // APIプッシュが有効かどうかを反映
    "enaudio": true,   // 直接再生が有効かどうかを反映
    "inputs": ["tts.utf-8"]
    // 注意: speaker_speed, audio_rateなどはモデルの内部設定の一部であり、
    // main.cppで明示的に追加されない限り、通常動的taskinfoにはリストされません。
  }
}
```

## モデル設定ファイルとの関連性

`llm-melotts`ユニットは、モデル固有のJSON設定ファイル（例：`mode_melotts-en-us.json`）に依存しており、これらは`projects/llm_framework/main_melotts/models/`ディレクトリまたはシステム全体のモデルパスに配置されています。

これらのモデル設定ファイルの主な側面：
-   **`mode`**: `setup` APIで使用されるモデル名（例：`"melotts-en-us"`）。
-   **`type`**: ユニットタイプ、通常は`"tts"`。
-   **`capabilities`**: 機能の説明、例：`"tts"`, `"English"`。
-   **`mode_param`オブジェクト**: MeloTTSエンジンの主要なパラメータが含まれています：
    -   `encoder`, `decoder`: テキストエンコードおよびオーディオデコード用のONNXモデルファイルへのパス（例：`"encoder-en.ort"`, `"decoder-en.axmodel"`）。これらはモデルのディレクトリからの相対パスです。
    -   `gbin`: スピーカー埋め込みファイルへのパス（例：`"g-en.bin"`）。
    -   `tokens`: 文字/トークンマッピングファイルへのパス（例：`"tokens.txt"`）。
    -   `lexicon`: 音素化のためのレキシコンファイルへのパス（例：`"lexicon.txt"`）。
    -   `spacker_speed` (float): デフォルトの読み上げ速度（例：`1.0`）。
    -   `mode_rate` (integer): モデルの内部音響表現のサンプルレート（例：`44100`）。
    -   `audio_rate` (integer): 出力オーディオのターゲットサンプルレート（例：`16000`）。これが`mode_rate`と異なる場合、リサンプリングが実行されます。
    -   `awake_delay` (integer): KWSと連携する際に使用される遅延時間（ミリ秒）。古いオーディオを処理しないようにするため。デフォルト1000ms。
    -   `noise_scale`, `length_scale`, `sdp_ratio`などの他のパラメータは、音声合成の品質の側面を制御します。

これらのパラメータは、特定のモデルでタスクがセットアップされるときにロードされます。`setup` APIは`mode_param`フィールドを介してこれらを上書きする可能性がありますが、現在の`main.cpp`は主にタスク初期化中にモデルのJSONファイルからこれらの値をロードします。

## 重要事項
-   **同時実行**: 複数のTTSユニットインスタンス（例：2つの`llm-melotts`インスタンス、または`llm-melotts`と`llm_tts`インスタンス）を同時に実行することは、NPUやオーディオ出力チャネルなどのハードウェアリソースを競合させる可能性があるため、一般的に推奨されません。エラーや予期しない動作を引き起こす可能性があります。
-   **音声再生**: `enaudio: true`で、かつ`response_format`がシステムオーディオ（例："sys.pcm"）を示す場合、`llm-melotts`ユニットは再生のために`llm_audio`ユニットに依存します。`llm_audio`が機能していることを確認してください。
-   **テキストセグメンテーション**: ユニットは入力テキストをセグメント単位で処理します。通常、句読点（コンマ、ピリオド、疑問符、感嘆符など）で区切られます。これにより、より長いテキストの応答性の高い合成が可能になります。
```

Now, I will use `overwrite_file_with_block` to save this translated content, as `create_file_with_block` indicated the file already exists and this is the safer option for updating.
