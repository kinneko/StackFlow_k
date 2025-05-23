# llm-vlm - マルチモーダル推論ユニット

`llm-vlm`ユニットは、LLMフレームワークの中核コンポーネントであり、強力なマルチモーダル推論サービスを提供するために設計されています。テキストと画像の両方の入力を処理し、テキストベースの応答を生成できます。このユニットは高度に設定可能であり、キーワードスポッティング（KWS）や自動音声認識（ASR）などの他のユニットとリンクして、洗練されたAIパイプラインを作成できます。

## 主な機能

*   **マルチモーダル入力**: 推論のためにテキストおよび画像データを受け入れます。
*   **柔軟な設定**: モデル、応答形式、および運用パラメータの詳細なセットアップが可能です。
*   **ストリーミングサポート**: 入力と出力の両方で、ストリーミングデータと非ストリーミングデータの両方を処理します。
*   **動的リンキング**: 他のサービスユニットと動的にリンクおよびアンリンクできます。
*   **タスク管理**: 複数の推論タスクをサポートし、それらのステータスに関する情報を提供します。

## APIアクション

`llm-vlm`ユニットは、その動作を制御するためのいくつかのアクションを公開しています。

### 1. `setup`

`llm-vlm`タスクを初期化および設定します。これは、推論を実行する前の最初のステップです。

**リクエストJSON:**

```json
{
  "request_id": "2",
  "work_id": "vlm",
  "action": "setup",
  "object": "vlm.setup",
  "data": {
    "model": "internvl2.5-1B-ax630c",
    "response_format": "vlm.utf-8.stream",
    "input": ["vlm.utf-8", "kws.1000"],
    "enoutput": true,
    "max_token_len": 256,
    "prompt": "あなたは様々な質問に答え、情報を提供できる知識豊富なアシスタントです。",
    "tokenizer_type": "TKT_Qwen", // 例: TKT_LLaMa, TKT_MINICPM, TKT_Phi3, TKT_Qwen, TKT_HTTP など
    "filename_tokenizer_model": "qwen.tiktoken", // トークナイザモデルのパスまたはHTTPエンドポイント
    "filename_tokens_embed": "model_embed.bin", // トークン埋め込みへのパス
    "filename_post_axmodel": "model_post.axmodel", // 後処理AXModelへのパス
    "filename_vpm_resampler_axmodedl": "model_vpm_resampler.axmodel", // VPMリサンプラーAXModelへのパス
    "template_filename_axmodel": "model_layer_{}.axmodel", // AXModelレイヤーファイルのテンプレート
    "b_use_topk": true,
    "b_vpm_two_stage": false,
    "b_bos": true, // 文頭トークンを追加
    "b_eos": true, // 文末トークンを追加
    "axmodel_num": 2, // AXModelレイヤーの数
    "tokens_embed_num": 1,
    "img_token_id": 27766,
    "tokens_embed_size": 4096,
    "b_use_mmap_load_embed": true,
    "b_dynamic_load_axmodel_layer": false,
    "temperature": 1.0,
    "top_p": 0.9,
    "vpm_width": 448,
    "vpm_height": 448
  }
}
```

**`data`内のパラメータ:**

*   `request_id` (string, 必須): リクエストの一意の識別子。
*   `work_id` (string, 必須): 新しいタスクを作成する場合は `vlm` に設定する必要があります。特定の `work_id` (例: `vlm.1003`) が指定された場合、その既存のタスクの再設定を試みます。
*   `action` (string, 必須): `setup` に設定します。
*   `object` (string, 必須): データ型、通常は `vlm.setup`。
*   `model` (string, 必須): 使用するマルチモーダルモデルを指定します (例: `internvl2.5-1B-ax630c`)。システムは、この名前に基づいて事前定義されたパス (`base_model_path_` および `base_model_config_path_`) からモデル設定ファイルをロードします。
*   `response_format` (string, 必須): 出力の形式を定義します。
    *   `vlm.utf-8`: 非ストリーミング、単一のUTF-8エンコードされたテキスト応答。
    *   `vlm.utf-8.stream`: ストリーミング、複数のUTF-8エンコードされたテキストチャンク。
