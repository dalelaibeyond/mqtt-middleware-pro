# SUO_UOS_DB_Spec Alignment Review

**Date:** 2026-02-28
**Reviewer:** Architect Mode
**Scope:** Review of `specs/SUO_UOS_DB_Spec.md` alignment with updated `specs/V5008_Spec.md` and `specs/v6800_spec.md`

---

## Executive Summary

This review identified **15 alignment issues** between [`SUO_UOS_DB_Spec.md`](specs/SUO_UOS_DB_Spec.md) and the device specifications ([`V5008_Spec.md`](specs/V5008_Spec.md) and [`v6800_spec.md`](specs/v6800_spec.md)). The issues range from missing fields in SIF to incorrect field mappings, device-specific field handling, and documentation gaps.

**Severity Breakdown:**
- **Critical (5):** Missing required fields, incorrect data structures
- **Major (6):** Device-specific field handling inconsistencies
- **Minor (4):** Documentation gaps, missing examples

---

## Critical Issues

### Issue #1: V5008 HEARTBEAT - Missing `onlineCount` Field in SIF

**Location:**
- [`SUO_UOS_DB_Spec.md:241`](specs/SUO_UOS_DB_Spec.md:241)
- [`V5008_Spec.md:179-197`](specs/V5008_Spec.md:179)

**Problem:**
[`SUO_HEARTBEAT`](specs/SUO_UOS_DB_Spec.md:194) schema includes `onlineCount` field in the modules array, but [`V5008 HEARTBEAT`](specs/V5008_Spec.md:179) SIF does not have this field.

**V5008 HEARTBEAT SIF:**
```json
{
  "data": [{"moduleIndex": ModAddr, "moduleId": "ModId", "uTotal": Total}]
}
```

**SUO_HEARTBEAT (incorrect):**
```json
{
  "data": {
    "modules": [
      {
        "moduleIndex": 1,
        "moduleId": "1234567890",
        "uTotal": 54,
        "onlineCount": 54  // ← Field not in V5008 SIF!
      }
    ]
  }
}
```

**Root Cause:**
The V5008 HEARTBEAT binary schema (line 183) is:
`Header(1) + [ModAddr(1) + ModId(4) + Total(1)] × 10 + MsgId(4)`

There is no `Count` field in the binary protocol, only `Total` (uTotal).

**Impact:**
Parser will fail when trying to extract `onlineCount` from V5008 HEARTBEAT SIF.

**Recommendation:**
Update [`SUO_UOS_DB_Spec.md:241`](specs/SUO_UOS_DB_Spec.md:241) to document that `onlineCount` is **V6800-only**:
```markdown
| data.modules[].onlineCount | number | null | Number of online sensors (V6800 only, null for V5008) |
```

---

### Issue #2: V5008 DOOR_STATE - Incorrect Data Structure

**Location:**
- [`SUO_UOS_DB_Spec.md:418-447`](specs/SUO_UOS_DB_Spec.md:418)
- [`V5008_Spec.md:316-348`](specs/V5008_Spec.md:316)

**Problem:**
[`SUO_DOOR_STATE`](specs/SUO_UOS_DB_Spec.md:418) example shows `door1State` and `door2State` at the top level of `data` object, but [`V5008 DOOR_STATE`](specs/V5008_Spec.md:316) SIF has these fields in a nested `data` array.

**V5008 DOOR_STATE SIF:**
```json
{
  "messageType": "DOOR_STATE",
  "data": [
    {
      "moduleIndex": ModAddr,
      "moduleId": "ModId",
      "door1State": State,
      "door2State": null
    }
  ]
}
```

**SUO_DOOR_STATE (incorrect):**
```json
{
  "moduleIndex": 2,
  "moduleId": "1234567891",
  "data": {
    "door1State": 1,    // ← Should be at top level after flattening
    "door2State": null
  }
}
```

**Root Cause:**
The SUO example shows the final flattened structure, but doesn't account for the fact that V5008 SIF has `data` as an array. After flattening, `moduleIndex` and `moduleId` should be at the top level, and `door1State`/`door2State` should be in `data`.

**Impact:**
Transformer will have incorrect data structure for V5008 DOOR_STATE.

**Recommendation:**
Update [`SUO_DOOR_STATE`](specs/SUO_UOS_DB_Spec.md:418) example to show correct flattened structure:
```json
{
  "moduleIndex": 2,
  "moduleId": "1234567891",
  "data": {
    "door1State": 1,
    "door2State": null
  }
}
```

