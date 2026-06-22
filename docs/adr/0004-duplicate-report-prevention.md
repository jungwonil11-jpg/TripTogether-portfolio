# ADR-0004: 중복 신고 방지 — DB UNIQUE + 서비스 사전 체크 + CANCELLED 재활성화 3중 방어

* Status: Accepted
* Date: 2026-04-15
* Decision-Makers: Victor Jung
* Consulted: N/A
* Informed: N/A

## Context and Problem Statement

같은 사용자가 같은 대상(글/댓글/유저)에 대해 여러 번 신고하는 것을 방지해야 했습니다. 동시에 **취소했던 신고를 다시 활성화하는 케이스** 도 자연스럽게 처리해야 합니다.

문제는 단일 방어 수단으로는 부족하다는 점입니다:

* DB UNIQUE 제약만 두면 → 애플리케이션 레이어에서 친절한 에러 메시지 제공 불가
* 서비스 SELECT 체크만 두면 → SELECT 와 INSERT 사이의 race condition 위험
* 둘 다 두더라도 → 사용자가 한 번 취소했다가 다시 신고하고 싶을 때 처리 방안 필요

## Decision Drivers

* **데이터 무결성** — 같은 (user, target) 조합에 대한 중복 row 절대 금지
* **사용자 경험** — 중복 신고 시도 시 의미 있는 응답 (그냥 500 에러 X)
* **동시성 안전** — race condition 으로 인한 우회 방지
* **CANCELLED 재신고 케이스 자연스러운 처리** — 사용자가 취소 후 다시 신고 가능해야 함

## Considered Options

* **Option A — DB UNIQUE 제약만**
  단순. 그러나 애플리케이션 레이어 분기 어려움.
* **Option B — 서비스 사전 SELECT 체크만**
  유연. 그러나 race condition 위험.
* **Option C — DB UNIQUE + 서비스 사전 체크 + CANCELLED 재활성화 3중 방어 (선택)**
  방어 다층화 + 사용자 의도 변경 케이스 자연스러운 처리.

## Decision Outcome

**Chosen option: "Option C"**, because 3중 방어로 동시성과 데이터 무결성을 동시에 보장하며, CANCELLED 재활성화 분기로 사용자 의도 변경 시나리오까지 커버합니다.

### 3중 방어 구조

#### Layer 1: DB UNIQUE 제약 (최후 보루)

```sql
ALTER TABLE REPORT
ADD UNIQUE KEY uq_report (user_idx, target_type, target_id);
```

* 어떤 경로로 INSERT 가 들어와도 같은 (user, target) 조합은 두 번째부터 차단
* 동시 요청 race condition 도 DB 레벨에서 막힘 (`DuplicateKeyException` 발생)

#### Layer 2: 서비스 사전 SELECT 체크 (사용자 친화 응답)

```text
ReportDto existing = reportMapper.selectReportByUserAndTarget(userIdx, targetType, targetId);
if (existing != null) {
    if ("CANCELLED".equals(existing.getStatus())) {
        // → Layer 3 (재활성화)
    }
    return false;  // 중복 신고 거부
}
```

* INSERT 시도 전 기존 신고 조회
* IN_REVIEW / RESOLVED / DISMISSED 상태면 거부 (false 반환 → 컨트롤러가 친절 메시지)

#### Layer 3: CANCELLED 재활성화 분기 (사용자 의도 변경)

```text
if ("CANCELLED".equals(existing.getStatus())) {
    reportMapper.reactivateCancelledReport(existing.getReportId(), reason, description, ...);
    return true;
}
```

* 사용자가 한 번 취소한 신고를 다시 활성화 가능
* UNIQUE 제약 위반 없이 자연스럽게 처리 (UPDATE 사용)

### Consequences

* Good
  * **동시성 안전** — DB UNIQUE 가 race condition 의 최후 방어선
  * **사용자 친화** — 중복 시 의미 있는 응답 가능
  * **데이터 무결성** — 같은 (user, target) 조합 중복 row 절대 발생 불가
  * **사용자 의도 변경 자연 처리** — CANCELLED → 재신고 분기로 행 1건 유지
  * **rate limit 부수 효과** — 같은 대상 도배 신고가 원천 불가 (별도 rate limit 불필요)
* Bad
  * 3가지 방어 수단을 모두 이해해야 코드 변경 시 안전 (학습 곡선)
  * SELECT + INSERT 2번 쿼리 (성능상 미세 부담)
* Neutral
  * `DuplicateKeyException` 캐치 누락 시 사용자에게 500 에러 노출 가능 → P0 보완 항목

## Pros and Cons of the Options

### Option A — UNIQUE 제약만

* Good, because 단순하고 데이터 무결성 보장
* Good, because 동시성 자동 처리
* Bad, because 사용자에게 친절한 응답 어려움 (`DuplicateKeyException` 잡고 분기해야 함)
* Bad, because **CANCELLED 재신고 케이스 처리 불가** — 새 INSERT 가 UNIQUE 위반

### Option B — 서비스 SELECT 체크만

* Good, because 사용자 친화 응답 가능
* Good, because CANCELLED 분기 가능
* Bad, because **race condition 위험** — SELECT 와 INSERT 사이 다른 트랜잭션이 INSERT 가능
* Bad, because 동시성 환경에서 중복 row 발생 가능

### Option C — 3중 방어 (선택)

* Good, because 모든 케이스 커버 (동시성 + UX + 의도 변경)
* Good, because 어떤 한 레이어가 실패해도 다른 레이어가 보완
* Bad, because 학습 곡선 있음
* Bad, because 쿼리 2번 (미세 성능 비용)

## More Information

### 관련 ADR

* [ADR-0001](./0001-report-no-auto-user-block.md) — 신고 누적 정책 (중복 방지가 누적 카운트의 신뢰성 기반)

### 코드 위치

* DB 스키마: `REPORT` 테이블 `UNIQUE KEY uq_report (user_idx, target_type, target_id)`
* 서비스 로직: `src/main/java/org/triptogether/report/service/ReportServiceImpl.java` `submitReport()` L73~97
* 사전 체크 쿼리: `ReportMapper.selectReportByUserAndTarget()`
* 재활성화 쿼리: `ReportMapper.reactivateCancelledReport()`

### 면접 어필 포인트

이 패턴은 단순 CRUD 를 넘어 **동시성·UX·사용자 의도 변경** 3가지 차원을 동시에 고려한 설계입니다. 실무에서 race condition 문제는 흔히 간과되는 영역이며, DB 제약과 애플리케이션 로직을 함께 설계하는 패턴은 시니어 수준의 의식이 필요합니다.
