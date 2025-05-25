# Axon Messaging Protocol - A Minimalist Design for Reliable and Scalable Mediated Client Channels

**Protocol Version: 0**

## 1. Overview

This document outlines the **Axon Messaging Protocol** (or **Axon Protocol**), a communication system designed to leverage the strengths and characteristics of **serverless architectures** for its relaying components. The protocol facilitates communication **Channels** that are conceptually one-to-one between two end-**Clients**. These two **Clients** individually connect to an intermediary component, termed the **Axon Relay**, which acts as a message buffer and relay for their shared **Channel**, ensuring **at-least-once message delivery**. The Axon Protocol is optimized for these scenarios where an Axon Relay mediates communication for a pair of Clients.

The protocol is **designed to be minimalistic, fostering reliability and high scalability**, particularly for scenarios involving a **massive number of concurrent Channels, each typically involving two active Clients and a mediating Axon Relay instance**. Key to its scalability is this focus on small, independent Channels and the expectation of **per-tenant/per-channel data partitioning** within the Axon Relay's backend. Its reliability is further supported by mechanisms for at-least-once delivery and acknowledged message handling. This design was initially inspired by and intended for environments like Cloudflare Workers utilizing Durable Objects for the Axon Relay's buffer storage, but it is adaptable to other serverless platforms and partitioned storage solutions.

The Axon Protocol aims to be:

- **Easy to implement** for both Client applications and Axon Relay services.
- **Requires minimal resources** in terms of computational power, bandwidth, and storage per Channel.
- Adaptable to **flexible buffering/storage backends** for the Axon Relay (e.g., SQLite, Turso, Cloudflare D1/Durable Objects, Amazon S3, Redis, local filesystem, or simple in-memory stores), especially those that support efficient partitioning.

While chat between two Clients serves as an intuitive primary use case for illustration, the protocol's core design allows it to also serve as a foundation for simple pub/sub systems or job queues where an Axon Relay facilitates the message exchange with at-least-once delivery guarantees.

A **Client** is an end-user application initiating operations or receiving messages. The **Axon Relay** is the intermediary component (e.g., serverless function, Durable Object) that provides the buffering and relaying services for a **Channel**. The Axon Relay is primarily concerned with message handling mechanics (storage, forwarding, TTL, ensuring at-least-once delivery semantics through acknowledgments) and is generally not concerned with the content of the message payload itself, which is passed between Clients.

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
- **Optimized Two-Client Channels via Axon Relay:** Tailored for interactions between two Clients mediated by an Axon Relay acting as a buffer.
- **Minimalistic and Efficient:** Low resource usage, high performance, and ease of implementation for Clients and Axon Relays.
- **Flexible, Partitioned Storage:** Compatible with various backends for Axon Relays, with an emphasis on partitioning data per Channel or tenant for scalability.
- **Versatile Design:** Suitable for chat-like applications, minimalistic pub/sub, or simple job queues.
- **Message Buffering with TTL:** Axon Relays can store messages; TTL defines max buffering duration.
- **At-Least-Once Delivery:** Mechanisms ensure messages are delivered at least once between Clients and their Axon Relay.
- **Acknowledged Deletion:** Axon Relays attempt to delete messages from their buffer upon `MSG_ACK` from a Client or TTL expiry.
- **Idempotent Operations:** `PUT_MSG` from a Client to an Axon Relay includes an idempotency key.
- **Connection Liveness:** Essential `PING`/`PONG` mechanism between Clients and Axon Relays with defined behaviors.
- **Unified Error Handling:** Dedicated `NACK` packet for reporting errors and connection status signals.
- **Extensibility:** Convention for non-standard packet types and negotiation via handshake.
- **Simple and Compact Default Wire Format:** Easy to parse and generate, minimizing bandwidth.

## 4. System Design and Behavior

This section describes the logical operations, expected behaviors of participating Clients and Axon Relays, and the overall system characteristics for a communication Channel.

### 4.1. Core Operations

The protocol is built around these fundamental operations between a Client and an Axon Relay:

#### 4.1.1. Liveness Check (PING/PONG Operation):

