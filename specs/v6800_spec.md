# V6800 Device MQTT Message Spec

**File Name**: `v6800_spec.md`

**Version**: v1.2

**Date**: 2/22/2026

**Scope**: V6800 device raw JSON message format and Spec from raw to SIF (Standard Intermediate Format) Conversion

---

---

## Part#1 Event or Response Messages (Device → Broker → App)

### 1. `DEV_MOD_INFO`

**Topic:** `V6800Upload/{deviceId}/Init`

**Message Format:**

```json
// Raw format
{
  "msg_type": "devies_init_req",
  "gateway_sn": "string",
  "gateway_ip": "string",
  "gateway_mac": "string",
  "uuid_number": "messageId",
  "data": [{
      "module_type": "string",
      "module_index": "string",
      "module_sn": "int",
      "module_m_num": "int",
      "module_u_num": "int",
      "module_sw_version": "string",
      "module_supplier": "string",
      "module_brand": "string",
      "module_model": "string"
    }]
}

// Spec from Raw → SIF
{
  "meta": { "topic": "...", "rawType": "msg_type" },
  "deviceId": "gateway_sn",
  "deviceType": "V6800",
  "messageType": "DEV_MOD_INFO",
  "messageId": "uuid_number",
  "ip": "gateway_ip",
  "mac": "gateway_mac",
  "data": [{ "moduleIndex": module_index, "fwVer": "module_sw_version", "moduleId": "module_sn", "uTotal": module_u_num }]
}
```

### 2. `MOD_CHNG_EVENT`

**Topic:** `V6800Upload/{deviceId}/DeviceChange`

**Message Format:**

```json
// Raw format
{
  "msg_type": "devices_changed_req",
  "gateway_sn": "string",
  "uuid_number": "string",
  "data": [
    {
      "module_type": "string",
      "module_sn": "string",
      "module_index": "int",
      "module_m_num": "int",
      "module_u_num": "int",
      "module_sw_version": "string",
      "module_supplier": "string",
      "module_brand": "string",
      "module_model": "string"
    }
  ]
}

// Spec from Raw → SIF
{
  "meta": { "topic": "...", "rawType": "msg_type" },
  "deviceId": "gateway_sn",
  "deviceType": "V6800",
  "messageType": "MOD_CHNG_EVENT",
  "messageId": "uuid_number",
  "data": [{ "moduleIndex": module_index, "fwVer": "module_sw_version", "moduleId": "module_sn", "uTotal": module_u_num }]
}
```

### 3. `HEARTBEAT`

**Topic:** `V6800Upload/{deviceId}/HeartBeat`

**Message Format:**

```json
// Raw format
{
  "msg_type": "heart_beat_req",
  "module_type": "string",
  "module_sn": "string",
  "bus_V": "string",
  "bus_I": "string",
  "main_power": "int",
  "backup_power": "int",
  "uuid_number": "string",
  "data": [
    {
      "module_index": "int",
      "module_sn": "string",
      "module_m_num": "int",
      "module_u_num": "int"
    }
  ]
}

// Spec from Raw → SIF
{
  "meta": { "topic": "...", "rawType": "msg_type" },
  "deviceId": "module_sn",
  "deviceType": "V6800",
  "messageType": "HEARTBEAT",
  "messageId": "uuid_number",
  "data": [{ "moduleIndex": module_index, "moduleId": "module_sn", "uTotal": module_u_num }]
}
```

**Note:** deviceId is extracted from `module_sn` field (not `gateway_sn` as in other messages). The following raw fields are intentionally discarded in SIF: `module_type`, `bus_V`, `bus_I`, `main_power`, `backup_power`.

### 4. `RFID_EVENT`

**Topic:** `V6800Upload/{deviceId}/LabelState`

**Message Format:**

```json
// Raw format
{
  "msg_type": "u_state_changed_notify_req",
  "gateway_sn": "string",
  "uuid_number": "string",
  "data": [{
      "host_gateway_port_index": "int",
      "extend_module_sn": "string",
      "u_data": [{"u_index": "int", "new_state": "int","old_state": "int", "tag_code": "string","warning": "int"}]
    }]
}

// Spec from Raw → SIF
{
  "meta": { "topic": "...", "rawType": "msg_type" },
  "deviceId": "gateway_sn",
  "deviceType": "V6800",
  "messageType": "RFID_EVENT",
  "messageId": "uuid_number",
  "data": [{
      "moduleIndex": host_gateway_port_index,
      "moduleId": "extend_module_sn",
      "data": [{ "sensorIndex": u_index, "action": "DETACHED", "tagId": "tag_code", "isAlarm": false|true }]
  }]
}

// Note:
1) new_state = 1, old_state = 0 => action = "ATTACHED"
2) new_state = 0, old_state = 1 => action = "DETACHED"
3) warning = 1 => isAlarm = true
4) warning = 0 => isAlarm = false
```

