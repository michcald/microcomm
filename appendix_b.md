# Appendix B: nRF24L01+ Implementation Guide for microcomm

This document describes how to implement the **microcomm** Layer 1 (Physical) driver using the nRF24L01+ radio. It leverages the chip's hardware features (Multiceiver, Auto-ACK, Dynamic Payload) to ensure maximum efficiency and zero-allocation performance in TinyGo.

## 1. Hardware Mapping (L1-L3)
The nRF24 uses 5-byte physical addresses. `microcomm` uses a 1-byte logical address.
*   **Base Address**: All nodes on a network MUST share the same 4-byte prefix (e.g., `0xE7E7E7E7`).
*   **Suffix**: The 5th byte of the nRF24 address is the **Logical L3 Address**.

| microcomm Addr | nRF24 HW Address | Usage |
| :--- | :--- | :--- |
| `0x01` | `0xE7E7E7E701` | Private Node Address |
| `0x00` | `0xE7E7E7E700` | Unassigned/Discovery Address |
| `0xFF` | `0xE7E7E7E7FF` | Global Broadcast Address |

---

## 2. Multiceiver & Pipe Configuration (The "Mailbox" Strategy)
To support simultaneous Private and Broadcast reception without CPU overhead, the driver MUST configure two hardware pipes:

### Pipe 0: Private / Direct (L3 Address)
*   **Address**: `Base_Prefix` + `Node_Logical_Addr`
*   **Auto-ACK**: **ENABLED** (`EN_AA` register).
*   **Behavior**: When the Hub sends a directed packet, the nRF24 hardware handles the retry logic and sends a hardware ACK immediately. The MCU only wakes up if the packet is valid and intended specifically for this node.

### Pipe 1: Global Broadcast (0xFF)
*   **Address**: `Base_Prefix` + `0xFF`
*   **Auto-ACK**: **DISABLED** (`EN_AA` register).
*   **Behavior**: Every node listens to this shared pipe. When a broadcast is sent, all nodes receive it simultaneously. Because Auto-ACK is disabled on this pipe, nodes stay silent, preventing an "ACK Storm" (radio collision).

---

## 3. Transmission Logic (The Sender)
The `Send(dest uint8, payload []byte)` function must switch hardware modes based on the destination:

1.  **Direct (dest != 0xFF)**:
    *   Set TX Address to `Base_Prefix + dest`.
    *   Use the standard `W_TX_PAYLOAD` command.
    *   The hardware will wait for an ACK and retry automatically (up to 15 times).

2.  **Broadcast (dest == 0xFF)**:
    *   Set TX Address to `Base_Prefix + 0xFF`.
    *   Use the **`W_TX_PAYLOAD_NOACK`** SPI command.
    *   The radio transmits once and immediately returns to standby.

---

## 4. Hardware Feature Requirements
To remain compliant with the `microcomm` stack, the following nRF24 features MUST be enabled:

*   **EN_DPL (Dynamic Payload Length)**: Allows packets smaller than 32 bytes to be sent without padding. This saves battery and airtime.
*   **EN_ACK_PAY (Optional)**: If the response is very small (1-2 bytes), it can be piggybacked on the hardware ACK for extreme latency reduction.
*   **CRC Config**: The nRF24 hardware CRC (usually 2-byte) should be enabled. If enabled, the `microcomm` **Layer 2 (Integrity)** can be set to **Transparent** to save 2 bytes of payload.

---

## 5. Implementation Structure (TinyGo)
To ensure zero-allocation, the driver should avoid `make()` or `new()` in the hot-path.

```go
type NRFDriver struct {
    spi      drivers.SPI
    cePin    machine.Pin
    irqPin   machine.Pin
    address  uint8
    prefix   [4]byte
    rxBuffer [32]byte // Fixed-size MTU buffer
}

// Receive checks the hardware status and fills the pre-allocated buffer.
func (d *NRFDriver) Receive(out []byte) (n int, isBroadcast bool, err error) {
    status := d.readStatus()
    pipe := (status >> 1) & 0x07

    if pipe > 1 { return 0, false, ErrNotForUs }
    
    n = d.readPayload(out)
    isBroadcast = (pipe == 1) // Pipe 1 is our Broadcast pipe
    return n, isBroadcast, nil
}
```

## 6. Power Management & Beaconing
For battery-powered nodes using the **Client-Pull** model:
1.  **Sleep**: The MCU puts the nRF24 into "Power Down" mode ( < 1µA).
2.  **Wake**: MCU wakes, puts nRF24 into "Standby-I," and sends the **HEARTBEAT (0xFE)** to the Hub.
3.  **Listen Window**: The node enters "RX Mode" on Pipe 0 and Pipe 1 for exactly 20ms.
4.  **Process**: If a packet arrives, process it. Otherwise, return to Power Down.

---

## 7. Summary of Register Settings
| Register | Bit | Value | Description |
| :--- | :--- | :--- | :--- |
| `CONFIG` | `EN_CRC` | `1` | Enable HW CRC. |
| `EN_AA` | `ENAA_P0` | `1` | Auto-ACK for Private Address. |
| `EN_AA` | `ENAA_P1` | `0` | **Disable Auto-ACK for Broadcast.** |
| `EN_RXADDR` | `ERX_P0/P1`| `1` | Enable Pipes 0 and 1. |
| `FEATURE` | `EN_DYN_ACK`| `1` | Allow `NO_ACK` command. |
| `FEATURE` | `EN_DPL` | `1` | Enable Dynamic Payload Length. |
