# llm-camera (カメラユニット)

`llm-camera`ユニットは、USB V4L2デバイスや専用のAXERA MIPIカメラなど、様々なソースからのビデオキャプチャを管理します。このビデオデータをZMQ PUB/SUB経由でストリーミングしたり、エンコードされたフレームをAPI呼び出しでプッシュしたり、HTTP経由で提供（MJPEGウェブストリーム）したり、RTSPストリームを提供したりするメカニズムを提供します。

## API呼び出しの共通規約

`llm-camera`へのAPI呼び出しはJSONメッセージを介して行われます。

### リクエスト構造

`llm-camera` APIへの標準的なリクエストは以下の構造に従います：

```json
{
  "request_id": "クライアント生成のUUIDまたはカウンター",
  "work_id": "cameraまたはcamera.インスタンスID",
  "action": "APIアクション名",
  "object": "オプションのオブジェクトタイプ文字列", // 該当する場合
  "data": { /* API固有のペイロード */ }     // または一部APIでは文字列
}
```

-   `request_id` (string, 必須): リクエストとレスポンスを関連付けるための、クライアントが生成する一意の識別子。
-   `work_id` (string, 必須):
    -   初期設定や一般的なクエリ（`list_camera`など）の場合: `"camera"`。
    -   特定のカメラタスクインスタンスに対する操作の場合: `"camera.XXXX"` (例: `"camera.1003"`)。`XXXX`は成功した`setup`呼び出しによって返されるインスタンスIDです。
-   `action` (string, 必須): 呼び出すAPIアクション (例: `"setup"`, `"list_camera"`)。
-   `object` (string, オプション): 送信するデータのタイプやコンテキストを指定します（該当する場合）。例：設定データの場合は`"camera.setup"`。
-   `data` (object または string, オプション): API呼び出しのペイロード。構造はAPIによって異なります。

### レスポンス構造

**成功レスポンス:**

API呼び出しが受け入れられ、正常に処理されたことを示します。

```json
{
  "request_id": "リクエストからミラーリング",
  "work_id": "cameraまたはcamera.インスタンスID", // ミラーリングまたは更新 (例: setup後の "camera.1003")
  "action": "APIアクション名",              // リクエストからミラーリング
  "created": 1678886400,                    // 整数: レスポンス生成のUnixタイムスタンプ
  "code": 0,                                // 整数: 0は成功を示す
  "message": "OK",                          // 文字列: 成功メッセージ
  "data": { /* API固有データ */ }       // オプション: APIによって返されるペイロード
}
```
-   API呼び出しが新しいカメラタスクインスタンスの作成に成功した場合（例：`setup`）、レスポンスの`work_id`は新しいインスタンスID（例：`"camera.1003"`）になり、多くの場合`data`オブジェクトにも含まれます。
-   レスポンスのデータ型を示すために`object`フィールドが存在する場合もあります（例：`list_camera`の場合は`"camera.devices"`）。

**エラーレスポンス:**

API呼び出し処理中の失敗を示します。

