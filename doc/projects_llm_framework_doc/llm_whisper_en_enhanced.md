# llm-whisper

The `llm-whisper` module is a sophisticated speech-to-text component. It leverages Whisper models to provide transcription services in multiple languages. It can be linked with Voice Activity Detection (VAD) and Keyword Spotting (KWS) units for more complex voice interaction scenarios.

## setup

Configures and initializes a Whisper ASR task.

Send JSON:

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
    // "t2s_model_path": "models/whisper-tiny/t2s.json", // Path is typically part of model's JSON config
    "whisper_sample_rate": 16000, // Further model-specific params can be set if needed
    "delay_audio_frame": 1000 // Number of audio frames to buffer before processing (if not using VAD/KWS)
  }
}
```

**Parameters in `data`:**

*   `request_id` (string, required): Reference for basic data explanation.
*   `work_id` (string, required): For initial configuration, use `whisper`. A specific `work_id` (e.g., `whisper.1002`) will be returned upon successful setup.
*   `action` (string, required): The method called, must be `setup`.
*   `object` (string, required): The data type being transferred, typically `whisper.setup`.
*   `model` (string, required): The Whisper model to be used (e.g., `whisper-tiny`, `whisper-base`, `whisper-small`). The system loads model configuration files (e.g., `mode_whisper-tiny.json`) based on this name from pre-defined paths (e.g., `projects/llm_framework/main_whisper/`).
*   `response_format` (string, required): Defines the format of the output.
    *   `asr.utf-8`: Non-streaming, single UTF-8 encoded text response.
    *   `asr.utf-8.stream`: Streaming, multiple UTF-8 encoded text chunks.
*   `input` (string or array of strings, required): Specifies the input sources and formats.
    *   `sys.pcm`: Raw PCM audio data from the system's audio input. The unit will call the `audio` unit (typically via `unit_call("audio", "cap", input)`) to capture data.
    *   `whisper.pcm.stream.base64`: Base64 encoded PCM audio stream.
    *   `whisper.wav.stream.base64`: Base64 encoded WAV audio stream (header will be stripped).
    *   `whisper.mp3.stream.base64`: Base64 encoded MP3 audio stream (decoding support might be limited or a placeholder).
    *   `kws.xxxx`: Links to a Keyword Spotting unit (e.g., `kws.1000`). If linked, Whisper task might start paused (`ensleep_ = true`) and activate upon KWS event.
    *   `vad.xxxx`: Links to a Voice Activity Detection unit (e.g., `vad.1001`). VAD signals control `endpoint_flage_` to manage audio processing segments.
*   `language` (string, required): The language for the model to recognize (e.g., "en", "zh", "ja"). A list of supported language codes (e.g., "sv", "sr", "no", corresponding to `WHISPER_LANG_NAMES` in source) and their corresponding integer IDs (`WHISPER_LANG_CODES`) is maintained internally. The `detect_language` function maps the string to the appropriate code.
*   `enoutput` (boolean, required): If `true`, the unit will send its transcription results. If `false`, output is suppressed.
*   `awake_delay` (integer, optional): Delay in milliseconds after a KWS awake event before starting audio processing. Defaults to a value from the model's config file (e.g., 50ms) or the value in this field.
*   `delay_audio_frame` (integer, optional): For `sys.pcm` or direct audio stream inputs without VAD/KWS, this specifies the number of audio frames to buffer before starting transcription. If `whisper.pcm.stream.base64` (or similar direct stream where `input.find("stream.base64") != std::string::npos`) is used, this is often set to 0. Default is 1000.

**Model Configuration (Loaded from JSON file based on `model` name, e.g., `projects/llm_framework/main_whisper/mode_whisper-tiny.json`):**
These parameters are primarily defined in the model's JSON configuration file.
*   `encoder` (string): Path to the encoder model file (e.g., `encoder.axmodel`).
*   `decoder_main` (string): Path to the main decoder model file (e.g., `decoder_main.axmodel`).
*   `decoder_loop` (string): Path to the loop decoder model file (e.g., `decoder_loop.axmodel`).
*   `positional_embedding` (string): Path to the positional embedding file (e.g., `positional_embedding.bin`).
*   `tokens` (string): Path to the tokens file (e.g., `tokens_zh.txt`). This file contains token strings, often base64 encoded, one per line with an ID.
*   `model_type` (string): Type of Whisper model (e.g., "tiny", "base", "small"). This determines network architecture parameters like `WHISPER_N_TEXT_STATE` (e.g., tiny:384, base:512, small:768).
*   `t2s` (string, optional): Path to the OpenCC configuration file for Traditional Chinese to Simplified Chinese conversion (e.g., `s2t.json`). Applied if `language_` (derived from setup `language`) is not "en" or "ja".
*   `whisper_sample_rate` (integer): Expected sample rate of the audio (e.g., 16000 Hz).
*   `whisper_n_fft` (integer): FFT window size (e.g., 400).
*   `whisper_hop_length` (integer): Hop length for STFT (e.g., 160).
*   `whisper_chunk_size` (integer): Audio chunk size in seconds for processing (e.g., 30s).
*   `whisper_n_mels` (integer): Number of Mel bands to generate (e.g., 80, but source shows 400 in one instance, likely configurable per model).
*   `whisper_sot` (integer): Start Of Transcription token ID (e.g., 50258).
*   `whisper_eot` (integer): End Of Transcription token ID (e.g., 50257).
*   `whisper_blank` (integer): Blank token ID (e.g., 220).
*   `whisper_no_timestamps` (integer): Token ID to suppress timestamps (e.g., 50363).
*   `whisper_no_speech` (integer): Token ID indicating no speech (e.g., 50362).
*   `whisper_translate` (integer): Token ID for translation task (e.g., 50358).
*   `whisper_transcribe` (integer): Token ID for transcription task (e.g., 50359).
*   `whisper_vocab_size` (integer): Size of the vocabulary (e.g., 51865).
*   `whisper_n_text_ctx` (integer): Context size for the text decoder (e.g., 448).
*   `awake_delay` (integer, optional): Default awake delay if not specified in the setup `data`.

**Response JSON (Success):**

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
  "work_id": "whisper.1002" // Unique work_id for this ASR task
}
```
**Response JSON (Error):**
If `task_count_` (max 1 for Whisper) limit is reached:
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
If model loading fails (e.g., file not found, AX engine init error):
```json
{
  "created": ...,
  "data": "None",
  "error": {
    "code": -5, // Or other codes like -2 (JSON error), -3 (pos_embed), -4 (encoder), -6 (decoder/config)
    "message": "Model loading failed." // Or more specific message like "config false"
  },
  "object": "None",
  "request_id": "2",
  "work_id": "whisper" // Note: error responses might use "asr" or "whisper" as work_id
}
```

