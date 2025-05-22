# LLM Framework Specification

## 1. Introduction

This document outlines the specification for the `llm_framework`, a flexible and extensible platform designed for building and deploying applications powered by Large Language Models (LLMs).

**Purpose:**

The primary purpose of the `llm_framework` is to provide a standardized, modular, and scalable environment for integrating various processing units (e.g., speech-to-text, text-to-speech, computer vision, LLM interfaces) to create sophisticated AI-driven applications. It aims to simplify the development, deployment, and management of complex LLM-based workflows.

**General Architecture:**

The `llm_framework` is built upon a **StackFlow-based architecture**. This means that data flows through a stack of interconnected processing units, where each unit performs a specific task and passes its output to the next unit in the sequence.

Communication between these units is facilitated by **ZeroMQ (ZMQ)**, a high-performance asynchronous messaging library. ZMQ ensures reliable and efficient data exchange, allowing for decoupled units that can potentially run on different processes or even different machines.

## 2. Core Components (Units)

This section will detail the individual processing units that form the building blocks of the `llm_framework`. Each unit is designed to be a self-contained module with a specific responsibility.

*(This section will be populated with details for each unit as they are defined. Examples might include: SpeechToTextUnit, TextToSpeechUnit, LLMQueryUnit, ImageInputUnit, ObjectDetectionUnit, etc.)*

---

*   **llm-sys (System/Orchestration Unit)**
    *   **Purpose:** The main control unit for the StackFlow framework. Responsible for initializing, configuring, connecting (linking), and resetting other processing units. It manages the overall state and data flow pipelines.
    *   **Inputs:** JSON messages for actions like "reset", "setup" (for other units), and "link" (between other units). Typically receives commands from an external interface (e.g., UART, TCP).
    *   **Outputs:** JSON responses indicating the status of requested actions (e.g., success/failure of unit setup or linking).
    *   **Configuration:**
        *   Manages global configurations.
        *   Receives setup parameters for other units.
        *   Example action:
            ```json
            {
                "request_id": "11212155",
                "work_id": "sys",
                "action": "reset"
            }
            ```

---

*   **llm-audio (Audio Input/Output Unit)**
    *   **Purpose:** Provides system audio support. This can include capturing audio from input devices (microphones) and potentially playing back audio to output devices, depending on its specific role in a workflow (e.g. `sys.pcm` often serves as an input source for other units like KWS or ASR, and can be an output for TTS units).
    *   **Inputs:** Control messages for configuration. For audio playback, it would receive audio data (e.g., PCM stream) from a ZMQ socket (e.g. from a TTS unit).
    *   **Outputs:** Provides audio data (e.g., `sys.pcm`) as a ZMQ stream to other units (like KWS, ASR). The format `sys.pcm` suggests raw audio data.
    *   **Configuration:** Specific audio device settings (e.g., input device ID, output device ID, sample rate, channels). Details not explicitly in READMEs but inferred from its role.

---

*   **llm-kws (Keyword Spotting Unit)**
    *   **Purpose:** Keyword detection unit, providing keyword wakeup service. It listens to an audio stream and triggers an event when a predefined keyword is detected.
    *   **Inputs:** Audio stream (e.g., `sys.pcm`) from a ZMQ socket (typically from `llm-audio`).
    *   **Outputs:** A signal or message (e.g., `kws.bool`) indicating keyword detection to a ZMQ socket. This output can be linked to other units (like ASR or LLM) to activate them.
    *   **Configuration:**
        *   `model`: The keyword spotting model to be used (e.g., "sherpa-onnx-kws-zipformer-wenetspeech-3.3M-2024-01-01").
        *   `response_format`: Expected output format (e.g., "kws.bool").
        *   `input`: Source of the audio stream (e.g., "sys.pcm").
        *   `enoutput`: Boolean, enable output stream.
        *   `kws`: The specific keyword(s) to detect (e.g., "你好你好").
        *   Example setup:
            ```json
            {
                "request_id": "1",
                "work_id": "kws",
                "action": "setup",
                "object": "kws.setup",
                "data": {
                    "model": "sherpa-onnx-kws-zipformer-wenetspeech-3.3M-2024-01-01",
                    "response_format": "kws.bool",
                    "input": "sys.pcm",
                    "enoutput": true,
                    "kws": "你好你好"
                }
            }
            ```

---

