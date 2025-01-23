# `btobController` - Retrieve Companies Data

## Basic Information

| Method | URL               |
|--------|-------------------|
| GET    | `/btob/companies` |

## Request

### Parameters(@RequestParam)

| Name          | Type         | Description                           | Required |
|---------------|--------------|---------------------------------------|----------|
| `categories`  | List(String) | 검색할 카테고리 (ex. GYM,PILATES,HOSPITAL 등) | No       |
| `companyName` | String       | 검색할 회사명 (SQL `LIKE` 검색입니다)            | No       |
| `sortBy`      | String       | 결과 정렬 기준                              | Yes      |

- **Valid `sortBy` values**:
    - `registrationDateAsc`: 등록일 기준 오름차순
    - `registrationDateDesc`: 등록일 기준 내림차순
    - `totalViewCountAsc`: 조회수 오름차순
    - `totalViewCountDesc`: 조회수 내림차순

## Response

### Body

| Name                | Type    | Description    |
|---------------------|---------|----------------|
| `companyId`         | String  | 회사 ID          |
| `companyName`       | String  | 회사명            |
| `mainImageUrl`      | String  | 회사의 메인 이미지 URL |
| `subImageUrl`       | String  | 회사의 서브 이미지 URL |
| `companyAddress`    | String  | 회사 주소          |
| `companyCategories` | List    | 회사 카테고리 목록     |
| `view`              | Object  | 조회수 관련 정보      |
| `open`              | Boolean | 회사가 열려있는지 여부   |

### Example

#### Request

```bash
curl -X 'GET' \
'http://15.165.125.100:8085/demo/v1/btob/companies?categories=GYM&companyName=First&sortBy=totalViewCountDesc'
```

#### Response

##### 200 OK

