# Auth Service

    - Basic Domain: https://carencoinc.com/kr/dev-auth
    - Basic IP: 15.165.125.100:8082

<!-- api-1-start -->
<details markdown="1">
<summary><strong>&nbsp;API: Reissue tokens</strong></summary>

## Basic Information

| Method | URL         |
|--------|-------------|
| POST   | `/v1/token` |

### Authorization

| Security Type   | Roles Allowed |
|-----------------|---------------|
| `@PreAuthorize` | `permitAll()` |

### Request

#### - Parameters(@Header)

| Name            | Type   | Description                                | Required | Remarks                                            |
|-----------------|--------|--------------------------------------------|----------|----------------------------------------------------|
| `Authorization` | String | `Bearer <token>` Refresh token in the form | Yes      | `--header 'Authorization: Bearer <refresh_token>'` |

#### - Parameters(@RequestParam)

| Name      | Type   | Description                                  | Required | Remarks                                                             |
|-----------|--------|----------------------------------------------|----------|---------------------------------------------------------------------|
| `version` | String | API version information (format: YYYY-MM-DD) | No       | If not provided, the latest API version will be used automatically. |

### Response

#### - Body

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

[![Run in Postman](https://run.pstmn.io/button.svg)](https://carenco.postman.co/workspace/Care%26CO~7c4d2551-cc9d-413f-b156-4c350b99eb32/request/27911837-87c5d2b9-ca93-4efd-a71f-3710aec3b07e?action=share&source=copy-link&creator=27911837&ctx=documentation)

### cURL 명령어

```bash
    curl -X POST 'https://carencoinc.com/kr/dev-auth/token?version=2025-01-31' \' \
         -H 'Authorization: Bearer <refresh_token>
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

---

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

</details>
<!-- api-1-end -->

<!-- api-2-start -->
<details markdown="1">
  <summary><strong>&nbsp;API: changePassword(Version="2025-01-31")</strong></summary>

## Basic Information

| Method | URL                                 |
|--------|-------------------------------------|
| POST   | `/v1/users/{userId}/changePassword` |

### Authorization

| Security Type   | Roles Allowed                 |
|-----------------|-------------------------------|
| `@PreAuthorize` | `hasAnyRole('ADMIN', 'USER')` |

### Request

#### - Parameters(@Header)

| Name            | Type   | Description                               | Required | Remarks                                           |
|-----------------|--------|-------------------------------------------|----------|---------------------------------------------------|
| `Authorization` | String | `Bearer <token>` Access token in the form | Yes      | `--header 'Authorization: Bearer <access_token>'` |

#### - Parameters(@PathVariable)

| Name     | Type   | Description            | Required | Remarks |
|----------|--------|------------------------|----------|---------|
| `userId` | String | User Unique identifier | Yes      |         |

#### - Parameters(@RequestParam)

| Name      | Type   | Description                                  | Required | Remarks                                                             |
|-----------|--------|----------------------------------------------|----------|---------------------------------------------------------------------|
| `version` | String | API version information (format: YYYY-MM-DD) | No       | If not provided, the latest API version will be used automatically. |

#### - Parameters(@ModelAttribute)

| Name                                    | Type   | Description  | Required | Remarks |
|-----------------------------------------|--------|--------------|----------|---------|
| `ChangePasswordRequest`                 | Object |              | Yes      |         |
| `ChangePasswordRequest.currentPassword` | Stirng | 기존 사용하던 비밀번호 | Yes      |         |
| `ChangePasswordRequest.newPassword`     | Stirng | 새로 변경할 비밀번호  | Yes      |         |

### Response

#### - Body

| Name      | Type   | Description        |
|-----------|--------|--------------------|
| `message` | String | API result message |

<details markdown=>
  <summary><strong>Example</strong></summary>

## Request

### Postman 요청

아래 버튼을 클릭하면 `Postman`에서 API 요청을 실행할 수 있습니다.

[![Run in Postman](https://run.pstmn.io/button.svg)](https://carenco.postman.co/workspace/Care%26CO~7c4d2551-cc9d-413f-b156-4c350b99eb32/request/27911837-e04e9b78-c9c1-40f6-84e4-676d67d30877?action=share&source=copy-link&creator=27911837&ctx=documentation)

### cURL 명령어

```bash
    curl --location --globoff 'https://carencoinc.com/kr/dev-auth/v1/users/{userId}/changePassword?version=2025-01-31' \
    --header 'Authorization: Bearer <access_token>' \
    --form 'currentPassword=""' \
    --form 'newPassword=""'
```

## Response

<details>
<summary><strong>200 OK</strong></summary>

###### Body

```json
{
  "message": "Change password success"
}
```

</details>

---

<details>
<summary><strong>400 BadRequest</strong></summary>

###### Body

```json
{
}
```

</details>

</details>

</details>
<!-- api-2-end -->

<!-- api-3-start -->
<details markdown="1">
  <summary><strong>&nbsp;API: Email(username)duplicationCheck(Version="2025-01-31")</strong></summary>

## Basic Information

| Method | URL                           |
|--------|-------------------------------|
| POST   | `/v1/users/duplication-check` |

### Authorization

| Security Type   | Roles Allowed |
|-----------------|---------------|
| `@PreAuthorize` | `permitAll()` |

### Request

#### - Parameters(@RequestParam)

| Name      | Type   | Description                                  | Required | Remarks                                                             |
|-----------|--------|----------------------------------------------|----------|---------------------------------------------------------------------|
| `version` | String | API version information (format: YYYY-MM-DD) | No       | If not provided, the latest API version will be used automatically. |
| `email`   | String | Emails to check for duplicates (username)    | Yes      |                                                                     |

### Response

#### - Body

| Name           | Type    | Description             |
|----------------|---------|-------------------------|
| `message`      | String  | API result message      |
| `data`         | Object  | API result data         |
| `data.success` | boolean | API result boolean data |

<details markdown=>
  <summary><strong>Example</strong></summary>

## Request

### Postman 요청

아래 버튼을 클릭하면 `Postman`에서 API 요청을 실행할 수 있습니다.

[![Run in Postman](https://run.pstmn.io/button.svg)](https://carenco.postman.co/workspace/Care%26CO~7c4d2551-cc9d-413f-b156-4c350b99eb32/request/27911837-037db517-a0ff-44f5-a26f-6149192e0a4c?action=share&source=copy-link&creator=27911837&ctx=documentation)

### cURL 명령어

```bash
    curl --location 'https://carencoinc.com/kr/dev-auth/v1/users/duplication-check?version=&email='
```

## Response

<details>
<summary><strong>200 OK</strong></summary>

###### Body

```json
{
  "message": "Email is duplicated. or Email is available.",
  "data": "true or false"
}
```

</details>

---

<details>
<summary><strong>400 BadRequest</strong></summary>

###### Body

```json
{
}
```

</details>

</details>

</details>
<!-- api-3-end -->
