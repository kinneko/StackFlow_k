# llm-yolo (YOLO物体検出ユニット)

`llm-yolo`ユニットは、さまざまなYOLOモデルを使用して、汎用的な物体検出、セグメンテーション、姿勢推定、および回転バウンディングボックス検出サービスを提供します。直接的な画像データやカメラストリームなど、さまざまなソースからの画像を処理できます。

## setup (セットアップ)

YOLO処理タスクを設定し、初期化します。

JSONを送信:

```json
{
  "request_id": "2",
  "work_id": "yolo",
  "action": "setup",
  "object": "yolo.setup",
  "data": {
    "model": "yolo11n",
    "response_format": "yolo.box",
    "input": ["yolo.jpg.base64", "camera.0"],
    "enoutput": true,
    "model_type": "detect", // "detect", "segment", "pose", "obb"
    "confidence_threshold": 0.45,
    "nms_threshold": 0.45
  }
}
```

**`data`内のパラメータ:**

*   `request_id` (string, 必須): リクエストの一意の識別子。
*   `work_id` (string, 必須): 初期設定には `yolo` を使用します。セットアップ成功時に特定の `work_id` (例: `yolo.1001`) が返されます。
*   `action` (string, 必須): 呼び出すメソッド。`setup` である必要があります。
*   `object` (string, 必須): 転送されるデータ型。通常は `yolo.setup` です。
*   `model` (string, 必須): 使用するYOLOモデルのベース名 (例: `yolo11n`, `yolo11n-seg`, `yolo11n-pose`)。システムは、この名前に基づいて事前定義されたパス (例: `projects/llm_framework/main_yolo/`) からモデル設定ファイル (例: `mode_yolo11n.json`) をロードします。
*   `response_format` (string, 必須): 出力の形式を定義します。
    *   `yolo.box`: 標準的なYOLO検出出力。座標やその他の値は文字列としてフォーマットされます (詳細はInferenceセクションを参照)。
    *   `yolo.boxV2`: `yolo.box` と同様ですが、数値 (信頼度、座標) は文字列ではなく浮動小数点数です。
    *   `.stream` を追加する (例: `yolo.box.stream`) と、個々の検出オブジェクトのストリーミングが有効になります。
*   `input` (string または stringの配列, 必須): 入力ソースと形式を指定します。
    *   `yolo.jpg.base64`: Base64エンコードされたJPEG画像データ。オブジェクトが `yolo.jpg.stream.base64` の場合はストリーミング可能です。
    *   `yolo.png.base64`: Base64エンコードされたPNG画像データ (`cv::imdecode` がサポートしていると仮定)。
    *   `camera.x`: カメラユニットへのリンク (例: `camera.0`)。データは生のYUV (YUYV) フレームとして期待されます。ユニットは `sys` を呼び出してカメラユニットの出力ポートを取得します。
    *   `yolo.yuv.raw`, `yolo.rgb.raw`, `yolo.bgr.raw` のような他のRAWフォーマットは、推論呼び出しのオブジェクト文字列が内部ハンドラ (`inference_raw_yuv`, `inference_raw_rgb`, `inference_raw_bgr`) と一致する場合にサポートされる可能性があります。これらは、オブジェクト文字列に `.base64` が含まれていればBase64デコードされたRAWバイトデータを期待します。
*   `enoutput` (boolean, 必須): `true` の場合、ユニットは検出結果を送信します。`false` の場合、出力は抑制されます。
*   **モデル設定パラメータ (オプション、通常はモデルのJSON設定ファイルからロードされますが、ここで上書きできます):**
    *   `model_type` (string, オプション): YOLOモデルの種類を指定します。デフォルトは "detect" です。
        *   `"detect"`: 標準的な物体検出 (バウンディングボックス)。
        *   `"segment"`: インスタンスセグメンテーション (バウンディングボックス + マスク)。
        *   `"pose"`: 姿勢推定 (バウンディングボックス + キーポイント)。
        *   `"obb"`: 回転バウンディングボックス検出 (バウンディングボックス + 角度)。
    *   `confidence_threshold` (float, オプション): 検出が有効と見なされるための最小確率スコア。例: `0.45`。(注意: モデルJSONは `pron_threshold` を使用する場合があります)。
    *   `nms_threshold` (float, オプション): 重複するバウンディングボックスをフィルタリングするために使用されるNon-Maximum Suppressionの閾値。例: `0.45`。
    *   `img_h` (integer, オプション): モデルが期待する画像の高さ (例: 640)。
    *   `img_w` (integer, オプション): モデルが期待する画像の幅 (例: 640)。
    *   `cls_num` (integer, オプション): モデルが学習したクラスの数 (例: COCOの場合は80)。
    *   `point_num` (integer, オプション): 姿勢モデルのキーポイントの数 (例: 人間の姿勢の場合は17)。
    *   `cls_name` (stringの配列, オプション): クラス名のリスト。モデルのJSONからロードされます。

