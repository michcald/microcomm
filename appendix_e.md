# Appendix E: CRC Implementation Reference

This appendix provides the technical specifications and test vectors for **Layer 2 (Integrity)**. Using these standard parameters ensures that different `microcomm` implementations (e.g., C, Go, Python) can verify each other's data.

## 1. CRC16 (Standard)
The default integrity check for `microcomm`. It provides a good balance between safety and computational overhead for 8/16-bit MCUs.

*   **Polynomial**: `0x1021` (CCITT-FALSE)
*   **Initial Value**: `0xFFFF`
*   **Final XOR**: `0x0000`
*   **Input Reflected**: No
*   **Result Reflected**: No
*   **Byte Order**: **Little-Endian** (LSB first on the wire).

### CRC16 Test Vectors
| Input (String) | Input (Hex) | Expected CRC16 (Hex) |
| :--- | :--- | :--- |
| `"123456789"` | `31 32 33 34 35 36 37 38 39` | `0x29B1` |
| `"microcomm"` | `6D 69 63 72 6F 63 6F 6D 6D` | `0x6F9F` |

---

## 2. CRC32 (Advanced)
Recommended for high-reliability systems or nodes with 32-bit hardware CRC accelerators (e.g., STM32).

*   **Polynomial**: `0x04C11DB7` (Ethernet / MPEG-2)
*   **Initial Value**: `0xFFFFFFFF`
*   **Final XOR**: `0x00000000`
*   **Input Reflected**: No
*   **Result Reflected**: No
*   **Byte Order**: **Little-Endian**.

### CRC32 Test Vectors
| Input (String) | Expected CRC32 (Hex) |
| :--- | :--- |
| `"123456789"` | `0x0376E6E7` |
| `"microcomm"` | `0x16545B3F` |

---

## 3. Implementation Logic (Zero-Allocation)
In languages like **Go/TinyGo**, avoid creating new objects. Use a rolling CRC calculation or a static lookup table.

```go
// Example TinyGo CRC16 (CCITT-FALSE)
func UpdateCRC16(crc uint16, data []byte) uint16 {
    for _, b := range data {
        crc ^= uint16(b) << 8
        for i := 0; i < 8; i++ {
            if crc&0x8000 != 0 {
                crc = (crc << 1) ^ 0x1021
            } else {
                crc <<= 1
            }
        }
    }
    return crc
}
```
