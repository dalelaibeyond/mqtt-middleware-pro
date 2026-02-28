# V6800 Spec Notes Enhancement Plan

## Overview
This plan documents the comprehensive notes to be added/updated for each message type in `specs/v6800_spec.md`. Each note section will clearly describe:
1. Which fields are discarded during conversion
2. How specific fields are created/transformed (e.g., action field)

---

## Part#1 Event or Response Messages (Raw → SIF)

### 1. DEV_MOD_INFO
**Current Status:** No notes section

**Proposed Notes:**
```markdown
// Note:
// Fields discarded from raw to SIF:
//   - gateway_ip: IP address not needed in SIF
//   - gateway_mac: MAC address not needed in SIF
//   - data[].module_type: Module type not needed in SIF
//   - data[].module_m_num: Manufacturing number not needed in SIF
//   - data[].module_supplier: Supplier info not needed in SIF
//   - data[].module_brand: Brand info not needed in SIF
//   - data[].module_model: Model info not needed in SIF
//
// Field transformations:
//   - gateway_sn → deviceId
//   - uuid_number → messageId
//   - data[].module_sn → data[].moduleId
//   - data[].module_index → data[].moduleIndex
//   - data[].module_u_num → data[].uTotal
//   - data[].module_sw_version → data[].fwVer
```

### 2. MOD_CHNG_EVENT
**Current Status:** No notes section

**Proposed Notes:**
```markdown
// Note:
// Fields discarded from raw to SIF:
//   - data[].module_type: Module type not needed in SIF
//   - data[].module_m_num: Manufacturing number not needed in SIF
//   - data[].module_supplier: Supplier info not needed in SIF
//   - data[].module_brand: Brand info not needed in SIF
//   - data[].module_model: Model info not needed in SIF
//
// Field transformations:
//   - gateway_sn → deviceId
//   - uuid_number → messageId
//   - data[].module_sn → data[].moduleId
//   - data[].module_index → data[].moduleIndex
//   - data[].module_u_num → data[].uTotal
//   - data[].module_sw_version → data[].fwVer
```

### 3. HEARTBEAT
**Current Status:** Has notes (line 133)

**Proposed Updated Notes:**
```markdown
// Note:
// deviceId is extracted from module_sn field (not gateway_sn as in other messages).
//
// Fields discarded from raw to SIF:
//   - module_type: Module type not needed in SIF
//   - bus_V: Bus voltage not needed in SIF
//   - bus_I: Bus current not needed in SIF
//   - main_power: Main power status not needed in SIF
//   - backup_power: Backup power status not needed in SIF
//   - data[].module_m_num: Manufacturing number not needed in SIF
//
// Field transformations:
//   - module_sn → deviceId
//   - uuid_number → messageId
//   - data[].module_sn → data[].moduleId
//   - data[].module_index → data[].moduleIndex
//   - data[].module_u_num → data[].uTotal
```

### 4. RFID_EVENT
**Current Status:** Has notes (lines 168-172)

**Proposed Updated Notes:**
```markdown
// Note:
// Fields discarded from raw to SIF:
//   - data[].u_data[].old_state: Old state not needed in SIF
//
// Field transformations:
//   - gateway_sn → deviceId
//   - uuid_number → messageId
//   - data[].host_gateway_port_index → data[].moduleIndex
//   - data[].extend_module_sn → data[].moduleId
//   - data[].u_data[].u_index → data[].data[].sensorIndex
//   - data[].u_data[].tag_code → data[].data[].tagId
//
// Action field creation logic:
//   - new_state = 1, old_state = 0 → action = "ATTACHED"
//   - new_state = 0, old_state = 1 → action = "DETACHED"
//
// isAlarm field creation logic:
//   - warning = 1 → isAlarm = true
//   - warning = 0 → isAlarm = false
```

### 5. RFID_SNAPSHOT
**Current Status:** Has notes (lines 209-211)

