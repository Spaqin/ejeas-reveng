# Protocol Specification: X10 Phantom Bluetooth Interface

## 1. Communication Architecture

### 1.1 Layer A: Command/Instruction (Outbound/Inbound)
**Frame Structure:** `[0xAA] [Command ID] [Payload Length/Params] [0xAA]`
*   **`0xAA`**: Start-of-Frame (SOF) delimiter.
*   **`Command ID`**: Single byte identifying the instruction.
*   **`Payload`**: Variable length (Sub-IDs, High/Low bytes, or Blobs).
*   **`0xAA`**: End-of-Frame (EOF) delimiter.

### 1.2 Layer B: Status/Response (Device-to-App)
**Frame Structure:** `[0xBB] [Command ID] [Payload] [0xBB]`
*   **`0xBB`**: Start-of-Frame (SOF) delimiter.
*   **`Command ID`**: Corresponds to the initiating Layer A Command.
*   **`Payload`**: Resulting data (Strings, MAC, Battery, etc.).
*   **`0xBB`**: End-of-Frame (EOF) delimiter.

---



## 2. Command Reference & Evidence Trace

### 2.1 Core Management
| Command ID | Name | Behavior & Payload Structure | Evidence Trace (Log Snippets) |
| :--- | :--- | :--- | :--- |
| `0x00` | Device Info Request | Returns device name, version, and SN. | `[TX] AA 00 00 00 AA` $\to$ `[RX] BB 00 12 58...00 BB` |
| `0x02` | Hardware Identity | Returns MAC address (redundantly framed) and model. | `[TX] AA 02 00 00 AA` $\to$ `[RX] BB 02 10 01 A1...00 BB` |
| `0x0F` | Version Check | Returns firmware version (e.g., `1.9.0`). | `[TX] AA 0F 01 02 00 AA` $\to$ `[RX] BB 0F 06 02 33.2E...00 BB` |
| `0x10` | Connection Status | Returns hardware adapter/connection state. | `[TX] AA 10 00 00 AA` $\to$ `[RX] BB 10 02 FB 01 00 BB` |
| `0xF0` | Activation Status | Returns `true`/`false` for device activation. | `[RX] BB F0 01 00 00 BB` $\to$ `0xF0 查询到设备激活状态: true` |

### 2.2 Tuning & Parameter Control (FM/Volume/Audio)
| Command ID | Name | Behavior & Payload Structure | Evidence Trace (Log Snippets) |
| :--- | :--- | :--- | :--- |
| **`0x1D`** | **Parameter Write** | **`[Sub-ID] [Val_H] [Val_L]`** <br> Adjusts frequency or audio gain. | `[TX] AA 1D 02 05 0B 00 AA` $\to$ `[RX] BB 1D 00 00 BB` <br> `[TX] AA 1D 02 00 03 00 AA` $\to$ `[RX] BB 1D 00 00 BB` |
| `0x35` | Tuner/Frequency Command | Triggers the frequency search/tuning logic. | `[TX] AA 35 00 00 AA` $\to$ `[RX] BB 35 00 00 BB` |
| `0x3E` | Reset/Clear Command | Resets internal hardware buffers. | `[TX] AA 3E 00 00 AA` $\to$ `[RX] BB 3E 01 FF 00 BB` |

### 2.3 Advanced Features & Discovery
| Command ID | Name | Behavior & Payload Structure | Evidence Trace (Log Snippets) |
| :--- | :--- | :--- | :--- |
| `0x26`/`0x27`| Mesh/Pairing State | High-entropy payload; handles mesh synchronization. | `[RX] BB 26 03 07 0A 06 00 BB` |
| `0x42`/`0x43`| Hardware Capability | Rapid-fire pulses used to query sensor presence. | `[RX] BB 42 01 03 00 BB` $\to$ `[RX] BB 43 01 00 00 BB` |
| `0x58` | Security/Config Blob | Large, high-entropy payload for keys/signatures. | `[RX] BB 58 08 6C 3B D3 C7 89 DE C2 81 00 BB` |
| `0x59` | Status Update | Periodic update for battery and power management. | `[TX] AA 59 00 00 AA` $\to$ `[RX] BB 59 01 70 00 BB` |
| `0x69` | Complex Configuration | Multi-byte configuration transfer for deep settings. | `[TX] AA 69 01 04 00 AA` $\to$ `[RX] BB 69 05 04 00 00 00 13 00 BB` |

---

## 3. Critical Implementation Notes

*   **MAC Redundancy**: Command `0x02` responses contain the MAC address twice (e.g., `...00 12 6F 1C 65 C3 00 12 6F 1C 65 C3 00...`) to verify integrity.
*   **Error Detection**: The application explicitly monitors for `TX Warning: Not have onResponse!`, indicating a dropped packet or timeout.
*   **Payload Parsing**: Parameter command `0x1D` uses a `[Sub-ID][High][Low]` pattern, essential for translating app-side slider/dial movements into device-side hex commands.
