# AA/BB Protocol Specification

## Overview
The X10 Phantom device uses a proprietary communication protocol over Bluetooth (SP0 or GATT). The protocol is structured into two distinct layers of framing and application-level commands.

## Framing Layers

### 1. Native Layer (Framing)
The Android native layer manages the physical byte stream. It identifies boundaries using specific delimiter bytes:
* **`0xBB`**: Identifies a "BB" packet/stream (typically telemetry, device responses, or multi-packet streams).
* **`0xFF`**: Identifies a "GAIA" protocol packet (used for OTA updates; not observed in this session).

The native layer is responsible for "decapsulating" the stream by splitting large incoming buffers into individual packets delimited by `0xBB`.

### 2. Application Layer (Command Construction)
The Flutter layer constructs specific commands for the device using an "**AA**" framing pattern:
* **Pattern**: `0xAA [Command ID] [Payload Length/Params] 0xAA`
* **Structure**:
    - Header: `0xAA`
    - Command ID: A single byte identifying the action.
    - Payload: Variable length, containing parameters (e.g., SN, keys).
    - Footer: `0xAA`

---

## AA Protocol Commands (App $\to$ Device)

| Command ID (Hex) | Description | Observed Response/Behavior / Payload Structure |
| :--- | :--- | :--- |
| `0x00` | Version Check | **Response**: Returns product name and version string. <br> *Example*: `BB 00 ... aa BB` $\to$ `BB 00 12 58...` (Version 1.9.0) |
| `0x02` | Get Device Info | **Response**: Returns hardware details including Serial Number (SN) and MAC address. <br> *Example*: `BB 02 10 ...` contains MAC/SN. |
| `0x0C` | Get Model Name | **Response**: Returns the model string (e.g., `X10 Phantom`). |
| `0x0E` | Extended Product Info | **Response**: Returns extended product name/info. <br> *Example*: `BB 0E 0B ...` $\to$ contains "X10 Phantom". |
| `0x0F` | Detailed Build Info | **Response**: Returns firmware/build details (e.g., `3.C.U`). |
| `0x10` | Device Type Query | Indicates if the device is an "Android SPP" type device. |
| `0x24` | [Unknown - High-level] | **Response Pattern**: Returns length/data based on hardware flag (e.g., `BB 24 00 00 BB`). |
| `0x28` | Status Check / Heartbeat| **Response**: Returns status/flag byte (e.g., `FF`). |
| `0x29` | Parameter Query | **Response**: Returns a parameter check value (`BB 29 01 FF 00 BB`). |
| `0x38` | Hardware Version Check | Checks for specific hardware revisions. |
| `0x3E` | Unknown Command / Test | Triggers a response containing raw diagnostic hex data (e.g., `0x61, 0xCA...`). |
| `0x3F` | Timer/Timeout Command | Returns a timeout or duration value (e.g., `BB 3F 01 0A 00 BB` $\to$ `10` dec). |
| `0x54` | Hardware Capability Query| Returns hardware capability flags. |
| `0x5A` | Periodic Pulse/Status | **Response**: Periodic telemetry pulse (e.g., `BB 5A 0x01 0x05 0x00 BB`). |
| `0x5C` | Parameter Write/Set | Sets a specific device parameter value. |
| `0x5E` | Command Status Check | Checks if previous command was successful. |
| `0x60` | Service Query | Returns service flags for the connected device. |
| `0x61` | Hardware Feature Bitmask | Returns bitmask representing hardware features (e.t., Audio/Camera). |
| `0x69` | Complex Config/Capability | Transmits multi-byte configuration blobs or feature lists. |
| `0xF0` | Firmware Verification | Used during boot or connectivity handshake for firmware integrity. |
| `0xF3` | Extended Name Query | Returns long version of the device name (e.g., `X10 PhantomB, 1.9.0`). |

---

## BB Protocol Responses & Telemetry (Device $\to$ App)

The **BB** stream contains incoming telemetry and deserialized responses. Each "packet" is bounded by `0xBB`.

### Found Packet Structures (Extracted from `ble.log`):

| Command ID/Pattern | Byte Breakdown (Hex) | Interpretation / Payload Content |
| :--- | :--- | :--- |
| **Version String** | `BB 00 12 58 ... 00 BB` | Header: `BB`, Length: `12` (hex), Payload: ASCII for "X10 PhantomB, 1.9.0", Footer: `00` (internal footer). |
| **Device Info** | `BB 02 10 ... 00 BB` | Header: `BB`, Length: `10`, Payload contains MAC/SN bits + repeats the address at end. |
| **Telemetry Pulse** | `BB 5A 01 05 00 BB` | Periodic heartbeat containing device stability indicator (`0x05`). |
| **Hardware Info** | `BB 61 01 00 00 BB` | Returns a bitmask for features like Camera or Audio. |
| **Unknown/Raw Data** | `BB 3E 01 FF 00 BB` | Generic hardware status response containing `FF`. |

