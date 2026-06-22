# ADR-0012: Spring Security CSRF 부분 도입 + 점진적 확장

* Status: Accepted
* Date: 2026-04-26
* Decision-Makers: Victor Jung
* Consulted: N/A
* Informed: 팀원 (다른 모듈 담당자)

## Context and Problem Statement

본 프로젝트는 자체 세션 + 인터셉터 + AOP 로 인증/인가를 처리하지만, **CSRF(Cross-Site Request Forgery) 보호가 없어** 모든 POST/PUT/DELETE 요청이 외부 페이지에서의 위조 요청에 노출되는 상태였습니다.

Spring Security 도입을 검토했으나 다음 제약이 있었습니다:

* 풀 도입 시 다른 팀원 담당 모듈(auth, courses, admin 등)의 모든 폼/AJAX 도 영향 → 회귀 위험 매우 큼
* 자체 인증 시스템과 Spring Security 의 인증 시스템이 충돌할 수 있음
* 협업 컨텍스트 — 다른 팀원과의 조율 비용

CSRF 보호는 도입하되, **회귀 위험과 협업 충돌을 최소화하는 부분 도입 전략**이 필요했습니다.

## Decision Drivers

* **CSRF 보호의 시급성** — 모든 POST/PUT/DELETE 가 무방어 상태
* **회귀 위험 최소화** — 다른 팀원 담당 모듈에 영향 없음
* **자체 인증 시스템 보존** — Spring Security 의 인증 메커니즘과 충돌 회피
* **점진적 확장 가능성** — 팀 합의 후 다른 모듈로 단계적 확장
* **개발자 경험** — 기존 fetch / jQuery $.ajax 코드를 손대지 않고 자동 토큰 첨부

## Considered Options

* **Option A — 풀 도입**: 모든 모듈에 CSRF 적용. 회귀 위험 매우 큼.
* **Option B — 부분 도입 (선택)**: 본인 담당 모듈만 CSRF 적용 + monkey-patch 로 토큰 자동 첨부.
* **Option C — 인프라만 + 적용 보류**: SecurityConfig 만 만들고 실제 검증은 나중에.

## Decision Outcome

**Chosen option: "Option B — 부분 도입"**, because 회귀 위험과 협업 충돌을 최소화하면서 본인 담당 영역의 CSRF 보호를 즉시 확보하고, ADR 로 점진적 확장 계획을 명시해 다른 모듈로의 마이그레이션 경로를 열어둘 수 있습니다.

### CSRF 적용 범위

```
적용  : /community/** /report/** /inquiry/** 의 POST / PUT / DELETE
미적용: 그 외 모든 요청 (다른 모듈 + GET 요청)
```

`SecurityConfig.victorModuleMatcher()` 의 `OrRequestMatcher` 로 명시. 다른 모듈은 CSRF 검증을 거치지 않으므로 기존 동작에 영향 없음.

### Spring Security 의 다른 기능은 모두 비활성화

자체 인증 시스템(세션 + 인터셉터 + AOP) 과 충돌 회피:

```text
.authorizeHttpRequests(auth -> auth.anyRequest().permitAll())  // 자체 인터셉터/AOP 가 담당
.formLogin(form -> form.disable())
.httpBasic(basic -> basic.disable())
.logout(logout -> logout.disable())
```

Spring Security 는 본질적으로 **CSRF 필터 1개만 동작하는 모드**가 됨.

### 토큰 자동 첨부 (개발자 경험)

기존 fetch / jQuery 코드를 손대지 않도록 `common/header.jsp` 에 monkey-patch 주입:

```textscript
// 1. CSRF 토큰을 메타 태그로 노출
<meta name="_csrf" content="${_csrf.token}">
<meta name="_csrf_header" content="${_csrf.headerName}">

// 2. window.fetch wrapping → 같은 origin 요청에 자동 헤더 첨부
// 3. jQuery.ajaxSetup → $.ajax 호출에도 자동 헤더 첨부
```

