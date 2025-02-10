# Measurement-Service

    - Basic Domain: https://carencoinc.com/kr/measurement-service
    - Basic IP: 15.165.125.100:8084


<!-- api-1-start -->
<details markdown="1">
<summary><strong>&nbsp;API: CreateFootprintRecord</strong></summary>



## Basic Information

| Method | URL                       |
|--------|---------------------------|
| POST   | `/v1/footprints/{userId}` |

### Request

#### Parameters(@PathVariable)

| Name     | Type   | Description            | Required | Remarks |
|----------|--------|------------------------|----------|---------|
| `userId` | String | User Unique identifier | Yes      |         |

#### Parameters(@RequestParam)

| Name        | Type          | Description                          | Required | Remarks |
|-------------|---------------|--------------------------------------|----------|---------|
| `timestamp` | LocalDateTime | Measurement date and time            | Yes      |         |
| `rawData`   | String        | Measured raw data                    | Yes      |         |

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

[![Run in Postman](https://run.pstmn.io/button.svg)](http://naver.com)

```bash
  curl POST 'https://carencoinc.com/kr/measurement-service/v1/footprints/{userId}?timestamp=&rawData='
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
    "userId": "",
    "measuredDateTime": "YYYY-MM-DDThh:mm:ss",
    "firstClassType": 0.0,
    "firstAccuracy": 0.0,
    "secondaryClassType": 0.0,
    "secondaryAccuracy": 0.0,
    "thirdClassType": 0.0,
    "thirdAccuracy": 0.0,
    "leftFootLength": 0.0,
    "leftFootWidth": 0.0,
    "rightFootLength": 0.0,
    "rightFootWidth": 0.0,
    "footprintImageUrl": "",
    "weight": 0.0
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


<!-- api-2-start -->
<details markdown="1">
<summary><strong>&nbsp;API: getFootprintRecords</strong></summary>


## Basic Information

| Method | URL                       |
|--------|---------------------------|
| GET    | `/v1/footprints/{userId}` |

### Request

#### Parameters(@PathVariable)

| Name     | Type   | Description            | Required | Remarks |
|----------|--------|------------------------|----------|---------|
| `userId` | String | User Unique identifier | Yes      |         |

#### Parameters(@RequestParam)

| Name   | Type          | Description                                  | Required | Remarks |
|--------|---------------|----------------------------------------------|----------|---------|
| `from` | LocalDateTime | Query start date and time                    | No       |         |
| `to`   | LocalDateTime | Query end date and time                      | No       |         |
| `size` | int           | Number of records to retrieve                | No       |         |
| `page` | int           | Page number of the queried data              | No       |         |
| `sort` | String        | Sorting method (e.g., measuredDateTime, desc) | No       |         |

> ### Additional Query Logic for `getFootprintRecords`
> The `findByUserIdAndDateTimeRange` method includes logic to filter records based on various conditions. Below are the details:
>
> #### Query Conditions
>
> 1. **Both `from` and `to` parameters are provided:**
     >     - Filters records where the measurement timestamp (`measuredDateTime`) is between `fromDateTime` and `toDateTime` (inclusive).
>     - Repository method: `findByUserIdAndMeasuredDateTimeBetween`.
>
> 2. **Only `from` parameter is provided:**
     >     - Filters records where the measurement timestamp (`measuredDateTime`) is after `fromDateTime`.
>     - Repository method: `findByUserIdAndMeasuredDateTimeAfter`.
>
> 3. **Only `to` parameter is provided:**
     >     - Filters records where the measurement timestamp (`measuredDateTime`) is before `toDateTime`.
>     - Repository method: `findByUserIdAndMeasuredDateTimeBefore`.
>
> 4. **Neither `from` nor `to` parameters are provided:**
     >     - Returns all records for the given user, without date filtering.
>     - Repository method: `findByUserId`.
>
> #### Pagination and Sorting
>
> - **Pagination:**
    >     - The `page` and `size` parameters determine the pagination behavior.
>     - These are passed into the `PageRequest` object to fetch the corresponding page of records.
>
> - **Sorting:**
    >     - The `sort` parameter defines the sorting behavior. It should follow the format: `field,direction`.
            >         - `field`: The name of the field to sort by (e.g., `measuredDateTime`).
>         - `direction`: Sorting direction (`asc` for ascending, `desc` for descending). Defaults to ascending if omitted.
>     - Example values:
        >         - `measuredDateTime,desc`: Sort by `measuredDateTime` in descending order.
>         - `weight,asc`: Sort by `weight` in ascending order.
>     - **Error Handling:**
        >         - If the `sort` parameter is invalid, an `IllegalArgumentException` is thrown with a message explaining the expected format.
>
> #### Example Query Scenarios
>
> 1. **Retrieve all records for a user within a specific date range, sorted by timestamp in descending order:**
     >     - Parameters: `from=2025-01-01T00:00:00`, `to=2025-01-31T23:59:59`, `sort=measuredDateTime,desc`.
>
> 2. **Retrieve all records after a specific date:**
     >     - Parameters: `from=2025-01-01T00:00:00`, `sort=measuredDateTime,asc`.
>
> 3. **Retrieve paginated records without any date filters:**
     >     - Parameters: `page=1`, `size=10`.
>
> #### Error Handling for Invalid Sorting
>
> - If the `sort` parameter is not in the correct format (e.g., missing field or direction), the following exception will be raised:
    >   ```json
>   {
>     "message": "Invalid sort parameter. Expected format: 'field,direction'.",
>     "error": "Detailed error message explaining the issue"
>   }

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

[![Run in Postman](https://run.pstmn.io/button.svg)](http://naver.com)


```bash
  curl GET 'https://carencoinc.com/kr/measurement-service/v1/footprints/{userId}'
```

## Response

<details>
<summary><strong>200 OK</strong></summary>

###### Body

```json
{
  "message": "Plantar pressure records retrieved",
  "data": [
    {
      "id": "",
      "userId": "",
      "measuredDateTime": "YYYY-MM-DDThh:mm:ss",
      "firstClassType": 0.0,
      "firstAccuracy": 0.0,
      "secondaryClassType": 0.0,
      "secondaryAccuracy": 0.0,
      "thirdClassType": 0.0,
      "thirdAccuracy": 0.0,
      "leftFootLength": 0.0,
      "leftFootWidth": 0.0,
      "rightFootLength": 0.0,
      "rightFootWidth": 0.0,
      "footprintImageUrl": "",
      "weight": 0.0
    },
    {
      "id": "",
      "userId": "",
      "measuredDateTime": "YYYY-MM-DDThh:mm:ss",
      "firstClassType": 0.0,
      "firstAccuracy": 0.0,
      "secondaryClassType": 0.0,
      "secondaryAccuracy": 0.0,
      "thirdClassType": 0.0,
      "thirdAccuracy": 0.0,
      "leftFootLength": 0.0,
      "leftFootWidth": 0.0,
      "rightFootLength": 0.0,
      "rightFootWidth": 0.0,
      "footprintImageUrl": "",
      "weight": 0.0
    }
  ]
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
  "message": "Error retrieving footprint records",
  "error" : "Error detail message"
}
```

</details>

</details>

---

</details>
<!-- api-2-end -->


<!-- api-3-start -->
<details markdown="1">
<summary><strong>&nbsp;API: DeleteFootprintRecord</strong></summary>

## Basic Information

| Method | URL                   |
|--------|-----------------------|
| POST   | `/v1/footprints/{id}` |

### Request

#### Parameters(@PathVariable)

| Name | Type | Description                       | Required | Remarks |
|------|------|-----------------------------------|----------|---------|
| `id` | UUID | FootprintRecord Unique identifier | Yes      |         |

### Response

#### Body

| Name                      | Type          | Description                                                                |
|---------------------------|---------------|----------------------------------------------------------------------------|
| `message`                 | String        | The result message of the API call (e.g., plantar pressure record deleted) |

<details markdown=>
  <summary><strong>Example</strong></summary>

## Request

### Postman 요청

아래 버튼을 클릭하면 `Postman`에서 API 요청을 실행할 수 있습니다.

[![Run in Postman](https://run.pstmn.io/button.svg)](http://naver.com)

```bash
  curl DELETE 'https://carencoinc.com/kr/measurement-service/v1/footprints/{id}
```

## Response

<details>
<summary><strong>200 OK</strong></summary>

###### Body

```json
{
  "message": "Plantar pressure record deleted"
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
  "message": "Error deleting footprint record",
  "error": "Error detail message"
}
```

</details>

</details>

---

</details>
<!-- api-1-end -->