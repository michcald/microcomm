# MicroComm (microcomm) Protocol for Microcontrollers - Specification v3.0

This document defines the **microcomm** protocol, a modular, hardware-agnostic communication stack specifically optimized for resource-constrained microcontrollers. It prioritizes memory efficiency (Zero-Copy), low latency, and support for both atomic transactions and continuous data streams on embedded hardware.

## Design Philosophy & Constraints
**microcomm** is designed for **reliability and low-traffic control systems**, not high-throughput data pipes.
*   **Sequential & Exclusive**: Servers handle only one client at a time to minimize RAM usage (static allocation).
*   **Fail-Fast**: It is better to drop a session quickly than to block the channel with indefinite retries.
*   **Resource Priority**: Reliability > Memory Efficiency > Throughput.

---

## 0. Global Configuration
The stack is customized at compile-time using the following parameters:

| Parameter | Type | Description |
| :--- | :--- | :--- |
| `PHYS_MTU` | `size_t` | The hardware packet limit (e.g., 32 for nRF24, 255 for LoRa, 1024 for Serial). |
| `ADDR_TYPE` | `type` | `uint8_t` (255 nodes) or `uint16_t` (65k nodes). |
| `L2_INTEGRITY` | `policy` | `CRC16`, `CRC32`, or `NONE` (No-Op). |
| `L4_CRYPTO` | `policy` | Custom implementation overhead (adds X bytes for IV/Tags). |
| `L5_RELIABLE` | `policy` | Defines retry limits and ACK timing. |
| `SESSION_TIMEOUT`| `ms` | Time before an inactive session is forcibly unlocked (default: 2000ms). |
| `ENDIANNESS` | `const` | **Little Endian** is mandatory for all multi-byte fields (CRC, Counters, etc.). |

## 1. Reserved Services
To facilitate discovery and network management, the following Service IDs are reserved at Layer 7:
*   `0x00`: **Discovery / Ping**. Returns device status, type, and capabilities.
*   `0xFF`: **System Reset / Bootloader**. (Optional) Triggers a remote reset.

---

## Layer 1: Physical Layer
The hardware-specific interface responsible for raw buffer transmission.
* **Requirements**: Must support sending/receiving a buffer of exactly `PHYS_MTU`.
* **Mediums**: nRF24L01, UART, LoRa, RS-485, Infrared, Laser, etc.

---

## Layer 2: Integrity Layer
Ensures data has not been corrupted during transit.
* **Standard**: Appends a Checksum (CRC16/32) to the end of the packet.
* **No-Op**: If the hardware (L1) provides guaranteed integrity, this layer is disabled to reclaim bytes.

---

## Layer 3: Addressing Layer
Provides source and destination filtering.
* **Header**: `[SourceAddress][DestinationAddress]`
* **Size**: $2 \times sizeof(ADDR\_TYPE)$
* **Broadcast**: `0xFF` or `0xFFFF` is reserved to target all nodes.

---

## Layer 4: Security Layer (Pluggable)
Provides confidentiality and authenticity. 
* **Customizable**: Developer-provided `encrypt()` and `decrypt()` functions.
* **Overhead**: Any cryptographic overhead (Initialization Vectors or Auth Tags) reduces the available MTU.
* **Context Reset**: Stateful ciphers (e.g., using rolling nonces) must be reset or re-keyed whenever a Session Lock is released to prevent sync errors.

---

## Layer 5: Reliability Layer
Guarantees packet delivery and ensures idempotency (prevents duplicate processing).
* **Header**: 1 byte (`[Type:2bits][PacketID:6bits]`)
* **Packet Types**:
    * `00`: **DATA** - Standard payload.
    * `01`: **ACK** - Positive Acknowledgement.
    * `10`: **NACK** - Negative Acknowledgement (e.g., CRC OK, but Buffer Full / Invalid State).
    * `11`: **BUSY** - Receiver is locked by another session.
* **ACK Logic**: An `ACK` is sent immediately upon valid processing. A `NACK` is sent if the packet is technically valid (CRC matches) but cannot be accepted by the upper layers.
* **Backpressure**: The sender retries on timeout. If it receives `BUSY` or `NACK`, it may abort or wait depending on the policy.
* **Head-of-Line Blocking**: To preserve channel availability, `MAX_RETRIES` should be kept low. A failed reliable transmission should terminate the L7 session immediately.

---

