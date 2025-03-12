# Version2 Measurement API Documents
    - BaseUrl: https://carencoinc.com/kr/api/v2/measurement







<!-- api-1-start -->
<details markdown="1">
<summary><strong>&nbsp;API: Retrieve User Records</strong></summary>


## Basic Information

| Method | URL                   |
|--------|-----------------------|
| GET    | `users/{uid}/records` |

### Request

#### Parameters(@PathVariable)

| Name  | Type   | Description            | Required | Remarks |
|-------|--------|------------------------|----------|---------|
| `uid` | String | User Unique identifier | Yes      |         |

#### Parameters(@RequestParam)

| Name        | Type          | Description                                   | Required | Remarks                                                              |
|-------------|---------------|-----------------------------------------------|----------|----------------------------------------------------------------------| 
| `version`   | String        | API version information (format: YYYY-MM-DD)  | No       | If not provided, the latest API version will be used automatically.  |
| `from`      | LocalDateTime | Query start date and time                     | No       |                                                                      |
| `to`        | LocalDateTime | Query end date and time                       | No       |                                                                      |
| `size`      | int           | Number of records to retrieve                 | No       |                                                                      |
| `page`      | int           | Page number of the queried data               | No       |                                                                      |
| `sort`      | String        | Sorting method (e.g., measuredDateTime, desc) | No       |                                                                      |

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

| Name                           | Type          | Description                                                               |
|--------------------------------|---------------|---------------------------------------------------------------------------|
| `data`                         | Object        | Contains data related to the user's measurements records                  |
| `data.userId`                  | String        | ID of the user who requested the measurement                              |
| `data.measuredDateTime`        | LocalDateTime | The date and time when the plantar pressure measurement was taken         |
| `data.footprint`               | Object        | Contains data related to the user's plantar pressure                      |
| `footprint.firstClassType`     | Double        | The first-ranked plantar pressure class type                              |
| `footprint.firstAccuracy`      | Double        | The accuracy (similarity) of the first-ranked class type                  |
| `footprint.secondaryClassType` | Double        | The second-ranked plantar pressure class type                             |
| `footprint.secondaryAccuracy`  | Double        | The accuracy (similarity) of the second-ranked class type                 |
| `footprint.thirdClassType`     | Double        | The third-ranked plantar pressure class type                              |
| `footprint.thirdAccuracy`      | Double        | The accuracy (similarity) of the third-ranked class type                  |
| `footprint.leftFootLength`     | Double        | Length of the left foot (in millimeters)                                  |
| `footprint.leftFootWidth`      | Double        | Width of the left foot (in millimeters)                                   |
| `footprint.rightFootLength`    | Double        | Length of the right foot (in millimeters)                                 |
| `footprint.rightFootWidth`     | Double        | Width of the right foot (in millimeters)                                  |
| `footprint.footprintImageUrl`  | String        | URL of the saved plantar pressure image                                   |
| `footprint.weight`             | Double        | The user's weight (in kilograms)                                          |
| `data.front`                   | Object        | Contains data related to the user's pose estimation from a frontal photo. |
| `front.angleFace`              | Double        | The degree of tilt of the face relative to the horizontal line            |
| `front.angleShoulder`          | Double        | The degree of tilt of the shoulder relative to the horizontal line        |
| `front.anglePelvis`            | Double        | The degree of tilt of the pelvis relative to the horizontal line          |
| `front.imageUrl`               | Double        | URL of the saved user's posture image                                     |
| `data.side`                    | Object        | Contains data related to the user's pose estimation from a side photo.    |
| `side.angleFace`               | Double        | The degree of tilt of the face relative to the horizontal line            |
| `side.angleShoulder`           | Double        | The degree of tilt of the shoulder relative to the horizontal line        |
| `side.anglePelvis`             | Double        | The degree of tilt of the pelvis relative to the horizontal line          |
| `side.imageUrl`                | Double        | URL of the saved user's posture image                                     |

#### Keypoints

> **Keypoints represent the body parts tracked in `data.front` and `data.side`**

| Type     |
|----------|
| `Object` |

| Index | Body part       | Index | Body part    |
|-------|-----------------|-------|--------------|
| `0`   | `nose`          | `9`   | `leftWrist`  | 
| `1`   | `leftEye`       | `10`  | `rightWrist` | 
| `2`   | `rightEye`      | `11`  | `leftHip`    | 
| `3`   | `leftEar`       | `12`  | `rightHip`   | 
| `4`   | `rightEar`      | `13`  | `leftKnee`   | 
| `5`   | `leftShoulder`  | `14`  | `rightKnee`  | 
| `6`   | `rightShoulder` | `15`  | `leftAnkle`  | 
| `7`   | `leftElbow`     | `16`  | `rightAnkle` | 
| `8`   | `rightElbow`    |

