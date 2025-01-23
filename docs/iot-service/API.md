# API Reference

- Ver: 1.0.1

## Host

| Domain Address                         |
|----------------------------------------|
| https://carencoinc.com/kr/iot-service/ |

## Device Status Check

### Basic Information

- Checks the device's status, including sensor status, battery information.

| Method | URL                                           |
|--------|-----------------------------------------------|
| GET    | `https://carencoinc.com/kr/iot-service/scale` |

### Request

#### Parameters(@RequestBody)

| Name           | Type   | Description                          | Required |
|----------------|--------|--------------------------------------|----------|
| `uid`          | String | Unique Identifier of User.           | Yes      |
| `macAddress`   | String | MAC address of the device.           | Yes      |
| `version`      | String | Version information of the device.   | Yes      |
| `batteryLevel` | String | Battery level of the device.         | Yes      |
| `sensorStatus` | String | Status of the sensors in the device. | Yes      |

### Response

#### Body

| Name      | Type   | Description                        |
|-----------|--------|------------------------------------|
| `message` | String | Device status check result message |

#### Example

##### Request

```bash
curl -X 'GET' \
'https://carencoinc.com/kr/iot-service/scale' \
-H 'Content-Type: application/json' \
-d '{
      "uid" : "Test_UID",
      "macAddress" : "Test_MacAddress",
      "version" : "Test_Version",
      "batteryLevel" : "Test_BatteryLevel",
      "sensorStatus" : "Test_SensorStatus"
}'
```

##### Response

###### 200 OK

```json
{
  "message": "Scale check passed"
}
```

###### 400 BadRequest

```json
{
  "error": "Bad Request",
  "message": "Invalid parameters"
}
```

---

## Device OTA Update

### Basic Information

- Performs an OTA update on the device.

| Method | URL                                               |
|--------|---------------------------------------------------|
| GET    | `https://carencoinc.com/kr/iot-service/scale/ota` |

### Request

#### Parameters(@RequestParam)

| Name      | Type   | Description                        | Required |
|-----------|--------|------------------------------------|----------|
| `version` | String | Version information of the device. | Yes      |

### Response

#### Body

| Name      | Type   | Description                 |
|-----------|--------|-----------------------------|
| `message` | String | Status check result message |

#### Example

##### Request

```bash
curl -X 'GET' \
'https://carencoinc.com/kr/iot-service/scale/ota?version='
```

##### Response

###### 200 OK

```json
{
  "message": "OTA update"
}
```

###### 400 BadRequest

```json
{
  "error": "Bad Request",
  "message": "Invalid parameters"
}
```

---

## Measurement Recording

### Basic Information

- Records measurement data from the device.

| Method | URL                                           |
|--------|-----------------------------------------------|
| POST   | `https://carencoinc.com/kr/iot-service/scale` |

### Request

#### Parameters(@RequestBody)

| Name         | Type   | Description                                           | Required |
|--------------|--------|-------------------------------------------------------|----------|
| `uid`        | String | Unique Identifier of the device.                      | Yes      |
| `macAddress` | String | MAC address of the device.                            | Yes      |
| `rawData`    | Byte   | The raw measurement data from the device.             | Yes      |
| `timestamp`  | String | The time the measurement was taken.                   | Yes      |
| `region`     | String | MRegion information where the measurement took place. | No       |

### Response

#### Body

| Name      | Type   | Description                                                          |
|-----------|--------|----------------------------------------------------------------------|
| `message` | String | Measurement data recorded with UID, Data, Time, MAC, and Region info |

#### Example

##### Request

```bash
curl -X 'POST' \
'https://carencoinc.com/kr/iot-service/scale' \
-H 'Content-Type: application/json' \
-d '[
    {
        "uid" : "Test_UID",
        "macAddress" : "Test_MacAddress",
        "rawData" : "VGVzdF9EYXRh",
        "timestamp" : "Test_Stamp",
        "region" : "Test_Region"
    },
    {
        "uid" : "Test_UID",
        "macAddress" : "Test_MacAddress",
        "rawData" : "VGVzdF9EYXRh",
        "timestamp" : "Test_Stamp",
        "region" : "Test_Region"
    }
]'
```

##### Response

###### 200 OK

```json
{
  "message": "Measurement recorded for 2 entries."
}
```

###### 400 BadRequest

```json
{
  "error": "Bad Request",
  "message": "Invalid parameters"
}
```