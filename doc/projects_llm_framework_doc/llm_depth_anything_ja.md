# llm-depth_anything (深度推定)

`llm-depth_anything`ユニットは、StackFlowフレームワーク内のコンピュータビジョンサービスであり、特に単眼深度推定を実行するために設計されています。画像を入力として受け取り、深度マップを生成します。これはJPEG画像として視覚化できます。このユニットは、カメラユニットから直接入力を受け取るか、Base64エンコードされた画像データを含むAPI呼び出しを介して入力を受け取ることができます。

## API呼び出しの共通規約

`llm-depth_anything`へのAPI呼び出しはJSONメッセージを介して行われます。

### リクエスト構造

`llm-depth_anything` APIへの標準的なリクエストは以下の構造に従います：

```json
{
  "request_id": "クライアント生成のUUIDまたはカウンター",
  "work_id": "depth_anythingまたはdepth_anything.インスタンスID",
  "action": "APIアクション名",
  "object": "オプションのオブジェクトタイプ文字列", // 該当する場合
  "data": { /* API固有のペイロード */ }     // または一部APIでは文字列
}
```

-   `request_id` (string, 必須): リクエストとレスポンスを関連付けるための、クライアントが生成する一意の識別子。
-   `work_id` (string, 必須):
    -   初期設定や一般的なクエリの場合: `"depth_anything"`。
    -   特定の深度推定タスクインスタンスに対する操作の場合: `"depth_anything.XXXX"` (例: `"depth_anything.1007"`)。`XXXX`は成功した`setup`呼び出しによって返されるインスタンスIDです。
-   `action` (string, 必須): 呼び出すAPIアクション (例: `"setup"`, `"exit"`)。
-   `object` (string, オプション): 送信するデータのタイプやコンテキストを指定します（該当する場合）。例：設定データの場合は`"depth_anything.setup"`、画像データの場合は`"cv.jpeg.base64"`。
-   `data` (object または string, オプション): API呼び出しのペイロード。構造はAPIによって異なります。

### レスポンス構造

**成功レスポンス:**

API呼び出しが受け入れられ、正常に処理されたことを示します。

```json
{
  "request_id": "リクエストからミラーリング",
  "work_id": "depth_anythingまたはdepth_anything.インスタンスID", // ミラーリングまたは更新
  "action": "APIアクション名",                               // リクエストからミラーリング
  "created": 1678886400,                                     // 整数: レスポンス生成のUnixタイムスタンプ
  "code": 0,                                                 // 整数: 0は成功を示す
  "message": "OK",                                           // 文字列: 成功メッセージ
  "data": { /* API固有データ */ }                        // オプション: APIによって返されるペイロード
}
```
-   API呼び出しが新しいタスクインスタンスの作成に成功した場合（例：`setup`）、レスポンスの`work_id`は新しいインスタンスID（例：`"depth_anything.1007"`）になり、多くの場合`data`オブジェクトにも含まれます。

**エラーレスポンス:**

API呼び出し処理中の失敗を示します。