**内部モデル設定 (モデルのJSONから。例: `mode_yolo11n.json`):**
*   `yolo_model` (string): 実際の `.axmodel` ファイルへのパス (例: `yolov8n.axmodel`)。このパスはモデルのディレクトリからの相対パスです。

**応答JSON (成功時):**
```json
{
  "created": 1737596640,
  "data": "None",
  "error": {
    "code": 0,
    "message": ""
  },
  "object": "None",
  "request_id": "2",
  "work_id": "yolo.1001" // このYOLOタスクの一意のwork_id
}
```
**応答JSON (エラー時):**
`task_count_` (YOLOの場合は最大1) 制限に達した場合:
```json
{
  "created": ...,
  "data": "None",
  "error": {"code": -21, "message": "task full"},
  "object": "None",
  "request_id": "2",
  "work_id": "yolo"
}
```
モデルのロードに失敗した場合:
```json
{
  "created": ...,
  "data": "None",
  "error": {"code": -5, "message": "Model loading failed."}, // またはJSONエラーの場合は-2、設定エラーの場合は-6
  "object": "None",
  "request_id": "2",
  "work_id": "yolo"
}
```

## inference (推論)

設定済みの `llm-yolo` タスクに画像データを送信して処理させます。このアクションは通常、リンクされたカメラ入力に依存しない場合に使用されます。

**リクエストJSON (Base64エンコードされたJPEG):**
```json
{
  "request_id": "5",
  "work_id": "yolo.1001",
  "action": "inference",
  "object": "yolo.jpg.base64", // またはストリーミングチャンクの場合は yolo.jpg.stream.base64 など
  "data": { // ストリーミングオブジェクトタイプの場合
    "delta": "/9j/4AAQSkZJRgABAQEAYABgAAD/... (base64エンコードされた画像データ) ...",
    "index": 0,
    "finish": true
  }
  // オブジェクトがストリーミングでない場合 (例: "yolo.jpg.base64")、dataは単なるbase64文字列です:
  // "data": "/9j/4AAQSkZJRgABAQEAYABgAAD/... (base64エンコードされた画像データ) ..."
}
```

**パラメータ:**
*   `request_id` (string, 必須): 一意のリクエストID。
*   `work_id` (string, 必須): YOLOタスクの `work_id` (例: `yolo.1001`)。
*   `action` (string, 必須): `inference` である必要があります。
*   `object` (string, 必須): 画像データの形式を指定します。
    *   `yolo.jpg.base64`: 完全なJPEG画像、Base64エンコード済み。
    *   `yolo.jpg.stream.base64`: ストリーミングJPEG画像、Base64エンコード済み。`data` フィールドは `delta`, `index`, `finish` を持つオブジェクトである必要があります。
    *   `cv::imdecode` が処理できる場合は、PNGなどの他のエンコーディングもサポートされる可能性があります。
    *   アプリケーションがRAWピクセルデータ (Base64デコード後) を送信する場合、`yolo.yuv.raw.base64`, `yolo.rgb.raw.base64`, `yolo.bgr.raw.base64` のようなRAWフォーマットを使用できます。RAWピクセルデータサイズは `img_w * img_h * channels` と一致する必要があります。
*   `data` (string または object, 必須): 画像データ。`object` がストリーミングを示す場合、これは `delta`, `index`, `finish` を持つオブジェクトです。それ以外の場合は文字列です (例: Base64エンコードされた画像)。

**画像処理ステップ (内部):**
1.  入力データがデコードされます (例: Base64、次に `cv::imdecode` によるJPEG/PNGデコード、またはRAW YUVからRGB/BGRへの変換)。
2.  画像は `common::get_input_data_letterbox` を使用して前処理されます。これにより、モデルの `img_w` と `img_h` に一致するように、アスペクト比を維持しながらパディングによってサイズ変更されます。色変換 (BGRからRGB) が行われる場合があります。
3.  処理された画像データがYOLOモデル (`yolo_->Run()`) に入力されます。
4.  結果は、スコア/信頼度閾値とNMSを適用して後処理されます (`yolo_->Post_Process()`)。
5.  検出されたオブジェクトは `response_format` に従ってフォーマットされます。

