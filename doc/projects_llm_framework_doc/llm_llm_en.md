# llm-llm (Large Language Model)

The `llm-llm` unit provides large language model inference services, enabling text generation in response to prompts. It can be configured to use various models and supports different input sources, including direct API calls, ASR unit transcriptions, and KWS unit triggers. Output can be streamed or delivered as a complete text.

## General API Call Conventions

API calls to `llm-llm` are made via JSON messages.

### Request Structure

A standard request to an `llm-llm` API follows this structure:

```json
{
  "request_id": "client_generated_uuid_or_counter",
  "work_id": "llm_or_llm.instance_id",
  "action": "api_action_name",
  "object": "optional_object_type_string", // If applicable
  "data": { /* API-specific payload */ }     // Or string/object for some APIs
}
```

-   `request_id` (string, mandatory): A unique client-generated identifier for correlating requests with responses.
-   `work_id` (string, mandatory):
    -   For initial setup or general queries: `"llm"`.
    -   For operations on a specific LLM task instance: `"llm.XXXX"` (e.g., `"llm.1002"`), where `XXXX` is the instance ID returned by a successful `setup` call.
-   `action` (string, mandatory): The API action to invoke (e.g., `"setup"`, `"inference"`).
-   `object` (string, optional): Specifies the type or context of the data being sent (e.g., `"llm.setup"`, `"llm.utf-8"`).
-   `data` (object or string, optional): The payload for the API call.

### Response Structure

**Success Response (for most API calls):**

Indicates the API call was accepted and processed successfully.

```json
{
  "request_id": "mirrored_from_request",
  "work_id": "llm_or_llm.instance_id", // Mirrored or updated (e.g., "llm.1002" after setup)
  "action": "api_action_name",       // Mirrored from the request
  "created": 1678886400,             // Integer: Unix timestamp of response generation
  "code": 0,                         // Integer: 0 indicates success
  "message": "OK",                   // String: Success message
  "data": { /* API-specific data */ } // Optional: Payload returned by the API
}
```
-   For the `inference` API, this synchronous response only acknowledges the request; the actual LLM output is sent asynchronously (see Data Input and Output section).
-   If an API call results in the creation of a new task instance (e.g., `setup`), the `work_id` in the response and in `data.work_id` will be the new instance ID.

**Error Response:**

Indicates a failure during API call processing.