---

### Issue #3: V5008 QUERY_COLOR_RESP - Incorrect `colorCodes` Structure

**Location:**
- [`SUO_UOS_DB_Spec.md:454-482`](specs/SUO_UOS_DB_Spec.md:454)
- [`V5008_Spec.md:424-468`](specs/V5008_Spec.md:424)

**Problem:**
[`SUO_COMMAND_RESULT`](specs/SUO_UOS_DB_Spec.md:454) shows `colorCodes` as an array of objects with `sensorIndex` and `colorCode`, but [`V5008 QUERY_COLOR_RESP`](specs/V5008_Spec.md:424) SIF has `data` as a flat array of integers.

**V5008 QUERY_COLOR_RESP SIF:**
```json
{
  "messageType": "QUERY_COLOR_RESP",
  "result": "Success",
  "moduleIndex": 1,
  "originalReq": "E401",
  "data": [0, 0, 0, 13, 13, 8]  // ← Flat array of integers
}
```

**SUO_COMMAND_RESULT (incorrect):**
```json
{
  "data": {
    "commandType": "QUERY_COLOR",
    "result": "Success",
    "originalReq": "E401",
    "colorCodes": [
      {
        "sensorIndex": 1,
        "colorCode": 0
      }
    ]
  }
}
```

**Root Cause:**
The SUO schema shows an object-based structure, but V5008 provides a flat array where the index in the array corresponds to the sensor index.

**Impact:**
Transformer will fail when trying to map flat array to object structure.

**Recommendation:**
Update [`SUO_COMMAND_RESULT`](specs/SUO_UOS_DB_Spec.md:454) schema to use flat array:
```json
{
  "data": {
    "commandType": "QUERY_COLOR",
    "result": "Success",
    "originalReq": "E401",
    "colorCodes": [0, 0, 0, 13, 13, 8]  // ← Flat array
  }
}
```

Update field description (line 481):
```markdown
| data.colorCodes | array | Color codes for sensors (QUERY_COLOR_RESP only, flat array where index = sensorIndex) |
```

---

### Issue #4: V6800 HEARTBEAT - Power Fields Missing from SIF

**Location:**
- [`SUO_UOS_DB_Spec.md:208-228`](specs/SUO_UOS_DB_Spec.md:208)
- [`v6800_spec.md:129-183`](specs/v6800_spec.md:129)

**Problem:**
[`SUO_HEARTBEAT`](specs/SUO_UOS_DB_Spec.md:208) includes power fields (`busVoltage`, `busCurrent`, `mainPower`, `backupPower`) as "V6800 only", but [`V6800 HEARTBEAT`](specs/v6800_spec.md:129) SIF explicitly discards these fields.

**V6800 HEARTBEAT SIF:**
```json
{
  "messageType": "HEARTBEAT",
  "data": [
    {
      "moduleIndex": module_index,
      "moduleId": "module_sn",
      "uTotal": module_u_num
    }
  ]
}
```

**SUO_HEARTBEAT (incorrect):**
```json
{
  "data": {
    "modules": [...],
    "busVoltage": "12.5",      // ← Marked as V6800 only
    "busCurrent": "2.3",       // ← But NOT in V6800 SIF!
    "mainPower": 1,
    "backupPower": 0
  }
}
```

**Root Cause:**
[`v6800_spec.md`](specs/v6800_spec.md:169-174) explicitly states these fields are "not needed in SIF":
- bus_V: Bus voltage not needed in SIF
- bus_I: Bus current not needed in SIF
- main_power: Main power status not needed in SIF
- backup_power: Backup power status not needed in SIF

But [`SUO_UOS_DB_Spec.md`](specs/SUO_UOS_DB_Spec.md:242-245) includes them as V6800-only fields.

**Impact:**
Transformer will fail when trying to extract non-existent fields from V6800 HEARTBEAT SIF.

**Recommendation:**
**Option 1:** Remove power fields from SUO_HEARTBEAT (simpler, but loses power monitoring data)
**Option 2:** Update [`v6800_spec.md`](specs/v6800_spec.md:129) to include power fields in SIF instead of discarding them (recommended for power monitoring)

