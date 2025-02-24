# Version2 Auth + User API Documents

- BaseUrl: https://carencoinc.com/api/v2/auth

## Sign Up(register)

### Endpoint

| Method | URL         |
|--------|-------------|
| POST   | `/register` |

### Authorization

| Security Type   | Roles Allowed |
|-----------------|---------------|
| `@PreAuthorize` | `permitAll()` |

### Request

#### - Parameters(@RequestParam)

| Name      | Type   | Description                                  | Required | Remarks                                                             |
|-----------|--------|----------------------------------------------|----------|---------------------------------------------------------------------|
| `version` | String | API version information (format: YYYY-MM-DD) | No       | If not provided, the latest API version will be used automatically. |

#### - Parameters(@RequestBody)

| Name       | Type   | Description          | Required | Remarks |
|------------|--------|----------------------|----------|---------|
| `username` | String | username으로 사용할 email | Yes |         |
| `password` | String | password             | Yes      |         |

### Response

#### - Header

| Name       | Type | Description                            |
|------------|------|----------------------------------------|
| `location` | URI  | The URI for user information retrieval |

#### - Body

| Name      | Type    | Description                                                |
|-----------|---------|------------------------------------------------------------|
| `success` | boolean | Indicates whether the API call was successful (true/false) |
| `data`    | Object  | Contains the data for the user                             |
| `data.id` | String  | Unique identifier (ID) for the user                        |



## Example
### Request

```bash
  curl POST 'https://carencoinc.com/api/v2/auth/register'
    --header 'Content-Type: application/json' \
    --data '{
        "username": "",
        "password": ""
    }'
```

### Response

#### 201 Created
###### Body

```json
{
  "success": true,
  "data": "71ce665d-10c9-4fed-bd70-d53f547bf917"
}
```
#### 400 Bad Request(Duplicated User)
###### Body

```json
{
  "success": false,
  "message": "A username or email that already exists.",
  "error": "CHECK_PARAMETER"
}
```

#### 400 Bad Request(Invalid Parameter)
###### Body

```json
{
  "success": false,
  "message": "Invalid parameter entry.",
  "error": "CHECK_PARAMETER"
}
```



---

## Login

### Endpoint

| Method | URL      |
|--------|----------|
| POST   | `/login` |

### Authorization

| Security Type   | Roles Allowed |
|-----------------|---------------|
| `@PreAuthorize` | `permitAll()` |

### Request

#### - Parameters(@RequestParam)

| Name      | Type   | Description                                  | Required | Remarks                                                             |
|-----------|--------|----------------------------------------------|----------|---------------------------------------------------------------------|
| `version` | String | API version information (format: YYYY-MM-DD) | No       | If not provided, the latest API version will be used automatically. |

#### - Parameters(@RequestBody)

| Name       | Type   | Description | Required | Remarks |
|------------|--------|-------------|----------|---------|
| `username` | String | username    | Yes      |         |
| `password` | String | password    | Yes      |         |

### Response

#### - Body

| Name                 | Type    | Description                                                                   |
|----------------------|---------|-------------------------------------------------------------------------------|
| `success`            | boolean | Indicates whether the API call was successful (true/false).                   |
| `token`              | Object  | Contains authentication tokens and related user data.                         |
| `token.userId`       | String  | Unique identifier for the user.                                               |
| `token.accessToken`  | String  | Access token used for authenticating API requests.                            |
| `token.refreshToken` | String  | Refresh token used to obtain a new access token when the current one expires. |



## Example
### Request

```bash
  curl POST 'https://carencoinc.com/api/v2/auth/login'
    --header 'Content-Type: application/json' \
    --data '{
        "username": "",
        "password": ""
    }'
```

### Response

#### 200 OK
###### Body

```json
{
  "success": true,
  "token": {
    "userId": "71ce665d-10c9-4fed-bd70-d53f547bf917",
    "accessToken": "",
    "refreshToken": ""
  }
}
```
#### 400 Bad Request(Invalid Parameter)
###### Body

```json
{
  "success": false,
  "message": "Invalid parameter entry.",
  "error": "CHECK_PARAMETER"
}
```

#### 400 Bad Request(Invalid Login Credentials)
###### Body

```json
{
  "success": false,
  "message": "Invalid username or password",
  "error": "CHECK_PARAMETER"
}
```




---

## GetUserinfo

### Endpoint