## inference (Implicit via Linked Inputs or Direct Audio Stream)

Unlike some other units, `llm-whisper` primarily receives audio data through linked units (`sys.pcm`, `kws`, `vad`) or direct audio streams specified in `input` during `setup`. There isn't a separate `inference` action to send audio chunks directly via a JSON command after setup. Audio data is processed as it arrives from the configured input sources by the `sys_pcm_on_data` method.

**Input Data Handling & Processing (`sys_pcm_on_data`):**
1.  **Buffering**: Audio data (raw PCM) is accumulated in `pcmdata` (a `buffer_t`). If `delay_audio_frame_` is set, it waits for that many frames unless `endpoint_flage_` is set (e.g., by VAD).
2.  **Format Conversion**: Raw 16-bit PCM samples are normalized to floating-point values between -1.0 and 1.0.
3.  **Mel Spectrogram**: `librosa::Feature::melspectrogram` calculates the Mel spectrogram from the float samples using parameters like `whisper_sample_rate`, `whisper_n_fft`, `whisper_hop_length`, `whisper_n_mels`.
4.  **Log & Normalize Spectrogram**: Values are log-scaled, clamped (max value - 8.0), and normalized: `(value + 4.0) / 4.0`. The spectrogram is padded/truncated to 3000 frames.
5.  **Encoder**: The processed spectrogram is fed into the Whisper encoder model (`encoder_->Run()`).
6.  **Language Detection & SOT Sequence**: The target language ID (e.g., for "zh") is determined using `detect_language()`. The Start-Of-Transcription sequence `SOT_SEQUENCE` is set (e.g., `[<SOT>, <lang_id>, <transcribe>, <no_timestamps>]`).
7.  **Decoder (Main Pass)**: `decoder_main_` processes the `SOT_SEQUENCE` and encoder outputs. Logits for the first token are obtained.
8.  **Token Suppression**: `supress_tokens()` is applied to logits (e.g., to prevent `EOT`, `BLANK`, `NO_TIMESTAMPS`, `SOT`, `NO_SPEECH`, `TRANSLATE` tokens at certain stages).
9.  **Argmax & First Token**: The token with the highest probability is chosen (`argmax(logits)`).
10. **Decoder (Loop)**: Iteratively:
    *   The last predicted token is fed back to `decoder_loop_`.
    *   Positional embeddings and attention masks are provided.
    *   `decoder_loop_->Run()` predicts the next token's logits.
    *   Tokens are suppressed, and `argmax` selects the next token.
    *   This continues until `EOT` is predicted or `whisper_n_text_ctx` is reached.