If choosing Option 2, update V6800 HEARTBEAT SIF:
```json
{
  "messageType": "HEARTBEAT",
  "busVoltage": "12.5",
  "busCurrent": "2.3",
  "mainPower": 1,
  "backupPower": 0,
  "data": [...]
}
```

---

### Issue #5: V6800 QUERY_COLOR_RESP - Missing `colorName` Field

**Location:**
- [`SUO_UOS_DB_Spec.md:454-482`](specs/SUO_UOS_DB_Spec.md:454)
- [`v6800_spec.md:546-594`](specs/v6800_spec.md:546)

**Problem:**
[`SUO_COMMAND_RESULT`](specs/SUO_UOS_DB_Spec.md:454) only includes `colorCode` (integer), but [`V6800 QUERY_COLOR_RESP`](specs/v6800_spec.md:546) SIF includes both `colorName` (string) and `colorCode` (integer).

**V6800 QUERY_COLOR_RESP SIF:**
```json
{
  "messageType": "QUERY_COLOR_RESP",
  "data": [{
    "moduleIndex": index,
    "moduleId": module_id,
    "uTotal": u_num,
    "data": [{
      "sensorIndex": index,
      "colorName": "color",  // ← String (e.g., "red", "green")
      "colorCode": code
    }]
  }]
}
```

**SUO_COMMAND_RESULT (incorrect):**
```json
{
  "data": {
    "commandType": "QUERY_COLOR",
    "colorCodes": [
      {
        "sensorIndex": 1,
        "colorCode": 0  // ← Missing colorName!
      }
    ]
  }
}
```

**Root Cause:**
V6800 raw message has both `color` (string) and `code` (integer), but SUO only preserves the integer code.

**Impact:**
Loss of human-readable color names for V6800 devices.

**Recommendation:**
Update [`SUO_COMMAND_RESULT`](specs/SUO_UOS_DB_Spec.md:454) schema to include both fields:
```json
{
  "data": {
    "commandType": "QUERY_COLOR",
    "colorCodes": [
      {
        "sensorIndex": 1,
        "colorName": "red",    // ← Add this field
        "colorCode": 0
      }
    ]
  }
}
```

Update field description:
```markdown
| data.colorCodes[].colorName | string | Color name (V6800 QUERY_COLOR_RESP only) |
| data.colorCodes[].colorCode | number | Color code integer |
```

---

## Major Issues

### Issue #6: V6800 SET_COLOR_RESP - Missing `originalReq` Field

**Location:**
- [`SUO_UOS_DB_Spec.md:454-482`](specs/SUO_UOS_DB_Spec.md:454)
- [`v6800_spec.md:504-544`](specs/v6800_spec.md:504)

**Problem:**
[`SUO_COMMAND_RESULT`](specs/SUO_UOS_DB_Spec.md:454) includes `originalReq` as a required field, but [`V6800 SET_COLOR_RESP`](specs/v6800_spec.md:504) SIF does not have this field.

**V6800 SET_COLOR_RESP SIF:**
```json
{
  "messageType": "SET_COLOR_RESP",
  "data": [{
    "moduleIndex": host_gateway_port_index,
    "moduleId": extend_module_sn,
    "result": "Success"|"Failure"
  }]
}
```

**SUO_COMMAND_RESULT (incorrect):**
```json
{
  "data": {
    "commandType": "SET_COLOR",
    "result": "Success",
    "originalReq": "E401"  // ← Field not in V6800 SIF!
  }
}
```

**Root Cause:**
V6800 SET_COLOR_RESP raw message does not echo back the original request, unlike V5008 which includes `originalReq`.

**Impact:**
Transformer will fail when trying to extract `originalReq` from V6800 SET_COLOR_RESP SIF.

**Recommendation:**
Update [`SUO_COMMAND_RESULT`](specs/SUO_UOS_DB_Spec.md:454) field description (line 481):
```markdown
| data.originalReq | string | null | Original request data (hex string) - V5008 only, null for V6800 |
```

---

### Issue #7: V6800 CLEAR_ALARM_RESP - Missing `sensorIndex` Field

**Location:**
- [`SUO_UOS_DB_Spec.md:454-482`](specs/SUO_UOS_DB_Spec.md:454)
- [`v6800_spec.md:596-639`](specs/v6800_spec.md:596)

