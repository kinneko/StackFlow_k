# llm-depth_anything (Depth Estimation)

The `llm-depth_anything` unit is a computer vision service within the StackFlow framework, specifically designed for performing monocular depth estimation. It takes an image as input and produces a depth map, which can be visualized as a JPEG image. This unit can receive input directly from a camera unit or via API calls with Base64 encoded image data.

## General API Call Conventions

API calls to `llm-depth_anything` are made via JSON messages.

### Request Structure

A standard request to an `llm-depth_anything` API follows this structure:

```json
{
  "request_id": "client_generated_uuid_or_counter",
  "work_id": "depth_anything_or_depth_anything.instance_id",
  "action": "api_action_name",
  "object": "optional_object_type_string", // If applicable
  "data": { /* API-specific payload */ }     // Or string for some APIs
}
```

-   `request_id` (string, mandatory): A unique client-generated identifier for correlating requests with responses.
-   `work_id` (string, mandatory):
    -   For initial setup or general queries: `"depth_anything"`.
    -   For operations on a specific depth estimation task instance: `"depth_anything.XXXX"` (e.g., `"depth_anything.1007"`), where `XXXX` is the instance ID returned by a successful `setup` call.
-   `action` (string, mandatory): The API action to invoke (e.g., `"setup"`, `"exit"`).
-   `object` (string, optional): Specifies the type or context of the data being sent, if applicable (e.g., `"depth_anything.setup"` for setup data, `"cv.jpeg.base64"` for image data).
-   `data` (object or string, optional): The payload for the API call. Structure varies by API.

### Response Structure

**Success Response:**

Indicates the API call was accepted and processed successfully.

```json
{
  "request_id": "mirrored_from_request",
  "work_id": "depth_anything_or_depth_anything.instance_id", // Mirrored or updated
  "action": "api_action_name",                               // Mirrored from the request
  "created": 1678886400,                                     // Integer: Unix timestamp of response generation
  "code": 0,                                                 // Integer: 0 indicates success
  "message": "OK",                                           // String: Success message
  "data": { /* API-specific data */ }                        // Optional: Payload returned by the API
}
```
-   If an API call results in the creation of a new task instance (e.g., `setup`), the `work_id` in the response will be the new instance ID (e.g., `"depth_anything.1007"`), often also included in the `data` object.

**Error Response:**

Indicates a failure during API call processing.