## Layer 6: Fragmentation & Sequencing Layer
Handles the transport of payloads larger than the available MTU.
* **Header**: 2 bytes (`[Index:8bits][Control:8bits]`)
* **Control Bitmask**:
    * `0x01` (**IsLast**): Set if this is the final fragment of a message or stream.
    * `0x02` (**IsStream**): Set for Continuous Streaming mode (bypasses reassembly buffer).
    * `0x04-0x80`: Reserved.
* **State Hygiene**: The reassembly buffer and expected index pointers must be aggressively sanitized/zeroed whenever a session ends or times out.

---

## Layer 7: Session & Application Layer
The top-level developer interface. Manages exclusive access (Locking) and Service routing.

### Interaction Patterns

#### 1. Atomic (Request/Response)
The stack reassembles all fragments into a single buffer before notifying the application.
* **Best for**: Commands, settings, and small status updates.
* **API Pattern**: `TheStack.request(target, serviceID, payload, length)`

#### 2. Streaming (Continuous)
Fragments are passed to the developer's callback immediately upon arrival. No reassembly buffer is used.
* **Best for**: Logs, telemetry, audio, and large file transfers.
* **API Pattern**: `TheStack.openStream(target, serviceID)` -> Returns a `Stream` object.

---

## Session Management & Safety
This protocol relies on a "Single Active Session" model to minimize RAM usage. This introduces critical safety requirements:

1. **Handshake**: A session begins with a `SESSION_START` packet identifying the Service and Pattern.
2. **Locking**: The Server locks itself to the Client's address. Other clients receive a `BUSY` response.
3. **Watchdog (Zombie Protection)**: 
    *   Since the server ignores other clients while locked, a client crashing mid-session acts as a Denial-of-Service.
    *   The `SESSION_TIMEOUT` must be checked via interrupts or every loop cycle, not just on packet arrival.
4. **Zero-Copy**: Data is processed in-place. The application receives a pointer to the hardware buffer to avoid RAM-heavy copying.
5. **Fail-Fast**: If a timeout or retry limit is reached, the session is immediately invalidated, the lock is released, and all L4/L6 state is wiped to prepare for the next client.

---

## Payload Calculation
The maximum available data space for the developer ($P_{dev}$) per packet is:
$$P_{dev} = PHYS\_MTU - (L2_{hdr} + L3_{hdr} + L4_{ovr} + L5_{hdr} + L6_{hdr})$$

### Typical Benchmarks (32-byte MTU)
| Mode | Overhead | Available Payload |
| :--- | :--- | :--- |
| **Minimal Wired** (No L2/L5) | 4 bytes | 28 bytes |
| **Standard Wireless** (CRC16 + L5) | 7 bytes | 25 bytes |
| **Secure Wireless** (AES-GCM + L5) | 23 bytes | 9 bytes |

---

## Detailed Packet Structure (On-the-Wire)

This diagram illustrates the full overhead for a **Reliable Data Packet** (L5 + L6 + L7).

```
[ L1 PHYS ... ] 
      |
      +-- [ L2 Integrity (Optional 2 bytes) ]
            |
            +-- [ L3 Addressing (2 bytes) ]
                  |
                  +-- [ L4 Security (Optional IV/Tag) ]
                        |
                        +-- [ L5 Reliability (1 byte) ]
                        |     [ Type:2 | ID:6 ]
                        |
                        +-- [ L6 Fragmentation (2 bytes) ]
                              |     [ Index:8 | Control:8 ]
                              |
                              +-- [ L7 Payload (Variable) ]
```

### Layer 7 Header Specification
To ensure interoperability, the first payload byte of **Index 0** (the first fragment) is reserved for the **L7 Header**.

*   **Structure**: `[ ServiceID: 7 bits | CMD: 1 bit ]`
    *   **CMD Bit (0)**: `REQUEST` - Client is asking for data/action.
    *   **CMD Bit (1)**: `RESPONSE` - Server is replying.
    *   **Service ID**: The target endpoint (0x00-0x7F).

### Layer 6 Stream Behavior
*   **Atomic Mode**: `Index` MUST NOT wrap. Maximum message size is `255 * Payload_Size`.
*   **Streaming Mode**: `Index` SHOULD wrap from `255` -> `0`. This allows for infinite streams (e.g., firmware updates). The receiver tracks continuity locally.

### Layer 5 Session Hygiene
*   **Packet ID Randomization**: When starting a **new** session (sending the first `Request`), the Client MUST randomize the initial `PacketID` (or increment it by a large step) to prevent the Server from mistaking it for a retry of a previous session.