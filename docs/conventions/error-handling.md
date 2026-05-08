# 에러 처리 컨벤션 (Result 패턴)

> 모든 신규 서비스 / 리팩터링 대상 서비스에 적용.
> **참고 구현 두 가지** (응답 시스템에 따라 선택):
> - **패턴 A (단순 / ProblemDetail)**: [`payment-service`](../../../payment-service/) — Customer, Transaction, Subscription, Refund 도메인
> - **패턴 B (CncException / CncResponse / i18n)**: [`user-service`](../../../user-service/) — User, Auth, Terms 도메인

---

## 왜 이 규칙이 필요한가

기존 코드베이스의 두 가지 문제:

1. **"두려움 기반" try-catch** — 모든 로직을 try-catch로 감싸 에러를 조용히 삼킴. 운영에서 원인 추적 불가능.
2. **모든 실패를 예외로 표현** — `NotFoundException`, `IllegalStateException`, 또는 typed `CncException` 까지 — 결국 throw 로 stack unwind. 컨트롤러 시그니처만 봐선 어떤 실패가 가능한지 모름. `@ControllerAdvice` 가 뒤늦게 잡아 일괄 매핑.

해결 원칙:

- **예상 가능한 비즈니스 실패** → `Result<T, E>` 로 값으로 반환.
- **진짜 시스템 이상**(네트워크/IO 실패, 불변식 위반, 외부 API 5xx, Spring Security 필터 체인) → 그대로 throw.

이렇게 하면 컴파일러가 "어떤 실패를 어디서 처리할지"를 강제하고, 컨트롤러는 자기 책임의 매핑을 명시적으로 하게 된다.

**핵심 통찰**: "에러 페이로드 타입화 (CncException)" 와 "에러 제어 흐름 타입화 (Result)" 는 분리 가능하고 별개다.
user-service 처럼 이미 typed CncException 을 가진 서비스도 Result 도입 가치가 있다 — 호출자가 컴파일 타임에 처리를 강제받는다는 점에서.

---

## 패턴 결정 — A vs B

| 항목 | 패턴 A (단순) | 패턴 B (CncException 기반) |
|---|---|---|
| 적용 서비스 | payment-service 등 | user-service 등 |
| 응답 타입 | `ProblemDetail` | `CncResponse<T>` (success/code/message/error) |
| 에러 코드 체계 | 없음 (status + message만) | `ErrorCode` enum (CMN-XXX-XXX) + i18n MessageSource |
| Error 인터페이스 | 순수 sealed (Spring 의존 0) | `ServiceError` 상속 (`ErrorCode`·`messageArgs`·`description` 노출) |
| 매핑 위치 | 컨트롤러 inline `toResponse(error)` | `CncErrorResponseBuilder @Component` 어댑터 |
| 글로벌 핸들러 | 서비스의 `*ExceptionHandler` (남은 throw 만) | commoncore `GlobalExceptionHandler` (sync 잔존 throw 처리) |

**판별 방법**:
1. `build.gradle` 가 `common-core` 의존성을 가지고
2. 컨트롤러가 `ResponseHelper.success(...)` / `CncResponse` 를 사용한다 → **패턴 B**
3. 그 외 → **패턴 A**

---

## Result vs Throw — 결정 규칙 (둘 다 적용)

| 상황 | 처리 | 이유 |
|---|---|---|
| 리소스 NotFound | **Result** | 클라이언트가 잘못 요청한 정상 흐름 |
| 비즈니스 검증 실패 (금액 한도, 중복, 형식, 권한 외) | **Result** | 사용자 입력 거절 |
| 상태 전이 거절 (이미 취소된 구독 취소) | **Result** | 비즈니스 규칙 |
| `@Valid` / `@ValidUuid` 1차 검증 | (Spring 자동) | 글로벌 핸들러가 400 |
| 외부 API 5xx, IO 실패 | **throw** | 시스템 이상, 재시도/알림 |
| Webhook payload ↔ 로컬 DB 불일치 (sync 경로) | **throw** | 시스템 이상, 글로벌 핸들러가 모니터링용으로 잡음 |
| Spring Security 필터 체인 (JwtTokenService, AuthenticationProvider 등) | **throw** | 프레임워크 계약 — 변경 시 인증 깨짐 |
| 백그라운드 워커 (outbox processor 등) | **throw 또는 Result→throw 재변환** | retry/dlq 메커니즘이 예외 기반 |
| 자기 코드의 invariant 위반 | **throw IllegalStateException** | 버그 신호 |

