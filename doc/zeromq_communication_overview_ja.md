# ZeroMQを利用したプロセス間通信の概要

このドキュメントは、本リポジトリのプログラム群におけるZeroMQを利用したプロセス間通信（IPC）の仕組みについて説明します。

## 1. 基本アーキテクチャ

本システムは、`StackFlow` フレームワークを基盤とし、複数の独立した機能モジュール（以下、Unit）が協調して動作するアーキテクチャを採用しています。これらのUnit間の通信や、外部インターフェースとの連携にZeroMQが積極的に活用されています。

中心的な役割を担うのが **Sys Module** です。Sys Moduleは以下の機能を提供します。

*   **Unit管理**: 各Unitの登録、識別情報（work\_idなど）の発行。
*   **設定情報管理**: システム全体の設定値（ポート番号、ファイルパスなど）の保持と提供。これらは内部のキーバリューストア (`key_sql`) で管理されます。
*   **名前解決/サービスディスカバリ**: Unitが他のUnitと通信する際に必要な接続情報（ZeroMQソケットURLなど）の提供。

各Unitは `StackFlow` クラスのインスタンスとして動作し、以下の特徴を持ちます。

*   **RPCサーバー機能**: 外部からのコマンド実行要求を受け付けるためのRPCサーバーを起動します。
*   **イベント駆動**: 内部にイベントループを持ち、非同期処理を行います。
*   **ZeroMQによる多様な通信**: 他のUnitやSys Moduleと、RPC、PUB/SUB、PUSH/PULLといった複数のZeroMQパターンを使い分けて通信します。

## 2. 通信パターンとプロトコル

主に以下のZeroMQ通信パターンが利用されています。

### 2.1. RPC (Remote Procedure Call)

*   **目的**: Unit間でのコマンド実行、Sys Moduleへの問い合わせ。
*   **パターン**: REQ/REP (リクエスト/リプライ)
*   **ソケットURL形式**:
    *   各UnitのRPCサーバー: `ipc:///tmp/rpc.<unit_name>` (例: `ipc:///tmp/rpc.llm_asr`)
    *   Sys ModuleのRPCサーバー: `ipc:///tmp/rpc.sys` または TCP (`tcp://<host>:<port>`) も設定により利用可能。
*   **主なアクション**:
    *   Unit共通: `setup`, `work`, `exit`, `link`, `unlink`, `taskinfo`
    *   Sys Module: `register_unit`, `release_unit`, `sql_select` (設定取得), `sql_set` (設定変更)
*   **ライブラリ**: `pzmq` ラッパークラス (`ext_components/StackFlow/stackflow/pzmq.hpp`) を介して利用。

### 2.2. PUB/SUB (Publish/Subscribe)

*   **目的**: 非同期のイベント通知、ストリーミングデータ配信。
*   **パターン**: PUB/SUB (パブリッシュ/サブスクライブ)
*   **ソケットURL形式**: Sys Moduleによって動的に割り当てられるか、固定のURL（例: `ipc:///tmp/llm/<port>.sock`, `tcp://<host>:<port>`）が利用されます。UnitはSys Moduleに問い合わせてこれらのURLを取得します。
*   **利用箇所**: `llm_channel_obj` クラス (`ext_components/StackFlow/stackflow/StackFlow.h`) がPUB/SUBチャネルの管理を行います。Unitはこれを利用してメッセージを配信したり、特定のトピックを購読したりします。

### 2.3. PUSH/PULL

*   **目的**: 特定の宛先への確実なメッセージ送信、タスクキューイング。
*   **パターン**: PUSH/PULL
*   **ソケットURL形式**:
    *   UART出力用: `ipc:///tmp/llm/5556.sock` (Sys Moduleの `config_serial_zmq_port` で設定)
    *   その他、Unit間で直接PUSH/PULL通信を行う場合もあり。
*   **利用箇所**: `llm_channel_obj` の `send_raw_to_usr` や `output_to_uart` など。

## 3. メッセージ形式

Unit間で交換されるメッセージは、主に **JSON形式** です。`nlohmann/json` ライブラリがシリアライズ/デシリアライズに使用されます。

一般的なメッセージには以下のキーが含まれます。

*   `request_id`: リクエストを一意に識別するためのID。
*   `work_id`: 処理を実行するUnitのインスタンスを識別するID (例: `llm_asr.1001`)。Sys Moduleによって発行・管理されます。
*   `object`: 実行する操作や対象のオブジェクトを示す文字列。
*   `data`: 操作に必要なパラメータや、処理結果のデータ本体。JSONオブジェクトまたはプリミティブ型。
*   `error`: エラー発生時にエラー情報（`code`, `message`）を格納。
*   `created`: メッセージ作成時刻のタイムスタンプ。

**メッセージ例 (RPCリクエスト/レスポンスの一部):**

```json
{
  "request_id": "unique_request_id_123",
  "work_id": "llm_tts.1002",
  "object": "tts_generate_speech",
  "data": {
    "text": "こんにちは、世界。",
    "speaker_id": "default"
  },
  "created": 1678886400,
  "error": {
    "code": 0,
    "message": ""
  }
}
```

