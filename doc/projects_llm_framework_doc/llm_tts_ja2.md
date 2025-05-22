# llm-tts (テキスト音声合成)

`llm-tts`ユニットは、テキスト音声合成（TTS）サービスを提供し、入力されたテキストを可聴音声に変換します。中国語や英語のモデルを含む様々なモデルをサポートしています。このユニットは`llm-melotts`とは異なり、異なる基盤となるTTSエンジン（例：NPUアクセラレーションなしのエンジン）を使用する場合があります。

## API呼び出しの共通規約

`llm-tts`へのAPI呼び出しはJSONメッセージを介して行われます。

### リクエスト構造

`llm-tts` APIへの標準的なリクエストは以下の構造に従います：

```json
{
  "request_id": "クライアント生成のUUIDまたはカウンター",
  "work_id": "ttsまたはtts.インスタンスID",
  "action": "APIアクション名",
  "object": "オプションのオブジェクトタイプ文字列", // 該当する場合
  "data": { /* API固有のペイロード */ }     // または一部APIでは文字列
}
```

-   `request_id` (string, 必須): リクエストとレスポンスを関連付けるための、クライアントが生成する一意の識別子。
-   `work_id` (string, 必須):
    -   初期設定や一般的なクエリの場合: `"tts"`。
    -   特定のTTSタスクインスタンスに対する操作の場合: `"tts.XXXX"` (例: `"tts.1003"`)。`XXXX`は成功した`setup`呼び出しによって返されるインスタンスIDです。
-   `action` (string, 必須): 呼び出すAPIアクション (例: `"setup"`, `"inference"`)。
-   `object` (string, オプション): 送信するデータのタイプやコンテキストを指定します（該当する場合）。例：設定データの場合は`"tts.setup"`、テキストデータの場合は`"tts.utf-8"`。
-   `data` (object または string, オプション): API呼び出しのペイロード。

### レスポンス構造

**成功レスポンス (ほとんどのAPI呼び出し):**

API呼び出しが受け入れられ、正常に処理されたことを示します。

```json
{
  "request_id": "リクエストからミラーリング",
  "work_id": "ttsまたはtts.インスタンスID", // ミラーリングまたは更新
  "action": "APIアクション名",        // リクエストからミラーリング
  "created": 1678886400,              // 整数: レスポンス生成のUnixタイムスタンプ
  "code": 0,                          // 整数: 0は成功を示す
  "message": "OK",                    // 文字列: 成功メッセージ
  "data": { /* API固有データ */ } // オプション: APIによって返されるペイロード
}
```
-   `inference` APIの場合、この同期レスポンスはリクエストの受領を確認するだけです。実際の合成音声は非同期に送信されるか再生されます。
-   API呼び出しが新しいタスクインスタンスの作成に成功した場合（例：`setup`）、レスポンスの`work_id`および`data.work_id`は新しいインスタンスIDになります。

**エラーレスポンス:**

API呼び出し処理中の失敗を示します。