```json
{
  "request_id": "mirrored_from_request",
  "work_id": "depth_anything_or_depth_anything.instance_id",
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
-   `-5`: Model loading failed (e.g., model files not found, invalid model configuration, engine initialization error).
-   `-6`: Specified task instance (e.g., `"depth_anything.1007"`) does not exist.
-   `-11`: Generic error during model inference or internal processing.
-   `-20`: Error related to input source linking or data subscription.
-   `-21`: Task limit reached (typically, only one instance is supported).
-   `-23`: Base64 decoding error for input image data.
-   `-24`: Image decoding error (e.g., invalid JPEG format).
-   `-25`: Image processing error (e.g., resizing or color conversion failed).

## Data Input and Output

### Input Data

The `llm-depth_anything` unit can receive image data for depth estimation through two main methods, configured via the `input` parameter in the `setup` API:

1.  **Direct Camera Input (via `camera.XXXX` instance ID):**
    -   When `input` is set to a camera instance ID (e.g., `"camera.1001"`), the unit subscribes to the raw video frame output from that camera instance (typically from its ZMQ PUB socket).
    -   **Crucial Requirement**: The camera unit must be configured to output frames at the exact resolution expected by the Depth Anything model. This is specified by `img_w` (width) and `img_h` (height) in the model's configuration JSON (e.g., for `mode_depth-anything-ax630c.json`, this is 384x256). The `llm-depth_anything` unit expects YUYV422 raw frames from the camera and will convert them to RGB/BGR for processing.
    -   **Example Camera Configuration for `depth-anything-ax630c`:**
        ```json
        {
          "request_id": "setup_cam_for_depth",
          "work_id": "camera",
          "action": "setup",
          "object": "camera.setup",
          "data": {
            "input": "/dev/video0", // Or appropriate camera device
            "response_format": "camera.raw", // Provides raw YUYV422 frames
            "frame_width": 384,
            "frame_height": 256,
            "enoutput": false // ZMQ PUB/SUB is used by depth_anything, not API push from camera
          }
        }
        ```

2.  **API Input (via Base64 Encoded JPEG):**
    -   When `input` is set to the `llm-depth_anything` instance's own `work_id` (e.g., `"depth_anything.1007"` itself, or an array containing it), the unit expects to receive image data via an API call to its own `work_id`.
    -   The request to send image data must have:
        -   `work_id`: The specific `depth_anything.XXXX` instance ID.
        -   `action`: This is not a separate "action" name but rather data pushed to the unit's default data handling mechanism.
        -   `object`: `"cv.jpeg.base64"`.
        -   `data`: A string containing the Base64 encoded JPEG image.
    -   The unit will decode the Base64 string, then decode the JPEG image, and process it. The image will be resized to the model's required input dimensions (`img_w` x `img_h`).

### Output Data (Depth Map)

If `enoutput` is `true` (default) in the `setup` call, the `llm-depth_anything` unit pushes the depth estimation result asynchronously. The output is a visual representation of the depth map, encoded as a JPEG image and then Base64 encoded.

The format of the output is determined by the `response_format` parameter in the `setup` API:

1.  **Streamed JPEG Output (`response_format: "jpeg.base64.stream"`)**
    -   The unit sends a stream of JSON objects. Each object represents a processed frame.
    -   **JSON Structure per Frame:**
        ```json
        {
          "request_id": "<original_setup_request_id>", // The request_id of the setup call
          "work_id": "depth_anything.XXXX",          // The specific instance ID
          "object": "jpeg.base64.stream",            // The response format
          "error": {"code": 0, "message": ""},       // Indicates success for this data push
          "data": {
            "index": 0,                              // Integer: Sequence number for this stream segment.
            "delta": "<Base64_encoded_JPEG_depth_map>", // String: Base64 encoded JPEG image of the depth map.
            "finish": true                           // Boolean: Always true as each image is a complete result.
          }
        }
        ```

2.  **Single JPEG Output (`response_format: "jpeg.base64"`)**
    -   The unit sends a single JSON message containing the Base64 encoded JPEG depth map.
    -   **JSON Structure:**
        ```json
        {
          "request_id": "<original_setup_request_id>",
          "work_id": "depth_anything.XXXX",
          "object": "jpeg.base64",                  // The response format
          "error": {"code": 0, "message": ""},       // Indicates success for this data push
          "data": "<Base64_encoded_JPEG_depth_map>"  // String: Base64 encoded JPEG image of the depth map.
        }
        ```

**Content of Output JPEG:**
The output JPEG image is a visualization of the estimated depth map. This is typically a grayscale image where pixel intensity corresponds to depth (e.g., brighter pixels are closer, darker pixels are farther, or vice-versa). The exact visual representation depends on the post-processing implemented in the unit.

## API Reference

### **setup**

Initializes and configures a new depth estimation task instance. Only one task instance is typically supported at a time.

-   **`work_id` in request**: `"depth_anything"`
-   **`work_id` in response (success)**: `"depth_anything.XXXX"` (e.g., `"depth_anything.1007"`)

**Request Parameters:**

| Parameter              | Type          | Required | Default (from Model JSON if applicable) | Description                                                                                                                                                                                             |
|------------------------|---------------|----------|-----------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `object`               | string        | Yes      | N/A                                     | Must be `"depth_anything.setup"`.                                                                                                                                                                     |
| `data.model`           | string        | Yes      | N/A                                     | Name of the model configuration file (without `.json`). E.g., `"depth-anything-ax630c"`.                                                                                                             |
| `data.response_format` | string        | Yes      | N/A                                     | Output format for results if `enoutput` is true. Supported: `"jpeg.base64.stream"`, `"jpeg.base64"`.                                                                                                 |
| `data.input`           | string/array  | Yes      | N/A                                     | Input source(s). Can be a camera instance ID (e.g., `"camera.1001"`) for raw frames, or the unit's own `work_id` (e.g., `"depth_anything.1007"`) for Base64 JPEG input via API calls to that `work_id`. |
| `data.enoutput`        | boolean       | No       | `true`                                  | Enable (`true`) or disable (`false`) asynchronous pushing of depth map results.                                                                                                                         |

**Request Example (Input from Camera):**
```json
{
  "request_id": "setup_depth_cam_001",
  "work_id": "depth_anything",
  "action": "setup",
  "object": "depth_anything.setup",
  "data": {
    "model": "depth-anything-ax630c",
    "response_format": "jpeg.base64.stream",
    "input": "camera.1001", // Assumes camera.1001 is configured for 384x256 raw output
    "enoutput": true
  }
}
```

**Request Example (Input via API as Base64 JPEG):**
```json
{
  "request_id": "setup_depth_api_002",
  "work_id": "depth_anything",
  "action": "setup",
  "object": "depth_anything.setup",
  "data": {
    "model": "depth-anything-ax630c",
    "response_format": "jpeg.base64",
    "input": "depth_anything.1008", // Instance will listen on its own work_id for image data
    "enoutput": true
  }
}
```
*(Note: For API input, the `work_id` in `input` should match the `work_id` that will be returned by this `setup` call, e.g., if this call returns `depth_anything.1008`, then `input` should ideally be configured as `depth_anything.1008` or an array containing it for clarity, though the unit internally subscribes to its own generated `work_id`.)*

**Response (Success):**
```json
{
  "request_id": "setup_depth_cam_001",
  "work_id": "depth_anything.1007", // New task instance ID
  "action": "setup",
  "created": 1678886400,
  "code": 0,
  "message": "OK",
  "data": {
    "work_id": "depth_anything.1007" // Confirms the created instance ID
  }
}
```

**Response (Error - Model Load Fail):**
```json
{
  "request_id": "setup_depth_003",
  "work_id": "depth_anything",
  "action": "setup",
  "created": 1678886401,
  "error": {
    "code": -5,
    "message": "Model loading failed."
  }
}
```

### **exit**

Stops and removes a depth estimation task instance, freeing its resources.

-   **`work_id` in request**: `"depth_anything.XXXX"` (specific task instance ID)

**Request Parameters**: None beyond standard `request_id`, `work_id`, `action`.

**Request Example:**
```json
{
  "request_id": "exit_depth_004",
  "work_id": "depth_anything.1007",
  "action": "exit"
}
```

**Response (Success):**
```json
{
  "request_id": "exit_depth_004",
  "work_id": "depth_anything.1007", // work_id of the exited task
  "action": "exit",
  "created": 1678886402,
  "code": 0,
  "message": "OK"
}
```

**Response (Error - Task Not Found):**
```json
{
  "request_id": "exit_depth_005",
  "work_id": "depth_anything.9999", // Non-existent task
  "action": "exit",
  "created": 1678886403,
  "error": {
    "code": -6,
    "message": "Unit Does Not Exist"
  }
}
```

### **taskinfo**

Retrieves information about active depth estimation tasks. Behavior depends on the `work_id` in the request.

**Case 1: General Unit Information**

-   **`work_id` in request**: `"depth_anything"`
-   **Description**: Returns a list of all active task instance IDs.

**Request Example:**
```json
{
  "request_id": "taskinfo_depth_general_006",
  "work_id": "depth_anything",
  "action": "taskinfo"
}
```

**Response (Success - One Task Active):**
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
    "depth_anything.1007" // List of active instance IDs
  ]
}
```

