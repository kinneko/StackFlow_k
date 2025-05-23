# llm-whisper (音声認識モジュール)

`llm-whisper`モジュールは、高度な音声テキスト変換コンポーネントです。Whisperモデルを活用して、多言語での文字起こしサービスを提供します。より複雑な音声対話シナリオのために、Voice Activity Detection (VAD) および Keyword Spotting (KWS) ユニットと連携できます。

## setup (セットアップ)

Whisper ASRタスクを設定し、初期化します。

JSONを送信:

```json
{
  "request_id": "2",
  "work_id": "whisper",
  "action": "setup",
  "object": "whisper.setup",
  "data": {
    "model": "whisper-tiny",
    "response_format": "asr.utf-8",
    "input": ["sys.pcm", "kws.1000", "vad.1001"],
    "language": "en",
    "enoutput": true,
    "awake_delay": 50,
    // "t2s_model_path": "models/whisper-tiny/t2s.json", // パスは通常モデルのJSON設定の一部
    "whisper_sample_rate": 16000, // 必要に応じて、さらにモデル固有のパラメータを設定可能
    "delay_audio_frame": 1000 // (VAD/KWSを使用しない場合)処理前にバッファリングするオーディオフレーム数
  }
}
```

**`data`内のパラメータ:**

*   `request_id` (string, 必須): リクエストの基本的なデータ説明のための参照。
*   `work_id` (string, 必須): 初期設定には `whisper` を使用します。セットアップ成功時に特定の `work_id` (例: `whisper.1002`) が返されます。
*   `action` (string, 必須): 呼び出されるメソッド。`setup` である必要があります。
*   `object` (string, 必須): 転送されるデータ型。通常は `whisper.setup` です。
*   `model` (string, 必須): 使用するWhisperモデル (例: `whisper-tiny`, `whisper-base`, `whisper-small`)。システムは、この名前に基づいて事前定義されたパス (例: `projects/llm_framework/main_whisper/`) からモデル設定ファイル (例: `mode_whisper-tiny.json`) をロードします。
*   `response_format` (string, 必須): 出力の形式を定義します。
    *   `asr.utf-8`: 非ストリーミング、単一のUTF-8エンコードされたテキスト応答。
    *   `asr.utf-8.stream`: ストリーミング、複数のUTF-8エンコードされたテキストチャンク。
*   `input` (string または stringの配列, 必須): 入力ソースと形式を指定します。
    *   `sys.pcm`: システムのオーディオ入力からの生のPCMオーディオデータ。ユニットは `audio` ユニットを呼び出して (通常は `unit_call("audio", "cap", input)` 経由で) データをキャプチャします。
    *   `whisper.pcm.stream.base64`: Base64エンコードされたPCMオーディオストリーム。
    *   `whisper.wav.stream.base64`: Base64エンコードされたWAVオーディオストリーム (ヘッダーは除去されます)。
    *   `whisper.mp3.stream.base64`: Base64エンコードされたMP3オーディオストリーム (デコードサポートは限定的か、プレースホルダーの可能性があります)。
    *   `kws.xxxx`: キーワードスポッティングユニットへのリンク (例: `kws.1000`)。リンクされている場合、Whisperタスクは一時停止状態で開始し (`ensleep_ = true`)、KWSイベントでアクティブになることがあります。
    *   `vad.xxxx`: Voice Activity Detectionユニットへのリンク (例: `vad.1001`)。VAD信号は `endpoint_flage_` を制御し、オーディオ処理セグメントを管理します。
