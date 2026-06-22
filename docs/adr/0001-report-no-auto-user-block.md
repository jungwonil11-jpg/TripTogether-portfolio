# ADR-0001: 신고 누적 자동 제재 범위 — BLUR 까지, 그 외 어드민 수동 판단

* Status: Accepted
* Date: 2026-04-23
* Decision-Makers: Victor Jung
* Consulted: N/A
* Informed: N/A

## Context and Problem Statement

커뮤니티 게시글/댓글에 대한 신고가 누적될 때, 시스템이 어디까지 자동으로 제재하고 어디부터 인간(어드민) 판단을 거칠지 정해야 했습니다.

일반적인 신고 시스템의 "교과서 패턴" 은 *N회 누적 → 자동 차단* 으로, 신고 횟수를 단일 시그널로 사용해 콘텐츠 삭제·유저 차단까지 자동화합니다. 그러나 실제 사용자 행동을 보면 단순 불편·취향 차이만으로도 신고가 발생하기 때문에, 자동 차단은 false positive(오신고로 인한 부당 제재) 비용이 매우 큽니다.

따라서 자동화의 경계를 어디에 그을지가 결정 사안이었습니다.

## Decision Drivers

* **False positive 비용 최소화** — 부당하게 차단된 사용자/콘텐츠는 신뢰도 회복이 어려움
* **Trust & Safety 분야의 일반 원칙** — 결정적 액션(삭제/차단)은 인간 판단이 표준
* **어드민 운영 현실성** — 신고 게시판이 어드민 판단 큐 역할로 동작해야 함
* **사용자 신뢰 보호** — 단순 불편 신고로 콘텐츠가 사라지면 사용자 이탈 위험

## Considered Options

* **Option A — 신고 N회 누적 시 콘텐츠 삭제 + 유저 자동 차단**
  교과서 패턴. 운영 부담은 적지만 false positive 비용 큼.
* **Option B — 신고 N회 누적 시 BLUR 처리 + 어드민 수동 판단 큐로 전송 (본 결정)**
  자동화는 약한 신호(BLUR)까지, 결정적 액션은 인간이 판단.
* **Option C — 모든 신고를 즉시 어드민에게만 알리고 자동 처리 없음**
  과한 어드민 부담. 명백한 악성 콘텐츠도 즉시 가려지지 않아 다른 사용자 노출 위험.

## Decision Outcome

**Chosen option: "Option B — BLUR 자동 처리 + 어드민 수동 판단"**, because Trust & Safety 분야의 표준 패턴인 **Human-in-the-Loop Moderation** 과 정확히 일치하며, false positive 비용과 운영 부담의 균형을 맞춥니다.

### 자동화 범위 (구체화)

| 트리거 | 자동 처리 | 어드민 수동 판단 영역 |
|---|---|---|
| 신고 1~2회 | `report_count` 증가만 | — |
| 신고 3회 누적 | **콘텐츠 BLUR 처리** + 신고 게시판 INSERT | BLUR 유지 / 해제 / 글 삭제 / 유저 차단 / 기각 |
| AI 독성 감지 (Perspective) | BLUR 처리 + 신고 게시판 INSERT | 동일 |

### 신고 게시판의 역할

* **어드민의 판단 큐 (Admin Decision Queue)** 로 동작
* 신고 1건이라도 들어오면 무조건 신고 게시판에 INSERT
* BLUR 처리된 글/댓글도 모두 신고 게시판에 누적
* 어드민이 각 건마다 5가지 액션 중 직접 결정:
  1. **BLUR 유지** (관망)
  2. **BLUR 해제** (오신고 판단, `clearPostBlur`)
  3. **글 삭제** (`status='DELETED'`)
  4. **유저 차단** (`account_status='BLOCKED'`)
  5. **기각** (`DISMISSED`)

### Consequences

* Good
  * 오신고로 인한 부당 제재 위험 제거
  * 어드민이 신고 컨텍스트(이유/설명/누적 횟수)를 보고 판단 가능
  * 명백한 악성 콘텐츠는 BLUR 로 즉시 다른 사용자 노출 차단
  * Trust & Safety 표준 패턴 적용으로 확장성 확보 (예: 향후 신뢰도 점수 도입 시 자연스럽게 결합)
* Bad
  * 어드민 운영 부담 (큐 누적 시 처리 지연 가능)
  * 자동 차단 시스템보다 구현 단순도는 낮음 (어드민 UI 도구 필요)
* Neutral
  * AI 코드 리뷰 도구가 *"자동 차단 미구현"* 을 결함으로 잘못 분류할 수 있음 → 본 ADR 로 보완

## Pros and Cons of the Options

### Option A — 자동 차단

* Good, because 어드민 부담이 거의 없음
* Good, because 구현 단순함
* Bad, because **false positive 비용이 매우 큼** — 부당 차단된 사용자/콘텐츠 회복 어려움
* Bad, because 단순 불편 신고로 정상 콘텐츠가 자동 삭제될 수 있음
* Bad, because 신고 도배(여러 계정 동원) 공격에 취약 — *brigading* 위험

### Option B — BLUR + 수동 판단 (선택)

* Good, because **false positive 비용을 인간 검토로 흡수**
* Good, because Reddit AutoModerator, Discord AutoMod, YouTube Trust & Safety, Stack Overflow 모더레이션 큐 등 업계 표준 패턴과 일치
* Good, because BLUR + 클릭 펼침 UX 로 점진적 공개 (사용자 판단 존중)
* Bad, because 어드민 운영 부담 발생
* Bad, because 어드민 판단 도구(UI) 가 별도로 잘 만들어져 있어야 함

### Option C — 자동 처리 없음

* Good, because false positive 위험 0
* Bad, because 명백한 악성 콘텐츠도 어드민 처리 전까지 노출됨
* Bad, because 어드민 부담 과중

## More Information

### 관련 ADR

* [ADR-0003](./0003-blur-vs-blocked-policy.md) — BLUR vs BLOCKED 분기 정책 (자동 BLUR 와 어드민 BLOCKED 의 표시 차이)
* [ADR-0004](./0004-duplicate-report-prevention.md) — 중복 신고 방지 (신고 누적 카운트의 신뢰성 보장)

### 업계 레퍼런스

* **Trust & Safety: Human-in-the-Loop Moderation** — 자동화는 약한 신호 처리, 결정적 액션은 인간 판단
* Reddit AutoModerator — 자동 가림 + 모더레이터 수동 처리
* Discord AutoMod — 자동 플래그 + 모더레이터 큐
* YouTube — 자동 감지 + Trust & Safety 팀 인간 검토
* Stack Overflow — 자동 플래그 + 커뮤니티 모더레이터 큐

### 코드 위치

* 신고 누적 카운트: `src/main/java/org/triptogether/community/service/CommunityServiceImpl.java` `updatePostReportCache()` / `updateCommentReportCache()`
* BLUR 렌더링 조건: `report_count >= 3 AND !isAdminMode` (post_status 조건 없음)
* 어드민 액션: `blockPost()` / `clearPostBlur()` / `bulkAction()` 등
* 신고 INSERT: `src/main/java/org/triptogether/report/service/ReportServiceImpl.java` `submitReport()`
* DB 스키마: `REPORT` 테이블, `COMMUNITY_POST.report_count` / `post_status` 컬럼