- A Client or Axon Relay sends a `PING` packet. A "simple PING" has no body (just the header). A "timestamped PING" includes a sender-side timestamp.
- The receiving party _must_ respond with a `PONG` packet, following specific rules based on the PING received (see "4.2. General Expected Behaviors (Client & Axon Relay)" below, specifically "PING/PONG Behavior").
- _Enables:_ Connection liveness verification, round-trip time estimation.

#### 4.1.2. Submitting a Message (The "PUT" Operation):

- A Client sends new message content, an idempotency key, and a suggested TTL to an Axon Relay for a specific Channel.
- The Axon Relay acknowledges with the idempotency key, the honored TTL, and a unique `message_id`.
- _Enables:_ Buffering on the Axon Relay, at-least-once submission from Client to Axon Relay, message lifecycle via TTL.

#### 4:1.3. Receiving a Message (The "PUSH" Operation):

- An Axon Relay proactively sends a `message_id` and its content (originating from the other Client in the Channel) to a connected Client.
- The Client acknowledges receipt of the `message_id`.
- _Enables:_ Proactive delivery from Axon Relay to Client, at-least-once delivery, Axon Relay buffer cleanup on acknowledgment.

#### 4.1.4. Listing Available Messages (The "LIST" Operation):

- A Client requests a list of `message_id`s from the Axon Relay. The request includes `limit`, `from` (exclusive cursor), and `to` (exclusive cursor) parameters.
- The Axon Relay responds with a `LIST_MSG_ACK` packet containing matching `message_id`s.
  - If `from < to`, messages are listed in ascending order of `message_id`.
  - If `from > to`, messages are listed in descending order of `message_id`.
- _Enables:_ Efficient, paginated synchronization and recovery. The `from` and `to` cursors can be full `message_id`s, partial `message_id`s (timestamp portion filled, rest zeroed for time-based queries), or special values (all-zero for start, all-one for end).

#### 4.1.5. Retrieving a Specific Message (The "GET" Operation):
- A Client requests specific message content from the Axon Relay using its `message_id`.
- The Axon Relay responds with the `message_id` and its content in a `GET_MSG_ACK` packet.
- The Client acknowledges receipt of the `message_id`'s content by sending a `MSG_ACK`.
- _Enables:_ On-demand retrieval by a Client, at-least-once retrieval, Axon Relay buffer cleanup.

### 4.2. General Expected Behaviors (Client & Axon Relay)

#### 4.2.1. Local Message Copies
- **Axon Relay:** MUST keep messages in its buffer for at least their defined TTL or until acknowledged by the recipient Client (via `MSG_ACK` following a `MSG` or `GET_MSG_ACK`).
- **Client:** SHOULD maintain its own local copy/history of sent and received messages.
#### 4.2.2. Good Actor Assumption
The protocol design assumes Clients and Axon Relays are generally cooperative. Security measures are layered on top.
#### 4.2.3. Handling of Empty Message Data
All parties SHOULD ignore `MSG`, `GET_MSG_ACK`, or `PUT_MSG` packets where the `data` field is empty, unless explicitly meaningful for the application.
#### 4.2.4. Idempotency
- **Client (`PUT_MSG`):** Must generate a unique `idempotency_key` for each new message submission to the Axon Relay.
- **Axon Relay (`PUT_MSG`):** Must use the `idempotency_key` to prevent duplicate message storage from retried `PUT_MSG` requests.
#### 4.2.5. Acknowledgments
`MSG_ACK`, `PUT_MSG_ACK`, `GET_MSG_ACK`, `LIST_MSG_ACK`
- These packets _always_ signify that the corresponding request operation (`MSG` delivery, `PUT_MSG`, `GET_MSG`, `LIST_MSG`) was successfully processed by the receiver according to the protocol rules.
- A "successful process" for `LIST_MSG` includes returning an empty list if no messages match the criteria.
- Actual operational errors encountered by the Axon Relay while attempting to fulfill a Client's request (`PUT_MSG`, `GET_MSG`, `LIST_MSG`) are reported back to the Client using a `NACK` packet.
- **Client:** Must send `MSG_ACK` upon successful receipt of `MSG` or `GET_MSG_ACK` from the Axon Relay.
- **Axon Relay:** Must send `PUT_MSG_ACK` upon successful storage of `PUT_MSG`. Must send `GET_MSG_ACK` upon successful retrieval for `GET_MSG`. Must send `LIST_MSG_ACK` upon successful processing of a `LIST_MSG` request (which may result in an empty list of `message_id`s).
#### 4.2.6. Duplicate/Out-of-Order Messages
Client applications must be prepared to handle duplicate or out-of-order `MSG` packets from the Axon Relay (e.g., by checking `message_id` against local history).
#### 4.2.7. PING/PONG Behavior:
- Both Client and Axon Relay MUST implement the ability to send and respond to PING/PONG packets.
- Any party (Client or Axon Relay) can send a `PING`.
- If a party receives a **simple PING** (a `PING` packet with no body/timestamp), it **MUST** respond with a **simple PONG** (a `PONG` packet with no body/timestamps).
- If a party receives a **timestamped PING** (a `PING` packet containing a `sender_timestamp`), it **MUST** respond with either:
  1. A **simple PONG** (a `PONG` packet with no body/timestamps), effectively ignoring the received timestamp.
  2. OR a **full PONG** containing all three timestamp fields: `original_sender_timestamp` (mirrored from the PING), `receiver_receipt_timestamp` (its own receipt time), and `receiver_transmit_timestamp` (its own transmit time). These latter two timestamp fields **MUST** be present in the packet structure if this option is chosen, even if their _values_ are zero (e.g., if the receiver does not provide actual timing but acknowledges the extended PONG structure).
