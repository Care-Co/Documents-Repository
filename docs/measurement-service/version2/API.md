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
| `success`                      | Boolean       | Whether the API request was successful                                    |
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
| `data.score`                   | Object        | Contains data related to the user's predicted age and body score          |
| `score.bodyScore`              | Double        | The body score value predicted by the system.                             |
| `score.predictedAge`           | Double        | The age estimated predicted by the system.                                |

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
  "success": true,
  "data": [
    {
      "userId": "71ce665d-10c9-4fed-bd70-d53f547bf917",
      "measuredDateTime": "2025-03-11T15:19:36",
      "footprint": {
        "firstClassType": 6.0,
        "firstAccuracy": 68.08,
        "secondaryClassType": 4.0,
        "secondaryAccuracy": 22.34,
        "thirdClassType": 0.0,
        "thirdAccuracy": 3.64,
        "leftFootLength": 206.89,
        "leftFootWidth": 54.54,
        "rightFootLength": 217.24,
        "rightFootWidth": 47.72,
        "footprintImageUrl": "https://kr-dev-milestone4.s3.ap-northeast-2.amazonaws.com/foot_print/71ce665d-10c9-4fed-bd70-d53f547bf917/2025-03-11_16-32-15-fData.png",
        "weight": 104.7
      },
      "front": {
        "angleFace": -0.9362807054730311,
        "anglePelvis": -1.4559726151714067,
        "angleShoulder": -1.017479746544972,
        "imageUrl": "https://kr-dev-milestone4.s3.ap-northeast-2.amazonaws.com/pose_estimation/71ce665d-10c9-4fed-bd70-d53f547bf917/71ce665d-10c9-4fed-bd70-d53f547bf917_2025-03-11T15%3A19%3A36_pData_front.jpg",
        "keypoint": {
          "nose": {
            "x": 129.0,
            "y": 63.0,
            "accuracy": 84.64
          },
          "leftEye": {
            "x": 140.0,
            "y": 52.0,
            "accuracy": 65.27
          },
          "rightEye": {
            "x": 118.0,
            "y": 53.0,
            "accuracy": 73.37
          },
          "leftEar": {
            "x": 154.0,
            "y": 61.0,
            "accuracy": 79.54
          },
          "rightEar": {
            "x": 105.0,
            "y": 63.0,
            "accuracy": 87.72
          },
          "leftShoulder": {
            "x": 183.0,
            "y": 135.0,
            "accuracy": 79.22
          },
          "rightShoulder": {
            "x": 78.0,
            "y": 137.0,
            "accuracy": 84.71
          },
          "leftElbow": {
            "x": 196.0,
            "y": 222.0,
            "accuracy": 87.93
          },
          "rightElbow": {
            "x": 62.0,
            "y": 222.0,
            "accuracy": 92.12
          },
          "leftWrist": {
            "x": 213.0,
            "y": 298.0,
            "accuracy": 82.98
          },
          "rightWrist": {
            "x": 39.0,
            "y": 292.0,
            "accuracy": 91.92
          },
          "leftHip": {
            "x": 160.0,
            "y": 275.0,
            "accuracy": 92.28
          },
          "rightHip": {
            "x": 95.0,
            "y": 276.0,
            "accuracy": 88.58
          },
          "leftKnee": {
            "x": 155.0,
            "y": 407.0,
            "accuracy": 86.94
          },
          "rightKnee": {
            "x": 100.0,
            "y": 407.0,
            "accuracy": 83.4
          },
          "leftAnkle": {
            "x": 149.0,
            "y": 523.0,
            "accuracy": 86.33
          },
          "rightAnkle": {
            "x": 104.0,
            "y": 528.0,
            "accuracy": 68.22
          }
        }
      },
      "side": {
        "angleFace": 15.361357539734005,
        "anglePelvis": 43.19802237802188,
        "angleShoulder": -15.752062245024387,
        "imageUrl": "https://kr-dev-milestone4.s3.ap-northeast-2.amazonaws.com/pose_estimation/71ce665d-10c9-4fed-bd70-d53f547bf917/71ce665d-10c9-4fed-bd70-d53f547bf917_2025-03-11T15%3A19%3A36_pData_side.jpg",
        "keypoint": {
          "nose": {
            "x": 95.0,
            "y": 70.0,
            "accuracy": 61.62
          },
          "leftEye": {
            "x": 104.0,
            "y": 58.0,
            "accuracy": 79.07
          },
          "rightEye": {
            "x": 102.0,
            "y": 57.0,
            "accuracy": 82.76
          },
          "leftEar": {
            "x": 136.0,
            "y": 66.0,
            "accuracy": 81.36
          },
          "rightEar": {
            "x": 132.0,
            "y": 65.0,
            "accuracy": 64.71
          },
          "leftShoulder": {
            "x": 136.0,
            "y": 129.0,
            "accuracy": 78.17
          },
          "rightShoulder": {
            "x": 129.0,
            "y": 131.0,
            "accuracy": 83.6
          },
          "leftElbow": {
            "x": 129.0,
            "y": 227.0,
            "accuracy": 62.1
          },
          "rightElbow": {
            "x": 127.0,
            "y": 228.0,
            "accuracy": 61.86
          },
          "leftWrist": {
            "x": 117.0,
            "y": 308.0,
            "accuracy": 59.52
          },
          "rightWrist": {
            "x": 116.0,
            "y": 305.0,
            "accuracy": 65.47
          },
          "leftHip": {
            "x": 127.0,
            "y": 283.0,
            "accuracy": 73.1
          },
          "rightHip": {
            "x": 124.0,
            "y": 280.0,
            "accuracy": 61.87
          },
          "leftKnee": {
            "x": 131.0,
            "y": 411.0,
            "accuracy": 67.67
          },
          "rightKnee": {
            "x": 130.0,
            "y": 411.0,
            "accuracy": 59.84
          },
          "leftAnkle": {
            "x": 136.0,
            "y": 532.0,
            "accuracy": 56.63
          },
          "rightAnkle": {
            "x": 137.0,
            "y": 527.0,
            "accuracy": 79.65
          }
        }
      },
      "score": {
        "body_score": 87,
        "predicted_age": 32
      }
    }
  ]
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