```json
{
  "request_id": "リクエストからミラーリング",
  "work_id": "cameraまたはcamera.インスタンスID",
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
-   `-5`: デバイスのオープンまたは設定に失敗しました（例：カメラが見つからない、デバイスでサポートされていないパラメータ）。これは、`setup`で基本的なカメラ操作が失敗した場合の一般的なメッセージです。
-   `-6`: 指定されたカメラタスクインスタンス（例：`"camera.1003"`）が存在しません。
-   `-21`: タスク制限に達しました（現在、`MAX_TASK_NUM`は1なので、一度に1つのカメラタスクインスタンスのみサポートされます）。
-   `-22`: サポートされていないフォーマットまたは設定が要求されました（例：無効な`response_format`または`rtsp`文字列パラメータ）。
-   `-23`: カメラ操作中のフレーム取得、エンコード、またはストリーミングエラー。

## カメラデータ出力と通知

`llm-camera`ユニットは、キャプチャされたビデオフレームにアクセスするためのいくつかの方法を提供します。これらは通常、非同期のデータストリームです。

### 1. ZMQ PUB/SUB (生フレームまたはリサイズフレーム)

-   **トピック**: ZMQ PUBトピックは、カメラタスクインスタンスの`work_id`です (例: `"camera.1003"`)。
-   **データフォーマット**: 生のYUYV422ビデオフレーム。
-   **解像度**: フレームは、`setup`呼び出しで指定された`frame_width`と`frame_height`に対応します。これらの次元がカメラのネイティブキャプチャ解像度と異なる場合、`llm-camera`ユニットは要求された出力サイズに合わせて画像を（中央から）クロップまたはパディングしようとします。
-   **使用法**: クライアントは、リアルタイム処理や表示のためにフレームの連続ストリームを受信するために、このZMQトピックを購読できます。

### 2. APIプッシュ (`enoutput: true`によるユーザー出力)

`setup`中に`enoutput`が`true`に設定されている場合、カメラタスクインスタンスは、エンコードされた画像データをAPI呼び出しを行ったクライアントに（または設定済みのZMQチャネル経由で）非同期にプッシュします。

-   **フォーマット**: `setup`呼び出しの`response_format`で指定されます (例: `"image.yuyv422.base64"`, `"image.jpeg.base64"`)。
-   **フレームごとのJSON構造**:
    ```json
    {
      "request_id": "<元のsetupリクエストID>", // 通常、このタスクを作成したsetup呼び出しのrequest_id
      "work_id": "camera.XXXX",                  // 特定のカメラタスクインスタンスID
      "object": "image.jpeg.base64",             // データのフォーマット (例: "image.yuyv422.base64")
      "error": {"code": 0, "message": ""},       // このデータプッシュの成功を示す
      "data": "<Base64エンコードされた画像データ>"
    }
    ```
    (注意: ここでの`error`フィールドは、この特定のデータプッシュのステータスを示し、一般的なAPIエラーではありません。`request_id`は、このタスクインスタンスを作成した初期の`setup`リクエストに関連付けられたチャネルオブジェクトから継承されます。)

### 3. Webストリーム (`enable_webstream: true`経由)

`setup`中に`enable_webstream`が`true`に設定されている場合、JPEG画像をストリーミングするためのHTTPサーバーが起動します。

-   **URL**: `http://{デバイスIP}:8989/` (`{デバイスIP}`は`llm-camera`を実行しているデバイスのIPアドレス)。
-   **フォーマット**: `multipart/x-mixed-replace`を使用したMJPEGストリーム。各パートにはJPEG画像が含まれます。
-   **使用法**: WebブラウザやMJPEGクライアントで表示できます。

### 4. RTSPストリーム (`rtsp`パラメータ経由)

互換性のあるAXERA MIPIカメラの場合、RTSPストリームを有効にできます。

-   **URL**: `rtsp://{デバイスIP}:8554/axstream0`
-   **設定**: `setup`呼び出しの`rtsp`パラメータがストリーム特性を定義します。
    -   フォーマット: `"rtsp.幅x高さ.コーデック"`
    -   例: `"rtsp.1280x720.h265"` は1280x720解像度のH.265ストリームを有効にします。
    -   サポートされるコーデック: `h264`, `h265`。
-   **注意**: この機能は主にAXERA MIPIカメラ向けに意図され、テストされています。UVCカメラは通常、このメカニズムによる直接のRTSPストリーミングをサポートしていません。エンコーダ設定は主に`camera.json`のデフォルトによって決定されます。

## APIリファレンス

### **list_camera**

システムで見つかったV4L2 (Video4Linux2) カメラデバイスをリストします。これは、`setup` APIの正しい`input`デバイス名を見つけるのに役立ちます。

-   リクエストの`work_id`: `"camera"`

**リクエストパラメータ**: 標準の`request_id`, `work_id`, `action`以外はありません。

**リクエスト例:**
```json
{
  "request_id": "list_cam_001",
  "work_id": "camera",
  "action": "list_camera"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "list_cam_001",
  "work_id": "camera",
  "action": "list_camera",
  "created": 1678886400,
  "code": 0,
  "message": "OK",
  "object": "camera.devices", // 返されるデータのタイプを示す
  "data": {
    "devices": [
      "/dev/video0",
      "/dev/video1"
      // 追加のデバイスがあればここに表示
    ]
  }
}
```