#### 4.2.8. Processing NACKs and Connection Lifecycle:
- If a party receives a `NACK` packet (Type `0xFF`) with `original_packet_type = 0xFF` and `error_code = 0x00` (Graceful Disconnect) or `error_code = 0xFF` (Critical Error) sent by the other party, it **MUST** prepare to close the connection. After processing this `NACK`, the party **MUST** close its end of the underlying connection.
- For all other `NACK` types (i.e., those reporting recoverable errors for specific operations where `original_packet_type` is not `0xFF`), the connection **MUST NOT** be closed by the protocol itself. The party should handle the reported error and may continue communication.
- For all standard packet types other than the terminating `NACK`s described above, the connection **MUST NOT** be closed by the protocol.
#### 4.2.9. Handling of Unrecognized Packet Types:
- If a party receives a packet with a `PacketType` whose MSB is 0 (indicating a standard type, values 0-127) but the type is unknown to the receiver (e.g., from a future protocol version it doesn't support), it SHOULD silently drop the packet.
- If a party receives a packet with a `PacketType` whose MSB is 1 and value is not `0xFF` (i.e., values 128-254, indicating a non-standard type) and this type has not been negotiated or is not supported, it SHOULD silently drop the packet.
- `NACK` (Type `0xFF`) is a standard packet and must be parsed at least to determine if it's a connection termination signal.
#### 4.2.10. Strict Adherence to Standard Format
For standard packet types (MSB of `PacketType` is 0, or `PacketType` is `0xFF`), Clients and Axon Relays MUST strictly follow the documented wire format.

### 4.3. Axon Relay Specific Behaviors

- **Error Reporting via `NACK`:**
  - If an Axon Relay encounters a recoverable error while processing a Client's request (`PUT_MSG`, `GET_MSG`, `LIST_MSG`)—such as "Message Not Found" for `GET_MSG`, "Idempotency Key Violation" for `PUT_MSG`, or an internal issue preventing a `LIST_MSG` (e.g., storage temporarily unavailable)—it SHALL respond to the requesting Client with a `NACK` packet.
  - The `NACK` will specify the `Original_PacketType_Errored`.
  - It will include an `ErrorCode` specific to that original packet type's context.
  - It MAY include `CorrelationData` (e.g., the `message_id` from the failed `GET_MSG`, or the `idempotency_key` from the failed `PUT_MSG`).
- **Critical Errors / Rate Limiting / Graceful Disconnect and Connection Closure:**
  - In the event of critical unrecoverable errors, authentication failures against a Client, or persistent rate limit violations by a Client, the Axon Relay **SHALL** send a `NACK` packet to that Client with:
    - `original_packet_type = 0xFF`
    - `error_code = 0xFF` (Critical Error)
  - If the Axon Relay wishes to perform a graceful shutdown of the connection for a Client (e.g., maintenance, idle timeout):
    - It **SHALL** send a `NACK` packet to that Client with:
      - `original_packet_type = 0xFF`
      - `error_code = 0x00` (Graceful Disconnect)
  - After sending either of these terminating `NACK` packets, the Axon Relay **MUST** close its end of the underlying connection to that Client.
  - An Axon Relay MAY send a non-standard packet (MSB=1 in `PacketType`, values 128-254) containing additional diagnostic information _before_ sending one of these standard terminating `NACK` packets, if such extended error reporting has been negotiated or is understood by the Client. The standard terminating `NACK` remains the official and required signal for connection closure.
- **Dynamic Channel Resource Management (Conceptual):** Axon Relay instances (or their underlying infrastructure) are expected to manage resources for communication Channels on demand, typically tied to the per-tenant/per-channel partitioning.
- **Tenant/Channel Isolation:** Paramount. Achieved through partitioned storage and instance isolation (e.g., one Durable Object per Channel).
- **One Channel Per Axon Relay Instance (Recommended):** For strong isolation and simplified state management, it is typical for one instance of an Axon Relay (e.g., one Durable Object) to manage a single Channel between two Clients.
- **Access Control & Rate Limiting (Implementation Detail):** The Axon Relay implementation MUST handle access control (authenticating Clients and authorizing their access to specific Channels) and SHOULD implement rate limiting. These are considered part of the Axon Relay's operational responsibilities rather than specific protocol messages.

### 4.4. Axon Relay Storage: Partitioning for Scalability

A critical design aspect for achieving scalability with a massive number of concurrent Channels is the **partitioning of storage on a per-tenant or per-Channel basis** by the Axon Relay component.

- **Why Partitioning is Crucial:** Isolation, scalability, resource management, data lifecycle, concurrency.
- **Conceptual Partitioning Examples:**
  1. **Object Storage (e.g., Amazon S3):** Prefix per Channel: `s3://bucket/channels/<channel_id>/<message_id>`.
  2. **Key-Value Stores (e.g., Redis):** Prefixed keys: `<channel_id>:<message_id>`.
  3. **Relational Databases (e.g., SQLite):** Separate database files per Channel: `channel_<channel_id>.db`.
  4. **Cloudflare Durable Objects:** Each Durable Object instance inherently acts as a partition for a single Channel, managing its state and storage.
- **Schema within a Partition (e.g., per-Channel SQLite DB or Durable Object storage):**
  ```sql
  CREATE TABLE messages (
      message_id BIGINT PRIMARY KEY, -- (8 bytes) Snowflake ID (contains creation timestamp)
      expiry     INTEGER,           -- (4 bytes, optional) Unix timestamp (seconds) for message expiry.
      data       BLOB               -- (Variable bytes) The raw message content
  );
  ```

### 4.5. Protocol Versioning, Handshake, and Backwards Compatibility

#### 4.5.1. Protocol Version Identification

- This document describes **Protocol Version 0** of the Axon Messaging Protocol.
- Each distinct version of this protocol specification will have a unique, non-negative integer version number. Future versions will increment this number (e.g., 1, 2, 3...).

#### 4.5.2. Handshake for Version Negotiation

- Implementations **SHOULD** establish the protocol version being used during an initial handshake between a Client and an Axon Relay.
- The Client typically initiates by stating the highest protocol version it supports.
- The Axon Relay responds with the protocol version it will use for the session, which should be less than or equal to the version the Client offered and the highest version the Axon Relay itself supports.
- If no common version can be agreed upon, the connection should be terminated gracefully (e.g., Axon Relay sends a `NACK` with `original_packet_type = 0xFF`, `error_code = 0x01` (Version Mismatch), then closes).

#### 4.5.3. Handling Different Versions

- Implementations supporting a newer protocol version SHOULD be capable of gracefully handling interactions with Clients/Axon Relays that only support older versions, typically by restricting communication to the features and packet types defined in the older, common version agreed upon during the handshake.
- All parties should gracefully ignore unknown standard packet types from newer versions if they adhere to the "drop unrecognized packet" rule, assuming the handshake has established an older common version.

#### 4.5.4. Negotiation for Non-Standard Packet Types

- If a Client and Axon Relay intend to use non-standard packet types (`PacketType` values 128-254), their usage and meaning MUST be negotiated during the handshake, independently of the core protocol version.

#### 4.5.5. Message Format Negotiation (Optional)

- The handshake _can_ also be used to negotiate alternative message encoding formats if desired.

### 4.6. Out-of-Scope for this Core Protocol

To maintain its minimalistic nature, the following are considered outside the direct scope of this messaging protocol:

- **Explicit Channel Lifecycle Management APIs:** (e.g., a separate REST API for creating/managing Channels or Axon Relay instances).
- **Axon Relay Statistics and Monitoring:** (e.g., via dedicated monitoring endpoints).

### 4.7. General System Design Notes

- **At-Least-Once Delivery:** Core goal, achieved via retries and acknowledgments between Clients and the Axon Relay.
- **Error Handling (Message-Level):** Recoverable errors are reported via `NACK`. Connection-terminating conditions are signaled by specific `NACK`s.
- **Security:** Primarily handled at layers above or below this core messaging protocol.
  - **Transport Layer Security (TLS):** Strongly recommended for Client-AxonRelay connections.
  - **Application-Level Encryption:** Recommended for end-to-end encryption of the `data` payload between Clients, where the Axon Relay only relays encrypted blobs. This requires using established libraries (e.g., OpenSSL, BoringSSL).
  - Detailed security patterns are outside this document.
- **Client Discovery & Channel Initiation:** Mechanisms for Clients to discover each other or be assigned to a Channel (and thus to an Axon Relay instance) are handled externally.
- **Snowflake IDs:** Recommended for `message_id` for uniqueness and embedded timestamp.

## 5. Deployment and Operational Considerations (for Axon Relays)

While the protocol itself defines the communication rules, the following deployment and operational practices for Axon Relay services are strongly recommended to ensure reliability, maintainability, and align with the core design principles of the system. This approach leverages DNS for routing and enables advanced operational patterns.

### 5.1. Versioned and Variant-Specific Endpoints via DNS

#### 5.1.1. Endpoint Naming Convention (Specific Revisions)
Axon Relay endpoints **MUST** embed the protocol version, a specific revision/build identifier, and transport directly into a **single DNS label** which forms the subdomain of the service's primary domain. If an alternative message encoding format (other than the default binary wire format) is used, the message format identifier is appended. The path component of the URL should remain consistent.

- **Structure of the DNS Label:**
  - For Default Binary Wire Format: `[version]-[revision]-[transport]` (3 components, 2 dashes)
  - For Alternative Message Format: `[version]-[revision]-[transport]-[messageformat]` (4 components, 3 dashes)
- **Full Domain Structure:**
  - Default: `[version]-[revision]-[transport].your-axon-relay-service.com`
  - Alternative: `[version]-[revision]-[transport]-[messageformat].your-axon-relay-service.com`

- **Components within the Label:**
  - **`[version]`**: The core Axon Protocol version (e.g., `v0`, `v1`). **Mandatory.**
  - **`[revision]`**: A specific revision, build number, or patch identifier of the Axon Relay software (e.g., `r1`, `b1023`, `v0patch1`). **Mandatory for specific, stable deployments.**
  - **`[transport]`**: The underlying transport protocol (e.g., `wss`, `ws`, `tcp`). **Mandatory.**
  - **`[messageformat]`**: (Only present if not using the default binary wire format, indicated by being the 4th component with 3 dashes total). Identifier for the alternative message encoding format (e.g., `json`).

- **Optional "Latest Revision" Alias Endpoints:** For convenience during development or for non-critical applications, an alias subdomain (again, a single DNS label) pointing to the latest known stable revision of a given Axon Protocol version _may_ be provided. This follows the same component and dash count logic.

  - **Alias for Default Binary Wire Format:**
    - DNS Label: `[version]-latest-[transport]`
    - Full Domain: `[version]-latest-[transport].your-axon-relay-service.com`
  - **Alias for Alternative Message Format:**
    - DNS Label: `[version]-latest-[transport]-[messageformat]`
    - Full Domain: `[version]-latest-[transport]-[messageformat].your-axon-relay-service.com`
  - Example: `v0-latest-wss.your-axon-relay-service.com` (default format) or `v0-latest-wss-json.your-axon-relay-service.com` (JSON format).
  - **Usage Warning:** Using these "latest" alias endpoints is **NOT RECOMMENDED for critical deployments or production applications where stability and predictable behavior are paramount.** Critical systems should always target specific revision subdomains.

#### 5.1.2. Single-Level Subdomain Requirement:<!-- {"fold":true} -->
The entire version/variant identifier string (e.g., `v0-r1-wss` or `v0-r1patch1-wss-json`) **MUST** form a single DNS label. This results in a single-level subdomain relative to the Axon Relay service's domain.

- **Rationale:** Constructing the version/variant identifier as a single DNS label ensures broad compatibility with diverse DNS providers and hosting platforms. This structure also greatly simplifies the issuance and management of wildcard TLS certificates (e.g., `*.your-axon-relay-service.com`), as a single certificate can effectively secure all such versioned endpoints. Adherence to this single-label approach promotes straightforward DNS configuration and supports a wider range of compatible infrastructure.
- Correct Example (Specific Revision, Default Format): `v0-r1patch1-wss.axon-relay.example.com`
- Correct Example (Specific Revision, JSON Format): `v0-r1-wss-json.axon-relay.example.com`
- Correct Example (Latest Alias, Default Format): `v0-latest-wss.axon-relay.example.com`

#### 5.1.3. Path Consistency<!-- {"fold":true} -->
The URL path (e.g., identifying a specific Channel or entry point to obtain a Channel) should remain consistent for Clients, with the Axon Relay software version/variant differentiation handled entirely by the single-label subdomain.

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
  - Most standard packet types (values 0-127, excluding `NACK`) that represent an operation with a direct response will exist in pairs: a request type and an acknowledgment type.
  - The request packet type value will have its Least Significant Bit (LSB) as `0` (i.e., it will be an even number).
  - The corresponding acknowledgment packet type value (`_ACK`) will have its LSB as `1` (i.e., it will be an odd number), and its value will be `request_type + 1`.
  - This convention applies to `MSG`/`MSG_ACK`, `GET_MSG`/`GET_MSG_ACK`, `PUT_MSG`/`PUT_MSG_ACK`, `LIST_MSG`/`LIST_MSG_ACK`.
  - `PING` (0) and `PONG` (1) also adhere to this pattern, with `PONG` acting as the acknowledgment to `PING`.
  - The `NACK` packet (255) is a special case and does not follow this pairing rule.
  - All future standard operational packet types MUST adhere to this LSB pairing convention.

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
    GetMsgAck = 5,  // Acknowledgment to GET_MSG, always success of retrieval.
    PutMsg = 6,
    PutMsgAck = 7,  // Acknowledgment to PUT_MSG, always success of storage.
    ListMsg = 8,
    ListMsgAck = 9, // Acknowledgment to LIST_MSG, always success of list generation (may be empty).

    // Values 10-127 are reserved for future standard types (must follow LSB pairing).
    // Values 128-254 (MSB set, not 0xFF) are for non-standard/experimental types.

    Nack = 255,
}
```

### 6.3. Detailed Packet Definitions (Standard Types)

All multi-byte integer fields are Big Endian / Network Byte Order. `message_id` is a `u64` representing a Snowflake ID. Timestamps are `u64`, e.g., Unix milliseconds.

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
// Total packet size:
// Simple PING: 1 byte (header)
// Timestamped PING: 1 byte (header) + 8 bytes (Ping struct) = 9 bytes
```