```json
[
  {
    "companyId": "478ad6b8-a583-11ef-9260-0eb8f00c2b77",
    "companyName": "The First Balance Gym",
    "mainImageUrl": "http://15.165.125.100:8085/CENTER/gym01.png",
    "subImageUrl": "http://15.165.125.100:8085/CENTER/gym01.png",
    "companyAddress": "4321 Fitness St, Suite 100, Los Angeles, CA 90001",
    "companyCategories": [
      {
        "categoryId": "478fa229-a583-11ef-9260-0eb8f00c2b77",
        "category": "GYM"
      },
      {
        "categoryId": "4793ec96-a583-11ef-9260-0eb8f00c2b77",
        "category": "PILATES"
      }
    ],
    "view": {
      "viewId": "4797e8c8-a583-11ef-9260-0eb8f00c2b77",
      "dailyViewCount": 7,
      "totalViewCount": 1200,
      "registeredUserCount": 123,
      "targetUserCount": 250,
      "registrationPercentage": 0.0
    },
    "open": true
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

# `btobController` - Retrieve Company Data

## Basic Information

| Method | URL                           |
|--------|-------------------------------|
| GET    | `/btob/companies/{comapnyId}` |

## Request

### Parameters(@PathVariable)

| Name        | Type | Description    | Required |
|-------------|------|----------------|----------|
| `comapnyId` | UUID | Company 고유 식별자 | Yes      |

## Response

### Body

| Name                                    | Type              | Description    |
|-----------------------------------------|-------------------|----------------|
| `companyId`                             | UUID              | 회사 고유 식별자      |
| `companyName`                           | String            | 회사 이름          |
| `mainImageUrl`                          | String            | 회사의 메인 이미지 URL |
| `subImageUrl`                           | String            | 회사의 서브 이미지 URL |
| `companyCategories`                     | List<Object>      | 회사 카테고리 목록     |
| `companyCategories[].categoryId`        | UUID              | 카테고리 고유 식별자    |
| `companyCategories[].category`          | String            | 카테고리 이름        |
| `companyAddress`                        | String            | 회사 주소          |
| `weekdayOpeningTime`                    | String (Time)     | 평일 영업 시작 시간    |
| `weekdayClosingTime`                    | String (Time)     | 평일 영업 종료 시간    |
| `weekendOpeningTime`                    | String (Time)     | 주말 영업 시작 시간    |
| `weekendClosingTime`                    | String (Time)     | 주말 영업 종료 시간    |
| `holidayOpeningTime`                    | String (Time)     | 공휴일 영업 시작 시간   |
| `holidayClosingTime`                    | String (Time)     | 공휴일 영업 종료 시간   |
| `closedDays`                            | String            | 회사의 휴일 정보      |
| `registrationDate`                      | String (DateTime) | 회사 등록 날짜       |
| `employees`                             | List<Object>      | 직원 목록          |
| `employees[].employeeId`                | UUID              | 직원 고유 식별자      |
| `employees[].employeeName`              | String            | 직원 이름          |
| `employees[].employeePosition`          | String            | 직원 직위          |
| `employees[].employeeInfo`              | String            | 직원 정보          |
| `employees[].features`                  | String            | 직원 특징          |
| `employees[].careerList`                | List<Object>      | 직원 경력 목록       |
| `employees[].careerList[].careerId`     | UUID              | 경력 고유 식별자      |
| `employees[].careerList[].careerOrder`  | Integer           | 경력 순서          |
| `employees[].careerList[].careerDetail` | String            | 경력 상세 정보       |
| `employees[].mainPhotoUrl`              | String (Nullable) | 직원 메인 사진 URL   |
| `employees[].photoUrls`                 | List<String>      | 직원의 기타 사진 목록   |
| `view`                                  | Object            | 조회 관련 정보       |
| `view.viewId`                           | UUID              | 조회 정보 고유 식별자   |
| `view.dailyViewCount`                   | Integer           | 일일 조회수         |
| `view.totalViewCount`                   | Integer           | 총 조회수          |
| `view.registeredUserCount`              | Integer           | 등록된 유저 수       |
| `view.targetUserCount`                  | Integer           | 타겟 유저 수        |
| `view.registrationPercentage`           | Float             | 등록된 비율 (%)     |
| `open`                                  | Boolean           | 회사가 열려 있는지 여부  |

### Example

#### Request

```bash
curl -X 'GET' \
'http://15.165.125.100:8085/demo/v1/btob/companies/{companyId}'
```

#### Response

##### 200 OK

```json
{
  "companyId": "478ad6b8-a583-11ef-9260-0eb8f00c2b77",
  "companyName": "The First Balance Gym",
  "mainImageUrl": "http://15.165.125.100:8085/CENTER/gym01.png",
  "subImageUrl": "http://15.165.125.100:8085/CENTER/gym01.png",
  "companyCategories": [
    {
      "categoryId": "478fa229-a583-11ef-9260-0eb8f00c2b77",
      "category": "GYM"
    },
    {
      "categoryId": "4793ec96-a583-11ef-9260-0eb8f00c2b77",
      "category": "PILATES"
    }
  ],
  "companyAddress": "4321 Fitness St, Suite 100, Los Angeles, CA 90001",
  "weekdayOpeningTime": "08:00:00",
  "weekdayClosingTime": "22:30:00",
  "weekendOpeningTime": "07:00:00",
  "weekendClosingTime": "23:00:00",
  "holidayOpeningTime": "07:00:00",
  "holidayClosingTime": "23:00:00",
  "closedDays": "Closed on the 7th day of every month",
  "registrationDate": "2024-02-04T09:00:00",
  "employees": [
    {
      "employeeId": "479cb1f9-a583-11ef-9260-0eb8f00c2b77",
      "employeeName": "Liam",
      "employeePosition": "Personal Trainer",
      "employeeInfo": "3rd Year",
      "features": "",
      "careerList": [
        {
          "careerId": "47a10609-a583-11ef-9260-0eb8f00c2b77",
          "careerOrder": 1,
          "careerDetail": "Currently at The First Balance Gym"
        },
        {
          "careerId": "47a554db-a583-11ef-9260-0eb8f00c2b77",
          "careerOrder": 2,
          "careerDetail": "Former Personal Trainer at MJ Fitness"
        },
        {
          "careerId": "47a99e5b-a583-11ef-9260-0eb8f00c2b77",
          "careerOrder": 3,
          "careerDetail": "Licensed Physical Therapist"
        },
        {
          "careerId": "47ae157f-a583-11ef-9260-0eb8f00c2b77",
          "careerOrder": 4,
          "careerDetail": "Level 2 Sports Instructor"
        },
        {
          "careerId": "47b261f9-a583-11ef-9260-0eb8f00c2b77",
          "careerOrder": 5,
          "careerDetail": "Completed Personal Trainer Course"
        }
      ],
      "mainPhotoUrl": "http://15.165.125.100:8085/TRAINER/Liam01.png",
      "photoUrl1": "http://15.165.125.100:8085/TRAINER/Liam02.jpg",
      "photoUrl2": "http://15.165.125.100:8085/TRAINER/Liam03.jpg",
      "photoUrl3": "http://15.165.125.100:8085/TRAINER/Liam04.jpg",
      "photoUrl4": null,
      "photoUrl5": null,
      "photoUrl6": null,
      "photoUrl7": null,
      "photoUrl8": null,
      "photoUrl9": null
    }
  ],
  "view": {
    "viewId": "4797e8c8-a583-11ef-9260-0eb8f00c2b77",
    "dailyViewCount": 8,
    "totalViewCount": 1201,
    "registeredUserCount": 123,
    "targetUserCount": 250,
    "registrationPercentage": 0.0
  },
  "open": true
}
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

