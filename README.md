# Axon Messaging Protocol - A Minimalist Design for Reliable and Scalable Mediated Client Channels

- **Protocol Version**: 0
- **Status**: `DRAFT`
- **Updated**: 2025-05-25T19:05:00Z

## 1. Overview

This document outlines the **Axon Messaging Protocol** (or Axon Protocol), a communication system designed to leverage the strengths and characteristics of **serverless architectures** for its relaying components. The protocol facilitates communication **Channels** that are conceptually one-to-one between two end-**Clients**. These two **Clients** individually connect to an intermediary component, termed the **Axon Relay**, which acts as a message buffer and relay for their shared **Channel**. The Axon Protocol offers modes for reliable, buffered message delivery with **at-least-once semantics**, as well as options for direct, non-persistent relay. It is optimized for scenarios where an Axon Relay mediates communication for a pair of Clients.

The protocol is **designed to be minimalistic, fostering reliability and high scalability**, particularly for scenarios involving a **massive number of concurrent Channels, each typically involving two Clients and a mediating Axon Relay instance**. Key to its scalability is this focus on small, independent Channels and the expectation of **per-tenant/per-channel data partitioning** within the Axon Relay's backend. Its reliability is further supported by mechanisms for at-least-once delivery (for buffered messages) and acknowledged message handling. This design was initially inspired by and intended for environments like Cloudflare Workers utilizing Durable Objects for the Axon Relay's buffer storage, but it is adaptable to other serverless platforms and partitioned storage solutions.

The Axon Protocol aims to be:

- **Easy to implement** for both Client applications and Axon Relay services.
- **Requires minimal resources** in terms of computational power, bandwidth, and storage per Channel.
- Adaptable to **flexible buffering/storage backends** for the Axon Relay (e.g., SQLite, Turso, Cloudflare D1/Durable Objects, Amazon S3, Redis, local filesystem, or simple in-memory stores), especially those that support efficient partitioning.

While chat between two Clients serves as an intuitive primary use case for illustration, the protocol's core design allows it to also serve as a foundation for simple pub/sub systems or job queues where an Axon Relay facilitates the message exchange.

A **Client** is an end-user application initiating operations or receiving messages. The **Axon Relay** is the intermediary component (e.g., serverless function, Durable Object) that provides the buffering and relaying services for a **Channel**. The Axon Relay is primarily concerned with message handling mechanics (storage, forwarding, TTL, ensuring appropriate delivery semantics through acknowledgments) and is generally not concerned with the content of the message payload itself, which is passed between Clients.

## 2. Document Versioning and Repository Structure

This document (the main `README.md`) describes **Protocol Version 0** of the Axon Messaging Protocol, as indicated at the top.

To facilitate access to different protocol versions and allow implementations to support them, the repository containing this specification should be structured as follows:

- **`/README.md`**: This file, always reflecting the latest stable/recommended version of the Axon Protocol.
- **`/versions/`**: A directory containing subdirectories for each distinct, published protocol version.
  - **`/versions/v0/`**:
    - `README.md`: The complete protocol specification document specific to version 0.
    - Potentially: `examples/`, `test-vectors/` specific to version 0.
  - **`/versions/vX/`**: (For future versions like v1, v2, etc.)
    - `README.md`: Specification for that version.
    - ...

Implementers wishing to support multiple protocol versions can refer to the specific `README.md` within each versioned directory.

## 3. Features

- **Highly Scalable for Concurrent Channels:** Designed to efficiently manage a vast number of independent, small (typically two-Client) Channels, facilitated by Axon Relays using per-tenant/per-channel data partitioning. Suitable for serverless architectures.
- **Optimized Two-Client Channels via Axon Relay:** Tailored for interactions between two Clients mediated by an Axon Relay acting as a buffer/relay.
- **Minimalistic and Efficient:** Low resource usage, high performance, and ease of implementation for Clients and Axon Relays.
- **Flexible, Partitioned Storage:** Compatible with various backends for Axon Relays, with an emphasis on partitioning data per Channel or tenant for scalability.
- **Versatile Design:** Suitable for chat-like applications, minimalistic pub/sub, or simple job queues.
- **Multiple Delivery Modes:**
  - **Buffered Delivery (`PUT_MSG`):** For messages requiring at-least-once delivery semantics with server-side buffering and TTL.
  - **Acknowledged Direct Relay (`DIRECT_SEND`):** Optional feature for low-latency, non-persistent relay with acknowledgment of the relay attempt.
  - **Unacknowledged Direct Relay (`FAST_SEND`):** Optional feature for very low-latency, non-persistent, "fire-and-forget" relay.
- **Message Buffering with TTL (for `PUT_MSG`):** Axon Relays can store messages; TTL defines max buffering duration.
- **At-Least-Once Delivery (for `PUT_MSG`):** Mechanisms ensure buffered messages are delivered at least once between Clients and their Axon Relay.
- **Acknowledged Deletion:** Axon Relays attempt to delete messages from their buffer upon `MSG_ACK` from a recipient Client (for buffered messages, identified by a non-zero `message_id`) or TTL expiry.
- **Idempotent Operations:** `PUT_MSG` and `DIRECT_SEND` include an idempotency key.
- **Connection Liveness:** Essential `PING`/`PONG` mechanism between Clients and Axon Relays with defined behaviors.
- **Unified Error Handling:** Dedicated `NACK` packet for reporting errors and connection status signals with standardized error codes.
- **Extensibility:** Convention for non-standard packet types and negotiation via handshake for optional features and custom extensions.
- **Simple and Compact Wire Format:** Easy to parse and generate; minimizing bandwidth, code complexity and resource usage.

## 4. System Design and Behavior

This section describes the logical operations, expected behaviors of participating Clients and Axon Relays, and the overall system characteristics for a communication Channel.

### 4.1. Core Operations

The protocol is built around these fundamental operations between a Client and an Axon Relay:

#### 4.1.1. Liveness Check (PING/PONG Operation)

- A Client or Axon Relay sends a `PING` packet.
- The receiving party _must_ respond with a `PONG` packet.
- _Enables:_ Connection liveness verification, round-trip time estimation.

#### 4.1.2. Submitting a Buffered Message (The "PUT" Operation)

- A Client sends `PUT_MSG` with message content, an `idempotency_key`, and a suggested `ttl > 0` to an Axon Relay.
- The Axon Relay acknowledges with `PUT_MSG_ACK`, including the honored `ttl` and a unique `message_id`.
- _Enables:_ Persistent buffering on the Axon Relay, at-least-once submission from Client to Relay buffer, message lifecycle via TTL.

#### 4.1.3. Sending a Direct Acknowledged Message (The "DIRECT_SEND" Operation - Optional Feature)