---

**6.3.2. `PONG` (Type 1)**

- Responds to a `PING`.
- A **simple PONG** packet consists only of the 1-byte `PacketType` header (`0x01`). Its body is 0 bytes.
  - This is the required response to a simple PING.
  - This is one valid response to a timestamped PING.
- A **full PONG** packet, which can be sent in response to a timestamped PING, includes a 24-byte body structured as defined by `struct Pong` below. If the receiver of the PING (sender of the PONG) chooses this full structure but does not provide actual timing for its own timestamps, these fields (`receiver_receipt_timestamp`, `receiver_transmit_timestamp`) MUST still be present in the packet structure and should be transmitted as zero.

```rust
// Represents the body of a full PONG.
// For a simple PONG, the body is absent.
pub struct Pong {
    pub original_sender_timestamp: u64, // (8 bytes) Mirrored from the timestamped PING
    pub receiver_receipt_timestamp: u64,  // (8 bytes) Time when PING was received (or 0)
    pub receiver_transmit_timestamp: u64, // (8 bytes) Time when PONG is sent (or 0)
}
// Total packet size:
// Simple PONG: 1 byte (header)
// Full PONG: 1 byte (header) + 24 bytes (Pong struct) = 25 bytes
```

---

**6.3.3. `MSG` (Type 2)**