**Problem:**
[`SUO_COMMAND_RESULT`](specs/SUO_UOS_DB_Spec.md:454) includes `sensorIndex` for CLEAR_ALARM_RESP, but [`V6800 CLEAR_ALARM_RESP`](specs/v6800_spec.md:596) SIF does not have this field.

**V6800 CLEAR_ALARM_RESP SIF:**
```json
{
  "messageType": "CLEAR_ALARM_RESP",
  "data": [{
    "moduleIndex": index,
    "moduleId": module_id,
    "uTotal": u_num,
    "result": "Success"|"Failure"
  }]
}
```

**SUO_COMMAND_RESULT (incorrect):**
```json
{
  "data": {
    "commandType": "CLEAR_ALARM",
    "sensorIndex": null  // ← Field not in V6800 SIF!
  }
}
```

**Root Cause:**
V6800 CLEAR_ALARM_RESP raw message does not include which sensor was cleared, only confirms success/failure. V5008 includes sensorIndex in the `originalReq` field.

**Impact:**
Transformer will fail when trying to extract `sensorIndex` from V6800 CLEAR_ALARM_RESP SIF.

**Recommendation:**
Update [`SUO_COMMAND_RESULT`](specs/SUO_UOS_DB_Spec.md:454) field description (line 482):
```markdown
| data.sensorIndex | number | null | Sensor index (V5008 CLEAR_ALARM_RESP only, null for V6800) |
```

---

### Issue #8: V6800 QUERY_DOOR_STATE_RESP - Missing `moduleId` Field

**Location:**
- [`SUO_UOS_DB_Spec.md:418-447`](specs/SUO_UOS_DB_Spec.md:418)
- [`v6800_spec.md:442-502`](specs/v6800_spec.md:442)

**Problem:**
[`SUO_DOOR_STATE`](specs/SUO_UOS_DB_Spec.md:418) requires `moduleId` at the top level, but [`V6800 QUERY_DOOR_STATE_RESP`](specs/v6800_spec.md:442) SIF does not have this field.

**V6800 QUERY_DOOR_STATE_RESP SIF:**
```json
{
  "messageType": "QUERY_DOOR_STATE_RESP",
  "data": [{
    "moduleIndex": gateway_port_index,
    "door1State": new_state1,
    "door2State": new_state2
  }]
}
```

**SUO_DOOR_STATE (incorrect):**
```json
{
  "moduleIndex": 2,
  "moduleId": "1234567891",  // ← Required field not in V6800 SIF!
  "data": {
    "door1State": 1,
    "door2State": null
  }
}
```

**Root Cause:**
V6800 QUERY_DOOR_STATE_RESP raw message does not include module serial number, only the module index from the topic.

**Impact:**
Transformer will fail when trying to extract `moduleId` from V6800 QUERY_DOOR_STATE_RESP SIF.

**Recommendation:**
Update [`SUO_DOOR_STATE`](specs/SUO_UOS_DB_Spec.md:418) to make `moduleId` optional:
```markdown
| moduleId | string | null | Module serial number (null for V6800 QUERY_DOOR_STATE_RESP) |
```

---

### Issue #9: SUO_HEARTBEAT - Inconsistent Device-Specific Fields

**Location:**
- [`SUO_UOS_DB_Spec.md:242-245`](specs/SUO_UOS_DB_Spec.md:242)

**Problem:**
Power fields (`busVoltage`, `busCurrent`, `mainPower`, `backupPower`) are marked as "V6800 only" but the spec doesn't clearly document that these should be `null` for V5008.

**Current Description:**
```markdown
| data.busVoltage | string | Bus voltage (V6800 only) |
| data.busCurrent | string | Bus current (V6800 only) |
| data.mainPower | number | Main power status (V6800 only) |
| data.backupPower | number | Backup power status (V6800 only) |
```

**Recommendation:**
Update descriptions to be explicit about null values:
```markdown
| data.busVoltage | string | null | Bus voltage in V (V6800 only, null for V5008) |
| data.busCurrent | string | null | Bus current in A (V6800 only, null for V5008) |
| data.mainPower | number | null | Main power status (V6800 only, null for V5008) |
| data.backupPower | number | null | Backup power status (V6800 only, null for V5008) |
```

---

### Issue #10: SUO_COMMAND_RESULT - Inconsistent Field Presence

**Location:**
- [`SUO_UOS_DB_Spec.md:474-482`](specs/SUO_UOS_DB_Spec.md:474)