- A Client sends `DIRECT_SEND` with message content and an `idempotency_key` to an Axon Relay.
- The Axon Relay attempts immediate, non-persistent relay to the other Client and acknowledges the _attempt_ with `DIRECT_SEND_ACK` (mirroring `idempotency_key`) or a `NACK`.
- _Enables:_ Low-latency, non-persistent relay with acknowledgment of relay initiation.

#### 4.1.4. Sending a Direct Fire-and-Forget Message (The "FAST_SEND" Operation - Optional Feature)

- A Client sends `FAST_SEND` with message content to an Axon Relay.
- No protocol-level acknowledgment (`FAST_SEND_ACK` is a reserved no-op type) is sent by the Axon Relay. Delivery is best-effort and non-persistent.
- _Enables:_ Very low-latency, non-persistent, unacknowledged relay.

#### 4.1.5. Receiving a Message (The "PUSH" Operation via `MSG`)

- An Axon Relay proactively sends an `MSG` packet to a connected Client.
  - If originating from a buffered `PUT_MSG`, the `MSG` packet contains a non-zero `message_id` and content.
  - If originating from a relayed `DIRECT_SEND` or `FAST_SEND`, the `MSG` packet **MUST** contain a `message_id` of `0` and the relayed content.
- For `MSG` packets with a non-zero `message_id` (i.e., from buffered messages), the Client acknowledges receipt with `MSG_ACK`.
- For `MSG` packets with `message_id = 0` (i.e., from `DIRECT_SEND` or `FAST_SEND`), the Client **MUST NOT** send an `MSG_ACK`. Sending an `MSG_ACK` with `message_id = 0` is a protocol violation.
- _Enables:_ Proactive delivery from Axon Relay to Client. For buffered messages (non-zero `message_id`), `MSG_ACK` contributes to at-least-once delivery and enables Axon Relay buffer cleanup. For direct/fast messages (`message_id = 0`), it's a simple push.

#### 4.1.6. Listing Available Buffered Messages (The "LIST" Operation)

- A Client requests a list of `message_id`s for buffered messages from the Axon Relay. The request includes `limit`, `from` (exclusive cursor), and `to` (exclusive cursor) parameters.
- The Axon Relay responds with a `LIST_MSG_ACK` packet containing matching `message_id`s.
- _Enables:_ Efficient, paginated synchronization and recovery of buffered message history.

#### 4.1.7. Retrieving a Specific Buffered Message (The "GET" Operation)

- A Client requests specific content of a buffered message from the Axon Relay using its `message_id`.
- The Axon Relay responds with the `message_id` and its content in a `GET_MSG_ACK` packet.
- The Client acknowledges receipt of the content by sending a `MSG_ACK`.
- _Enables:_ On-demand retrieval of buffered messages, contributes to at-least-once retrieval, Axon Relay buffer cleanup.

### 4.2. Message Time-to-Live (TTL) and Buffered Delivery (`PUT_MSG` with TTL > 0)

The `PUT_MSG` operation is used for submitting messages intended for **reliable, buffered delivery** by the Axon Relay.

#### 4.2.1. **Client Submission**

The Client includes a `ttl` field (in seconds, which **MUST** be `> 0` for this packet type) and an `idempotency_key` in the `PUT_MSG` packet.

#### 4.2.2. **Axon Relay Authority**

The Axon Relay MAY adjust the suggested TTL based on its own policies (e.g., maximum or minimum allowable TTL).

#### 4.2.3. **Honored TTL in Acknowledgment**

The Axon Relay MUST include the actual (honored) TTL it has applied (which will also be `> 0`) in the `PUT_MSG_ACK` packet sent back to the Client. This represents the duration for which the Axon Relay commits to attempting to buffer the message if not acknowledged earlier by the recipient Client.

#### 4.2.4. **Persistence**

The Axon Relay WILL persist the message according to its storage backend capabilities for the honored TTL duration or until acknowledged as received by the recipient Client (via an `MSG_ACK` after the Relay pushes the message via `MSG`).

#### 4.2.5. **Client Awareness**

Clients should use the `ttl` value from `PUT_MSG_ACK` as the authoritative buffering duration for that message on the Axon Relay.

### 4.3. Direct Message Delivery (Non-Persistent - Optional Features)

The Axon Protocol supports two modes for direct, non-persistent message relaying. These are **optional features**, and their support **SHOULD** be negotiated during the handshake (see section 4.7.4). Messages sent via these modes **MUST NOT** be knowingly persisted to durable storage or have their content logged by the Axon Relay. They should primarily reside in volatile memory on the Relay during the relay attempt.

- **Client Trust and Encryption:** Clients **SHOULD NOT** blindly trust an Axon Relay's adherence to the no-persistence rule. For sensitive data, end-to-end encryption performed between Clients remains the strongest privacy guarantee, regardless of the delivery mode.
- **Axon Relay Support:** If an Axon Relay receives a packet for a direct delivery mode (`DIRECT_SEND`, `FAST_SEND`) that it does not support (either because it was not negotiated or is generally unsupported by that Relay implementation), it **SHOULD** respond with a `NACK` (e.g., `error_code = ErrorCode::UnsupportedOptionalFeature` or `ErrorCode::UnsupportedStandardPacketType`) and then **SHOULD** terminate the connection. It **MUST NOT** treat such a request as a standard buffered `PUT_MSG`.

#### 4.3.1. Acknowledged Direct Delivery (`DIRECT_SEND` / `DIRECT_SEND_ACK`)

This mode provides an acknowledgment from the Axon Relay about the _attempt_ to relay the message.

- **Operation:**
  1. Client sends `DIRECT_SEND` containing an `idempotency_key` (must be non-zero for correlation) and `data`.
  2. The Axon Relay attempts to relay the `data` to the other Client in the Channel. This delivery attempt should generally be prioritized.
  3. The Axon Relay responds to the sending Client with `DIRECT_SEND_ACK` (mirroring `idempotency_key`) or a `NACK`.

- **`DIRECT_SEND_ACK` Semantics:**
  - Indicates the Axon Relay has accepted the request and initiated the delivery attempt to the recipient Client. It **DOES NOT GUARANTEE** that the recipient Client has received or processed the message.
  - The Axon Relay should send the `DIRECT_SEND_ACK` after it has passed the message to its transport layer for delivery to the recipient Client or can otherwise reasonably assume the delivery process has begun.

