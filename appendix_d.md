# Appendix D: Session State Machine

This appendix defines the lifecycle of a **microcomm** session. Implementing this state machine ensures that the server (node) correctly manages client locks, prevents memory leaks during fragmentation, and recovers gracefully from communication failures.

## Server-Side Session State

The server follows a "Single Active Session" model. It locks itself to a specific Client Address until the transaction is complete or a timeout occurs.

```mermaid
stateDiagram-v2
    [*] --> IDLE
    
    IDLE --> LOCKED : Recv REQUEST (IsStateful=1)
    IDLE --> IDLE : Recv Stateless DATA (Process immediately)

    state LOCKED {
        [*] --> REASSEMBLING
        REASSEMBLING --> REASSEMBLING : Recv Fragment (L6 Index++)
        REASSEMBLING --> NOTIFY_APP : Recv Fragment (L6 IsLast=1)
        
        NOTIFY_APP --> SEND_RESPONSE : App processing complete
        SEND_RESPONSE --> [*] : Response ACK'd by Client
    }

    LOCKED --> IDLE : SUCCESS (Transaction complete)
    LOCKED --> IDLE : FAIL (L5 NACK sent)
    LOCKED --> IDLE : TIMEOUT (No activity for SESSION_TIMEOUT)
    
    Note right of LOCKED: While LOCKED, any REQUEST from<br/>other addresses triggers a L5 BUSY.
```

## State Definitions

| State | Description |
| :--- | :--- |
| **IDLE** | The default state. The node listens for any incoming packet. |
| **LOCKED** | The node has accepted a stateful session. The `Session_Client_Addr` is stored. |
| **REASSEMBLING** | The node is collecting L6 fragments into the `Max_Buffer`. |
| **NOTIFY_APP** | The full message is available. The L8 Service handler is invoked. |
| **SEND_RESPONSE** | The node is transmitting the result back to the client. |
| **TIMEOUT** | A safety transition. If the client stops talking mid-session, the node resets to IDLE. |

## Implementation Notes
*   **Buffer Safety**: Upon transitioning to `IDLE` (regardless of success or failure), the `Max_Buffer` and L6 `Expected_Index` **MUST** be zeroed/reset.
*   **Watchdog**: The `SESSION_TIMEOUT` timer (usually 100ms–500ms) should be checked every loop cycle.