| Method | URL                |
|--------|--------------------|
| GET    | `/users/{id}/info` |

### Authorization

| Security Type   | Roles Allowed     |
|-----------------|-------------------|
| `@PreAuthorize` | `hasRole('USER')` |

### Request

#### - Parameters(@Header)

| Name            | Type   | Description                               | Required | Remarks                                           |
|-----------------|--------|-------------------------------------------|----------|---------------------------------------------------|
| `Authorization` | String | `Bearer <token>` Access token in the form | Yes      | `--header 'Authorization: Bearer <access_token>'` |

#### - Parameters(@RequestParam)

| Name      | Type   | Description                                  | Required | Remarks                                                             |
|-----------|--------|----------------------------------------------|----------|---------------------------------------------------------------------|
| `id`      | String | Unique identifier for the user.              | Yes      |                                                                     |
| `version` | String | API version information (format: YYYY-MM-DD) | No       | If not provided, the latest API version will be used automatically. |

### Response

#### - Body

| Name               | Type      | Description                                                 |
|--------------------|-----------|-------------------------------------------------------------|
| `success`          | boolean   | Indicates whether the API call was successful (true/false). |
| `data`             | Object    | Contains user profile information.                          |
| `data.id`          | UUID      | Unique identifier for the user.                             |
| `data.firstName`   | String    | User's first name.                                          |
| `data.lastName`    | String    | User's last name.                                           |
| `data.email`       | String    | User's email address.                                       |
| `data.phoneNumber` | String    | User's phone number.                                        |
| `data.photoUrl`    | String    | URL to the user's profile photo.                            |
| `data.gender`      | String    | User's gender.                                              |
| `data.birthday`    | LocalDate | User's date of birth.                                       |
| `data.height`      | double    | User's height (e.g., in centimeters).                       |
| `data.weight`      | double    | User's weight (e.g., in kilograms).                         |




## Example
### Request

```bash
  curl GET 'https://carencoinc.com/api/v2/auth/users/{id}/info'
  --header 'Authorization: Bearer <access_token>
```

### Response

#### 200 OK
###### Body

```json
{
  "success": true,
  "data": {
    "id": "71ce665d-10c9-4fed-bd70-d53f547bf917",
    "firstName": "Doe",
    "lastName": "Jhon",
    "email": "sample@test.com",
    "phoneNumber": "010-1234-5678",
    "photoUrl": null,
    "gender": "male",
    "birthday": "2000-01-01",
    "height": 180,
    "weight": 60
  }
}
```
#### 403 Forbidden
###### Body

```json
{
  "success": false,
  "message": "Invalid access handling.",
  "error": "ACCESS_DENIED"
}
```

#### 401 Unauthorized(Invalid Token)
###### Body

```json
{
  "success": false,
  "message": "An error occurred while attempting to decode the Jwt: Malformed token",
  "error": "INTERNAL_SERVER_ERROR"
}
```
#### 401 Unauthorized(Expired Token)
###### Body

```json
{
  "success": false,
  "message": "Your token is invalid or expired",
  "error": "TOKEN_EXPIRED"
}
```



---

## UpdateUserinfo

### Endpoint

| Method | URL                |
|--------|--------------------|
| PUT    | `/users/{id}/info` |

### Authorization

| Security Type   | Roles Allowed     |
|-----------------|-------------------|
| `@PreAuthorize` | `hasRole('USER')` |

### Request

#### - Parameters(@Header)

| Name            | Type   | Description                               | Required | Remarks                                           |
|-----------------|--------|-------------------------------------------|----------|---------------------------------------------------|
| `Authorization` | String | `Bearer <token>` Access token in the form | Yes      | `--header 'Authorization: Bearer <access_token>'` |

#### - Parameters(@RequestParam)

| Name      | Type   | Description                                  | Required | Remarks                                                             |
|-----------|--------|----------------------------------------------|----------|---------------------------------------------------------------------|
| `id`      | String | Unique identifier for the user.              | Yes      |                                                                     |
| `version` | String | API version information (format: YYYY-MM-DD) | No       | If not provided, the latest API version will be used automatically. |

#### - Parameters(@RequestBody)

