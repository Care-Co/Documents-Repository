# Auth Service

    - Basic Domain: https://carencoinc.com/kr/dev-auth
    - Basic IP: 15.165.125.100:8082

<!-- api-1-start -->
<details markdown="1">
<summary><strong>&nbsp;API: User Register</strong></summary>

## Basic Information

| Method | URL         |
|--------|-------------|
| POST   | `/register` |

    - Email verification is required after registration to enable login.

### Request

#### Parameters(@RequestBody)

**RegisterDto**

| Name       | Type   | Description                                                                  | Required | Remarks |
|------------|--------|------------------------------------------------------------------------------|----------|---------|
| `email`    | String | A valid email address to be used as ID(username) (must be unique)            | Yes      |         |
| `password` | String | A password consisting of at least 8 characters (currently no specific rules) | Yes      |         |
| `role`     | String | The role level of the account to be created (e.g., ADMIN, USER)              | Yes      |         |

### Response

#### Header

| Name       | Type | Description                            |
|------------|------|----------------------------------------|
| `location` | URI  | The URI for user information retrieval |

#### Body

| Name          | Type   | Description                            |
|---------------|--------|----------------------------------------|
| `message`     | String | API result message                     |
| `data`        | Object | API result data                        |
| `data.userId` | String | Unique identifier for the created user |

<details markdown=>
  <summary><strong>Example</strong></summary>


## Request

### Postman 요청

아래 버튼을 클릭하면 `Postman`에서 API 요청을 실행할 수 있습니다.

[![Run in Postman](https://run.pstmn.io/button.svg)](http://naver.com)

```bash
curl --location 'https://carencoinc.com/kr/dev-auth/register' \
--header 'Content-Type: application/json' \
--data '{
    "email": "",
    "password": "",
    "role": ""
}'

```

## Response

<details>
<summary><strong>201 Created</strong></summary>


###### Header

    `Location` : `https://carencoinc.com/kr/users/{id}`

###### Body

```json
{
  "message": "user registered successfully",
  "data": "{id}"
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
<summary><strong>&nbsp;API: Login</strong></summary>


## Basic Information

| Method | URL      |
|--------|----------|
| POST   | `/login` |

### Request

#### Parameters(@RequestParam)

| Name       | Type   | Description | Required | Remarks |
|------------|--------|-------------|----------|---------|
| `username` | String |             | Yes      |         |
| `password` | String |             | Yes      |         |

### Response

#### Body

| Name                     | Type          | Description                                                   |
|--------------------------|---------------|---------------------------------------------------------------|
| `message`                | String        | API result message                                            |
| `token`                  | Object        | Contains token-related information for the authenticated user |
| `token.userId`           | String        | The unique identifier (ID) of the authenticated user          |
| `token.accessToken`      | String        | The access token used for user authentication                 |
| `token.accessExpiresAt`  | LocalDateTime | The expiration date and time of the issued access token       |
| `token.refreshToken`     | String        | The refresh token used to obtain a new access token           |
| `token.refreshExpiresAt` | LocalDateTime | The expiration date and time of the issued refresh token      |

<details markdown=>
  <summary><strong>Example</strong></summary>

## Request

### Postman 요청

아래 버튼을 클릭하면 `Postman`에서 API 요청을 실행할 수 있습니다.

[![Run in Postman](https://run.pstmn.io/button.svg)](http://naver.com)


```bash
curl --location --request POST 'https://carencoinc.com/kr/dev-auth/login?username=&password=' \
```

## Response

<details>
<summary><strong>200 OK</strong></summary>

###### Body

```json
{
  "message": "user login successfully",
  "token": {
    "userId": "",
    "accessToken": "",
    "accessExpiresAt": "",
    "refreshToken": "",
    "refreshExpiresAt": ""
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

<!-- api-3-start -->
<details markdown="1">
<summary><strong>&nbsp;API: Refresh Token</strong></summary>



## API: refreshToken

### Basic Information

| Method | URL      |
|--------|----------|
| POST   | `/token` |

### Request

#### Parameters(@RequestParam)

| Name           | Type   | Description | Required | Remarks |
|----------------|--------|-------------|----------|---------|
| `refreshToken` | String |             | Yes      |         |

### Response

#### Body

| Name                     | Type          | Description                                                   |
|--------------------------|---------------|---------------------------------------------------------------|
| `message`                | String        | API result message                                            |
| `token`                  | Object        | Contains token-related information for the authenticated user |
| `token.userId`           | String        | The unique identifier (ID) of the authenticated user          |
| `token.accessToken`      | String        | The access token used for user authentication                 |
| `token.accessExpiresAt`  | LocalDateTime | The expiration date and time of the issued access token       |
| `token.refreshToken`     | String        | The refresh token used to obtain a new access token           |
| `token.refreshExpiresAt` | LocalDateTime | The expiration date and time of the issued refresh token      |

<details markdown=>
  <summary><strong>Example</strong></summary>


## Request

### Postman 요청

아래 버튼을 클릭하면 `Postman`에서 API 요청을 실행할 수 있습니다.

[![Run in Postman](https://run.pstmn.io/button.svg)](http://naver.com)

```bash
curl --location --request POST 'https://carencoinc.com/kr/dev-auth/token?refreshToken=' \
```

## Response

<details>
<summary><strong>200 OK</strong></summary>

###### Body

```json
{
  "message": "token refreshed successfully",
  "token": {
    "userId": "",
    "accessToken": "",
    "accessExpiresAt": "",
    "refreshToken": "",
    "refreshExpiresAt": ""
  }
}
```
</details>

<details>
<summary><strong>400 BadRequest</strong></summary>

###### Body

```json
{
  "message": "token refresh failed: + e.getMassage"
}
```
</details>

</details>

---

</details>
<!-- api-3-end -->
