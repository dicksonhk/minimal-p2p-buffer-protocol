# minimal-p2p-buffer-protocol - a Minimalistic P2P Serverless Communication Protocol

## Overview

This document outlines a communication protocol for serverless peer-to-peer (P2P) systems. The protocol is **designed to be minimalistic and highly scalable**, particularly for scenarios involving a **massive number of concurrent communication channels, each typically involving 0-2 active peers**. Key to its scalability is the focus on P2P interactions and the expectation of **per-tenant/per-channel data partitioning** on the buffering peers.

The protocol aims to be:

-   **Easy to implement** across diverse systems and programming languages.
-   **Requires minimal resources** in terms of computational power, bandwidth, and storage per channel.
-   Adaptable to **flexible buffering/storage backends** (e.g., SQLite, Turso, Cloudflare D1/Durable Objects, Amazon S3, Redis, local filesystem, or simple in-memory stores), especially those that support efficient partitioning.

While P2P chat serves as an intuitive primary use case for illustration, the protocol's core design allows it to also serve as a foundation for simple pub/sub systems or job queues. The design prioritizes **P2P communication**, where one peer can act as a message buffer for another. It supports **message buffering** with Time-to-Live (TTL) and aims for **at-least-once message delivery**, with message deletion from the buffer upon acknowledgment or TTL expiry.

In this protocol, "client" refers to the peer initiating an operation or receiving a pushed message, and "server" refers to its counterpart peer that is currently storing or relaying messages for it.

## Features

-   **Highly Scalable for Concurrent Channels:** Designed to efficiently manage a vast number of independent, small (0-2 peer) communication channels through per-tenant/per-channel data partitioning.
-   **Minimalistic and Efficient:** Low resource usage, high performance, and ease of implementation.
-   **Flexible, Partitioned Storage:** Compatible with various backends, with an emphasis on partitioning data per communication channel or tenant for scalability.
-   **Optimized Two-Peer Channels:** Tailored for direct P2P interaction with one peer acting as a buffer.
-   **Versatile Design:** Suitable for P2P chat, minimalistic pub/sub, or simple job queues.
-   **Message Buffering with TTL:** Peers can store messages; TTL defines max buffering duration.
-   **At-Least-Once Delivery:** Mechanisms ensure messages are delivered at least once.
-   **Acknowledged Deletion:** Buffering peer attempts to delete messages upon `MSG_ACK` or TTL expiry.
-   **Idempotent Operations:** `PUT_MSG` includes an idempotency key.
-   **Connection Liveness:** Essential `PING`/`PONG` mechanism with defined behaviors.
-   **Unified Error Handling:** Dedicated `NACK` packet for reporting errors and connection status signals.
-   **Extensibility:** Convention for non-standard packet types and negotiation via handshake.
-   **Simple and Compact Default Wire Format:** Easy to parse and generate, minimizing bandwidth.

## System Design and Behavior

This section describes the logical operations, expected behaviors of participating peers, and the overall system characteristics.

### Core Operations

The protocol is built around these fundamental operations:

1.  **Liveness Check (PING/PONG Operation):**

    -   A peer sends a `PING` packet. A "simple PING" has no body (just the header). A "timestamped PING" includes a client-side timestamp.
    -   The receiving peer _must_ respond with a `PONG` packet, following specific rules based on the PING received (see "PING/PONG Behavior" below).
    -   _Enables:_ Connection liveness verification, round-trip time estimation.

2.  **Submitting a Message (The "PUT" Operation):**

    -   A client sends new message content, an idempotency key, and a suggested TTL to its peer (server/buffer).
    -   The server acknowledges with the idempotency key, the honored TTL, and a unique `message_id`.
    -   _Enables:_ Buffering, at-least-once submission, message lifecycle via TTL.

3.  **Receiving a Message (The "PUSH" Operation):**

    -   A server proactively sends a `message_id` and its content to a client.
    -   The client acknowledges receipt of the `message_id`.
    -   _Enables:_ Proactive delivery, at-least-once delivery, buffer cleanup on acknowledgment.