> **Each keypoint consists of the following data:**

| Name       | Type   | Description                          |
|------------|--------|--------------------------------------|
| `x`        | Double | X-coordinate of the point            |
| `y`        | Double | Y-coordinate of the point            |
| `accuracy` | Double | Confidence score (0–100, percentage) |




<details markdown=>
  <summary><strong>Example</strong></summary>

## Request

```bash
  curl GET 'https://carencoinc.com/kr/api/v2/measurement/users/{uid}/records'
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

}
```

</details>

<details>
<summary><strong>500 InternalServerError</strong></summary>

###### Body

```json
{
  "success": false,
  "message": "External service error",
  "error": "EXTERNAL_SERVICE_ISSUE"
}
```

</details>

</details>

---

</details>
<!-- api-1-end -->




<!-- api-2-start -->
<details markdown="1">
<summary><strong>&nbsp;API: Create User Records</strong></summary>


## Basic Information

| Method | URL                   |
|--------|-----------------------|
| POST   | `users/{uid}/records` |

### Request

#### Parameters(@PathVariable)

| Name  | Type   | Description            | Required | Remarks |
|-------|--------|------------------------|----------|---------|
| `uid` | String | User Unique identifier | Yes      |         |

#### Parameters(@RequestParam)

| Name        | Type          | Description                                   | Required | Remarks                                                              |
|-------------|---------------|-----------------------------------------------|----------|----------------------------------------------------------------------| 
| `version`   | String        | API version information (format: YYYY-MM-DD)  | No       | If not provided, the latest API version will be used automatically.  |


#### Body(@RequestPart)

| Name               | Type           | Description                                              | Required | Remarks |
|--------------------|----------------|----------------------------------------------------------|----------|---------|
| `measuredDateTime` | LocalDateTime  | The date and time when the plantar measurement was taken |          |         |
| `rawData`          | String         |                                                          |          |         |
| `gender`           | String         |                                                          |          |         |
| `front`            | MultipartFile  |                                                          |          |         |
| `side`             | MultipartFile  |                                                          |          |         |

### Response

#### Body

| Name                           | Type          | Description                                                               |
|--------------------------------|---------------|---------------------------------------------------------------------------|
| `data`                         | Object        | Contains data related to the user's measurements records                  |
| `data.userId`                  | String        | ID of the user who requested the measurement                              |
| `data.measuredDateTime`        | LocalDateTime | The date and time when the plantar pressure measurement was taken         |
| `data.footprint`               | Object        | Contains data related to the user's plantar pressure                      |
| `footprint.firstClassType`     | Double        | The first-ranked plantar pressure class type                              |
| `footprint.firstAccuracy`      | Double        | The accuracy (similarity) of the first-ranked class type                  |
| `footprint.secondaryClassType` | Double        | The second-ranked plantar pressure class type                             |
| `footprint.secondaryAccuracy`  | Double        | The accuracy (similarity) of the second-ranked class type                 |
| `footprint.thirdClassType`     | Double        | The third-ranked plantar pressure class type                              |
| `footprint.thirdAccuracy`      | Double        | The accuracy (similarity) of the third-ranked class type                  |
| `footprint.leftFootLength`     | Double        | Length of the left foot (in millimeters)                                  |
| `footprint.leftFootWidth`      | Double        | Width of the left foot (in millimeters)                                   |
| `footprint.rightFootLength`    | Double        | Length of the right foot (in millimeters)                                 |
| `footprint.rightFootWidth`     | Double        | Width of the right foot (in millimeters)                                  |
| `footprint.footprintImageUrl`  | String        | URL of the saved plantar pressure image                                   |
| `footprint.weight`             | Double        | The user's weight (in kilograms)                                          |
| `data.front`                   | Object        | Contains data related to the user's pose estimation from a frontal photo. |
| `front.angleFace`              | Double        | The degree of tilt of the face relative to the horizontal line            |
| `front.angleShoulder`          | Double        | The degree of tilt of the shoulder relative to the horizontal line        |
| `front.anglePelvis`            | Double        | The degree of tilt of the pelvis relative to the horizontal line          |
| `front.imageUrl`               | Double        | URL of the saved user's posture image                                     |
| `data.side`                    | Object        | Contains data related to the user's pose estimation from a side photo.    |
| `side.angleFace`               | Double        | The degree of tilt of the face relative to the horizontal line            |
| `side.angleShoulder`           | Double        | The degree of tilt of the shoulder relative to the horizontal line        |
| `side.anglePelvis`             | Double        | The degree of tilt of the pelvis relative to the horizontal line          |
| `side.imageUrl`                | Double        | URL of the saved user's posture image                                     |