*   `input` (string または stringの配列, 必須): 入力ソースと形式を指定します。
    *   `vlm.utf-8`: VLMへの直接テキスト入力。
    *   `vlm.jpeg.stream.base64`: Base64エンコードされたJPEG画像データ。
    *   `kws.xxxx`: キーワードスポッティングユニットへのリンク (例: `kws.1000`)。
    *   `asr.xxxx`: 自動音声認識ユニットへのリンク (例: `asr.1001`)。
    *   `whisper.xxxx`: Whisper ASRユニットへのリンク。
*   `enoutput` (boolean, 必須): `true` の場合、ユニットは出力を送信します。`false` の場合、出力は抑制されます。
*   `max_token_len` (integer, オプション): 生成される応答の最大トークン数。指定しない場合は、モデル固有の値がデフォルトになります。
*   `prompt` (string, オプション): モデルの動作をガイドするためのシステムプロンプト。
*   **モデル設定パラメータ (オプション、通常はモデルのJSON設定ファイルからロードされますが、ここで上書きできます):**
    *   `tokenizer_type`: トークナイザの種類 (例: `TKT_LLaMa`, `TKT_Qwen`)。
    *   `filename_tokenizer_model`: トークナイザモデルのパスまたはHTTP URL。HTTP URLが指定された場合、ローカルのPythonトークナイザサーバーが起動されることがあります。
    *   `filename_tokens_embed`: トークン埋め込みファイルへのパス。
    *   `filename_post_axmodel`: 後処理AXModelファイルへのパス。
    *   `filename_vpm_resampler_axmodedl`: Vision Preprocessing Module (VPM) リサンプラーAXModelへのパス。
    *   `template_filename_axmodel`: AXModelレイヤーをロードするためのテンプレート文字列 (例: `model_layer_{}.axmodel`)。
    *   `b_use_topk` (boolean): top-kサンプリングを使用するかどうか。
    *   `b_vpm_two_stage` (boolean): VPMが2段階プロセスを使用するかどうか。
    *   `b_bos` (boolean): 入力に文頭トークン (BOS) を追加します。
    *   `b_eos` (boolean): 入力に文末トークン (EOS) を追加します。
    *   `axmodel_num` (integer): AXModelレイヤーの数。
    *   `tokens_embed_num` (integer): トークン埋め込みファイル/セクションの数。
    *   `img_token_id` (integer): 入力シーケンスで画像を表すために使用される特別なトークンID。
    *   `tokens_embed_size` (integer): トークン埋め込みのサイズ。
    *   `b_use_mmap_load_embed` (boolean): 埋め込みのロードにメモリマッピングを使用します (高速)。
    *   `b_dynamic_load_axmodel_layer` (boolean): 必要に応じてAXModelレイヤーを動的にロードします。
    *   `temperature` (float): 生成におけるランダム性を制御します。値が高いほどランダム性が高くなります。
    *   `top_p` (float): Nucleusサンプリングパラメータ。
    *   `vpm_width` (integer): VPMによる画像前処理のターゲット幅。
    *   `vpm_height` (integer): VPMによる画像前処理のターゲット高さ。

**応答JSON (成功時):**

```json
{
  "created": 1737599810, // 作成時のUnixタイムスタンプ
  "data": "None",
  "error": {
    "code": 0,
    "message": ""
  },
  "object": "None",
  "request_id": "2",
  "work_id": "vlm.1003" // このタスクインスタンスに割り当てられた一意のwork_id
}
```

**応答JSON (エラー時):**
`task_count_` 制限に達した場合:
```json
{
  "created": 1737599811,
  "data": "None",
  "error": {
    "code": -21,
    "message": "task full"
  },
  "object": "None",
  "request_id": "2",
  "work_id": "vlm"
}
```
モデルのロードに失敗した場合:
```json
{
  "created": 1737599812,
  "data": "None",
  "error": {
    "code": -5,
    "message": "Model loading failed."
  },
  "object": "None",
  "request_id": "2",
  "work_id": "vlm"
}
```

### 2. `inference`

設定済みの `llm-vlm` タスクにデータ (テキストまたは画像) を送信して処理させます。

**リクエストJSON (テキスト入力):**

