# llm-kws (Keyword Spotting)

The `llm-kws` unit provides keyword spotting (voice wake-up) services. It listens to an audio stream and triggers an event when predefined keywords are detected. It supports different models for various languages, such as Chinese and English.

## General API Call Conventions

API calls to `llm-kws` are made via JSON messages.

### Request Structure

A standard request to an `llm-kws` API follows this structure:

```json
{
  "request_id": "client_generated_uuid_or_counter",
  "work_id": "kws_or_kws.instance_id",
  "action": "api_action_name",
  "object": "optional_object_type_string", // If applicable
  "data": { /* API-specific payload */ }     // Or string for some APIs
}
```

-   `request_id` (string, mandatory): A unique client-generated identifier for correlating requests with responses.
-   `work_id` (string, mandatory):
    -   For initial setup or general queries: `"kws"`.
    -   For operations on a specific KWS task instance: `"kws.XXXX"` (e.g., `"kws.1000"`), where `XXXX` is the instance ID returned by a successful `setup` call.
-   `action` (string, mandatory): The API action to invoke (e.g., `"setup"`, `"pause"`).
-   `object` (string, optional): Specifies the type or context of the data being sent, if applicable (e.g., `"kws.setup"` for setup data).
-   `data` (object or string, optional): The payload for the API call. Structure varies by API.

### Response Structure

**Success Response:**

Indicates the API call was accepted and processed successfully.

```json
{
  "request_id": "mirrored_from_request",
  "work_id": "kws_or_kws.instance_id", // Mirrored or updated (e.g., "kws.1000" after setup)
  "action": "api_action_name",      // Mirrored from the request
  "created": 1678886400,            // Integer: Unix timestamp of response generation
  "code": 0,                        // Integer: 0 indicates success
  "message": "OK",                  // String: Success message
  "data": { /* API-specific data */ } // Optional: Payload returned by the API
}
```
-   If an API call results in the creation of a new KWS task instance (e.g., `setup`), the `work_id` in the response will be the new instance ID (e.g., `"kws.1000"`), often also included in the `data` object.

**Error Response:**

Indicates a failure during API call processing.

```json
{
  "request_id": "mirrored_from_request",
  "work_id": "kws_or_kws.instance_id",
  "action": "api_action_name",        // Mirrored from the request
  "created": 1678886400,              // Integer: Unix timestamp of response generation
  "error": {
    "code": -1,                       // Integer: Non-zero error code
    "message": "Error description"    // String: Detailed error message
  }
}
```

**Common Error Codes:**

-   `-2`: Invalid JSON format in the request `data`.
-   `-3`: KWS engine initialization failed (underlying Sherpa-ONNX error).
-   `-5`: Model loading failed (e.g., model files not found, invalid model configuration).
-   `-6`: Specified KWS task instance (e.g., `"kws.1000"`) does not exist.
-   `-11`: Generic error during keyword spotting or internal processing.
-   `-21`: Task limit reached (typically, only one KWS instance is supported per model type or overall).
-   `-23`: Error during keyword file generation (e.g., `llm-kws_text2token.py` script failure, invalid keyword format, or issues with `tokens.txt` or `bpe.model`).
-   `-25`: Failed to play wake-up audio (if `enwake_audio` is true), possibly due to an issue with the `llm_audio` unit or the specified `wake_wav_file`.

## Wake-up Event Notification

When a configured keyword is detected and `enoutput` is `true` (the default), the `llm-kws` unit pushes an event notification. The primary `response_format` for this unit is `"kws.bool"`, indicating a boolean event upon detection.