4.  **Listing Available Messages (The "LIST" Operation):**

    -   A client requests a list of `message_id`s, optionally with range/limit filters.
    -   The server responds with a list of matching `message_id`s.
    -   _Enables:_ Synchronization, recovery, efficient discovery.

5.  **Retrieving a Specific Message (The "GET" Operation):**
    -   A client requests specific message content using its `message_id`.
    -   The server responds with the `message_id` and its content.
    -   The client acknowledges receipt of the `message_id`'s content.
    -   _Enables:_ On-demand retrieval, at-least-once retrieval, buffer cleanup.

### General Expected Peer Behaviors (Client & Server)

-   **Local Message Copies:**
    -   **Server (Buffering Peer):** MUST keep messages in its buffer for at least their defined TTL or until acknowledged by the recipient (via `MSG_ACK` following a `PUSH_MSG` or `GET_MSG_ACK`).
    -   **Client Peer:** SHOULD maintain its own local copy/history of sent and received messages.
-   **Good Actor Assumption:** The protocol design assumes peers are generally cooperative. Security measures are layered on top.
-   **Handling of Empty Message Data:** Peers SHOULD ignore `PUSH_MSG`, `GET_MSG_ACK`, or `PUT_MSG` packets where the `data` field is empty, unless explicitly meaningful for the application.
-   **Idempotency:**
    -   **Client (`PUT_MSG`):** Must generate a unique `idempotency_key` for each new message submission.
    -   **Server (`PUT_MSG`):** Must use the `idempotency_key` to prevent duplicate message storage from retried `PUT_MSG` requests.
-   **Acknowledgments (`MSG_ACK`, `PUT_MSG_ACK`, `GET_MSG_ACK`, `LIST_MSG_ACK`):** These packets always signify success for the corresponding operation. Errors are reported via `NACK`.
    -   **Client:** Must send `MSG_ACK` upon successful receipt of `PUSH_MSG` or `GET_MSG_ACK`.
    -   **Server:** Must send `PUT_MSG_ACK` upon successful storage of `PUT_MSG`. Must send `GET_MSG_ACK` upon successful retrieval for `GET_MSG`. Must send `LIST_MSG_ACK` upon successful listing for `LIST_MSG`.
-   **Duplicate/Out-of-Order Messages:** Client applications must be prepared to handle duplicate or out-of-order `PUSH_MSG` packets (e.g., by checking `message_id` against local history).
-   **PING/PONG Behavior:**
    -   Both client and server MUST implement the ability to send and respond to PING/PONG packets.
    -   Any peer can send a `PING`.
    -   If a peer receives a **simple PING** (a `PING` packet with no body/timestamp), it **MUST** respond with a **simple PONG** (a `PONG` packet with no body/timestamps).
    -   If a peer receives a **timestamped PING** (a `PING` packet containing a `client_timestamp`), it **MUST** respond with either:
        1.  A **simple PONG** (a `PONG` packet with no body/timestamps), effectively ignoring the received timestamp.
        2.  OR a **full PONG** containing all three timestamp fields: `original_client_timestamp` (mirrored from the PING), `server_receipt_timestamp`, and `server_transmit_timestamp`. The server timestamp fields (`server_receipt_timestamp`, `server_transmit_timestamp`) **MUST** be present in the packet structure if this option is chosen, even if their _values_ are zero (e.g., if the server does not provide actual timing but acknowledges the extended PONG structure).
-   **Processing NACKs and Connection Lifecycle:**
    -   If a peer receives a `NACK` packet (Type `0xFF`) with `original_packet_type = 0xFF` and `error_code = 0x00` (Graceful Disconnect) or `error_code = 0xFF` (Critical Error), it **MUST** prepare to close the connection. After processing this `NACK`, the peer **MUST** close its end of the underlying connection.
    -   For all other `NACK` types (i.e., those reporting recoverable errors for specific operations where `original_packet_type` is not `0xFF`), the connection **MUST NOT** be closed by the protocol itself. The peer should handle the reported error and may continue communication.
    -   For all standard packet types other than the terminating `NACK`s described above, the connection **MUST NOT** be closed by the protocol.