```json
{
  "request_id": "4",
  "work_id": "vlm.1003", // setupで取得したターゲットwork_id
  "action": "inference",
  "object": "vlm.utf-8.stream", // 非ストリーミングの場合は "vlm.utf-8"
  "data": {
    "delta": "あなたの名前は何ですか？", // テキスト入力チャンク
    "index": 0, // 現在のチャンクのインデックス (ストリーミング用)
    "finish": true // これが最後のチャンクの場合はtrue
  }
}
```

**リクエストJSON (画像入力 - Base64エンコードされたJPEG):**

```json
{
  "request_id": "4",
  "work_id": "vlm.1003",
  "action": "inference",
  "object": "vlm.jpeg.stream.base64", // Base64エンコードされたJPEG画像を示す
  "data": {
    "delta": "U29tZSByYW5kb20gYmFzZTY0IGRhdGEgZm9yIHlvdSB0byBjb25zdWx0IGFwcGxpY2F0aW9ucy4=", // Base64データ
    "index": 0,
    "finish": true
  }
}
```
> **注意:** 画像データの場合、`delta` には完全なBase64エンコードされた画像データを含め、`finish` は `true` にする必要があります。画像がより大きな入力シーケンスの一部である場合、ストリーミングの `index` は引き続き使用される可能性がありますが、画像自体は1つの "delta" として送信されます。

**`data`内のパラメータ (ストリーミング `object` タイプの場合):**
*   `delta` (string): データチャンク。テキストの場合、これはユーザーのクエリの一部です。画像の場合、これはBase64エンコードされた画像データです。
*   `index` (integer): データチャンクのシーケンシャルインデックス。0から始まります。
*   `finish` (boolean): これが現在の入力シーケンスの最後のチャンクである場合は `true` に設定します。

**応答JSON (非ストリーミング - `vlm.utf-8`):**

```json
{
  "created": 1737600915,
  "data": "私はリトルAIという名前のAIアシスタントです。今日はどのようにお手伝いできますか？", // 完全な応答
  "error": {
    "code": 0,
    "message": ""
  },
  "object": "vlm.utf-8",
  "request_id": "4",
  "work_id": "vlm.1003"
}
```

**応答JSON (ストリーミング - `vlm.utf-8.stream`):**
応答の一部を含む一連のメッセージ。

```json
// メッセージ1
{
  "created": 1737600539,
  "data": {
    "delta": "私は", // 応答の一部
    "finish": false,    // さらなるデータが続くことを示す
    "index": 0          // この応答チャンクのインデックス
  },
  "error": {"code": 0, "message": ""},
  "object": "vlm.utf-8.stream",
  "request_id": "4",
  "work_id": "vlm.1003"
}
```
```json
// ... さらなるメッセージ ...
```
```json
// ストリームの最後のメッセージ
{
  "created": 1737600540,
  "data": {
    "delta": "",       // 最後のメッセージの空のdelta
    "finish": true,    // これがストリームの終わりであることを示す
    "index": 4
  },
  "error": {"code": 0, "message": ""},
  "object": "vlm.utf-8.stream",
  "request_id": "4",
  "work_id": "vlm.1003"
}
```

### 3. `link`

別のユニット (例: `kws`, `asr`) の出力をこの `llm-vlm` タスクへの入力として接続します。これにより、`llm-vlm` ユニットはシステム内の他の部分からのイベントやデータに反応できます。

**リクエストJSON:**

```json
{
  "request_id": "3",
  "work_id": "vlm.1003", // リンク先のllm-vlmタスク
  "action": "link",
  "object": "work_id",   // 'data'にwork_idが含まれることを指定
  "data": "kws.1000"   // リンク元のユニットのwork_id (例: KWSタスク)
}
```

**動作:**
*   `kws.xxxx` がリンクされている場合: KWSユニットがキーワードを検出すると、`llm-vlm` は現在の推論を停止し (もしあれば)、KWS出力を使用するか、単に後続のASR入力をリッスンするようにトリガーされる可能性があります。`main.cpp` の `kws_awake` 関数は `lLaMa_->Stop()` を呼び出すことでこれを処理します。
*   `asr.xxxx` (または `whisper.xxxx`) がリンクされている場合: ASRユニットからの認識されたテキストが、推論のために `llm-vlm` タスクへの入力として供給されます。`task_asr_data` 関数がこれを処理します。

**応答JSON (成功時):**

