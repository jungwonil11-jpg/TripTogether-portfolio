# ADR-0010: AI 모더레이션 풀 스택 파이프라인 — Perspective + ai_flagged + JSP BLUR

* Status: Accepted
* Date: 2026-04-22
* Decision-Makers: Victor Jung
* Consulted: N/A
* Informed: N/A

## Context and Problem Statement

커뮤니티 게시글·댓글, 문의 본문 등 사용자 입력에 대해 **AI 기반 독성 감지 + 자동 가림(BLUR)** 을 구현해야 했습니다. 요구사항:

* 사용자 작성 시점에 동기 체크하면 응답 지연
* 거짓 양성(false positive)이 있을 수 있어 어드민 판단 여지 필요
* 일반 사용자에게는 가려서 보여주되, 어드민에게는 원본 노출 + 해제 도구 제공

이 구조는 단일 레이어로 구현 불가능하며, **API 호출 / DB 컬럼 / JSP 렌더링 / 어드민 도구** 가 함께 동작해야 합니다.

## Decision Drivers

* **응답 지연 회피** — AI 호출은 비동기로
* **사용자 판단 존중** — 자동 차단(blind) 보다 점진적 공개(BLUR) 우선
* **어드민 결정권 보존** — false positive 시 BLUR 해제 가능
* **레이어 간 일관성** — DB 플래그 ↔ JSP 렌더링 ↔ JS 액션이 같은 신호로 동작

## Considered Options

* **Option A — 동기 호출 + 자동 차단**
  사용자 작성 시 Perspective 호출 결과로 즉시 차단/통과. 응답 지연 + false positive 비용 큼.
* **Option B — 비동기 호출 + 자동 차단**
  비동기 처리는 OK. 그러나 자동 차단의 false positive 비용 여전.
* **Option C — 비동기 호출 + ai_flagged 컬럼 + JSP BLUR + 어드민 해제 (선택)**
  비동기로 응답 지연 회피 + BLUR 점진적 공개 + 어드민 수동 결정권.

## Decision Outcome

**Chosen option: "Option C — 풀 스택 파이프라인"**, because ADR-0001 의 Human-in-the-Loop Moderation 패턴을 AI 시그널에도 일관되게 적용하며, 응답 지연과 false positive 비용을 동시에 회피합니다.

### 풀 스택 구조

```
[사용자 작성 / 컨트롤러]
   ↓
[글 INSERT 후 PerspectiveService 비동기 호출]
   ↓ (Perspective 응답 후 비동기 콜백)
[setInquiryAiFlagged / setPostAiFlagged → ai_flagged=1]
   ↓
[다음 페이지 로드 시 SELECT 결과에 ai_flagged 포함]
   ↓ ┬ [일반 사용자] → JSP isBlurred=true → BLUR 오버레이 + 펼침
     └ [어드민] → JSP 어드민 배지 + clearBlur 버튼
   ↓ (어드민 클릭 시)
[clearInquiryBlur / clearCommentBlur → ai_flagged=0]
```

### 모듈별 동등 구현

| 모듈 | 컬럼 | 서비스 메서드 | JSP 분기 |
|---|---|---|---|
| 커뮤니티 게시글 | `community_post.ai_flagged` | `flagPostAsToxic` / `clearPostBlur` | `report-blurred-wrap` |
| 커뮤니티 댓글 | `community_comment.ai_flagged` | `flagCommentAsToxic` / `clearCommentBlur` | `report-blurred-wrap` |
| 문의 | `inquiry_post.ai_flagged` | `flagInquiryAsToxic` / `clearInquiryBlur` | `inq-title-blurred` |

→ 세 모듈이 **동일한 패턴** 으로 구현되어 있어 학습 곡선이 낮음.

### Consequences

* Good
  * 사용자 작성 응답 지연 0 (비동기)
  * Human-in-the-Loop 일관성 (자동 차단 회피)
  * 어드민이 false positive 즉시 정정 가능
  * 모듈 간 동등 구현으로 코드 재사용성↑
* Bad
  * 코드만 보고는 풀 스택 흐름 안 보임 — JSP/JS 까지 봐야 이해 가능
  * **AI 코드 분석이 "JSP/JS 미확인 → BLUR UI 미구현" 으로 잘못 분류할 위험** → 본 ADR 로 보완
  * 비동기 호출 실패 시 ai_flagged 갱신 안 됨 (재시도 정책 별도 필요)
* Neutral
  * Perspective API 비용은 호출량에 비례

## Pros and Cons of the Options

### Option A — 동기 + 자동 차단

* Good, because 즉각 반영
* Bad, because 사용자 응답 지연 (Perspective API 호출 동기 대기)
* Bad, because false positive 자동 차단 (정상 콘텐츠 차단 위험)

### Option B — 비동기 + 자동 차단

* Good, because 응답 지연 해소
* Bad, because false positive 자동 차단 위험 여전

### Option C — 비동기 + ai_flagged + BLUR + 어드민 해제 (선택)

* Good, because Human-in-the-Loop 일관성
* Good, because 응답 지연 0 + false positive 인간 검토
* Good, because 모듈 간 동등 패턴
* Bad, because 풀 스택이라 AI 분석 시 이해 어려움 → ADR 로 보완

## More Information

### 관련 ADR

* [ADR-0001](./0001-report-no-auto-user-block.md) — Human-in-the-Loop Moderation 의 일관 적용
* [ADR-0003](./0003-blur-vs-blocked-policy.md) — BLUR vs BLOCKED 분기 (AI BLUR 와 어드민 BLOCKED 의 표시 차이)
* [ADR-0009](./0009-moderation-policy-externalization.md) — 임계값 정책 외부화

### 코드 위치 (문의 모듈 기준)

* Perspective 호출: `InquiryController` 가 `PerspectiveService` 비동기 호출
* DB 컬럼: `INQUIRY_POST.ai_flagged` (0=정상, 1=감지됨)
* 서비스: `InquiryServiceImpl.flagInquiryAsToxic()` / `clearInquiryBlur()`
* Mapper: `InquiryMapper.setInquiryAiFlagged()` / `clearInquiryBlur()`
* JSP 일반 사용자 BLUR: `list.jsp` L153 `<c:set var="isBlurred" value="${inq.aiFlagged and !isAdmin}"/>` + L207 `<div class="inq-title-blur-overlay">`
* JSP 어드민 배지 + 해제 버튼: `list.jsp` L198~204, `detail.jsp` L116~121
* JS 비동기 해제: `list.jsp` L330, `detail.jsp` L497

### 면접 어필 포인트

이 패턴은 **AI 신호를 자동 차단에 직접 연결하지 않고 "약한 시그널"로 다룬다는 설계 철학**의 적용 사례입니다. AI 의 false positive 비용을 인간 검토로 흡수하면서 응답 지연도 회피합니다. AI 코드 분석 도구가 "JSP 까지 보지 못해 미구현으로 잘못 분류" 하기 쉽지만, 실제로는 DB → 서비스 → JSP → JS 까지 일관되게 동작합니다.
