# V5008 Device MQTT Message RAW and SIF Spec

**File Name:** `V5008_Spec.md`

**Version**: v1.3

**Date**: 2/28/2026

**Scope:** V5008 device raw binary message format and Spec from raw to SIF (Standard Intermediate Format) Conversion

---

---

## 1. Parsing Strategy

### 1.1 Message Identification Logic

The parser determines `messageType` using this strict precedence:

1. **Topic Suffix Check:**
    - `.../LabelState` → `RFID_SNAPSHOT`
    - `.../TemHum` → `TEMP_HUM`
    - `.../Noise` → `NOISE_LEVEL`
2. **Header Byte Check (Byte 0):**
    - `0xBA` → `DOOR_STATE`
    - `0xCC` or `0xCB` → `HEARTBEAT`
3. **Extended Header Check (Bytes 0-1):**
    - `0xEF01` → `DEVICE_INFO`
    - `0xEF02` → `MODULE_INFO`
4. **Command Response Check (Header 0xAA):**
    - Read Byte 5 for `0xE4` (QUERY_COLOR_RESP):
        - `0xE4` → `QUERY_COLOR_RESP`
    - Read Byte 6 for other command codes:
        - `0xE1` → `SET_COLOR_RESP`
        - `0xE2` → `CLEAR_ALARM_RESP`

**Note:** All multi-byte fields are Big-Endian.

**Topic Extraction Notation:**
- `Topic[1]` refers to the `{deviceId}` placeholder in the MQTT topic path
- Example: `V5008Upload/{deviceId}/OpeAck` → Extract the second segment as deviceId
- Example: `V5008Upload/2437871205/OpeAck` → deviceId = "2437871205"

---

## 2. Binary Field to SIF Mapping

**CRITICAL:** The parser must map the **Binary Field Name** (from Section 6 Schemas) to the specific **SIF JSON Key**.

| Binary Field Name | Byte Size | SIF JSON Key | Parsing Rule / Data Type |
| --- | --- | --- | --- |
| **Common Fields** |  |  |  |
| `DeviceId` | 4B | `deviceId` | **Context Dependent:**<br>1. Header `AA`: Bytes [1-4] → String.<br>2. Others: Extract from MQTT Topic. |
| `MsgId` | 4B | `messageId` | Last 4 bytes of packet → String. Refer to Algorithm D |
| `ModId` | 4B | `moduleId` | `uint32_be` → String. Refer to Algorithm D |
| `ModAddr` | 1B | `moduleIndex` | `uint8` (Range 1-5). use `moduleIndex: modAddr` directly |
| `Res` | 1B | - | Reserved field. Not used in SIF (discarded by design). |
| **Sensor Indices** |  |  |  |
| `Addr` (Temp) | 1B | `sensorIndex` | `uint8` (Range 10-15). |
| `Addr` (Noise) | 1B | `sensorIndex` | `uint8` (Range 16-18). |
| `uPos` | 1B | `sensorIndex` | `uint8` (Range 1-54). |
| **Values** |  |  |  |
| `Total` | 1B | `uTotal` | `uint8`. |
| `Count` | 1B | `onlineCount` | `uint8`. |
| `Alarm` | 1B | `isAlarm` | `0x00`=false, `0x01`=true. |
| `TagId` | 4B | `tagId` | Hex String (Uppercase). |
| `State` | 1B | `doorState` | `uint8` (0 or 1). |
| `Result` | 1B | `result` | `0xA1`="Success", `0xA0`="Failure". |
| `ColorCode` | 1B | - | Used in `data` array as integer. |
| **Device Meta** |  |  |  |
| `Model` | 2B | `model` | Hex String (Uppercase). |
| `Fw` | 4B | `fwVer` | Binary field name: `Fw` → SIF JSON key: `fwVer`. `uint32_be` → String. Refer to Algorithm D |
| `IP` | 4B | `ip` | Dot-notation String (e.g., "192.168.0.1"). |
| `Mask` | 4B | `mask` | Dot-notation String. |
| `Gw` | 4B | `gwIp` | Dot-notation String. |
| `Mac` | 6B | `mac` | Hex String with colons (e.g., "AA:BB..."). |
| `OriginalReq` | Var | `originalReq` | Hex String. See Algorithm B. |