**Problem:**
[`SUO_COMMAND_RESULT`](specs/SUO_UOS_DB_Spec.md:454) has multiple optional fields that are device-specific, but the spec doesn't clearly document which fields apply to which device types and command types.

**Current Schema:**
```json
{
  "data": {
    "commandType": "string",
    "result": "string",
    "originalReq": "string",      // ← V5008 only
    "colorCodes": array,         // ← QUERY_COLOR_RESP only
    "sensorIndex": number        // ← CLEAR_ALARM_RESP (V5008) only
  }
}
```

**Recommendation:**
Create device-specific field mapping table:
```markdown
### SUO_COMMAND_RESULT Field Presence by Device and Command

| Field | V5008 QUERY_COLOR | V5008 SET_COLOR | V5008 CLEAR_ALARM | V6800 QUERY_COLOR | V6800 SET_COLOR | V6800 CLEAR_ALARM |
|--------|-------------------|------------------|-------------------|---------------------|-------------------|----------------------|
| originalReq | ✓ | ✓ | ✓ | ✗ | ✗ | ✗ |
| colorCodes | ✓ (flat array) | ✗ | ✗ | ✓ (array of objects) | ✗ | ✗ |
| sensorIndex | ✗ | ✗ | ✓ (in originalReq) | ✗ | ✗ | ✗ |
```

---

### Issue #11: Missing V6800 Examples in SUO Schemas

**Location:**
- [`SUO_UOS_DB_Spec.md:418-482`](specs/SUO_UOS_DB_Spec.md:418)

**Problem:**
Most SUO schema examples only show V5008 devices, missing V6800-specific examples.

**Examples Missing V6800:**
- [`SUO_HEARTBEAT`](specs/SUO_UOS_DB_Spec.md:194) - Should show V6800 with power fields
- [`SUO_DOOR_STATE`](specs/SUO_UOS_DB_Spec.md:418) - Should show V6800 with dual door
- [`SUO_COMMAND_RESULT`](specs/SUO_UOS_DB_Spec.md:454) - Should show V6800 QUERY_COLOR_RESP with colorName

**Recommendation:**
Add V6800-specific examples for each SUO type that has device-specific fields.

---

## Minor Issues

### Issue #12: SUO_RFID_SNAPSHOT - Missing V6800 Example

**Location:**
- [`SUO_UOS_DB_Spec.md:247-293`](specs/SUO_UOS_DB_Spec.md:247)

**Problem:**
[`SUO_RFID_SNAPSHOT`](specs/SUO_UOS_DB_Spec.md:247) example only shows V5008 device, should also show V6800 example to demonstrate multi-module flattening.

**Recommendation:**
Add V6800 example showing flattening from multi-module SIF to multiple SUO messages.

---

### Issue #13: SUO_TEMP_HUM - Missing V6800 Example

**Location:**
- [`SUO_UOS_DB_Spec.md:328-374`](specs/SUO_UOS_DB_Spec.md:328)

**Problem:**
[`SUO_TEMP_HUM`](specs/SUO_UOS_DB_Spec.md:328) example only shows V5008 device.

**Recommendation:**
Add V6800 example showing QUERY_TEMP_HUM_RESP mapping.

---

### Issue #14: Flattening Rules - QUERY_COLOR_RESP Not Listed

**Location:**
- [`SUO_UOS_DB_Spec.md:93-110`](specs/SUO_UOS_DB_Spec.md:93)
- [`SUO_UOS_DB_Spec.md:1147-1173`](specs/SUO_UOS_DB_Spec.md:1147)

**Problem:**
Flattening rules in Section 3.2 don't list QUERY_COLOR_RESP, but it can have multiple modules in V6800.

**Current Rules:**
```markdown
**Apply Flattening:**
- RFID_SNAPSHOT
- RFID_EVENT
- TEMP_HUM
- QUERY_TEMP_HUM_RESP
- DOOR_STATE_EVENT
- QUERY_DOOR_STATE_RESP

**Do NOT Apply Flattening:**
- DEV_MOD_INFO
- MOD_CHNG_EVENT
- HEARTBEAT
```

**Missing:** QUERY_COLOR_RESP is not in either list.

**Recommendation:**
Add explicit note that command responses (QUERY_COLOR_RESP, SET_COLOR_RESP, CLEAR_ALARM_RESP) are NOT flattened, even if they contain multiple modules.