*   **llm-asr (Automatic Speech Recognition Unit)**
    *   **Purpose:** Speech-to-text unit, providing speech-to-text service. Converts spoken audio into textual representation.
    *   **Inputs:** Audio stream (e.g., `sys.pcm`) from a ZMQ socket (typically from `llm-audio`, often activated by `llm-kws`).
    *   **Outputs:** Transcribed text (e.g., `asr.utf-8.stream`) to a ZMQ socket.
    *   **Configuration:**
        *   `model`: The ASR model to be used (e.g., "sherpa-ncnn-streaming-zipformer-zh-14M-2023-02-23").
        *   `response_format`: Output text format (e.g., "asr.utf-8.stream").
        *   `input`: Source of the audio stream (e.g., "sys.pcm").
        *   `enoutput`: Boolean, enable output stream.
        *   `enkws`: Boolean, enable linking with KWS (suggests it might wait for a KWS trigger if true).
        *   `rule1`, `rule2`, `rule3`: Model-specific parameters (e.g., 2.4, 1.2, 30.1).
        *   Example setup:
            ```json
            {
                "request_id": "2",
                "work_id": "asr",
                "action": "setup",
                "object": "asr.setup",
                "data": {
                    "model": "sherpa-ncnn-streaming-zipformer-zh-14M-2023-02-23",
                    "response_format": "asr.utf-8.stream",
                    "input": "sys.pcm",
                    "enoutput": true,
                    "enkws":true,
                    "rule1":2.4,
                    "rule2":1.2,
                    "rule3":30.1
                }
            }
            ```

---