```rust
// Body structure after the 1-byte PacketType header:
pub struct Msg {
    pub message_id: u64,      // (8 bytes)
    pub data: Vec<u8>,        // (remaining bytes)
}
```

---

**6.3.4. `MSG_ACK` (Type 3)**

- Acknowledges successful receipt of `MSG` (Type 2) or `GET_MSG_ACK` (Type 5).

```rust
// Body structure after the 1-byte PacketType header:
pub struct MsgAck {
    pub message_id: u64,      // (8 bytes) ID of message being acknowledged.
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

- Response to `GET_MSG` (Type 4), indicating successful retrieval.
- If the message cannot be retrieved (e.g., not found, access denied), an Axon Relay sends a `NACK` packet.

```rust
// Body structure after the 1-byte PacketType header:
pub struct GetMsgAck {
    pub message_id: u64,      // (8 bytes)
    pub data: Vec<u8>,        // (remaining bytes) Message content.
}
```

---

**6.3.7. `PUT_MSG` (Type 6)**

```rust
// Body structure after the 1-byte PacketType header:
pub struct PutMsg {
    pub idempotency_key: u32, // (4 bytes)
    pub ttl: u32,             // (4 bytes) TTL in seconds
    pub data: Vec<u8>,        // (remaining bytes)
}
```

---

**6.3.8. `PUT_MSG_ACK` (Type 7)**

- Response to `PUT_MSG` (Type 6), indicating successful storage.
- If the message cannot be stored (e.g., idempotency violation, storage error, access denied), an Axon Relay sends a `NACK` packet.

```rust
// Body structure after the 1-byte PacketType header:
pub struct PutMsgAck {
    pub idempotency_key: u32, // (4 bytes) Mirrored from PUT_MSG
    pub ttl: u32,             // (4 bytes) Honored TTL in seconds
    pub message_id: u64,      // (8 bytes) Axon Relay-assigned Snowflake ID.
}
```

---

**6.3.9. `LIST_MSG` (Type 8)**

```rust
// Body structure after the 1-byte PacketType header:
pub struct ListMsg {
    pub limit: u16,           // (2 bytes) Maximum number of message IDs to return.
                              // A limit of 0 is valid and will result in an empty list.
    pub from: u64,            // (8 bytes) Exclusive 'from' cursor.
    pub to: u64,              // (8 bytes) Exclusive 'to' cursor.
}
```

- **Cursor Behavior (`from`, `to`):**
  - These fields act as exclusive cursors for pagination.
  - If `from < to`, the Axon Relay lists messages with IDs `> from` and `< to` in ascending order, up to `limit`.
  - If `from > to`, the Axon Relay lists messages with IDs `< from` and `> to` in descending order, up to `limit`.
  - If `from == to` (or if `limit` is 0, or if no messages fall within the valid exclusive range), no messages are returned (Axon Relay sends an empty `LIST_MSG_ACK`).
- **Special Cursor Values for `from` and `to`:**
  - **Full `message_id`:** A valid Snowflake ID.
  - **Timestamp-based:** A `u64` where the most significant bits represent a desired timestamp (matching the Snowflake timestamp portion), and the remaining less significant bits are zero-filled. This allows querying for messages around a specific time.
  - **`0` (all bits zero):** Represents the beginning of time / the very first possible record (when used appropriately as `from` or `to` relative to the other cursor and ordering).
  - **`u64::MAX` (all bits one):** Represents the end of time / the very last possible record (when used appropriately as `from` or `to` relative to the other cursor and ordering).

---

**6.3.10. `LIST_MSG_ACK` (Type 9)**

- Response to `LIST_MSG` (Type 8). Always indicates successful processing of the list request, even if the list is empty.
- If messages are found matching the criteria and `limit > 0`, the body contains an array of `message_id`s.
- If no messages match the `LIST_MSG` criteria (e.g., the range is empty, `from == to`, `limit == 0`), the Axon Relay **MUST** send a `LIST_MSG_ACK` packet consisting only of the 1-byte `PacketType` header (`0x09`). The body will be 0 bytes in this case.
- If the Axon Relay encounters an issue preventing the list operation itself (e.g., storage error, fundamental parameter error beyond yielding an empty list, authorization issues), it sends a `NACK` packet.

```rust
// Body structure after the 1-byte PacketType header IF message_ids are present:
pub struct ListMsgAck {
    pub message_ids: Vec<u64>,    // (n * 8 bytes) Array of message IDs.
}
// If message_ids is empty (due to limit:0, from==to, or no matching messages),
// the packet on the wire is just the 1-byte header.
```

---

**6.3.11. `NACK` (Type 255 / `0xFF`)**

```rust
// Body structure after the 1-byte PacketType header:
pub struct Nack {
    pub original_packet_type: u8, // (1 byte)
    pub error_code: u8,           // (1 byte)
    pub correlation_data: Vec<u8>, // (Variable bytes) Optional.
}
```

- **Connection Handling with NACK:**
  - If `original_packet_type == 0xFF` AND (`error_code == 0x00` OR `error_code == 0xFF`), the sender **MUST** close its connection end after sending. The receiver **MUST** close its connection end after processing.
  - Other `NACK`s report recoverable errors and **MUST NOT** cause protocol-mandated connection closure.

---

## 7. Why This Default Wire Format?

The choice to define this specific custom binary wire format, rather than mandating an existing generic format, is driven by the primary goal of **minimizing the effort required to implement the protocol's encoding/decoding layer**. This allows developers to focus more on the overall system design, application logic, and operational management of Clients and Axon Relays. While the wire format is a critical piece, it's understood to be only part of the larger puzzle. The aim here is to provide a clear, simple, and efficient default, enabling quicker progress on other core system components.

- **Comparison to Generic Formats:**

  - **ASN.1, Protocol Buffers, Thrift, Avro:** Offer rich features and tooling but can introduce complexity, dependencies, and overhead not strictly necessary for this protocol's focused scope.
  - **JSON, BSON:** While widely supported, text-based parsing (JSON) or even structured binary (BSON with field names) can be less performant and more complex to implement efficiently from scratch compared to this direct binary mapping.

- **Advantages of this Custom Default Format:**
  - **Simplicity:** Fixed-offset fields for headers and reliance on the transport layer for overall packet sizing make parsing and generation straightforward.
  - **Minimal Dependencies:** Can be implemented with basic byte manipulation, without external libraries or code generators for the core encoding.
  - **Efficiency:** Compact representation with no redundant metadata (like field names) in the packets.
  - **Flexibility in Payload:** The `data[]` fields are opaque byte arrays. End-Clients are free to choose any format for the actual message body (e.g., plain text, JSON, custom binary), allowing application-specific flexibility where it matters most. The Axon Relay typically treats this payload as an opaque blob.

While this custom binary format is recommended for its alignment with the Axon Protocol's design goals, the "Protocol Versioning and Handshake" mechanism acknowledges that implementations might choose alternative encodings if agreed upon by Clients and Axon Relays, provided the logical operations and data fields are preserved.