---

## 공통 — `Result<T, E>`

```java
import com.carenco.commoncore.result.Result;

Result<MyResponse, MyError> r = ...;
```

서비스마다 만들지 말 것. **이미 common-core 에 있다.** (carenco-platform 0.0.42+)

---

## 패턴 A — 단순 (ProblemDetail)

### 1. 도메인별 Error sealed interface

위치: `<domain>/domain/<Domain>Error.java`

```java
public sealed interface RefundError {
    record NotFound(String id) implements RefundError {}
    record TransactionNotRefundable(String txnId, TransactionStatus status) implements RefundError {}
    record AmountExceedsOriginal(String itemId, String requested, String max) implements RefundError {}
    // ...
}
```

규칙:
- **sealed** — 컨트롤러 switch 에서 case 누락 시 컴파일 에러를 받기 위함.
- **record** — 실패 정보(어떤 id, 어떤 상태)를 데이터로 보존.
- **Spring/HTTP 의존 금지** — Error 는 순수 데이터. ProblemDetail 매핑은 컨트롤러에서.

### 2. Validator (선제 검증)

위치: `<domain>/service/<Domain>RequestValidator.java`

복잡한 검증(여러 필드 교차 검증, 외부 데이터 비교)은 별도 컴포넌트로 분리.

```java
@Component
public class RefundRequestValidator {
    public Result<Void, RefundError> validate(CreateRefundRequest req, Transaction txn) {
        if (!REFUNDABLE.contains(txn.getStatus()))
            return Result.failure(new RefundError.TransactionNotRefundable(...));
        // 여러 검증 후
        return Result.success(null);
    }
}
```

규칙:
- **본 로직 진입 전에 호출**. catch로 수습하지 말 것.
- 첫 번째 위반에서 즉시 `Result.failure` 반환 (fail-fast).
- `@Valid` 로 단일 필드 검증이 가능하면 그쪽이 우선.

### 3. Service — Result 반환

```java
@Transactional
public Result<RefundResponse, RefundError> createRefund(CreateRefundRequest req) {
    var txn = transactionRepository.findById(req.transactionId()).orElse(null);
    if (txn == null) return Result.failure(new RefundError.TransactionNotFound(req.transactionId()));

    var validation = validator.validate(req, txn);
    if (validation instanceof Result.Failure<Void, RefundError> f) return Result.failure(f.error());

    // 여기부터는 검증 통과. 시스템적 실패만 가능 → throw 그대로 둠.
    var paddleData = paddleApiClient.createAdjustment(...);
    return Result.success(RefundResponse.from(repo.save(...)));
}
```

규칙:
- **컨트롤러 진입점이 되는 메서드만** Result 로. 내부 헬퍼는 throw 해도 됨.
- 검증 통과 후의 코드는 "성공 경로"만 다룸. 외부 호출 실패는 try-catch 하지 말고 위로 올림.

### 4. Controller — switch 로 매핑

```java
@PostMapping
public ResponseEntity<?> create(@Valid @RequestBody CreateRefundRequest req) {
    return switch (refundService.createRefund(req)) {
        case Result.Success<...> s -> ResponseEntity.created(...).body(s.value());
        case Result.Failure<...> f -> toResponse(f.error());
    };
}

private ResponseEntity<ProblemDetail> toResponse(RefundError error) {
    ProblemDetail pd = switch (error) {
        case RefundError.NotFound e ->
                ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, "...");
        case RefundError.TransactionNotRefundable e ->
                ProblemDetail.forStatusAndDetail(HttpStatus.CONFLICT, "...");
        // sealed 라 모든 case 강제 — 새 에러 추가 시 컴파일 에러로 알림
    };
    return ResponseEntity.status(pd.getStatus()).body(pd);
}
```