- **`NACK` Semantics (for `DIRECT_SEND`):** The Axon Relay **MUST** reply with a `NACK` if the delivery attempt could not even be initiated (e.g., the recipient Client's connection was already known to be closed _before_ any delivery attempt was made, or an immediate internal error prevented the attempt). The `NACK` should include the `idempotency_key` from the `DIRECT_SEND` in its `correlation_data`.

- **`message_id` in relayed `MSG`:** The Axon Relay **MUST** use a `message_id` of `0` (all-zero bits) for the `MSG` packet pushed to the recipient Client.

#### 4.3.2. Unacknowledged "Fire-and-Forget" Direct Delivery (`FAST_SEND`)

This mode requires minimal resources and provide best performance for real-time, best-effort, unacknowledged relay where the sending Client does not require a protocol-level acknowledgment of the relay attempt.

- **Operation:**
  1. Client sends `FAST_SEND`. The packet body, immediately following the `PacketType` header, is the raw message `data`.
  2. The Axon Relay attempts to relay the `data` to the other Client in the Channel.
  3. When an Axon Relay relays a `FAST_SEND` message to the recipient Client as an `MSG` packet, it **MUST** use a `message_id` of `0` (all-zero bits) in that `MSG` packet.
  4. The Axon Relay **MUST NOT** send a `FAST_SEND_ACK` (Type 13); this packet type is reserved as a no-operation.

- **`NACK` Behavior (Optional by Relay for `FAST_SEND`):**
  - Sending a `NACK` in response to a `FAST_SEND` (e.g., if the recipient is known to be disconnected before attempt) is **OPTIONAL** for the Axon Relay. An Axon Relay might choose not to send `NACK`s for `FAST_SEND` failures to minimize overhead.
  - If a `NACK` is sent, its `original_packet_type` will be `PacketType::FastSend`, and its `correlation_data` field **MUST** be empty.

- **Client Expectations for `FAST_SEND`:**
  - Clients using this mode accept that there is no guarantee of relay attempt or delivery.
  - Clients **MUST** still be prepared to handle a `NACK` if the Relay chooses to send one (e.g., for persistent issues) and SHOULD react appropriately (e.g., pause sending, log the event).

> **Alternative Protocols**
>
> If an application's primary or exclusive mode of communication is unacknowledged, fire-and-forget direct delivery, developers should evaluate whether a simpler custom WebSocket relay or a protocol specifically designed for such high-throughput, low-guarantee streaming (e.g., UDP socket, WebRTC data channels) might be more appropriate.
>
> The Axon Protocol can still be used to exchange the necessary information to establish such alternative connections.

### 4.4. General Expected Behaviors (Client & Axon Relay)

#### 4.4.1. **Local Message Copies**

- **Axon Relay:** SHOULD keep messages submitted via `PUT_MSG` in its buffer for at least their defined (and acknowledged) TTL or until acknowledged by the recipient Client.
- **Client:** SHOULD maintain its own local copy/history of sent and received messages, they SHOULD keep messages submitted via `PUT_MSG` in its buffer for at least the server defined TTL.

#### 4.4.2. **Good Actor Assumption**

The protocol design assumes Clients and Axon Relays are generally cooperative. Security measures are layered on top.

Clients MUST aim for a low error-rate and implement rate limiting.

Relays SHOULD isolate the buffer and use aggressive rate limiting or banning policy to protect overall system stability and good actors.

#### 4.4.3. **Handling of Empty Message Data**

All parties SHOULD ignore `MSG`, `GET_MSG_ACK`, `PUT_MSG`, `DIRECT_SEND`, or `FAST_SEND` packets where the `data` field is effectively empty (0-length), unless an empty payload is explicitly meaningful for the application.

#### 4.4.4. **Idempotency**

- **Client (`PUT_MSG`, `DIRECT_SEND`):** Must generate a unique `idempotency_key` for each `PUT_MSG` (for buffered messages) and `DIRECT_SEND` (for acknowledged direct messages).
- **Axon Relay (`PUT_MSG`):** Must use `idempotency_key` to prevent duplicate storage of buffered messages.
- **Axon Relay (`DIRECT_SEND`):** The `idempotency_key` is primarily for correlating the `DIRECT_SEND_ACK` or a `NACK` if the relay attempt fails at initiation. The Relay MAY use it to de-duplicate rapid relay attempts if an ACK was lost and the Client retries.

#### 4.4.5. **Acknowledgments (`MSG_ACK`, `PUT_MSG_ACK`, `DIRECT_SEND_ACK`, `GET_MSG_ACK`, `LIST_MSG_ACK`)**

- These packets _always_ signify that the corresponding request operation was successfully processed by the receiver according to the protocol rules (noting specific semantics for `DIRECT_SEND_ACK`). `FAST_SEND_ACK` is a reserved no-op and is not sent.
- A "successful process" for `LIST_MSG` includes returning an empty list if no messages match the criteria.
- Actual operational errors encountered by the Axon Relay while attempting to fulfill a Client's request are reported back to the Client using a `NACK` packet.
- **Client:** Must send `MSG_ACK` upon successful receipt of:
  - An `MSG` packet from the Axon Relay, **only if** the `MSG` packet contains a non-zero `message_id`.
  - A `GET_MSG_ACK` packet from the Axon Relay (which always refers to a buffered message with a non-zero `message_id`).
  - Sending `MSG_ACK` for an `MSG` packet that contained `message_id = 0` is a protocol violation by the Client.
- **Axon Relay:**
  - Must send `PUT_MSG_ACK` upon successful storage of `PUT_MSG`.
  - Must send `DIRECT_SEND_ACK` upon successful initiation of a `DIRECT_SEND` relay attempt.
  - Must send `GET_MSG_ACK` upon successful retrieval for `GET_MSG`.
  - Must send `LIST_MSG_ACK` upon successful processing of a `LIST_MSG` request.
  - Must NOT send `FAST_SEND_ACK`.

#### 4.4.6. **Duplicate/Out-of-Order Messages**

Client applications must be prepared to handle duplicate or out-of-order `MSG` packets from the Axon Relay by checking `message_id` against local history.

`LIST_MSG_ACK` will inherently include outgoig messages, the other Client would not be able to receive the buffered message if a client sends a `MSG_ACK` packet for message they sent.

#### 4.4.7. **PING/PONG Behavior**

- Both Client and Axon Relay MUST implement the ability to send and respond to PING/PONG packets.
- Any party (Client or Axon Relay) can send a `PING`.
- If a party receives a **simple PING** (a `PING` packet with no body/timestamp), it **MUST** respond with a **simple PONG** (a `PONG` packet with no body/timestamps).
- If a party receives a **timestamped PING** (a `PING` packet containing a `sender_timestamp`), it **MUST** respond with either:
  1. A **simple PONG** (a `PONG` packet with no body/timestamps), effectively ignoring the received timestamp.
  2. OR a **full PONG** containing all three timestamp fields: `original_sender_timestamp` (mirrored from the PING), `receiver_receipt_timestamp` (its own receipt time), and `receiver_transmit_timestamp` (its own transmit time). These latter two timestamp fields **MUST** be present in the packet structure if this option is chosen, even if their _values_ are zero (e.g., if the receiver does not provide actual timing but acknowledges the extended PONG structure).

#### 4.4.8. **Processing NACKs and Connection Lifecycle**

- If a party receives a `NACK` packet (Type `0xFF`), it MUST process the `original_packet_type` and `error_code` to determine the nature of the error and the appropriate action, as guided by the `ErrorCode` categories (see 6.3.15).
- If the `error_code` within a `NACK` is not recognized by the receiver for the given `original_packet_type`, it **SHOULD** be treated as equivalent to receiving `ErrorCode::CriticalErrorAbort (0xFF)`, leading to connection termination.
- **Connection Terminating NACKs:**
  - A `NACK` with `original_packet_type = PacketType::Nack (0xFF)` AND (`error_code == ErrorCode::GracefulDisconnect (0x00)` OR `error_code == ErrorCode::CriticalErrorAbort (0xFF)`) signals a mandatory connection closure. Both sender and receiver MUST close the connection.
  - For any `NACK` with an `error_code` in the range `0xE0-0xFF` (Critical Errors), both the sender and receiver of the `NACK` **SHOULD** close the connection. The sender closes after attempting to send the NACK; the receiver closes after receiving it.
- **Non-Terminating NACKs (by default):** For `NACK`s reporting errors typically in the range `0x01-0xDF` (unless it's `GracefulDisconnect`), the connection **MUST NOT** be closed by the protocol itself solely due to receiving the `NACK`.

#### 4.4.9. **Handling of Invalid, Forbidden, or Unnegotiated Packets**

- **Malformed Standard Packets:**

  If a party receives a standard packet type that it recognizes but whose body is structurally invalid (e.g., incorrect length for fixed fields) according to this specification, it **SHOULD** attempt to send a `NACK` (`original_packet_type` = offending type, `error_code = ErrorCode::MalformedPacket (0xF0)`) and then **SHOULD** terminate the connection.

- **Forbidden Standard Packets:**

  If a party receives a standard packet explicitly forbidden in the current context or by that party's role (e.g., an Axon Relay receiving `FAST_SEND_ACK`, or a Client sending `MSG_ACK` for `message_id = 0`), it **SHOULD** attempt to send a `NACK` (`original_packet_type` = offending type, `error_code = ErrorCode::ProtocolViolation (0xF1)`) and then **SHOULD** terminate the connection.

- **Unnegotiated/Unsupported Non-Standard Packets (MSB=1):**

  If a party receives a packet with `PacketType` MSB=1 (values 128-254) and this type has not been negotiated or is not supported, it **SHOULD** attempt to send a `NACK` (`original_packet_type` = offending type, `error_code = ErrorCode::UnsupportedNonStandardPacketType (0xF3)`) and then **SHOULD** terminate the connection. This is especially recommended for public Axon Relay services.

- **Unrecognized/Unsupported Standard Packets (Violating Version):**

  If a party receives a standard packet type not defined in the protocol version associated with the endpoint, it **SHOULD** attempt to send a `NACK` (`original_packet_type` = offending type, `error_code = ErrorCode::UnsupportedStandardPacketType (0xF2)`) in the transport layer. The connection **SHOULD NOT** be terminated by the receiver solely for a single instance; persistent sending may lead to termination by other mechanisms (e.g. rate limiting).

- **Protocol Version or Format Mismatch During Handshake:**

  If a protocol version or message format mismatch is detected during handshake, the Axon Relay **MUST** send a `NACK` (`original_packet_type = PacketType::Nack (0xFF)`, `error_code = ErrorCode::ProtocolVersionMismatch (0x01)`) in the transport layer and then **MUST** close the connection. The Client **MUST** also close the connection upon receiving this error.

#### 4.4.10. **Strict Adherence to Standard Format**

For standard packet types (MSB of `PacketType` is 0, or `PacketType` is `0xFF`), Clients and Axon Relays MUST strictly follow the documented wire format

### 4.5. Axon Relay Specific Behaviors

#### 4.5.1. **Error Reporting via `NACK`**

- If an Axon Relay encounters a recoverable error while processing a Client's request (`PUT_MSG`, `DIRECT_SEND`, `GET_MSG`, `LIST_MSG`), it SHALL respond with a `NACK`.
- If a request contains structurally valid parameters that are logically invalid or violate Relay policy in a way that indicates a Client error/bug (e.g., an impossible range or cursors in `LIST_MSG` that cannot be resolved to an empty set, a value known to be out of bounds for a field), it **SHOULD** respond with `NACK` and `error_code = ErrorCode::InvalidParametersCritical (0xF4)`, then **SHOULD** terminate the connection.
- For less severe policy violations (e.g., `ttl` for `PUT_MSG` outside preferred but still acceptable bounds resulting in adjustment), an `error_code` from the `0x20-0x9F` (e.g., `ErrorCode::InvalidTtlForPut`) or `0xA0-0xDF` ranges should be used, and the connection typically remains open. The honored TTL is communicated via `PUT_MSG_ACK`.

#### 4.5.2. **Internal Errors/Assertion Failures**

If an Axon Relay encounters an unrecoverable internal error or an assertion failure, it **SHOULD** attempt to send a `NACK` with `original_packet_type = PacketType::Nack (0xFF)` and `error_code = ErrorCode::CriticalErrorAbort (0xFF)` or `ErrorCode::UnspecifiedRelayInternalError (0xFE)` before terminating the connection

#### 4.5.3. **Critical Errors / Rate Limiting / Graceful Disconnect and Connection Closure**

In cases of critical errors not covered above, or for enforcing rate limits that necessitate connection closure, or for planned graceful disconnects, the Axon Relay will use appropriate `NACK` packets with `original_packet_type = PacketType::Nack (0xFF)` and relevant `error_code` values (`0x00`, `0xF7`, `0xFF`), then close the connection

#### 4.5.4. **Dynamic Channel Resource Management (Conceptual)**

Axon Relay instances are expected to manage resources for Channels on demand

#### 4.5.5. **Tenant/Channel Isolation**

Paramount. Achieved through partitioned storage and instance isolation

#### 4.5.6. **One Channel Per Logical Axon Relay Instance (Recommended)**

Simplifies state management. A "logical instance" refers to the entity managing a single Channel's state and buffer, e.g., a Cloudflare Durable Object. A single physical serverless function execution might host multiple such logical instances if the platform supports it

#### 4.5.7. **Access Control & Rate Limiting (Implementation Detail)**

The Axon Relay implementation MUST handle access control and SHOULD implement rate limiting

#### 4.5.8. **Note on Error Logging and Rate Limiting**

Axon Relay implementations SHOULD log `NACK`-triggering events. The type and rate of errors (especially `0xA0-0xDF` warnings and `0xF0-0xFF` critical client errors) from a specific Client can be used as factors in dynamic rate-limiting decisions or for identifying clients to block/disconnect

### 4.6. Axon Relay Storage: Partitioning for Scalability

A critical design aspect for achieving scalability with a massive number of concurrent Channels is the **partitioning of storage on a per-tenant or per-Channel basis** by the Axon Relay component.

#### 4.6.1. **Why Partitioning is Crucial**

Isolation, scalability, resource management, data lifecycle, concurrency

#### 4.6.2. **Conceptual Partitioning Examples**

1. **Object Storage (e.g., Amazon S3):** Prefix per Channel: `s3://bucket/channels/<channel_id>/<message_id>`.
2. **Key-Value Stores (e.g., Redis):** Prefixed keys: `<channel_id>:<message_id>`.
3. **Relational Databases (e.g., SQLite):** Separate database files per Channel: `channel_<channel_id>.db`.
4. **Cloudflare Durable Objects:** Each Durable Object instance inherently acts as a partition for a single Channel.

#### 4.6.3. **Logical Schema within a Partition**

```sql
CREATE TABLE messages (
    message_id BIGINT PRIMARY KEY, -- (8 bytes) Snowflake ID (non-zero for buffered messages)
    expiry     INTEGER,           -- (4 bytes, optional) Unix timestamp for message expiry. This field is only populated and meaningful for buffered messages with a TTL > 0.
    data       BLOB               -- (Variable bytes) The raw message content
);
```

#### 4.6.4. **Note on Non-Persistent Messages**

Messages sent via `DIRECT_SEND` or `FAST_SEND` (effectively TTL `0`, relayed with `message_id = 0`) are not intended for persistent storage within this backend by the Axon Relay

### 4.7. Protocol Versioning, Handshake, and Backwards Compatibility

#### 4.7.1. Protocol Version Identification

- This document describes **Protocol Version 0** of the Axon Messaging Protocol.
- Future versions will have unique, incrementing non-negative integer version numbers.

#### 4.7.2. Handshake for Version Negotiation

- The protocol version and message format are chosen by the Client **before** initiating a handshake, typically by selecting the appropriate Axon Relay endpoint (e.g., via DNS subdomain as described in section 5.1).
- The handshake **MUST explicitly include** the protocol version (and, if applicable, message format) being used. Both Client and Axon Relay **MUST** check these fields during handshake.
- Both Client and Axon Relay **MUST** use the protocol version and message format associated with the chosen endpoint from the outset.
- If either party detects a mismatch in protocol version or message format during the handshake, this is an immediate protocol violation.
- Upon detecting such a mismatch, the Axon Relay **MUST** send a `NACK` (`original_packet_type = PacketType::Nack (0xFF)`, `error_code = ErrorCode::ProtocolVersionMismatch (0x01)`) in the transport layer, and then **MUST** close the connection. The Client **MUST** also close the connection upon receiving this error.

#### 4.7.3. Handling Different Versions

- If a version or format mismatch is detected during handshake, the Axon Relay **MUST** send a `NACK` (`original_packet_type = PacketType::Nack (0xFF)`, `error_code = ErrorCode::ProtocolVersionMismatch (0x01)`) in the transport layer and then **MUST** close the connection. The Client **MUST** also close the connection upon receiving this error.
- Only the logic for the currently supported version should exist in the codebase.
- Implementations **MUST NOT** retain code paths for compatibility with older or future protocol versions. Retaining such compatibility code would violate the protocol's minimalistic design and make the codebase harder to maintain.
- Explicit version checking before connection establishment (via endpoint selection and handshake) prevents version mismatches in normal operation.
#### 4.7.4. Negotiation for Optional Standard Features and Non-Standard Packet Types

- Support for optional standard features (e.g., `DIRECT_SEND`, `FAST_SEND` operations) and any intended use of non-standard packet types (`PacketType` values 128-254) **SHOULD** be established or negotiated during the handshake. An Axon Relay is not obligated to support all optional features.

#### 4.7.5. Message Format Negotiation (Optional)

- The handshake _can_ also be used to negotiate alternative message encoding formats.

### 4.8. Out-of-Scope for this Core Protocol

- **Explicit Channel Lifecycle Management APIs.**
- **Axon Relay Statistics and Monitoring.**
- **Primary/Exclusive Fire-and-Forget Communication.** (See [section 4.3.2](#432-unacknowledged-fire-and-forget-direct-delivery-fast_send) for alternative recommendations.)

### 4.9. General System Design Notes

- **At-Least-Once Delivery (for Buffered Messages):** Core goal for `PUT_MSG`.
- **Error Handling:** Recoverable errors via `NACK`. Terminating conditions via specific `NACK`s.
- **Security:** TLS for transport; application-level E2E encryption for payload recommended.
- **Client Discovery & Channel Initiation:** External mechanisms.
- **Snowflake IDs:** Required for `message_id` for buffered messages; use Twitter epoch.

## 5. Deployment and Operational Considerations (for Axon Relays)

While the protocol itself defines the communication rules, the following deployment and operational practices for Axon Relay services are strongly recommended to ensure reliability, maintainability, and align with the core design principles of the system. This approach leverages DNS for routing and enables advanced operational patterns.

### 5.1. Versioned and Variant-Specific Endpoints via DNS

#### 5.1.1. **Endpoint Naming Convention**

Axon Relay endpoints **MUST** embed the protocol version, a specific revision/build identifier, and transport directly into a **single DNS label** which forms the subdomain of the service's primary domain. If an alternative message encoding format (other than the binary wire format) is used, the message format identifier is appended. The path component of the URL should remain consistent.

- **Structure of the DNS Label:**
  - Binary Wire Format: `[version]-[revision]-[transport]` (3 components, 2 dashes)
  - Alternative Message Format: `[version]-[revision]-[transport]-[messageformat]` (4 components, 3 dashes)
- **Full Domain Structure:**
  - Binary Wire Format: `[version]-[revision]-[transport].your-axon-relay-service.com`
  - Alternative Message Format: `[version]-[revision]-[transport]-[messageformat].your-axon-relay-service.com`

**The protocol version and message format are determined by the DNS label and are not negotiated after connection. The handshake MUST include the version and format for mismatch detection.**

#### 5.1.2. **Components within the Label**

1. **`[version]`**: The core Axon Protocol version (e.g., `v0`, `v1`). **Mandatory.**
2. **`[revision]`**: A specific revision, build number, or patch identifier of the Axon Relay software (e.g., `r1`, `b1023`, `v0patch1`). **Mandatory for specific, stable deployments.**
3. **`[transport]`**: The underlying transport protocol (e.g., `wss`, `ws`, `tcp`). **Mandatory.**
4. **`[messageformat]`**: (Only present if not using the binary wire format, indicated by being the 4th component with 3 dashes total). Identifier for the alternative message encoding format (e.g., `json`).

#### 5.1.3. **Optional "Latest Revision" Alias Endpoints**

> **Usage Warning:**
>
> Using these "latest" alias endpoints is **NOT RECOMMENDED for critical deployments or production applications where stability and predictable behavior are paramount.**
>
> Critical systems should always target specific revision subdomains.

For convenience during development or for non-critical applications, an alias subdomain (again, a single DNS label) pointing to the latest known stable revision of a given Axon Protocol version _may_ be provided. This follows the same component and dash count logic, with the revision set to "`latest`".

- Example: `v0-latest-wss.your-axon-relay-service.com` (default format) or `v0-latest-wss-json.your-axon-relay-service.com` (JSON format).

#### 5.1.2. **Single-Level Subdomain Requirement**

The entire version/variant identifier string (e.g., `v0-r1-wss` or `v0-r1patch1-wss-json`) **MUST** form a single DNS label. This results in a single-level subdomain relative to the Axon Relay service's domain.

- **Rationale:** Constructing the version/variant identifier as a single DNS label ensures broad compatibility with diverse DNS providers and hosting platforms. This structure also greatly simplifies the issuance and management of wildcard TLS certificates (e.g., `*.your-axon-relay-service.com`), as a single certificate can effectively secure all such versioned endpoints. Adherence to this single-label approach promotes straightforward DNS configuration and supports a wider range of compatible infrastructure.
- Correct Example (Specific Revision, Default Format): `v0-r1patch1-wss.axon-relay.example.com`
- Correct Example (Specific Revision, JSON Format): `v0-r1-wss-json.axon-relay.example.com`
- Correct Example (Latest Alias, Default Format): `v0-latest-wss.axon-relay.example.com`

#### 5.1.3. **Path Consistency**

The URL path (e.g., identifying a specific Channel or entry point to obtain a Channel) should remain consistent for Clients, with the Axon Relay software version/variant differentiation handled entirely by the single-label subdomain

### 5.2. DNS-Level Routing and Operational Benefits (for Axon Relay Services)

- **Distinct Deployments:** Each unique specific revision subdomain points to a distinct, independent Axon Relay service deployment. "Latest" aliases point to one of these specific revision deployments.
- **Wildcard Certificates:** When using TLS (e.g., for `wss`), a wildcard certificate for `*.your-axon-relay-service.com` can secure all these dynamically created versioned/variant endpoints, simplifying certificate management.
- **Hot Swapping / Critical Fixes:** A new deployment of Axon Relay software with a critical fix for revision (e.g., `v0-r1`) would be deployed with an updated revision identifier (e.g., `v0-r1p1` or `v0-r2`). Client configurations targeting specific revisions would need to be updated. If a "latest" alias is used, it would be updated to point to the new patched revision.
- **Incremental Rollouts / Canary Releases:** New specific revisions of Axon Relay software can be introduced. Clients targeting specific revisions are unaffected. A small percentage of Clients can be directed to them for testing.
- **Enhanced Reliability & No Rollback Hell:** As older specific revision endpoints for the Axon Relay service remain operational, issues in a new deployment do not affect existing users on stable, pinned revisions.
- **Explicit Client Choice:** Clients targeting specific revision subdomains of the Axon Relay service have a clear, unchanging contract regarding the protocol and relay software version. Clients opting for "latest" aliases accept the risk of the underlying target changing.

### 5.3. Managing Old Versions (of Axon Relay Deployments)

- **Keep Old Specific Revision Endpoints Running:** Given that Axon Relay deployments (especially on serverless platforms) are designed to be lightweight and can scale to zero, keep older, stable specific revision endpoints operational.
- **Resource Management for Old Versions:** Apply more aggressive resource limits to older, less-used Axon Relay deployments if necessary.
- **Deprecation Strategy:** Communicate a clear deprecation timeline for retiring old specific revision endpoints. "Latest" aliases should always point to a supported, stable revision.

This DNS-centric deployment strategy, with mandatory revisions for stable deployments and optional "latest" aliases for flexibility, offers a robust framework for managing the lifecycle of Axon Relay services.

## 6. Default Wire Format Specification

### 6.1. General Packet Structure

All communication occurs via packets (byte arrays). **It is assumed that the transport layer handles packet framing and size.** This protocol defines the content _within_ that byte array.

1. **Header (1 byte):** An 8-bit unsigned integer `PacketType`.
2. **Body (variable size):** Payload, structure depends on `PacketType`. The interpretation of the body (and the presence of certain fields) can depend on the total packet length provided by the transport layer.

- **Standard vs. Non-Standard Packet Types:**
  - If MSB of `PacketType` is 0 (values 0-127), it's a standard data/control packet.
  - If `PacketType` is `255` (`0xFF`), it's the standard `NACK` packet.
  - If MSB of `PacketType` is 1 and value is not `255` (i.e., 128-254), it's a non-standard/experimental type.

- **Standard Packet Type Pairing Convention:**
  - All standard packet types (values 0–127) exist in pairs: a request type and its acknowledgment type.
  - The request type always has an even value (LSB = 0).
  - The acknowledgment type is always the next odd value (LSB = 1), i.e., `request_type + 1`.
  - `NACK` (255) is a special case and does not follow this rule.
  - Future standard packet types SHOULD follow this LSB pairing convention.

### 6.2. Packet Types (`PacketType`)

The `PacketType` is represented as a `u8` on the wire.

```rust
// Axon Messaging Protocol Version: 0
#[repr(u8)]
pub enum PacketType {
    Ping = 0,
    Pong = 1,

    Msg = 2,
    MsgAck = 3,
    GetMsg = 4,
    GetMsgAck = 5,
    PutMsg = 6,         // For buffered messages (TTL > 0)
    PutMsgAck = 7,

    ListMsg = 8,
    ListMsgAck = 9,

    DirectSend = 10,    // Optional: Acknowledged direct, non-persistent send
    DirectSendAck = 11, // Optional: Ack for DirectSend

    FastSend = 12,      // Optional: Unacknowledged ("fire-and-forget") direct, non-persistent send
    FastSendAck = 13,   // Optional: Reserved No-Op. Client MUST NOT send. Relay MUST NOT send.

    // Values 14-127 are reserved for future standard types (must follow LSB pairing).
    // Values 128-254 (MSB set, not 0xFF) are for non-standard/experimental types.

    Nack = 255,
}
```

### 6.3. Detailed Packet Definitions (Standard Types)

All multi-byte integer fields are Big Endian / Network Byte Order. `message_id` is a `u64` representing a Snowflake ID (or `0` for direct/fast relayed messages). Timestamps are `u64`, e.g., Unix milliseconds.

---
**6.3.1. `PING` (Type 0)**

- A **simple PING** packet consists only of the 1-byte `PacketType` header (`0x00`). Its body is 0 bytes.
- A **timestamped PING** packet includes an 8-byte body structured as defined by `struct Ping` below.

```rust
// Represents the body of a timestamped PING.
// For a simple PING, the body is absent.
pub struct Ping {
    pub sender_timestamp: u64, // (8 bytes) e.g., Unix milliseconds
}
```

---
**6.3.2. `PONG` (Type 1)**

- Responds to a `PING`.
- A **simple PONG** packet consists only of the 1-byte `PacketType` header (`0x01`). Its body is 0 bytes.
- A **full PONG** packet (response to timestamped PING) includes a 24-byte body (`struct Pong`).

```rust
// Represents the body of a full PONG.
// For a simple PONG, the body is absent.
pub struct Pong {
    pub original_sender_timestamp: u64, // (8 bytes) Mirrored from the timestamped PING
    pub receiver_receipt_timestamp: u64,  // (8 bytes) Time when PING was received (or 0 if not providing actual timing)
    pub receiver_transmit_timestamp: u64, // (8 bytes) Time when PONG is sent (or 0 if not providing actual timing)
}
```

---
**6.3.3. `MSG` (Type 2)**

```rust
// Body structure after the 1-byte PacketType header:
pub struct Msg {
    pub message_id: u64,      // (8 bytes) Non-zero for buffered messages. MUST be 0 if relaying DIRECT_SEND or FAST_SEND.
    pub data: Vec<u8>,        // (remaining bytes)
}
```

---
**6.3.4. `MSG_ACK` (Type 3)**

```rust
// Body structure after the 1-byte PacketType header:
pub struct MsgAck {
    pub message_id: u64,      // (8 bytes) ID of message being acknowledged (from MSG or GET_MSG_ACK). MUST be non-zero.
}
```

---
**6.3.5. `GET_MSG` (Type 4)**

```rust
// Body structure after the 1-byte PacketType header:
pub struct GetMsg {
    pub message_id: u64,      // (8 bytes)
}
```

---
**6.3.6. `GET_MSG_ACK` (Type 5)**

- Indicates successful retrieval. Errors via `NACK`.

```rust
// Body structure after the 1-byte PacketType header:
pub struct GetMsgAck {
    pub message_id: u64,      // (8 bytes)
    pub data: Vec<u8>,        // (remaining bytes) Message content.
}
```

---
**6.3.7. `PUT_MSG` (Type 6)**

- Used for submitting messages intended for persistent buffering (`ttl` MUST be `> 0`).

```rust
// Body structure after the 1-byte PacketType header:
pub struct PutMsg {
    pub idempotency_key: u32, // (4 bytes)
    pub ttl: u32,             // (4 bytes) TTL in seconds (MUST be > 0).
    pub data: Vec<u8>,        // (remaining bytes)
}
```

---
**6.3.8. `PUT_MSG_ACK` (Type 7)**

- Indicates successful storage of a `PUT_MSG`. Errors via `NACK`.

```rust
// Body structure after the 1-byte PacketType header:
pub struct PutMsgAck {
    pub idempotency_key: u32, // (4 bytes) Mirrored from PUT_MSG.
    pub ttl: u32,             // (4 bytes) Honored TTL (will be > 0).
    pub message_id: u64,      // (8 bytes) Axon Relay-assigned Snowflake ID (non-zero).
}
```

---
**6.3.9. `LIST_MSG` (Type 8)**

```rust
// Body structure after the 1-byte PacketType header:
pub struct ListMsg {
    pub limit: u16,           // (2 bytes) Max IDs. 0 is valid (results in empty LIST_MSG_ACK).
    pub from: u64,            // (8 bytes) Exclusive 'from' cursor.
    pub to: u64,              // (8 bytes) Exclusive 'to' cursor.
}
```

- **Cursor Behavior & Special Values:** (As previously defined: ascending/descending, timestamp-based, 0/u64::MAX). An empty list is returned if `from == to`, `limit == 0`, or no messages match.

---
**6.3.10. `LIST_MSG_ACK` (Type 9)**

- Response to `LIST_MSG`. Always indicates successful processing of the list request. Errors preventing the list operation send `NACK`.
- If no messages match, the body is 0 bytes (packet is 1 byte header).

```rust
// Body structure after the 1-byte PacketType header IF message_ids are present:
pub struct ListMsgAck {
    pub message_ids: Vec<u64>,    // (n * 8 bytes)
}
```

---
**6.3.11. `DIRECT_SEND` (Type 10) - Optional Feature**

- For acknowledged, direct, non-persistent message relay.

```rust
// Body structure after the 1-byte PacketType header:
pub struct DirectSend {
    pub idempotency_key: u32, // (4 bytes, must be non-zero by Client convention for this packet type)
    pub data: Vec<u8>,        // (remaining bytes)
}
```

---
**6.3.12. `DIRECT_SEND_ACK` (Type 11) - Optional Feature**

- Acknowledges successful initiation of `DIRECT_SEND` relay attempt.

```rust
// Body structure after the 1-byte PacketType header:
pub struct DirectSendAck {
    pub idempotency_key: u32, // (4 bytes) Mirrored from DIRECT_SEND.
}
```

---
**6.3.13. `FAST_SEND` (Type 12) - Optional Feature**

- For unacknowledged, direct, non-persistent message relay.
- The body, immediately following the `PacketType` header, is the raw message `data`.
- (No Rust struct definition for the body as it's just `Vec<u8>` data directly).

---
**6.3.14. `FAST_SEND_ACK` (Type 13) - Optional Feature / Reserved No-Op**

- Reserved for LSB pairing. **Clients MUST NOT send. Axon Relays MUST NOT send.**
- If Relay receives from Client: Protocol Violation. Relay SHOULD send `NACK` (`error_code=ErrorCode::ProtocolViolation`) then SHOULD terminate.
- If Client receives from Relay: Protocol Violation. Client SHOULD log, ignore, MAY terminate.
- Packet, if (incorrectly) sent, is 1-byte header only (`0x0D`).

---
**6.3.15. `NACK` (Type 255 / `0xFF`)**

```rust
// Body structure after the 1-byte PacketType header:
pub struct Nack {
    pub original_packet_type: u8, // (1 byte) PacketType of offending packet, or PacketType::Nack (0xFF) for general.
    pub error_code: u8,           // (1 byte) See ErrorCode enum.
    pub correlation_data: Vec<u8>,// (Variable bytes) Optional. Empty for FAST_SEND NACKs.
}
```

- **`ErrorCode` Categories and Expected Handling:**
  The 8-bit `error_code` field provides information about the NACK. If an implementation receives an unrecognized `error_code` for the given `original_packet_type`, it **SHOULD** treat it as `ErrorCode::CriticalErrorAbort (0xFF)`. Implementations sending a `NACK` MUST use defined error codes.

  ```rust
  pub enum ErrorCode {
      // --- 0x00 - 0x1F: Informational & Standard Operational Outcomes ---
      // Reaction: Normal flow. Log debug only. Connection status per code.
      GracefulDisconnect = 0x00,              // OK: (orig_type=0xFF). Connection MUST close.
      ProtocolVersionMismatch = 0x01,         // INFO: (orig_type=0xFF). Connection MUST close by sender, receiver too.
      MsgNotFound = 0x02,                     // INFO: For GET_MSG. Connection open. (Context: original_packet_type=GetMsg)
      // 0x03 - 0x1E: Reserved for future informational codes.
      NoOperationPerformed = 0x1F,            // INFO: Request valid, no action needed. Connection open.
  
      // --- 0x20 - 0x9F: Recoverable Client-Side Errors / Minor Policy Issues ---
      // Client Reaction: Client logic error/invalid input. Log, modify request, may retry. Connection open.
      InvalidTtlForPut = 0x20,                // FAIL: PUT_MSG TTL violates Relay policy. (Context: original_packet_type=PutMsg)
      InvalidListParametersNonCritical = 0x21,// FAIL: LIST_MSG params logically unserviceable for non-empty result but not a bug. (Context: original_packet_type=ListMsg)
      IdempotencyKeyReusedWithDifferentData = 0x22, // FAIL: PUT_MSG (buffered) key reused with different data. (Context: original_packet_type=PutMsg)
      // 0x23 - 0x9F: Reserved for other client-correctable errors.
  
      // --- 0xA0 - 0xDF: Warnings / Relay Policy Enforcement / Service Degradation ---
      // Client Reaction: Log. Inform user. Adaptive logic (slow down, fallback). Connection open.
      RateLimitWarning = 0xA0,                // WARN: Approaching rate limits.
      RelayResourceWarning = 0xA1,            // WARN: Relay high load. Service may be degraded.
      FeatureTemporarilyDisabled = 0xA2,      // WARN: Negotiated optional feature temporarily disabled by Relay.
      AuthWarning = 0xA3,                     // WARN: Auth credential nearing expiry/needs attention.
      DirectDeliveryUnsupportedByRelay = 0xA4, // WARN/FAIL: DIRECT_SEND/FAST_SEND received but feature not enabled/supported by this Relay (if negotiated as optional).
      // 0xA5 - 0xDF: Reserved for other warning/policy types.
  
      // --- 0xE0 - 0xEF: Critical Transient/Environmental Errors (Primarily Relay-Side) ---
      // Reaction: Log extensively. Client may retry/reconnect with exponential backoff. Connection SHOULD close by both.
      TemporaryRelayUnavailable = 0xE0,       // CRIT: Relay temporarily unable to process.
      InternalStorageErrorTransient = 0xE1,   // CRIT: Relay backend storage issue, might recover.
      NetworkErrorToBackend = 0xE2,           // CRIT: Relay cannot reach its own necessary backend services.
      // 0xE3 - 0xEF: Reserved.
  
      // --- 0xF0 - 0xFE: Critical Protocol/Implementation/Parameter Errors (Bugs or Violations) ---
      // Reaction: Log extensively. Client generally should NOT retry operation. Connection SHOULD close by both.
      MalformedPacket = 0xF0,                 // CRIT BUG: Packet violates structural rules.
      ProtocolViolation = 0xF1,               // CRIT BUG: Semantic rule violation (e.g., forbidden packet FAST_SEND_ACK from Client, or Client sending MSG_ACK for message_id=0).
      UnsupportedStandardPacketType = 0xF2,   // CRIT BUG: Standard packet type sent not in negotiated version or unknown.
      UnsupportedNonStandardPacketType = 0xF3,// CRIT BUG: Unnegotiated/Unsupported MSB=1 packet type.
      InvalidParametersCritical = 0xF4,       // CRIT BUG: Structurally valid, but logically invalid parameters indicating client bug.
      AuthenticationFailure = 0xF5,           // CRIT SEC: Client authentication failed.
      AuthorizationFailure = 0xF6,            // CRIT SEC: Client authenticated but not authorized for operation/channel.
      RateLimitHardExceeded = 0xF7,           // CRIT: Client exceeded hard rate limits.
      // 0xF8 - 0xFD: Reserved.
      UnspecifiedRelayInternalError = 0xFE,  // CRIT BUG: Unrecoverable internal error in Relay.
  
      // --- Critical Error Abort (Original Packet Type will be 0xFF for this specific code) ---
      CriticalErrorAbort = 0xFF,              // CRIT BUG: Unspecified critical error initiated by sender, forces abort. Connection MUST close.
  }
  ```

- **Connection Handling with NACK (Summary):**
  - If `original_packet_type == PacketType::Nack (0xFF)` AND (`error_code == ErrorCode::GracefulDisconnect (0x00)` OR `error_code == ErrorCode::CriticalErrorAbort (0xFF)`), connection MUST be closed by both parties.
  - For any `NACK` with an `error_code` in the range `0xE0-0xFF`, both the sender and receiver of the `NACK` **SHOULD** close the connection.
  - Other `NACK`s (typically `error_code` `0x01-0xDF`) by default DO NOT cause protocol-mandated connection closure from the receiver's side.

## 7. Why This Default Wire Format?

The choice to define this specific custom binary wire format, rather than mandating an existing generic format, is driven by the primary goal of **minimizing the effort required to implement the protocol's encoding/decoding layer**. This allows developers to focus more on the overall system design, application logic, and operational management of Clients and Axon Relays. While the wire format is a critical piece, it's understood to be only part of the larger puzzle. The aim here is to provide a clear, simple, and efficient default, enabling quicker progress on other core system components.

- **Comparison to Generic Formats:**
  - **ASN.1, Protocol Buffers, Thrift, Avro:** Offer rich features and tooling but can introduce complexity, dependencies, and overhead not strictly necessary for this protocol's focused scope.
  - **JSON, BSON:** While widely supported, text-based parsing (JSON) or even structured binary (BSON with field names) can be less performant and more complex to implement efficiently from scratch compared to this direct binary mapping.

- **Advantages of this Custom Default Format:**
  - **Simplicity:** Fixed-offset fields for headers and reliance on the transport layer for overall packet sizing make parsing and generation straightforward.
  - **Minimal Dependencies:** Can be implemented with basic byte manipulation, without external libraries or code generators for the core encoding.
  - **Efficiency:** Compact representation with no redundant metadata (like field names) in the packets.
  - **Flexibility in Payload:** The `data` fields are opaque byte arrays. End-Clients are free to choose any format for the actual message body (e.g., plain text, JSON, custom binary), allowing application-specific flexibility where it matters most. The Axon Relay typically treats this payload as an opaque blob.

While this custom binary format is recommended for its alignment with the Axon Protocol's design goals, the "Protocol Versioning and Handshake" mechanism acknowledges that implementations might choose alternative encodings if agreed upon by Clients and Axon Relays, provided the logical operations and data fields are preserved.
