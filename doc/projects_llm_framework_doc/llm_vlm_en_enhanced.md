# llm-vlm - Multimodal Inference Unit

The `llm-vlm` unit is a core component of the LLM framework, designed to provide powerful multimodal inference services. It can process both text and image inputs and generate text-based responses. This unit is highly configurable and can be linked with other units like Keyword Spotting (KWS) or Automatic Speech Recognition (ASR) to create sophisticated AI pipelines.

## Key Features

*   **Multimodal Input**: Accepts text and image data for inference.
*   **Flexible Configuration**: Allows detailed setup of models, response formats, and operational parameters.
*   **Streaming Support**: Handles both streaming and non-streaming data for inputs and outputs.
*   **Dynamic Linking**: Can be dynamically linked and unlinked with other service units.
*   **Task Management**: Supports multiple inference tasks and provides information about their status.

## API Actions

The `llm-vlm` unit exposes several actions to control its behavior:

### 1. `setup`

Initializes and configures an `llm-vlm` task. This is the first step before performing inference.

**Request JSON:**

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
    "prompt": "You are a knowledgeable assistant capable of answering various questions and providing information.",
    "tokenizer_type": "TKT_Qwen", // Example: Can be TKT_LLaMa, TKT_MINICPM, TKT_Phi3, TKT_Qwen, TKT_HTTP
    "filename_tokenizer_model": "qwen.tiktoken", // Path or HTTP endpoint for tokenizer model
    "filename_tokens_embed": "model_embed.bin", // Path to token embeddings
    "filename_post_axmodel": "model_post.axmodel", // Path to post-processing AXModel
    "filename_vpm_resampler_axmodedl": "model_vpm_resampler.axmodel", // Path to VPM resampler AXModel
    "template_filename_axmodel": "model_layer_{}.axmodel", // Template for AXModel layer files
    "b_use_topk": true,
    "b_vpm_two_stage": false,
    "b_bos": true, // Add Beginning-Of-Sentence token
    "b_eos": true, // Add End-Of-Sentence token
    "axmodel_num": 2, // Number of AXModel layers
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

**Parameters in `data`:**

*   `request_id` (string, required): A unique identifier for the request.
*   `work_id` (string, required): Must be set to `vlm` when creating a new task. If a specific `work_id` (e.g., `vlm.1003`) is provided, it attempts to reconfigure that existing task.
*   `action` (string, required): Set to `setup`.
*   `object` (string, required): Data type, typically `vlm.setup`.
*   `model` (string, required): Specifies the multimodal model to be used (e.g., `internvl2.5-1B-ax630c`). The system loads model configuration files based on this name from predefined paths (`base_model_path_` and `base_model_config_path_`).
*   `response_format` (string, required): Defines the format of the output.
    *   `vlm.utf-8`: Non-streaming, single UTF-8 encoded text response.
    *   `vlm.utf-8.stream`: Streaming, multiple UTF-8 encoded text chunks.
*   `input` (string or array of strings, required): Specifies the input sources and formats.
    *   `vlm.utf-8`: Direct text input to the VLM.
    *   `vlm.jpeg.stream.base64`: Base64 encoded JPEG image data.
    *   `kws.xxxx`: Links to a Keyword Spotting unit (e.g., `kws.1000`).
    *   `asr.xxxx`: Links to an Automatic Speech Recognition unit (e.g., `asr.1001`).
    *   `whisper.xxxx`: Links to a Whisper ASR unit.