---

## 3. Special Parsing Algorithms

### Algorithm A: Signed Sensor Values (Temp/Noise)

*Used for fields: `temp`, `hum`, `noise`.Binary Input: [IntegerByte, FractionByte]*

```jsx
function parseSignedFloat(integerByte, fractionByte) {
  // 1. Check Sign Bit (Two's Complement)
  let signedInt = (integerByte & 0x80) ? (0xFF - integerByte + 1) * -1 : integerByte;

  // 2. Combine with Fraction
  // Note: Fraction adds magnitude to the signed base
  let value = signedInt + (Math.sign(signedInt || 1) * (fractionByte / 100));

  return Number(value.toFixed(2));
}
```

### Algorithm B: Dynamic `originalReq` Length

*Used for* `QUERY_COLOR_RESP`*,* `SET_COLOR_RESP`*,* `CLEAR_ALARM_RESP`*.*

```jsx
// Header (AA) is at index 0.
let cmdCode;
let reqLength;
let reqOffset;

// For QUERY_COLOR_RESP: Command Code is at index 5 (first byte of OriginalReq)
// For SET_COLOR_RESP and CLEAR_ALARM_RESP: Command Code is at index 6 (first byte of OriginalReq)
const cmdCode5 = buffer[5];
const cmdCode6 = buffer[6];

if (cmdCode5 === 0xE4) {
    // QUERY_COLOR_RESP: Result is at byte 5, OriginalReq is at bytes 6-7 (after Result)
    cmdCode = 0xE4;
    reqOffset = 5;
    reqLength = 2; // Fixed length for Query Color (E4 + ModAddr)
} else {
    // SET_COLOR_RESP and CLEAR_ALARM_RESP: OriginalReq is at byte 7 (after Result)
    cmdCode = cmdCode6;
    reqOffset = 7;
    // Variable length: Total - Overhead (Header+Id+Result+MsgId)
    // Overhead = 10 bytes (Header:1 + DevId:4 + Result:1 + MsgId:4)
    reqLength = buffer.length - 10;
}
// Read `reqLength` bytes starting at reqOffset -> `originalReq`

// Note: Schema overhead breakdown:
// - QUERY_COLOR_RESP: Header(1) + DevId(4) + Result(1) + OriginalReq(2) + ColorCodes(N) + MsgId(4) = 8 + N bytes total
// - SET_COLOR_RESP: Header(1) + DevId(4) + Result(1) + OriginalReq(var) + MsgId(4) = 10 + var bytes
// - CLEAR_ALARM_RESP: Header(1) + DevId(4) + Result(1) + OriginalReq(3) + MsgId(4) = 13 bytes total
```

### **Algorithm C: Parsing originalReq (Header AA)**

*Goal: Extract the Module Index from the echoed command.*

```jsx
// 1. Determine Req Length and Offset (Algorithm B)
// 2. Extract Buffer slice for originalReq
const reqBuffer = buffer.slice(reqOffset, reqOffset + reqLength);

// 3. Extract Module Index (Byte 1 of the command)
// Example: E4 01 (Query Mod 1) -> 01
const moduleIndex = reqBuffer.readUInt8(1);

// 4. Return both the Hex String and the Index
return { originalReq: reqBuffer.toString('hex').toUpperCase(), moduleIndex };
```
**Note:** Module index is always at byte position 1 of the originalReq buffer, regardless of command type.

---

### **Algorithm D: Parsing 4-byte field** to **String**

Read it as an **unsigned 32-bit integer in Big-Endian format** to get the same decimal value as Windows Calculator. For example, if buffer contains `27 00 DC F6`, it should return `654367990`. Use for  `ModId`, `MsgId` , `Fw`, `fwVer` .

```markdown
// Example: Raw message containing 4 bytes
const message = Buffer.from([0x27, 0x00, 0xDC, 0xF6]);

// Read as Unsigned 32-bit Integer, Big Endian (like Win 11 Calc)
const decimalValue = message.readUInt32BE(0).toString(); 

console.log(decimalValue); // Result: 654367990
```

### **Data Validation Rule:**

If the raw binary value for Temperature, Humidity, or Noise is exactly 0x00 (Zero), map the field to null in the SIF. **Do not** output 0.

