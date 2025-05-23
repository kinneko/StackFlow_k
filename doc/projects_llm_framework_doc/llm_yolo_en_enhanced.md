# llm-yolo

The `llm-yolo` unit provides versatile object detection, segmentation, pose estimation, and oriented bounding box detection services using various YOLO models. It can process images from different sources, including direct image data and camera streams.

## setup

Configures and initializes a YOLO processing task.

Send JSON:

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

**Parameters in `data`:**

*   `request_id` (string, required): A unique identifier for the request.
*   `work_id` (string, required): For initial configuration, use `yolo`. A specific `work_id` (e.g., `yolo.1001`) will be returned upon successful setup.
*   `action` (string, required): The method to call, must be `setup`.
*   `object` (string, required): Data type being transmitted, typically `yolo.setup`.
*   `model` (string, required): The base name of the YOLO model to be used (e.g., `yolo11n`, `yolo11n-seg`, `yolo11n-pose`). The system loads model configuration files (e.g., `mode_yolo11n.json`) based on this name from pre-defined paths (e.g., `projects/llm_framework/main_yolo/`).
*   `response_format` (string, required): Defines the format of the output.
    *   `yolo.box`: Standard YOLO detection output, with coordinates and other values formatted as strings (see Inference section for details).
    *   `yolo.boxV2`: Similar to `yolo.box`, but numerical values (confidence, coordinates) are floats instead of strings.
    *   Appending `.stream` (e.g., `yolo.box.stream`) enables streaming of individual detection objects.
*   `input` (string or array of strings, required): Specifies the input sources and formats.
    *   `yolo.jpg.base64`: Base64 encoded JPEG image data. Can be streaming if object is `yolo.jpg.stream.base64`.
    *   `yolo.png.base64`: Base64 encoded PNG image data (assuming `cv::imdecode` supports it).
    *   `camera.x`: Links to a camera unit (e.g., `camera.0`). Data is expected as raw YUV (YUYV) frames. The unit will call `sys` to get the output port of the camera unit.
    *   Other raw formats like `yolo.yuv.raw`, `yolo.rgb.raw`, `yolo.bgr.raw` might be supported if the object string in an inference call matches internal handlers (`inference_raw_yuv`, `inference_raw_rgb`, `inference_raw_bgr`). These would expect raw byte data, base64 decoded if the object string includes `.base64`.