**レスポンス (エラー):**
```json
{
  "request_id": "list_cam_001",
  "work_id": "camera",
  "action": "list_camera",
  "created": 1678886401,
  "error": {
    "code": -1, // エラーコード例
    "message": "カメラデバイスのリスト表示に失敗しました"
  }
}
```

### **setup**

新しいカメラタスクインスタンスを初期化し、設定します。通常、一度に1つのタスクインスタンスのみサポートされます (`MAX_TASK_NUM = 1` in `main.cpp`)。

-   リクエストの`work_id`: `"camera"`
-   レスポンスの`work_id` (成功時): `"camera.XXXX"` (例: `"camera.1003"`)

**リクエストパラメータ:**

| パラメータ          | 型      | 必須 | デフォルト         | 説明                                                                                                                                                               |
|--------------------|---------|------|-----------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `object`           | string  | はい | N/A             | `camera.setup`である必要があります。                                                                                                                                     |
| `data.input`       | string  | はい | N/A             | カメラデバイス識別子。例: `"/dev/video0"` (UVCカメラ用), `"axera_single_sc850sl"` (特定のAXERA MIPIカメラ用)。UVCデバイスを見つけるには`list_camera`を使用します。 |
| `data.response_format` | string  | はい | N/A             | `enoutput`がtrueの場合のAPIプッシュ用出力フォーマット。サポート: `"image.yuyv422.base64"`, `"image.jpeg.base64"`。これはZMQ PUB/SUB (常にYUYV422) やRTSPには影響しません。 |
| `data.frame_width` | integer | はい | N/A             | ZMQ PUB/SUBおよびAPIプッシュ用の出力ビデオフレームの希望幅。カメラはこの解像度を設定しようとします。正確な解像度がサポートされていない場合、最も近いサポート解像度からこの出力に一致するようにクロッピング/パディングが適用されることがあります。RTSPの場合、幅は`rtsp`文字列経由で設定されます。 |
| `data.frame_height`| integer | はい | N/A             | ZMQ PUB/SUBおよびAPIプッシュ用の出力ビデオフレームの希望高。`frame_width`と同様の解像度一致動作。RTSPの場合、高さは`rtsp`文字列経由で設定されます。 |
| `data.enoutput`    | boolean | いいえ | `false`         | `true`の場合、エンコードされたフレーム（`response_format`によるフォーマット）のクライアントへの非同期プッシュを有効にします。データ量が多くなる可能性があるため注意して使用してください。                 |
| `data.enable_webstream` | boolean | いいえ | `false`         | `true`の場合、`http://{デバイスIP}:8989/`でMJPEGウェブストリームを有効にします。                                                                                                   |
| `data.rtsp`        | string  | いいえ | `""` (無効)     | 互換性のあるAXERA MIPIカメラでRTSPストリーミングを有効にします。フォーマット: `"rtsp.幅x高さ.コーデック"`、例: `"rtsp.1280x720.h265"` または `"rtsp.1920x1080.h264"`。           |
| `data.jpeg_quality` | integer | いいえ | 約80-90 (HAL/libjpegのデフォルト) | JPEGエンコード用（`image.jpeg.base64`およびウェブストリームで使用）。品質は0-100。実際のデフォルトはOpenCV/libjpegまたはカメラHALに依存します。全てのカメラタイプで上書きがサポートされているわけではありません。 |
| `data.h26x_bitrate` | integer | いいえ | (`camera.json`より) | H.264/H.265 RTSPストリームのターゲットビットレート(kbps)。主に`camera.json`で設定。ここでの上書きはHALによっては限定的な効果しかない場合があります。                                                                                             |
| `data.h26x_gop`    | integer | いいえ | (`camera.json`より) | H.264/H.265 RTSPストリームのGOP (Group of Pictures)サイズ。主に`camera.json`で設定。ここでの上書きは限定的な効果しかない場合があります。                                                                                      |