```json
{
  "created": 1737599866,
  "data": "None",
  "error": {"code": 0, "message": ""},
  "object": "None",
  "request_id": "3",
  "work_id": "vlm.1003"
}
```
**応答JSON (エラー時):**
リンクターゲットが無効であるか、リンキングに失敗した場合:
```json
{
  "created": 1737599867,
  "data": "None",
  "error": {"code": -20, "message": "link false"},
  "object": "None",
  "request_id": "3",
  "work_id": "vlm.1003"
}
```

> **注意:** リンキングは、`input` 配列に他のユニットの `work_id` を含めることにより、`setup` フェーズ中に設定することもできます。

### 4. `unlink`

以前にリンクされたユニットを切断します。

**リクエストJSON:**

```json
{
  "request_id": "4",
  "work_id": "vlm.1003",
  "action": "unlink",
  "object": "work_id",
  "data": "kws.1000" // アンリンクするユニットのwork_id
}
```

**応答JSON (成功時):**

```json
{
  "created": 1737600178,
  "data": "None",
  "error": {"code": 0, "message": ""},
  "object": "None",
  "request_id": "4",
  "work_id": "vlm.1003"
}
```

### 5. `pause`

`llm-vlm` タスクを一時的に停止します。進行中の推論はすべて停止されます。ユニットは一時停止中は新しい入力データを処理しません。

**リクエストJSON:**

```json
{
  "request_id": "5",
  "work_id": "vlm.1003",
  "action": "pause"
}
```

**応答JSON (成功時):**

```json
{
  "created": 1731488402,
  "data": "None",
  "error": {"code": 0, "message": ""},
  "object": "None",
  "request_id": "5",
  "work_id": "vlm.1003"
}
```

### 6. `work`

一時停止中の `llm-vlm` タスクを再開します。ユニットは新しい入力データの処理を開始し、モデルが再開をサポートしている場合は以前に中断されたタスクを続行できます (通常は最初からやり直します)。

**リクエストJSON:**

```json
{
  "request_id": "6",
  "work_id": "vlm.1003",
  "action": "work"
}
```
> **注意:** `llm_llm` クラス (`llm_task` インスタンスを管理するクラス) の `main.cpp` 実装には、現在 `StackFlow::work` を直接オーバーライドする明示的な `work` メソッドはありません。作業の再開は、通常、`pause` 後に新しい推論データを送信することによって処理されます。`StackFlow` 基本クラスには、デフォルトの `work` 実装がある可能性があります。

**応答JSON (成功時):**

```json
{
  "created": 1737600236,
  "data": "None",
  "error": {"code": 0, "message": ""},
  "object": "None",
  "request_id": "6",
  "work_id": "vlm.1003"
}
```

### 7. `exit`

`llm-vlm` タスクを終了し、そのリソースを解放します。これには、モデルの非初期化や、Pythonトークナイザサーバーなどの関連プロセスの停止が含まれます。

**リクエストJSON:**

```json
{
  "request_id": "7",
  "work_id": "vlm.1003",
  "action": "exit"
}
```

**応答JSON (成功時):**

```json
{
  "created": 1737600704,
  "data": "None",
  "error": {"code": 0, "message": ""},
  "object": "None",
  "request_id": "7",
  "work_id": "vlm.1003"
}
```
指定された `work_id` が存在しない場合:
```json
{
  "created": 1737600705,
  "data": "None",
  "error": {
    "code": -6,
    "message": "Unit Does Not Exist"
  },
  "object": "None",
  "request_id": "7",
  "work_id": "vlm.1003" // 要求されたwork_id
}
```

### 8. `taskinfo`

実行中の `llm-vlm` タスクに関する情報を取得します。

**リクエストJSON (すべての `vlm` タスクのリストを取得):**

```json
{
  "request_id": "2",
  "work_id": "vlm", // vlmユニットタイプの一般的なwork_id
  "action": "taskinfo"
}
```

**応答JSON (タスクのリスト):**

```json
{
  "created": 1737600076,
  "data": [
    "vlm.1003", // アクティブなwork_idのリスト
    "vlm.1004"
  ],
  "error": {"code": 0, "message": ""},
  "object": "vlm.tasklist",
  "request_id": "2",
  "work_id": "vlm"
}
```

**リクエストJSON (特定のタスクパラメータを取得):**