| Name               | Type          | Description                                              | Required | Remarks |
|--------------------|---------------|----------------------------------------------------------|----------|---------|
| `measuredDateTime` | LocalDateTime | The date and time when the plantar measurement was taken |          |         |
| `rawData`          | String        | Measured raw data                                        |          |         |
| `gender`           | String        | The gender of the user                                   |          |         |
| `height`           | Double        | The height of the user                                   |          |         | 
| `age`              | Int           | The age of the user                                      |          |         |
| `front`            | MultipartFile | The file for measuring pose                              |          |         |
| `side`             | MultipartFile | The file for measuring pose                              |          |         |



### Response

#### Body

| Name                           | Type          | Description                                                               |
|--------------------------------|---------------|---------------------------------------------------------------------------|
| `success`                      | Boolean       | Whether the API request was successful                                    |
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
| `data.score`                   | Object        | Contains data related to the user's predicted age and body score          |
| `score.bodyScore`              | Double        | The body score value predicted by the system.                             |
| `score.predictedAge`           | Double        | The age estimated predicted by the system.                                |

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
  curl POST 'https://carencoinc.com/kr/api/v2/measurement/users/{uid}/records' \
    --form 'measuredDateTime=""' \
    --form 'rawData=""' \
    --form 'gender=""' \
    --form 'front="sample_front.jpg"' \
    --form 'side="sample_side.jpg"'
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
      "userId": "71ce665d-10c9-4fed-bd70-d53f547bf917",
      "measuredDateTime": "2025-03-11T15:19:36",
      "footprint": {
        "firstClassType": 6.0,
        "firstAccuracy": 68.08,
        "secondaryClassType": 4.0,
        "secondaryAccuracy": 22.34,
        "thirdClassType": 0.0,
        "thirdAccuracy": 3.64,
        "leftFootLength": 206.89,
        "leftFootWidth": 54.54,
        "rightFootLength": 217.24,
        "rightFootWidth": 47.72,
        "footprintImageUrl": "https://kr-dev-milestone4.s3.ap-northeast-2.amazonaws.com/foot_print/71ce665d-10c9-4fed-bd70-d53f547bf917/2025-03-11_16-32-15-fData.png",
        "weight": 104.7
      },
      "front": {
        "angleFace": -0.9362807054730311,
        "anglePelvis": -1.4559726151714067,
        "angleShoulder": -1.017479746544972,
        "imageUrl": "https://kr-dev-milestone4.s3.ap-northeast-2.amazonaws.com/pose_estimation/71ce665d-10c9-4fed-bd70-d53f547bf917/71ce665d-10c9-4fed-bd70-d53f547bf917_2025-03-11T15%3A19%3A36_pData_front.jpg",
        "keypoint": {
          "nose": {
            "x": 129.0,
            "y": 63.0,
            "accuracy": 84.64
          },
          "leftEye": {
            "x": 140.0,
            "y": 52.0,
            "accuracy": 65.27
          },
          "rightEye": {
            "x": 118.0,
            "y": 53.0,
            "accuracy": 73.37
          },
          "leftEar": {
            "x": 154.0,
            "y": 61.0,
            "accuracy": 79.54
          },
          "rightEar": {
            "x": 105.0,
            "y": 63.0,
            "accuracy": 87.72
          },
          "leftShoulder": {
            "x": 183.0,
            "y": 135.0,
            "accuracy": 79.22
          },
          "rightShoulder": {
            "x": 78.0,
            "y": 137.0,
            "accuracy": 84.71
          },
          "leftElbow": {
            "x": 196.0,
            "y": 222.0,
            "accuracy": 87.93
          },
          "rightElbow": {
            "x": 62.0,
            "y": 222.0,
            "accuracy": 92.12
          },
          "leftWrist": {
            "x": 213.0,
            "y": 298.0,
            "accuracy": 82.98
          },
          "rightWrist": {
            "x": 39.0,
            "y": 292.0,
            "accuracy": 91.92
          },
          "leftHip": {
            "x": 160.0,
            "y": 275.0,
            "accuracy": 92.28
          },
          "rightHip": {
            "x": 95.0,
            "y": 276.0,
            "accuracy": 88.58
          },
          "leftKnee": {
            "x": 155.0,
            "y": 407.0,
            "accuracy": 86.94
          },
          "rightKnee": {
            "x": 100.0,
            "y": 407.0,
            "accuracy": 83.4
          },
          "leftAnkle": {
            "x": 149.0,
            "y": 523.0,
            "accuracy": 86.33
          },
          "rightAnkle": {
            "x": 104.0,
            "y": 528.0,
            "accuracy": 68.22
          }
        }
      },
      "side": {
        "angleFace": 15.361357539734005,
        "anglePelvis": 43.19802237802188,
        "angleShoulder": -15.752062245024387,
        "imageUrl": "https://kr-dev-milestone4.s3.ap-northeast-2.amazonaws.com/pose_estimation/71ce665d-10c9-4fed-bd70-d53f547bf917/71ce665d-10c9-4fed-bd70-d53f547bf917_2025-03-11T15%3A19%3A36_pData_side.jpg",
        "keypoint": {
          "nose": {
            "x": 95.0,
            "y": 70.0,
            "accuracy": 61.62
          },
          "leftEye": {
            "x": 104.0,
            "y": 58.0,
            "accuracy": 79.07
          },
          "rightEye": {
            "x": 102.0,
            "y": 57.0,
            "accuracy": 82.76
          },
          "leftEar": {
            "x": 136.0,
            "y": 66.0,
            "accuracy": 81.36
          },
          "rightEar": {
            "x": 132.0,
            "y": 65.0,
            "accuracy": 64.71
          },
          "leftShoulder": {
            "x": 136.0,
            "y": 129.0,
            "accuracy": 78.17
          },
          "rightShoulder": {
            "x": 129.0,
            "y": 131.0,
            "accuracy": 83.6
          },
          "leftElbow": {
            "x": 129.0,
            "y": 227.0,
            "accuracy": 62.1
          },
          "rightElbow": {
            "x": 127.0,
            "y": 228.0,
            "accuracy": 61.86
          },
          "leftWrist": {
            "x": 117.0,
            "y": 308.0,
            "accuracy": 59.52
          },
          "rightWrist": {
            "x": 116.0,
            "y": 305.0,
            "accuracy": 65.47
          },
          "leftHip": {
            "x": 127.0,
            "y": 283.0,
            "accuracy": 73.1
          },
          "rightHip": {
            "x": 124.0,
            "y": 280.0,
            "accuracy": 61.87
          },
          "leftKnee": {
            "x": 131.0,
            "y": 411.0,
            "accuracy": 67.67
          },
          "rightKnee": {
            "x": 130.0,
            "y": 411.0,
            "accuracy": 59.84
          },
          "leftAnkle": {
            "x": 136.0,
            "y": 532.0,
            "accuracy": 56.63
          },
          "rightAnkle": {
            "x": 137.0,
            "y": 527.0,
            "accuracy": 79.65
          }
        }
      },
      "score": {
        "body_score": 87,
        "predicted_age": 32
      }
    }
  ]
}
```

</details>

<details>
<summary><strong>409 Conflict</strong></summary>
###### Body

```json
{
  "success": false,
  "message": "Duplicate request or existing data",
  "error": "DUPLICATE_REQUEST"
}
```

</details>

<details>
<summary><strong>500 InternalServerError</strong></summary>

###### Body

```json
{
  "success": false,
  "message": "Unknown server error",
  "error": "INTERNAL_SERVER_ERROR"
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
  curl DELETE 'https://carencoinc.com/kr/api/v2/measurement/users/{uid}/records'
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
  
}
```

</details>

</details>

---

</details>
<!-- api-3-end -->








<!-- api-4-start -->
<details markdown="1">
<summary><strong>&nbsp;API: Delete User Weight Records</strong></summary>


## Basic Information

| Method | URL                   |
|--------|-----------------------|
| DELETE | `users/{uid}/weights` |

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
  curl DELETE 'https://carencoinc.com/kr/api/v2/measurement/users/{uid}/weights'
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