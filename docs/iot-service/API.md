# IoT Service
    - Basic Domain: https://carencoinc.com/kr/iot-service/
    - Version: 1.0.1

<!-- api-1-start -->
<details markdown="1">
<summary><strong>&nbsp;API: Device Status Check</strong></summary>


## Basic Information

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


<details markdown=>
  <summary><strong>Example</strong></summary>

## Request

### Postman 요청

아래 버튼을 클릭하면 `Postman`에서 API 요청을 실행할 수 있습니다.

[![Run in Postman](https://run.pstmn.io/button.svg)](https://carenco.postman.co/workspace/Care%26CO~7c4d2551-cc9d-413f-b156-4c350b99eb32/request/27911837-3413a747-26c6-4b63-a406-5992e43ad072?action=share&creator=32584424&ctx=documentation)

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

## Response

<details>
<summary><strong>200 OK</strong></summary>

```json
{
  "message": "Scale check passed"
}
```
</details>

<details>
<summary><strong>400 BadRequest</strong></summary>

```json
{
  "error": "Bad Request",
  "message": "Invalid parameters"
}
```

</details>

</details>

---

</details>
<!-- api-1-end -->


<!-- api-2-start -->
<details markdown="1">
<summary><strong>&nbsp;API: Device OTA Update</strong></summary>


## Basic Information

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

<details markdown=>
  <summary><strong>Example</strong></summary>

## Request

### Postman 요청

아래 버튼을 클릭하면 `Postman`에서 API 요청을 실행할 수 있습니다.

[![Run in Postman](https://run.pstmn.io/button.svg)](https://carenco.postman.co/workspace/Care%26CO~7c4d2551-cc9d-413f-b156-4c350b99eb32/request/27911837-613ffcc6-eeb0-4308-9bc7-5c8f08214c60?action=share&creator=32584424&ctx=documentation)


```bash
curl -X 'GET' \
'https://carencoinc.com/kr/iot-service/scale/ota?version='
```

## Response

<details>
<summary><strong>200 OK</strong></summary>

```json
{
  "message": "OTA update"
}
```

</details>

<details>
<summary><strong>400 BadRequest</strong></summary>

```json
{
  "error": "Bad Request",
  "message": "Invalid parameters"
}
```

</details>

</details>

---

</details>
<!-- api-2-end -->

<!-- api-3-start -->
<details markdown="1">
<summary><strong>&nbsp;API: Measurement Recording</strong></summary>


## Basic Information

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

<details markdown=>
  <summary><strong>Example</strong></summary>

## Request

### Postman 요청

아래 버튼을 클릭하면 `Postman`에서 API 요청을 실행할 수 있습니다.

[![Run in Postman](https://run.pstmn.io/button.svg)](https://carenco.postman.co/workspace/Care%26CO~7c4d2551-cc9d-413f-b156-4c350b99eb32/request/27911837-057af183-fbc0-4599-991b-20168d4eda9c?action=share&creator=32584424&ctx=documentation)

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

## Response

<details>
<summary><strong>200 OK</strong></summary>

```json
{
  "message": "Measurement recorded for 2 entries."
}
```

</details>

<details>
<summary><strong>400 BadRequest</strong></summary>

```json
{
  "error": "Bad Request",
  "message": "Invalid parameters"
}
```
</details>

</details>

---

</details>
<!-- api-3-end -->