## 4. Message Structure Schemas (Binary Layout) and SIF

The parser must iterate through the binary buffer based on these structures to populate the SIF object.

### 4.1 `HEARTBEAT`

- **Topic:** `V5008Upload/{deviceId}/OpeAck`
- **Header:** `0xCC` or `0xCB`
- **Schema:** `Header(1)` + `[ModAddr(1) + ModId(4) + Total(1)] × 10` + `MsgId(4)`
- **Parsing Logic:** Loop 10 times. **Filter out** slots where `ModId == 0` or `ModAddr > 5`.
- **Spec from Raw → SIF**

```jsx
{
  "meta": { "topic": "...", "rawHex": "CC01..." },
  "deviceType": "V5008",
  "deviceId": "Topic[1]",
  "messageType": "HEARTBEAT",
  "messageId": "MsgId",
  "data": [{"moduleIndex": ModAddr , "moduleId": "ModId", "uTotal": Total }]
}

```

### 4.2 `RFID_SNAPSHOT`

- **Topic:** `V5008Upload/{deviceId}/LabelState`
- **Header:** `0xBB`
- **Schema:** `Header(1) + ModAddr(1) + ModId(4) + Res(1) + Total(1) + Count(1)` + `[uPos(1) + Alarm(1) + TagId(4)] × Count` + `MsgId(4)`
- **Spec from Raw → SIF**

```jsx
{
  "meta": { "topic": "...", "rawHex": "BB02..." },
  "deviceType": "V5008",
  "deviceId": "2437871205",
  "messageType": "RFID_SNAPSHOT",
  "messageId": "MsgId",
  "moduleIndex": ModAddr,
  "moduleId": "ModId",
  "data": [{ "sensorIndex": uPos, "isAlarm": false|true, "tagId": "DD344A44" }]
}

// Note:
// Fields discarded from raw to SIF:
//   - Res: Reserved byte not used in SIF (discarded by design)
//
// Field transformations:
//   - Topic[1] → deviceId
//   - MsgId → messageId
//   - ModAddr → moduleIndex
//   - ModId → moduleId
//   - Total → uTotal
//   - uPos → data[].sensorIndex
//   - TagId → data[].tagId
//
// isAlarm field creation logic:
//   - Alarm = 1 → isAlarm = true
//   - Alarm = 0 → isAlarm = false
```

### 4.3 `TEMP_HUM`

- **Topic:** `V5008Upload/{deviceId}/TemHum`
- **Header:** None (identified by topic suffix only)
- **Schema:** `ModAddr(1) + ModId(4)` + `[Addr(1) + T_Int(1) + T_Frac(1) + H_Int(1) + H_Frac(1)] × 6` + `MsgId(4)`
- **Note:** Fixed 6 slots. If `Addr === 0`, skip. Use Algorithm A for values.
- **Spec from Raw → SIF**

```jsx
{
  "meta": { "topic": "...", "rawHex": "01EC..." },
  "deviceType": "V5008",
  "deviceId": "Topic[1]",
  "messageType": "TEMP_HUM",
  "messageId": "MsgId",
  "moduleIndex": ModAddr,
  "moduleId": "ModId",
  "data": [{ "sensorIndex": Addr, "temp": xx.xx, "hum": xx.xx }]
}

// Note:
// Fields discarded from raw to SIF:
//   - Res: Reserved byte not used in SIF (discarded by design)
//
// Field transformations:
//   - Topic[1] → deviceId
//   - MsgId → messageId
//   - ModAddr → moduleIndex
//   - ModId → moduleId
//   - Addr → data[].sensorIndex
//
// Parsing logic:
//   - Fixed 6 slots
//   - If Addr === 0, skip (no sensor data)
//   - Use Algorithm A for T_Int/T_Frac and H_Int/H_Frac values
//
// Formatting:
//   - Format temp and hum to float "xx.xx" (Algorithm A)
```

### 4.4 `NOISE_LEVEL`

- **Topic:** `V5008Upload/{deviceId}/Noise`
- **Header:** None (identified by topic suffix only)
- **Schema:** `ModAddr(1) + ModId(4)` + `[Addr(1) + N_Int(1) + N_Frac(1)] × 3` + `MsgId(4)`
- **Note:** Fixed 3 slots. If `Addr === 0`, skip. Use Algorithm A for values.
- **Spec from Raw to SIF**

