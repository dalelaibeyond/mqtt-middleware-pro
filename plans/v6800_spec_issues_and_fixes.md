# V6800 Spec Issues and Fixes

## Analysis Date
2026-02-27

## Summary
The `specs/v6800_spec.md` file has several structural inconsistencies that should be addressed to maintain a unified JSON structure across all message types.

---

## Critical Issues

### 1. QUERY_DOOR_STATE_RESP Structure Inconsistency ⚠️ **HIGH PRIORITY**

**Location:** Lines 348-358

**Current Structure:**
```json
{
  "meta": { "topic": "...", "rawType": "msg_type" },
  "deviceId": "Topic[1]",
  "deviceType": "V6800",
  "messageType": "QUERY_DOOR_STATE_RESP",
  "messageId": "uuid_number",
  "moduleIndex": gateway_port_index,
  "door1State": new_state1,
  "door2State": new_state2
}
```

**Issue:** Fields are at top level instead of inside `"data": [{...}]` array.

**Comparison with DOOR_STATE_EVENT (Lines 308-316):**
```json
{
  "meta": { "topic": "...", "rawType": "msg_type" },
  "deviceId": "gateway_sn",
  "deviceType": "V6800",
  "messageType": "DOOR_STATE",
  "messageId": "uuid_number",
  "data": [{ "moduleId": "extend_module_sn", "moduleIndex": host_gateway_port_index, "door1State": 1, "door2State": 1 }]
}
```

**Proposed Fix:**
```json
{
  "meta": { "topic": "...", "rawType": "msg_type" },
  "deviceId": "Topic[1]",
  "deviceType": "V6800",
  "messageType": "QUERY_DOOR_STATE_RESP",
  "messageId": "uuid_number",
  "data": [{
    "moduleIndex": gateway_port_index,
    "door1State": new_state1,
    "door2State": new_state2
  }]
}
```

**Impact:** This aligns with all other response messages and maintains consistency.

---

### 2. DOOR_STATE_EVENT messageType Naming Inconsistency

**Location:** Line 313

**Current:** `"messageType": "DOOR_STATE"`

**Issue:** The section header says `DOOR_STATE_EVENT` but the messageType value is `"DOOR_STATE"`. This breaks the naming convention where event messages end with `_EVENT`.

**Proposed Fix:** Change to `"messageType": "DOOR_STATE_EVENT"`

**Impact:** Maintains naming consistency with other event messages like `RFID_EVENT`, `MOD_CHNG_EVENT`, etc.

---

## Medium Priority Issues

### 3. Section Numbering Inconsistency in Part#2

**Location:** Line 561

**Current:** `### 6. SET_COLOR`

**Issue:** Should be `### 2.5 SET_COLOR` to maintain sequential numbering (2.1, 2.2, 2.3, 2.4, 2.5, 2.6, 2.7).

**Proposed Fix:** Change to `### 2.5 SET_COLOR`

---

### 4. SET_COLOR Uses "payload" Instead of "data"

**Location:** Lines 581-591

**Current:**
```json
{
  "deviceId": "2437871205",
  "deviceType": "V6800",
  "messageType": "SET_COLOR",
  "payload": {
    "moduleIndex": 1,
    "sensorIndex": 10,
    "colorCode": 1
  }
}
```

**Issue:** All other command messages use `"data": {...}` but SET_COLOR uses `"payload": {...}`.

**Proposed Fix:**
```json
{
  "deviceId": "2437871205",
  "deviceType": "V6800",
  "messageType": "SET_COLOR",
  "data": {
    "moduleIndex": 1,
    "sensorIndex": 10,
    "colorCode": 1
  }
}
```

**Impact:** Maintains consistency with all other command messages.

---

## Low Priority Issues

### 5. Missing moduleId in QUERY_DOOR_STATE_RESP

**Location:** Lines 329-346

**Issue:** The raw format for `QUERY_DOOR_STATE_RESP` doesn't include `extend_module_sn` field, while `DOOR_STATE_EVENT` does. This might be intentional but worth noting for consistency.

**Note:** This may be by design as the response might not include module info, but should be verified with device protocol documentation.

---

### 6. Typo in Comment

