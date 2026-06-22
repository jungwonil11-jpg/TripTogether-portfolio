# ADR-0006: 캐시 컬럼 + Reconcile 스케줄러 패턴 — like/comment/report count 정합성

* Status: Accepted
* Date: 2026-04-18
* Decision-Makers: Victor Jung
* Consulted: N/A
* Informed: N/A

## Context and Problem Statement

커뮤니티 게시글의 좋아요·댓글·신고 개수는 리스트/상세 페이지에서 **매우 빈번하게 조회**됩니다. 매 조회마다 `COUNT(*)` 서브쿼리를 돌리면:

* 리스트 페이지에서 N개 글 × 3종 카운트 = N×3 서브쿼리 → 성능 급락
* `COUNT(*)` 자체가 InnoDB 에서 비싼 연산 (특히 신고 누적 글 많을 때)
* 페이지네이션·정렬(인기순) 시 매번 재계산

반면 캐시 컬럼(`like_count` 등)에 카운트를 누적 저장하면 조회 비용은 0이지만:

* 좋아요 토글 / 댓글 작성·삭제 / 신고 시점에 카운트 갱신 누락 시 정합성 깨짐
* 동시성 이슈 (두 트랜잭션이 동시 토글 시 +1 누락)
* DB 직접 조작·복구 후 카운트 어긋남

## Decision Drivers

* **조회 성능** — 리스트 페이지의 빈번한 카운트 조회 비용 최소화
* **데이터 정합성** — 캐시 컬럼이 실제 row 수와 일치해야 함
* **운영 안전망** — 카운트 누락이 발생해도 자동 복구 메커니즘 필요
* **인기순 정렬 지원** — `like_count DESC` 같은 정렬을 인덱스로 처리 가능해야 함

## Considered Options

* **Option A — 매 조회 시 `COUNT(*)` 서브쿼리**
  단순. 성능 부담 큼.
* **Option B — 캐시 컬럼만 (갱신 시점 동기화)**
  성능 좋음. 정합성 깨질 가능성.
* **Option C — 캐시 컬럼 + Reconcile 스케줄러 (선택)**
  성능 + 정합성 안전망 동시 확보.
* **Option D — Redis 캐시**
  외부 의존성 추가. 현 규모에서 오버엔지니어링.

## Decision Outcome

**Chosen option: "Option C — 캐시 컬럼 + Reconcile 스케줄러"**, because 평시에는 캐시 컬럼으로 빠른 조회를 보장하고, 주기적 reconcile 잡이 정합성 안전망 역할을 하여 운영 중 카운트 어긋남이 자동 복구됩니다.

### 캐시 컬럼 구조

| 테이블 | 컬럼 | 의미 |
|---|---|---|
| `COMMUNITY_POST` | `like_count` | 좋아요 누적 |
| `COMMUNITY_POST` | `comment_count` | 댓글 누적 (대댓글 포함) |
| `COMMUNITY_POST` | `report_count` | 신고 누적 (BLUR 임계값 판단용) |
| `COMMUNITY_COMMENT` | `like_count` | 댓글 좋아요 |
| `COMMUNITY_COMMENT` | `report_count` | 댓글 신고 |

### 갱신 시점 (실시간 동기화)

```text
// 좋아요 토글
if (alreadyLiked) {
    deleteLike();
    decrementLikeCount();  // UPDATE COMMUNITY_POST SET like_count = like_count - 1
} else {
    insertLike();
    incrementLikeCount();
}

// 댓글 작성
insertComment();
incrementCommentCount();

// 신고
insertReport();
incrementReportCount();
```

* 모든 갱신은 동일 트랜잭션(`@Transactional`) 안에서 수행
* 트랜잭션 실패 시 카운트도 롤백

### Reconcile 스케줄러 (정합성 안전망)

```text
// CommunityCacheReconcileScheduler
@Scheduled(cron = "0 0 4 * * *")  // 매일 새벽 4시
public void reconcileCounts() {
    // UPDATE COMMUNITY_POST p
    // SET like_count = (SELECT COUNT(*) FROM COMMUNITY_LIKE WHERE post_id = p.post_id),
    //     comment_count = (SELECT COUNT(*) FROM COMMUNITY_COMMENT WHERE post_id = p.post_id AND comment_status != 'DELETED'),
    //     report_count = (SELECT COUNT(*) FROM REPORT WHERE target_type='post' AND target_id = p.post_id AND status='IN_REVIEW')
}
```

* 매일 새벽 1회 전체 카운트 재계산
* 실시간 갱신 누락이 있어도 24시간 내 자동 복구
* DB 직접 조작 후에도 다음날 자동 정합성 회복

### Consequences

* Good
  * **조회 성능 극대화** — 리스트 페이지에서 카운트 서브쿼리 0개
  * **인기순 정렬 인덱스 활용 가능** — `ORDER BY like_count DESC` + 인덱스
  * **자동 복구 메커니즘** — 실시간 갱신 누락이 운영 중 자동 회복
  * **DB 직접 조작 친화** — 운영자가 SQL 로 데이터 수정해도 다음날 정합성 회복
* Bad
  * 갱신 코드를 빠뜨리면 그날 하루는 카운트 어긋남 (다음 reconcile 까지)
  * 동시 토글 시 race condition 가능성 (실무상 좋아요 동시 토글 빈도는 매우 낮음)
  * 스케줄러 구동 비용 (전체 row 재계산이라 데이터 많아지면 무거워짐)
* Neutral
  * 캐시 컬럼이 인덱스 대상이 되면 갱신 비용 증가 (현재는 인기순 정렬용 인덱스 한정)

## Pros and Cons of the Options

### Option A — `COUNT(*)` 서브쿼리

* Good, because 항상 정확
* Bad, because 리스트 페이지 성능 급락
* Bad, because 인기순 정렬에 인덱스 활용 불가

### Option B — 캐시 컬럼만

* Good, because 조회 빠름
* Bad, because **갱신 누락 시 정합성 영구 깨짐**
* Bad, because 운영 중 복구 수단 없음

### Option C — 캐시 + Reconcile (선택)

* Good, because B 의 성능 + 정합성 안전망
* Good, because 운영 안정성 높음
* Bad, because 스케줄러 추가 코드 부담

### Option D — Redis

* Good, because 매우 빠름
* Good, because 분산 환경 친화
* Bad, because 외부 의존성 추가 비용 큼
* Bad, because 현 규모에서 오버엔지니어링

## More Information

### 관련 ADR

* [ADR-0001](./0001-report-no-auto-user-block.md) — `report_count` 가 BLUR 임계값 판단의 기준
* [ADR-0008](./0008-soft-delete-pattern.md) — 댓글 카운트가 soft delete 상태 고려해야 함

### 코드 위치

* 갱신 메서드: `CommunityServiceImpl` `updatePostLikeCount()` / `updatePostCommentCount()` / `updatePostReportCache()` 등
* Reconcile 스케줄러: `src/main/java/org/triptogether/community/service/CommunityCacheReconcileScheduler.java`
* DB 컬럼: `COMMUNITY_POST.like_count` / `comment_count` / `report_count`

### 면접 어필 포인트

이 패턴은 **CAP 관점에서 Eventually Consistent** 한 설계입니다. 실시간 강한 일관성 대신 24시간 내 정합성 회복을 보장함으로써 평시 성능을 극대화하고 운영 안정성을 동시에 확보합니다. 같은 패턴이 Twitter/Facebook 의 follower count, GitHub 의 star count 등에서도 사용됩니다.