---

### Issue #15: Appendix A - Incomplete Flattening Column

**Location:**
- [`SUO_UOS_DB_Spec.md:1147-1173`](specs/SUO_UOS_DB_Spec.md:1147)

**Problem:**
Appendix A mapping table shows "Flattening Required" column, but doesn't clearly indicate which messages are multi-module vs. single-module.

**Recommendation:**
Add "Multi-Module" column to clarify:
```markdown
| SIF Message Type | SUO Type | Multi-Module | Flattening Required |
|------------------|------------|---------------|---------------------|
| V6800: RFID_SNAPSHOT | SUO_RFID_SNAPSHOT | Yes | Yes |
| V6800: QUERY_COLOR_RESP | SUO_COMMAND_RESULT | Yes | No |
```

---

## Summary of Required Changes

### High Priority (Critical Issues #1-5)

1. **Fix V5008 HEARTBEAT** - Remove `onlineCount` or mark as V6800-only
2. **Fix V5008 DOOR_STATE** - Correct data structure
3. **Fix V5008 QUERY_COLOR_RESP** - Use flat array for colorCodes
4. **Fix V6800 HEARTBEAT** - Either remove power fields or update V6800 SIF to include them
5. **Fix V6800 QUERY_COLOR_RESP** - Add `colorName` field

### Medium Priority (Major Issues #6-11)

6. **Fix V6800 SET_COLOR_RESP** - Make `originalReq` optional (V5008-only)
7. **Fix V6800 CLEAR_ALARM_RESP** - Make `sensorIndex` optional (V5008-only)
8. **Fix V6800 QUERY_DOOR_STATE_RESP** - Make `moduleId` optional
9. **Clarify SUO_HEARTBEAT** - Document null values for V5008 power fields
10. **Document SUO_COMMAND_RESULT** - Create device-specific field mapping table
11. **Add V6800 examples** - Add V6800-specific examples to all relevant SUO types

### Low Priority (Minor Issues #12-15)

12. **Add V6800 example to SUO_RFID_SNAPSHOT**
13. **Add V6800 example to SUO_TEMP_HUM**
14. **Clarify flattening for command responses**
15. **Enhance Appendix A** - Add "Multi-Module" column

---

## Decision Points

The following issues require architectural decisions:

### Decision #1: V6800 HEARTBEAT Power Fields

**Question:** Should V6800 HEARTBEAT include power monitoring fields?

**Options:**
- **Option A:** Remove power fields from SUO_HEARTBEAT (simpler, but loses data)
- **Option B:** Update V6800 HEARTBEAT SIF to include power fields (recommended for power monitoring)

**Recommendation:** Option B - Power monitoring is important for IoT devices.

### Decision #2: V6800 QUERY_COLOR_RESP Color Data

**Question:** Should SUO include both colorName and colorCode for V6800?

**Options:**
- **Option A:** Include both fields (preserves all data)
- **Option B:** Include only colorCode (simpler, but loses human-readable names)

**Recommendation:** Option A - Preserves all data from V6800 devices.

### Decision #3: V6800 QUERY_DOOR_STATE_RESP Module ID

**Question:** How to handle missing moduleId in V6800 QUERY_DOOR_STATE_RESP?

**Options:**
- **Option A:** Make moduleId optional (null for V6800)
- **Option B:** Update V6800 protocol to include module serial number in response

**Recommendation:** Option A - Cannot change V6800 protocol, must handle missing field.

---

## Conclusion

The [`SUO_UOS_DB_Spec.md`](specs/SUO_UOS_DB_Spec.md) requires significant updates to align with the updated device specifications. The main issues are:

1. **Missing fields in SIF** that are referenced in SUO
2. **Incorrect data structures** (flat vs. object arrays)
3. **Device-specific field handling** not clearly documented
4. **Missing V6800 examples** in SUO schemas

**Priority:** Fix Critical Issues #1-5 first, as these will cause parser/transformer failures. Then address Major Issues #6-11 to ensure correct data handling for both device types.

**Next Steps:**
1. Review and approve this review
2. Update [`SUO_UOS_DB_Spec.md`](specs/SUO_UOS_DB_Spec.md) based on approved changes
3. Update [`v6800_spec.md`](specs/v6800_spec.md) if Decision #1 is approved
4. Validate all changes against device specifications
5. Update database schema if needed

---

**Document End**