*   `language` (string, 必須): モデルが認識する言語 (例: "en", "zh", "ja")。サポートされている言語コード (例: "sv", "sr", "no"。ソース内の `WHISPER_LANG_NAMES` に対応) とそれに対応する整数ID (`WHISPER_LANG_CODES`) のリストが内部で維持されます。`detect_language` 関数が文字列を適切なコードにマッピングします。
*   `enoutput` (boolean, 必須): `true` の場合、ユニットは文字起こし結果を送信します。`false` の場合、出力は抑制されます。
*   `awake_delay` (integer, オプション): KWSウェイクイベント後、オーディオ処理を開始するまでの遅延時間 (ミリ秒)。モデルの設定ファイルの値 (例: 50ms) またはこのフィールドの値がデフォルトになります。
*   `delay_audio_frame` (integer, オプション): VAD/KWSなしの `sys.pcm` または直接オーディオストリーム入力の場合、文字起こしを開始する前にバッファリングするオーディオフレーム数を指定します。`whisper.pcm.stream.base64` (または同様の直接ストリームで `input.find("stream.base64") != std::string::npos` の場合) が使用される場合、これはしばしば0に設定されます。デフォルトは1000です。

**モデル設定 (`model`名に基づいてJSONファイルからロード。例: `projects/llm_framework/main_whisper/mode_whisper-tiny.json`):**
これらのパラメータは、主にモデルのJSON設定ファイルで定義されます。
*   `encoder` (string): エンコーダモデルファイルへのパス (例: `encoder.axmodel`)。
*   `decoder_main` (string): メインデコーダモデルファイルへのパス (例: `decoder_main.axmodel`)。
*   `decoder_loop` (string): ループデコーダモデルファイルへのパス (例: `decoder_loop.axmodel`)。
*   `positional_embedding` (string): 位置エンベディングファイルへのパス (例: `positional_embedding.bin`)。
*   `tokens` (string): トークンファイルへのパス (例: `tokens_zh.txt`)。このファイルには、トークン文字列が含まれており、多くの場合Base64エンコードされ、IDとともに1行に1つずつ記述されています。
*   `model_type` (string): Whisperモデルのタイプ (例: "tiny", "base", "small")。これは `WHISPER_N_TEXT_STATE` のようなネットワークアーキテクチャパラメータを決定します (例: tiny:384, base:512, small:768)。
*   `t2s` (string, オプション):繁体字中国語から簡体字中国語への変換のためのOpenCC設定ファイルへのパス (例: `s2t.json`)。`language_` (setupの `language` から派生) が "en" または "ja" でない場合に適用されます。
*   `whisper_sample_rate` (integer): オーディオの期待されるサンプルレート (例: 16000 Hz)。
*   `whisper_n_fft` (integer): FFTウィンドウサイズ (例: 400)。
*   `whisper_hop_length` (integer): STFTのホップ長 (例: 160)。
*   `whisper_chunk_size` (integer): 処理用のオーディオチャンクサイズ (秒単位) (例: 30秒)。
*   `whisper_n_mels` (integer): 生成するメルバンドの数 (例: 80。ただし、ソースではあるインスタンスで400と表示されており、モデルごとに設定可能であると思われます)。
*   `whisper_sot` (integer): Start Of TranscriptionトークンID (例: 50258)。
*   `whisper_eot` (integer): End Of TranscriptionトークンID (例: 50257)。
*   `whisper_blank` (integer): ブランク トークンID (例: 220)。
*   `whisper_no_timestamps` (integer): タイムスタンプを抑制するトークンID (例: 50363)。
*   `whisper_no_speech` (integer): 無音を示すトークンID (例: 50362)。
*   `whisper_translate` (integer): 翻訳タスクのトークンID (例: 50358)。
*   `whisper_transcribe` (integer): 文字起こしタスクのトークンID (例: 50359)。
*   `whisper_vocab_size` (integer): 語彙サイズ (例: 51865)。
*   `whisper_n_text_ctx` (integer): テキストデコーダのコンテキストサイズ (例: 448)。
*   `awake_delay` (integer, オプション): setup `data` で指定されていない場合のデフォルトのウェイク遅延。

**応答JSON (成功時):**