-   **Mechanism**: Asynchronous JSON message pushed to the client (or ZMQ subscribers connected to the unit's output).
-   **Trigger**: Detection of one of the configured keywords with sufficient confidence.

**Event JSON Structure (`response_format: "kws.bool"`):**

```json
{
  "request_id": "<original_setup_request_id>", // request_id from the setup call that created this instance
  "work_id": "kws.XXXX",                     // The specific KWS task instance ID
  "action": "kws_event",                     // Action indicating a keyword spotting event
  "object": "kws.bool",                      // The response format specified
  "created": 1678886410,                     // Integer: Unix timestamp of event generation
  "code": 0,
  "message": "Keyword Detected",             // Or similar informative message (actual message may vary)
  "data": true                               // Boolean: true indicates a keyword was detected
}
```
-   The `action` field for the event (e.g., `"kws_event"`) is a convention for clarity; the core information is in `object` and `data`. The actual message pushed by `main.cpp`'s `send_obj_true` might not contain an "action" field for the event itself but will have the `object` and `data`.

## API Reference

### **setup**

Initializes and configures a new Keyword Spotting (KWS) task instance.

-   **`work_id` in request**: `"kws"`
-   **`work_id` in response (success)**: `"kws.XXXX"` (e.g., `"kws.1000"`)

**Request Parameters:**

| Parameter         | Type           | Required | Default (from Model JSON if applicable) | Description                                                                                                                                                                                                                                                           |
|-------------------|----------------|----------|-----------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `object`          | string         | Yes      | N/A                                     | Must be `"kws.setup"`.                                                                                                                                                                                                                                                |
| `data.model`      | string         | Yes      | N/A                                     | Name of the KWS model configuration file (without `.json`). Example: `"sherpa-onnx-kws-zipformer-gigaspeech-3.3M-2024-01-01"`.                                                                                                                                      |
| `data.response_format`| string    | Yes      | `"kws.bool"`                            | Output format for keyword detection events. Currently, `"kws.bool"` is the primary supported format, indicating a boolean `true` event upon detection.                                                                                                                     |
| `data.input`      | string/array   | Yes      | N/A                                     | Audio input source(s). Typically `"sys.pcm"` for system microphone input, or an audio stream from another unit (e.g., an audio processing unit ID).                                                                                                                |
| `data.enoutput`   | boolean        | No       | `true`                                  | Enable (`true`) or disable (`false`) pushing of keyword detection events.                                                                                                                                                                                             |
| `data.kws`        | string / array | Yes      | (Uses default `keywords_file` from model JSON) | Keyword(s) to spot. Can be a single string or an array of strings. These are processed by `llm-kws_text2token.py` to generate a model-specific keyword file. <br>English example: `"HEY SIRI"`, `["HELLO WORLD", "OK GOOGLE"]` <br>Chinese example: `"你好小爱"`, `["小爱同学", "你好问问"]` |
| `data.enwake_audio`| boolean       | No       | `true` (from example, but `main.cpp` default seems `false` if not specified) | If `true`, plays a wake-up sound upon keyword detection. The sound file is specified by `wake_wav_file` in the model's JSON configuration. This uses the `llm_audio` unit's `play_raw` API internally.                                                              |

**Request Example (English Keywords):**
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

**Request Example (Chinese Keywords - assuming a Chinese model is specified):**
```json
{
  "request_id": "setup_kws_cn_002",
  "work_id": "kws",
  "action": "setup",
  "object": "kws.setup",
  "data": {
    "model": "sherpa-onnx-kws-zipformer-wenetspeech-3.3M-2024-01-01", // Example Chinese KWS model name
    "response_format": "kws.bool",
    "input": "sys.pcm",
    "kws": "你好小爱" 
  }
}
```

**Response (Success):**
```json
{
  "request_id": "setup_kws_en_001",
  "work_id": "kws.1000", // New KWS task instance ID
  "action": "setup",
  "created": 1678886400,
  "code": 0,
  "message": "OK",
  "data": {
    "work_id": "kws.1000" // Confirms the created instance ID
  }
}
```

**Response (Error - Model Load Fail):**
```json
{
  "request_id": "setup_kws_003",
  "work_id": "kws",
  "action": "setup",
  "created": 1678886401,
  "error": {
    "code": -5,
    "message": "Model loading failed."
  }
}
```

**Response (Error - Keyword Processing Fail):**
```json
{
  "request_id": "setup_kws_004",
  "work_id": "kws",
  "action": "setup",
  "created": 1678886402,
  "error": {
    "code": -23,
    "message": "Failed to process keywords or generate keyword file."
  }
}
```

### **pause**

Pauses the KWS task instance. While paused, it will not listen for or detect keywords.

-   **`work_id` in request**: `"kws.XXXX"` (specific KWS task instance ID)

**Request Parameters**: None beyond standard `request_id`, `work_id`, `action`.

**Request Example:**
```json
{
  "request_id": "pause_kws_005",
  "work_id": "kws.1000",
  "action": "pause"
}
```

**Response (Success):**
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

Resumes a paused KWS task instance.

-   **`work_id` in request**: `"kws.XXXX"`

**Request Parameters**: None beyond standard `request_id`, `work_id`, `action`.

**Request Example:**
```json
{
  "request_id": "work_kws_006",
  "work_id": "kws.1000",
  "action": "work"
}
```

**Response (Success):**
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

Stops and removes a KWS task instance, freeing its resources.

-   **`work_id` in request**: `"kws.XXXX"`

**Request Parameters**: None beyond standard `request_id`, `work_id`, `action`.

**Request Example:**
```json
{
  "request_id": "exit_kws_007",
  "work_id": "kws.1000",
  "action": "exit"
}
```

**Response (Success):**
```json
{
  "request_id": "exit_kws_007",
  "work_id": "kws.1000", // work_id of the exited task
  "action": "exit",
  "created": 1678886405,
  "code": 0,
  "message": "OK"
}
```

### **taskinfo**

Retrieves information about active KWS tasks. Behavior depends on the `work_id` in the request.

**Case 1: General KWS Unit Information**

-   **`work_id` in request**: `"kws"`
-   **Description**: Returns a list of all active KWS task instance IDs.

**Request Example:**
```json
{
  "request_id": "taskinfo_kws_general_008",
  "work_id": "kws",
  "action": "taskinfo"
}
```

**Response (Success - One Task Active):**
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
    "kws.1000" // List of active instance IDs
  ]
}
```

**Case 2: Specific KWS Task Instance Information**

-   **`work_id` in request**: `"kws.XXXX"` (e.g., `"kws.1000"`)
-   **Description**: Returns runtime parameters and configuration of the specified KWS task instance.

**Request Example:**
```json
{
  "request_id": "taskinfo_kws_specific_009",
  "work_id": "kws.1000",
  "action": "taskinfo"
}
```

**Response (Success):**
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
    "inputs": [ "sys.pcm" ], // Array of linked input sources
    "kws": "HELLO M5,HEY STACK", // Effective keywords being used (from the generated file or default)
    "enwake_audio": true
  }
}
```