**リクエスト例 (UVCカメラ APIプッシュおよびZMQ用):**
```json
{
  "request_id": "setup_uvc_002",
  "work_id": "camera",
  "action": "setup",
  "object": "camera.setup",
  "data": {
    "input": "/dev/video0",
    "response_format": "image.jpeg.base64",
    "frame_width": 640,
    "frame_height": 480,
    "enoutput": true,
    "jpeg_quality": 85
  }
}
```

**リクエスト例 (AXERA MIPIカメラ RTSP用):**
```json
{
  "request_id": "setup_axera_003",
  "work_id": "camera",
  "action": "setup",
  "object": "camera.setup",
  "data": {
    "input": "axera_single_sc850sl",
    "response_format": "image.yuyv422.base64", // enoutput=falseの場合、使用されない可能性があっても必須
    "frame_width": 1280, // RTSP固有の内部ロジックで上書きされない限りZMQに使用
    "frame_height": 720,
    "enoutput": false,
    "rtsp": "rtsp.1920x1080.h264" // RTSPは1920x1080 H.264でストリーミング
  }
}
```

**レスポンス (成功):**
```json
{
  "request_id": "setup_uvc_002",
  "work_id": "camera.1003", // 新しいカメラタスクインスタンスID
  "action": "setup",
  "created": 1678886401,
  "code": 0,
  "message": "OK",
  "data": {
    "work_id": "camera.1003" // 作成されたインスタンスIDを確認
  }
}
```

**レスポンス (エラー - デバイスオープン/設定失敗):**
```json
{
  "request_id": "setup_cam_004",
  "work_id": "camera",
  "action": "setup",
  "created": 1678886402,
  "error": {
    "code": -5,
    "message": "モデルの読み込みに失敗しました。" // 一般的なメッセージ、実質的にはカメラ設定失敗を示す
  }
}
```

**レスポンス (エラー - タスク制限到達):**
```json
{
  "request_id": "setup_cam_005",
  "work_id": "camera",
  "action": "setup",
  "created": 1678886403,
  "error": {
    "code": -21,
    "message": "task full"
  }
}
```

### **exit**

カメラタスクインスタンスを停止し、削除してリソースを解放します（カメラデバイスのクローズ、ストリームの停止を含む）。

-   リクエストの`work_id`: `"camera.XXXX"` (特定のカメラタスクインスタンスID)

**リクエストパラメータ**: 標準の`request_id`, `work_id`, `action`以外はありません。

**リクエスト例:**
```json
{
  "request_id": "exit_cam_006",
  "work_id": "camera.1003",
  "action": "exit"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "exit_cam_006",
  "work_id": "camera.1003", // 終了したタスクのwork_id
  "action": "exit",
  "created": 1678886403,
  "code": 0,
  "message": "OK"
}
```

**レスポンス (エラー - タスクが見つからない):**
```json
{
  "request_id": "exit_cam_007",
  "work_id": "camera.9999", // 存在しないタスク
  "action": "exit",
  "created": 1678886404,
  "error": {
    "code": -6,
    "message": "ユニットが存在しません"
  }
}
```

### **taskinfo**

アクティブなカメラタスクに関する情報を取得します。動作はリクエストの`work_id`に依存します。

**ケース1: 一般的なカメラユニット情報**

-   リクエストの`work_id`: `"camera"`
-   **説明**: 全てのアクティブなカメラタスクインスタンスIDのリストを返します。`MAX_TASK_NUM`は通常1なので、リストには1つのインスタンスIDが含まれるか空になります。

**リクエスト例:**
```json
{
  "request_id": "taskinfo_cam_general_008",
  "work_id": "camera",
  "action": "taskinfo"
}
```

**レスポンス (成功 - 1タスクアクティブ時):**
```json
{
  "request_id": "taskinfo_cam_general_008",
  "work_id": "camera",
  "action": "taskinfo",
  "created": 1678886404,
  "code": 0,
  "message": "OK",
  "object": "camera.tasklist",
  "data": [
    "camera.1003" // アクティブなインスタンスIDのリスト
  ]
}
```

**ケース2: 特定のカメラタスクインスタンス情報**