```json
{
  "request_id": "mirrored_from_request",
  "work_id": "llm_or_llm.instance_id",
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
-   `-3`: LLM engine initialization failed (`LLM::Init` internal error).
-   `-5`: Model loading failed (e.g., model files not found, tokenizer issues, invalid model configuration).
-   `-6`: Specified LLM task instance (e.g., `"llm.1002"`) does not exist.
-   `-11`: Generic error during model inference or internal processing.
-   `-20`: Error related to input source linking or data subscription.
-   `-21`: Task limit reached (e.g., `MAX_TASK_NUM` for LLM instances, currently 2).
-   `-23`: Tokenization error or invalid prompt structure.
-   `-25`: Inference error (e.g., an issue during the `lLaMa_->Run` call).

## Data Input and Output (Inference Results)

### Input Data (Prompts)

The `llm-llm` unit can receive prompts for inference in several ways:

1.  **Via `inference` API to a specific `llm.XXXX` instance:**
    *   **Non-streaming prompt (`object: "llm.utf-8"`)**:
        -   The `data` field is a single string containing the full prompt.
    *   **Streaming prompt (`object: "llm.utf-8.stream"`)**:
        -   The `data` field is a JSON object: `{"index": <integer>, "delta": "<prompt_chunk>", "finish": <boolean>}`.
        -   `index`: Sequence number of the chunk, starting from 0.
        -   `delta`: A segment of the prompt.
        -   `finish`: `true` if this is the final chunk of the prompt, `false` otherwise. This allows for sending long prompts in segments.

2.  **Via Linked Units (configured in `setup` API's `input` parameter):**
    *   **`"llm.utf-8"` or `"llm.utf-8.stream"`**: The instance listens for prompts sent to its own `work_id` using the `inference` API as described above.
    *   **`"asr.XXXX"` (e.g., `"asr.1001"`)**: The LLM instance subscribes to the output of the specified ASR (Automatic Speech Recognition) unit. The transcribed text from ASR (either the full text or the `delta` from a stream when `finish` is true) is used as the prompt for the LLM.
    *   **`"kws.XXXX"` (e.g., `"kws.1000"`)**: The LLM instance subscribes to events from the specified KWS (Keyword Spotting) unit. When a keyword is detected, the KWS unit typically sends a boolean or the keyword itself. The `llm-llm` unit's `kws_awake` function is triggered, which calls `lLaMa_->Stop()`. This is used to interrupt any ongoing LLM generation, effectively acting as a stop signal or a trigger for a new, potentially predefined, interaction flow (not for directly prompting with the KWS data).

### Output Data (Generated Text)

If `enoutput` is `true` (default) in the `setup` call, the `llm-llm` unit pushes the generated text asynchronously after an `inference` request is accepted. The format is determined by `response_format` in the `setup` API:

1.  **Streaming Output (`response_format: "llm.utf-8.stream"`)**
    -   The unit sends a stream of JSON objects, each representing a chunk of the generated text.
    -   **JSON Structure per Chunk:**
        ```json
        {
          "request_id": "<original_inference_request_id>",
          "work_id": "llm.XXXX",                     // The specific LLM task instance ID
          "action": "llm_event",                     // Indicates an LLM-generated event/data
          "object": "llm.utf-8.stream",              // The response format
          "created": 1678886410,                     // Integer: Unix timestamp of this chunk's generation
          "code": 0,                                 // Integer: 0 for successful chunk
          "message": "OK",                           // String: "OK"
          "data": {
            "index": 0,                              // Integer: Sequence number of this chunk.
            "delta": "<generated_text_chunk>",       // String: A segment of the generated text.
            "finish": false                          // Boolean: `true` if this is the final chunk of the response, `false` otherwise.
          }
        }
        ```
    -   The `request_id` in these asynchronous messages will match the `request_id` of the `inference` call that initiated the generation.

2.  **Non-Streaming Output (`response_format: "llm.utf-8"`)**
    -   The unit sends a single JSON message containing the entire generated text once inference is complete.
    -   **JSON Structure:**
        ```json
        {
          "request_id": "<original_inference_request_id>",
          "work_id": "llm.XXXX",
          "action": "llm_event",
          "object": "llm.utf-8",
          "created": 1678886415,
          "code": 0,
          "message": "OK",
          "data": "<full_generated_text_string>" // String: The complete generated text.
        }
        ```

## API Reference

### **setup**

Initializes and configures a new Large Language Model (LLM) task instance.

-   **`work_id` in request**: `"llm"`
-   **`work_id` in response (success)**: `"llm.XXXX"` (e.g., `"llm.1002"`)

**Request Parameters:**

| Parameter              | Type          | Required | Default (from Model JSON if applicable) | Description                                                                                                                                                                                                                                                              |
|------------------------|---------------|----------|-----------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `object`               | string        | Yes      | N/A                                     | Must be `"llm.setup"`.                                                                                                                                                                                                                                                   |
| `data.model`           | string        | Yes      | N/A                                     | Name of the LLM configuration file (without `.json`). Examples: `"qwen2.5-0.5B-prefill-20e"`, `"qwen1.5-0.5B-Chat"`. Models are located in `projects/llm_framework/main_llm/models/`.                                                                                  |
| `data.response_format` | string        | Yes      | N/A                                     | Output format for generated text if `enoutput` is true. Supported: `"llm.utf-8.stream"` (stream of text chunks), `"llm.utf-8"` (single text string).                                                                                                               |
| `data.input`           | string/array  | Yes      | N/A                                     | Input source(s) for prompts. Can be a string (e.g., `"llm.utf-8"`) or an array (e.g., `["llm.utf-8", "asr.1001", "kws.1000"]`). See "Input Data" section for details on how different sources are handled.                                                              |
| `data.enoutput`        | boolean       | No       | `true`                                  | Enable (`true`) or disable (`false`) asynchronous pushing of generated text results.                                                                                                                                                                                     |
| `data.max_token_len`   | integer       | No       | (from model JSON)                       | Maximum number of tokens to generate in the response. This overrides the `max_token_len` from the model's JSON if provided. Limited by the model's absolute maximum sequence length.                                                                                     |
| `data.prompt`          | string        | No       | (from model JSON, or empty)             | A system prompt or initial context to set for the LLM instance. This overrides the `system_prompt` from the model's JSON if provided.                                                                                                                                     |
| `data.mode_param`      | object        | No       | {}                                      | Allows overriding specific inference parameters defined in the model's JSON, such as `temperature`, `top_p`. **Note:** Based on current `main.cpp`, this is NOT a generic override mechanism for all `LLMAttrType` fields. Specific parsing for these is not implemented in `setup`'s `load_model`. Inference parameters are primarily controlled by the chosen model's JSON file. |

**Request Example:**
```json
{
  "request_id": "setup_llm_001",
  "work_id": "llm",
  "action": "setup",
  "object": "llm.setup",
  "data": {
    "model": "qwen2.5-0.5B-prefill-20e",
    "response_format": "llm.utf-8.stream",
    "input": ["llm.utf-8", "asr.1001"],
    "enoutput": true,
    "max_token_len": 512,
    "prompt": "You are a helpful AI assistant."
  }
}
```

**Response (Success):**
```json
{
  "request_id": "setup_llm_001",
  "work_id": "llm.1002", // New LLM task instance ID
  "action": "setup",
  "created": 1678886400,
  "code": 0,
  "message": "OK",
  "data": {
    "work_id": "llm.1002" // Confirms the created instance ID
  }
}
```

**Response (Error - Model Load Fail):**
```json
{
  "request_id": "setup_llm_002",
  "work_id": "llm",
  "action": "setup",
  "created": 1678886401,
  "error": {
    "code": -5,
    "message": "Model loading failed."
  }
}
```

### **inference**

Submits a prompt to a specific LLM task instance for text generation. The actual generated text is sent asynchronously.

-   **`work_id` in request**: `"llm.XXXX"` (specific LLM task instance ID)

**Request Parameters:**

| Parameter | Type          | Required | Default | Description                                                                                                                               |
|-----------|---------------|----------|---------|-------------------------------------------------------------------------------------------------------------------------------------------|
| `object`  | string        | Yes      | N/A     | Specifies the format of the prompt data: `"llm.utf-8"` (for a single, complete prompt) or `"llm.utf-8.stream"` (for a chunked prompt).      |
| `data`    | string/object | Yes      | N/A     | If `object` is `"llm.utf-8"`, `data` is a string containing the full prompt. If `object` is `"llm.utf-8.stream"`, `data` is an object: `{"index": <int>, "delta": "<prompt_chunk>", "finish": <boolean>}`. |

**Request Example (Non-streaming):**
```json
{
  "request_id": "infer_llm_003",
  "work_id": "llm.1002",
  "action": "inference",
  "object": "llm.utf-8",
  "data": "What is the capital of France?"
}
```

**Request Example (Streaming Prompt - Final Chunk):**
```json
{
  "request_id": "infer_llm_004",
  "work_id": "llm.1002",
  "action": "inference",
  "object": "llm.utf-8.stream",
  "data": {
    "index": 0,
    "delta": "Tell me a short story about a brave robot.",
    "finish": true
  }
}
```

**Response (Success - Request Accepted):**
This response only confirms that the prompt has been received. The generated text will follow in asynchronous messages.
```json
{
  "request_id": "infer_llm_003",
  "work_id": "llm.1002",
  "action": "inference",
  "created": 1678886402,
  "code": 0,
  "message": "OK" // Or a message like "Inference process started"
}
```

**Response (Error - Task Not Found):**
```json
{
  "request_id": "infer_llm_005",
  "work_id": "llm.9999", // Non-existent task
  "action": "inference",
  "created": 1678886403,
  "error": {
    "code": -6,
    "message": "Unit Does Not Exist"
  }
}
```

### **link**

Links an LLM task instance to an input source (e.g., an ASR or KWS unit) dynamically after setup.

-   **`work_id` in request**: `"llm.XXXX"`

**Request Parameters:**

| Parameter | Type   | Required | Default | Description                                                                    |
|-----------|--------|----------|---------|--------------------------------------------------------------------------------|
| `object`  | string | Yes      | N/A     | Typically `"work_id"` indicating the data field contains a unit ID to link to. |
| `data`    | string | Yes      | N/A     | The `work_id` of the unit to link as an input source (e.g., `"asr.1001"`).      |

**Request Example:**
```json
{
  "request_id": "link_llm_006",
  "work_id": "llm.1002",
  "action": "link",
  "object": "work_id",
  "data": "asr.1001"
}
```

**Response (Success):**
```json
{
  "request_id": "link_llm_006",
  "work_id": "llm.1002",
  "action": "link",
  "created": 1678886404,
  "code": 0,
  "message": "OK"
}
```

### **unlink**

Unlinks a previously linked input source from an LLM task instance.

-   **`work_id` in request**: `"llm.XXXX"`

**Request Parameters:**

| Parameter | Type   | Required | Default | Description                                                                        |
|-----------|--------|----------|---------|------------------------------------------------------------------------------------|
| `object`  | string | Yes      | N/A     | Typically `"work_id"` indicating the data field contains a unit ID to unlink.      |
| `data`    | string | Yes      | N/A     | The `work_id` of the input source unit to unlink (e.g., `"asr.1001"`).             |

**Request Example:**
```json
{
  "request_id": "unlink_llm_007",
  "work_id": "llm.1002",
  "action": "unlink",
  "object": "work_id",
  "data": "asr.1001"
}
```

**Response (Success):**
```json
{
  "request_id": "unlink_llm_007",
  "work_id": "llm.1002",
  "action": "unlink",
  "created": 1678886405,
  "code": 0,
  "message": "OK"
}
```

### **pause**

Pauses the LLM task instance, causing it to stop any ongoing text generation.

-   **`work_id` in request**: `"llm.XXXX"`

**Request Parameters**: None beyond standard `request_id`, `work_id`, `action`.

**Request Example:**
```json
{
  "request_id": "pause_llm_008",
  "work_id": "llm.1002",
  "action": "pause"
}
```

**Response (Success):**
```json
{
  "request_id": "pause_llm_008",
  "work_id": "llm.1002",
  "action": "pause",
  "created": 1678886406,
  "code": 0,
  "message": "OK"
}
```

### **work**

Resumes a paused LLM task instance. Note: This does not restart a completed inference but allows a paused generation (if supported by the underlying model) to continue or new inferences to be accepted. The primary way to generate text is via the `inference` API.

-   **`work_id` in request**: `"llm.XXXX"`

**Request Parameters**: None beyond standard `request_id`, `work_id`, `action`.

**Request Example:**
```json
{
  "request_id": "work_llm_009",
  "work_id": "llm.1002",
  "action": "work"
}
```

**Response (Success):**
```json
{
  "request_id": "work_llm_009",
  "work_id": "llm.1002",
  "action": "work",
  "created": 1678886407,
  "code": 0,
  "message": "OK"
}
```

### **exit**

Stops and removes an LLM task instance, freeing its resources.

-   **`work_id` in request**: `"llm.XXXX"`

**Request Parameters**: None beyond standard `request_id`, `work_id`, `action`.

**Request Example:**
```json
{
  "request_id": "exit_llm_010",
  "work_id": "llm.1002",
  "action": "exit"
}
```

**Response (Success):**
```json
{
  "request_id": "exit_llm_010",
  "work_id": "llm.1002", // work_id of the exited task
  "action": "exit",
  "created": 1678886408,
  "code": 0,
  "message": "OK"
}
```

### **taskinfo**

Retrieves information about active LLM tasks.

**Case 1: General LLM Unit Information**

-   **`work_id` in request**: `"llm"`
-   **Description**: Returns a list of all active LLM task instance IDs.

**Request Example:**
```json
{
  "request_id": "taskinfo_llm_general_011",
  "work_id": "llm",
  "action": "taskinfo"
}
```

**Response (Success):**
```json
{
  "request_id": "taskinfo_llm_general_011",
  "work_id": "llm",
  "action": "taskinfo",
  "created": 1678886409,
  "code": 0,
  "message": "OK",
  "object": "llm.tasklist",
  "data": [
    "llm.1002" // List of active instance IDs
  ]
}
```

**Case 2: Specific LLM Task Instance Information**

-   **`work_id` in request**: `"llm.XXXX"` (e.g., `"llm.1002"`)
-   **Description**: Returns runtime parameters and configuration of the specified LLM task instance.

**Request Example:**
```json
{
  "request_id": "taskinfo_llm_specific_012",
  "work_id": "llm.1002",
  "action": "taskinfo"
}
```

**Response (Success):**
```json
{
  "request_id": "taskinfo_llm_specific_012",
  "work_id": "llm.1002",
  "action": "taskinfo",
  "created": 1678886410,
  "code": 0,
  "message": "OK",
  "object": "llm.taskinfo",
  "data": {
    "model": "qwen2.5-0.5B-prefill-20e",
    "response_format": "llm.utf-8.stream",
    "enoutput": true,
    "inputs": ["llm.utf-8", "asr.1001"], // Array of configured input sources
    "max_token_len": 512,
    "prompt": "You are a helpful AI assistant."
    // Other relevant parameters like temperature, top_p are part of the model's internal config
    // and not typically listed in taskinfo unless explicitly added in main.cpp.
  }
}
```

> **Note on `work_id`**: The numeric part of a task instance `work_id` (e.g., `1002` in `llm.1002`) is dynamically assigned.

## Relation to Configuration Files and Scripts

The `llm-llm` unit's behavior is heavily defined by model-specific JSON configuration files and, for some models, external tokenizer services run by Python scripts.

### Model Configuration JSON (e.g., `models/mode_qwen2.5-0.5B-prefill-20e.json`)

These files, located in the `projects/llm_framework/main_llm/models/` directory (or a system-wide model path), contain parameters for the LLM engine:

-   **`mode`**: The model name used in the `setup` API (e.g., `"qwen2.5-0.5B-prefill-20e"`).
-   **`type`**: `"llm"`.
-   **`capabilities`**: E.g., `"text_generation"`, `"chat"`.
-   **`mode_param` Object**: Contains core model and inference settings:
    -   `tokenizer_type` (integer): Specifies the tokenizer algorithm (e.g., LLaMa, Qwen).
    -   `filename_tokenizer_model` (string): Path to the tokenizer model file (e.g., `"qwen.tiktoken"`) or an HTTP URL (e.g., `"http://localhost:PORT"`) if a tokenizer server is used.
    -   `filename_tokens_embed` (string): Path to token embeddings file.
    -   `filename_post_axmodel` (string): Path to the post-processing model file.
    -   `template_filename_axmodel` (string): Template for layer-specific model files (e.g., `"qwen2_p128_l%d_together.axmodel"`).
    -   `axmodel_num` (integer): Number of model layers.
    -   `tokens_embed_num`, `tokens_embed_size`: Token embedding dimensions.
    -   `max_token_len` (integer): Default maximum number of tokens to generate. Can be overridden by `setup` API.
    -   `temperature`, `top_p`, `top_k`, `repetition_penalty`: Standard LLM inference parameters controlling randomness and creativity. These are loaded by `LLM::Init` and are **not** typically overridable by the `setup` API's `data.mode_param` field as per current `main.cpp` logic.
    -   `system_prompt` (string, often found in chat models): A default system prompt. Can be overridden by `setup` API's `prompt` field.
    -   Other model-specific paths and flags (e.g., `b_use_topk`, `b_bos`, `b_eos`).

### Tokenizer Server Scripts (e.g., `scripts/*_tokenizer.py`)

-   **Purpose**: Some models, particularly those using tokenizers like `tiktoken` (e.g., Qwen models), may utilize an external Python-based tokenizer server.
-   **Operation**:
    1.  If the `filename_tokenizer_model` in the model's JSON configuration is an HTTP URL (e.g., `"http://localhost:DYNAMIC_PORT"`), the `llm-llm` unit's `main.cpp` will attempt to fork a Python script.
    2.  The script (e.g., `scripts/qwen2.5-0.5B-prefill-20e_tokenizer.py` or a generic one) starts a local HTTP server on a dynamically assigned port (starting from 8080).
    3.  The `LLM::Init` then configures the C++ tokenizer client to communicate with this Python server for tokenization tasks (encode/decode).
    4.  The `prompt` provided in the `setup` API might be passed to this server during its initialization.
-   This mechanism allows using complex Python-based tokenizers that might be difficult to implement or maintain purely in C++.

Understanding these configurations is crucial for selecting models, setting appropriate parameters, and diagnosing issues related to model loading or inference behavior.