11. **Result Assembly**: Predicted token IDs are collected.
12. **Token to Text**: Each token ID is mapped to its text representation (from `mode_config_.token_tables`, which are often base64 decoded).
13. **UTF-8 Validation**: `fix_utf8_string()` ensures the concatenated string is valid UTF-8.
14. **Chinese T2S**: If language is not "en" or "ja", and `mode_config_.t2s` is set, OpenCC converts the string to Simplified Chinese.
15. **Output Callback**: The final string is sent via `out_callback_`.
16. **Sleep**: If `ensleep_` is true (typically when linked with KWS), `pause()` is called.

**Output JSON (Example for `asr.utf-8`):**
```json
{
  "created": 1737597600, // Timestamp of transcription
  "data": "Hello world.",  // Transcribed text
  "error": {"code": 0, "message": ""},
  "object": "asr.utf-8",
  "request_id": "N/A", // May not be directly applicable as output is event-driven
  "work_id": "whisper.1002"
}
```
**Output JSON (Example for `asr.utf-8.stream`):**
A series of messages.
```json
// Message 1
{
  "created": 1737597601,
  "data": {
    "delta": "Hello ",
    "finish": false,
    "index": 0
  },
  "error": {"code": 0, "message": ""},
  "object": "asr.utf-8.stream",
  "request_id": "N/A",
  "work_id": "whisper.1002"
}
// Final Message
{
  "created": 1737597602,
  "data": {
    "delta": "world.", // Final part of text. The code appends "." if finish is true.
    "finish": true,
    "index": 1 // Or higher if more chunks
  },
  "error": {"code": 0, "message": ""},
  "object": "asr.utf-8.stream",
  "request_id": "N/A",
  "work_id": "whisper.1002"
}
```

## link

Links the output of an upper-level unit (like `kws` or `vad`) as a control or data source for the Whisper task.

Send JSON:

```json
{
  "request_id": "3",
  "work_id": "whisper.1002",
  "action": "link",
  "object": "work_id",
  "data": "vad.1000" // work_id of the unit to link (e.g., VAD or KWS)
}
```

**Behavior:**
*   **Linking `vad.xxxx`:**
    *   The `vad_endpoint` callback in Whisper is triggered by messages from the VAD unit.
    *   If VAD sends `"true"` (speech ended), `endpoint_flage_` becomes `true`, signaling Whisper to process buffered audio.
    *   If VAD sends `"false"` (speech started), `endpoint_flage_` becomes `false`.
*   **Linking `kws.xxxx`:**
    *   The `kws_awake` callback in Whisper is triggered.
    *   The Whisper task, if initially paused (due to `ensleep_ = true` from setup, which sets `pause` to be called), will be woken up. `task_work` is called internally by `kws_awake` after `awake_delay_`.
    *   `task_work` re-enables audio capture if `audio_url_` is configured.
