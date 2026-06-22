# Architecture Decision Records (ADR)

이 폴더는 TripTogether 프로젝트의 주요 아키텍처/정책 결정을 기록한 ADR(Architecture Decision Record) 모음입니다.

## 형식

[MADR 0.6](https://adr.github.io/madr/) 표준을 따릅니다. 새 ADR 작성 시 [`template.md`](./template.md) 를 복사해서 사용하세요.

## 번호 매김 규칙

* 4자리 0-패딩 번호 (`0001`, `0002`, ...)
* 임팩트 순서로 번호 부여 (사후 정리 컨텍스트)
* 한 번 부여된 번호는 재사용하지 않음
* 결정이 폐기/대체되면 Status 만 `Deprecated` / `Superseded by ADR-XXXX` 로 변경

## 인덱스

### 신고 모듈 정책

| ID | 제목 | Status |
|---|---|---|
| [ADR-0001](./0001-report-no-auto-user-block.md) | 신고 누적 자동 제재 범위 — BLUR 까지, 그 외 어드민 수동 판단 | Accepted |
| [ADR-0003](./0003-blur-vs-blocked-policy.md) | BLUR vs BLOCKED 분기 정책 — 신고 누적은 점진적 공개, 어드민 차단은 완전 숨김 | Accepted |
| [ADR-0004](./0004-duplicate-report-prevention.md) | 중복 신고 방지 — DB UNIQUE + 서비스 사전 체크 + CANCELLED 재활성화 3중 방어 | Accepted |

### 커뮤니티 모듈

| ID | 제목 | Status |
|---|---|---|
| [ADR-0002](./0002-summernote-editor.md) | 커뮤니티 글쓰기 WYSIWYG — Summernote 채택 | Accepted |
| [ADR-0006](./0006-counter-cache-reconcile.md) | 캐시 컬럼 + Reconcile 스케줄러 패턴 — like/comment/report count 정합성 | Accepted |

### 보안/공통

| ID | 제목 | Status |
|---|---|---|
| [ADR-0005](./0005-xss-server-sanitize.md) | XSS 방지 — jsoup Safelist 서버측 sanitize | Accepted |
| [ADR-0008](./0008-soft-delete-pattern.md) | Soft Delete 패턴 — `status='DELETED'` 상태 컬럼 활용 | Accepted |
| [ADR-0009](./0009-moderation-policy-externalization.md) | 모더레이션 정책 외부화 — `ContentModerationPolicyVO` 패턴 | Accepted |
| [ADR-0010](./0010-ai-moderation-pipeline.md) | AI 모더레이션 풀 스택 파이프라인 — Perspective + ai_flagged + JSP BLUR | Accepted |
| [ADR-0011](./0011-authorization-aop-and-global-exception-handler.md) | 어노테이션 기반 권한 체크(AOP) + 글로벌 예외 처리 | Accepted |
| [ADR-0012](./0012-spring-security-csrf-partial-adoption.md) | Spring Security CSRF 부분 도입 + 점진적 확장 | Accepted |
| [ADR-0013](./0013-i18n-api-message-coverage.md) | API 응답 메시지 i18n 적용 — JSP 외 컨트롤러/서비스까지 4개 언어 | Accepted |
| [ADR-0014](./0014-junit-test-strategy.md) | JUnit 테스트 전략 — Service 단위 + ADR 정책 검증 | Accepted |

### 인프라

| ID | 제목 | Status |
|---|---|---|
| [ADR-0007](./0007-cloudinary-image-storage.md) | 이미지 스토리지 — Cloudinary 외부 CDN 채택 | Accepted |

## 작성 가이드

1. `template.md` 를 복사해서 `NNNN-slug-with-dashes.md` 파일명으로 저장
2. 슬러그는 영문 소문자 + 하이픈 (예: `0009-rate-limiting-policy.md`)
3. Status 는 `Proposed` 로 시작 → 합의/구현 완료 시 `Accepted`
4. 본 README 인덱스에 새 행 추가
5. 관련 코드에 주석으로 ADR 번호 참조 권장 (예: `// 정책: ADR-0001 참조`)

## 참고

* [MADR 공식 사이트](https://adr.github.io/madr/)
* [Michael Nygard, "Documenting Architecture Decisions" (2011)](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions)
* [ThoughtWorks Technology Radar — ADR (ADOPT)](https://www.thoughtworks.com/radar/techniques/lightweight-architecture-decision-records)