*   `enoutput` (boolean, required): If `true`, the unit will send its output. If `false`, output is suppressed.
*   `max_token_len` (integer, optional): Maximum number of tokens in the generated response. Defaults to a model-specific value if not provided.
*   `prompt` (string, optional): A system prompt to guide the model's behavior.
*   **Model Configuration Parameters (optional, typically loaded from model's JSON config file but can be overridden here):**
    *   `tokenizer_type`: Type of tokenizer (e.g., `TKT_LLaMa`, `TKT_Qwen`).
    *   `filename_tokenizer_model`: Path or HTTP URL for the tokenizer model. If an HTTP URL is provided, a local Python tokenizer server might be launched.
    *   `filename_tokens_embed`: Path to token embedding files.
    *   `filename_post_axmodel`: Path to the post-processing AXModel file.
    *   `filename_vpm_resampler_axmodedl`: Path to the Vision Preprocessing Module (VPM) resampler AXModel.
    *   `template_filename_axmodel`: Template string for loading AXModel layers (e.g., `model_layer_{}.axmodel`).
    *   `b_use_topk` (boolean): Whether to use top-k sampling.
    *   `b_vpm_two_stage` (boolean): Whether VPM uses a two-stage process.
    *   `b_bos` (boolean): Prepend a Beginning-Of-Sentence token to the input.
    *   `b_eos` (boolean): Append an End-Of-Sentence token to the input.
    *   `axmodel_num` (integer): Number of AXModel layers.
    *   `tokens_embed_num` (integer): Number of token embedding files/sections.
    *   `img_token_id` (integer): Special token ID used for representing images in the input sequence.
    *   `tokens_embed_size` (integer): Size of the token embeddings.
    *   `b_use_mmap_load_embed` (boolean): Use memory-mapping for loading embeddings (faster).
    *   `b_dynamic_load_axmodel_layer` (boolean): Load AXModel layers dynamically as needed.
    *   `temperature` (float): Controls randomness in generation. Higher values mean more randomness.
    *   `top_p` (float): Nucleus sampling parameter.
    *   `vpm_width` (integer): Target width for image preprocessing by VPM.
    *   `vpm_height` (integer): Target height for image preprocessing by VPM.

**Response JSON (Success):**

```json
{
  "created": 1737599810, // Unix timestamp of creation
  "data": "None",
  "error": {
    "code": 0,
    "message": ""
  },
  "object": "None",
  "request_id": "2",
  "work_id": "vlm.1003" // Unique work_id assigned to this task instance
}
```

**Response JSON (Error):**
If `task_count_` limit is reached:
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
If model loading fails:
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

Sends data (text or image) to the configured `llm-vlm` task for processing.

**Request JSON (Text Input):**

```json
{
  "request_id": "4",
  "work_id": "vlm.1003", // Target work_id from setup
  "action": "inference",
  "object": "vlm.utf-8.stream", // or "vlm.utf-8" for non-streaming
  "data": {
    "delta": "May I know your name?", // Text input chunk
    "index": 0, // Index of the current chunk (for streaming)
    "finish": true // True if this is the last chunk
  }
}
```

**Request JSON (Image Input - Base64 Encoded JPEG):**

```json
{
  "request_id": "4",
  "work_id": "vlm.1003",
  "action": "inference",
  "object": "vlm.jpeg.stream.base64", // Indicates base64 encoded JPEG image
  "data": {
    "delta": "U29tZSByYW5kb20gYmFzZTY0IGRhdGEgZm9yIHlvdSB0byBjb25zdWx0IGFwcGxpY2F0aW9ucy4=", // Base64 data
    "index": 0,
    "finish": true
  }
}
```
> **Note:** For image data, `delta` should contain the complete base64 encoded image data, and `finish` should be `true`. The streaming `index` might still be used if the image is part of a larger sequence of inputs, but the image itself is sent as one "delta".

**Parameters in `data` (for streaming `object` types):**
*   `delta` (string): The data chunk. For text, this is part of the user's query. For images, this is the base64 encoded image data.
*   `index` (integer): The sequential index of the data chunk. Starts from 0.
*   `finish` (boolean): Set to `true` if this is the last chunk of the current input sequence.

**Response JSON (Non-streaming - `vlm.utf-8`):**

```json
{
  "created": 1737600915,
  "data": "I am an AI assistant whose name is LittleAI. How can I help you today?", // Complete response
  "error": {
    "code": 0,
    "message": ""
  },
  "object": "vlm.utf-8",
  "request_id": "4",
  "work_id": "vlm.1003"
}
```

**Response JSON (Streaming - `vlm.utf-8.stream`):**
A series of messages, each containing a part of the response.

```json
// Message 1
{
  "created": 1737600539,
  "data": {
    "delta": "I am an", // Part of the response
    "finish": false,    // Indicates more data is coming
    "index": 0          // Index of this response chunk
  },
  "error": {"code": 0, "message": ""},
  "object": "vlm.utf-8.stream",
  "request_id": "4",
  "work_id": "vlm.1003"
}
```
```json
// ... more messages ...
```
```json
// Final message for the stream
{
  "created": 1737600540,
  "data": {
    "delta": "",       // Empty delta for the final message
    "finish": true,    // Indicates this is the end of the stream
    "index": 4
  },
  "error": {"code": 0, "message": ""},
  "object": "vlm.utf-8.stream",
  "request_id": "4",
  "work_id": "vlm.1003"
}
```

### 3. `link`

Connects the output of another unit (e.g., `kws`, `asr`) as an input to this `llm-vlm` task. This allows the `llm-vlm` unit to react to events or data from other parts of the system.

**Request JSON:**

```json
{
  "request_id": "3",
  "work_id": "vlm.1003", // The llm-vlm task to link to
  "action": "link",
  "object": "work_id",   // Specifies that 'data' contains a work_id
  "data": "kws.1000"   // The work_id of the unit to link from (e.g., a KWS task)
}
```

**Behavior:**
*   If `kws.xxxx` is linked: When the KWS unit detects a keyword, `llm-vlm` might stop its current inference (if any) and potentially use the KWS output or simply be triggered to listen for subsequent ASR input. The `kws_awake` function in `main.cpp` handles this by calling `lLaMa_->Stop()`.
*   If `asr.xxxx` (or `whisper.xxxx`) is linked: The recognized text from the ASR unit is fed as input to the `llm-vlm` task for inference. The `task_asr_data` function processes this.

**Response JSON (Success):**

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
**Response JSON (Error):**
If the link target is invalid or linking fails:
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

> **Note:** Linking can also be configured during the `setup` phase by including other unit `work_id`s in the `input` array.

### 4. `unlink`

Disconnects a previously linked unit.

**Request JSON:**

```json
{
  "request_id": "4",
  "work_id": "vlm.1003",
  "action": "unlink",
  "object": "work_id",
  "data": "kws.1000" // The work_id of the unit to unlink
}
```

**Response JSON (Success):**

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

Temporarily stops the `llm-vlm` task. Any ongoing inference will be halted. The unit will not process new input data while paused.

**Request JSON:**

```json
{
  "request_id": "5",
  "work_id": "vlm.1003",
  "action": "pause"
}
```

**Response JSON (Success):**

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

Resumes a paused `llm-vlm` task. The unit will start processing new input data and can continue any previously interrupted tasks if the model supports resumption (typically, it starts fresh).

**Request JSON:**

```json
{
  "request_id": "6",
  "work_id": "vlm.1003",
  "action": "work"
}
```
> **Note:** The `main.cpp` implementation of `llm_llm` (the class managing `llm_task` instances) does not currently have an explicit `work` method that directly overrides `StackFlow::work`. Resumption of work is typically handled by sending new inference data after a `pause`. The `StackFlow` base class might have a default `work` implementation.

**Response JSON (Success):**

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

Terminates the `llm-vlm` task and releases its resources. This includes de-initializing the model and stopping any associated processes like a Python tokenizer server.

**Request JSON:**

```json
{
  "request_id": "7",
  "work_id": "vlm.1003",
  "action": "exit"
}
```

**Response JSON (Success):**

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
If the specified `work_id` does not exist:
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
  "work_id": "vlm.1003" // The requested work_id
}
```

### 8. `taskinfo`

Retrieves information about running `llm-vlm` tasks.

**Request JSON (Get list of all `vlm` tasks):**

```json
{
  "request_id": "2",
  "work_id": "vlm", // General work_id for the vlm unit type
  "action": "taskinfo"
}
```

**Response JSON (List of tasks):**

```json
{
  "created": 1737600076,
  "data": [
    "vlm.1003", // List of active work_ids
    "vlm.1004"
  ],
  "error": {"code": 0, "message": ""},
  "object": "vlm.tasklist",
  "request_id": "2",
  "work_id": "vlm"
}
```

**Request JSON (Get specific task parameters):**

```json
{
  "request_id": "2",
  "work_id": "vlm.1003", // Specific work_id of the task
  "action": "taskinfo"
}
```

**Response JSON (Specific task parameters):**

```json
{
  "created": 1737600098,
  "data": {
    "model": "internvl2.5-1B-ax630c",
    "response_format": "vlm.utf-8.stream",
    "enoutput": true,
    "inputs": [ // Current input sources/links
      "vlm.utf-8",
      "kws.1000"
    ]
    // Other setup parameters like max_token_len, prompt, etc., may also be included.
  },
  "error": {"code": 0, "message": ""},
  "object": "vlm.taskinfo",
  "request_id": "2",
  "work_id": "vlm.1003"
}
```
If the specified `work_id` for specific task info does not exist:
```json
{
  "created": 1737600099,
  "data": "None", // Or specific error object in data
  "error": {
    "code": -6,
    "message": "Unit Does Not Exist"
  },
  "object": "None", // Or "vlm.taskinfo" with error details in "data"
  "request_id": "2",
  "work_id": "vlm.1003"
}
```

## Prompt Formatting

The `prompt_complete` method in `main.cpp` shows that different prompt structures are used based on the `tokenizer_type`:
*   **TKT_LLaMa:** `<|user|>\n{input}</s><|assistant|>\n`
*   **TKT_MINICPM:** `<用户>{input}<AI>`
*   **TKT_Phi3:** `{input} ` (a space is appended)
*   **TKT_Qwen:** `<|im_start|>system\n{prompt}.<|im_end|>\n<|im_start|>user\n{input}<|im_end|>\n<|im_start|>assistant\n`
*   **TKT_HTTP/Default:** The input is used as is.

The `{prompt}` placeholder is taken from the `prompt` field in the `setup` configuration, and `{input}` is the user-provided data from the `inference` call.

## Error Codes

Common error codes encountered:
*   `0`: No error, success.
*   `-2`: JSON format error in request.
*   `-5`: Model loading failed (e.g., model files not found, configuration issues).
*   `-6`: Unit/Task Does Not Exist (e.g., specified `work_id` is invalid or task has exited).
*   `-11`: Model run failed (generic inference error).
*   `-20`: Link operation failed.
*   `-21`: Task full (maximum number of tasks reached, controlled by `task_count_`).
*   `-23`: Base64 decoding error.
*   `-25`: Stream data index error (e.g., missing or out-of-order chunks).

## Internal Operations (from `main.cpp`)

*   **Tokenizer Server**: For certain tokenizer types (e.g., `TKT_HTTP` or when `filename_tokenizer_model` is an HTTP URL), a Python-based tokenizer server can be automatically launched using `fork` and `execl`. This server listens on a local port (starting from 8090, incrementing for each task requiring it).
*   **Model Configuration Loading**: Model-specific parameters (paths to model files, embedding sizes, etc.) are loaded from JSON configuration files located in a base model path (e.g., `/opt/m5stack/models/`) combined with the model name specified in the `setup` call.
*   **Image Processing**: When image data (`vlm.jpeg.stream.base64`) is received, it's decoded from base64. The raw image bytes are then passed to `lLaMa_->Encode(src, img_embed)` to get image embeddings, which are then combined with text prompt embeddings for multimodal inference.
*   **Callback Mechanism**: The `llm_llm` class uses `std::bind` to create callbacks for handling output data (`task_output`), data from linked units (`task_asr_data`, `kws_awake`), and user data (`task_user_data`).

> **Note:** `work_id`s for tasks (e.g., `vlm.1003`) are generated sequentially. The numeric part is not a fixed index but an incrementing counter.
