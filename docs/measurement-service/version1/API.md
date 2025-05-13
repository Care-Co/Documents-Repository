# Measurement-Service

    - Basic Domain: https://carencoinc.com
    - Basic IP: 15.165.125.100:8084


<!-- api-1-start -->
<details markdown="1">
<summary><strong>&nbsp;API: CreateFootprintRecord</strong></summary>



## Basic Information

| Method | URL                                              |
|--------|--------------------------------------------------|
| POST   | `/kr/measurement-service/v1/footprints/{userId}` |

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

[![Run in Postman](https://run.pstmn.io/button.svg)](https://carenco.postman.co/workspace/Care%26CO~7c4d2551-cc9d-413f-b156-4c350b99eb32/request/27911837-2e1ffee0-55b5-4be5-89d5-a181a8ae21b6?action=share&creator=32584424&ctx=documentation)

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
<summary><strong>&nbsp;API: GetFootprintRecords</strong></summary>


## Basic Information

| Method | URL                                              |
|--------|--------------------------------------------------|
| GET    | `/kr/measurement-service/v1/footprints/{userId}` |

### Request

#### Parameters(@PathVariable)

| Name     | Type   | Description            | Required | Remarks |
|----------|--------|------------------------|----------|---------|
| `userId` | String | User Unique identifier | Yes      |         |

#### Parameters(@RequestParam)

| Name   | Type          | Description                                   | Required | Remarks |
|--------|---------------|-----------------------------------------------|----------|---------|
| `from` | LocalDateTime | Query start date and time                     | No       |         |
| `to`   | LocalDateTime | Query end date and time                       | No       |         |
| `size` | int           | Number of records to retrieve                 | No       |         |
| `page` | int           | Page number of the queried data               | No       |         |
| `sort` | String        | Sorting method (e.g., measuredDateTime, desc) | No       |         |

> ### Additional Query Logic for `GetFootprintRecords`
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

[![Run in Postman](https://run.pstmn.io/button.svg)](https://carenco.postman.co/workspace/Care%26CO~7c4d2551-cc9d-413f-b156-4c350b99eb32/request/27911837-fb32bbe2-8e0d-46e2-ad45-3972a173ac52?action=share&creator=32584424&ctx=documentation)


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

| Method | URL                                          |
|--------|----------------------------------------------|
| POST   | `/kr/measurement-service/v1/footprints/{id}` |

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

[![Run in Postman](https://run.pstmn.io/button.svg)](https://carenco.postman.co/workspace/Care%26CO~7c4d2551-cc9d-413f-b156-4c350b99eb32/request/27911837-8ec36dc1-f0e3-4df9-bb88-7f4b313af967?action=share&creator=32584424&ctx=documentation)

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
<!-- api-3-end -->


<!-- api-4-start -->
<details markdown="1">
<summary><strong>&nbsp;API: GetWeightRecords</strong></summary>


## Basic Information

| Method | URL                                           |
|--------|-----------------------------------------------|
| GET    | `/kr/measurement-service/v1/weights/{userId}` |

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

> ### Additional Query Logic for `GetWeightRecords`
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

| Name                      | Type          | Description                                             |
|---------------------------|---------------|---------------------------------------------------------|
| `data`                    | Object        | Contains data related to the user's weight              |
| `data.id`                 | UUID          | Unique identifier (ID) for the weight record            |
| `data.userId`             | String        | ID of the user who requested the measurement            |
| `data.measuredDateTime`   | LocalDateTime | The date and time when the weight measurement was taken |
| `data.measurementMethod`  | String        | The method used to measure                              |
| `data.weight`             | Double        | The user's weight (in kilograms)                        |


<details markdown=>
  <summary><strong>Example</strong></summary>

## Request

### Postman 요청

아래 버튼을 클릭하면 `Postman`에서 API 요청을 실행할 수 있습니다.

[![Run in Postman](https://run.pstmn.io/button.svg)](https://carenco.postman.co/workspace/Care%26CO~7c4d2551-cc9d-413f-b156-4c350b99eb32/request/27911837-0797420f-ad6a-4cfc-87cf-67ce5162b75d?action=share&creator=32584424&ctx=documentation)


```bash
  curl GET 'https://carencoinc.com/kr/measurement-service/v1/weights/{userId}'
```

## Response

<details>
<summary><strong>200 OK</strong></summary>

###### Body

```json
{
  "message": "The weight records retrieved",
  "data": [
    {
      "id": "",
      "userId": "",
      "measuredDateTime": "YYYY-MM-DDThh:mm:ss",
      "measurementMethod": "",
      "weight": 0.0
    },
    {
      "id": "",
      "userId": "",
      "measuredDateTime": "YYYY-MM-DDThh:mm:ss",
      "measurementMethod": "",
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
  "message": "Error retrieving weight records",
  "error" : "Error detail message"
}
```

</details>

</details>

---

</details>
<!-- api-4-end -->


<!-- api-5-start -->
<details markdown="1">
<summary><strong>&nbsp;API: DeleteWeightRecord</strong></summary>

## Basic Information

| Method | URL                                       |
|--------|-------------------------------------------|
| POST   | `/kr/measurement-service/v1/weights/{id}` |

### Request

#### Parameters(@PathVariable)

| Name | Type | Description                        | Required | Remarks |
|------|------|------------------------------------|----------|---------|
| `id` | UUID | Unique identifier of Weight record | Yes      |         |

### Response

#### Body

| Name                      | Type          | Description                                                       |
|---------------------------|---------------|-------------------------------------------------------------------|
| `message`                 | String        | The result message of the API call (e.g., weights record deleted) |

<details markdown=>
  <summary><strong>Example</strong></summary>

## Request

### Postman 요청

아래 버튼을 클릭하면 `Postman`에서 API 요청을 실행할 수 있습니다.

[![Run in Postman](https://run.pstmn.io/button.svg)](https://carenco.postman.co/workspace/Care%26CO~7c4d2551-cc9d-413f-b156-4c350b99eb32/request/27911837-5b354fca-a850-4e16-bf9b-2ae399372f4d?action=share&creator=32584424&ctx=documentation)

```bash
  curl DELETE 'https://carencoinc.com/kr/measurement-service/v1/weights/{id}
```

## Response

<details>
<summary><strong>200 OK</strong></summary>

###### Body

```json
{
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
  "message": "Error deleting weight record",
  "error": "Error detail message"
}
```

</details>

</details>

---

</details>
<!-- api-5-end -->


<!-- api-6-start -->
<details markdown="1">
<summary><strong>&nbsp;API: CreatePoseRecord</strong></summary>



## Basic Information

| Method | URL                                        |
|--------|--------------------------------------------|
| POST   | `/kr/measurement-service/v1/pose/{userId}` |

### Request

#### Parameters(@PathVariable)

| Name     | Type   | Description            | Required | Remarks |
|----------|--------|------------------------|----------|---------|
| `userId` | String | User Unique identifier | Yes      |         |

#### Parameters(@RequestParam)

| Name        | Type          | Description                 | Required | Remarks |
|-------------|---------------|-----------------------------|----------|---------|
| `timestamp` | LocalDateTime | Measurement date and time   | Yes      |         |
| `file`      | MultipartFile | The file for measuring pose | Yes      |         |

### Response

#### Body

| Name                    | Type          | Description                                                          |
|-------------------------|---------------|----------------------------------------------------------------------|
| `message`               | String        | The result message of the API call (e.g., pose measurement complete) |
| `data`                  | Object        | Contains data related to the user's pose estimation                  |
| `data.id`               | UUID          | Unique identifier (ID) for the pose record                           |
| `data.userId`           | String        | ID of the user who requested the measurement                         |
| `data.measuredDateTime` | LocalDateTime | The date and time when the pose estimation was taken                 |
| `data.angleFace`        | Double        | Represents the angle of the person's face in degrees.                |
| `data.anglePelvis`      | Double        | Represents the angle of the person's pelvis in degrees.              |
| `data.angleShoulder`    | Double        | Represents the angle of the person's shoulder in degrees.            |
| `data.classType`        | Int           | The classType of the pose estimation result                          |
| `data.accuracy`         | Double        | The accuracy of the classType                                        |
| `data.imageUrl`         | String        | URL of the saved pose image                                          |

<details markdown=>
  <summary><strong>Example</strong></summary>

## Request

### Postman 요청

아래 버튼을 클릭하면 `Postman`에서 API 요청을 실행할 수 있습니다.

[![Run in Postman](https://run.pstmn.io/button.svg)](https://carenco.postman.co/workspace/Care%26CO~7c4d2551-cc9d-413f-b156-4c350b99eb32/request/27911837-965fbe4d-6592-400b-bd89-31a24946fa3e?action=share&creator=32584424&ctx=documentation)

```bash
  curl POST 'https://carencoinc.com/kr/measurement-service/v1/pose/{userId}?timestamp='
```

## Response

<details>
<summary><strong>200 OK</strong></summary>

###### Body

```json
{
  "message": "Pose estimation complete",
  "data": {
    "id": "",
    "userId": "",
    "measuredDateTime": "",
    "angleFace": 0.0,
    "anglePelvis": 0.0,
    "angleShoulder": 0.0,
    "classType": 0,
    "accuracy": 0.0,
    "imageUrl": ""
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
  "message": "Error creating pose record",
  "error": "Error detail message"
}
```

</details>

</details>

---

</details>
<!-- api-6-end -->



<!-- api-7-start -->
<details markdown="1">
<summary><strong>&nbsp;API: CreateActivityRecord</strong></summary>



## Basic Information

| Method | URL                                          |
|--------|----------------------------------------------|
| POST   | `/api/v1/measurement/users/{uid}/activities` |

### Request

#### Parameters(@PathVariable)

| Name  | Type | Description            | Required | Remarks |
|-------|------|------------------------|----------|---------|
| `uid` | UUID | User Unique identifier | Yes      |         |

#### Parameters(@RequestBody)

| Name             | Type      | Description                                     | Required | Remarks |
|------------------|-----------|-------------------------------------------------|----------|---------|
| `category`       | String    | Type of activity record                         |          |         |
| `activity_date`  | LocalDate | Date of the activity                            |          |         |
| `start_time`     | LocalTime | Start time of the activity                      |          |         |
| `end_time`       | LocalTime | End time of the activity                        |          |         |
| `pain_decreased` | boolean   | Whether the pain was reduced after the activity | No       |         |
| `same_pain`      | boolean   | Whether the pain remained unchanged             | No       |         |
| `mild_pain`      | boolean   | Whether the pain was mild                       | No       |         |
| `moderate_pain`  | boolean   | Whether the pain was moderate                   | No       |         |
| `severe_pain`    | boolean   | Whether the pain was severe                     | No       |         |


### Response

#### Body

| Name                    | Type      | Description                                                |
|-------------------------|-----------|------------------------------------------------------------|
| `success`               | boolean   | Indicates whether the API call was successful (true/false) |
| `data`                  | Object    | Contains data related to the user's activity record        |
| `data.id`               | UUID      | Unique identifier (ID) for the activity record             |
| `data.uid`              | UUID      | ID of the user who requested                               |
| `data.type`             | String    | Type of activity record                                    |
| `data.activityDate`     | LocalDate | Date of the activity                                       |
| `data.startTime`        | LocalTime | Start time of the activity                                 |
| `data.endTime`          | LocalTime | End time of the activity                                   |
| `data.painLog`          | Object    | Contains data related to the user's activity pain record   |
| `painLog.id`            | UUID      | Unique identifier (ID) for the pain record                 |
| `painLog.painDecreased` | boolean   | Whether the pain was reduced after the activity            | 
| `painLog.samePain`      | boolean   | Whether the pain remained unchanged                        | 
| `painLog.mildPain`      | boolean   | Whether the pain was mild                                  | 
| `painLog.moderatePain`  | boolean   | Whether the pain was moderate                              | 
| `painLog.severePain`    | boolean   | Whether the pain was severe                                | 


<details markdown=>
  <summary><strong>Example</strong></summary>

## Request

### Postman 요청

아래 버튼을 클릭하면 `Postman`에서 API 요청을 실행할 수 있습니다.

[![Run in Postman](https://run.pstmn.io/button.svg)](https://carenco.postman.co/workspace/Care%26CO~7c4d2551-cc9d-413f-b156-4c350b99eb32/example/27911837-f3383c46-0500-401a-b925-3c1dbe08e935?action=share&creator=32584424&ctx=documentation)

```bash
  curl POST 'https://carencoinc.com/api/v1/measurement/users/{uid}/activities'
    --header 'Content-Type: application/json' \
    --data '{
        "category": "",
        "activity_date": "",
        "start_time": "",
        "end_time": "",
        "pain_decreased": "",
        "same_pain": "",
        "mild_pain": "",
        "moderate_pain": "",
        "severe_pain": ""
    }'
```

## Response

<details>
<summary><strong>200 OK</strong></summary>

###### Body

```json
{
  "success": true,
  "data": {
    "id": "dcad5dfa-8b36-4736-84f7-04e68b40a0ac",
    "uid": "469a4b9a-986d-429f-82ba-0795ab91c2d3",
    "type": "ORIENTAL_TREATMENT",
    "activityDate": "2025-05-13",
    "startTime": "13:00:00",
    "endTime": "14:00:00",
    "painLog": {
      "id": "2884133d-bfe1-4dff-a668-a4f475833cde",
      "painDecreased": true,
      "samePain": false,
      "mildPain": false,
      "moderatePain": false,
      "severePain": false
    }
  }
}
```

</details>

<details>
<summary><strong>400 BadRequest</strong></summary>

###### Body

```json
{
    "code": "CHECK_PARAMETER"
}
```
</details>

<details>
<summary><strong>500 InternalServerError</strong></summary>

###### Body

```json
{
  "message": "Error creating activity record",
  "error": "Error detail message"
}
```

</details>

</details>

---

</details>
<!-- api-7-end -->



<!-- api-8-start -->
<details markdown="1">
<summary><strong>&nbsp;API: RetrieveActivityRecords</strong></summary>



## Basic Information

| Method | URL                                          |
|--------|----------------------------------------------|
| GET    | `/api/v1/measurement/users/{uid}/activities` |

### Request

#### Parameters(@PathVariable)

| Name  | Type | Description            | Required | Remarks |
|-------|------|------------------------|----------|---------|
| `uid` | UUID | User Unique identifier | Yes      |         |


#### Parameters(@RequestParam)

| Name      | Type      | Description                            | Required | Remarks |
|-----------|-----------|----------------------------------------|----------|---------|
| `from`    | LocalDate | Query start date                       | No       |         |
| `to`      | LocalDate | Query end date                         | No       |         |
| `size`    | int       | Number of records to retrieve          | No       |         |
| `page`    | int       | Page number of the queried data        | No       |         |
| `sort`    | String    | Sorting method (e.g., startTime, desc) | No       |         |

> ### Additional Query Logic for `GetActivityRecords`
> The `findByUidAndDateRange` method includes logic to filter records based on various conditions. Below are the
> details:
>
> #### Query Conditions
>
> 1. **Both `from` and `to` parameters are provided:**
     >     - Filters records where the measurement timestamp (`activityDate`) is between `fromDate` and
     `toDate` (inclusive).
     >     - Repository method: `findByUidAndActivityDateBetween`.
>
> 2. **Only `from` parameter is provided:**
     >     - Filters records where the measurement timestamp (`activityDate`) is after `fromDate`.
     >     - Repository method: `findByUidAndActivityDateAfter`.
>
> 3. **Only `to` parameter is provided:**
     >     - Filters records where the measurement timestamp (`activityDate`) is before `toDate`.
     >     - Repository method: `findByUidAndActivityDateBefore`.
>
> 4. **Neither `from` nor `to` parameters are provided:**
     >     - Returns all records for the given user, without date filtering.
     >     - Repository method: `findByUid`.
>
> #### Pagination and Sorting
>
> - **Pagination:**
    >     - The `page` and `size` parameters determine the pagination behavior.
    >     - These are passed into the `PageRequest` object to fetch the corresponding page of records.
>
> - **Sorting:**
    >     - The `sort` parameter defines the sorting behavior. It should follow the format: `field,direction`.
    >         - `field`: The name of the field to sort by (e.g., `startTime`).
    >         - `direction`: Sorting direction (`asc` for ascending, `desc` for descending). Defaults to ascending if
    omitted.
    >     - Example values:
    >         - `startTime,desc`: Sort by `startTime` in descending order.
    >         - `weight,asc`: Sort by `weight` in ascending order.
    >
- **Error Handling:**
  >         - If the `sort` parameter is invalid, an `IllegalArgumentException` is thrown with a message explaining the
  expected format.
>
> #### Example Query Scenarios
>
> 1. **Retrieve all records for a user within a specific date range, sorted by timestamp in descending order:**
     >     - Parameters: `from=2025-01-01`, `to=2025-01-31`, `sort=startTime,desc`.
>
> 2. **Retrieve all records after a specific date:**
     >     - Parameters: `from=2025-01-01`, `sort=startTime,asc`.
>
> 3. **Retrieve paginated records without any date filters:**
     >     - Parameters: `page=1`, `size=10`.


### Response

#### Body

| Name                    | Type      | Description                                                |
|-------------------------|-----------|------------------------------------------------------------|
| `success`               | boolean   | Indicates whether the API call was successful (true/false) |
| `data`                  | Object    | Contains data related to the user's activity record        |
| `data.id`               | UUID      | Unique identifier (ID) for the activity record             |
| `data.uid`              | UUID      | ID of the user who requested                               |
| `data.type`             | String    | Type of activity record                                    |
| `data.activityDate`     | LocalDate | Date of the activity                                       |
| `data.startTime`        | LocalTime | Start time of the activity                                 |
| `data.endTime`          | LocalTime | End time of the activity                                   |
| `data.painLog`          | Object    | Contains data related to the user's activity pain record   |
| `painLog.id`            | UUID      | Unique identifier (ID) for the pain record                 |
| `painLog.painDecreased` | boolean   | Whether the pain was reduced after the activity            | 
| `painLog.samePain`      | boolean   | Whether the pain remained unchanged                        | 
| `painLog.mildPain`      | boolean   | Whether the pain was mild                                  | 
| `painLog.moderatePain`  | boolean   | Whether the pain was moderate                              | 
| `painLog.severePain`    | boolean   | Whether the pain was severe                                | 

<details markdown=>
  <summary><strong>Example</strong></summary>

## Request

### Postman 요청

아래 버튼을 클릭하면 `Postman`에서 API 요청을 실행할 수 있습니다.

[![Run in Postman](https://run.pstmn.io/button.svg)](https://carenco.postman.co/workspace/Care%26CO~7c4d2551-cc9d-413f-b156-4c350b99eb32/example/27911837-a11ce1a8-27df-4591-bf9b-decfc4bddd13?action=share&creator=32584424&ctx=documentation)

```bash
  curl GET 'https://carencoinc.com/api/v1/measurement/users/{uid}/activities?from=2025-05-13&to=2025-05-13&size=1&page=0&sort=startTime,asc'
```

## Response

<details>
<summary><strong>200 OK</strong></summary>

###### Body

```json
{
  "success": true,
  "data": [
    {
      "id": "dcad5dfa-8b36-4736-84f7-04e68b40a0ac",
      "uid": "469a4b9a-986d-429f-82ba-0795ab91c2d3",
      "type": "ORIENTAL_TREATMENT",
      "activityDate": "2025-05-13",
      "startTime": "13:00:00",
      "endTime": "14:00:00",
      "painLog": {
        "id": "2884133d-bfe1-4dff-a668-a4f475833cde",
        "painDecreased": true,
        "samePain": false,
        "mildPain": false,
        "moderatePain": false,
        "severePain": false
      }
    },
    {
      "id": "a48ec36c-9caf-4227-b4c0-05118696b4ba",
      "uid": "469a4b9a-986d-429f-82ba-0795ab91c2d3",
      "type": "ORIENTAL_TREATMENT",
      "activityDate": "2025-05-13",
      "startTime": "12:00:00",
      "endTime": "13:00:00",
      "painLog": {
        "id": "f87e05b6-666a-4e4f-8881-1921c7923c42",
        "painDecreased": true,
        "samePain": false,
        "mildPain": false,
        "moderatePain": false,
        "severePain": false
      }
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
    "code": "CHECK_PARAMETER"
}
```
</details>

<details>
<summary><strong>500 InternalServerError</strong></summary>

###### Body

```json
{
  "message": "Error retrieving activity record",
  "error": "Error detail message"
}
```

</details>

</details>

---

</details>
<!-- api-8-end -->



<!-- api-9-start -->
<details markdown="1">
<summary><strong>&nbsp;API: DeleteActivityRecord</strong></summary>



## Basic Information

| Method | URL                                               |
|--------|---------------------------------------------------|
| DELETE | `/api/v1/measurement/users/{uid}/activities/{id}` |

### Request

#### Parameters(@PathVariable)

| Name  | Type | Description                                    | Required | Remarks |
|-------|------|------------------------------------------------|----------|---------|
| `uid` | UUID | User Unique identifier                         | Yes      |         |
| `id`  | UUID | Unique identifier (ID) for the activity record | Yes      |         |

### Response

#### Body

| Name                    | Type      | Description                                                |
|-------------------------|-----------|------------------------------------------------------------|
| `success`               | boolean   | Indicates whether the API call was successful (true/false) |

<details markdown=>
  <summary><strong>Example</strong></summary>

## Request

### Postman 요청

아래 버튼을 클릭하면 `Postman`에서 API 요청을 실행할 수 있습니다.

[![Run in Postman](https://run.pstmn.io/button.svg)](https://carenco.postman.co/workspace/Care%26CO~7c4d2551-cc9d-413f-b156-4c350b99eb32/example/27911837-0f34102c-173b-4a7a-a534-b95b88f86bca?action=share&creator=32584424&ctx=documentation)

```bash
  curl POST 'https://carencoinc.com/api/v1/measurement/users/{uid}/activities/{id}'
```

## Response

<details>
<summary><strong>200 OK</strong></summary>

###### Body

```json
{
  "success": true
}

```

</details>

<details>
<summary><strong>400 BadRequest</strong></summary>

###### Body

```json
{
  "code": "CHECK_PARAMETER"
}
```
</details>

<details>
<summary><strong>500 InternalServerError</strong></summary>

###### Body

```json
{
  "message": "Error deleting activity record",
  "error": "Error detail message"
}
```

</details>

</details>

---

</details>
<!-- api-9-end -->