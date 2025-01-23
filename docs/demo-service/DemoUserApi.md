###


# `btobController` - Create User Data

## Basic Information

| Method | URL      |
|--------|----------|
| POST   | `/users` |

## Request

### Parameters(@RequestBody)

| Name       | Type   | Description                               | Required |
|------------|--------|-------------------------------------------|----------|
| `gender`   | String | 성별 (대소문자 구별 없이 `male`, `female`, `other`) | Yes      |
| `birthday` | String | 생년 (4자리 년도, 예: `1990`)                    | Yes      |
| `height`   | Double | 키 (소수점을 포함한 실수 값)                         | Yes      |

## Response

### Body

| Name                   | Type              | Description                 |
|------------------------|-------------------|-----------------------------|
| `uid`                  | UUID              | 사용자 고유 식별자                  |
| `firstName`            | String (Nullable) | 사용자 이름 (First Name)         |
| `lastName`             | String (Nullable) | 사용자 성 (Last Name)           |
| `email`                | String            | 사용자 이메일 주소                  |
| `phoneNumber`          | String (Nullable) | 사용자 전화번호                    |
| `gender`               | String            | 성별 (`MALE`, `FEMALE`, 기타 값) |
| `birthday`             | String (Date)     | 사용자 생년월일 (`YYYY-MM-DD` 형식)  |
| `height`               | Double            | 사용자 키 (cm 단위)               |
| `weight`               | Double (Nullable) | 사용자 몸무게                     |
| `address`              | String (Nullable) | 사용자 주소                      |
| `detailedAddress`      | String (Nullable) | 상세 주소                       |
| `underlyingConditions` | String (Nullable) | 기저질환 정보                     |
| `occupation`           | String (Nullable) | 사용자 직업                      |
| `userRecords`          | List              | 사용자 기록 목록                   |

### Example

#### Request

```bash
curl -X 'POST' \
'http://15.165.125.100:8085/demo/v1/users' \
--header 'Content-Type: application/json' \
--data '{
  "gender": "String(대소문자 구별없이 male, female, other)",
  "birthday": "String(4글자 년도)",
  "height": "Double"
}'

```

#### Response

##### 201 Created

```json
  {
  "uid": "b24aed12-3716-494c-8e20-94c7f429bd71",
  "firstName": null,
  "lastName": null,
  "email": null,
  "phoneNumber": null,
  "gender": "MALE",
  "birthday": "1995-01-01",
  "height": 180.0,
  "weight": null,
  "address": null,
  "detailedAddress": null,
  "underlyingConditions": null,
  "occupation": null,
  "userRecords": []
}
```

##### 400 BadRequest

```json
[
  "gender: Invalid gender. birthday: Birthday must be a 4-digit year.."
]
```

---

# `btobController` - Retrieve User Data

## Basic Information

| Method | URL               |
|--------|-------------------|
| GET    | `/users/{userId}` |

## Request

### Parameters(@PathVariable)

| Name     | Type | Description | Required |
|----------|------|-------------|----------|
| `userId` | UUID | User 고유 식별자 | Yes      |

## Response

### Body

| Name                   | Type              | Description                 |
|------------------------|-------------------|-----------------------------|
| `uid`                  | UUID              | 사용자 고유 식별자                  |
| `firstName`            | String (Nullable) | 사용자 이름 (First Name)         |
| `lastName`             | String (Nullable) | 사용자 성 (Last Name)           |
| `email`                | String            | 사용자 이메일 주소                  |
| `phoneNumber`          | String (Nullable) | 사용자 전화번호                    |
| `gender`               | String            | 성별 (`MALE`, `FEMALE`, 기타 값) |
| `birthday`             | String (Date)     | 사용자 생년월일 (`YYYY-MM-DD` 형식)  |
| `height`               | Double            | 사용자 키 (cm 단위)               |
| `weight`               | Double (Nullable) | 사용자 몸무게                     |
| `address`              | String (Nullable) | 사용자 주소                      |
| `detailedAddress`      | String (Nullable) | 상세 주소                       |
| `underlyingConditions` | String (Nullable) | 기저질환 정보                     |
| `occupation`           | String (Nullable) | 사용자 직업                      |
| `userRecords`          | List<Object>      | 사용자 기록 목록                   |

### Example

#### Request

```bash
curl -X 'GET' \
'http://15.165.125.100:8085/demo/v1/users/{userId}'
```

#### Response

##### 200 OK

```json
[
  {
    "uid": "0ca1fde8-0f50-4f1b-bebe-f0b32bb44d0a",
    "firstName": "John",
    "lastName": "Doe",
    "email": "johndoe123@example.com",
    "phoneNumber": "+82-10-1234-5678",
    "gender": "MALE",
    "birthday": "1987-06-05",
    "height": 180.0,
    "weight": 75.0,
    "address": "57, Oryundae-ro, Geumgeong-gu, Busan, Republic of Korea(46252)",
    "detailedAddress": "B-108",
    "underlyingConditions": null,
    "occupation": null,
    "userRecords": [],
    "userPoseRecords": []
  }
]
```

##### 404 NotFound

```json
[]
```

---

# `DemoController` - Create Footprint Data

## Basic Information

| Method | URL                          |
|--------|------------------------------|
| POST   | `/users/{userId}/footprints` |

## Request

### Parameters(@PathVariable)

| Name     | Type | Description | Required |
|----------|------|-------------|----------|
| `userId` | UUID | User 고유 식별자 | Yes      |

### Parameters(@RequestBody)

| Name              | Type   | Description                     | Required |
|-------------------|--------|---------------------------------|----------|
| `rawData`         | String | 측정된 데이터                         | Yes      |
| `mesuredDateTime` | String | 측정된 시간 정보(YYYY-MM-DDTHH:MM:SSZ) | Yes      |