> **Note on `work_id` for task instances**: The numeric part of a task instance `work_id` (e.g., `1000` in `kws.1000`) is dynamically assigned by the system upon creation via the `setup` API.

## Relation to Model Configuration Files and Scripts

The `llm-kws` unit relies on model-specific JSON configuration files and a Python script for keyword processing.

### Model Configuration JSON (e.g., `mode_sherpa-onnx-kws-zipformer-gigaspeech-3.3M-2024-01-01.json`)

-   **`mode`**: Name of the model configuration.
-   **`type`**: Unit type, e.g., `"kws"`.
-   **`capabilities`**: Describes function, e.g., `"Keyword_spotting"`, `"English"`.
-   **`mode_param` Object**: Contains crucial parameters for the Sherpa-ONNX KWS engine:
    -   `model_config.transducer.*`, `model_config.tokens`: Paths to the neural network model components (encoder, decoder, joiner) and token list. These paths are relative to the model's directory (e.g., `sherpa-onnx-kws-zipformer-gigaspeech-3.3M-2024-01-01/`).
    -   `keywords_file`: Default keyword list file (e.g., `"keywords.txt"`) used if `kws` is not provided in the `setup` API. This file is also the output target for the `llm-kws_text2token.py` script.
    -   `feat_config.sample_rate`, `feat_config.feature_dim`: Audio feature configuration.
    -   `model_config.num_threads`: Number of threads for computation.
    -   `keywords_score`, `keywords_threshold`: Tuning parameters for keyword detection sensitivity.
    -   `text2token-bpe-model`, `text2token-tokens-type`: Parameters passed to the `llm-kws_text2token.py` script for processing keywords, defining the tokenization strategy (e.g., `cjkchar+bpe`, `ppinyin`).
    -   `wake_wav_file`: Path to the audio file played when `enwake_audio` is `true` and a keyword is detected (e.g., `"/opt/m5stack/data/audio/wakeup_en_us.wav"`).

### Keyword Processing Script (`llm-kws_text2token.py`)

-   **Purpose**: This Python script is crucial for converting human-readable keywords provided in the `setup` API's `kws` parameter (or a text file) into a format (token IDs) that the Sherpa-ONNX KWS engine can understand.
-   **Operation**:
    1.  When the `setup` API is called with the `kws` parameter, the `llm-kws` unit saves these keywords to a temporary file (e.g., `/tmp/kws_awake.txt.tmp`).
    2.  It then invokes `llm-kws_text2token.py` using paths and parameters defined in the model's JSON configuration (e.g., Python interpreter path, script path, paths to `tokens.txt`, `bpe.model` if used, and the output path for the processed `keywords_file`).
    3.  The script reads the temporary keyword file, processes the keywords based on the model's tokenization type (`text2token-tokens-type`), and writes the output to the `keywords_file` specified in the model's JSON (e.g., `sherpa-onnx-kws-zipformer-gigaspeech-3.3M-2024-01-01/keywords.txt`).
    4.  The KWS engine then loads this generated `keywords_file`.
-   **Example Invocation (derived from `text2token.txt` and `main.cpp` logic)**:
    ```bash
    /usr/bin/python3 /opt/m5stack/scripts/llm-kws_text2token.py \
      --text /tmp/kws_awake.txt.tmp \
      --tokens <path_to_model_specific_tokens.txt> \
      --tokens-type <type_from_model_json> \
      --bpe-model <path_to_model_specific_bpe.model> \ # If BPE is used
      --output <path_to_model_specific_keywords.txt>
    ```
    This shows how keywords are processed for a specific model using its token list and potentially a BPE model. The exact paths are constructed dynamically based on the loaded model's configuration.

Understanding these relationships is key to configuring custom keywords and troubleshooting KWS behavior. The `llm-kws` unit abstracts much of this complexity, but the underlying files define the capabilities and default behaviors.