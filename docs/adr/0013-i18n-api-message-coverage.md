# ADR-0013: API 응답 메시지 i18n 적용 — JSP 외 컨트롤러/서비스까지 4개 언어 일관 적용

* Status: Accepted
* Date: 2026-04-26
* Decision-Makers: Victor Jung
* Consulted: 신성륜(번역 담당), notetester(공통 인프라)
* Informed: 팀원 전체

## Context and Problem Statement

CLAUDE.md 의 i18n 규칙은 **"JSP/JSPF 의 사용자 노출 문자열은 반드시 언어팩(.properties)을 통해 출력"** 만 명시되어 있습니다. 즉:

* JSP 의 `<spring:message code="..."/>` 는 명시적으로 강제됨
* 그러나 **컨트롤러의 `result.put("message", "한국어")`** 같은 응답 메시지는 규칙 외 영역
* **서비스 레이어의 `IllegalStateException("한국어")`** 도 동일

본 모듈(`community / report / inquiry`)의 컨트롤러/서비스에 한국어 메시지가 다수 하드코딩되어 있어, 다른 로케일 사용자에게도 한국어 응답이 그대로 전달되는 문제가 있었습니다.

JSP 만 i18n 적용하고 백엔드 응답 메시지는 한국어 노출 → **다국어 일관성 결여**.

## Decision Drivers

* **다국어 일관성** — JSP 만이 아니라 API 응답 / 예외 메시지까지 4개 언어 지원
* **CLAUDE.md 규칙 정신 확장** — "사용자 노출 문자열" 의 정신은 JSP 한정이 아님
* **신성륜의 i18n 패턴 존중** — 표준 Spring i18n 방식과 일치
* **notetester 가 깔아둔 인프라 활용** — `community/report/inquiry` 의 messages 파일이 이미 존재
* **본인 담당 영역 한정** — 다른 모듈은 점진적 확장 (협업 충돌 회피)

## Considered Options

* **Option A — 한국어 하드코딩 유지 (현 상태)**
  단순. 다국어 일관성 결여.
* **Option B — 본인 담당 컨트롤러 + 서비스 응답 메시지를 모두 i18n 화 (선택)**
  본인 영역 일관성. 다른 모듈 영향 0.
* **Option C — 풀 도입 (모든 모듈)**
  팀 전체 변경. 협업 충돌 위험.

## Decision Outcome

**Chosen option: "Option B"**, because 본인 담당 영역의 다국어 일관성을 즉시 확보하면서 다른 팀원 영역에는 영향을 주지 않고, 신성륜의 표준 i18n 패턴과 호환되며, notetester 가 만든 messages 인프라를 자연스럽게 보강합니다.

### 적용 범위

```
적용  : community / report / inquiry 의
        - 컨트롤러 응답 메시지 (result.put("message", ...))
        - 서비스 레이어 IllegalStateException 메시지
미적용: 그 외 모듈 (auth/flight/shop/admin/myPage 등)
        → 각 담당자 영역. 점진적 확장.
```

### 키 네이밍 규칙

```
{module}.api.error.{메서드명/동작}      ← 컨트롤러 4xx/5xx 응답
{module}.api.success.{동작}             ← 컨트롤러 성공 응답
{module}.service.error.{동작}           ← 서비스 예외 메시지
```

예시:
- `community.api.error.loginRequired`
- `report.api.success.submitted`
- `inquiry.service.error.rateLimit` (placeholder `{0}`, `{1}` 사용)

### 인프라 (`MessageUtil`)

```text
@Component @RequiredArgsConstructor
public class MessageUtil {
    private final MessageSource messageSource;

    public String get(String code) { ... }
    public String get(String code, Object... args) { ... }
}
```

* 현재 요청의 `LocaleContextHolder.getLocale()` 기준
* placeholder `{0}, {1}` 지원

### 사용 패턴 (Before/After)

**Before**:
```text
result.put("message", "로그인이 필요합니다.");
throw new IllegalStateException(
    policy.getCommentWindowMinutes() + "분 내 댓글을 "
    + policy.getCommentMaxCount() + "개 이상 작성할 수 없습니다.");
```

**After**:
```text
result.put("message", msg.get("community.api.error.loginRequired"));
throw new IllegalStateException(msg.get("community.service.error.commentRateLimit",
    policy.getCommentWindowMinutes(), policy.getCommentMaxCount()));
```

### 추가된 키 통계

| 도메인 | 키 수 | 4개 언어 합계 |
|---|---|---|
| `community` (api + service) | 21개 | 84줄 |
| `report` (api) | 9개 | 36줄 |
| `inquiry` (api + service + history) | 13개 + 8개(이전) | 84줄 |
| **합계** | **약 51개** | **약 204줄** |

### Consequences