**Proposed Updated Notes:**
```markdown
// Note:
// Fields discarded from raw to SIF:
//   - code: Response code not needed in SIF
//   - data[].u_data[].u_state: State not needed in SIF (snapshot only)
//
// Field transformations:
//   - gateway_sn → deviceId
//   - uuid_number → messageId
//   - data[].host_gateway_port_index → data[].moduleIndex
//   - data[].extend_module_sn → data[].moduleId
//   - data[].u_data[].u_index → data[].data[].sensorIndex
//   - data[].u_data[].tag_code → data[].data[].tagId
//
// isAlarm field creation logic:
//   - warning = 1 → isAlarm = true
//   - warning = 0 → isAlarm = false
```

### 6. TEMP_HUM
**Current Status:** Has notes (lines 251-252)

**Proposed Updated Notes:**
```markdown
// Note:
// Fields discarded from raw to SIF:
//   - None (all fields are used)
//
// Field transformations:
//   - gateway_sn → deviceId
//   - uuid_number → messageId
//   - data[].host_gateway_port_index → data[].moduleIndex
//   - data[].extend_module_sn → data[].moduleId
//   - data[].th_data[].temper_position → data[].data[].sensorIndex
//   - data[].th_data[].temper_swot → data[].data[].temp
//   - data[].th_data[].hygrometer_position → data[].data[].sensorIndex (same index)
//   - data[].th_data[].hygrometer_swot → data[].data[].hum
//
// Formatting:
//   - Format temper_swot and hygrometer_swot to float "xx.xx"
```

### 7. QUERY_TEMP_HUM_RESP
**Current Status:** Has notes (lines 289-290)

**Proposed Updated Notes:**
```markdown
// Note:
// Fields discarded from raw to SIF:
//   - code: Response code not needed in SIF
//
// Field transformations:
//   - gateway_sn → deviceId
//   - uuid_number → messageId
//   - data[].host_gateway_port_index → data[].moduleIndex
//   - data[].extend_module_sn → data[].moduleId
//   - data[].th_data[].temper_position → data[].data[].sensorIndex
//   - data[].th_data[].temper_swot → data[].data[].temp
//   - data[].th_data[].hygrometer_position → data[].data[].sensorIndex (same index)
//   - data[].th_data[].hygrometer_swot → data[].data[].hum
//
// Formatting:
//   - Format temper_swot and hygrometer_swot to float "xx.xx"
```

### 8. DOOR_STATE_EVENT
**Current Status:** Has notes (lines 317-319)

**Proposed Updated Notes:**
```markdown
// Note:
// Fields discarded from raw to SIF:
//   - None (all fields are transformed)
//
// Field transformations:
//   - gateway_sn → deviceId
//   - uuid_number → messageId
//   - data[].extend_module_sn → data[].moduleId
//   - data[].host_gateway_port_index → data[].moduleIndex
//
// door1State and door2State unification logic:
//   - Single door sensor (has new_state field):
//     * raw.data[].new_state → sif.data[].door1State
//     * sif.data[].door2State = null
//   - Dual door sensor (has new_state1 and new_state2 fields):
//     * raw.data[].new_state1 → sif.data[].door1State
//     * raw.data[].new_state2 → sif.data[].door2State
```

### 9. QUERY_DOOR_STATE_RESP
**Current Status:** Has notes (lines 361-366)

**Proposed Updated Notes:**
```markdown
// Note:
// deviceId is extracted from MQTT topic: V6800Upload/{deviceId}/Door
// Example: V6800Upload/2437871205/Door → deviceId = "2437871205"
//
// Fields discarded from raw to SIF:
//   - code: Response code not needed in SIF
//   - gateway_port_index: Moved to data[].moduleIndex
//
// Field transformations:
//   - Topic[1] → deviceId
//   - uuid_number → messageId
//   - gateway_port_index → data[].moduleIndex
//
// door1State and door2State unification logic:
//   - Single door sensor (has new_state field):
//     * raw.new_state → sif.data[0].door1State
//     * sif.data[0].door2State = null
//   - Dual door sensor (has new_state1 and new_state2 fields):
//     * raw.new_state1 → sif.data[0].door1State
//     * raw.new_state2 → sif.data[0].door2State
```

