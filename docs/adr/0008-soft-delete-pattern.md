# ADR-0008: Soft Delete 패턴 — `status='DELETED'` 상태 컬럼 활용

* Status: Accepted
* Date: 2026-04-10
* Decision-Makers: Victor Jung
* Consulted: N/A
* Informed: N/A

## Context and Problem Statement

게시글·댓글 등 사용자 콘텐츠 삭제를 어떻게 처리할지 결정해야 했습니다. Hard Delete(실제 row 삭제)는 단순하지만 다음 문제가 있습니다:

* **신고 이력 추적 불가** — 삭제된 글의 신고 사유/처리 이력이 사라짐
* **감사 로그 부재** — 어드민이 누가 언제 무엇을 삭제했는지 확인 불가
* **복구 불가** — 사용자 실수로 삭제 시 되돌릴 수 없음
* **통계 무결성** — `comment_count`, `like_count` 등 캐시 컬럼이 어긋남
* **FK 연쇄 삭제 위험** — 댓글이 글에 FK 면 글 삭제 시 댓글도 사라짐

특히 어드민이 신고 게시판(ADR-0001 의 판단 큐)에서 신고된 글을 검토할 때, 사용자가 그 사이 삭제했더라도 **컨텍스트가 보존되어야** 합니다.

## Decision Drivers

* **신고/감사 이력 보존** — 삭제된 콘텐츠의 컨텍스트가 어드민에게 필요
* **사용자 실수 복구 가능성** — 일반 사용자 자율 삭제도 운영자가 복구 가능해야 함
* **캐시 컬럼 정합성** — `comment_count` 등 통계 컬럼 무결성 유지
* **단순한 쿼리 분기** — 리스트 조회 시 한 줄로 필터 가능해야 함

## Considered Options

* **Option A — Hard Delete (`DELETE FROM ...`)**
  단순. 컨텍스트 영구 손실.
* **Option B — Soft Delete: status 컬럼에 'DELETED' 마킹 (선택)**
  데이터 보존. 모든 쿼리에 필터 추가 필요.
* **Option C — 별도 아카이브 테이블로 이동**
  쿼리 단순. 마이그레이션/조인 비용.
* **Option D — `deleted_at` 타임스탬프 컬럼 (Rails/Laravel paranoid 스타일)**
  Soft Delete 의 변형.

## Decision Outcome

**Chosen option: "Option B — `status` 컬럼 'DELETED' 마킹"**, because 신고/감사 컨텍스트 보존이 필요하고, 이미 `post_status` / `comment_status` 같은 상태 머신 컬럼이 존재하므로 `'DELETED'` 값을 추가하는 비용이 가장 낮습니다.

### 상태 값 매트릭스

| 테이블 | 컬럼 | 값 | 의미 |
|---|---|---|---|
| `COMMUNITY_POST` | `post_status` | `ACTIVE` | 정상 |
| | | `BLOCKED` | 어드민 직접 차단 (ADR-0003) |
| | | `DELETED` | 작성자 또는 어드민 삭제 |
| `COMMUNITY_COMMENT` | `comment_status` | `ACTIVE` / `BLOCKED` / `DELETED` | 동일 |
| `USERS` | `account_status` | `ACTIVE` / `BLOCKED` / `DELETED` | 회원 탈퇴/차단 |

### 표준 쿼리 패턴

```xml
<!-- 일반 사용자 리스트: ACTIVE 만 -->
WHERE post_status = 'ACTIVE'

<!-- 어드민 리스트: DELETED 제외, BLOCKED 포함 -->
WHERE post_status != 'DELETED'

<!-- 신고 게시판 컨텍스트: 모든 상태 (DELETED 포함) -->
-- WHERE 절에 status 필터 없음
```

### 캐시 컬럼 정합성

```text
// 댓글 삭제 시
updateCommentStatus("DELETED");
decrementPostCommentCount();  // 글의 comment_count 감소

// Reconcile 스케줄러 (ADR-0006)
SELECT COUNT(*) FROM COMMUNITY_COMMENT
WHERE post_id = ? AND comment_status != 'DELETED'
```

