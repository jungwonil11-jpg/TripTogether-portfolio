# ADR-0003: BLUR vs BLOCKED 분기 정책 — 신고 누적은 점진적 공개, 어드민 차단은 완전 숨김

* Status: Accepted
* Date: 2026-04-20
* Decision-Makers: Victor Jung
* Consulted: N/A
* Informed: N/A

## Context and Problem Statement

ADR-0001 에서 신고 누적 시 자동 처리는 BLUR 까지로 정했고, 어드민이 직접 차단을 결정하는 경로도 별도로 존재합니다. 이때 두 경로의 **표시 방식과 상태 표현이 어떻게 달라야 하는지** 가 결정 사안이었습니다.

초기 설계에서는 *"신고 3회 → BLOCKED 전환 + BLUR 표시"* 로 잘못 합쳐져 있었으나, BLOCKED 가 되면 자동으로 blind 되어야 하므로 BLUR 오버레이라는 개념 자체가 성립하지 않는 모순이 있었습니다. 두 경로를 명확히 분리할 필요가 있었습니다.

## Decision Drivers

* **자동 처리(BLUR)와 어드민 처리(BLOCKED)의 의미 분리** — 두 경로가 다른 신뢰 신호를 표현
* **사용자 판단 존중** — 신고 누적은 약한 신호이므로 완전 숨김보다 점진적 공개가 적절
* **어드민 결정의 명확성** — 어드민이 직접 차단한 콘텐츠는 완전 숨김이 의도
* **상태 표현 단순성** — `post_status` 와 `report_count` 두 컬럼 조합으로 충분히 표현

## Considered Options

* **Option A — 신고 3회 누적 시 status=BLOCKED 전환 + BLUR 표시**
  초기 설계. BLOCKED 의미가 모순됨.
* **Option B — 신고 3회 누적은 status=ACTIVE 유지 + BLUR 오버레이만, BLOCKED 는 어드민 차단 전용 (선택)**
  두 경로 명확히 분리.
* **Option C — 신고 누적도 즉시 완전 숨김 (BLOCKED 와 동일 처리)**
  점진적 공개 UX 포기.

## Decision Outcome

**Chosen option: "Option B"**, because 자동 처리(약한 신호)와 어드민 처리(강한 결정)는 표시 방식도 달라야 하며, BLUR 오버레이는 사용자 판단 존중과 점진적 공개라는 UX 철학에 부합합니다.

### 상태 매트릭스

| 상황 | `post_status` | `report_count` | 일반 사용자 표시 | 어드민 모드 표시 |
|---|---|---|---|---|
| 정상 | `ACTIVE` | < 3 | 정상 노출 | 정상 노출 |
| 신고 3회 누적 (자동) | **`ACTIVE` (유지)** | >= 3 | **BLUR 오버레이** ("⚠️ 신고된 콘텐츠입니다. 클릭하여 확인") | 정상 노출 + 신고 배지 |
| 어드민 직접 차단 | **`BLOCKED`** | (무관) | **완전 숨김** (리스트에서 제거) | 차단 표시 + 해제 버튼 |
| 작성자 삭제 | `DELETED` | (무관) | 완전 숨김 | 완전 숨김 (감사 로그에만 존재) |

### BLUR 렌더링 조건

```
report_count >= 3  AND  !isAdminMode
```

`post_status` 조건 없음. 즉 `ACTIVE + report_count>=3` 만으로 BLUR 결정.

### 사용자 인터랙션 (BLUR 케이스)

1. 리스트/상세 페이지에서 본문이 흐림 처리 + 오버레이 텍스트
2. 사용자 클릭 시 블러 벗겨지고 본문 공개
3. 사용자가 자기 책임 하에 콘텐츠를 볼지 결정

### Consequences

* Good
  * 자동 처리(BLUR)와 어드민 처리(BLOCKED)의 의미가 명확히 분리됨
  * 사용자 판단 존중 — 신고가 부정확할 수 있음을 UX 로 표현
  * 어드민 직접 차단은 강한 결정 신호로 완전 숨김과 일치
  * 상태 머신이 단순 (4가지 표시 케이스만)
* Bad
  * `post_status` 와 `report_count` 두 컬럼을 함께 봐야 표시 결정 가능 (단일 컬럼 분기 불가)
  * BLUR 렌더링 조건을 잘못 쓰면(`post_status='BLOCKED' AND report_count>=3` 등) 정책 위반 — 코드 리뷰 시 주의 필요
* Neutral
  * 리스트 쿼리에 `OR (post_status='BLOCKED' AND report_count>=3)` 같은 dead branch 가 남아 있을 수 있음 (기능 영향 없으나 의미 없음)

## Pros and Cons of the Options

### Option A — BLOCKED + BLUR 합침

* Good, because 단일 컬럼(`post_status`) 분기로 표시 결정 가능
* Bad, because **의미 모순** — BLOCKED 면 blind 되어야 하므로 BLUR 오버레이 개념 성립 안 됨
* Bad, because 자동/수동 처리 구분이 사라짐

### Option B — ACTIVE 유지 + BLUR (선택)

* Good, because 자동/수동 처리 의미 분리
* Good, because 점진적 공개 UX 가능
* Good, because 어드민 차단의 강한 결정성 보존
* Bad, because 두 컬럼을 함께 봐야 함 (단일 분기 불가)

### Option C — 신고 누적도 즉시 완전 숨김

* Good, because 단순함 (한 가지 처리 방식)
* Bad, because false positive 비용 큼 — 부당하게 가려진 콘텐츠 회복 어려움
* Bad, because 사용자 판단 기회 박탈

## More Information

### 관련 ADR

* [ADR-0001](./0001-report-no-auto-user-block.md) — 신고 누적 자동 제재 BLUR 까지만
* [ADR-0008](./0008-soft-delete-pattern.md) — `status` 컬럼 활용한 상태 관리

### 코드 위치

* BLUR 렌더링 조건: 리스트/상세 JSP 의 `${reportCount >= 3 && !isAdminMode}` 분기
* CSS: `report-blurred` / `report-blurred-overlay` (커뮤니티 모듈)
* 신고 카운트 갱신: `CommunityServiceImpl.updatePostReportCache()` / `updateCommentReportCache()` (block 호출 안 함)
* 어드민 직접 차단: `blockPost()` / `blockComment()` — `post_status='BLOCKED'` 로 전환
* BLUR 해제 (어드민 오신고 판정): `clearPostBlur()` — `report_count` 리셋

### 정책 변경 이력

* 2026-04-20: 초기 잘못된 설계(BLOCKED + BLUR 합침) → 현재 정책(ACTIVE + BLUR / BLOCKED 완전 숨김)으로 정정
