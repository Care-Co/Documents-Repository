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
| `username` | String | username으로 사용할 email | Yes      |         |
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