**応答JSON (`yolo.box` または `yolo.boxV2`):**
出力はJSON配列で、配列内の各オブジェクトが検出されたアイテムを表します。ストリーミングが有効になっている場合 (`response_format` に `.stream` が含まれる場合)、個々の検出オブジェクトはストリームメッセージの `delta` として送信され、最終メッセージにはすべての検出が含まれるか、空のdeltaと `finish: true` が含まれます。

**各検出オブジェクトの構造 (応答の `data` 配列内):**

*   **共通フィールド:**
    *   `class` (string): 検出されたクラス名 (例: "person", "car")。設定の `cls_name` から。
    *   `confidence` (`yolo.box` の場合はstring、`yolo.boxV2` の場合はfloat): 検出の信頼度スコア (例: "0.87" または `0.87`)。
    *   `bbox` (array): バウンディングボックスの座標。
        *   `yolo.box` の場合: `["x1", "y1", "x2", "y2"]` (文字列、小数点以下2桁にフォーマット)。これらは通常、レターボックス除去後の元画像の絶対ピクセル値です。
        *   `yolo.boxV2` の場合: `[x, y, width, height]` (浮動小数点数)。これらはレターボックス化された画像または元画像に対する相対値である可能性があります。特定のモデルランナーを確認する必要があります。ソースは `obj.rect.x, obj.rect.y, obj.rect.width, obj.rect.height` を示唆しており、これらは通常、再スケーリング後の絶対値です。

*   **`model_type` に基づく追加フィールド:**
    *   `model_type` が `"segment"` の場合:
        *   `mask` (`yolo.box` の場合はstringの配列、`yolo.boxV2` の場合はfloatの配列): マスクの特徴または輪郭点。`obj.mask_feat` からのこれらの "mask_feat" 値の正確な性質には、さらなるコンテキストが必要です (例: RAWマスクピクセル、RLE、ポリゴンポイント)。
    *   `model_type` が `"pose"` の場合:
        *   `kps` (`yolo.box` の場合はstringの配列、`yolo.boxV2` の場合はfloatの配列): `obj.kps_feat` からのキーポイント特徴。通常、各キーポイントに対して `[x, y, score]` または `[x, y, visibility, score]` の配列で、最大 `point_num` 個のキーポイント。
    *   `model_type` が `"obb"` (回転バウンディングボックス) の場合:
        *   `angle` (float): バウンディングボックスの回転角度。

**応答例 (非ストリーミング, `yolo.box`, `model_type: "detect"`):**
```json
{
  "created": 1737596700,
  "data": [
    {
      "class": "cat",
      "confidence": "0.92",
      "bbox": ["100.50", "150.25", "250.00", "300.75"]
    },
    {
      "class": "dog",
      "confidence": "0.88",
      "bbox": ["320.00", "180.50", "450.20", "350.00"]
    }
  ],
  "error": {"code": 0, "message": ""},
  "object": "yolo.box", // response_formatと一致
  "request_id": "5",
  "work_id": "yolo.1001"
}
```
推論中にエラーが発生した場合 (例: データが空、Base64デコードエラー、モデル実行失敗):
```json
{
  "created": ...,
  "data": "None",
  "error": {"code": -11, "message": "Model run failed."}, // または -23, -24
  "object": "None",
  "request_id": "5",
  "work_id": "yolo.1001"
}
```

## link (リンク)

別のユニット (例: カメラユニット) の出力をこのYOLOタスクへの入力としてリンクします。

**リクエストJSON:**
```json
{
  "request_id": "3",
  "work_id": "yolo.1001",
  "action": "link",
  "object": "work_id",
  "data": "camera.0" // リンク元のユニットのwork_id (例: カメラタスク)
}
```
**動作:**
*   `data` がカメラユニットを参照する場合 (例: "camera.0"):
    *   YOLOユニットは `sys` (`unit_call("sys", "sql_select", data + ".out_port")`) を呼び出してカメラの出力ポート (ZMQ URL) を取得します。
    *   次に、このURLをサブスクライブします。受信データ (RAW YUVフレームと期待される) は `task_camera_data` によって処理されます。
*   `data` が別のYOLOユニットまたは汎用画像ソースを参照する場合 (例: "yolo.some_other_instance_id"):
    *   `work_id` を直接サブスクライブし、`task_user_data` に適した画像データを期待します。