* `comment_count` 는 ACTIVE + BLOCKED 만 포함, DELETED 는 제외
* 이렇게 하면 사용자 표시 카운트가 자연스럽게 일치

### Consequences

* Good
  * **신고/감사 컨텍스트 영구 보존** — ADR-0001 의 판단 큐가 정확히 동작
  * **복구 가능** — 어드민이 `status='ACTIVE'` 로 되돌리기 가능
  * **FK 연쇄 삭제 회피** — 글이 DELETED 여도 row 는 살아있어 FK 깨짐 없음
  * **상태 머신 일관성** — `BLOCKED`, `DELETED`, `ACTIVE` 가 같은 컬럼에서 표현
  * **이미 존재하는 패턴** — 추가 컬럼 없이 기존 status 컬럼 재사용
* Bad
  * **모든 리스트 쿼리에 `WHERE status='ACTIVE'` 필요** — 누락 시 삭제된 콘텐츠 노출 위험
  * **인덱스 부담** — status 컬럼이 인덱스에 포함돼야 효율적
  * **물리적 디스크 사용량 증가** — 삭제 콘텐츠도 row 유지
  * **GDPR / 개인정보 보호 요구사항** — 일정 기간 후 hard delete 정책 필요할 수 있음
* Neutral
  * 검색 인덱싱(향후 도입 시) 에서 DELETED 제외 로직 필요

## Pros and Cons of the Options

### Option A — Hard Delete

* Good, because 단순함, 디스크 절약
* Bad, because **컨텍스트 영구 손실** — 신고 이력/감사 불가
* Bad, because 사용자 실수 복구 불가
* Bad, because FK 연쇄 삭제 위험

### Option B — `status='DELETED'` (선택)

* Good, because 컨텍스트 보존 + 복구 가능
* Good, because 기존 status 컬럼 재사용으로 추가 비용 0
* Good, because BLOCKED 와 같은 상태 머신 안에서 일관성
* Bad, because 모든 쿼리에 필터 필요

### Option C — 아카이브 테이블 이동

* Good, because 운영 테이블 깨끗
* Bad, because **트랜잭션 비용 큼** (DELETE + INSERT)
* Bad, because 조인이 두 테이블로 분산됨
* Bad, because 아카이브 스키마 동기화 부담

### Option D — `deleted_at` 타임스탬프

* Good, because 삭제 시각 자동 기록
* Bad, because **이미 status 컬럼이 있으므로 중복** (BLOCKED 와 별도 표현 필요)
* Bad, because 현재 스키마와 자연스럽지 않음

## More Information

### 관련 ADR

* [ADR-0001](./0001-report-no-auto-user-block.md) — 신고 게시판 컨텍스트 보존이 본 패턴의 직접 동기
* [ADR-0003](./0003-blur-vs-blocked-policy.md) — 같은 status 컬럼이 BLOCKED 도 표현
* [ADR-0006](./0006-counter-cache-reconcile.md) — 캐시 컬럼이 DELETED 를 제외하도록 설계

### 코드 위치

* 게시글 삭제: `CommunityServiceImpl.deletePost()` — UPDATE `post_status='DELETED'`
* 댓글 삭제: `CommunityServiceImpl.deleteComment()` — UPDATE `comment_status='DELETED'`
* 표준 필터: 모든 리스트 쿼리에 `WHERE post_status='ACTIVE'` (또는 어드민 모드 시 `!= 'DELETED'`)
* DB 컬럼: `COMMUNITY_POST.post_status`, `COMMUNITY_COMMENT.comment_status`, `USERS.account_status`

### TODO / 향후 검토

* GDPR 준수가 필요해지면 **일정 기간 후 hard delete 정책** 추가 (예: 6개월 경과 DELETED row → physical delete)
* 어드민 콘솔에서 **DELETED 콘텐츠 복구 UI** 제공 검토

### 참고

* [Soft Delete vs Hard Delete (Martin Fowler)](https://martinfowler.com/bliki/SoftDelete.html)