```json
{
  "created": 1737597583,
  "data": "None",
  "error": {
    "code": 0,
    "message": ""
  },
  "object": "None",
  "request_id": "2",
  "work_id": "whisper.1002" // このASRタスクの一意のwork_id
}
```
**応答JSON (エラー時):**
`task_count_` (Whisperの場合は最大1) 制限に達した場合:
```json
{
  "created": ...,
  "data": "None",
  "error": {
    "code": -21,
    "message": "task full"
  },
  "object": "None",
  "request_id": "2",
  "work_id": "whisper"
}
```
モデルのロードに失敗した場合 (例: ファイルが見つからない、AXエンジンの初期化エラー):
```json
{
  "created": ...,
  "data": "None",
  "error": {
    "code": -5, // または -2 (JSONエラー), -3 (pos_embed), -4 (エンコーダ), -6 (デコーダ/設定) のような他のコード
    "message": "Model loading failed." // または "config false" のようなより具体的なメッセージ
  },
  "object": "None",
  "request_id": "2",
  "work_id": "whisper" // 注意: エラー応答は work_id として "asr" または "whisper" を使用する場合があります
}
```

## inference (推論) (リンクされた入力または直接オーディオストリーム経由で暗黙的に実行)

他のいくつかのユニットとは異なり、`llm-whisper` は主に、`setup` 時に `input` で指定されたリンクされたユニット (`sys.pcm`, `kws`, `vad`) または直接オーディオストリームを介してオーディオデータを受信します。セットアップ後にJSONコマンドを介してオーディオチャンクを直接送信するための個別の `inference` アクションはありません。オーディオデータは、設定された入力ソースから到着すると `sys_pcm_on_data` メソッドによって処理されます。

**入力データの処理フロー (`sys_pcm_on_data`):**
1.  **バッファリング**: オーディオデータ (生のPCM) は `pcmdata` (`buffer_t`) に蓄積されます。`delay_audio_frame_` が設定されている場合、`endpoint_flage_` が設定される (例: VADによる) まで、そのフレーム数を待ちます。
2.  **フォーマット変換**: 生の16ビットPCMサンプルは、-1.0から1.0の間の浮動小数点値に正規化されます。
3.  **メルスペクトログラム**: `librosa::Feature::melspectrogram` は、`whisper_sample_rate`、`whisper_n_fft`、`whisper_hop_length`、`whisper_n_mels` などのパラメータを使用して、浮動小数点サンプルからメルスペクトログラムを計算します。
4.  **スペクトログラムの対数化と正規化**: 値は対数スケールに変換され、クランプされ (最大値 - 8.0)、正規化されます: `(value + 4.0) / 4.0`。スペクトログラムは3000フレームにパディング/切り捨てられます。
5.  **エンコーダ**: 処理されたスペクトログラムはWhisperエンコーダモデル (`encoder_->Run()`) に入力されます。
6.  **言語検出とSOTシーケンス**: `detect_language()` を使用してターゲット言語ID (例: "zh" の場合) が決定されます。Start-Of-Transcriptionシーケンス `SOT_SEQUENCE` が設定されます (例: `[<SOT>, <lang_id>, <transcribe>, <no_timestamps>]`)。
7.  **デコーダ (メインパス)**: `decoder_main_` は `SOT_SEQUENCE` とエンコーダ出力を処理します。最初のトークンのロジットが取得されます。
8.  **トークン抑制**: `supress_tokens()` がロジットに適用されます (例: 特定の段階で `EOT`, `BLANK`, `NO_TIMESTAMPS`, `SOT`, `NO_SPEECH`, `TRANSLATE` トークンを防ぐため)。
9.  **Argmaxと最初のトークン**: 最も確率の高いトークンが選択されます (`argmax(logits)`)。
10. **デコーダ (ループ)**: 反復的に:
    *   最後に予測されたトークンが `decoder_loop_` にフィードバックされます。
    *   位置エンベディングとアテンションマスクが提供されます。
    *   `decoder_loop_->Run()` が次のトークンのロジットを予測します。
    *   トークンが抑制され、`argmax` が次のトークンを選択します。
    *   これは `EOT` が予測されるか、`whisper_n_text_ctx` に達するまで続きます。