```jsx
{
  "meta": { "topic": "...", "rawHex": "01EC..." },
  "deviceType": "V5008",
  "deviceId": "Topic[1]",
  "messageType": "NOISE_LEVEL",
  "messageId": "MsgId",
  "moduleIndex": ModAddr,
  "moduleId": "ModId",
  "data": [{ "sensorIndex": Addr, "noise": xx.xx }]
}

// Note:
// Fields discarded from raw to SIF:
//   - Res: Reserved byte not used in SIF (discarded by design)
//
// Field transformations:
//   - Topic[1] → deviceId
//   - MsgId → messageId
//   - ModAddr → moduleIndex
//   - ModId → moduleId
//   - Addr → data[].sensorIndex
//
// Parsing logic:
//   - Fixed 3 slots
//   - If Addr === 0, skip (no sensor data)
//   - Use Algorithm A for N_Int/N_Frac values
//
// Formatting:
//   - Format noise to float "xx.xx" (Algorithm A)
```

### 4.5 `DOOR_STATE`

- **Topic:** `V5008Upload/{deviceId}/OpeAck`
- **Header:** `0xBA`
- **Schema:** `Header(1) + ModAddr(1) + ModId(4) + State(1) + MsgId(4)`
- **Spec from Raw to SIF**

```jsx
{
  "meta": { "topic": "...", "rawHex": "BA01..." },
  "deviceType": "V5008",
  "deviceId": "Topic[1]",
  "messageType": "DOOR_STATE",
  "messageId": "MsgId",
  "data": [{ "moduleIndex": ModAddr, "moduleId": "ModId", "door1State": State, "door2State": null }]
}

// Note:
// Fields discarded from raw to SIF:
//   - None (all fields are transformed)
//
// Field transformations:
//   - Topic[1] → deviceId
//   - MsgId → messageId
//   - ModAddr → data[].moduleIndex
//   - ModId → data[].moduleId
//
// door1State and door2State unification logic:
//   - V5008 has single door sensor:
//     * raw.State → sif.data[].door1State
//     * sif.data[].door2State = null
```



### 4.6 `DEVICE_INFO`

- **Topic:** `V5008Upload/{deviceId}/OpeAck`
- **Header:** `0xEF01`
- **Schema:** `Header(2) + Model(2) + Fw(4) + IP(4) + Mask(4) + Gw(4) + Mac(6) + MsgId(4)`
- **Spec from Raw to SIF**

```jsx
{
  "meta": { "topic": "...", "rawHex": "EF01..." },
  "deviceType": "V5008",
  "deviceId": "Topic[1]",
  "messageType": "DEVICE_INFO",
  "messageId": "MsgId",
  "fwVer": "Fw",//"2509101151",
  "ip": "IP",   //"192.168.0.211",
  "mask": "Mask",//"255.255.0.0",
  "gwIp": "Gw", //"192.168.0.1",
  "mac": "Mac", //"80:82:91:4E:F6:65"
}

// Note:
// Fields discarded from raw to SIF:
//   - None (all fields are used)
//
// Field transformations:
//   - Topic[1] → deviceId
//   - MsgId → messageId
//   - Fw → fwVer
//   - IP → ip
//   - Mask → mask
//   - Gw → gwIp
//   - Mac → mac
//
// Formatting:
//   - IP, Mask, Gw: Dot-notation String (e.g., "192.168.0.1")
//   - Mac: Hex String with colons (e.g., "AA:BB:CC...")
//   - Fw: Use Algorithm D to parse as String
```

### 4.7 `MODULE_INFO`

- **Topic:** `V5008Upload/{deviceId}/OpeAck`
- **Header:** `0xEF02`
- **Schema:** `Header(2)` + `[ModAddr(1) + Fw(4)] × N` + `MsgId(4)`
- **Logic:** `N = (Buffer.length - 6) / 5`
- **Spec from Raw to SIF**

