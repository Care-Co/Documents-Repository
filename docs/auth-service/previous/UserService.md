# User Service

    - Basic Domain: https://carencoinc.com/kr/dev-auth
    - Basic IP: 15.165.125.100:8082

<!-- api-1-start -->
<details markdown="1">
<summary><strong>&nbsp;API: getUserinfo</strong></summary>


## API: getUserinfo

### Basic Information

| Method | URL          |
|--------|--------------|
| GET    | `/user/{id}` |

### Request

#### Parameters(@PathVariable)

| Name | Type   | Description       | Required | Remarks |
|------|--------|-------------------|----------|---------|
| `id` | String | Unique identifier | Yes      |         |

### Response

#### Body

| Name               | Type      | Description                                          |
|--------------------|-----------|------------------------------------------------------|
| `message`          | String    | The result message of the API call                   |
| `data`             | Object    | Contains the data for the user                       |
| `data.id`          | String    | Unique identifier (ID) for the user                  |
| `data.firstName`   | String    | The first name of the user                           |
| `data.lastName`    | String    | The last name of the user                            |
| `data.phoneNumber` | String    | The phone number of the user                         |
| `data.photoUrl`    | String    | The URL of the profile picture for the user          |
| `data.gender`      | String    | The gender of the user (e.g., Male, Female, Other)   |
| `data.birthday`    | LocalDate | The date of birth of the user in `YYYY-MM-DD` format |
| `data.height`      | Double    | The height of the user in centimeters                |
| `data.weight`      | Double    | The weight of the user in kilograms                  |

<details markdown=>
  <summary><strong>Example</strong></summary>


## Request

### Postman 요청

아래 버튼을 클릭하면 `Postman`에서 API 요청을 실행할 수 있습니다.

[![Run in Postman](https://run.pstmn.io/button.svg)](https://carenco.postman.co/workspace/Care%26CO~7c4d2551-cc9d-413f-b156-4c350b99eb32/request/27911837-dfccbfa3-27ad-49b1-8528-e956646d3614?action=share&creator=32584424&ctx=documentation)

```bash
curl GET 'https://carencoinc.com/kr/dev-auth/users/{id}'
```

## Response

<details>
<summary><strong>200 OK</strong></summary>


###### Body

```json
{
  "message": "user found successfully",
  "data": {
    "id": "",
    "firstName": "",
    "lastName": "",
    "email": "",
    "phoneNumber": "",
    "photoUrl": "",
    "gender": "",
    "birthday": "",
    "height": 0.0,
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
}
```
</details>

</details>

---

</details>
<!-- api-1-end -->


<!-- api-2-start -->
<details markdown="1">
<summary><strong>&nbsp;API: updateUserinfo</strong></summary>


## API: updateUserinfo

### Basic Information

| Method | URL          |
|--------|--------------|
| PUT    | `/user/{id}` |

### Request

#### Parameters(@PathVariable)

| Name | Type   | Description       | Required | Remarks |
|------|--------|-------------------|----------|---------|
| `id` | String | Unique identifier | Yes      |         |

#### Parameters(@RequestBody)

**UserDto**

| Name          | Type      | Description                                               | Required | Remarks                   |
|---------------|-----------|-----------------------------------------------------------|----------|---------------------------|
| `id`          | String    | Unique identifier for the user (currently unused)         | No       | Not in use                |
| `firstName`   | String    | The first name of the user                                | No       |                           |
| `lastName`    | String    | The last name of the user                                 | No       |                           |
| `email`       | String    | A valid email address for the user (must be unique)       | No       | Testing for modifiability |
| `phoneNumber` | String    | The phone number of the user                              | No       |                           |
| `photoUrl`    | String    | The URL for the user's profile picture (currently unused) | No       | Not in use                |
| `gender`      | String    | The gender of the user (e.g., Male, Female, Other)        | No       |                           |
| `birthday`    | LocalDate | The user's date of birth in `YYYY-MM-DD` format           | No       |                           |
| `height`      | Double    | The height of the user in centimeters                     | No       |                           |
| `weight`      | Double    | The weight of the user in kilograms                       | No       |                           |

### Response

#### Body

| Name               | Type      | Description                                          |
|--------------------|-----------|------------------------------------------------------|
| `message`          | String    | The result message of the API call                   |
| `data`             | Object    | Contains the data for the user                       |
| `data.id`          | String    | Unique identifier (ID) for the user                  |
| `data.firstName`   | String    | The first name of the user                           |
| `data.lastName`    | String    | The last name of the user                            |
| `data.phoneNumber` | String    | The phone number of the user                         |
| `data.photoUrl`    | String    | The URL of the profile picture for the user          |
| `data.gender`      | String    | The gender of the user (e.g., Male, Female, Other)   |
| `data.birthday`    | LocalDate | The date of birth of the user in `YYYY-MM-DD` format |
| `data.height`      | Double    | The height of the user in centimeters                |
| `data.weight`      | Double    | The weight of the user in kilograms                  |


<details markdown=>
  <summary><strong>Example</strong></summary>


## Request

### Postman 요청

아래 버튼을 클릭하면 `Postman`에서 API 요청을 실행할 수 있습니다.

[![Run in Postman](https://run.pstmn.io/button.svg)](https://carenco.postman.co/workspace/Care%26CO~7c4d2551-cc9d-413f-b156-4c350b99eb32/request/27911837-b66132cf-28f1-499d-98b8-a290888238f4?action=share&creator=32584424&ctx=documentation)

```bash
curl PUT 'https://carencoinc.com/kr/dev-auth/users/{id}' \
--header 'Content-Type: application/json' \
--data '{
    "firstName": "",
    "lastName": "",
    "phoneNumber": "",
    "gender": "",
    "birthday": "",
    "height": 0.0,
    "weight": 0.0
}'
```

## Response

<details>
<summary><strong>200 OK</strong></summary>


###### Body

```json
{
  "message": "user update successfully",
  "data": {
    "id": "",
    "firstName": "",
    "lastName": "",
    "email": "",
    "phoneNumber": "",
    "photoUrl": "",
    "gender": "",
    "birthday": "",
    "height": 0.0,
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
}
```

</details>

</details>

---

</details>
<!-- api-2-end -->
