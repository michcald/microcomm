# Appendix C: Service Discovery & Broadcast Handshake

This appendix provides a visual and technical breakdown of the **microcomm** discovery process. It illustrates how a new node transitions from an unassigned state (`0x00`) to a fully associated network member.

## The Discovery Sequence

The following diagram illustrates the interaction between a new **Client** (e.g., a sensor) and a **Server** (e.g., a Hub).

```mermaid
sequenceDiagram
    participant C as New Client (Addr: 0x00)
    participant S as Hub Server (Addr: 0x02)

    Note over C: Startup Jitter (Random delay)
    
    Note over C,S: 1. SEARCH (Broadcast)
    C->>S: L3:Src=0x00, Dst=0xFF | L8:Service=0x00, Op=0x01 (SEARCH)
    
    Note over S: Validate UID & Capabilities
    Note over S: Response Jitter (Random 10-100ms)

    Note over C,S: 2. OFFER (Directed to Unassigned)
    S->>C: L3:Src=0x02, Dst=0x00 | L8:Service=0x00, Op=0x02 (OFFER)
    Note right of S: Payload contains Assigned Address (e.g., 0x05)

    Note over C: Verify Target UID
    Note over C: Update L3 Address to 0x05
    
    Note over C,S: 3. ASSOCIATION (First Directed Data)
    C->>S: L3:Src=0x05, Dst=0x02 | L8:Service=0x00, Op=0xFE (HEARTBEAT)
    S-->>C: L5: ACK
    
    Note over C,S: 4. CAPABILITY SYNC (Optional)
    C->>S: L3:Src=0x05, Dst=0x02 | L8:Service=0x00, Op=0x03 (GET_SERVICES)
    S->>C: L8:Service=0x00, Op=0x04 (SERVICES_LIST)
```

## Key Transitions

### 1. The Broadcast Phase (L3: 0xFF / 0x00)
*   **Search**: The client uses `0xFF` as the destination. Every node hears this, but only a Server (Hub) processes Service `0x00`.
*   **Offer**: The server responds to `0x00`. Since multiple nodes might be searching, the server includes the **Client's UID** in the payload. Every searching node reads the packet, but only the one with the matching UID accepts the `Assigned Address`.
*   **Reliability**: Layer 5 ACKs are **DISABLED** during this phase to prevent radio collisions.

### 2. The Directed Phase (L3: Assigned Address)
*   Once the client receives its `Assigned Address` (e.g., `0x05`), it reconfigures its **Layer 1 Driver** (e.g., updating nRF24 Pipe 0).
*   All subsequent communication uses the Assigned Address.
*   **Reliability**: Layer 5 ACKs are now **ENABLED**, ensuring reliable command delivery.

## Anti-Collision Mechanisms
*   **Startup Jitter**: Prevents a "Thundering Herd" if multiple sensors are powered on simultaneously (e.g., after a power outage).
*   **Response Jitter**: If multiple Hubs exist, this prevents their `OFFER` packets from colliding in the air.
*   **Target UID Echo**: Acts as a software-level filter to ensure the `OFFER` is only acted upon by the correct physical device.
