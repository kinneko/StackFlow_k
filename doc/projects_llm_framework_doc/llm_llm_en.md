# llm-llm

Large Model Unit, used to provide large model inference services.

## setup

Configure the unit.

Send json:

```json
{
  "request_id": "2",
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

- request_id: Refer to the basic data explanation.
- work_id: For configuration unit, it is `llm`.
- action: The method called is `setup`.
- object: The type of data transmitted is `llm.setup`.
- model: The model used is the Chinese model `qwen2.5-0.5B-prefill-20e`.
- response_format: The result returned is in `llm.utf-8.stream`, utf-8 streaming output.
- input: The input is `llm.utf-8`, representing input from the user.
- enoutput: Whether to enable user result output.
- max_token_len: Maximum output token, this value is limited by the model's maximum limit.
- prompt: The prompt for the model.

Response json:

```json
{
  "created": 1731488402,
  "data": "None",
  "error": {
    "code": 0,
    "message": ""
  },
  "object": "None",
  "request_id": "2",
  "work_id": "llm.1002"
}
```

- created: Message creation time, unix time.
- work_id: The successfully created work_id unit.

## inference

### streaming input

```json
{
    "request_id": "2",
    "work_id": "llm.1003",
    "action": "inference",
    "object": "llm.utf-8.stream",
    "data": {
        "delta": "What's ur name?",
        "index": 0,
        "finish": true
    }
}
```
- object: The data type transmitted is llm.utf-8.stream, indicating a streaming input from the user's UTF-8.
- delta: Segment data of the streaming input.
- index: Index of the segment in the streaming input.
- finish: A flag indicating whether the streaming input has completed.

### non-streaming input

```json
{
    "request_id": "2",
    "work_id": "llm.1003",
    "action": "inference",
    "object": "llm.utf-8",
    "data": "What's ur name?"
}
```

- object: The data type transmitted is llm.utf-8, indicating a non-streaming input from the user's UTF-8.
- data: Data for non-streaming input.

streaming response json:

```json
{"created":1742779468,"data":{"delta":"I am not","finish":false,"index":0},"error":{"code":0,"message":""},"object":"llm.utf-8.stream","request_id":"2","work_id":"llm.1003"}
{"created":1742779469,"data":{"delta":" a person,","finish":false,"index":1},"error":{"code":0,"message":""},"object":"llm.utf-8.stream","request_id":"2","work_id":"llm.1003"}
{"created":1742779469,"data":{"delta":" but I'm","finish":false,"index":2},"error":{"code":0,"message":""},"object":"llm.utf-8.stream","request_id":"2","work_id":"llm.1003"}
{"created":1742779470,"data":{"delta":" here to assist","finish":false,"index":3},"error":{"code":0,"message":""},"object":"llm.utf-8.stream","request_id":"2","work_id":"llm.1003"}
{"created":1742779471,"data":{"delta":" you with any","finish":false,"index":4},"error":{"code":0,"message":""},"object":"llm.utf-8.stream","request_id":"2","work_id":"llm.1003"}
{"created":1742779472,"data":{"delta":" questions or tasks","finish":false,"index":5},"error":{"code":0,"message":""},"object":"llm.utf-8.stream","request_id":"2","work_id":"llm.1003"}
{"created":1742779473,"data":{"delta":" you may have","finish":false,"index":6},"error":{"code":0,"message":""},"object":"llm.utf-8.stream","request_id":"2","work_id":"llm.1003"}
{"created":1742779474,"data":{"delta":". How can","finish":false,"index":7},"error":{"code":0,"message":""},"object":"llm.utf-8.stream","request_id":"2","work_id":"llm.1003"}
{"created":1742779474,"data":{"delta":" I help you","finish":false,"index":8},"error":{"code":0,"message":""},"object":"llm.utf-8.stream","request_id":"2","work_id":"llm.1003"}
{"created":1742779475,"data":{"delta":" today?","finish":false,"index":9},"error":{"code":0,"message":""},"object":"llm.utf-8.stream","request_id":"2","work_id":"llm.1003"}
{"created":1742779475,"data":{"delta":"","finish":true,"index":10},"error":{"code":0,"message":""},"object":"llm.utf-8.stream","request_id":"2","work_id":"llm.1003"}
```

non-streaming response json:

```json
{
    "created": 1742780120,
    "data": "As an artificial intelligence, I don't have a name in the traditional sense. However, I am here to assist you with any questions or information you may need. How can I help you today?",
    "error": {
        "code": 0,
        "message": ""
    },
    "object": "llm.utf-8",
    "request_id": "2",
    "work_id": "llm.1003"
}
```

## link

Link the output of the upper unit.

Send json:

```json
{
  "request_id": "3",
  "work_id": "llm.1002",
  "action": "link",
  "object": "work_id",
  "data": "kws.1000"
}
```

Response json:

```json
{
  "created": 1731488402,
  "data": "None",
  "error": {
    "code": 0,
    "message": ""
  },
  "object": "None",
  "request_id": "3",
  "work_id": "llm.1002"
}
```

error::code of 0 indicates successful execution.

Link the llm and kws units so that when kws issues wake-up data, the llm unit stops the previous unfinished inference
for repeated wake-up functionality.

> **When linking, ensure that kws has been configured and is in working status. Linking can also be done during the
setup phase.**

Example:

```json
{
  "request_id": "2",
  "work_id": "llm",
  "action": "setup",
  "object": "llm.setup",
  "data": {
    "model": "qwen2.5-0.5B-prefill-20e",
    "response_format": "llm.utf-8.stream",
    "input": [
      "llm.utf-8",
      "asr.1001",
      "kws.1000"
    ],
    "enoutput": true,
    "max_token_len": 256,
    "prompt": "You are a knowledgeable assistant capable of answering various questions and providing information."
  }
}
```

## unlink

Unlink.

Send json:

```json
{
  "request_id": "4",
  "work_id": "llm.1002",
  "action": "unlink",
  "object": "work_id",
  "data": "kws.1000"
}
```

Response json:

```json
{
  "created": 1731488402,
  "data": "None",
  "error": {
    "code": 0,
    "message": ""
  },
  "object": "None",
  "request_id": "4",
  "work_id": "llm.1002"
}
```

error::code of 0 indicates successful execution.

## pause

Pause the unit.

Send json:

```json
{
  "request_id": "5",
  "work_id": "llm.1002",
  "action": "pause"
}
```

Response json:

```json
{
  "created": 1731488402,
  "data": "None",
  "error": {
    "code": 0,
    "message": ""
  },
  "object": "None",
  "request_id": "5",
  "work_id": "llm.1002"
}
```

error::code of 0 indicates successful execution.

## work

Resume the unit.

Send json:

```json
{
  "request_id": "6",
  "work_id": "llm.1002",
  "action": "work"
}
```

Response json:

```json
{
  "created": 1731488402,
  "data": "None",
  "error": {
    "code": 0,
    "message": ""
  },
  "object": "None",
  "request_id": "6",
  "work_id": "llm.1002"
}
```

error::code of 0 indicates successful execution.

## exit

Exit the unit.

Send json:

```json
{
  "request_id": "7",
  "work_id": "llm.1002",
  "action": "exit"
}
```

Response json:

```json
{
  "created": 1731488402,
  "data": "None",
  "error": {
    "code": 0,
    "message": ""
  },
  "object": "None",
  "request_id": "7",
  "work_id": "llm.1002"
}
```

error::code of 0 indicates successful execution.

## Task Information

Get task list.

Send JSON:

```json
{
  "request_id": "2",
  "work_id": "llm",
  "action": "taskinfo"
}
```

Response JSON:

```json
{
  "created": 1731652149,
  "data": [
    "llm.1002"
  ],
  "error": {
    "code": 0,
    "message": ""
  },
  "object": "llm.tasklist",
  "request_id": "2",
  "work_id": "llm"
}
```

Get task runtime parameters.

Send JSON:

```json
{
  "request_id": "2",
  "work_id": "llm.1002",
  "action": "taskinfo"
}
```

Response JSON:

```json
{
  "created": 1731652187,
  "data": {
    "enoutput": true,
    "inputs_": [
      "llm.utf-8"
    ],
    "model": "qwen2.5-0.5B-prefill-20e",
    "response_format": "llm.utf-8.stream"
  },
  "error": {
    "code": 0,
    "message": ""
  },
  "object": "llm.taskinfo",
  "request_id": "2",
  "work_id": "llm.1002"
}
```

> **Note: work_id increases according to the initialization registration order of the unit and is not a fixed index
value.**