11. **結果の組み立て**: 予測されたトークンIDが収集されます。
12. **トークンからテキストへ**: 各トークンIDは、そのテキスト表現にマッピングされます (`mode_config_.token_tables` から。多くの場合Base64デコードされます)。
13. **UTF-8検証**: `fix_utf8_string()` は、連結された文字列が有効なUTF-8であることを保証します。
14. **中国語T2S**: 言語が "en" または "ja" でなく、`mode_config_.t2s` が設定されている場合、OpenCCが文字列を簡体字中国語に変換します。
15. **出力コールバック**: 最終的な文字列は `out_callback_` を介して送信されます。
16. **スリープ**: `ensleep_` がtrueの場合 (通常はKWSとリンクされている場合)、`pause()` が呼び出されます。

**出力JSON (例: `asr.utf-8`):**
```json
{
  "created": 1737597600, // 文字起こしのタイムスタンプ
  "data": "こんにちは世界。",  // 文字起こしされたテキスト
  "error": {"code": 0, "message": ""},
  "object": "asr.utf-8",
  "request_id": "N/A", // 出力はイベント駆動型であるため、直接適用できない場合があります
  "work_id": "whisper.1002"
}
```
**出力JSON (例: `asr.utf-8.stream`):**
一連のメッセージ。
```json
// メッセージ1
{
  "created": 1737597601,
  "data": {
    "delta": "こんにちは ",
    "finish": false,
    "index": 0
  },
  "error": {"code": 0, "message": ""},
  "object": "asr.utf-8.stream",
  "request_id": "N/A",
  "work_id": "whisper.1002"
}
// 最終メッセージ
{
  "created": 1737597602,
  "data": {
    "delta": "世界。", // テキストの最後の部分。finishがtrueの場合、コードは "." を追加します。
    "finish": true,
    "index": 1 // またはチャンクが多い場合はそれ以上
  },
  "error": {"code": 0, "message": ""},
  "object": "asr.utf-8.stream",
  "request_id": "N/A",
  "work_id": "whisper.1002"
}
```

## link (リンク)

上位ユニット (KWSやVADなど) の出力をWhisperタスクの制御またはデータソースとしてリンクします。

JSONを送信:

```json
{
  "request_id": "3",
  "work_id": "whisper.1002",
  "action": "link",
  "object": "work_id",
  "data": "vad.1000" // リンクするユニットのwork_id (例: VADまたはKWS)
}
```

**動作:**
*   **`vad.xxxx` のリンク:**
    *   Whisperの `vad_endpoint` コールバックは、VADユニットからのメッセージによってトリガーされます。
    *   VADが `"true"` (発話終了) を送信すると、`endpoint_flage_` が `true` になり、Whisperにバッファリングされたオーディオを処理するよう通知します。
    *   VADが `"false"` (発話開始) を送信すると、`endpoint_flage_` が `false` になります。
*   **`kws.xxxx` のリンク:**
    *   Whisperの `kws_awake` コールバックがトリガーされます。
    *   Whisperタスクは、(セットアップからの `ensleep_ = true` により `pause` が呼び出されるため) 初期状態で一時停止している場合、起動されます。`kws_awake` は `awake_delay_` の後に内部的に `task_work` を呼び出します。
    *   `task_work` は `audio_url_` が設定されていればオーディオキャプチャを再度有効にします。
*   **`sys.pcm` (または該当する場合、他のオーディオソースユニット。例: `audio.xxxx`) のリンク:**
    *   セットアップ時にまだ行われていない場合 (例: `input` が `whisper.pcm.stream.base64` だった場合)、これによりオーディオデータストリームが確立されます。`audio_url_` (`data` を引数として `audio` ユニットの `cap` 関数を呼び出すことで取得) がサブスクライブされます。
    *   `audio_flage_` が `true` に設定されます。
    *   タスクの `inputs_` リストが更新されます。