```jsx
{
  "meta": { "topic": "...", "rawHex": "EF02..." },
  "deviceType": "V5008",
  "deviceId": "Topic[1]",
  "messageType": "MODULE_INFO",
  "messageId": "MsgId",
  "data": [{ "moduleIndex": ModAddr, "fwVer": "Fw" }]
}

// Note:
// Fields discarded from raw to SIF:
//   - None (all fields are used)
//
// Field transformations:
//   - Topic[1] → deviceId
//   - MsgId → messageId
//   - ModAddr → data[].moduleIndex
//   - Fw → data[].fwVer
//
// Parsing logic:
//   - N = (Buffer.length - 6) / 5 (number of module entries)
//   - Loop N times to extract module info
```

### 4.8 `QUERY_COLOR_RESP`

- **Topic:** `V5008Upload/{deviceId}/OpeAck`
- **Header:** `0xAA`
- **Schema:** `Header(1) + DeviceId(4) + Result(1) + OriginalReq(2) + [ColorCode × N] + MsgId(4)`
    - OriginalReq: `[E4]+[ModAddr]`
    - Payload: Color codes for all sensors (variable count N)
- **Logic:** `N = Buffer.length - 12` (number of color codes in payload)
- **Spec from Raw to SIF**

```json
{
  "meta": { "topic": "...", "rawHex": "AA..." },
  "deviceType": "V5008",
  "deviceId": "DeviceId",
  "messageType": "QUERY_COLOR_RESP",
  "messageId": "MsgId",
  "result": "Success"|"Failure",
  "originalReq": "OriginalReq",
  "moduleIndex": 1,
  "data": [0, 0, 0, 13, 13, 8]
}

// Note:
// Fields discarded from raw to SIF:
//   - None (all fields are used)
//
// Field transformations:
//   - DeviceId → deviceId
//   - MsgId → messageId
//   - OriginalReq → originalReq
//
// moduleIndex extraction logic:
//   - moduleIndex is extracted from byte 1 of originalReq hex string
//   - Example: originalReq = "E401" → moduleIndex = 0x01 = 1
//
// data array structure:
//   - data is a flat array of color codes (not objects)
//   - Each value is the color code for corresponding sensor position
//   - Count N = Buffer.length - 12
//
// result field creation logic:
//   - Result = 0xA1 → result = "Success"
//   - Result = 0xA0 → result = "Failure"
```

### 4.9 `SET_COLOR_RESP`

- **Topic:** `V5008Upload/{deviceId}/OpeAck`
- **Header:** `0xAA`
- **Schema:** `Header(1) + DeviceId(4) + Result(1) + OriginalReq(var) + MsgId(4)`
    - OriginalReq: `[E1]+[ModAddr] + (uIndex + colorCode) x N`
    - No additional payload (color codes are in originalReq)
- **Logic:** `var = Buffer.length - 10`, `N = (var - 2)/2` (number of sensor-color pairs in originalReq)
- **Spec from Raw to SIF**

```json
{
  "meta": { "topic": "...", "rawHex": "AA..." },
  "deviceType": "V5008",
  "deviceId": "DeviceId",
  "messageType": "SET_COLOR_RESP",
  "messageId": "MsgId",
  "result": "Success"|"Failure",
  "moduleIndex": 1,
  "originalReq": "E10105020601"
}

// Note:
// Fields discarded from raw to SIF:
//   - None (all fields are used)
//
// Field transformations:
//   - DeviceId → deviceId
//   - MsgId → messageId
//   - OriginalReq → originalReq
//
// moduleIndex extraction logic:
//   - moduleIndex is extracted from byte 1 of originalReq hex string
//   - Example: originalReq = "E101..." → moduleIndex = 0x01 = 1
//
// originalReq format breakdown:
//   - Format: [E1][ModAddr][uIndex1][colorCode1][uIndex2][colorCode2]...
//   - Example: "E10105020601" = E1 + 01 + 05 + 01 + 02 + 06 + 01
//     - E1: Command code
//     - 01: Module index
//     - 05: Sensor index 1
//     - 01: Color code for sensor 1
//     - 02: Sensor index 2
//     - 06: Color code for sensor 2
//     - 01: Color code for sensor 3 (if present)
//
// result field creation logic:
//   - Result = 0xA1 → result = "Success"
//   - Result = 0xA0 → result = "Failure"
```

### 4.10 `CLEAR_ALARM_RESP`