# `btobController` - Create Company Data

## Basic Information

| Method | URL               |
|--------|-------------------|
| POST   | `/btob/companies` |

## Request

### Parameters(@RequestBody)

| Name                 | Type         | Description                     | Required |
|----------------------|--------------|---------------------------------|----------|
| `companyName`        | String       | 회사 이름                           | Yes      |
| `companyCategories`  | List<String> | 회사 카테고리 목록 (ex: GYM, PILATES 등) | Yes      |
| `companyAddress`     | String       | 회사 주소                           | Yes      |
| `mainImageUrl`       | String       | 회사의 메인 이미지 URL                  | Yes      |
| `subImageUrl`        | String       | 회사의 서브 이미지 URL                  | Yes      |
| `isOpen`             | boolean      | 회사가 열려있는지 여부                    | Yes      |
| `weekdayOpeningTime` | LocalTime    | 평일 영업 시작 시간                     | Yes      |
| `weekdayClosingTime` | LocalTime    | 평일 영업 종료 시간                     | Yes      |
| `weekendOpeningTime` | LocalTime    | 주말 영업 시작 시간                     | Yes      |
| `weekendClosingTime` | LocalTime    | 주말 영업 종료 시간                     | Yes      |
| `holidayOpeningTime` | LocalTime    | 공휴일 영업 시작 시간                    | Yes      |
| `holidayClosingTime` | LocalTime    | 공휴일 영업 종료 시간                    | Yes      |
| `closedDays`         | String       | 휴일 정보 (ex: 일요일, 공휴일)            | Yes      |

## Response

### Body

| Name                | Type    | Description    |
|---------------------|---------|----------------|
| `companyId`         | String  | 회사 ID          |
| `companyName`       | String  | 회사명            |
| `mainImageUrl`      | String  | 회사의 메인 이미지 URL |
| `subImageUrl`       | String  | 회사의 서브 이미지 URL |
| `companyAddress`    | String  | 회사 주소          |
| `companyCategories` | List    | 회사 카테고리 목록     |
| `view`              | Object  | 조회수 관련 정보      |
| `open`              | Boolean | 회사가 열려있는지 여부   |