---

## 패턴 B — CncException 기반 (CncResponse / i18n 보존)

### 1. 인프라 (서비스당 1번 셋업)

위치: `<service>/src/main/java/com/carenco/<service>/common/error/`

```java
// ServiceError.java — 모든 sealed Error 가 구현하는 베이스
public interface ServiceError {
    ErrorCode code();
    default Object[] messageArgs() { return new Object[0]; }
    default String description() { return null; }
}

// CncErrorResponseBuilder.java — Result.Failure → CncResponse 어댑터
// (commoncore GlobalExceptionHandler.handleCustomException 와 동일 로직)
@Component
public class CncErrorResponseBuilder {
    private final MessageSource messageSource;

    public CncErrorResponseBuilder(MessageSource messageSource) {
        this.messageSource = messageSource;
    }

    public ResponseEntity<CncResponse<Object>> toResponseEntity(ServiceError error) {
        ErrorCode code = error.code();
        String resolved = messageSource.getMessage(
            code.getMessageKey(), error.messageArgs(),
            code.getDefaultMessage(), Locale.ENGLISH);
        return ResponseEntity.status(code.getHttpStatus()).body(
            CncResponse.builder()
                .success(false).code(code.getCode())
                .message(resolved).error(error.description()).build());
    }
}
```

이미 셋업된 서비스 (user-service) 는 그대로 사용. 신규 서비스는 위 두 파일을 추가.

### 2. 도메인별 sealed Error — `ErrorCode` 보유

위치: `<domain>/domain/error/<Domain>Error.java` (또는 `application/.../error/`)

```java
public sealed interface UserError extends ServiceError {
    record NotFound(String userId) implements UserError {
        public ErrorCode code() { return ErrorCode.RESOURCE_NOT_FOUND; }
        public Object[] messageArgs() { return new Object[]{"userId", userId}; }
        public String description() { return "User not found: " + userId; }
    }
    record AccountInDeletion(String userId) implements UserError {
        public ErrorCode code() { return ErrorCode.ACCESS_DENIED; }
        public String description() { return "Account deletion in progress: " + userId; }
    }
}
```

규칙:
- **각 record 가 `ErrorCode` 를 들고 있게** — 응답 코드 + i18n 키 + HTTP status 가 자동 연결.
- 기존 throw 를 1:1 매핑: `throw new CncException(ErrorCode.X, cause, args)` → record 가 같은 ErrorCode + 같은 args 보유.
- **응답 byte-identical** 보장 — 이미 배포된 클라이언트 영향 0.

### 3. Validator (선제 검증) — 패턴 A 와 동일

`<Domain>RequestValidator @Component` 로 분리, 본 로직 진입 전 호출, fail-fast.

### 4. 도메인 간 cascade 시 wrapper Error

다른 도메인 Error 를 자기 흐름으로 끌어올릴 때:

```java
public sealed interface PasswordError extends ServiceError {
    record UserBased(UserError underlying) implements PasswordError {
        public ErrorCode code() { return underlying.code(); }
        public Object[] messageArgs() { return underlying.messageArgs(); }
        public String description() { return underlying.description(); }
    }
    record OAuthUserCannotUpdate(String userId) implements PasswordError { ... }
}
```

PasswordService 가 UserService.get 의 UserError 를 받았을 때 `Result.failure(new PasswordError.UserBased(userError))` 로 forward.

### 5. Service — Result 반환, 기존 throw 치환

```java
public Result<User, UserError> get(GetUserQuery query) {
    var user = userRepository.getUser(query.userId());
    if (user == null) return Result.failure(new UserError.NotFound(query.userId()));
    if (user.getAccountStatus() == AccountStatus.DELETION_PENDING)
        return Result.failure(new UserError.AccountInDeletion(query.userId()));
    return Result.success(user);
}
```