### 5. `RFID_SNAPSHOT`

**Topic:** `V6800Upload/{deviceId}/LabelState`

**Message Format:**

```json
// Raw format
{
  "msg_type": "u_state_resp",
  "code": "int",
  "gateway_sn": "string",
  "uuid_number": "string",
  "data": [{
      "extend_module_sn": "string",
      "host_gateway_port_index": "int",
      "u_data": [{ "u_index": "int", "u_state": "int","tag_code": "string", "warning": "int"}]
    }]
}

// Spec from Raw → SIF
{
  "meta": { "topic": "...", "rawType": "msg_type" },
  "deviceId": "gateway_sn",
  "deviceType": "V6800",
  "messageType": "RFID_SNAPSHOT",
  "messageId": "uuid_number",
  "data": [{
      "moduleIndex": host_gateway_port_index,
      "moduleId": "extend_module_sn",
      "data": [{ "sensorIndex": u_index, "tagId": "tag_code", "isAlarm": false|true }]
  }]
}

// Note:
1) warning = 1 => isAlarm = true
3) warning = 0 => isAlarm = false
```

### 6. `TEMP_HUM`

**Topic:** `V6800Upload/{deviceId}/TemHum`

**Message Format:**

```json
// Raw format
// Note: Device may send "temper_humidity_exception_nofity_req" (typo: "nofity" instead of "notify")
// Both spellings are accepted and mapped to TEMP_HUM messageType
{
  "msg_type": "temper_humidity_exception_notify_req",
  "gateway_sn": "string",
  "uuid_number": "string",
  "data": [
    {
      "host_gateway_port_index": "int",
      "extend_module_sn": "string",
      "th_data": [{"temper_position": "int", "temper_swot": "float",   "hygrometer_position": "int",
          "hygrometer_swot": "float"
        }]
    }]
}
// Spec from Raw → SIF
{
  "meta": { "topic": "...", "rawType": "msg_type" },
  "deviceId": "gateway_sn",
  "deviceType": "V6800",
  "messageType": "TEMP_HUM",
  "messageId": "uuid_number",
  "data": [{
      "moduleIndex": host_gateway_port_index,
      "moduleId": "extend_module_sn",
      "data": [{ "sensorIndex": temper_position, "temp": temper_swot, "hum": hygrometer_swot}]
  }]
}

// Note:
1) format temper_swot and hygrometer_swot to float "xx.xx"
```

### 7. `QUERY_TEMP_HUM_RESP`

**Topic:** `V6800Upload/{deviceId}/TemHum`

**Message Format:**

```json
// Raw format
{
  "msg_type": "temper_humidity_resp",
  "code": "int",
  "gateway_sn": "string",
  "uuid_number": "string",
  "data": [{
      "extend_module_sn": "string",
      "host_gateway_port_index": "int",
      "th_data": [{"temper_position": "int","temper_swot": "float","hygrometer_position": "int", "hygrometer_swot": "float" }]
    }]
}

// Spec from Raw → SIF
{
  "meta": { "topic": "...", "rawType": "msg_type" },
  "deviceId": "gateway_sn",
  "deviceType": "V6800",
  "messageType": "QUERY_TEMP_HUM_RESP",
  "messageId": "uuid_number",
  "data": [{
      "moduleIndex": host_gateway_port_index,
      "moduleId": "extend_module_sn",
      "data": [{ "sensorIndex": temper_position, "temp": temper_swot, "hum": hygrometer_swot}]
    }]
}

// Note:
1) format temper_swot and hygrometer_swot to float "xx.xx"
```

### 8. `DOOR_STATE_EVENT`

**Topic:** `V6800Upload/{deviceId}/Door`

**Message Format:**