- **Topic:** `V5008Upload/{deviceId}/OpeAck`
- **Header:** `0xAA`
- **Schema:** `Header(1) + DeviceId(4) + Result(1) + OriginalReq(3) + MsgId(4)`
    - OriginalReq: `[E2]+[ModAddr]+[uIndex]`
    - No additional payload
- **Spec from Raw to SIF**

```json
{
  "meta": { "topic": "...", "rawHex": "AA..." },
  "deviceType": "V5008",
  "deviceId": "DeviceId",
  "messageType": "CLEAR_ALARM_RESP",
  "messageId": "MsgId",
  "result": "Success"|"Failure",
  "moduleIndex": 1,
  "originalReq": "E20106"
}

// Note:
// Fields discarded from raw to SIF:
//   - None (all fields are used)
//
// Field transformations:
//   - DeviceId → deviceId
//   - MsgId → messageId
//   - OriginalReq → originalReq
//
// moduleIndex extraction logic:
//   - moduleIndex is extracted from byte 1 of originalReq hex string
//   - Example: originalReq = "E201..." → moduleIndex = 0x01 = 1
//
// originalReq format:
//   - Format: [E2][ModAddr][uIndex]
//   - Example: "E20106" = E2 + 01 + 06
//     - E2: Command code
//     - 01: Module index
//     - 06: Sensor index (uIndex)
//
// result field creation logic:
//   - Result = 0xA1 → result = "Success"
//   - Result = 0xA0 → result = "Failure"
```

## 5. Query or Set Command Messages (App → Broker → Device)

### 5.1 `QUERY_DEVICE_INFO`

**Topic:** `V5008Download/{deviceId}`

**Message Format:**

```json
// Raw hex
[EF0100]

// SIF
{
  "deviceId": "2437871205",
  "deviceType": "V5008",
  "messageType": "QUERY_DEVICE_INFO",
  "data": {}
}

// Note:
// Fields discarded from SIF to Raw:
//   - deviceType: Device type not sent in raw message
//   - messageType: Message type not sent in raw message
//   - data: Empty data object not sent in raw message
//
// Field transformations:
//   - deviceId → Not used (extracted from topic in middleware)
//   - messageType "QUERY_DEVICE_INFO" → msg_type "get_devies_init_req" (Note: msg_type not in spec, should be "get_device_info_req")
//   - msg_code = 200 (fixed value)
```

### 5.2 `QUERY_MODULE_INFO`

**Topic:** `V5008Download/{deviceId}`

**Message Format:**

```json
// Raw hex
[EF0200]

// SIF
{
  "deviceId": "2437871205",
  "deviceType": "V5008",
  "messageType": "QUERY_MODULE_INFO",
  "data": {}
}

// Note:
// Fields discarded from SIF to Raw:
//   - deviceType: Device type not sent in raw message
//   - messageType: Message type not sent in raw message
//   - data: Empty data object not sent in raw message
//
// Field transformations:
//   - deviceId → Not used (extracted from topic in middleware)
//   - messageType "QUERY_MODULE_INFO" → msg_type "get_module_info_req" (Note: msg_type not in spec, should be "get_module_info_req")
//   - msg_code = 200 (fixed value)
```

### 5.3 `QUERY_RFID_SNAPSHOT`

**Topic:** `V5008Download/{deviceId}`

**Message Format:**

```json
// Raw format
[E901][moduleIndex]

// SIF
{
  "deviceId": "2437871205",
  "deviceType": "V5008",
  "messageType": "QUERY_RFID_SNAPSHOT",
  "data": {"moduleIndex": 4}
}

// Note:
// Fields discarded from SIF to Raw:
//   - deviceType: Device type not sent in raw message
//   - messageType: Message type not sent in raw message
//   - data.sensorIndex: Not sent in raw message (only moduleIndex is sent)
//
// Field transformations:
//   - deviceId → Not used (extracted from topic in middleware)
//   - data.moduleIndex → [moduleIndex] (array format)
//   - messageType "QUERY_RFID_SNAPSHOT" → msg_type "u_state_req"
```

### 5.4 `QUERY_DOOR_STATE`

**Topic:** `V5008Download/{deviceId}`

**Message Format:**