*   リンクされた `data` (work_id) はYOLOタスクの `inputs_` リストに追加されます。

**応答JSON (成功時):**
```json
{
  "created": ...,
  "data": "None",
  "error": {"code": 0, "message": ""},
  "object": "None",
  "request_id": "3",
  "work_id": "yolo.1001"
}
```

## unlink (アンリンク)

以前に接続されたユニットをアンリンクします。

**リクエストJSON:**
```json
{
  "request_id": "4",
  "work_id": "yolo.1001",
  "action": "unlink",
  "object": "work_id",
  "data": "camera.0" // アンリンクするユニットのwork_id
}
```
**動作:**
*   `stop_subscriber_work_id()` を使用するか、カメラの場合は直接ZMQサブスクリプションを停止することにより、指定された `data` (アンリンクされるユニットのwork_id) へのサブスクリプションを停止します。
*   YOLOタスクの `inputs_` リストから `data` を削除します。

**応答JSON (成功時):**
```json
{
  "created": ...,
  "data": "None",
  "error": {"code": 0, "message": ""},
  "object": "None",
  "request_id": "4",
  "work_id": "yolo.1001"
}
```

## exit (終了)

YOLOタスクを終了し、そのリソースを解放します。

JSONを送信: (元のドキュメントと同じ)
```json
{
  "request_id": "7",
  "work_id": "yolo.1001",
  "action": "exit"
}
```
**動作:**
*   `llm_task_obj->stop()` を呼び出します。これにより、内部の推論スレッドに残りのアイテムの処理を完了させて結合するよう通知します。
*   アクティブなZMQサブスクリプションを停止します。
*   YOLOモデル (`yolo_->Release()`) を解放し、これがそれを使用している最後のタスクである場合はAXエンジンを非初期化します (`_ax_deinit()`)。

応答JSON: (元のドキュメントと同じ)
```json
{
  "created": 1737596871,
  "data": "None",
  "error": {"code": 0, "message": ""},
  "object": "None",
  "request_id": "7",
  "work_id": "yolo.1001"
}
```

## taskinfo (タスク情報)

実行中のYOLOタスクに関する情報を取得します。

**リクエストJSON (すべての `yolo` タスクのリストを取得):** (元のドキュメントと同じ)
```json
{
  "request_id": "2",
  "work_id": "yolo",
  "action": "taskinfo"
}
```
応答JSON (タスクのリスト): (元のドキュメントと同じ)
```json
{
  "created": 1737596668,
  "data": ["yolo.1001"],
  "error": {"code": 0, "message": ""},
  "object": "yolo.tasklist",
  "request_id": "2",
  "work_id": "yolo"
}
```

**リクエストJSON (特定のタスクパラメータを取得):** (元のドキュメントのyolo.1003を修正し、例と一致するようにyolo.1001に変更)
```json
{
  "request_id": "2",
  "work_id": "yolo.1001",
  "action": "taskinfo"
}
```
応答JSON (特定のタスクパラメータ): (`model_type` およびその他のセットアップパラメータを含む)
```json
{
  "created": 1737596698,
  "data": {
    "model": "yolo11n",
    "response_format": "yolo.box",
    "enoutput": true,
    "inputs": ["yolo.jpg.base64", "camera.0"], // 現在の入力ソース/リンクを反映
    "model_type": "detect", // およびセットアップからの他のパラメータ
    "confidence_threshold": 0.45,
    "nms_threshold": 0.45
  },
  "error": {"code": 0, "message": ""},
  "object": "yolo.taskinfo",
  "request_id": "2",
  "work_id": "yolo.1001"
}
```

> **注意:** `work_id` はユニットの初期化順序に従って増加し、固定インデックス値ではありません。YOLOの `task_count_` は通常1に制限されます。

## エラーコード (共通)
*   `0`: 成功。
*   `-2`: リクエストデータのJSON形式エラー。
*   `-5`: モデルの初期化失敗 (例: `.axmodel` ファイルの問題)。
*   `-6`: セットアップ中の設定エラー (例: モデルのJSON設定ファイルが見つからない、または破損している)。
*   `-11`: 推論中のモデル実行失敗。
*   `-20`: リンク操作失敗。
*   `-21`: タスクがいっぱいです (最大タスク数、通常YOLOでは1、に達しました)。
*   `-23`: 画像データのBase64デコードエラー。
*   `-24`: 推論データが空です。
*   `-25`: ストリームデータインデックスエラー (入力がストリーミングの場合)。
