# ADR-0011: 어노테이션 기반 권한 체크(AOP) + 글로벌 예외 처리

* Status: Accepted
* Date: 2026-04-26
* Decision-Makers: Victor Jung
* Consulted: N/A
* Informed: N/A

## Context and Problem Statement

기존 컨트롤러는 메서드마다 다음과 같은 보일러플레이트가 반복됐습니다:

```text
if (!isAdmin(session)) {
    result.put("success", false);
    result.put("message", "운영진만 ...");
    return ResponseEntity.status(403).body(result);
}
try {
    inquiryService.updateAnswer(...);
    result.put("success", true);
} catch (Exception e) {
    log.error(...);
    result.put("success", false);
    return ResponseEntity.status(500).body(result);
}
```

또한 `getLoginUserIdx(session)`, `isAdmin(session)` 등이 **리플렉션 ad-hoc 패턴** 으로 컨트롤러마다 사본 정의돼 있어 다음 문제가 있었습니다:

* 권한 체크 누락 위험 (메서드마다 직접 작성)
* 응답 포맷 불일치 (메서드마다 success/message 직접 작성)
* 리플렉션 호출 비용 + 타입 안전성 결여
* 변경 시 모든 사본 동기화 부담

## Decision Drivers

* **반복 코드 제거** — 권한 체크 + 예외→응답 변환 보일러플레이트
* **선언적 표현** — 메서드 시그니처만 봐도 권한 요구가 명확
* **타입 안전성** — 리플렉션 제거, `UsersVO` 직접 주입
* **일관된 응답 포맷** — AJAX 응답이 `{success, message, ...}` 표준화
* **점진적 마이그레이션** — 기존 코드 동시 변경 부담 회피

## Considered Options

* **Option A — 현 상태 유지 (리플렉션 ad-hoc)**
* **Option B — HandlerInterceptor 로 URL 패턴 기반 차단**
* **Option C — AOP Aspect + 어노테이션 + ArgumentResolver + RestControllerAdvice (선택)**

## Decision Outcome

**Chosen option: "Option C"**, because 메서드 단위 선언적 권한 체크를 위해서는 URL 패턴 기반 인터셉터로는 부족하고, AOP 어노테이션 패턴이 의도를 가장 명확히 표현합니다. 또한 `@LoginUser` 파라미터 자동 주입과 결합해 컨트롤러 본문 라인 수를 크게 줄입니다.

### 구성 요소

```
common/
├── annotation/
│   ├── @RequireLogin  (METHOD)        ← 로그인 필수 마킹
│   ├── @RequireAdmin  (METHOD)        ← 운영진 권한 마킹
│   └── @LoginUser     (PARAMETER)     ← 로그인 유저 자동 주입
├── aop/
│   └── AuthorizationAspect            ← @Before 로 진입 전 검증
├── resolver/
│   └── LoginUserArgumentResolver      ← @LoginUser 파라미터 → UsersVO 주입
└── exception/
    ├── BusinessException (base)
    ├── UnauthorizedException (401)
    ├── ForbiddenException    (403)
    ├── NotFoundException     (404)
    └── GlobalExceptionHandler         ← @RestControllerAdvice 표준 응답 변환
```

### 시범 적용 결과 (Before/After)

**Before** (`InquiryController.editAnswer` 23줄):

```text
@PostMapping("/{inquiryId}/answer/edit")
@ResponseBody
public ResponseEntity<Map<String, Object>> editAnswer(
        @PathVariable Long inquiryId,
        @RequestParam String content,
        HttpSession session) {
    Map<String, Object> result = new HashMap<>();
    if (!isAdmin(session)) {
        result.put("success", false);
        result.put("message", "운영진만 답변을 수정할 수 있어요.");
        return ResponseEntity.status(403).body(result);
    }
    try {
        inquiryService.updateAnswer(inquiryId, content, getLoginUserIdx(session));
        result.put("success", true);
    } catch (Exception e) {
        log.error("답변 수정 오류", e);
        result.put("success", false);
        return ResponseEntity.status(500).body(result);
    }
    return ResponseEntity.ok(result);
}
```

**After** (8줄):