```json
// Raw format
[E903][moduleIndex]

// SIF Format
{
  "deviceId": "2437871205",
  "deviceType": "V5008",
  "messageType": "QUERY_DOOR_STATE",
  "data": {"moduleIndex": 4}
}

// Note:
// Fields discarded from SIF to Raw:
//   - deviceType: Device type not sent in raw message
//   - messageType: Message type not sent in raw message
//   - data.sensorIndex: Not sent in raw message (only moduleIndex is sent)
//
// Field transformations:
//   - deviceId → Not used (extracted from topic in middleware)
//   - data.moduleIndex → [moduleIndex] (array format)
//   - messageType "QUERY_DOOR_STATE" → msg_type "door_state_req"
```

### 5.5 `QUERY_TEMP_HUM`

**Topic:** `V5008Download/{deviceId}`

**Message Format:**

```json
// Raw format
[E902][moduleIndex]

// SIF format
{
  "deviceId": "2437871205",
  "deviceType": "V5008",
  "messageType": "QUERY_TEMP_HUM",
  "data": {"moduleIndex": 4}
}

// Note:
// Fields discarded from SIF to Raw:
//   - deviceType: Device type not sent in raw message
//   - messageType: Message type not sent in raw message
//   - data.sensorIndex: Not sent in raw message (only moduleIndex is sent)
//
// Field transformations:
//   - deviceId → Not used (extracted from topic in middleware)
//   - data.moduleIndex → [moduleIndex] (array format)
//   - messageType "QUERY_TEMP_HUM" → msg_type "temper_humidity_req"
```

### 5.6 `SET_COLOR`

**Topic:** `V5008Download/{deviceId}`

**Message Format:**

```json
// Raw
[E1][01][0A01]

// SIF
{
  "deviceId": "2437871205",
  "deviceType": "V5008",
  "messageType": "SET_COLOR",
  "payload": {
    "moduleIndex": 1,
    "sensorIndex": 10,
    "colorCode": 1
  }
}

// Note:
// Fields discarded from SIF to Raw:
//   - deviceType: Device type not sent in raw message
//   - messageType: Message type not sent in raw message
//   - data.sensorIndex: Not sent in raw message (mapped to u_index)
//
// Field transformations:
//   - deviceId → Not used (extracted from topic in middleware)
//   - data.moduleIndex → data[].host_gateway_port_index
//   - data.sensorIndex → data[].u_color_data[].u_index
//   - data.colorCode → data[].u_color_data[].color_code
//   - messageType "SET_COLOR" → msg_type "set_module_property_req"
//   - set_property_type = 8001 (fixed value)
//   - module_type = 2 (fixed value)
//   - extend_module_sn = null
```

### 5.7 `QUERY_COLOR`

**Topic:** `V5008Download/{deviceId}`

**Message Format:**

```json
// Raw format
[E4][moduleIndex]

// SIF format
{
  "deviceId": "2437871205",
  "deviceType": "V5008",
  "messageType": "QUERY_COLOR",
  "data": {"moduleIndex": 4 }
}

// Note:
// Fields discarded from SIF to Raw:
//   - deviceType: Device type not sent in raw message
//   - messageType: Message type not sent in raw message
//   - data.sensorIndex: Not sent in raw message (only moduleIndex is sent)
//
// Field transformations:
//   - deviceId → Not used (extracted from topic in middleware)
//   - data.moduleIndex → [moduleIndex] (array format)
//   - messageType "QUERY_COLOR" → msg_type "get_u_color"
//   - code = 0 (default value, may vary)
```

### 5.8 `CLEAR_ALARM`

**Topic:** `V5008Download/{deviceId}`

**Message Format:**

```json
// Raw format
[E2][01][0A]

// SIF format
{
  "deviceId": "2437871205",
  "deviceType": "V5008",
  "messageType": "CLEAR_ALARM",
  "data": {"moduleIndex": 1, "sensorIndex": 10}
}

// Note:
// Fields discarded from SIF to Raw:
//   - deviceType: Device type not sent in raw message
//   - messageType: Message type not sent in raw message
//
// Field transformations:
//   - deviceId → Not used (extracted from topic in middleware)
//   - data.moduleIndex → data[].index
//   - data.sensorIndex → data[].warning_data[0]
//   - messageType "CLEAR_ALARM" → msg_type "clear_u_warning"
//   - code = 0 (default value, may vary)
//   - data[].warning_data = [sensorIndex] (array format)
```