| Name          | Type      | Description                                        | Required | Remarks                    |
|---------------|-----------|----------------------------------------------------|----------|----------------------------|
| `firstName`   | String    | User's first name.                                 | No       | Typically 2-50 characters. |
| `lastName`    | String    | User's last name.                                  | No       | Typically 2-50 characters. |
| `phoneNumber` | String    | User's phone number in international format.       | No       | e.g., +821012345678        |
| `gender`      | String    | User's gender. Allowed values: `male` or `female`. | No       | Case insensitive.          |
| `birthday`    | LocalDate | User's date of birth in the format `yyyy-mm-dd`.   | No       | Example: 1990-01-01        |
| `height`      | double    | User's height in centimeters.                      | No       | e.g., 175.5                |
| `weight`      | double    | User's weight in kilograms.                        | No       | e.g., 70.0                 |

### Response

#### - Body

| Name               | Type      | Description                                                 |
|--------------------|-----------|-------------------------------------------------------------|
| `success`          | boolean   | Indicates whether the API call was successful (true/false). |
| `data`             | Object    | Contains user profile information.                          |
| `data.id`          | UUID      | Unique identifier for the user.                             |
| `data.firstName`   | String    | User's first name.                                          |
| `data.lastName`    | String    | User's last name.                                           |
| `data.email`       | String    | User's email address.                                       |
| `data.phoneNumber` | String    | User's phone number.                                        |
| `data.photoUrl`    | String    | URL to the user's profile photo.                            |
| `data.gender`      | String    | User's gender.                                              |
| `data.birthday`    | LocalDate | User's date of birth.                                       |
| `data.height`      | double    | User's height (e.g., in centimeters).                       |
| `data.weight`      | double    | User's weight (e.g., in kilograms).                         |




## Example
### Request

```bash
  curl PUT 'https://carencoinc.com/api/v2/auth/users/{id}/info'
    --header 'Content-Type: application/json' \
    --header 'Authorization: Bearer auth_key' \
    --data '{
        "firstName": "",
        "lastName": "",
        "phoneNumber": "",
        "gender": "",
        "birthday": "",
        "height": 0,
        "weight": 0
    }'
```
### Response

#### 200 OK
###### Body

```json
{
  "success": true,
  "data": {
    "id": "71ce665d-10c9-4fed-bd70-d53f547bf917",
    "firstName": "Doe",
    "lastName": "Jhon",
    "email": "sample@test.com",
    "phoneNumber": "010-1234-5678",
    "photoUrl": null,
    "gender": "male",
    "birthday": "2000-01-01",
    "height": 180,
    "weight": 60
  }
}
```
#### 401 Unauthorized
###### Body

```json
{
  "success": false,
  "message": "Your token is invalid or expired",
  "error": "TOKEN_EXPIRED"
}
```

#### 403 Forbidden
###### Body

```json
{
  "success": false,
  "message": "Invalid access handling.",
  "error": "ACCESS_DENIED"
}
```




--- 

## Withdrawal

### Endpoint

| Method | URL               |
|--------|-------------------|
| Delete | `/users/{id}` |

### Authorization

| Security Type   | Roles Allowed     |
|-----------------|-------------------|
| `@PreAuthorize` | `hasRole('USER')` |

### Request

#### - Parameters(@Header)

| Name            | Type   | Description                               | Required | Remarks                                           |
|-----------------|--------|-------------------------------------------|----------|---------------------------------------------------|
| `Authorization` | String | `Bearer <token>` Access token in the form | Yes      | `--header 'Authorization: Bearer <access_token>'` |

#### - Parameters(@RequestParam)

| Name      | Type   | Description                                  | Required | Remarks                                                             |
|-----------|--------|----------------------------------------------|----------|---------------------------------------------------------------------|
| `id`      | String | Unique identifier for the user.              | Yes      |                                                                     |
| `version` | String | API version information (format: YYYY-MM-DD) | No       | If not provided, the latest API version will be used automatically. |

### Response

#### - Body

| Name               | Type      | Description                                                 |
|--------------------|-----------|-------------------------------------------------------------|
| `success`          | boolean   | Indicates whether the API call was successful (true/false). |




## Example
### Request

```bash
  curl DELETE 'https://carencoinc.com/api/v2/auth/users/{id}'
    --header 'Authorization: Bearer auth_key' 
```

### Response


#### 401 Unauthorized
###### Body

```json
{
  "success": false,
  "message": "Your token is invalid or expired",
  "error": "TOKEN_EXPIRED"
}
```

#### 403 Forbidden
###### Body

```json
{
  "success": false,
  "message": "Invalid access handling.",
  "error": "ACCESS_DENIED"
}
```