## 4. プログラム間相互関係図

以下は、主要コンポーネントとZeroMQを介した通信関係の概要図です。

```mermaid
graph TD
    subgraph "User/External"
        UART_Interface["UART Interface"]
        TCP_Interface["TCP Interface"]
    end

    subgraph "StackFlow System"
        SysModule["Sys Module (RPC Server, Config DB)"]

        UnitA["Unit A (StackFlow Instance)"]
        UnitB["Unit B (StackFlow Instance)"]
        UnitN["Unit N (StackFlow Instance)"]

        subgraph "ZMQ Sockets (IPC / TCP)"
            ZmqRpcSys["ipc:///tmp/rpc.sys"]
            ZmqPubSub1["ipc/tcp pub/sub channel 1"]
            ZmqPushUart["ipc:///tmp/llm/5556.sock (UART PUSH)"]
            ZmqRpcUnitA["ipc:///tmp/rpc.UnitA"]
            ZmqRpcUnitB["ipc:///tmp/rpc.UnitB"]
            ZmqRpcUnitN["ipc:///tmp/rpc.UnitN"]
        end
    end

    %% Sys Module Interactions
    UnitA -- "RPC: register_unit, sql_select" --> SysModule
    UnitB -- "RPC: register_unit, sql_select" --> SysModule
    UnitN -- "RPC: register_unit, sql_select" --> SysModule

    SysModule -- "Manages/Provides" --> ZmqRpcSys
    SysModule -- "Manages/Provides" --> ZmqPubSub1
    SysModule -- "Manages/Provides" --> ZmqPushUart
    SysModule -- "Manages/Provides" --> ZmqRpcUnitA
    SysModule -- "Manages/Provides" --> ZmqRpcUnitB
    SysModule -- "Manages/Provides" --> ZmqRpcUnitN


    %% Unit to Unit Interactions (via ZMQ Sockets managed by SysModule)
    UnitA -- "ZMQ RPC Call" --> ZmqRpcUnitB
    UnitB -- "ZMQ RPC Response" --> ZmqRpcUnitA

    UnitA -- "ZMQ PUB" --> ZmqPubSub1
    UnitB -- "ZMQ SUB" --> ZmqPubSub1
    UnitN -- "ZMQ SUB" --> ZmqPubSub1

    %% UART Interaction
    SysModule -- "ZMQ PUSH" --> ZmqPushUart
    UnitA -- "ZMQ PUSH (via llm_channel_obj)" --> ZmqPushUart
    ZmqPushUart --> UART_Interface

    %% TCP Interaction (Conceptual)
    TCP_Interface -- "TCP Request" --> SysModule % Or directly to a specific unit if configured
    SysModule -- "Forwards to Unit / ZMQ" --> UnitN % Example
    UnitN -- "Response via ZMQ/SysModule" --> TCP_Interface

    %% Unit RPC Servers
    UnitA -- "Hosts RPC Server" --> ZmqRpcUnitA
    UnitB -- "Hosts RPC Server" --> ZmqRpcUnitB
    UnitN -- "Hosts RPC Server" --> ZmqRpcUnitN

    classDef sys fill:#f9f,stroke:#333,stroke-width:2px;
    classDef unit fill:#bbf,stroke:#333,stroke-width:2px;
    classDef zmq fill:#lightgrey,stroke:#333,stroke-width:1px;
    classDef external fill:#ccf,stroke:#333,stroke-width:2px;

    class SysModule sys;
    class UnitA,UnitB,UnitN unit;
    class ZmqRpcSys,ZmqPubSub1,ZmqPushUart,ZmqRpcUnitA,ZmqRpcUnitB,ZmqRpcUnitN zmq;
    class UART_Interface,TCP_Interface external;
```

**図の説明:**

*   **Sys Module** が中心となり、各 **Unit** のRPCエンドポイント (`ipc:///tmp/rpc.UnitX`) やPUB/SUBチャネル、UARTへのPUSHソケット (`ipc:///tmp/llm/5556.sock`) を管理・提供します。
*   各 **Unit** は起動時にSys Moduleに自身を登録 (`register_unit`) し、他のUnitと通信する際にはSys Moduleに接続情報を問い合わせ (`sql_select`) ます。
*   **Unit** 間は、Sys Moduleから得た情報に基づいて、RPCコールやPUB/SUBメッセージングを行います。
*   **UART Interface** へは、主に `ZmqPushUart` を介してデータがPUSHされます。これはSys Moduleまたは他のUnitから開始されることがあります。
*   **TCP Interface** は、外部ネットワークからの接続点として機能し、Sys Moduleを介して内部のUnitと連携する概念を示しています。具体的な実装はUnitや設定に依存します。

## 5. まとめ

本システムのプロセス間通信は、ZeroMQの柔軟な通信パターンと、Sys Moduleによる集中管理を組み合わせることで、スケーラブルで疎結合なモジュール連携を実現しています。JSON形式のメッセージプロトコルにより、異なるプログラミング言語で書かれたモジュール間の連携も比較的容易になっています (ただし、本リポジトリでは主にC++が使用されています)。