```json
{
  "request_id": "リクエストからミラーリング",
  "work_id": "ttsまたはtts.インスタンスID",
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
-   `-3`: TTSエンジンの初期化に失敗しました。
-   `-5`: モデルの読み込みに失敗しました（例：モデルファイルまたは必要なデータが見つからない）。
-   `-6`: 指定されたTTSタスクインスタンス（例：`"tts.1003"`）が存在しません。
-   `-11`: 音声合成中または内部処理中の一般的なエラー。
-   `-20`: 入力ソースのリンクまたはデータ購読に関するエラー。
-   `-21`: タスク制限に達しました。
-   `-23`: 入力テキストの処理エラー（例：テキストが長すぎる、文字エンコーディングの問題）。

## データ入出力 (テキストと合成音声)

### 入力データ (合成用テキスト)

`llm-tts`ユニットは、いくつかの方法で合成用のテキストを受信できます：

1.  **特定の`tts.XXXX`インスタンスへの`inference` API経由:**
    *   **非ストリーミングテキスト (`object: "tts.utf-8"`)**:
        -   `data`フィールドは、合成する完全なテキストを含む単一の文字列です。
    *   **ストリーミングテキスト (`object: "tts.utf-8.stream"`)**:
        -   `data`フィールドはJSONオブジェクトです: `{"index": <整数>, "delta": "<テキストチャンク>", "finish": <真偽値>}`。
        -   テキストは内部的に蓄積されます。音声セグメントの合成は通常、蓄積されたテキスト内で句読点（コンマ、ピリオド、感嘆符、疑問符など）が検出されたとき、または`finish`が`true`のときに行われます。

2.  **リンクされたユニット経由 (`setup` APIの`input`パラメータで設定):**
    *   **`"tts.utf-8"` または `"tts.utf-8.stream"`**: インスタンスは、上記のように`inference` APIを使用して自身の`work_id`に送信されたテキストをリッスンします。
    *   **`"llm.XXXX"` または `"vlm.XXXX"`**: TTSインスタンスは、指定されたLLMまたはVLMユニットの出力を購読します。これらのユニットによって生成されたテキストは、合成のための入力として処理されます。
    *   **`"asr.XXXX"`**: TTSインスタンスは、ASRユニットの出力を購読します。書き起こされたテキストが合成のための入力として使用されます。
    *   **`"kws.XXXX"`**: TTSインスタンスは、KWSユニットからのイベントを購読します。キーワードが検出されると、通常、`superior_id_`（LLMなど）からの進行中のTTSが中断され、その`superior_id_`への購読が再確立されます。これは、KWSイベントデータ自体を直接合成するのではなく、制御フローメカニズムとして機能します。

### 出力データ (合成音声)

合成された音声は、主に2つの方法で処理できます：

1.  **`llm_audio`経由の直接再生 (`response_format`に`"sys.pcm"`または`"sys"`が含まれ、かつ `enaudio: true`の場合)**
    -   `response_format`が`"sys.pcm"`（または類似の形式）に設定され、かつ`enaudio`が`true`（デフォルトは`true`）の場合、生成されたPCMオーディオデータはセグメントごとに`llm_audio`ユニットに送信され、即時再生されます（`audio->queue_play`を使用）。
    -   これは、直接的な聴覚フィードバックのための主要なモードです。

2.  **オーディオデータのAPIプッシュ (`enoutput: true`の場合)**
    -   `enoutput`が`true`の場合、合成されたオーディオデータ（PCM）はBase64エンコードされ、タスクを開始したクライアントに非同期にプッシュされます。
    -   **ストリーミング出力 (`response_format: "tts.pcm.base64.stream"`)**:
        -   音声は合成されるとチャンク単位で送信されます。
        -   チャンクごとのJSON構造:
            ```json
            {
              "request_id": "<元のinferenceリクエストID>",
              "work_id": "tts.XXXX",
              "action": "tts_audio_event", // 慣例的なアクション名の例
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
        -   完全な入力テキストが処理された後、合成された音声全体が単一のBase64エンコード文字列として送信されます。
        -   JSON構造:
            ```json
            {
              "request_id": "<元のinferenceリクエストID>",
              "work_id": "tts.XXXX",
              "action": "tts_audio_event", // 慣例的なアクション名の例
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

新しいテキスト音声合成（TTS）タスクインスタンスを初期化し、設定します。

-   リクエストの`work_id`: `"tts"`
-   レスポンスの`work_id` (成功時): `"tts.XXXX"` (例: `"tts.1003"`)

**リクエストパラメータ:**

| パラメータ             | 型            | 必須 | デフォルト (モデルJSONより該当する場合) | 説明                                                                                                                                                                                                                                                           |
|------------------------|---------------|------|-----------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `object`               | string        | はい | N/A                                     | `tts.setup`である必要があります。                                                                                                                                                                                                                      |
| `data.model`           | string        | はい | N/A                                     | TTSモデル設定ファイルの名前（`.json`拡張子なし）。例: `"single-speaker-english-fast"`, `"single-speaker-fast-mini"`。モデルは`projects/llm_framework/main_tts/models/`に配置されます。                               |
| `data.response_format` | string        | はい | N/A                                     | 音声出力の処理方法を定義します。`"sys.pcm"`（または"sys"を含む任意のフォーマット）は、`enaudio`がtrueの場合に`llm_audio`経由の直接再生を有効にします。`"tts.pcm.base64"`や`"tts.pcm.base64.stream"`のようなフォーマットは、`enoutput`もtrueの場合にオーディオデータのAPIプッシュを有効にします。 |
| `data.input`           | string/array  | はい | N/A                                     | テキストの入力ソース。例: `"tts.utf-8"`（このインスタンスへの直接API入力用）、`"llm.1002"`（LLMの出力へリンクするため）。「入力データ」セクションを参照してください。                                                                    |
| `data.enoutput`        | boolean       | いいえ | `false` (初期ドキュメント例より)         | `true`の場合、`response_format`に従ってBase64エンコードされたPCMオーディオデータのクライアントへの非同期プッシュを有効にします。                                                                                                                            |
| `data.enaudio`         | boolean       | いいえ | `true`                                  | `true`で、かつ`response_format`がシステムオーディオ（例："sys.pcm"）を示す場合、`llm_audio`経由で合成音声を再生します。`false`の場合、`response_format`が"sys.pcm"であっても音声は再生されません。                                           |
| `data.mode_param`      | object        | いいえ | {}                                      | モデルのJSONから特定のパラメータ（例：`{"spacker_speed": 1.0, "spacker_role": 0}`）を上書きできます。サポートされる上書きは`main.cpp`の実装に依存します。 |

**リクエスト例:**
```json
{
  "request_id": "setup_tts_001",
  "work_id": "tts",
  "action": "setup",
  "object": "tts.setup",
  "data": {
    "model": "single-speaker-english-fast",
    "response_format": "sys.pcm", // 音声を直接再生
    "input": "tts.utf-8",
    "enoutput": false, 
    "enaudio": true,
    "mode_param": {
      "spacker_speed": 1.0 
    }
  }
}
```

**レスポンス (成功):**
```json
{
  "request_id": "setup_tts_001",
  "work_id": "tts.1003", // 新しいTTSタスクインスタンスID
  "action": "setup",
  "created": 1678886400,
  "code": 0,
  "message": "OK",
  "data": {
    "work_id": "tts.1003" // 作成されたインスタンスIDを確認
  }
}
```

### **inference**

特定のTTSタスクインスタンスに音声合成のためのテキストを送信します。音声出力は非同期に処理されます。

-   リクエストの`work_id`: `"tts.XXXX"` (特定のTTSタスクインスタンスID)

**リクエストパラメータ:**

| パラメータ | 型            | 必須 | デフォルト | 説明                                                                                                                               |
|-----------|---------------|------|------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| `object`  | string        | はい | N/A        | 入力テキストのフォーマットを指定します: `"tts.utf-8"` (単一の完全なテキストの場合) または `"tts.utf-8.stream"` (チャンク化されたテキストの場合)。 |
| `data`    | string/object | はい | N/A        | `object`が`"tts.utf-8"`の場合、`data`は文字列です。`object`が`"tts.utf-8.stream"`の場合、`data`はオブジェクトです: `{"index": <整数>, "delta": "<テキストチャンク>", "finish": <真偽値>}`。 |

**リクエスト例 (非ストリーミング):**
```json
{
  "request_id": "infer_tts_002",
  "work_id": "tts.1003",
  "action": "inference",
  "object": "tts.utf-8",
  "data": "こんにちは、これはテストです。"
}
```

**レスポンス (成功 - リクエスト受付):**
このレスポンスは、テキストが合成のために受信されたことを確認するだけです。
```json
{
  "request_id": "infer_tts_002",
  "work_id": "tts.1003",
  "action": "inference",
  "created": 1678886402,
  "code": 0,
  "message": "OK"
}
```

### **link**

TTSタスクインスタンスを入力ソースに動的にリンクします。

-   リクエストの`work_id`: `"tts.XXXX"`

**リクエストパラメータ:**

| パラメータ | 型     | 必須 | デフォルト | 説明                                                                       |
|-----------|--------|------|------------|----------------------------------------------------------------------------|
| `object`  | string | はい | N/A        | 通常は`"work_id"`で、dataフィールドにリンクするユニットIDが含まれることを示します。 |
| `data`    | string | はい | N/A        | 入力ソースとしてリンクするユニットの`work_id` (例: `"llm.1002"`)。             |

**リクエスト例:**
```json
{
  "request_id": "link_tts_003",
  "work_id": "tts.1003",
  "action": "link",
  "object": "work_id",
  "data": "llm.1002"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "link_tts_003",
  "work_id": "tts.1003",
  "action": "link",
  "created": 1678886404,
  "code": 0,
  "message": "OK"
}
```

### **unlink**

以前にリンクされた入力ソースをTTSタスクインスタンスから解除します。

-   リクエストの`work_id`: `"tts.XXXX"`

**リクエストパラメータ:**

| パラメータ | 型     | 必須 | デフォルト | 説明                                                                           |
|-----------|--------|------|------------|--------------------------------------------------------------------------------|
| `object`  | string | はい | N/A        | 通常は`"work_id"`で、dataフィールドに解除するユニットIDが含まれることを示します。     |
| `data`    | string | はい | N/A        | 解除する入力ソースユニットの`work_id` (例: `"llm.1002"`)。                 |

**リクエスト例:**
```json
{
  "request_id": "unlink_tts_004",
  "work_id": "tts.1003",
  "action": "unlink",
  "object": "work_id",
  "data": "llm.1002"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "unlink_tts_004",
  "work_id": "tts.1003",
  "action": "unlink",
  "created": 1678886405,
  "code": 0,
  "message": "OK"
}
```

### **pause**

TTSタスクインスタンスを一時停止します。これは通常、さらなる音声生成と再生を停止します。

-   リクエストの`work_id`: `"tts.XXXX"`

**リクエスト例:**
```json
{
  "request_id": "pause_tts_005",
  "work_id": "tts.1003",
  "action": "pause"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "pause_tts_005",
  "work_id": "tts.1003",
  "action": "pause",
  "created": 1678886406,
  "code": 0,
  "message": "OK"
}
```

### **work**

一時停止したTTSタスクインスタンスを再開します。

-   リクエストの`work_id`: `"tts.XXXX"`

**リクエスト例:**
```json
{
  "request_id": "work_tts_006",
  "work_id": "tts.1003",
  "action": "work"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "work_tts_006",
  "work_id": "tts.1003",
  "action": "work",
  "created": 1678886407,
  "code": 0,
  "message": "OK"
}
```

### **exit**

TTSタスクインスタンスを停止し、削除してリソースを解放します。

-   リクエストの`work_id`: `"tts.XXXX"`

**リクエスト例:**
```json
{
  "request_id": "exit_tts_007",
  "work_id": "tts.1003",
  "action": "exit"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "exit_tts_007",
  "work_id": "tts.1003",
  "action": "exit",
  "created": 1678886408,
  "code": 0,
  "message": "OK"
}
```

### **taskinfo**

アクティブなTTSタスクに関する情報を取得します。

**ケース1: 一般的なユニット情報**

-   リクエストの`work_id`: `"tts"`
-   **説明**: 全てのアクティブなタスクインスタンスIDのリストを返します。

**リクエスト例:**
```json
{
  "request_id": "taskinfo_tts_general_008",
  "work_id": "tts",
  "action": "taskinfo"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "taskinfo_tts_general_008",
  "work_id": "tts",
  "action": "taskinfo",
  "created": 1678886409,
  "code": 0,
  "message": "OK",
  "object": "tts.tasklist",
  "data": [
    "tts.1003"
  ]
}
```

**ケース2: 特定のタスクインスタンス情報**

-   リクエストの`work_id`: `"tts.XXXX"`
-   **説明**: 指定されたタスクインスタンスのランタイムパラメータを返します。

**リクエスト例:**
```json
{
  "request_id": "taskinfo_tts_specific_009",
  "work_id": "tts.1003",
  "action": "taskinfo"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "taskinfo_tts_specific_009",
  "work_id": "tts.1003",
  "action": "taskinfo",
  "created": 1678886410,
  "code": 0,
  "message": "OK",
  "object": "tts.taskinfo",
  "data": {
    "model": "single-speaker-english-fast",
    "response_format": "sys.pcm",
    "enoutput": false,
    "enaudio": true,
    "inputs": ["tts.utf-8"]
    // spacker_speed のような他のパラメータは、main.cpp で task_info の一部として追加されていれば含まれる可能性があります
  }
}
```

## モデル設定ファイルとの関連性

`llm-tts`ユニットは、モデル固有のJSON設定ファイル（例：`mode_single-speaker-english-fast.json`）に依存しており、これらは`projects/llm_framework/main_tts/models/`ディレクトリまたはシステム全体のモデルパスに配置されています。

これらのモデル設定ファイルの主な側面：
-   **`mode`**: `setup` APIで使用されるモデル名（例：`"single-speaker-english-fast"`）。
-   **`type`**: ユニットタイプ、通常は`"tts"`。
-   **`capabilities`**: 機能の説明、例：`"tts"`, `"English"`。
-   **`mode_param`オブジェクト**: TTSエンジンの主要なパラメータが含まれています：
    -   `ttsModelName` (string): 実際のTTSモデルのファイル名（例：`"single_speaker_english_fast.bin"`）。このファイルはモデルの特定のディレクトリ内にあります。
    -   `spacker_speed` (float, オプション): デフォルトの読み上げ速度（例：`1.0`）。`setup`の`mode_param`で上書き可能。
    -   `spacker_role` (integer, オプション): デフォルトのスピーカーID/ロール（モデルが複数のスピーカーをサポートしている場合）。`setup`の`mode_param`で上書き可能。
    -   `sample_rate` (integer, オプション): モデルが学習された、または出力を期待するサンプルレート。
    -   `awake_delay` (integer): KWSと連携する際に使用される遅延時間（ミリ秒）。

これらのパラメータは、特定のモデルでタスクがセットアップされるときにロードされます。

## 重要事項
-   **同時実行**: 「同じタイプのユニットを複数同時に動作させることはできません」という注意書きはTTSユニットに適用されます。複数のTTSインスタンス（例：2つの`llm-tts`、または`llm-tts`と`llm-melotts`）を同時に実行すると、リソース競合（特にNPUのようなハードウェアアクセラレーションを使用する場合）や音声再生の問題が発生する可能性があります。
-   **音声再生**: `enaudio: true`で、かつ`response_format`がシステムオーディオ（例："sys.pcm"）を示す場合、`llm-tts`ユニットは再生のために`llm_audio`ユニットに依存します。`llm_audio`が機能していることを確認してください。
-   **テキストセグメンテーション**: ユニットは入力テキストをセグメント単位で処理します。通常、句読点（コンマ、ピリオド、疑問符、感嘆符など）で区切られます。これにより、より長いテキストの応答性の高い合成が可能になります。
```
