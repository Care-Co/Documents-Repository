# Measurement-Service

    - Basic Domain: https://carencoinc.com/kr/measurement-service
    - Basic IP: 15.165.125.100:8084


<!-- api-1-start -->
<details markdown="1">
<summary><strong>&nbsp;API: CreateFootprintRecord</strong></summary>



## Basic Information

| Method | URL              |
|--------|------------------|
| POST   | `/v2/footprints` |

### Request

#### Parameters(@RequestParam)

| Name        | Type     | Description                                  | Required | Remarks                                                               |
|-------------|----------|----------------------------------------------|----------|-----------------------------------------------------------------------|
| `version`   | String   | API version information (format: YYYY-MM-DD) | No       | If not provided, the latest API version will be used automatically.   |

#### Parameters(@RequestBody)
| Name                 | Type          | Description                                               | Required | Remarks |
|----------------------|---------------|-----------------------------------------------------------|----------|---------|
| `userId`             | String        | User Unique identifier                                    | Yes      |         |
| `measuredDateTime`   | LocalDateTime | Measurement date and time                                 | Yes      |         |
| `rawData`            | String        | Measured raw data                                         | Yes      |         |
| `firstClassType`     | Double        | The first-ranked plantar pressure class type              | Yes      |         |
| `firstAccuracy`      | Double        | The accuracy (similarity) of the first-ranked class type  | Yes      |         |
| `secondaryClassType` | Double        | The second-ranked plantar pressure class type             | Yes      |         |
| `secondaryAccuracy`  | Double        | The accuracy (similarity) of the second-ranked class type | Yes      |         |
| `thirdClassType`     | Double        | The third-ranked plantar pressure class type              | Yes      |         |
| `thirdAccuracy`      | Double        | The accuracy (similarity) of the third-ranked class type  | Yes      |         |
| `leftFootLength`     | Double        | Length of the left foot (in millimeters)                  | Yes      |         |
| `leftFootWidth`      | Double        | Width of the left foot (in millimeters)                   | Yes      |         |
| `rightFootLength`    | Double        | Length of the right foot (in millimeters)                 | Yes      |         |
| `rightFootWidth`     | Double        | Width of the right foot (in millimeters)                  | Yes      |         |
| `userId`             | String        | URL of the saved plantar pressure image                   | Yes      |         |
| `weight`             | Double        | The user's weight (in kilograms)                          | Yes      |         |


### Response

#### Body

| Name                      | Type          | Description                                                                      |
|---------------------------|---------------|----------------------------------------------------------------------------------|
| `message`                 | String        | The result message of the API call (e.g., plantar pressure measurement complete) |
| `data`                    | Object        | Contains data related to the user's plantar pressure                             |
| `data.id`                 | UUID          | Unique identifier (ID) for the plantar pressure record                           |
| `data.userId`             | String        | ID of the user who requested the measurement                                     |
| `data.measuredDateTime`   | LocalDateTime | The date and time when the plantar pressure measurement was taken                |
| `data.firstClassType`     | Double        | The first-ranked plantar pressure class type                                     |
| `data.firstAccuracy`      | Double        | The accuracy (similarity) of the first-ranked class type                         |
| `data.secondaryClassType` | Double        | The second-ranked plantar pressure class type                                    |
| `data.secondaryAccuracy`  | Double        | The accuracy (similarity) of the second-ranked class type                        |
| `data.thirdClassType`     | Double        | The third-ranked plantar pressure class type                                     |
| `data.thirdAccuracy`      | Double        | The accuracy (similarity) of the third-ranked class type                         |
| `data.leftFootLength`     | Double        | Length of the left foot (in millimeters)                                         |
| `data.leftFootWidth`      | Double        | Width of the left foot (in millimeters)                                          |
| `data.rightFootLength`    | Double        | Length of the right foot (in millimeters)                                        |
| `data.rightFootWidth`     | Double        | Width of the right foot (in millimeters)                                         |
| `data.footprintImageUrl`  | String        | URL of the saved plantar pressure image                                          |
| `data.weight`             | Double        | The user's weight (in kilograms)                                                 |

<details markdown=>
  <summary><strong>Example</strong></summary>

## Request

### Postman 요청

아래 버튼을 클릭하면 `Postman`에서 API 요청을 실행할 수 있습니다.

[![Run in Postman](https://run.pstmn.io/button.svg)](https://carenco.postman.co/workspace/Care%26CO~7c4d2551-cc9d-413f-b156-4c350b99eb32/request/27911837-83f0189c-4197-4fa5-85d6-fc0ee4003610?action=share&creator=32584424&ctx=documentation)

```bash
  curl POST 'https://carencoinc.com/kr/measurement-service/v2/footprints/?version='\
--header 'Content-Type: application/json' \
--data '{
    "userId": "469a4b9a-986d-429f-82ba-0795ab91c2d3",
    "measuredDateTime": "2025-02-18T10:15:30",
    "rawData": "<rawData sample>",
    "firstClassType": 1.0,
    "firstAccuracy": 98.5,
    "secondaryClassType": 2.0,
    "secondaryAccuracy": 97.0,
    "thirdClassType": 3.0,
    "thirdAccuracy": 96.5,
    "leftFootLength": 250.0,
    "leftFootWidth": 100.0,
    "rightFootLength": 255.0,
    "rightFootWidth": 105.0,
    "footprintImageUrl": "http://example.com/footprint.jpg",
    "weight": 70.0
}'
```


## Response

<details>
<summary><strong>200 OK</strong></summary>

###### Body

```json
{
  "message": "Plantar pressure measurement complete",
  "data": {
    "id": "",
    "userId": "469a4b9a-986d-429f-82ba-0795ab91c2d3",
    "measuredDateTime": "2025-02-18T10:15:30",
    "firstClassType": 1.0,
    "firstAccuracy": 98.5,
    "secondaryClassType": 2.0,
    "secondaryAccuracy": 97.0,
    "thirdClassType": 3.0,
    "thirdAccuracy": 96.5,
    "leftFootLength": 250.0,
    "leftFootWidth": 100.0,
    "rightFootLength": 255.0,
    "rightFootWidth": 105.0,
    "footprintImageUrl": "http://example.com/footprint.jpg",
    "weight": 70.0
  }
}
```

</details>

<details>
<summary><strong>400 BadRequest</strong></summary>

###### Body

```json
{
  "message": "Invalid request parameters",
  "error": "Error detail message"
}
```
</details>

<details>
<summary><strong>500 InternalServerError</strong></summary>

###### Body

```json
{
  "message": "Error creating footprint record",
  "error": "Error detail message"
}
```

</details>

</details>

---

</details>
<!-- api-1-end -->