## Protocol Summary (Final)

The protocol specification has been enriched with real-time transaction data captured during a live session. 

#### **1. Expanded Command Map (App $\to$ Device)**
Based on the instruction traces in `bluetooth_msg_tool.dart` and `bluetooth_msg_tool_old.dart`, I have identified several previously unmapped command IDs:

| Command ID | Context/Description | Payload Structure / Observed Trace |
| :--- | :--- | :--- |
| **`0x00`** | Version Check | `AA 00 00 00 AA` $\to$ Returns ASCII version string. |
| **`0x02`** | Get Device Info | `AA 02 00 00 AA` $\to$ Returns MAC/SN payload. |
| **`0x0C`** | Get Model Name | `AA 0C 00 00 AA` $\to$ Returns model string. |
| **`0x1E`** | Device Status Pulse | `AA 1E ...` $\to$ Triggers parsing for battery/connectivity status updates. |
| **`0x29`** | Parameter Query | `AA 29 00 00 AA` $\to$ Response observed: `BB 29 01 FF 00 BB`. |
| **`0x3E`** | Diagnostic Trace | Returns raw hex dumps (e.g., `0x61, 0xCA...`) for testing. |
| **`0x5A`** | Periodic Heartbeat | `AA 5A ...` $\to$ Triggers reactive updates to telemetry models. |

#### **2. Device Response Analysis (Device $\to$ App)**
Using `ble.log`, I have performed a byte-level mapping of the **BB-deliminted** responses:

| Packet ID (Hex) | Byte Breakdown (Hex) | Interpretation / Payload Content |
| :--- | :--- | :--- |
| **`0x0C_RESP`** | `BB 0C 08 [8 bytes] 00 BB` | Header, Len=8, Serial Data... Footer. |
| **`0x0F_RESP`** | `BB 0F 06 [ASCII...] 00 BB` | Returns firmware version (e.g., `31.2E.3C.2E.55`). |
| **`0x47_RESP`** | `BB 47 01 01 00 BB` | Boolean-type response for pairing/presence state. |
| **`0x61_RESP`** | `BB 61 01 00 00 BB` | Feature bitmask return (e.g., Camera/Audio status). |

#### **3. Decapsulation Logic**
The application utilizes the following logic for parsing:
- **Incoming**: `[BB] [Len] [Data...] [BB]`
- **Outbound**: `[AA] [CmdID] [Payload...] [AA]` (where payload contents vary by command length).

## Inferred/Potential Command Scopes (Unverified)

Based on the application's structure, certain advanced features like **FM Mode** or **MESH Networking** are likely managed through existing command channels rather than standalone IDs. 

### 1. Feature Activation via Parameter Write (`0x5C`)
It is highly probable that switching modes (e.g., toggling FM frequency scanning or M_ESH topology) is performed by sending a byte payload to the `0x5C` command. This would involve:
*   **Structure**: `0xAA [0x5C] [Mode_Byte] [Freq/Param_Bytes] 0xAA`
*   **Example Inference**: `0xAA 5C 01 ... 0xAA` (where `0x01` might represent the FM-enable bit).

### 2. Hardware Feature Bitmask (`0x61`)
The app already implements a parser for Command `0x61`, which reads bits to determine if features like **Camera** or **Audio** are present. It is architecturally consistent that "MESH" status or "FM capability" would be encoded as a bit in this same response stream:
*   **Structure**: `0xBB 61 [Bitmask_Byte] 0x00 0xBB`
*   **Example Inference**: A bitwise `AND` operation on the received byte determines if MESH functionality is active.

### 3. Contextual Re-use of Command IDs
Observations in legacy code (`bluetooth_msg_tool_old.dart`) suggest that some IDs (like `0x02`) may serve different roles depending on the communication direction:
*   **App $\to$ Device**: Triggering a request (e.g., "Get Info").
*   **Device $\to$ App**: Reporting a state change (e.g., "Power State Change").

---

## Protocol Implementation Details

-   **Handling Logic**: The application uses `BleMsgTool::rxMessage` to iterate through the incoming `BB` stream, identifying individual packet segments.
-   **Integrity Checks**: For "AA" commands, the app validates that the payload length matches the header/footer structure. For bytes in the middle of a sequence (e_g., `0x2D, 0x01...`), it may perform bitwise masking to determine features like `isCan47`.
-   **Data Formats**:
    -   **ASCII**: Strings for names and firmware versions (`X10 Phantom`).
    -   **Hex/Binary**: MAC addresses (e.g., `00:12:6F...`) and Serial Numbers.
    -   **Integer/Flag**: Bit-packed bytes representing hardware capabilities or feature support.
