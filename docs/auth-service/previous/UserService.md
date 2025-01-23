# User Service

    - Basic Domain: https://carencoinc.com/kr/dev-auth
    - Basic IP: 15.165.125.100:8082

## API: getUserinfo

### Basic Information

| Method | URL          |
|--------|--------------|
| GET    | `/user/{id}` |

#### Request

###### Parameters(@PathVariable)

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

### Example

#### Request

```bash
curl GET 'https://carencoinc.com/kr/dev-auth/users/{id}'
```

#### Response

##### 200 Ok

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

##### 400 BadRequest

###### Body

```json
{
}
```


---

## API: updateUserinfo

### Basic Information

| Method | URL          |
|--------|--------------|
| PUT    | `/user/{id}` |

#### Request

###### Parameters(@PathVariable)

| Name | Type   | Description       | Required | Remarks |
|------|--------|-------------------|----------|---------|
| `id` | String | Unique identifier | Yes      |         |

###### Parameters(@RequestBody)

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

### Example

#### Request

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

#### Response

##### 200 Ok

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

##### 400 BadRequest

###### Body

```json
{
}
```

---