#### Keypoints

> **Keypoints represent the body parts tracked in `data.front` and `data.side`**

| Index | Body part       | Index | Body part    |
|-------|-----------------|-------|--------------|
| `0`   | `nose`          | `9`   | `leftWrist`  | 
| `1`   | `leftEye`       | `10`  | `rightWrist` | 
| `2`   | `rightEye`      | `11`  | `leftHip`    | 
| `3`   | `leftEar`       | `12`  | `rightHip`   | 
| `4`   | `rightEar`      | `13`  | `leftKnee`   | 
| `5`   | `leftShoulder`  | `14`  | `rightKnee`  | 
| `6`   | `rightShoulder` | `15`  | `leftAnkle`  | 
| `7`   | `leftElbow`     | `16`  | `rightAnkle` | 
| `8`   | `rightElbow`    |

> **Each keypoint consists of the following data:**

| Field     | Description                          |
|-----------|--------------------------------------|
| `x`       | X-coordinate of the point            |
| `y`       | Y-coordinate of the point            |
| `accuracy`| Confidence score (0–100, percentage) |




<details markdown=>
  <summary><strong>Example</strong></summary>

## Request

```bash
  curl GET 'https://carencoinc.com/kr/api/v2/measurement/users/{uid}/records'
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

}
```

</details>

<details>
<summary><strong>500 InternalServerError</strong></summary>

###### Body

```json
{
  "success": false,
  "message": "External service error",
  "error": "EXTERNAL_SERVICE_ISSUE"
}
```

</details>

</details>

---

</details>
<!-- api-2-end -->





<!-- api-3-start -->
<details markdown="1">
<summary><strong>&nbsp;API: Delete User Records</strong></summary>


## Basic Information

| Method | URL                   |
|--------|-----------------------|
| DELETE | `users/{uid}/records` |

### Request

#### Parameters(@PathVariable)

| Name  | Type   | Description            | Required | Remarks |
|-------|--------|------------------------|----------|---------|
| `uid` | String | User Unique identifier | Yes      |         |

#### Parameters(@RequestParam)

| Name               | Type          | Description                                                | Required | Remarks                                                             |
|--------------------|---------------|------------------------------------------------------------|----------|---------------------------------------------------------------------| 
| `version`          | String        | API version information (format: YYYY-MM-DD)               | No       | If not provided, the latest API version will be used automatically. |
| `measuredDateTime` | LocalDateTime | The date and time when the plantar measurement was taken   |          |                                                                     |

### Response




<details markdown=>
  <summary><strong>Example</strong></summary>

## Request

```bash
  curl GET 'https://carencoinc.com/kr/api/v2/measurement/users/{uid}/records'
```

## Response