*   **Linking `sys.pcm` (or other audio source unit if applicable, e.g., `audio.xxxx`):**
    *   If not already done during setup (e.g. `input` was `whisper.pcm.stream.base64`), this establishes the audio data stream. `audio_url_` (obtained by calling the `audio` unit's `cap` function with `data` as argument) is subscribed to.
    *   `audio_flage_` is set to `true`.
    *   The task's `inputs_` list is updated.

**Response JSON (Success):** (Same as original doc)
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
**Response JSON (Error):**
If linking fails (e.g., target unit does not exist or subscription fails):
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

> **Note:** Linking can also be specified during `setup` via the `input` array. Ensure that the source units (KWS, VAD) are configured and, if necessary, linked to each other.

## unlink

Unlinks a previously connected unit.

Send JSON: (Same as original doc)
```json
{
  "request_id": "4",
  "work_id": "whisper.1002",
  "action": "unlink",
  "object": "work_id",
  "data": "vad.1000"
}
```
**Behavior:**
*   Stops the subscription to the specified `data` (work_id of the unit being unlinked) using `stop_subscriber_work_id()`.
*   Removes the `data` from the `inputs_` list of the Whisper task.
*   If `superior_id_` matched the `work_id`, `superior_flage_` is set to false (purpose of superior_id/flage is not fully clear from context).

Response JSON (Success): (Same as original doc)
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

## pause

Pauses the Whisper module's audio processing. If it's capturing audio via `sys.pcm` (i.e., `audio_flage_` is true), it stops the audio subscription.

Send JSON: (Same as original doc)
```json
{
  "request_id": "5",
  "work_id": "whisper.1002",
  "action": "pause"
}
```
**Behavior:**
*   The `task_pause` function is enqueued and executed.
*   If `audio_flage_` is true (meaning it's subscribed to an audio stream like `sys.pcm` via `audio_url_`), it stops the subscription to `audio_url_`. `audio_flage_` becomes `false`.

Response JSON (Success): (Same as original doc)
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

## work

Resumes a paused Whisper module. If it was previously linked to `sys.pcm` (and `audio_url_` is known), it re-subscribes to the audio stream.

Send JSON: (Same as original doc)
```json
{
  "request_id": "6",
  "work_id": "whisper.1002",
  "action": "work"
}
```
**Behavior:**
*   The `task_work` function is called.
*   If `audio_url_` is set and `audio_flage_` is `false`, it re-subscribes to the audio stream, and `audio_flage_` becomes `true`.
*   Effectively, this re-enables audio capture and processing. It also calls `kws_awake()` on the task object, setting its internal `awake_flage_` to true.


Response JSON (Success): (Same as original doc)
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

## exit

Exits the Whisper module task and releases associated resources, including loaded models and AX engine instances if it's the last Whisper task.

Send JSON: (Same as original doc)
```json
{
  "request_id": "7",
  "work_id": "whisper.1002",
  "action": "exit"
}
```
**Behavior:**
*   Stops the task (`llm_task_obj->stop()`, though this method is empty in the provided code).
*   Stops any active subscriptions via the channel (`llm_channel->stop_subscriber("")`).
*   If `audio_flage_` is true, it calls the `audio` unit's `cap_stop` function (`unit_call("audio", "cap_stop", "None")`).
*   Removes the task from `llm_task_` map.
*   The `llm_task` destructor releases model resources (`encoder_->Release()`, etc.) and calls `_ax_deinit()`.
*   `_ax_deinit()` decrements `ax_init_flage_` and if it reaches zero, deinitializes the AX system and engine (`AX_ENGINE_Deinit`, `AX_SYS_Deinit`).

Response JSON (Success): (Same as original doc)
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

## taskinfo

Gets information about running Whisper tasks.

**Request JSON (Get list of all `whisper` tasks):** (Same as original doc)
```json
{
  "request_id": "2",
  "work_id": "whisper",
  "action": "taskinfo"
}
```
Response JSON (List of tasks): (Same as original doc)
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

**Request JSON (Get specific task parameters):** (Same as original doc)
```json
{
  "request_id": "2",
  "work_id": "whisper.1002",
  "action": "taskinfo"
}
```
Response JSON (Specific task parameters): (Includes `language` and other setup parameters)
```json
{
  "created": 1737598415,
  "data": {
    "model": "whisper-tiny",
    "response_format": "asr.utf-8",
    "enoutput": true,
    "inputs": ["sys.pcm", "kws.1000", "vad.1001"], // Reflects current input sources/links
    "language": "en" // The language specified during setup
    // Other parameters from setup might be included here.
  },
  "error": {"code": 0, "message": ""},
  "object": "whisper.taskinfo",
  "request_id": "2",
  "work_id": "whisper.1002"
}
```

> **Note:** `work_id` increases according to the initialization order of units and is not a fixed index value. The `task_count_` for Whisper is typically limited to 1.

## Error Codes (Common)
*   `0`: Success.
*   `-2`: JSON format error in request.
*   `-3`, `-4`, `-5`, `-6` (during setup): Model loading failed (specific component like positional embeddings, encoder, decoder_main, decoder_loop, or general config issue like file paths).
*   `-11`: Model run failed (generic inference error).
*   `-20`: Link operation failed.
*   `-21`: Task full (max task count, usually 1 for Whisper, reached).
*   `-23`: Base64 decoding error (for audio stream inputs).
*   `-25`: Stream data index error (for audio stream inputs, e.g., `decode_stream` failure).