**応答JSON (成功時):** (元のドキュメントと同じ)
```json
{
  "created": 1737597688,
  "data": "None",
  "error": {"code": 0, "message": ""},
  "object": "None",
  "request_id": "3",
  "work_id": "whisper.1002"
}
```
**応答JSON (エラー時):**
リンクに失敗した場合 (例: ターゲットユニットが存在しない、またはサブスクリプションに失敗):
```json
{
  "created": ...,
  "data": "None",
  "error": {"code": -20, "message": "link false"},
  "object": "None",
  "request_id": "3",
  "work_id": "whisper.1002"
}
```

> **注意:** リンクは、`input` 配列を介して `setup` 時に指定することもできます。ソースユニット (KWS, VAD) が設定され、必要に応じて相互にリンクされていることを確認してください。

## unlink (アンリンク)

以前に接続されたユニットをアンリンクします。

JSONを送信: (元のドキュメントと同じ)
```json
{
  "request_id": "4",
  "work_id": "whisper.1002",
  "action": "unlink",
  "object": "work_id",
  "data": "vad.1000"
}
```
**動作:**
*   `stop_subscriber_work_id()` を使用して、指定された `data` (アンリンクされるユニットのwork_id) へのサブスクリプションを停止します。
*   Whisperタスクの `inputs_` リストから `data` を削除します。
*   `superior_id_` が `work_id` と一致した場合、`superior_flage_` はfalseに設定されます (superior_id/flageの目的はコンテキストからは完全には明らかではありません)。

応答JSON (成功時): (元のドキュメントと同じ)
```json
{
  "created": 1737598243,
  "data": "None",
  "error": {"code": 0, "message": ""},
  "object": "None",
  "request_id": "4",
  "work_id": "whisper.1002"
}
```

## pause (一時停止)

Whisperモジュールのオーディオ処理を一時停止します。`sys.pcm` (`audio` ユニット経由) でオーディオをキャプチャしている場合 (つまり `audio_flage_` がtrueの場合)、オーディオストリームへのサブスクリプションを停止します。

JSONを送信: (元のドキュメントと同じ)
```json
{
  "request_id": "5",
  "work_id": "whisper.1002",
  "action": "pause"
}
```
**動作:**
*   `task_pause` 関数がエンキューされ、実行されます。
*   `audio_flage_` がtrueの場合 (つまり、`audio_url_` を介して `sys.pcm` のようなオーディオストリームをサブスクライブしている場合)、`audio_url_` へのサブスクリプションを停止します。`audio_flage_` は `false` になります。

応答JSON (成功時): (元のドキュメントと同じ)
```json
{
  "created": 1737598297,
  "data": "None",
  "error": {"code": 0, "message": ""},
  "object": "None",
  "request_id": "5",
  "work_id": "whisper.1002"
}
```

## work (再開)

一時停止中のWhisperモジュールを再開します。以前に `sys.pcm` にリンクされていた場合 (かつ `audio_url_` が既知の場合)、オーディオストリームに再サブスクライブします。

JSONを送信: (元のドキュメントと同じ)
```json
{
  "request_id": "6",
  "work_id": "whisper.1002",
  "action": "work"
}
```
**動作:**
*   `task_work` 関数が呼び出されます。
*   `audio_url_` が設定されていて `audio_flage_` が `false` の場合、オーディオストリームに再サブスクライブし、`audio_flage_` は `true` になります。
*   これにより、効果的にオーディオキャプチャと処理が再度有効になります。また、タスクオブジェクトの `kws_awake()` を呼び出し、その内部の `awake_flage_` をtrueに設定します。

応答JSON (成功時): (元のドキュメントと同じ)
```json
{
  "created": 1737598333,
  "data": "None",
  "error": {"code": 0, "message": ""},
  "object": "None",
  "request_id": "6",
  "work_id": "whisper.1002"
}
```

