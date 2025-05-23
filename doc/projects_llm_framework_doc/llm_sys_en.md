# llm-sys

The basic service unit of StackFlow, providing external serial and TCP channels and some system function services, while
internally handling port resource allocation and a simple in-memory database.

## External API

- sys.ping: Test if communication with the LLM is possible.
```json
{
    "request_id": "2",
    "work_id": "sys",
    "action": "ping"
}
```
- sys.lsmode: Models that have existed in the system in the past.
```json
{
    "request_id": "2",
    "work_id": "sys",
    "action": "lsmode"
}
```
- sys.bashexec: Execute bash commands.
```json
{
    "request_id": "2",
    "work_id": "sys",
    "action": "bashexec",
    "object": "sys.utf-8.stream",
    "data" : {
        "index": 0,
        "delta": "ls",
        "finish": true
    }
}
```
- sys.hwinfo: Retrieve onboard CPU, memory, and temperature parameters of the LLM.
```json
{
    "request_id": "2",
    "work_id": "sys",
    "action": "hwinfo"
}
```
- sys.uartsetup: Set serial port parameters, effective for a single session.
```json
{
    "request_id": "2",
    "work_id": "sys",
    "action": "uartsetup",
    "object": "sys.uartsetup",
    "data" : {
        "baud": 115200,
        "data_bits": 8,
        "stop_bits": 1,
        "parity": 110
    }
}
```
- sys.reset: Reset the entire LLM framework application.
```json
{
    "request_id": "2",
    "work_id": "sys",
    "action": "reset"
}
```
- sys.reboot: Restart the system.
```json
{
    "request_id": "2",
    "work_id": "sys",
    "action": "reboot"
}
```
- sys.version: Get the version of the LLM framework program.
```json
{
    "request_id": "2",
    "work_id": "sys",
    "action": "version"
}
```

## Internal API

- sql_select: Query key-value pairs in the small in-memory KV database.
- register_unit: Register a unit.
- release_unit: Release a unit.
- sql_set: Set key-value pairs in the small in-memory KV database.
- sql_unset: Delete key-value pairs in the small in-memory KV database.
