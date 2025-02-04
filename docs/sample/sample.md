<!-- api-?-start -->
<details markdown="1">
  <summary><strong>&nbsp;API: (Version="YYYY-MM-DD")</strong></summary>

## Basic Information

| Method | URL |
|--------|-----|
| POST   | `/` |

### Authorization

| Security Type   | Roles Allowed                 |
|-----------------|-------------------------------|
| `@PreAuthorize` | `permitAll()`                 |
| `@PreAuthorize` | `hasAnyRole('ADMIN', 'USER')` |

### Request

#### - Parameters(@Header)

| Name            | Type   | Description                               | Required | Remarks                                           |
|-----------------|--------|-------------------------------------------|----------|---------------------------------------------------|
| `Authorization` | String | `Bearer <token>` Access token in the form | Yes      | `--header 'Authorization: Bearer <access_token>'` |

#### - Parameters(@PathVariable)

| Name | Type | Description | Required | Remarks |
|------|------|-------------|----------|---------|
| ``   |      |             |          |         |

#### - Parameters(@RequestParam)

| Name      | Type   | Description                                  | Required | Remarks                                                             |
|-----------|--------|----------------------------------------------|----------|---------------------------------------------------------------------|
| `version` | String | API version information (format: YYYY-MM-DD) | No       | If not provided, the latest API version will be used automatically. |

#### - Parameters(@RequestBody)

| Name | Type | Description | Required | Remarks |
|------|------|-------------|----------|---------|
| ``   |      |             |          |         |

#### - Parameters(@ModelAttribute)

| Name | Type | Description | Required | Remarks |
|------|------|-------------|----------|---------|
| ``   |      |             |          |         |

### Response

#### - Body

| Name           | Type    | Description             |
|----------------|---------|-------------------------|
| `message`      | String  | API result message      |
| `data`         | Object  | API result data         |

<details markdown=>
  <summary><strong>Example</strong></summary>

## Request

### Postman 요청

아래 버튼을 클릭하면 `Postman`에서 API 요청을 실행할 수 있습니다.

[![Run in Postman](https://run.pstmn.io/button.svg)](https://carenco.postman.co/workspace/Care%26CO~7c4d2551-cc9d-413f-b156-4c350b99eb32/request/27911837-037db517-a0ff-44f5-a26f-6149192e0a4c?action=share&source=copy-link&creator=27911837&ctx=documentation)

### cURL 명령어

```bash
    curl --location 'https://carencoinc.com/kr/?version='
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
<!-- api-?-end -->