* Good
  * 본인 담당 영역의 다국어 일관성 확보 (JSP + API 응답 + 예외 메시지)
  * 신성륜의 표준 Spring i18n 패턴과 정확히 호환
  * notetester 가 만든 messages 인프라 자연스러운 보강
  * `setUseCodeAsDefaultMessage(true)` 활성 상태라 키 누락 시 깨짐 없이 키 코드만 노출
  * 신입 어필: 4개국어 일관 적용 + 정책 외부화 의식
* Bad
  * 다른 모듈은 여전히 한국어 하드코딩 → 일관성 결여 (점진적 확장 필요)
  * 새 메시지 추가 시 4개 언어 동시 작성 부담
  * 키 누락 시 사용자에게 키 코드 그대로 노출 (런타임 검증 어려움)
* Neutral
  * `MessageUtil` 의 `LocaleContextHolder` 의존 — 비요청 컨텍스트(스케줄러 등)에서는 기본 로케일

## Pros and Cons of the Options

### Option A — 하드코딩 유지

* Good, because 추가 작업 0
* Bad, because **다국어 일관성 결여** — JSP 만 영어인데 응답 메시지는 한국어
* Bad, because CLAUDE.md 의 i18n 정신 위반

### Option B — 본인 담당만 i18n (선택)

* Good, because 본인 영역 즉시 일관성 확보
* Good, because 다른 모듈/팀원 영향 0
* Good, because 신성륜 패턴 + notetester 인프라 자연스럽게 결합
* Bad, because 부분 일관성 (다른 모듈은 추후)

### Option C — 풀 도입

* Good, because 전체 일관성
* Bad, because **다른 팀원 담당 영역 일방 변경** — 협업 충돌 위험
* Bad, because 4개 언어 × 수백 개 키 작업 부담

## More Information

### 협업 컨텍스트

* **신성륜** — 번역 담당. admin/header/home/course/explore/package 등의 messages 파일 작성. 표준 Spring i18n 패턴.
* **notetester** — `community/inquiry/report` messages 인프라 작성. CLAUDE.md i18n 규칙 추가 (커밋 `ac1682b`).
* **Victor (본인)** — 본 ADR 로 컨트롤러/서비스 응답 메시지까지 i18n 확장.

→ 본 결정은 신성륜 패턴 + notetester 인프라 와 충돌 없이 보강.

### 점진적 확장 계획

| 단계 | 적용 범위 | 진행 시점 |
|---|---|---|
| Phase 1 (현재) | community/report/inquiry 컨트롤러/서비스 메시지 | 2026-04-26 |
| Phase 2 | community/report/inquiry JSP 의 `alert()` / `confirm()` 잔여 한국어 | 후속 |
| Phase 3 | 다른 모듈 컨트롤러/서비스 (팀 합의 후) | 추후 |

### 코드 위치

* `MessageUtil`: `src/main/java/org/triptogether/common/util/MessageUtil.java`
* messages 파일: `src/main/resources/messages/{community,report,inquiry}_{ko,en,ja,zh}.properties` 의 `*.api.*` / `*.service.*` 영역
* 적용 컨트롤러: `CommunityController` / `ReportController` / `InquiryController`
* 적용 서비스: `CommunityServiceImpl` / `InquiryServiceImpl`

### 참고

* [Spring i18n / MessageSource 공식 문서](https://docs.spring.io/spring-framework/reference/core/beans/context-introduction.html#context-functionality-messagesource)
* [CLAUDE.md "다국어(i18n) 규칙" 섹션](../../CLAUDE.md) — notetester 작성, 본 ADR 의 정신적 토대

### 본 ADR 의 위치 — 신규 도입이 아닌 협업 일관성 보강

본 ADR 의 i18n 인프라(`messages/*.properties`, Spring `MessageSource`) 는 **팀 공통 자산**으로, 신성륜(번역 담당) 및 notetester(`community/inquiry/report` messages 인프라 작성) 의 작업으로 이미 마련되어 있었다.

본인의 기여 범위:
- 본인 담당 컨트롤러/서비스의 한국어 하드코딩 응답 메시지를 4개 언어 i18n 키로 일관 적용 (47 + 6곳)
- `MessageUtil` 호출 패턴 헬퍼 추가
- 협업 컨텍스트(신성륜 패턴 / notetester 인프라와의 호환) 를 ADR 로 명시

즉 **인프라 신규 도입이 아니라 "기존 팀 인프라를 본인 영역에 일관 적용" 한 작업**이다. 신규 도입 사례는 [ADR-0011 (AOP 권한 체크)](./0011-authorization-aop-and-global-exception-handler.md) / [ADR-0012 (Spring Security CSRF)](./0012-spring-security-csrf-partial-adoption.md) 등을 참조.