## exit (終了)

Whisperモジュールタスクを終了し、ロードされたモデルや、最後のWhisperタスクである場合はAXエンジンインスタンスを含む関連リソースを解放します。

JSONを送信: (元のドキュメントと同じ)
```json
{
  "request_id": "7",
  "work_id": "whisper.1002",
  "action": "exit"
}
```
**動作:**
*   タスクを停止します (`llm_task_obj->stop()`。ただし、提供されたコードではこのメソッドは空です)。
*   チャネルを介したアクティブなサブスクリプションを停止します (`llm_channel->stop_subscriber("")`)。
*   `audio_flage_` がtrueの場合、`audio` ユニットの `cap_stop` 関数を呼び出します (`unit_call("audio", "cap_stop", "None")`)。
*   `llm_task_` マップからタスクを削除します。
*   `llm_task` デストラクタはモデルリソース (`encoder_->Release()` など) を解放し、`_ax_deinit()` を呼び出します。
*   `_ax_deinit()` は `ax_init_flage_` をデクリメントし、それがゼロに達するとAXシステムとエンジンを非初期化します (`AX_ENGINE_Deinit`, `AX_SYS_Deinit`)。

応答JSON (成功時): (元のドキュメントと同じ)
```json
{
  "created": 1737598447,
  "data": "None",
  "error": {"code": 0, "message": ""},
  "object": "None",
  "request_id": "7",
  "work_id": "whisper.1002"
}
```

## taskinfo (タスク情報)

実行中のWhisperタスクに関する情報を取得します。

**リクエストJSON (すべての `whisper` タスクのリストを取得):** (元のドキュメントと同じ)
```json
{
  "request_id": "2",
  "work_id": "whisper",
  "action": "taskinfo"
}
```
応答JSON (タスクのリスト): (元のドキュメントと同じ)
```json
{
  "created": 1737598371,
  "data": ["whisper.1002"],
  "error": {"code": 0, "message": ""},
  "object": "whisper.tasklist",
  "request_id": "2",
  "work_id": "whisper"
}
```

**リクエストJSON (特定のタスクパラメータを取得):** (元のドキュメントと同じ)
```json
{
  "request_id": "2",
  "work_id": "whisper.1002",
  "action": "taskinfo"
}
```
応答JSON (特定のタスクパラメータ): (`language` およびその他のセットアップパラメータを含む)
```json
{
  "created": 1737598415,
  "data": {
    "model": "whisper-tiny",
    "response_format": "asr.utf-8",
    "enoutput": true,
    "inputs": ["sys.pcm", "kws.1000", "vad.1001"], // 現在の入力ソース/リンクを反映
    "language": "en" // セットアップ時に指定された言語
    // セットアップからの他のパラメータもここに含まれる場合があります。
  },
  "error": {"code": 0, "message": ""},
  "object": "whisper.taskinfo",
  "request_id": "2",
  "work_id": "whisper.1002"
}
```

> **注意:** `work_id` はユニットの初期化順序に従って増加し、固定インデックス値ではありません。Whisperの `task_count_` は通常1に制限されます。

## エラーコード (共通)
*   `0`: 成功。
*   `-2`: リクエストのJSON形式エラー。
*   `-3`, `-4`, `-5`, `-6` (セットアップ中): モデルのロード失敗 (位置エンベディング、エンコーダ、デコーダメイン、デコーダループなどの特定のコンポーネント、またはファイルパスなどの一般的な設定の問題)。
*   `-11`: モデル実行失敗 (一般的な推論エラー)。
*   `-20`: リンク操作失敗。
*   `-21`: タスクがいっぱいです (最大タスク数、通常Whisperでは1、に達しました)。
*   `-23`: Base64デコードエラー (オーディオストリーム入力用)。
*   `-25`: ストリームデータインデックスエラー (オーディオストリーム入力用。例: `decode_stream` の失敗)。
