# ADR-0009: 모더레이션 정책 외부화 — `ContentModerationPolicyVO` 패턴

* Status: Accepted
* Date: 2026-04-15
* Decision-Makers: Victor Jung
* Consulted: N/A
* Informed: N/A

## Context and Problem Statement

도배 방지(rate limiting), 신고 BLUR 임계값, AI 독성 감지 임계값 등 **모더레이션 관련 정책 값들** 을 어디에 둘지 결정해야 했습니다. 후보:

* 코드에 숫자 리터럴 직접 작성 (5분, 3개 같은 magic number)
* `application.properties` 외부 설정
* DB 테이블 + 정책 객체로 외부화

문제는 **운영 중 정책이 자주 조정**된다는 점입니다:
* 도배가 늘면 윈도우/최대치 강화
* 신고 누적 BLUR 임계값(현재 3회) 조정
* 모듈마다 다른 정책 값 필요 (커뮤니티 댓글 vs 문의 vs 게시글)

게다가 같은 정책 값이 여러 모듈에서 사용되므로, **단일 진실 출처(Single Source of Truth)** 가 필요했습니다.

## Decision Drivers

* **운영 중 정책 변경의 빈도** — 코드 재배포 없이 조정 가능해야 함
* **다중 모듈 일관성** — 커뮤니티·문의·신고 모듈에서 같은 정책을 참조
* **Magic number 제거** — 숫자 리터럴이 곳곳에 흩어지지 않도록
* **테스트 용이성** — 정책을 mock 으로 주입 가능하도록

## Considered Options

* **Option A — 코드 리터럴 직접 작성**
  단순. 변경 시 재배포 필요.
* **Option B — `application.properties` + `@Value`**
  설정 파일에서 주입. 변경 시 재시작 필요.
* **Option C — DB 테이블 + 정책 VO + 서비스 (선택)**
  운영 중 변경 가능. 단일 출처. 캐싱 가능.

## Decision Outcome

**Chosen option: "Option C — DB 테이블 + `ContentModerationPolicyVO` + `ModerationPolicyService`"**, because 운영 중 정책 조정 가능성과 다중 모듈 단일 출처 요구를 동시에 만족합니다.

### 정책 매트릭스

| 정책 키 | 적용 영역 | 의미 |
|---|---|---|
| `commentWindowMinutes` / `commentMaxCount` | 커뮤니티 댓글 도배 방지 | N분 내 M개 초과 시 차단 |
| `inquiryWindowMinutes` / `inquiryMaxCount` | 문의 도배 방지 | 동일 |
| `reportThreshold` | 신고 누적 BLUR 임계값 | reportCount >= threshold 시 BLUR |
| (확장 가능) | AI 독성 감지 임계값 등 | |

### 사용 패턴

```text
// 어떤 모듈에서든 동일하게:
ContentModerationPolicyVO policy = moderationPolicyService.getPolicy();
if (countRecent(...) >= policy.getInquiryMaxCount()) {
    throw new IllegalStateException("..."
        + policy.getInquiryWindowMinutes() + "분 내 ..." );
}
```

코드의 숫자(`policy.getInquiryMaxCount()`) 는 정책 객체에서 가져온 결과 — **magic number 아님**.

### Consequences

* Good
  * 운영 중 정책 조정 가능 (DB UPDATE 만으로)
  * 다중 모듈에서 일관된 값 참조 (단일 출처)
  * Magic number 가 코드에 흩어지지 않음
  * Mock 주입으로 테스트 용이
* Bad
  * 정책 조회 비용 (캐싱으로 완화)
  * 새 정책 키 추가 시 VO/서비스/DB 모두 수정
  * **AI 코드 분석이 숫자 리터럴 부재로 "하드코딩" 으로 잘못 분류할 위험** → 본 ADR 로 보완
* Neutral
  * 기본값 fallback 필요 (DB 미초기화 시)

## Pros and Cons of the Options

### Option A — 코드 리터럴

* Good, because 단순
* Bad, because 변경 시 재배포 필수
* Bad, because 다중 모듈 동기화 부담
* Bad, because 진정한 magic number 가 코드 곳곳에 흩어짐

### Option B — application.properties + @Value

* Good, because 코드와 분리
* Bad, because 변경 시 서버 재시작 필요
* Bad, because 정책 조회 패턴이 모듈마다 흩어짐

### Option C — DB + 정책 VO + 서비스 (선택)

* Good, because 운영 중 변경 가능
* Good, because 단일 출처 + 다중 모듈 일관성
* Good, because 테스트 용이
* Bad, because 신규 키 추가 시 다층 변경

## More Information

### 관련 ADR

* [ADR-0001](./0001-report-no-auto-user-block.md) — `reportThreshold` 정책의 직접 사용처
* [ADR-0010](./0010-ai-moderation-pipeline.md) — AI 모더레이션도 같은 정책 객체 활용

### 코드 위치

* 정책 VO: `src/main/java/org/triptogether/moderation/vo/ContentModerationPolicyVO.java`
* 정책 서비스: `src/main/java/org/triptogether/moderation/service/ModerationPolicyService.java`
* 사용 예 (커뮤니티 댓글 도배 방지): `CommunityServiceImpl.addComment()` L478~482
* 사용 예 (문의 도배 방지): `InquiryServiceImpl.writeInquiry()` L84~87
* 사용 예 (신고 BLUR 임계값): `CommunityServiceImpl.updatePostReportCache()` L664

### 면접 어필 포인트

이 패턴은 **정책-코드 분리(Policy-Code Separation)** 와 **단일 출처(Single Source of Truth)** 원칙의 적용 사례입니다. AI 코드 리뷰 도구가 "5분/3개 magic number 하드코딩" 으로 잘못 분류하기 쉽지만, 실제로는 정책 객체에서 가져온 값으로 운영 중 조정 가능합니다.