### 6. Controller — switch + 어댑터 호출

```java
@GetMapping(value = "/users/{userId}", version = "1.0.0")
public ResponseEntity<?> getUser(@PathVariable @ValidUuid String userId) {
    return switch (accountQueryService.get(userRequestMapper.toGetUserQuery(userId))) {
        case Result.Success<User, UserError> s ->
                ResponseHelper.success(HttpStatus.OK, userResponseMapper.toResponse(s.value()), null);
        case Result.Failure<User, UserError> f ->
                errorResponseBuilder.toResponseEntity(f.error());
    };
}
```

성공 응답은 기존 `ResponseHelper.success(...)` 그대로 — `CncResponse` 통일성 유지. 실패만 어댑터 한 줄.

---

## 내부 cascade 처리 패턴

`UserService.get` 처럼 Result 반환 메서드를 다른 서비스 / 헬퍼가 호출할 때:

| 호출자 | 처리 방법 |
|---|---|
| 컨트롤러 | `switch (result)` 매핑 |
| 같은 응용 서비스 (Result 반환) | sealed wrapper 정의 (`PasswordError.UserBased(UserError)`) 후 forward |
| 응용 서비스 내부 헬퍼 (void 반환, 이미 인증된 흐름) | private `requireUser()` 헬퍼로 Result 풀어서 IllegalStateException — invariant 위반 |
| 백그라운드 워커 | Result.failure 받으면 retry 위해 CncException 으로 재변환 throw |
| gRPC 서비스 | switch 후 gRPC `Status` 매핑 (UserAccountGrpcService 참고) |

---

## HTTP 상태 매핑 컨벤션

패턴 A: 컨트롤러 `toResponse` 에서 직접 매핑.
패턴 B: ErrorCode 의 `httpStatus` 가 자동 적용.

| Error 종류 | HTTP | ErrorCode (패턴 B) |
|---|---|---|
| `NotFound` | 404 | `RESOURCE_NOT_FOUND`, `USER_NOT_FOUND` |
| `IllegalStateTransition`, `*NotXxxable` (상태 충돌) | 409 Conflict | `DUPLICATE_REQUEST` 등 |
| 입력값 검증 실패 (`InvalidItem`, `AmountExceedsOriginal`, `DuplicateItem`) | 400 Bad Request | `CHECK_PARAMETER`, `VALIDATION_FAILED` |
| 권한 없음 | 403 Forbidden | `ACCESS_DENIED`, `PERMISSION_DENIED` |
| 인증 실패 | 401 Unauthorized | `TOKEN_INVALID`, `AUTHENTICATION_FAILED` |
| 잠금/만료 | 403 Forbidden | `ACCOUNT_LOCKED`, `ACCOUNT_EXPIRED` |

---

## 기존 서비스 마이그레이션 레시피

1. **응답 시스템 확인** → 패턴 A 또는 B 결정 (위 판별 방법)
2. **공통 의존성**
   - `build.gradle` 가 `com.carenco:common-core` 0.0.42+ 인지 확인 (Result 포함 버전)
3. **인프라 셋업** (패턴 B 만)
   - `common/error/ServiceError.java` + `common/error/CncErrorResponseBuilder.java`
4. **도메인별 sealed Error**
   - `<Domain>Error.java` 작성. 기존 throw 케이스 식별 → record 로 변환.
   - 패턴 B: 각 record 에 같은 `ErrorCode` + 같은 `messageArgs` 보유 (응답 호환성)
   - 단순 throw 는 인라인 검증, 복잡한 다중 검증은 `<Domain>RequestValidator` 로 분리
5. **Service**
   - 컨트롤러가 호출하는 메서드 시그니처를 `Result<T, E>` 로 변경
   - 패턴 A: `throw new XxxNotFoundException` → `Result.failure(new XxxError.NotFound(...))`
   - 패턴 B: `throw new CncException(ErrorCode.X, ...)` → `Result.failure(new XxxError.Y(...))`
   - **sync/webhook/Spring Security 필터 경로는 throw 유지** (시스템 계약)