### 10. SET_COLOR_RESP
**Current Status:** Has notes (lines 395-397)

**Proposed Updated Notes:**
```markdown
// Note:
// Fields discarded from raw to SIF:
//   - set_property_type: Property type identifier not needed in SIF
//   - data[].module_type: Module type not needed in SIF
//
// Field transformations:
//   - gateway_sn → deviceId
//   - uuid_number → messageId
//   - data[].host_gateway_port_index → data[].moduleIndex
//   - data[].extend_module_sn → data[].moduleId
//
// result field creation logic:
//   - set_property_result = 1 → result = "Failure"
//   - set_property_result = 0 → result = "Success"
```

### 11. QUERY_COLOR_RESP
**Current Status:** Has notes (line 434)

**Proposed Updated Notes:**
```markdown
// Note:
// Fields discarded from raw to SIF:
//   - code: Response code not needed in SIF (represents overall status, not per-sensor data)
//   - count: Count field not needed in SIF
//
// Field transformations:
//   - gateway_id → deviceId
//   - uuid_number → messageId
//   - data[].index → data[].moduleIndex
//   - data[].module_id → data[].moduleId
//   - data[].u_num → data[].uTotal
//   - data[].color_data[].index → data[].data[].sensorIndex
//   - data[].color_data[].color → data[].data[].colorName
//   - data[].color_data[].code → data[].data[].colorCode
```

### 12. CLEAR_ALARM_RESP
**Current Status:** Has notes (lines 463-465)

**Proposed Updated Notes:**
```markdown
// Note:
// Fields discarded from raw to SIF:
//   - count: Count field not needed in SIF
//   - code: Response code not needed in SIF
//   - data[].ctr_flag: Control flag transformed to result field
//
// Field transformations:
//   - gateway_id → deviceId
//   - uuid_number → messageId
//   - data[].index → data[].moduleIndex
//   - data[].module_id → data[].moduleId
//   - data[].u_num → data[].uTotal
//
// result field creation logic:
//   - ctr_flag = true → result = "Success"
//   - ctr_flag = false → result = "Failure"
```

---

## Part#2 Query or Set Command Messages (SIF → Raw)

### 2.1 QUERY_DEV_MOD_INFO
**Current Status:** No notes section

**Proposed Notes:**
```markdown
// Note:
// Fields discarded from SIF to Raw:
//   - deviceType: Device type not sent in raw message
//   - messageType: Message type not sent in raw message
//   - data: Empty data object not sent in raw message
//
// Field transformations:
//   - deviceId → Not used (extracted from topic in middleware)
//   - messageType "QUERY_DEV_MOD_INFO" → msg_type "get_devies_init_req"
//   - msg_code = 200 (fixed value)
```

### 2.2 QUERY_RFID_SNAPSHOT
**Current Status:** No notes section

**Proposed Notes:**
```markdown
// Note:
// Fields discarded from SIF to Raw:
//   - deviceType: Device type not sent in raw message
//   - messageType: Message type not sent in raw message
//   - data.moduleId: Not sent (extend_module_sn set to null)
//   - data.sensorIndex: Not sent (u_index_list set to null)
//
// Field transformations:
//   - deviceId → gateway_sn
//   - data.moduleIndex → data[].host_gateway_port_index
//   - messageType "QUERY_RFID_SNAPSHOT" → msg_type "u_state_req"
//   - data[].extend_module_sn = null
//   - data[].u_index_list = null
```

### 2.3 QUERY_DOOR_STATE
**Current Status:** No notes section

**Proposed Notes:**
```markdown
// Note:
// Fields discarded from SIF to Raw:
//   - deviceType: Device type not sent in raw message
//   - messageType: Message type not sent in raw message
//   - data.moduleId: Not sent (extend_module_sn set to null)
//
// Field transformations:
//   - deviceId → gateway_sn
//   - data.moduleIndex → host_gateway_port_index
//   - messageType "QUERY_DOOR_STATE" → msg_type "door_state_req"
//   - extend_module_sn = null
```