→ 기존 컨트롤러/JSP/JS 의 fetch 호출은 **무수정**으로 CSRF 토큰을 자동 전송.

### Consequences

* Good
  * 본인 담당 모듈의 CSRF 무방어 상태 해소
  * 다른 모듈 영향 0 (회귀 위험 차단)
  * 기존 fetch/jQuery 코드 무수정 (개발자 경험)
  * Spring Security 인프라 도입 완료 → 점진적 확장 토대 마련
  * `ADR-0011 (어드민 권한 AOP)` 와 결합해 보안 전반 일관성 강화
* Bad
  * **부분 도입은 일관성 결여** — 같은 프로젝트에 CSRF 적용/미적용 영역 혼재
  * 다른 모듈은 여전히 CSRF 무방어 (점진적 확장 필요)
  * Monkey-patch 는 fetch 가 다른 라이브러리에서 재패치되면 충돌 가능
  * 메타 태그가 모든 페이지에 노출되므로 토큰 유출 위험은 XSS 의존 (XSS 차단은 ADR-0005 로 보강 완료)
* Neutral
  * Spring Security 의존성이 추가되지만 인증 기능은 사용 안 함

## Pros and Cons of the Options

### Option A — 풀 도입

* Good, because 일관성 + 전체 보안 강화
* Bad, because **모든 form/AJAX 수정 필요** (다른 팀원 영역 포함)
* Bad, because 회귀 위험 매우 큼
* Bad, because 협업 조율 비용

### Option B — 부분 도입 (선택)

* Good, because 본인 담당 영역 즉시 보호
* Good, because 다른 모듈 영향 0
* Good, because Monkey-patch 로 기존 코드 무수정
* Bad, because 일관성 결여 (부분 적용)

### Option C — 인프라만

* Good, because 위험 0
* Bad, because 실제 보호 동작 안 함 — 면접 어필 약함
* Bad, because 적용 시점 미정

## More Information

### 점진적 확장 계획

| 단계 | 적용 범위 | 진행 시점 |
|---|---|---|
| Phase 1 (현재) | `/community, /report, /inquiry` POST/PUT/DELETE | 2026-04-26 |
| Phase 2 | `/myPage, /admin` 등 본인 추가 담당/공동 영역 | 팀 합의 후 |
| Phase 3 | `/auth, /courses` 등 다른 팀원 담당 영역 | 팀 합의 후 |
| Phase 4 | 모든 모듈 풀 적용 + `requireCsrfProtectionMatcher` 제거 | 팀 통합 시점 |

### 코드 위치

* SecurityConfig: `src/main/java/org/triptogether/config/SecurityConfig.java`
* 토큰 노출 + 자동 첨부 JS: `src/main/webapp/WEB-INF/views/common/header.jsp` 상단
* 적용 매처: `SecurityConfig.victorModuleMatcher()`

### 동작 검증

| 시나리오 | 기대 동작 |
|---|---|
| 정상 페이지에서 fetch POST | 자동 헤더 첨부 → 200 |
| 정상 페이지에서 jQuery $.ajax | 자동 헤더 첨부 → 200 |
| 외부 페이지에서 위조 POST | CSRF 토큰 누락 → 403 |
| GET 요청 | CSRF 검증 안 함 → 정상 |
| 다른 모듈 POST (예: /auth/...) | CSRF 검증 안 함 → 정상 (기존 동작 유지) |

### 면접 어필 포인트

* **Spring Security CSRF 인프라 도입** + **자체 인증 시스템과 공존** 설계
* **부분 도입 전략으로 회귀 위험 관리** — 신입 수준 넘는 위험 의식
* **Monkey-patch 로 기존 코드 무수정** — 개발자 경험과 점진적 마이그레이션 균형
* **점진적 확장 계획을 ADR 에 명시** — 팀 협업 컨텍스트 고려

### 참고

* [Spring Security CSRF 공식 문서](https://docs.spring.io/spring-security/reference/servlet/exploits/csrf.html)
* [OWASP CSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