```text
@PostMapping("/{inquiryId}/answer/edit")
@ResponseBody
@RequireAdmin
public ResponseEntity<Map<String, Object>> editAnswer(
        @PathVariable Long inquiryId,
        @RequestParam String content,
        @LoginUser UsersVO user) {
    inquiryService.updateAnswer(inquiryId, content, user.getUserIdx());
    return ResponseEntity.ok(Map.of("success", true));
}
```

→ 라인 수 64% 감소, 의도 명확성 ↑

### Consequences

* Good
  * 권한 체크 누락 위험 제거 (어노테이션 빠뜨리기는 코드 리뷰에서 즉시 보임)
  * 응답 포맷 일관 (`{success, message}`)
  * 리플렉션 제거 + 타입 안전성 확보
  * 어노테이션이 의도를 선언적으로 표현
  * `@LoginUser` 주입으로 메서드 시그니처에서 의존성 명시
* Bad
  * 시범 적용 단계라 기존 명시적 체크와 어노테이션이 **혼재** — 점진적 마이그레이션 진행 중
  * `@RestControllerAdvice` 가 일반 페이지 컨트롤러까지 영향을 미칠 수 있음 → 도메인 예외는 시범 적용 메서드에서만 발생시킴 (가정)
  * AOP 추가 의존성 (`spring-boot-starter-aop`)
* Neutral
  * 일반 사용자 페이지(view 반환)는 어노테이션을 적용하지 않으므로 영향 없음

## Pros and Cons of the Options

### Option A — 리플렉션 ad-hoc 유지

* Good, because 추가 인프라 0
* Bad, because 보일러플레이트 다수
* Bad, because 권한 체크 누락 위험
* Bad, because 리플렉션 호출 비용 + 타입 안전성 결여

### Option B — HandlerInterceptor

* Good, because Spring 표준
* Good, because 이미 LoginInterceptor 등 사용 중이라 친숙
* Bad, because URL 패턴 매칭은 메서드 단위 분기에 부적합
* Bad, because 어노테이션과 결합하려면 복잡도 증가

### Option C — AOP + 어노테이션 + ArgumentResolver (선택)

* Good, because 메서드 단위 선언적 권한 체크
* Good, because `@LoginUser` 파라미터 주입과 결합으로 시그니처 명료
* Good, because 도메인 예외 + 글로벌 핸들러로 응답 일관성
* Bad, because 학습 곡선 약간 있음
* Bad, because 인프라 신규 컴포넌트 다수

## More Information

### 마이그레이션 계획

| 단계 | 범위 |
|---|---|
| Phase 1 (현재) | 인프라 구축 + 시범 적용 (`InquiryController.editAnswer`, `getAnswerHistory`, `clearBlur`) |
| Phase 2 (예정) | InquiryController 의 모든 어드민 메서드 → 어노테이션 |
| Phase 3 (예정) | CommunityController, ReportController 의 어드민 메서드 |
| Phase 4 (예정) | 일반 로그인 필요 메서드(`@RequireLogin`) 마이그레이션 |
| Phase 5 (예정) | 기존 `getLoginUserIdx`, `isAdmin` 헬퍼 메서드 제거 |

각 컨트롤러를 한 번에 옮기지 않고 메서드 단위로 점진 적용하므로 회귀 위험을 최소화한다.

### 코드 위치

* 어노테이션: `src/main/java/org/triptogether/common/annotation/`
* AOP: `src/main/java/org/triptogether/common/aop/AuthorizationAspect.java`
* ArgumentResolver: `src/main/java/org/triptogether/common/resolver/LoginUserArgumentResolver.java`
* 도메인 예외: `src/main/java/org/triptogether/common/exception/`
* 글로벌 핸들러: `src/main/java/org/triptogether/common/exception/GlobalExceptionHandler.java`
* WebConfig 등록: `src/main/java/org/triptogether/config/WebConfig.java` `addArgumentResolvers`
* 시범 적용: `InquiryController.editAnswer / getAnswerHistory / clearBlur`

### 면접 어필 포인트

* **Spring AOP + 커스텀 어노테이션** 패턴 적용
* **HandlerMethodArgumentResolver** 로 의존성 주입 깔끔하게 표현
* **선언적 보안** (declarative security) 의 작은 사례 — Spring Security 미적용 환경에서도 비슷한 표현력 확보
* **점진적 마이그레이션** 전략 — 회귀 위험 최소화하며 모던 패턴 도입