```json
{
  "request_id": "リクエストからミラーリング",
  "work_id": "depth_anythingまたはdepth_anything.インスタンスID",
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
-   `-5`: モデルの読み込みに失敗しました（例：モデルファイルが見つからない、モデル設定が無効、エンジン初期化エラー）。
-   `-6`: 指定されたタスクインスタンス（例：`"depth_anything.1007"`）が存在しません。
-   `-11`: 一般的なモデルランタイムエラーまたは処理中の内部エラー。
-   `-20`: 入力ソースのリンクまたはデータ購読に関するエラー。
-   `-21`: タスク制限に達しました（通常、1つのインスタンスのみサポートされます）。
-   `-23`: 入力画像データのBase64デコードエラー。
-   `-24`: 画像デコードエラー（例：無効なJPEGフォーマット）。
-   `-25`: 画像処理エラー（例：リサイズまたは色変換の失敗）。

## データ入出力

### 入力データ

`llm-depth_anything`ユニットは、`setup` APIの`input`パラメータを介して設定される2つの主要な方法で深度推定用の画像データを受信できます。

1.  **直接カメラ入力 (`camera.XXXX`インスタンスID経由):**
    -   `input`がカメラインスタンスID（例：`"camera.1001"`）に設定されている場合、ユニットはそのカメラインスタンスからの生ビデオフレーム出力（通常はそのZMQ PUBソケットから）を購読します。
    -   **重要要件**: カメラユニットは、Depth Anythingモデルが期待する正確な解像度でフレームを出力するように設定する必要があります。これは、モデルの設定JSON内の`img_w`（幅）と`img_h`（高さ）で指定されます（例：`mode_depth-anything-ax630c.json`の場合は384x256）。`llm-depth_anything`ユニットはカメラからYUYV422形式の生フレームを期待し、処理のためにRGB/BGRに変換します。
    -   **`depth-anything-ax630c`用のカメラ設定例:**
        ```json
        {
          "request_id": "setup_cam_for_depth",
          "work_id": "camera",
          "action": "setup",
          "object": "camera.setup",
          "data": {
            "input": "/dev/video0", // または適切なカメラデバイス
            "response_format": "camera.raw", // YUYV422生フレームを提供
            "frame_width": 384,
            "frame_height": 256,
            "enoutput": false // depth_anythingはZMQ PUB/SUBを使用し、カメラからのAPIプッシュは使用しない
          }
        }
        ```

2.  **API入力 (Base64エンコードJPEG経由):**
    -   `input`が`llm-depth_anything`インスタンス自身の`work_id`（例：`"depth_anything.1007"`自体、またはそれを含む配列）に設定されている場合、ユニットはその`work_id`へのAPI呼び出しを介して画像データを受信することを期待します。
    -   画像データを送信するリクエストには以下が必要です：
        -   `work_id`: 特定の`depth_anything.XXXX`インスタンスID。
        -   `action`: これは別の「アクション」名ではなく、ユニットのデフォルトデータ処理メカニズムが購読しているZMQチャネルにプッシュされるデータです。
        -   `object`: `"cv.jpeg.base64"`。
        -   `data`: Base64エンコードされたJPEG画像を含む文字列。
    -   ユニットはBase64文字列をデコードし、次にJPEG画像をデコードして処理します。画像はモデルが必要とする入力寸法（`img_w` x `img_h`）にリサイズされます。

### 出力データ (深度マップ)

`setup`呼び出しで`enoutput`が`true`（デフォルト）の場合、`llm-depth_anything`ユニットは深度推定結果を非同期にプッシュします。出力は、JPEG画像としてエンコードされ、その後Base64エンコードされた深度マップの視覚的表現です。

出力のフォーマットは、`setup` APIの`response_format`パラメータによって決定されます。

1.  **ストリームJPEG出力 (`response_format: "jpeg.base64.stream"`)**
    -   ユニットはJSONオブジェクトのストリームを送信します。各オブジェクトは処理されたフレームを表します。
    -   **フレームごとのJSON構造:**
        ```json
        {
          "request_id": "<元のsetupリクエストID>", // setup呼び出しのrequest_id
          "work_id": "depth_anything.XXXX",          // 特定のインスタンスID
          "object": "jpeg.base64.stream",            // レスポンスフォーマット
          "error": {"code": 0, "message": ""},       // このデータプッシュの成功を示す
          "data": {
            "index": 0,                              // 整数: このストリームセグメントのシーケンス番号
            "delta": "<Base64エンコードされたJPEG深度マップ>", // 文字列: Base64エンコードされた深度マップのJPEG画像
            "finish": true                           // 真偽値: 各画像が完全な結果であるため、通常はtrue
          }
        }
        ```

2.  **単一JPEG出力 (`response_format: "jpeg.base64"`)**
    -   ユニットは、Base64エンコードされたJPEG深度マップを含む単一のJSONメッセージを送信します。
    -   **JSON構造:**
        ```json
        {
          "request_id": "<元のsetupリクエストID>",
          "work_id": "depth_anything.XXXX",
          "object": "jpeg.base64",                  // レスポンスフォーマット
          "error": {"code": 0, "message": ""},       // このデータプッシュの成功を示す
          "data": "<Base64エンコードされたJPEG深度マップ>"  // 文字列: Base64エンコードされた深度マップのJPEG画像
        }
        ```

**出力JPEGの内容:**
出力されるJPEG画像は、推定された深度マップの視覚化です。これは通常、ピクセルの強度が深度に対応するグレースケール画像です（例：明るいピクセルが近く、暗いピクセルが遠い、またはその逆）。ユニットに実装されている後処理によっては、色付けされた深度マップである場合もあります。

## APIリファレンス

### **setup**

新しい深度推定タスクインスタンスを初期化し、設定します。通常、一度に1つのタスクインスタンスのみサポートされます。

-   リクエストの`work_id`: `"depth_anything"`
-   レスポンスの`work_id` (成功時): `"depth_anything.XXXX"` (例: `"depth_anything.1007"`)

**リクエストパラメータ:**

| パラメータ              | 型            | 必須 | デフォルト (モデルJSONより該当する場合) | 説明                                                                                                                                                                                             |
|------------------------|---------------|------|-----------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `object`               | string        | はい | N/A                                     | `depth_anything.setup`である必要があります。                                                                                                                                                         |
| `data.model`           | string        | はい | N/A                                     | モデル設定ファイルの名前（`.json`なし）。例: `"depth-anything-ax630c"`。                                                                                                                             |
| `data.response_format` | string        | はい | N/A                                     | `enoutput`がtrueの場合の結果の出力フォーマット。サポート: `"jpeg.base64.stream"`, `"jpeg.base64"`。                                                                                                     |
| `data.input`           | string/array  | はい | N/A                                     | 入力ソース。カメラインスタンスID（例: `"camera.1001"`）で生フレームを入力、またはユニット自身の`work_id`（例: `"depth_anything.1007"`）でその`work_id`へのAPI呼び出しによるBase64 JPEG入力を指定。 |
| `data.enoutput`        | boolean       | いいえ | `true`                                  | 深度マップ結果の非同期プッシュを有効 (`true`) または無効 (`false`) にします。                                                                                                                               |

**リクエスト例 (カメラからの入力):**
```json
{
  "request_id": "setup_depth_cam_001",
  "work_id": "depth_anything",
  "action": "setup",
  "object": "depth_anything.setup",
  "data": {
    "model": "depth-anything-ax630c",
    "response_format": "jpeg.base64.stream",
    "input": "camera.1001", // camera.1001が384x256のraw出力に設定されていると仮定
    "enoutput": true
  }
}
```

**リクエスト例 (API経由のBase64 JPEG入力):**
```json
{
  "request_id": "setup_depth_api_002",
  "work_id": "depth_anything",
  "action": "setup",
  "object": "depth_anything.setup",
  "data": {
    "model": "depth-anything-ax630c",
    "response_format": "jpeg.base64",
    "input": "depth_anything.1008", // インスタンスは自身のwork_idで画像データを受信待機
    "enoutput": true
  }
}
```
*(注意: API入力の場合、`input`内の`work_id`は、この`setup`呼び出しによって返される`work_id`と一致する必要があります。例えば、この呼び出しが`depth_anything.1008`を返す場合、明確性のために`input`は`depth_anything.1008`またはそれを含む配列として設定するのが理想的ですが、ユニットは内部的に自身が生成した`work_id`を購読します。)*

**レスポンス (成功):**
```json
{
  "request_id": "setup_depth_cam_001",
  "work_id": "depth_anything.1007", // 新しいタスクインスタンスID
  "action": "setup",
  "created": 1678886400,
  "code": 0,
  "message": "OK",
  "data": {
    "work_id": "depth_anything.1007" // 作成されたインスタンスIDを確認
  }
}
```

**レスポンス (エラー - モデル読み込み失敗):**
```json
{
  "request_id": "setup_depth_003",
  "work_id": "depth_anything",
  "action": "setup",
  "created": 1678886401,
  "error": {
    "code": -5,
    "message": "モデルの読み込みに失敗しました。"
  }
}
```

### **exit**

深度推定タスクインスタンスを停止し、削除してリソースを解放します。

-   リクエストの`work_id`: `"depth_anything.XXXX"` (特定のタスクインスタンスID)

**リクエストパラメータ**: 標準の`request_id`, `work_id`, `action`以外はありません。

**リクエスト例:**
```json
{
  "request_id": "exit_depth_004",
  "work_id": "depth_anything.1007",
  "action": "exit"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "exit_depth_004",
  "work_id": "depth_anything.1007", // 終了したタスクのwork_id
  "action": "exit",
  "created": 1678886402,
  "code": 0,
  "message": "OK"
}
```

**レスポンス (エラー - タスクが見つからない):**
```json
{
  "request_id": "exit_depth_005",
  "work_id": "depth_anything.9999", // 存在しないタスク
  "action": "exit",
  "created": 1678886403,
  "error": {
    "code": -6,
    "message": "ユニットが存在しません"
  }
}
```

### **taskinfo**

アクティブな深度推定タスクに関する情報を取得します。動作はリクエストの`work_id`に依存します。

**ケース1: 一般的なユニット情報**

-   リクエストの`work_id`: `"depth_anything"`
-   **説明**: 全てのアクティブなタスクインスタンスIDのリストを返します。

**リクエスト例:**
```json
{
  "request_id": "taskinfo_depth_general_006",
  "work_id": "depth_anything",
  "action": "taskinfo"
}
```

**レスポンス (成功 - 1タスクアクティブ時):**
```json
{
  "request_id": "taskinfo_depth_general_006",
  "work_id": "depth_anything",
  "action": "taskinfo",
  "created": 1678886404,
  "code": 0,
  "message": "OK",
  "object": "depth_anything.tasklist",
  "data": [
    "depth_anything.1007" // アクティブなインスタンスIDのリスト
  ]
}
```

**ケース2: 特定のタスクインスタンス情報**

-   リクエストの`work_id`: `"depth_anything.XXXX"` (例: `"depth_anything.1007"`)
-   **説明**: 指定されたタスクインスタンスのランタイムパラメータと設定を返します。

**リクエスト例:**
```json
{
  "request_id": "taskinfo_depth_specific_007",
  "work_id": "depth_anything.1007",
  "action": "taskinfo"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "taskinfo_depth_specific_007",
  "work_id": "depth_anything.1007",
  "action": "taskinfo",
  "created": 1678886405,
  "code": 0,
  "message": "OK",
  "object": "depth_anything.taskinfo",
  "data": {
    "model": "depth-anything-ax630c",
    "response_format": "jpeg.base64.stream",
    "enoutput": true,
    "inputs": [ "camera.1001" ] // リンクされた入力ソースの配列
  }
}
```

> **タスクインスタンスの`work_id`に関する注意**: タスクインスタンスの`work_id`の数値部分（例：`depth_anything.1007`の`1007`）は、`setup` APIによる作成時にシステムによって動的に割り当てられます。

## モデル設定ファイルとの関連性

`llm-depth_anything`ユニットは、主要な操作パラメータについて、モデル固有のJSON設定ファイル（例：`mode_depth-anything-ax630c.json`）に依存しています。これらのファイルは通常、モデル用のシステムディレクトリ（例：`projects/llm_framework/main_depth_anything/models/`または一般的なパス`/opt/m5stack/data/models/`）に配置されます。

これらのモデル設定ファイルの主な側面：
-   **`mode`**: モード/モデルの名前（例：`"depth-anything-ax630c"`）。これは`setup` APIで使用されます。
-   **`type`**: ユニットのタイプ（例：コンピュータビジョンの場合は`"cv"`）。
-   **`capabilities`**: 機能の説明。Depth Anythingの場合、これは`"Depth Estimation"`（深度推定）として理解されるべきです。（注意：`mode_depth-anything-ax630c.json`の例では`"Segmentation"`（セグメンテーション）となっていますが、これはモデル名と目的に基づくとこのモデルタイプとしては不正確と思われます。）
-   **`input_type`**: APIベースの入力で期待される入力データタイプのリスト（例：`"cv.jpeg.base64"`）。
-   **`output_type`**: 潜在的な出力データタイプのリスト（例：`"cv.jpeg.base64"`）。
-   **`mode_param`オブジェクト**: このオブジェクトには、モデルとその操作に関する重要なパラメータが含まれています：
    -   `depth_anything_model` (string): 実際の深度推定モデルのファイル名（例：AXERAハードウェア用の`depth_anything.axmodel`、または他のプラットフォーム用の`.onnx`ファイル）。このファイルはモデルの特定のディレクトリ（例：`depth-anything-ax630c/depth_anything.axmodel`）内にあります。
    -   `img_h` (integer): モデルが必要とする入力画像の高さ（例：`256`）。
    -   `img_w` (integer): モデルが必要とする入力画像の幅（例：`384`）。
    -   `model_type` (string): モデルのサブタイプまたは特定のバリアントを示す場合があります（例：サンプルJSONでは`"segment"`となっていますが、"depth"または"depth_estimation"の方がより適切です）。C++コードはこの値を使用して後処理ロジックを変更する可能性があります。

特定の`model`名を指定して`setup` API呼び出しが行われると、システムは対応するJSON設定ファイルをロードします。このファイル内のパラメータ、特に`depth_anything_model`、`img_h`、および`img_w`が、ロードするモデルと期待される入力画像の寸法を決定します。カメラユニットから入力する場合、最適なパフォーマンスを得てエラーを回避するために、そのカメラがこれらの寸法に一致するフレームを提供するように設定する必要があります。
```