### 2.4 QUERY_TEMP_HUM
**Current Status:** No notes section

**Proposed Notes:**
```markdown
// Note:
// Fields discarded from SIF to Raw:
//   - deviceType: Device type not sent in raw message
//   - messageType: Message type not sent in raw message
//   - data.moduleId: Not sent (extend_module_sn set to null)
//
// Field transformations:
//   - deviceId → gateway_sn
//   - data.moduleIndex → data[0] (array of port indices)
//   - messageType "QUERY_TEMP_HUM" → msg_type "temper_humidity_req"
//   - extend_module_sn = null
//   - data = [moduleIndex] (array format)
```

### 2.5 SET_COLOR
**Current Status:** No notes section

**Proposed Notes:**
```markdown
// Note:
// Fields discarded from SIF to Raw:
//   - deviceType: Device type not sent in raw message
//   - messageType: Message type not sent in raw message
//   - data.moduleId: Not sent (extend_module_sn set to null)
//
// Field transformations:
//   - deviceId → gateway_sn
//   - data.moduleIndex → data[].host_gateway_port_index
//   - data.sensorIndex → data[].u_color_data[].u_index
//   - data.colorCode → data[].u_color_data[].color_code
//   - messageType "SET_COLOR" → msg_type "set_module_property_req"
//   - set_property_type = 8001 (fixed value)
//   - module_type = 2 (fixed value)
//   - extend_module_sn = null
```

### 2.6 QUERY_COLOR
**Current Status:** No notes section

**Proposed Notes:**
```markdown
// Note:
// Fields discarded from SIF to Raw:
//   - deviceType: Device type not sent in raw message
//   - messageType: Message type not sent in raw message
//   - data.sensorIndex: Not sent in raw message
//
// Field transformations:
//   - deviceId → Not used (extracted from topic in middleware)
//   - data.moduleIndex → data[0] (array of port indices)
//   - messageType "QUERY_COLOR" → msg_type "get_u_color"
//   - code = 0 (default value, may vary)
//   - data = [moduleIndex] (array format)
```

### 2.7 CLEAR_ALARM
**Current Status:** No notes section

**Proposed Notes:**
```markdown
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

---

## Summary of Changes

### Messages requiring NEW notes sections:
1. DEV_MOD_INFO
2. MOD_CHNG_EVENT
3. QUERY_DEV_MOD_INFO (2.1)
4. QUERY_RFID_SNAPSHOT (2.2)
5. QUERY_DOOR_STATE (2.3)
6. QUERY_TEMP_HUM (2.4)
7. SET_COLOR (2.5)
8. QUERY_COLOR (2.6)
9. CLEAR_ALARM (2.7)

### Messages requiring UPDATED notes sections:
1. HEARTBEAT - Add discarded fields and field transformations
2. RFID_EVENT - Add discarded fields and reorganize
3. RFID_SNAPSHOT - Add discarded fields and reorganize
4. TEMP_HUM - Add field transformations
5. QUERY_TEMP_HUM_RESP - Add discarded fields and field transformations
6. DOOR_STATE_EVENT - Add field transformations and reorganize
7. QUERY_DOOR_STATE_RESP - Add discarded fields and reorganize
8. SET_COLOR_RESP - Add discarded fields and reorganize
9. QUERY_COLOR_RESP - Add discarded fields and field transformations
10. CLEAR_ALARM_RESP - Add discarded fields and reorganize

---

## Implementation Notes

- All notes should follow a consistent format with clear sections:
  - Fields discarded (if any)
  - Field transformations (mapping from raw to SIF or vice versa)
  - Special field creation logic (e.g., action, isAlarm, result fields)
  - Formatting rules (if any)

- Use bullet points or numbered lists for clarity
- Keep descriptions concise but complete
- Use code comments style (//) for consistency with existing notes