```json
// Raw format
{
  "msg_type": "door_state_changed_notify_req",
  "gateway_sn": "string",
  "uuid_number": "string",
  "data": [{ "extend_module_sn": "string", "host_gateway_port_index": "int",  ("new_state": "int") | ("new_state1": "int",  "new_state2": "int" ) }]
}

// Spec from Raw → SIF
{
  "meta": { "topic": "...", "rawType": "msg_type" },
  "deviceId": "gateway_sn",
  "deviceType": "V6800",
  "messageType": "DOOR_STATE",
  "messageId": "uuid_number",
  "data": [{ "moduleId": "extend_module_sn", "moduleIndex": host_gateway_port_index, "door1State": 1, "door2State": 1 }]
}
// Note: In SIF, unify new_state, new_state1, new_state2 to door1State and door2State, rule as below:
// 1) if single door sensor, raw.data.new_state → sif.data.door1State, sif.data.door2State = null
// 2) if dual door sensor, raw.data.new_state1 → sif.data.door1State, raw.data.new_state2 → sif.data.door2State
```

### 9. `QUERY_DOOR_STATE_RESP`

**Topic:** `V6800Upload/{deviceId}/Door`

**Message Format:**

```json
// Raw format (single door)
{
  "msg_type": "door_state_resp",
  "code": "int",
  "gateway_port_index": "int",
  "new_state": "int",
  "uuid_number": "string"
}

// Raw format (dual door)
{
  "msg_type": "door_state_resp",
  "code": "int",
  "gateway_port_index": "int",
  "new_state1": "int",
  "new_state2": "int",
  "uuid_number": "string"
}

// Spec from Raw → SIF
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
// Note:
// deviceId is extracted from MQTT topic: V6800Upload/{deviceId}/Door
// Example: V6800Upload/2437871205/Door → deviceId = "2437871205"
// In SIF, unify new_state, new_state1, new_state2 to door1State and door2State, rule as below:
// 1) if single door sensor, raw.new_state → sif.door1State, door2State = null
// 2) if dual door sensor, raw.new_state1 → sif.door1State, raw.new_state2 → sif.door2State

```

### 10. `SET_COLOR_RESP`

**Topic:** `V6800Upload/{deviceId}/OpeAck`

**Message Format:**

```json
// Raw format
{
  "msg_type": "set_module_property_result_req",
  "set_property_type": 8001,
  "gateway_sn": "string",
  "uuid_number": "string",
  "data": [{ "host_gateway_port_index": "int", "extend_module_sn": "string", "module_type": "string", "set_property_result": "int" }]
}

// Spec from Raw → SIF
{
  "meta": { "topic": "...", "rawType": "msg_type" },
  "deviceId": "gateway_sn",
  "deviceType": "V6800",
  "messageType": "SET_COLOR_RESP",
  "messageId": "uuid_number",
  "data": [{ "moduleIndex":host_gateway_port_index, "moduleId": "extend_module_sn", "result": "Success"|"Failure" }]
}
// Note:
// 1) raw.data.set_property_result = 1 → sif.data.result = "Failure"
// 2) raw.data.set_property_result = 0 → sif.data.result = "Success"

```

### 11. `QUERY_COLOR_RESP`

**Topic:** `V6800Upload/{deviceId}/OpeAck`

**Message Format:**

```json
// Raw format
{
  "msg_type": "u_color",
  "gateway_id": "string",
  "count": "int",
  "code": "int",
  "uuid_number": "string",
  "data": [{"index": "int", "module_id": "string", "u_num": "int", 
				    "color_data": [{"index": "int", "color": "string", "code": "int" }]
    }]
}

// Spec from Raw → SIF
{
  "meta": { "topic": "...", "rawType": "msg_type" },
  "deviceId": "gateway_id",
  "deviceType": "V6800",
  "messageType": "QUERY_COLOR_RESP",
  "messageId": "uuid_number",
  "data": [{
      "moduleIndex": index,
      "moduleId": "module_id",
      "uTotal": u_num,
      "data": [{ "sensorIndex": index, "colorName": "color", "colorCode": code }]
    }]
}
// Note: The `code` field at the top level of the raw message is ignored in SIF as it represents the overall response status, not per-sensor data.
```

### 12. `CLEAR_ALARM_RESP`

**Topic:** `V6800Upload/{deviceId}/OpeAck`

**Message Format:**

```json
// Raw format
{
  "msg_type": "clear_u_warning",
  "gateway_id": "string",
  "count": "int",
  "code": "int",
  "uuid_number": "string",
  "data": [{ "index": "int", "module_id": "string", "u_num": "int", "ctr_flag": "bool" }]
}

// Spec from Raw → SIF
{
  "meta": { "topic": "...", "rawType": "msg_type" },
  "deviceId": "gateway_id",
  "deviceType": "V6800",
  "messageType": "CLEAR_ALARM_RESP",
  "messageId": "uuid_number",
  "data": [{ "moduleIndex": index, "moduleId": "module_id", "uTotal": u_num, "result": "Success"|"Failure" }]
}
// Note:
// ctr_flag = true → result = "Success"
// ctr_flag = false → result = "Failure"
```