**Case 2: Specific Task Instance Information**

-   **`work_id` in request**: `"depth_anything.XXXX"` (e.g., `"depth_anything.1007"`)
-   **Description**: Returns runtime parameters and configuration of the specified task instance.

**Request Example:**
```json
{
  "request_id": "taskinfo_depth_specific_007",
  "work_id": "depth_anything.1007",
  "action": "taskinfo"
}
```

**Response (Success):**
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
    "inputs": [ "camera.1001" ] // Array of linked input sources
  }
}
```

> **Note on `work_id` for task instances**: The numeric part of a task instance `work_id` (e.g., `1007` in `depth_anything.1007`) is dynamically assigned by the system upon creation via the `setup` API.

## Relation to Model Configuration Files

The `llm-depth_anything` unit relies on model-specific JSON configuration files (e.g., `mode_depth-anything-ax630c.json`) for its core operational parameters. These files are typically located in a system directory for models (e.g., `projects/llm_framework/main_depth_anything/models/` or a general path like `/opt/m5stack/data/models/`).

Key aspects of these model configuration files:
-   **`mode`**: Name of the mode/model (e.g., `"depth-anything-ax630c"`). This is used in the `setup` API.
-   **`type`**: Type of the unit (e.g., `"cv"` for Computer Vision).
-   **`capabilities`**: Describes the function. For Depth Anything, this should be understood as `"Depth Estimation"`. (Note: The example `mode_depth-anything-ax630c.json` has `"Segmentation"`, which appears to be a misclassification for this model type.)
-   **`input_type`**: Lists expected input data types for API-based input (e.g., `"cv.jpeg.base64"`).
-   **`output_type`**: Lists potential output data types (e.g., `"cv.jpeg.base64"`).
-   **`mode_param` Object**: This object contains crucial parameters for the model and its operation:
    -   `depth_anything_model` (string): The filename of the actual depth estimation model (e.g., `depth_anything.axmodel` for AXERA hardware, or an `.onnx` file for other platforms). This file is located within the model's specific directory (e.g., `depth-anything-ax630c/depth_anything.axmodel`).
    -   `img_h` (integer): The required input image height for the model (e.g., `256`).
    -   `img_w` (integer): The required input image width for the model (e.g., `384`).
    -   `model_type` (string): May indicate a sub-type or specific variant (e.g., `"segment"` in the example JSON, though "depth" or "depth_estimation" would be more fitting). The C++ code uses this to potentially alter post-processing logic.

When a `setup` API call is made with a specific `model` name, the system loads the corresponding JSON configuration file. The parameters within this file, especially `depth_anything_model`, `img_h`, and `img_w`, dictate the model to be loaded and the expected input image dimensions. If input is from a camera unit, that camera must be configured to provide frames matching these dimensions for optimal performance and to avoid errors.