**Location:** Line 222

**Current:** `// Note: Device may send "temper_humidity_exception_nofity_req" (typo: "nofity" instead of "notify")`

**Issue:** The comment correctly identifies the typo but could be clearer.

---

## Consistency Analysis

### All Response Messages Use "data":[{...}] Structure:

| Message Type | Uses data:[{...}] | Structure |
|--------------|-------------------|-----------|
| DEV_MOD_INFO | ✅ | data:[{moduleIndex, fwVer, moduleId, uTotal}] |
| MOD_CHNG_EVENT | ✅ | data:[{moduleIndex, fwVer, moduleId, uTotal}] |
| HEARTBEAT | ✅ | data:[{moduleIndex, moduleId, uTotal}] |
| RFID_EVENT | ✅ | data:[{moduleIndex, moduleId, data:[{sensorIndex, action, tagId, isAlarm}]}] |
| RFID_SNAPSHOT | ✅ | data:[{moduleIndex, moduleId, data:[{sensorIndex, tagId, isAlarm}]}] |
| TEMP_HUM | ✅ | data:[{moduleIndex, moduleId, data:[{sensorIndex, temp, hum}]}] |
| QUERY_TEMP_HUM_RESP | ✅ | data:[{moduleIndex, moduleId, data:[{sensorIndex, temp, hum}]}] |
| DOOR_STATE_EVENT | ✅ | data:[{moduleId, moduleIndex, door1State, door2State}] |
| **QUERY_DOOR_STATE_RESP** | ❌ **BROKEN** | moduleIndex, door1State, door2State at top level |
| SET_COLOR_RESP | ✅ | data:[{moduleIndex, moduleId, result}] |
| QUERY_COLOR_RESP | ✅ | data:[{moduleIndex, moduleId, uTotal, data:[{sensorIndex, colorName, colorCode}]}] |
| CLEAR_ALARM_RESP | ✅ | data:[{moduleIndex, moduleId, uTotal, result}] |

### All Command Messages Use "data":{...} Structure:

| Message Type | Uses data:{} | Structure |
|--------------|--------------|-----------|
| QUERY_DEV_MOD_INFO | ✅ | data:{} |
| QUERY_RFID_SNAPSHOT | ✅ | data:{moduleIndex} |
| QUERY_DOOR_STATE | ✅ | data:{moduleIndex} |
| QUERY_TEMP_HUM | ✅ | data:{moduleIndex} |
| **SET_COLOR** | ❌ **BROKEN** | payload:{moduleIndex, sensorIndex, colorCode} |
| QUERY_COLOR | ✅ | data:{moduleIndex} |
| CLEAR_ALARM | ✅ | data:{moduleIndex, sensorIndex} |

---

## Recommended Action Plan

1. **Fix QUERY_DOOR_STATE_RESP** - Move fields into `"data": [{...}]` array structure
2. **Fix DOOR_STATE_EVENT** - Change messageType from "DOOR_STATE" to "DOOR_STATE_EVENT"
3. **Fix section numbering** - Change "### 6. SET_COLOR" to "### 2.5 SET_COLOR"
4. **Fix SET_COLOR** - Change "payload" to "data" in SIF format
5. **Review and validate** - Ensure all changes maintain backward compatibility where possible

---

## Mermaid Diagram: Message Structure Consistency

```mermaid
graph TD
    A[V6800 Message Types] --> B[Response Messages]
    A --> C[Command Messages]
    
    B --> B1[11 messages use data:[{...}]]
    B --> B2[QUERY_DOOR_STATE_RESP - BROKEN]
    
    C --> C1[6 messages use data:{...}]
    C --> C2[SET_COLOR - BROKEN]
    
    B2 --> D[Fix: Move fields to data:[{...}]]
    C2 --> E[Fix: Change payload to data]
    
    D --> F[Unified Structure]
    E --> F
```

---

## Questions for User

1. Should we add `moduleId` field to `QUERY_DOOR_STATE_RESP` for consistency with `DOOR_STATE_EVENT`?
2. Is the missing `extend_module_sn` in the raw response intentional or an oversight?
3. Should we update the version number after making these changes?
4. Are there any other message types that need similar alignment?