## Part#2 Query or Set Command Messages (App → Broker → Device)

### 2.1 `QUERY_DEV_MOD_INFO`

**Topic:** `V6800Download/{deviceId}`

**Message Format:**

```json
// Raw
{
  "msg_type": "get_devies_init_req",
  "msg_code": 200
}

// SIF
{
  "deviceId": "2437871205",
  "deviceType": "V6800",
  "messageType": "QUERY_DEV_MOD_INFO",
  "data": {}
}
```

### 2.2 `QUERY_RFID_SNAPSHOT`

**Topic:** `V6800Download/{deviceId}`

**Message Format:**

```json
// Raw format
{
  "msg_type": "u_state_req",
  "gateway_sn": "string",
  "data": [{"extend_module_sn": null, "host_gateway_port_index": 4, "u_index_list": null}]
}

// SIF
{
  "deviceId": "2437871205",
  "deviceType": "V6800", 
  "messageType": "QUERY_RFID_SNAPSHOT",
  "data": {"moduleIndex": 4}
}
```

### 2.3 `QUERY_DOOR_STATE`

**Topic:** `V6800Download/{deviceId}`

**Message Format:**

```json
// Raw format
{
  "msg_type": "door_state_req",
  "gateway_sn": "string",
  "extend_module_sn": "string",
  "host_gateway_port_index": 4
}

// SIF Format
{
  "deviceId": "2437871205",
  "deviceType": "V6800", 
  "messageType": "QUERY_DOOR_STATE",
  "data": {"moduleIndex": 4}
}
```

### 2.4 `QUERY_TEMP_HUM`

**Topic:** `V6800Download/{deviceId}`

**Message Format:**

```json
// Raw format
{
  "msg_type": "temper_humidity_req",
  "gateway_sn": "string",
  "extend_module_sn": null,
  "data": [4]
}

// SIF format
{
  "deviceId": "2437871205",
  "deviceType": "V6800", 
  "messageType": "QUERY_TEMP_HUM",
  "data": {"moduleIndex": 4}
}
```

### 6. `SET_COLOR`

Topic: `V6800Download/{deviceId}`

Message Format

```jsx
//raw
{
  "msg_type" : "set_module_property_req",
  "gateway_sn" : "2437871205",
  "set_property_type" : 8001,
  "data" : [ {
    "host_gateway_port_index" : 1,
    "extend_module_sn" : null,
    "module_type" : 2,
    "u_color_data" : [ {"u_index" : 10, "color_code" : 1} ]
  } ]
}

//sif
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

### 2.6 `QUERY_COLOR`

**Topic:** `V6800Download/{deviceId}`

**Message Format:**

```json
// Raw format
{
  "msg_type": "get_u_color",
  "code": "int",
  "data": [4]
}
// SIF format
{
  "deviceId": "2437871205",
  "deviceType": "V6800", 
  "messageType": "QUERY_COLOR",
  "data": {"moduleIndex": 4 }
}
```

### 2.7 `CLEAR_ALARM`

**Topic:** `V6800Download/{deviceId}`

**Message Format:**

```json
// Raw format
{
  "msg_type": "clear_u_warning",
  "code": "int",
  "data": [{"index": 4, "warning_data": [10]}]
}

// SIF format
{
  "deviceId": "2437871205",
  "deviceType": "V6800",
  "messageType": "CLEAR_ALARM",
  "data": {"moduleIndex": 4, "sensorIndex": 10}
}
```

## Note

- `{deviceId}`: Replace with the actual global unique ID of the V6800 Device.
- Data type description: `string` (character string), `int` (integer), `float` (floating point number), `array` (array), `bool` (boolean: true/false).
- The fields with no actual value in the message can be filled with `null` according to the business scenario.

## Field Naming Convention

**Note:** Raw message fields use snake_case (e.g., `module_sn`, `u_index`) while SIF fields use camelCase (e.g., `moduleId`, `sensorIndex`). This transformation is intentional for consistency.

## Data Type Handling

- If a raw field is `null` or missing, set the corresponding SIF field to `null`
- String fields: Trim whitespace and validate format
- Numeric fields: Validate range and convert to appropriate type
- Boolean fields: Convert integer `0`/`1` to `false`/`true`

## Error Handling

- If required fields are missing, log error and skip message
- If data types don't match expected format, log warning and use default/null values
- If message format is unrecognizable, log error with raw payload for debugging