-   リクエストの`work_id`: `"camera.XXXX"` (例: `"camera.1003"`)
-   **説明**: 指定されたカメラタスクインスタンスのランタイムパラメータと設定を返します。

**リクエスト例:**
```json
{
  "request_id": "taskinfo_cam_specific_009",
  "work_id": "camera.1003",
  "action": "taskinfo"
}
```

**レスポンス (成功):**
```json
{
  "request_id": "taskinfo_cam_specific_009",
  "work_id": "camera.1003",
  "action": "taskinfo",
  "created": 1678886405,
  "code": 0,
  "message": "OK",
  "object": "camera.taskinfo",
  "data": {
    "input": "/dev/video0",
    "response_format": "image.jpeg.base64",
    "enoutput": true,
    "frame_width": 640,
    "frame_height": 480,
    "enable_webstream": false,
    "rtsp": "" // 現在のRTSP設定文字列（アクティブな場合）
    // jpeg_qualityのような他のパラメータもインスタンスから読み取り可能であれば含まれる可能性あり
  }
}
```

## `camera.json`との関連性

`projects/llm_framework/main_camera/camera.json`ファイルは、主に`llm-camera`ユニットが使用するビデオエンコーダ（JPEG、H.264、H.265）のデフォルト設定パラメータを提供します。

-   **`jpeg_config_param`**: 様々なデフォルトJPEGエンコードパラメータが含まれています。これらは、`response_format`が`"image.jpeg.base64"`の場合やウェブストリームがアクティブな場合に、基盤となるカメラハードウェア抽象化レイヤ（HAL）や画像エンコードライブラリ（OpenCVの`imencode`など）によって使用される可能性があります。`setup` APIの`jpeg_quality`パラメータはこれに影響を与えることを試みることができますが、最終的な有効品質はHALの能力にも依存する可能性があります。
-   **`h264_config_param` / `h265_config_param`**: これらのオブジェクトは、それぞれH.264およびH.265ビデオエンコードの広範なデフォルト設定を定義します。これには以下が含まれます：
    -   ビデオ属性 (`stVencAttr.*`): 最大/ソース画像寸法、バッファサイズ、プロファイル、レベル、ティア。
    -   レート制御 (`stRcAttr.*`): GOPサイズ (`u32Gop`)、ビットレート (`u32BitRate`)、異なるレート制御モード（CBR、VBRなど）の最小/最大QP値。
    -   GOP構造 (`stGopAttr.*`)。
-   **RTSPでの使用**: `setup` APIの`rtsp`パラメータ（例：`"rtsp.1280x720.h265"`）を介してRTSPストリームが設定されると、`llm-camera`ユニットは`camera.json`から対応するH.264またはH.265のデフォルトパラメータをロードします。`rtsp`文字列で指定された解像度（例：1280x720）は、`camera.json`のデフォルト`u32PicWidthSrc`および`u32PicHeightSrc`を上書きします。ビットレート（`h26x_bitrate`）やGOPサイズ（`h26x_gop`）のような他のエンコードパラメータも、基盤となるハードウェアおよびソフトウェアスタックがそのような動的設定をサポートしている場合、`setup` API呼び出しで提供されることで`camera.json`のデフォルトを上書きできます。
-   **上書き**: `camera.json`はエンコーダ設定のベースラインを提供しますが、`setup` APIはある程度カスタマイズが可能です：
    -   RTSPの場合、`rtsp`文字列が直接解像度とコーデックを決定します。
    -   `setup` APIの`jpeg_quality`、`h26x_bitrate`、`h26x_gop`のようなパラメータは、`camera.json`のデフォルトを上書きすることを意図していますが、その有効性は特定のカメラハードウェアとそのドライバの能力に依存する可能性があります。

ほとんどのユーザーにとっては、`setup` APIで適切な`input`デバイス、`response_format`、希望する出力寸法（`frame_width`、`frame_height`）、および必要に応じてRTSP設定を選択することが主な操作となります。`camera.json`を直接変更することで、特に互換性のあるハードウェアでのRTSPストリームのデフォルトエンコーダ動作をより深く調整できます。