### Example

#### Request

```bash
curl -X 'POST' \
'http://15.165.125.100:8085/demo/v1/btob/companies' \
--header 'Content-Type: application/json' \
--data '{
  "companyName": "String",
  "companyCategories": ["GYM","PILATES","HOSPITAL"],
  "companyAddress": "String",
  "weekdayOpeningTime": "LocalTime(HH:MM:SS)",
  "weekdayClosingTime": "LocalTime(HH:MM:SS)",
  "weekendOpeningTime": "LocalTime(HH:MM:SS)",
  "weekendClosingTime": "LocalTime(HH:MM:SS)",
  "holidayOpeningTime": "LocalTime(HH:MM:SS)",
  "holidayClosingTime": "LocalTime(HH:MM:SS)",
  "closedDays": "String",
  "isOpen": "boolean"
}'

```

#### Response

##### 200 OK

```json
[
  {
    "companyId": "e3b4271c-ead7-4a68-9d8b-85735fe986a0",
    "companyName": "TestCompany1",
    "mainImageUrl": "http://15.165.125.100:8085/Center/center01.png",
    "subImageUrl": "http://15.165.125.100:8085/Center/center01.png",
    "companyAddress": "TestAddress",
    "companyCategories": [
      {
        "categoryId": "2c6f43d5-3f93-46bb-9509-b79edd168bc4",
        "category": "GYM"
      },
      {
        "categoryId": "6335801b-c2bb-448e-a8bc-4d2708f0ba69",
        "category": "HOSPITAL"
      },
      {
        "categoryId": "d8b0c3ff-33db-48b5-8afd-6adf36777c9f",
        "category": "PILATES"
      }
    ],
    "view": {
      "viewId": "dd74995a-9ac7-46ab-aee2-bd4c765f926c",
      "dailyViewCount": 0,
      "totalViewCount": 0,
      "registeredUserCount": 0,
      "targetUserCount": 0,
      "registrationPercentage": 0.0
    },
    "open": false
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

# `btobController` - Delete Company Data

## Basic Information

| Method | URL                           |
|--------|-------------------------------|
| DELETE | `/btob/companies/{companyId}` |

## Request

### Parameters(@PathVariable)

| Name        | Type | Description    | Required |
|-------------|------|----------------|----------|
| `comapnyId` | UUID | Company 고유 식별자 | Yes      |

## Response

### Body

| Name          | Type   | Description |
|---------------|--------|-------------|
| `Description` | String | 결과안내        |

### Example

#### Request

```bash
curl -X 'DELETE' \
'http://15.165.125.100:8085/demo/v1/btob/companies/{companyId}'
```

#### Response

##### 200 OK

```json
[
  "Btob Companies deleted successfully"
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

# `btobController` - Retrieve Employee Data

## Basic Information

| Method | URL                            |
|--------|--------------------------------|
| GET    | `/btob/employees/{employeeId}` |

## Request

### Parameters(@PathVariable)

| Name         | Type | Description     | Required |
|--------------|------|-----------------|----------|
| `employeeId` | UUID | Employee 고유 식별자 | Yes      |

## Response

### Body

| Name                                       | Type              | Description         |
|--------------------------------------------|-------------------|---------------------|
| `employeeId`                               | UUID              | 직원 고유 식별자           |
| `employeeName`                             | String            | 직원 이름               |
| `employeePosition`                         | String            | 직원 직위               |
| `employeeInfo`                             | String            | 직원 정보               |
| `features`                                 | String            | 직원 특징               |
| `mainPhotoUrl`                             | String (Nullable) | 직원 메인 사진 URL        |
| `photoUrl1`                                | String            | 직원의 추가 사진 URL (1)   |
| `photoUrl2`                                | String            | 직원의 추가 사진 URL (2)   |
| `photoUrl3`                                | String            | 직원의 추가 사진 URL (3)   |
| `photoUrl4`                                | String            | 직원의 추가 사진 URL (4)   |
| `photoUrl5`                                | String            | 직원의 추가 사진 URL (5)   |
| `photoUrl6`                                | String            | 직원의 추가 사진 URL (6)   |
| `photoUrl7`                                | String            | 직원의 추가 사진 URL (7)   |
| `photoUrl8`                                | String            | 직원의 추가 사진 URL (8)   |
| `photoUrl9`                                | String            | 직원의 추가 사진 URL (9)   |
| `careerList`                               | List<Object>      | 직원의 경력 목록           |
| `careerList[].careerId`                    | UUID              | 경력 고유 식별자           |
| `careerList[].careerOrder`                 | Integer           | 경력 순서               |
| `careerList[].careerDetail`                | String            | 경력 상세 정보            |
| `company`                                  | Object            | 직원이 속한 회사 정보        |
| `company.companyId`                        | UUID              | 회사 고유 식별자           |
| `company.companyName`                      | String            | 회사 이름               |
| `company.mainImageUrl`                     | String            | 회사의 메인 이미지 URL      |
| `company.subImageUrl`                      | String            | 회사의 서브 이미지 URL      |
| `company.companyCategories`                | List<Object>      | 회사 카테고리 목록          |
| `company.companyCategories[].categoryId`   | UUID              | 카테고리 고유 식별자         |
| `company.companyCategories[].categoryName` | String            | 카테고리 이름 (현재 null 값) |
| `company.companyAddress`                   | String            | 회사 주소               |
| `company.weekdayOpeningTime`               | String (Time)     | 평일 영업 시작 시간         |
| `company.weekdayClosingTime`               | String (Time)     | 평일 영업 종료 시간         |
| `company.weekendOpeningTime`               | String (Time)     | 주말 영업 시작 시간         |
| `company.weekendClosingTime`               | String (Time)     | 주말 영업 종료 시간         |
| `company.holidayOpeningTime`               | String (Time)     | 공휴일 영업 시작 시간        |
| `company.holidayClosingTime`               | String (Time)     | 공휴일 영업 종료 시간        |
| `company.closedDays`                       | String            | 회사의 휴일 정보           |
| `company.registrationDate`                 | String (DateTime) | 회사 등록 날짜            |
| `company.view`                             | Object            | 회사 조회 관련 정보         |
| `company.view.viewId`                      | UUID              | 조회 정보 고유 식별자        |
| `company.view.dailyViewCount`              | Integer           | 일일 조회수              |
| `company.view.totalViewCount`              | Integer           | 총 조회수               |
| `company.view.registeredUserCount`         | Integer           | 등록된 유저 수            |
| `company.view.targetUserCount`             | Integer           | 타겟 유저 수             |
| `company.view.registrationPercentage`      | Float             | 등록된 비율 (%)          |
| `company.open`                             | Boolean           | 회사가 열려 있는지 여부       |

### Example

#### Request

```bash
curl -X 'GET' \
'http://15.165.125.100:8085/demo/v1/btob/employees/{employeeId}'
```

#### Response

##### 200 OK

```json
{
  "employeeId": "479cb1f9-a583-11ef-9260-0eb8f00c2b77",
  "employeeName": "Liam",
  "employeePosition": "Personal Trainer",
  "employeeInfo": "3rd Year",
  "features": "",
  "mainPhotoUrl": "http://15.165.125.100:8085/TRAINER/Liam01.png",
  "photoUrl1": "http://15.165.125.100:8085/TRAINER/Liam02.jpg",
  "photoUrl2": "http://15.165.125.100:8085/TRAINER/Liam03.jpg",
  "photoUrl3": "http://15.165.125.100:8085/TRAINER/Liam04.jpg",
  "photoUrl4": null,
  "photoUrl5": null,
  "photoUrl6": null,
  "photoUrl7": null,
  "photoUrl8": null,
  "photoUrl9": null,
  "careerList": [
    {
      "careerId": "47a10609-a583-11ef-9260-0eb8f00c2b77",
      "careerOrder": 1,
      "careerDetail": "Currently at The First Balance Gym"
    },
    {
      "careerId": "47a554db-a583-11ef-9260-0eb8f00c2b77",
      "careerOrder": 2,
      "careerDetail": "Former Personal Trainer at MJ Fitness"
    },
    {
      "careerId": "47a99e5b-a583-11ef-9260-0eb8f00c2b77",
      "careerOrder": 3,
      "careerDetail": "Licensed Physical Therapist"
    },
    {
      "careerId": "47ae157f-a583-11ef-9260-0eb8f00c2b77",
      "careerOrder": 4,
      "careerDetail": "Level 2 Sports Instructor"
    },
    {
      "careerId": "47b261f9-a583-11ef-9260-0eb8f00c2b77",
      "careerOrder": 5,
      "careerDetail": "Completed Personal Trainer Course"
    }
  ],
  "company": {
    "companyId": "478ad6b8-a583-11ef-9260-0eb8f00c2b77",
    "companyName": "The First Balance Gym",
    "mainImageUrl": "http://15.165.125.100:8085/CENTER/gym01.png",
    "subImageUrl": "http://15.165.125.100:8085/CENTER/gym01.png",
    "companyCategories": [
      {
        "categoryId": "478fa229-a583-11ef-9260-0eb8f00c2b77",
        "categoryName": null
      },
      {
        "categoryId": "4793ec96-a583-11ef-9260-0eb8f00c2b77",
        "categoryName": null
      }
    ],
    "companyAddress": "4321 Fitness St, Suite 100, Los Angeles, CA 90001",
    "weekdayOpeningTime": "08:00:00",
    "weekdayClosingTime": "22:30:00",
    "weekendOpeningTime": "07:00:00",
    "weekendClosingTime": "23:00:00",
    "holidayOpeningTime": "07:00:00",
    "holidayClosingTime": "23:00:00",
    "closedDays": "Closed on the 7th day of every month",
    "registrationDate": "2024-02-04T09:00:00",
    "view": {
      "viewId": "4797e8c8-a583-11ef-9260-0eb8f00c2b77",
      "dailyViewCount": 9,
      "totalViewCount": 1202,
      "registeredUserCount": 123,
      "targetUserCount": 250,
      "registrationPercentage": 0.0
    },
    "open": true
  }
}
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

# `btobController` - Create Employee Data

## Basic Information

| Method | URL                                     |
|--------|-----------------------------------------|
| POST   | `/btob/companies/{companyId}/employees` |

## Request

### Parameters(@PathVariable)

| Name        | Type | Description    | Required |
|-------------|------|----------------|----------|
| `companyId` | UUID | Company 고유 식별자 | Yes      |

### Parameters(@RequestBody)

| Name                        | Type         | Description | Required |
|-----------------------------|--------------|-------------|----------|
| `employeeName`              | String       | 직원 이름       | Yes      |
| `employePosition`           | String       | 직원 직위       | Yes      |
| `employeeInfo`              | String       | 직원 정보       | No       |
| `features`                  | String       | 직원 특징       | No       |
| `careerList`                | List<Object> | 직원 경력 목록    | No       |
| `careerList[].careerOrder`  | Integer      | 경력 순서       | Yes      |
| `careerList[].careerDetail` | String       | 경력 상세 정보    | Yes      |

## Response

### Body

| Name                                       | Type              | Description         |
|--------------------------------------------|-------------------|---------------------|
| `employeeId`                               | UUID              | 직원 고유 식별자           |
| `employeeName`                             | String            | 직원 이름               |
| `employeePosition`                         | String            | 직원 직위               |
| `employeeInfo`                             | String            | 직원 정보               |
| `features`                                 | String            | 직원 특징               |
| `mainPhotoUrl`                             | String (Nullable) | 직원 메인 사진 URL        |
| `photoUrl1`                                | String            | 직원의 추가 사진 URL (1)   |
| `photoUrl2`                                | String            | 직원의 추가 사진 URL (2)   |
| `photoUrl3`                                | String            | 직원의 추가 사진 URL (3)   |
| `photoUrl4`                                | String            | 직원의 추가 사진 URL (4)   |
| `photoUrl5`                                | String            | 직원의 추가 사진 URL (5)   |
| `photoUrl6`                                | String            | 직원의 추가 사진 URL (6)   |
| `photoUrl7`                                | String            | 직원의 추가 사진 URL (7)   |
| `photoUrl8`                                | String            | 직원의 추가 사진 URL (8)   |
| `photoUrl9`                                | String            | 직원의 추가 사진 URL (9)   |
| `careerList`                               | List<Object>      | 직원의 경력 목록           |
| `careerList[].careerId`                    | UUID              | 경력 고유 식별자           |
| `careerList[].careerOrder`                 | Integer           | 경력 순서               |
| `careerList[].careerDetail`                | String            | 경력 상세 정보            |
| `company`                                  | Object            | 직원이 속한 회사 정보        |
| `company.companyId`                        | UUID              | 회사 고유 식별자           |
| `company.companyName`                      | String            | 회사 이름               |
| `company.mainImageUrl`                     | String            | 회사의 메인 이미지 URL      |
| `company.subImageUrl`                      | String            | 회사의 서브 이미지 URL      |
| `company.companyCategories`                | List<Object>      | 회사 카테고리 목록          |
| `company.companyCategories[].categoryId`   | UUID              | 카테고리 고유 식별자         |
| `company.companyCategories[].categoryName` | String            | 카테고리 이름 (현재 null 값) |
| `company.companyAddress`                   | String            | 회사 주소               |
| `company.weekdayOpeningTime`               | String (Time)     | 평일 영업 시작 시간         |
| `company.weekdayClosingTime`               | String (Time)     | 평일 영업 종료 시간         |
| `company.weekendOpeningTime`               | String (Time)     | 주말 영업 시작 시간         |
| `company.weekendClosingTime`               | String (Time)     | 주말 영업 종료 시간         |
| `company.holidayOpeningTime`               | String (Time)     | 공휴일 영업 시작 시간        |
| `company.holidayClosingTime`               | String (Time)     | 공휴일 영업 종료 시간        |
| `company.closedDays`                       | String            | 회사의 휴일 정보           |
| `company.registrationDate`                 | String (DateTime) | 회사 등록 날짜            |
| `company.view`                             | Object            | 회사 조회 관련 정보         |
| `company.view.viewId`                      | UUID              | 조회 정보 고유 식별자        |
| `company.view.dailyViewCount`              | Integer           | 일일 조회수              |
| `company.view.totalViewCount`              | Integer           | 총 조회수               |
| `company.view.registeredUserCount`         | Integer           | 등록된 유저 수            |
| `company.view.targetUserCount`             | Integer           | 타겟 유저 수             |
| `company.view.registrationPercentage`      | Float             | 등록된 비율 (%)          |
| `company.open`                             | Boolean           | 회사가 열려 있는지 여부       |

### Example

#### Request

```bash
curl -X 'POST' \
'http://15.165.125.100:8085/demo/v1/btob/companies/{companyId}/employees' \
--header 'Content-Type: application/json' \
--data '{
  "employeeName": "String",
  "employePosition": "String",
  "employeeInfo": "String",
  "features": "String",
  "careerList": [
    {
      "careerOrder": "integer",
      "careerDetail": "String"
    }
  ]
}'
```

#### Response

##### 200 OK

```json
[
  {
    "employeeId": "2e25559e-3098-4d46-b1a7-8941d4dd8886",
    "employeeName": "TestEmployee3",
    "employeePosition": "TestPosition",
    "employeeInfo": "TestInfo",
    "features": "TestFeatures",
    "mainPhotoUrl": null,
    "photoUrl1": "http://15.165.125.100:8085/Trainer/trainer01.png",
    "photoUrl2": "http://15.165.125.100:8085/Trainer/trainer01.png",
    "photoUrl3": "http://15.165.125.100:8085/Trainer/trainer01.png",
    "photoUrl4": "http://15.165.125.100:8085/Trainer/trainer02.png",
    "photoUrl5": "http://15.165.125.100:8085/Trainer/trainer02.png",
    "photoUrl6": "http://15.165.125.100:8085/Trainer/trainer02.png",
    "photoUrl7": "http://15.165.125.100:8085/Trainer/trainer03.png",
    "photoUrl8": "http://15.165.125.100:8085/Trainer/trainer03.png",
    "photoUrl9": "http://15.165.125.100:8085/Trainer/trainer03.png",
    "careerList": [
      {
        "careerId": "52b754ed-9a4c-496d-890e-be69802cc00c",
        "careerOrder": 1,
        "careerDetail": "careerDetail"
      },
      {
        "careerId": "9e19470e-0cc9-49ef-b5c8-dbecc46f2095",
        "careerOrder": 2,
        "careerDetail": "careerDetail"
      },
      {
        "careerId": "27acd6ac-ac0e-4699-a146-5437af192660",
        "careerOrder": 3,
        "careerDetail": "careerDetail"
      },
      {
        "careerId": "456f0f3c-cea8-4982-8670-d5f17046eb83",
        "careerOrder": 4,
        "careerDetail": "careerDetail"
      },
      {
        "careerId": "4f1522db-581d-41eb-8328-c54f2520bb9e",
        "careerOrder": 5,
        "careerDetail": "careerDetail"
      }
    ],
    "company": {
      "companyId": "36bdbe92-9108-462f-a227-cd8e54de86bf",
      "companyName": "TestCompany4",
      "mainImageUrl": "http://15.165.125.100:8085/Center/center03.png",
      "subImageUrl": "http://15.165.125.100:8085/Center/center03.png",
      "companyCategories": [
        {
          "categoryId": "ac1e5d0d-2ac1-4c66-b984-b7330646e5ad",
          "categoryName": null
        }
      ],
      "companyAddress": "TestAddress",
      "weekdayOpeningTime": "05:00:00",
      "weekdayClosingTime": "23:00:00",
      "weekendOpeningTime": "10:00:00",
      "weekendClosingTime": "19:00:00",
      "holidayOpeningTime": "10:00:00",
      "holidayClosingTime": "19:00:00",
      "closedDays": "TestCloseDays",
      "registrationDate": "2024-09-25T18:06:58.840413",
      "view": {
        "viewId": "cf6f86f2-6f7c-400a-89ae-1fc2039f2d9f",
        "dailyViewCount": 5,
        "totalViewCount": 8,
        "registeredUserCount": 0,
        "targetUserCount": 0,
        "registrationPercentage": 0.0
      },
      "open": false
    }
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

# `btobController` - Delete Employee Data

## Basic Information

| Method | URL                            |
|--------|--------------------------------|
| DELETE | `/btob/employees/{employeeId}` |

## Request

### Parameters(@PathVariable)

| Name         | Type | Description     | Required |
|--------------|------|-----------------|----------|
| `employeeId` | UUID | Employee 고유 식별자 | Yes      |

## Response

### Body

| Name          | Type   | Description |
|---------------|--------|-------------|
| `Description` | String | 결과안내        |

### Example

#### Request

```bash
curl -X 'DELETE' \
'http://15.165.125.100:8085/demo/v1/btob/employees/{employeeId}'
```

#### Response

##### 200 OK

```json
[
  "Btob Employee deleted successfully"
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