*   `enoutput` (boolean, required): If `true`, the unit will send its detection results. If `false`, output is suppressed.
*   **Model Configuration Parameters (optional, typically loaded from the model's JSON config file but can be overridden here):**
    *   `model_type` (string, optional): Specifies the type of YOLO model. Default is "detect".
        *   `"detect"`: Standard object detection (bounding boxes).
        *   `"segment"`: Instance segmentation (bounding boxes + masks).
        *   `"pose"`: Pose estimation (bounding boxes + keypoints).
        *   `"obb"`: Oriented Bounding Box detection (bounding boxes + angle).
    *   `confidence_threshold` (float, optional): The minimum probability score for a detection to be considered valid. Example: `0.45`. (Note: model JSON might use `pron_threshold`).
    *   `nms_threshold` (float, optional): The Non-Maximum Suppression threshold used to filter overlapping bounding boxes. Example: `0.45`.
    *   `img_h` (integer, optional): Image height the model expects (e.g., 640).
    *   `img_w` (integer, optional): Image width the model expects (e.g., 640).
    *   `cls_num` (integer, optional): Number of classes the model is trained on (e.g., 80 for COCO).
    *   `point_num` (integer, optional): Number of keypoints for pose models (e.g., 17 for human pose).
    *   `cls_name` (array of strings, optional): List of class names. Loaded from model's JSON.

**Internal Model Configuration (from model's JSON, e.g., `mode_yolo11n.json`):**
*   `yolo_model` (string): Path to the actual `.axmodel` file (e.g., `yolov8n.axmodel`). This path is relative to the model's directory.

**Response JSON (Success):**
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
  "work_id": "yolo.1001" // Unique work_id for this YOLO task
}
```
**Response JSON (Error):**
If `task_count_` (max 1 for YOLO) limit is reached:
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
If model loading fails:
```json
{
  "created": ...,
  "data": "None",
  "error": {"code": -5, "message": "Model loading failed."}, // Or -2 for JSON error, -6 for config error
  "object": "None",
  "request_id": "2",
  "work_id": "yolo"
}
```

## inference

Sends image data to the configured `llm-yolo` task for processing. This action is typically used when not relying on a linked camera input.

**Request JSON (Base64 Encoded JPEG):**
```json
{
  "request_id": "5",
  "work_id": "yolo.1001",
  "action": "inference",
  "object": "yolo.jpg.base64", // Or e.g., yolo.jpg.stream.base64 for streaming chunks
  "data": { // For streaming object type
    "delta": "/9j/4AAQSkZJRgABAQEAYABgAAD/... (base64 encoded image data) ...",
    "index": 0,
    "finish": true
  }
  // If object is not streaming (e.g. "yolo.jpg.base64"), data is just the base64 string:
  // "data": "/9j/4AAQSkZJRgABAQEAYABgAAD/... (base64 encoded image data) ..."
}
```

**Parameters:**
*   `request_id` (string, required): Unique request ID.
*   `work_id` (string, required): The `work_id` of the YOLO task (e.g., `yolo.1001`).
*   `action` (string, required): Must be `inference`.
*   `object` (string, required): Specifies the format of the image data.
    *   `yolo.jpg.base64`: Complete JPEG image, base64 encoded.
    *   `yolo.jpg.stream.base64`: Streamed JPEG image, base64 encoded. `data` field should be an object with `delta`, `index`, `finish`.
    *   Other encodings like PNG might be supported if `cv::imdecode` handles them.
    *   Raw formats like `yolo.yuv.raw.base64`, `yolo.rgb.raw.base64`, `yolo.bgr.raw.base64` could be used if the application sends raw pixel data (after base64 decoding). The raw pixel data size must match `img_w * img_h * channels`.
*   `data` (string or object, required): The image data. If `object` indicates streaming, this is an object with `delta`, `index`, `finish`. Otherwise, it's a string (e.g., base64 encoded image).

**Image Processing Steps (internal):**
1.  Input data is decoded (e.g., base64, then JPEG/PNG via `cv::imdecode`, or raw YUV is converted to RGB/BGR).
2.  The image is preprocessed using `common::get_input_data_letterbox` which resizes while maintaining aspect ratio by padding, to match the model's `img_w` and `img_h`. Color conversion (BGR to RGB) might occur.
3.  The processed image data is fed to the YOLO model (`yolo_->Run()`).
4.  Results are post-processed (`yolo_->Post_Process()`) applying score/confidence thresholds and NMS.
5.  Detected objects are formatted according to `response_format`.

**Response JSON (`yolo.box` or `yolo.boxV2`):**
The output is a JSON array, where each object in the array represents a detected item. If streaming is enabled (`.stream` in `response_format`), individual detection objects might be sent as `delta` in stream messages, with the final message containing all detections or an empty delta and `finish: true`.

**Structure of each detection object (within the `data` array of the response):**

*   **Common fields:**
    *   `class` (string): The detected class name (e.g., "person", "car"). From `cls_name` in config.
    *   `confidence` (string for `yolo.box`, float for `yolo.boxV2`): The confidence score of the detection (e.g., "0.87" or `0.87`).
    *   `bbox` (array): The bounding box coordinates.
        *   For `yolo.box`: `["x1", "y1", "x2", "y2"]` (strings, formatted to 2 decimal places). These are typically absolute pixel values in the original image dimensions after letterbox removal.
        *   For `yolo.boxV2`: `[x, y, width, height]` (floats). These might be relative to the letterboxed image or original image, requires checking specific model runner. Source suggests `obj.rect.x, obj.rect.y, obj.rect.width, obj.rect.height` which are usually absolute after scaling back.

*   **Additional fields based on `model_type`:**
    *   If `model_type` is `"segment"`:
        *   `mask` (array of strings for `yolo.box`, array of floats for `yolo.boxV2`): Mask features or contour points. The exact nature of these "mask_feat" values from `obj.mask_feat` needs further context (e.g., raw mask pixels, RLE, polygon points).
    *   If `model_type` is `"pose"`:
        *   `kps` (array of strings for `yolo.box`, array of floats for `yolo.boxV2`): Keypoint features from `obj.kps_feat`. Typically an array of `[x, y, score]` or `[x, y, visibility, score]` for each keypoint, up to `point_num` keypoints.
    *   If `model_type` is `"obb"` (Oriented Bounding Box):
        *   `angle` (float): The orientation angle of the bounding box.

**Example Response (Non-streaming, `yolo.box`, `model_type: "detect"`):**
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
  "object": "yolo.box", // Matches response_format
  "request_id": "5",
  "work_id": "yolo.1001"
}
```
If an error occurs during inference (e.g., empty data, base64 decode error, model run failure):
```json
{
  "created": ...,
  "data": "None",
  "error": {"code": -11, "message": "Model run failed."}, // Or -23, -24
  "object": "None",
  "request_id": "5",
  "work_id": "yolo.1001"
}
```

## link

Links the output of another unit (e.g., a camera unit) as an input to this YOLO task.

**Request JSON:**
```json
{
  "request_id": "3",
  "work_id": "yolo.1001",
  "action": "link",
  "object": "work_id",
  "data": "camera.0" // The work_id of the unit to link from (e.g., a camera task)
}
```
**Behavior:**
*   If `data` refers to a camera unit (e.g., "camera.0"):
    *   The YOLO unit calls `sys` (`unit_call("sys", "sql_select", data + ".out_port")`) to get the camera's output port (a ZMQ URL).
    *   It then subscribes to this URL. Received data (expected to be raw YUV frames) is processed by `task_camera_data`.
*   If `data` refers to another YOLO unit or a generic image source (e.g., "yolo.some_other_instance_id"):
    *   It subscribes to the `work_id` directly, expecting image data suitable for `task_user_data`.
*   The linked `data` (work_id) is added to the `inputs_` list of the YOLO task.

**Response JSON (Success):**
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

## unlink

Unlinks a previously connected unit.

**Request JSON:**
```json
{
  "request_id": "4",
  "work_id": "yolo.1001",
  "action": "unlink",
  "object": "work_id",
  "data": "camera.0" // The work_id of the unit to unlink
}
```
**Behavior:**
*   Stops the subscription to the specified `data` (work_id of the unit being unlinked) using `stop_subscriber_work_id()` or by stopping the direct ZMQ subscription if it was a camera.
*   Removes `data` from the `inputs_` list of the YOLO task.

**Response JSON (Success):**
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

## exit

Terminates the YOLO task and releases its resources.

Send JSON: (Same as original doc)
```json
{
  "request_id": "7",
  "work_id": "yolo.1001",
  "action": "exit"
}
```
**Behavior:**
*   Calls `llm_task_obj->stop()`, which signals the internal inference thread to finish processing remaining items and join.
*   Stops any active ZMQ subscriptions.
*   Releases the YOLO model (`yolo_->Release()`) and deinitializes the AX engine if this is the last task using it (`_ax_deinit()`).

Response JSON: (Same as original doc)
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

## taskinfo

Retrieves information about running YOLO tasks.

**Request JSON (Get list of all `yolo` tasks):** (Same as original doc)
```json
{
  "request_id": "2",
  "work_id": "yolo",
  "action": "taskinfo"
}
```
Response JSON (List of tasks): (Same as original doc)
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

**Request JSON (Get specific task parameters):** (Original doc had yolo.1003, corrected to yolo.1001 to match example)
```json
{
  "request_id": "2",
  "work_id": "yolo.1001",
  "action": "taskinfo"
}
```
Response JSON (Specific task parameters): (Includes `model_type` and other setup params)
```json
{
  "created": 1737596698,
  "data": {
    "model": "yolo11n",
    "response_format": "yolo.box",
    "enoutput": true,
    "inputs": ["yolo.jpg.base64", "camera.0"], // Reflects current input sources/links
    "model_type": "detect", // And other parameters from setup
    "confidence_threshold": 0.45,
    "nms_threshold": 0.45
  },
  "error": {"code": 0, "message": ""},
  "object": "yolo.taskinfo",
  "request_id": "2",
  "work_id": "yolo.1001"
}
```

> **Note:** `work_id` increases according to the order of unit initialization, not a fixed index value. The `task_count_` for YOLO is typically limited to 1.

## Error Codes (Common)
*   `0`: Success.
*   `-2`: JSON format error in request data.
*   `-5`: Model initialization failed (e.g., `.axmodel` file issue).
*   `-6`: Configuration error during setup (e.g., model's JSON config file missing or corrupt).
*   `-11`: Model run failed during inference.
*   `-20`: Link operation failed.
*   `-21`: Task full (maximum task count, usually 1 for YOLO, reached).
*   `-23`: Base64 decoding error for image data.
*   `-24`: Inference data is empty.
*   `-25`: Stream data index error (if input is streamed).