6. **Controller**
   - `ResponseEntity<XxxResponse>` → `ResponseEntity<?>` (성공/실패 body 타입 다름)
   - `switch (result)` 분기
   - 패턴 A: `toResponse(error)` 헬퍼로 ProblemDetail 매핑
   - 패턴 B: `errorResponseBuilder.toResponseEntity(f.error())` 한 줄
7. **내부 cascade 처리** (위 표 참고)
   - 다른 서비스 / 헬퍼 / 워커 / gRPC 호출자 업데이트
8. **GlobalExceptionHandler 정리**
   - 컨트롤러 진입점에서 더 이상 던지지 않는 예외 핸들러 제거
   - sync 경로에서 여전히 던지는 예외 (Customer/Transaction NotFoundException 등) 는 유지
   - 사용처가 사라진 예외 클래스 파일은 삭제
9. **테스트**
   - `assertThrows(...)` → `assertInstanceOf(Result.Failure.class, ...)`
10. **빌드 검증**: `./gradlew compileJava compileTestJava`
11. **서비스 루트에 `docs/error-handling.md` 1장 추가** (도메인별 에러 매트릭스 + 시스템 throw 로 남긴 것 명시)

---

## 안티패턴 (하지 말 것)

❌ Result 를 받고 다시 throw 로 바꾸기
```java
var r = service.foo();
if (r instanceof Result.Failure f) throw new RuntimeException(...);  // 의미 없음
```

❌ 패턴 A 의 Error 타입에 Spring 의존 끼워넣기
```java
public record NotFound(...) implements XxxError {
    public ResponseEntity<?> toHttp() { ... }  // 도메인이 HTTP 를 알면 안 됨
}
```
(패턴 B 의 ServiceError 는 ErrorCode 만 노출하므로 OK — Spring 의존 아님)

❌ Result 로 외부 API 5xx 까지 흘리기 — IO 실패는 throw 가 맞음. Result 는 비즈니스 거절용.

❌ try-catch 로 Result.failure 만들기
```java
try { ... } catch (Exception e) { return Result.failure(...); }  // 진짜 예외를 삼킴
```
(예외: `DataIntegrityViolationException` 같이 명확한 비즈니스 의미를 가진 catch 는 OK — RegistrationServiceImpl 참고)

❌ Result 안 까보고 Success 캐스팅 — sealed switch 또는 `isSuccess()` 체크 없이 `((Success) r).value()` 금지.

❌ Spring Security 필터 체인 throw 를 Result 로 변경 — 인증 깨짐.

❌ 컨트롤러 응답 shape 변경 (배포된 클라이언트 영향) — 패턴 B 의 `CncErrorResponseBuilder` 가 byte-identical 보장하므로 그대로 사용.

---

## 참고 구현 (실전 예시)

### 패턴 A: `payment-service`
- Result 적용 4개 도메인: `customer`, `transaction`, `subscription`, `refund`
- 가장 풍부한 검증 사례: `refund/service/RefundRequestValidator.java`
- 상태 전이 검증 사례: `subscription/service/SubscriptionService.java#mutate`
- 도메인별 매트릭스: [`payment-service/docs/error-handling.md`](../../../payment-service/docs/error-handling.md)

### 패턴 B: `user-service`
- Result 적용 3개 도메인: `user`, `auth`, `terms` (6개 sealed Error)
- 인프라: `common/error/ServiceError.java`, `common/error/CncErrorResponseBuilder.java`
- Cascade wrapper 사례: `user/application/password/error/PasswordError.java#UserBased`
- gRPC 처리 사례: `grpc/UserAccountGrpcService.java#getUser`
- 백그라운드 워커 재변환 사례: `user/infrastructure/outbox/worker/UserDeletionLocalTransactionService.java`
- Spring Security 의도적 throw 유지: `auth/infrastructure/security/jwt/JwtTokenService.java` (6개), `auth/application/LoginAttemptService.java`