*   **llm-llm (Large Language Model Unit)**
    *   **Purpose:** Large model unit, providing AI large model inference service. Processes input text (and potentially other modalities in the future) to generate responses, perform reasoning, etc.
    *   **Inputs:** Text input (e.g., `llm.utf-8` or from `asr.utf-8.stream`) from a ZMQ socket.
    *   **Outputs:** LLM-generated text (e.g., `llm.utf-8.stream`) to a ZMQ socket.
    *   **Configuration:**
        *   `model`: The LLM model to be used (e.g., "qwen2.5-0.5B-prefill-20e").
        *   `response_format`: Output text format (e.g., "llm.utf-8.stream").
        *   `input`: Source of the input text (e.g., "llm.utf-8", which implies it can receive direct text input, or it can be linked from ASR's output).
        *   `enoutput`: Boolean, enable output stream.
        *   `max_token_len`: Maximum token length for the response (e.g., 256).
        *   `prompt`: System prompt to guide the LLM's behavior (e.g., "You are a knowledgeable assistant...").
        *   Example setup:
            ```json
            {
                "request_id": "3",
                "work_id": "llm",
                "action": "setup",
                "object": "llm.setup",
                "data": {
                    "model": "qwen2.5-0.5B-prefill-20e",
                    "response_format": "llm.utf-8.stream",
                    "input": "llm.utf-8",
                    "enoutput": true,
                    "max_token_len": 256,
                    "prompt": "You are a knowledgeable assistant capable of answering various questions and providing information."
                }
            }
            ```

---

*   **llm-melotts (Melody Text-to-Speech Unit)**
    *   **Purpose:** NPU-accelerated TTS unit, providing text-to-speech and playback service. Converts text into audible speech, optimized with NPU.
    *   **Inputs:** Text input (e.g., `tts.utf-8`, typically from `llm-llm`) from a ZMQ socket.
    *   **Outputs:** Synthesized audio (e.g., `sys.pcm`) to a ZMQ socket, presumably for `llm-audio` to play. `enoutput: false` in the example suggests its output might be directly consumed by an audio playback mechanism tied to `sys.pcm` without needing a separate link for playback if `llm-audio` handles it.
    *   **Configuration:**
        *   `model`: The TTS model to be used (e.g., "melotts_zh-cn").
        *   `response_format`: Output audio format (e.g., "sys.pcm").
        *   `input`: Source of the input text (e.g., "tts.utf-8").
        *   `enoutput`: Boolean, enable output stream (set to `false` in example, which might mean it directly plays or sends to a predefined audio output channel).
        *   Example setup:
            ```json
            {
                "request_id": "4",
                "work_id": "melotts",
                "action": "setup",
                "object": "melotts.setup",
                "data": {
                    "model": "melotts_zh-cn",
                    "response_format": "sys.pcm",
                    "input": "tts.utf-8",
                    "enoutput": false
                }
            }
            ```

---

*   **llm-tts (CPU Text-to-Speech Unit)**
    *   **Purpose:** CPU-computed TTS unit, providing text-to-speech and playback service. An alternative to `llm-melotts`, likely used when NPU acceleration is unavailable or not desired.
    *   **Inputs:** Text input (similar to `llm-melotts`, e.g., `tts.utf-8`) from a ZMQ socket.
    *   **Outputs:** Synthesized audio (similar to `llm-melotts`, e.g., `sys.pcm`) to a ZMQ socket.
    *   **Configuration:** Similar to `llm-melotts` but would use CPU-based models. Specific model names and parameters would differ. (Detailed JSON example not available in provided READMEs, but functionality is stated).

---

*   **llm-camera (Camera Input Unit)**
    *   **Purpose:** Captures video frames from a camera device.
    *   **Inputs:** Device path for the camera (e.g., "/dev/video0"). Configuration messages.
    *   **Outputs:** Raw camera frames (e.g., `camera.raw`) to a ZMQ socket.
    *   **Configuration:**
        *   `response_format`: Output frame format (e.g., "camera.raw").
        *   `input`: Camera device path (e.g., "/dev/video0").
        *   `enoutput`: Boolean, enable output stream (set to `false` in example, may imply direct use by a linked unit or internal handling).
        *   `frame_width`: Width of the captured frame (e.g., 320).
        *   `frame_height`: Height of the captured frame (e.g., 320).
        *   Example setup:
            ```json
            {
               "request_id":"4",
               "work_id":"camera",
               "action":"setup",
               "object":"camera.setup",
               "data":{
                  "response_format":"camera.raw",
                  "input":"/dev/video0",
                  "enoutput":false,
                  "frame_width":320,
                  "frame_height":320
               }
            }
            ```

---

*   **llm-yolo (Object Detection Unit)**
    *   **Purpose:** Performs object detection on input images using YOLO models.
    *   **Inputs:** Image data (e.g., `camera.raw` from `llm-camera.1000` instance) from a ZMQ socket.
    *   **Outputs:** Object detection results (e.g., bounding boxes, class labels, in `yolo.yolobox` format) to a ZMQ socket.
    *   **Configuration:**
        *   `model`: The YOLO model to be used (e.g., "yolo11n").
        *   `response_format`: Output format for detection results (e.g., "yolo.yolobox").
        *   `input`: Source of the image data (e.g., "camera.1000", referring to an instance of `llm-camera`).
        *   `enoutput`: Boolean, enable output stream.
        *   Example setup:
            ```json
            {
               "request_id":"5",
               "work_id":"yolo",
               "action":"setup",
               "object":"yolo.setup",
               "data":{
                  "model":"yolo11n",
                  "response_format":"yolo.yolobox",
                  "input":"camera.1000",
                  "enoutput":true
               }
            }
            ```

**General Unit Structure:**

For each unit, the following information will be provided:

*   **[Unit Name]**
    *   **Purpose:** A brief description of what the unit does.
    *   **Inputs:** Expected input data format and source.
    *   **Outputs:** Produced output data format and destination.
    *   **Configuration:** Specific parameters and settings for the unit.

---

---

## 3. System Workflows

This section describes common operational modes or workflows that can be implemented using the `llm_framework` by connecting various units in specific configurations.

### 3.1. Voice Assistant Mode

**Description:** This workflow enables a voice-controlled assistant experience. The system listens for a keyword, transcribes the subsequent speech, processes it with an LLM to get a response, and then synthesizes that response back into audio. This mode typically involves the `llm-sys` unit for orchestration and `llm-audio` for handling raw audio input and output.

**Unit Sequence and Data Flow:**

1.  **`llm-audio` (Implicit):** Continuously provides PCM audio stream (`sys.pcm`).
2.  **`llm-kws` (Keyword Spotting):**
    *   **Input:** PCM audio stream (`sys.pcm`) from `llm-audio`.
    *   **Action:** Detects a predefined keyword (e.g., "你好你好").
    *   **Output:** A trigger signal (e.g., `kws.bool`) upon keyword detection.
3.  **`llm-asr` (Automatic Speech Recognition):**
    *   **Input:** PCM audio stream (`sys.pcm`) from `llm-audio`, activated by the `llm-kws` trigger.
    *   **Action:** Transcribes the spoken audio into text.
    *   **Output:** Text stream (e.g., `asr.utf-8.stream`).
4.  **`llm-llm` (Large Language Model):**
    *   **Input:** Text stream (`asr.utf-8.stream`) from `llm-asr`.
    *   **Action:** Processes the input text, performs reasoning, and generates a response based on its configured prompt and model.
    *   **Output:** Text stream (e.g., `llm.utf-8.stream`) containing the LLM's response.
5.  **`llm-melotts` or `llm-tts` (Text-to-Speech):**
    *   **Input:** Text stream (`llm.utf-8.stream`) from `llm-llm`.
    *   **Action:** Synthesizes the text response into speech. `llm-melotts` uses NPU acceleration, while `llm-tts` uses CPU.
    *   **Output:** Synthesized audio data (e.g., in `sys.pcm` format), which is then played back, typically via `llm-audio`.

**Simplified Data Flow:**
PCM Audio (`llm-audio`) -> Keyword Detection Signal (`llm-kws`) -> Text (`llm-asr`) -> Text Response (`llm-llm`) -> Synthesized PCM Audio (`llm-melotts`/`llm-tts` for playback via `llm-audio`).

**Orchestration:**
The `llm-sys` unit is responsible for setting up each of these units (e.g., loading models, defining input/output topics like `sys.pcm`, `asr.utf-8.stream`) and linking their inputs and outputs to form the described pipeline. For example, `llm-asr` is linked to `llm-kws` to start transcription after wakeup, `llm-llm` is linked to `llm-asr` to process the transcribed text, and `llm-melotts`/`llm-tts` is linked to `llm-llm` to synthesize the response.

### 3.2. CV (Computer Vision) Mode - Object Detection Example

**Description:** This workflow focuses on processing visual information. A common example is capturing an image from a camera and then performing object detection to identify and locate objects within the image. This mode uses `llm-sys` for orchestration.

**Unit Sequence and Data Flow:**

1.  **`llm-camera` (Camera Input):**
    *   **Action:** Captures images/video frames from a configured camera device (e.g., `/dev/video0`).
    *   **Output:** Raw image data stream (e.g., `camera.raw`).
2.  **`llm-yolo` (Object Detection):**
    *   **Input:** Raw image data stream (`camera.raw`) from an instance of `llm-camera` (e.g., `camera.1000`).
    *   **Action:** Processes the image using a YOLO model (e.g., "yolo11n") to detect objects.
    *   **Output:** Object detection results, typically including bounding boxes, object classes, and confidence scores (e.g., in `yolo.yolobox` format).

**Simplified Data Flow:**
Raw Image Data (`llm-camera`) -> Object Detection Results (`llm-yolo`).

**Orchestration:**
The `llm-sys` unit sets up the `llm-camera` unit (specifying camera parameters like resolution) and the `llm-yolo` unit (specifying the model and input source). It then links the output of the `llm-camera` instance to the input of the `llm-yolo` instance to create the processing pipeline. The results from `llm-yolo` can then be published for other applications or units to consume.

## 4. Communication Protocol

Units within the `llm_framework` communicate using **ZeroMQ (ZMQ)**, a high-performance asynchronous messaging library. This allows for a distributed architecture where units can operate independently or collaboratively. Data and control signals are exchanged via ZMQ channels using a standardized **JSON message format**.

**General JSON Message Structure (Requests):**

Control messages sent to units (typically via `llm-sys` or directly for configuration) follow a common structure:

```json
{
    "request_id": "string", // Unique identifier for the request
    "work_id": "string",    // Target unit or system component (e.g., "sys", "kws", "asr.1001")
    "action": "string",     // The operation to perform (e.g., "reset", "setup", "link")
    "object": "string",     // Specifies the context or type of action (e.g., "kws.setup", "work_id")
    "data": {}              // A JSON object containing parameters specific to the action
}
```

**Common Action Messages:**

*   **`reset` (System-Level):**
    Used to reset the state of the system or specific units.
    *Example (System Reset):*
    ```json
    {
        "request_id": "11212155",
        "work_id": "sys",
        "action": "reset"
    }
    ```

*   **`setup` (Unit Configuration):**
    Used to initialize and configure a specific unit with its operational parameters, including models, input/output topics, and other settings.
    *Example (Setting up `llm-kws` unit):*
    ```json
    {
        "request_id": "1",
        "work_id": "kws",
        "action": "setup",
        "object": "kws.setup",
        "data": {
            "model": "sherpa-onnx-kws-zipformer-wenetspeech-3.3M-2024-01-01",
            "response_format": "kws.bool",
            "input": "sys.pcm", // ZMQ topic for input audio
            "enoutput": true,   // Enable output publishing
            "kws": "你好你好"     // Keyword to detect
        }
    }
    ```
    Upon successful setup, a unit instance is created (e.g., `kws.1000`).

*   **`link` (Connecting Unit Inputs/Outputs):**
    Used to establish a data flow pipeline by connecting the output channel of one unit instance to the input channel of another.
    *Example (Linking `asr.1001` to listen to `kws.1000`):*
    ```json
    {
        "request_id": "2",
        "work_id": "asr.1001", // Target unit instance
        "action": "link",
        "object": "work_id",   // Specifies linking by work_id
        "data": "kws.1000"     // Source unit instance whose output will be consumed
    }
    ```

**Data Exchange:**

Units perform their processing and then **publish their results to their configured output ZMQ channels (topics)**. Other units that are configured to use this output (via `link` messages or direct configuration of input topics) can **subscribe** to these channels to receive the data. This enables the internal data flow within the StackFlow architecture, allowing different units to work together. The data itself is typically encapsulated within a JSON structure, often including fields like `created` (timestamp), `data` (payload), `error` (if any), `object`, `request_id`, and `work_id` (of the publishing unit instance).

Example of a response/status message from a unit (can also represent data published):
```json
{
    "created": 1731488371,
    "data": "None", // Or actual payload if it's a data message
    "error": {"code": 0, "message": ""},
    "object": "None",
    "request_id": "3", // Corresponds to the initial request
    "work_id": "asr.1001" // The unit instance providing the response/data
}
```

## 5. Configuration

Configuration for StackFlow units is managed through **JSON files**. These files define both the operational parameters of the units and the parameters for the AI models they use.

**Configuration File Locations:**

The primary directories where configuration files are typically located are:

*   `/opt/m5stack/data/models/` (Primarily for model files and related model configurations)
*   `/opt/m5stack/share/` (Often for unit operation parameter configurations and other shared resources)

**Types of Configuration:**

1.  **Unit Operation Parameter Configuration:**
    *   These settings define how a unit behaves, its ZMQ input/output topics, specific hardware settings (if any), and other functional aspects.
    *   Examples include setting the `input` source (e.g., "sys.pcm"), `response_format` (e.g., "asr.utf-8.stream"), and `enoutput` flags as seen in the `setup` messages in the "Communication Protocol" section.

2.  **Model Operation Parameter Configuration:**
    *   These settings are specific to the AI models being used by a unit. This can include paths to model files, pre/post-processing parameters, inference settings (like `max_token_len` for LLMs or `rule1, rule2, rule3` for ASR), and prompts.
    *   The framework allows for flexibility, as "All units can be fully configured with operational parameters, allowing for model swapping and modification of model parameters within the same data flow processing scenario." This means that by changing these configuration files, users can adapt the behavior of units and the models they use without altering the core unit code.

For instance, the `llm-llm` unit's `setup` data includes `model: "qwen2.5-0.5B-prefill-20e"` and `prompt: "You are a knowledgeable assistant..."`. These would typically be loaded from or defined within such configuration files. Similarly, the `llm-kws` unit's `model: "sherpa-onnx-kws-zipformer-wenetspeech-3.3M-2024-01-01"` and `kws: "你好你好"` are also part of its configuration.

## 6. Glossary (Optional but Recommended)

*   **StackFlow:** An architectural pattern where data flows sequentially through a series of processing stages (units).
*   **Unit:** A modular component within the `llm_framework` responsible for a specific processing task.
*   **ZMQ (ZeroMQ):** A high-performance asynchronous messaging library used for inter-unit communication.
*   **Endpoint:** A ZMQ address (e.g., `tcp://localhost:5555`) used for sending or receiving messages.
*   **Message:** A data packet, typically in JSON format, exchanged between units.
*   **Workflow:** A specific configuration and sequence of connected units designed to achieve a larger task.

---
*This is an initial draft and will be expanded and refined as the project progresses.*