```json
{
  "request_id": "2",
  "work_id": "vlm.1003", // タスクの特定のwork_id
  "action": "taskinfo"
}
```

**応答JSON (特定のタスクパラメータ):**

```json
{
  "created": 1737600098,
  "data": {
    "model": "internvl2.5-1B-ax630c",
    "response_format": "vlm.utf-8.stream",
    "enoutput": true,
    "inputs": [ // 現在の入力ソース/リンク
      "vlm.utf-8",
      "kws.1000"
    ]
    // max_token_len、promptなどの他のセットアップパラメータも含まれる場合があります。
  },
  "error": {"code": 0, "message": ""},
  "object": "vlm.taskinfo",
  "request_id": "2",
  "work_id": "vlm.1003"
}
```
特定のタスク情報に対して指定された `work_id` が存在しない場合:
```json
{
  "created": 1737600099,
  "data": "None", // またはdata内の特定のエラーオブジェクト
  "error": {
    "code": -6,
    "message": "Unit Does Not Exist"
  },
  "object": "None", // または "data" にエラー詳細を含む "vlm.taskinfo"
  "request_id": "2",
  "work_id": "vlm.1003"
}
```

## プロンプトのフォーマット

`main.cpp` の `prompt_complete` メソッドは、`tokenizer_type` に基づいて異なるプロンプト構造が使用されることを示しています。
*   **TKT_LLaMa:** `<|user|>\n{input}</s><|assistant|>\n`
*   **TKT_MINICPM:** `<ユーザー>{input}<AI>`
*   **TKT_Phi3:** `{input} ` (スペースが追加されます)
*   **TKT_Qwen:** `<|im_start|>system\n{prompt}.<|im_end|>\n<|im_start|>user\n{input}<|im_end|>\n<|im_start|>assistant\n`
*   **TKT_HTTP/デフォルト:** 入力はそのまま使用されます。

`{prompt}` プレースホルダーは `setup` 設定の `prompt` フィールドから取得され、`{input}` は `inference` 呼び出しからのユーザー提供データです。

## エラーコード

一般的に発生するエラーコード:
*   `0`: エラーなし、成功。
*   `-2`: リクエストのJSON形式エラー。
*   `-5`: モデルのロード失敗 (例: モデルファイルが見つからない、設定の問題)。
*   `-6`: ユニット/タスクが存在しません (例: 指定された `work_id` が無効、またはタスクが終了している)。
*   `-11`: モデルの実行失敗 (一般的な推論エラー)。
*   `-20`: リンク操作失敗。
*   `-21`: タスクがいっぱいです (最大タスク数に達しました。`task_count_` によって制御されます)。
*   `-23`: Base64デコードエラー。
*   `-25`: ストリームデータのインデックスエラー (例: チャンクの欠落または順序不正)。

## 内部操作 (`main.cpp` より)

*   **トークナイザサーバー**: 特定のトークナイザタイプ (例: `TKT_HTTP` または `filename_tokenizer_model` がHTTP URLの場合) の場合、`fork` と `execl` を使用してPythonベースのトークナイザサーバーが自動的に起動されることがあります。このサーバーはローカルポート (8090から始まり、それを必要とする各タスクに対してインクリメントされます) でリッスンします。
*   **モデル設定のロード**: モデル固有のパラメータ (モデルファイルへのパス、埋め込みサイズなど) は、ベースモデルパス (例: `/opt/m5stack/models/`) と `setup` 呼び出しで指定されたモデル名を組み合わせた場所にあるJSON設定ファイルからロードされます。
*   **画像処理**: 画像データ (`vlm.jpeg.stream.base64`) を受信すると、Base64からデコードされます。生の画像バイトは `lLaMa_->Encode(src, img_embed)` に渡されて画像埋め込みを取得し、その後、マルチモーダル推論のためにテキストプロンプト埋め込みと結合されます。
*   **コールバックメカニズム**: `llm_llm` クラスは `std::bind` を使用して、出力データ (`task_output`)、リンクされたユニットからのデータ (`task_asr_data`, `kws_awake`)、およびユーザーデータ (`task_user_data`) を処理するためのコールバックを作成します。

> **注意:** タスクの `work_id` (例: `vlm.1003`) は順次生成されます。数値部分は固定インデックスではなく、インクリメントカウンターです。