-   **Handling of Unrecognized Packet Types:**
    -   If a peer receives a packet with a `PacketType` whose MSB is 0 (indicating a standard type, values 0-127) but the type is unknown to the receiver (e.g., from a future protocol version it doesn't support), it SHOULD silently drop the packet.
    -   If a peer receives a packet with a `PacketType` whose MSB is 1 and value is not `0xFF` (i.e., values 128-254, indicating a non-standard type) and this type has not been negotiated or is not supported, it SHOULD silently drop the packet.
    -   `NACK` (Type `0xFF`) is a standard packet and must be parsed at least to determine if it's a connection termination signal.
-   **Strict Adherence to Standard Format:** For standard packet types (MSB of `PacketType` is 0, or `PacketType` is `0xFF`), clients and servers MUST strictly follow the documented wire format.

### Server Peer (Buffering Peer) Specific Behaviors

-   **Error Reporting via `NACK`:** For any recoverable error related to a client request (`PUT_MSG`, `GET_MSG`, `LIST_MSG`), the server SHALL respond with a `NACK` packet.
    -   The `NACK` will specify the `Original_PacketType_Errored`.
    -   It will include an `ErrorCode` specific to that original packet type's context (e.g., "Message Not Found" for `GET_MSG`, "Idempotency Key Violation" for `PUT_MSG`).
    -   It MAY include `CorrelationData` (e.g., the `message_id` from the failed `GET_MSG`, or the `idempotency_key` from the failed `PUT_MSG`).
-   **Critical Errors / Rate Limiting / Graceful Disconnect and Connection Closure:**
    -   In the event of critical unrecoverable errors, authentication failures, or persistent rate limit violations, the server **SHALL** send a `NACK` packet with:
        -   `original_packet_type = 0xFF`
        -   `error_code = 0xFF` (Critical Error)
    -   If the server wishes to perform a graceful shutdown of the connection for a client (e.g., maintenance, idle timeout):
        -   It **SHALL** send a `NACK` packet with:
            -   `original_packet_type = 0xFF`
            -   `error_code = 0x00` (Graceful Disconnect)
    -   After sending either of these terminating `NACK` packets, the server **MUST** close its end of the underlying connection.
    -   A server MAY send a non-standard packet (MSB=1 in `PacketType`, values 128-254) containing additional diagnostic information _before_ sending one of these standard terminating `NACK` packets, if such extended error reporting has been negotiated or is understood by the client. The standard terminating `NACK` remains the official and required signal for connection closure.
-   **Dynamic Channel Management (Conceptual):** Servers are expected to manage resources for communication channels on demand, typically tied to the per-tenant/per-channel partitioning.
-   **Tenant/Channel Isolation:** Paramount. Achieved through partitioned storage.
-   **One Channel Per Connection (Recommended):** Simplifies context.
-   **Access Control & Rate Limiting (Implementation Detail):** The server implementation MUST handle access control and SHOULD implement rate limiting. These are considered part of the server's operational responsibilities rather than specific protocol messages.

### Server-Side Storage: Partitioning for Scalability

A critical design aspect for achieving scalability with a massive number of concurrent channels is the **partitioning of storage on a per-tenant or per-channel basis** by the buffering peer.

-   **Why Partitioning is Crucial:** Isolation, scalability, resource management, data lifecycle, concurrency.
-   **Conceptual Partitioning Examples:**
    1.  **Object Storage (e.g., Amazon S3):** Prefix per channel: `s3://bucket/channels/<channel_id>/<message_id>`.
    2.  **Key-Value Stores (e.g., Redis):** Prefixed keys: `<channel_id>:<message_id>`.
    3.  **Relational Databases (e.g., SQLite):** Separate database files per channel: `channel_<channel_id>.db`.
-   **Schema within a Partition (e.g., per-channel SQLite DB):**
    ```sql
    CREATE TABLE messages (
        message_id BIGINT PRIMARY KEY, -- (8 bytes) Snowflake ID (contains creation timestamp)
        expiry     INTEGER,           -- (4 bytes, optional) Unix timestamp (seconds) for message expiry.
        data       BLOB               -- (Variable bytes) The raw message content
    );
    ```

### Protocol Versioning and Handshake

To ensure interoperability, allow for future evolution, and provide flexibility:

1.  **Protocol Versioning:** Implementations SHOULD establish the protocol version during an initial handshake.
2.  **Negotiation for Non-Standard Packet Types:** If peers intend to use non-standard packet types (`PacketType` values 128-254), their usage and meaning MUST be negotiated during the handshake. Standard packet types (0-127 and 255) do not require explicit negotiation beyond establishing a compatible protocol version.
3.  **Message Format Negotiation (Optional):** The handshake _can_ also be used to negotiate alternative message encoding formats if desired.
4.  **Consistency & Handshake Mechanism:** If an alternative encoding is used, the logical structure of packets should remain consistent. The handshake mechanism itself is outside this core protocol definition.

### Out-of-Scope for this Core Protocol

To maintain its minimalistic nature, the following are considered outside the direct scope of this messaging protocol:

-   **Explicit Channel Management APIs:** (e.g., via a separate REST API).
-   **Server Statistics and Monitoring:** (e.g., via dedicated monitoring endpoints).

### General System Design Notes

-   **At-Least-Once Delivery:** Core goal, achieved via retries and acknowledgments.
-   **Error Handling (Message-Level):** Recoverable errors are reported via `NACK`. Connection-terminating conditions are signaled by specific `NACK`s.
-   **Security:** Primarily handled at layers above or below this core messaging protocol.
    -   **Transport Layer Security (TLS):** Strongly recommended.
    -   **Application-Level Encryption:** Recommended for end-to-end encryption of the `data` payload, using established libraries (e.g., OpenSSL, BoringSSL). For P2P, application-level certificates should use non-public CNs if intended for private trust.
    -   Detailed security patterns are outside this document.
-   **Peer Discovery & Connection Management:** Handled externally.
-   **Snowflake IDs:** Recommended for `message_id` for uniqueness and embedded timestamp.

## Default Wire Format Specification

### General Packet Structure

All communication occurs via packets (byte arrays). **It is assumed that the transport layer handles packet framing and size.** This protocol defines the content _within_ that byte array.

1.  **Header (1 byte):** An 8-bit unsigned integer `PacketType`.
2.  **Body (variable size):** Payload, structure depends on `PacketType`. Length of variable fields (`data[]`, `message_id[]`, `correlation_data[]`) is derived from the total packet size (from transport) minus fixed-size fields.

-   **Standard vs. Non-Standard Packet Types:**

    -   If MSB of `PacketType` is 0 (values 0-127), it's a standard data/control packet.
    -   If `PacketType` is `255` (`0xFF`), it's the standard `NACK` packet.
    -   If MSB of `PacketType` is 1 and value is not `255` (i.e., 128-254), it's a non-standard/experimental type.

-   **Standard Packet Type Pairing Convention:**
    -   Most standard packet types (values 0-127, excluding `NACK`) that represent an operation with a direct response will exist in pairs: a request type and an acknowledgment type.
    -   The request packet type value will have its Least Significant Bit (LSB) as `0` (i.e., it will be an even number).
    -   The corresponding acknowledgment packet type value (`_ACK`) will have its LSB as `1` (i.e., it will be an odd number), and its value will be `request_type + 1`.
    -   This convention applies to `MSG`/`MSG_ACK`, `GET_MSG`/`GET_MSG_ACK`, `PUT_MSG`/`PUT_MSG_ACK`, `LIST_MSG`/`LIST_MSG_ACK`.
    -   `PING` (0) and `PONG` (1) also adhere to this pattern, with `PONG` acting as the acknowledgment to `PING`.
    -   The `NACK` packet (255) is a special case and does not follow this pairing rule.
    -   All future standard operational packet types MUST adhere to this LSB pairing convention.

### Packet Types (`PacketType`)

```c
enum PacketType {
    // LSB=0 for request, LSB=1 for acknowledgment
    PING = 0,         // Liveness Check (Request)
    PONG = 1,         // Liveness Check Response (Acknowledgment to PING)

    MSG = 2,          // Push Message Data (Considered a "request" from server to client)
    MSG_ACK = 3,      // Acknowledge Message Receipt (Acknowledgment to MSG or GET_MSG_ACK)
    GET_MSG = 4,      // Request Specific Message (Request)
    GET_MSG_ACK = 5,  // Specific Message Data (Acknowledgment to GET_MSG, always success)
    PUT_MSG = 6,      // Submit New Message (Request)
    PUT_MSG_ACK = 7,  // Acknowledge Message Submission (Acknowledgment to PUT_MSG, always success)
    LIST_MSG = 8,     // Request Message List (Request)
    LIST_MSG_ACK = 9, // Message List Data (Acknowledgment to LIST_MSG, always success)

    // Values 10-127 are reserved for future standard types (must follow LSB pairing).
    // Values 128-254 (MSB set, not 0xFF) are for non-standard/experimental types.

    NACK = 255        // Negative Acknowledgment / Error / Connection Status Signal (Special Case)
};
```

### Detailed Packet Definitions (Standard Types)

(All multi-byte integer fields are Big Endian / Network Byte Order. `message_id` is a 64-bit Snowflake ID. Timestamps are typically 64-bit, e.g., Unix milliseconds.)

---

**1. `PING` (Type 0)**

-   A **simple PING** has a 0-byte body (just the 1-byte header).
-   A **timestamped PING** includes an 8-byte client-generated timestamp.

```c
struct Ping {
    // Optional:
    uint64_t client_timestamp; // (8 bytes) e.g., Unix milliseconds
};
// Body is 0 bytes for simple PING, 8 bytes for timestamped PING.
```

---

**2. `PONG` (Type 1)**

-   Responds to a `PING`.
-   If responding to a **simple PING**, this is a **simple PONG** with a 0-byte body.
-   If responding to a **timestamped PING**, this can be a **simple PONG** (0-byte body) OR a **full PONG** (24-byte body) containing all three timestamp fields. If the server chooses the full PONG structure but doesn't provide actual timing, the server timestamp fields should be present and set to zero.

```c
struct Pong {
    // Fields for a full PONG (24-byte body):
    uint64_t original_client_timestamp; // (8 bytes) Mirrored from PING
    uint64_t server_receipt_timestamp;  // (8 bytes) Server time when PING was received (or 0)
    uint64_t server_transmit_timestamp; // (8 bytes) Server time when PONG is sent (or 0)
};
// Body is 0 bytes for simple PONG, 24 bytes for full PONG.
```

---

**3. `MSG` (Type 2)**

```c
struct Message {
    uint64_t message_id;      // (8 bytes)
    char     data[];          // (remaining bytes)
};
```

---

**4. `MSG_ACK` (Type 3)**

-   Acknowledges successful receipt of `MSG` (Type 2) or `GET_MSG_ACK` (Type 5).

```c
struct MessageAck {
    uint64_t message_id;      // (8 bytes) ID of message being acknowledged.
};
```

---

**5. `GET_MSG` (Type 4)**

```c
struct GetMessage {
    uint64_t message_id;      // (8 bytes)
};
```

---

**6. `GET_MSG_ACK` (Type 5)**

-   Response to `GET_MSG` (Type 4) on success. Errors via `NACK` (Type 255).

```c
struct GetMessageAck {
    uint64_t message_id;      // (8 bytes)
    char     data[];          // (remaining bytes) Message content.
};
```

---

**7. `PUT_MSG` (Type 6)**

```c
struct PutMessage {
    uint32_t idempotency_key; // (4 bytes)
    uint32_t ttl;             // (4 bytes) TTL in seconds
    char     data[];          // (remaining bytes)
};
```

---

**8. `PUT_MSG_ACK` (Type 7)**

-   Response to `PUT_MSG` (Type 6) on success. Errors via `NACK` (Type 255).

```c
struct PutMessageAck {
    uint32_t idempotency_key; // (4 bytes) Mirrored from PUT_MSG
    uint32_t ttl;             // (4 bytes) Honored TTL in seconds
    uint64_t message_id;      // (8 bytes) Server-assigned Snowflake ID.
};
```

---

**9. `LIST_MSG` (Type 8)**

```c
struct ListMessages {
    uint16_t limit;           // (2 bytes)
    uint64_t from;            // (8 bytes)
    uint64_t to;              // (8 bytes)
};
```

---

**10. `LIST_MSG_ACK` (Type 9)**

-   Response to `LIST_MSG` (Type 8) on success. Errors via `NACK` (Type 255).

```
struct ListMessagesAck {
    uint64_t message_id[];    // (n * 8 bytes) Array of message IDs.
};
```

---

**11. `NACK` (Type 255 / `0xFF`)**

-   Reports errors or signals connection status.

```c
struct NackPacket {
    // PacketType header is 0xFF
    uint8_t  original_packet_type; // (1 byte) Type of packet that caused error.
                                   // 0xFF for general error/connection status.
    uint8_t  error_code;           // (1 byte) Specific error code.
                                   // If original_packet_type is 0xFF:
                                   //   0x00: Graceful disconnect signal.
                                   //   0xFF: Critical error / Abort signal.
    char     correlation_data[];   // (Variable bytes) Optional. Data to correlate
                                   // NACK to original request (e.g., idempotency_key,
                                   // message_id).
};
```

-   **Connection Handling with NACK:**
    -   If `original_packet_type == 0xFF` AND (`error_code == 0x00` OR `error_code == 0xFF`), the sender **MUST** close its connection end after sending. The receiver **MUST** close its connection end after processing.
    -   Other `NACK`s report recoverable errors and **MUST NOT** cause protocol-mandated connection closure.

---

## Why This Default Wire Format?

The choice to define this specific custom binary wire format, rather than mandating an existing generic format, is driven by the primary goal of **minimizing the effort required to implement the protocol's encoding/decoding layer**. This allows developers to focus more on the overall system design, application logic, and operational management. While the wire format is a critical piece, it's understood to be only part of the larger puzzle. The aim here is to provide a clear, simple, and efficient default, enabling quicker progress on other core system components.

-   **Comparison to Generic Formats:**

    -   **ASN.1, Protocol Buffers, Thrift, Avro:** Offer rich features and tooling but can introduce complexity, dependencies, and overhead not strictly necessary for this protocol's focused scope.
    -   **JSON, BSON:** While widely supported, text-based parsing (JSON) or even structured binary (BSON with field names) can be less performant and more complex to implement efficiently from scratch compared to this direct binary mapping.

-   **Advantages of this Custom Default Format:**
    -   **Simplicity:** Fixed-offset fields for headers and reliance on the transport layer for overall packet sizing make parsing and generation straightforward.
    -   **Minimal Dependencies:** Can be implemented with basic byte manipulation, without external libraries or code generators for the core encoding.
    -   **Efficiency:** Compact representation with no redundant metadata (like field names) in the packets.
    -   **Flexibility in Payload:** The `data[]` fields are opaque byte arrays. End-users are free to choose any format for the actual message body (e.g., plain text, JSON, custom binary), allowing application-specific flexibility where it matters most.

While this custom binary format is recommended for its alignment with the protocol's design goals, the "Protocol Versioning and Handshake" mechanism acknowledges that implementations might choose alternative encodings if agreed upon by peers, provided the logical operations and data fields are preserved.
