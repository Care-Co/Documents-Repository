# device-service

> OpenAPI 3.1 — placeholder. No paths defined yet.

Source: `/Users/jonghak/GitHub/Care&Co/device-service`
Updated: 2026-04-27

```yaml
openapi: 3.1.0
info:
  title: device-service
  version: 0.0.0
  description: |
    Bootstrap-only Spring Boot project. No REST controllers implemented.
paths: {}
```

## Status

**No REST controllers implemented.** The repository currently contains only the Spring Boot bootstrap class and minimal configuration.

| File | Notes |
|---|---|
| `src/main/java/com/carenco/deviceservice/DeviceServiceApplication.java` | bootstrap |
| `src/main/resources/application.yaml` | application name only |
| `src/test/java/com/carenco/deviceservice/DeviceServiceApplicationTests.java` | empty test placeholder |

## Stack (configured)

- Spring Boot 4.0.5, Java 25
- Spring Web MVC, Spring Data JPA
- PostgreSQL driver
- Eureka discovery client, Spring Cloud Config client
- Spring Boot Admin client
- Lombok, DevTools

## Intended domain

Skeleton suggests an IoT device registry / device lifecycle service (registration, telemetry, ownership). Once controllers are added, regenerate this document by re-running the explorer.