## Response

### Body

| Name              | Type          | Description                  |
|-------------------|---------------|------------------------------|
| `recordId`        | UUID          | 기록 고유 식별자                    |
| `measuredDate`    | String (Date) | 측정 날짜 (YYYY-MM-DD 형식)        |
| `measuredTime`    | String (Time) | 측정 시간 (HH:mm:ss 형식)          |
| `rawData`         | String        | 원시 데이터 (센서 등에서 수집된 원시 데이터)   |
| `leftFootLength`  | Double        | 측정된 왼발 길이 (cm 단위)            |
| `leftFootWidth`   | Double        | 측정된 왼발 넓이 (cm 단위)            |
| `rightFootLength` | Double        | 측정된 오른발 길이 (cm 단위)           |
| `rightFootWidth`  | Double        | 측정된 오른발 넓이 (cm 단위)           |
| `weight`          | Double        | 측정된 몸무게 (kg 단위)              |
| `classType`       | Integer       | 분류 타입 (데이터 분류에 사용되는 값)       |
| `accuracy`        | Integer       | 정확도 (데이터의 신뢰성이나 정확도를 나타내는 값) |
| `imageUrl`        | String        | 측정 결과 이미지 URL                |

### Example

#### Request

```bash
curl -X 'POST' \
'http://15.165.125.100:8085/demo/v1/users/{userId}/footprints' \
--header 'Content-Type: application/json' \
--data '{
  "rawData": "",
  "mesuredDateTime": "2024-08-23T12:00:00Z"
}'

```

#### Response

##### 200 OK

```json
[
  {
    "recordId": "3ca57faf-3da5-4554-abb0-22f464af1fd5",
    "measuredDate": "2024-08-23",
    "measuredTime": "12:00:00",
    "rawData": "0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000F0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000010E171200000000000000130E1D0611000000000021050B23163300000000004B47261305260000000000270D03121B7957000000683F2E3B0605110000000000010201071C1A90000000857B5F0100000000000000000000000002393900000045382600000000000000000000000000000000000000001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000001300000000000000000000000000000000000000000B2A000000000000100000000000000000000000000918000000000000001630130302000000000003042A210C00000000000000251C162806000000000007372C2027000000000000000B1619481400000000000A293B4D45000000000000000F382B2D3500000000004D57361921000000000000000B2147285700000000008A2D251D1900000000000000011E353A4400000000000C38321C07000000000000000009232B00000000000001010100000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000F00F4E0254F00F",
    "leftFootLength": 0.0,
    "leftFootWidth": 0.0,
    "rightFootLength": 0.0,
    "rightFootWidth": 0.0,
    "weight": 78.2,
    "classType": 6,
    "accuracy": 58,
    "imageUrl": "https://carenco-service.s3.ap-northeast-2.amazonaws.com/FootPrints/b24aed12-3716-494c-8e20-94c7f429bd71/e2d67f49-6f92-4d6d-a468-d174d83c1e12_2024-09-26_15-05-21_b24aed12-3716-494c-8e20-94c7f429bd71.png"
  }
]
```

##### 400 BadRequest

```json
[]
```

##### 404 NotFound

```json
[]
```

---

# `DemoController` - Create Pose Data

## Basic Information

| Method | URL                    |
|--------|------------------------|
| POST   | `/users/{userId}/pose` |

## Request

### Parameters(@PathVariable)

| Name     | Type | Description | Required |
|----------|------|-------------|----------|
| `userId` | UUID | User 고유 식별자 | Yes      |

### Parameters(@RequestParam)

| Name   | Type          | Description | Required |
|--------|---------------|-------------|----------|
| `file` | MultipartFile | 측정된 영상 데이터  | Yes      |

## Response

### Body

| Name            | Type          | Description                              |
|-----------------|---------------|------------------------------------------|
| `recordId`      | UUID          | 기록 고유 식별자                                |
| `measuredDate`  | String (Date) | 측정 날짜 (YYYY-MM-DD 형식)                    |
| `measuredTime`  | String (Time) | 측정 시간 (HH:mm:ss 형식)                      |
| `angleFace`     | Double        | 측정된 얼굴 기울기 ('-' 값의 경우 오른쪽, '+' 값의 경우 왼쪽) |
| `angleShoulder` | Double        | 측정된 어깨 기울기 ('-' 값의 경우 오른쪽, '+' 값의 경우 왼쪽) |
| `anglePelvis`   | Double        | 측정된 골반 기울기 ('-' 값의 경우 오른쪽, '+' 값의 경우 왼쪽) |
| `classType`     | Integer       | 분류 타입 (데이터 분류에 사용되는 값)                   |
| `accuracy`      | Integer       | 정확도 (데이터의 신뢰성이나 정확도를 나타내는 값)             |
| `imageUrl`      | String        | 측정 결과 이미지 URL                            |

### Example

#### Request

```bash
curl -X 'POST' \
'http://15.165.125.100:8085/demo/v1/users/{userId}/pose' \
--header 'Content-Type: application/json' \
--form 'file=@"/path/to/file"'

```

#### Response

##### 200 OK

```json
[
  {
    "recordId": "213aec33-af64-4467-824a-abde6d52402d",
    "measuredDate": "2024-09-30",
    "measuredTime": "14:24:58.988639",
    "angleFace": -4.969740728110304,
    "anglePelvis": 1.357465844530609,
    "angleShoulder": 1.7214860981507567,
    "classType": 7,
    "accuracy": 51,
    "imageUrl": null
  }
]
```

##### 400 BadRequest

```json
[]
```

##### 404 NotFound

```json
[]
```