#### Body

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

}
```

</details>

<details>
<summary><strong>500 InternalServerError</strong></summary>

###### Body

```json
{
  "success": false,
  "message": "External service error",
  "error": "EXTERNAL_SERVICE_ISSUE"
}
```

</details>

</details>

---

</details>
<!-- api-3-end -->








<!-- api-4-start -->
<details markdown="1">
<summary><strong>&nbsp;API: Delete User Records</strong></summary>


## Basic Information

| Method | URL                   |
|--------|-----------------------|
| DELETE | `users/{uid}/records` |

### Request

#### Parameters(@PathVariable)

| Name  | Type   | Description            | Required | Remarks |
|-------|--------|------------------------|----------|---------|
| `uid` | String | User Unique identifier | Yes      |         |

#### Parameters(@RequestParam)

| Name        | Type          | Description                                   | Required | Remarks                                                              |
|-------------|---------------|-----------------------------------------------|----------|----------------------------------------------------------------------| 
| `version`   | String        | API version information (format: YYYY-MM-DD)  | No       | If not provided, the latest API version will be used automatically.  |
| `from`      | LocalDateTime | Query start date and time                     | No       |                                                                      |
| `to`        | LocalDateTime | Query end date and time                       | No       |                                                                      |
| `size`      | int           | Number of records to retrieve                 | No       |                                                                      |
| `page`      | int           | Page number of the queried data               | No       |                                                                      |
| `sort`      | String        | Sorting method (e.g., measuredDateTime, desc) | No       |                                                                      |

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



<details markdown=>
  <summary><strong>Example</strong></summary>

## Request

```bash
  curl DELETE 'https://carencoinc.com/kr/api/v2/measurement/users/{uid}/records'
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

}
```

</details>

<details>
<summary><strong>500 InternalServerError</strong></summary>

###### Body

```json
{
  "success": false,
  "message": "External service error",
  "error": "EXTERNAL_SERVICE_ISSUE"
}
```

</details>

</details>

---

</details>
<!-- api-4-end -->





















































<!-- Unused
<details markdown="1">
<summary><strong>&nbsp;[Unused] Previous Api Call Method</strong></summary>



# Version2 Measurement API Documents

- BaseUrl: https://carencoinc.com/api/v2/measurement

## GetFootprints

### Endpoint

| Method | URL                          |
|--------|------------------------------|
| POST   | `/users/{userId}/footprints` |

### Request

#### Parameters(@RequestParam)

| Name      | Type          | Description                                   | Required | Remarks                                                             |
|-----------|---------------|-----------------------------------------------|----------|---------------------------------------------------------------------|
| `version` | String        | API version information (format: YYYY-MM-DD)  | No       | If not provided, the latest API version will be used automatically. |
| `from`    | LocalDateTime | Query start date and time                     | No       |                                                                     |
| `to`      | LocalDateTime | Query end date and time                       | No       |                                                                     |
| `size`    | int           | Number of records to retrieve                 | No       |                                                                     |
| `page`    | int           | Page number of the queried data               | No       |                                                                     |
| `sort`    | String        | Sorting method (e.g., measuredDateTime, desc) | No       |                                                                     |

> ### Additional Query Logic for `GetFootprintRecords`
> The `findByUserIdAndDateTimeRange` method includes logic to filter records based on various conditions. Below are the
> details:
>
> #### Query Conditions
>
> 1. **Both `from` and `to` parameters are provided:**
     >     - Filters records where the measurement timestamp (`measuredDateTime`) is between `fromDateTime` and
     `toDateTime` (inclusive).
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
    >         - `direction`: Sorting direction (`asc` for ascending, `desc` for descending). Defaults to ascending if
    omitted.
    >     - Example values:
    >         - `measuredDateTime,desc`: Sort by `measuredDateTime` in descending order.
    >         - `weight,asc`: Sort by `weight` in ascending order.
    >
- **Error Handling:**
  >         - If the `sort` parameter is invalid, an `IllegalArgumentException` is thrown with a message explaining the
  expected format.
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

### Response

#### Body

| Name                      | Type          | Description                                                       |
|---------------------------|---------------|-------------------------------------------------------------------|
| `success`                 | boolean       | Indicates whether the API call was successful (true/false)        |
| `data`                    | Object        | Contains data related to the user's plantar pressure              |
| `data.id`                 | UUID          | Unique identifier (ID) for the plantar pressure record            |
| `data.userId`             | String        | ID of the user who requested the measurement                      |
| `data.measuredDateTime`   | LocalDateTime | The date and time when the plantar pressure measurement was taken |
| `data.firstClassType`     | Double        | The first-ranked plantar pressure class type                      |
| `data.firstAccuracy`      | Double        | The accuracy (similarity) of the first-ranked class type          |
| `data.secondaryClassType` | Double        | The second-ranked plantar pressure class type                     |
| `data.secondaryAccuracy`  | Double        | The accuracy (similarity) of the second-ranked class type         |
| `data.thirdClassType`     | Double        | The third-ranked plantar pressure class type                      |
| `data.thirdAccuracy`      | Double        | The accuracy (similarity) of the third-ranked class type          |
| `data.leftFootLength`     | Double        | Length of the left foot (in millimeters)                          |
| `data.leftFootWidth`      | Double        | Width of the left foot (in millimeters)                           |
| `data.rightFootLength`    | Double        | Length of the right foot (in millimeters)                         |
| `data.rightFootWidth`     | Double        | Width of the right foot (in millimeters)                          |
| `data.footprintImageUrl`  | String        | URL of the saved plantar pressure image                           |
| `data.weight`             | Double        | The user's weight (in kilograms)                                  |

---

</details>

-->