# ADR-0005: XSS 방지 — jsoup Safelist 서버측 sanitize

* Status: Accepted
* Date: 2026-04-24
* Decision-Makers: Victor Jung
* Consulted: N/A
* Informed: N/A

## Context and Problem Statement

ADR-0002 에서 Summernote WYSIWYG 에디터를 채택하면서 사용자 입력이 **HTML 그대로** 서버에 저장되는 구조가 됐습니다. 이는 XSS(Cross-Site Scripting) 공격에 직접 노출되는 환경입니다:

* `<script>alert(1)</script>` 같은 직접 스크립트 삽입
* `<img src=x onerror=...>` 같은 이벤트 핸들러 인젝션
* `<iframe src="javascript:...">` URL 스킴 공격
* `<a href="javascript:...">` 링크 공격

클라이언트(Summernote) 가 일부 필터링을 하더라도 우회 가능하므로 **서버측 방어가 필수**였습니다.

## Decision Drivers

* **신뢰 경계 원칙** — 클라이언트는 신뢰할 수 없음. 서버에서 반드시 검증/정화
* **알려진 검증된 라이브러리 사용** — XSS 정화 직접 구현은 위험 (놓치는 케이스 다수)
* **콘텐츠 표현력 보존** — 단순 escape 로 모든 HTML 태그 제거하면 WYSIWYG 의미 사라짐
* **저장 시점 정화 vs 출력 시점 정화** — 저장 시 정화하면 출력 비용/누락 위험 절감

## Considered Options

* **Option A — 클라이언트 측 정화만 (Summernote 기본 + 직접 JS 필터)**
  서버는 신뢰. 우회 위험 큼.
* **Option B — 출력 시점 escape (`<c:out>` / `fn:escapeXml`) 만**
  모든 태그 제거. WYSIWYG 의미 상실.
* **Option C — 저장 시점 jsoup Safelist 화이트리스트 정화 (선택)**
  허용 태그/속성만 통과시키고 나머지 제거.
* **Option D — OWASP Java HTML Sanitizer**
  Google 제공. 강력하지만 의존성 더 무거움.

## Decision Outcome

**Chosen option: "Option C — jsoup Safelist 서버측 정화"**, because jsoup 은 이미 인라인 이미지 orphan 정리 스케줄러에서 사용 중이라 추가 의존성이 없고, Safelist API 가 직관적이며, **저장 시점에 한 번만 정화하면 출력 시 누락 위험이 없습니다**.

### 정화 정책

```text
// CommunityServiceImpl.java L48
private static final Safelist COMMUNITY_SAFELIST = Safelist.basicWithImages()
        .addTags(...)        // 추가 허용 태그
        .addAttributes(...)  // 추가 허용 속성
        ;

private String sanitizeHtml(String html) {
    return Jsoup.clean(html, COMMUNITY_SAFELIST);
}
```

* `Safelist.basicWithImages()` 기반 (a, b, blockquote, br, em, i, img, li, ol, p, strong, sub, sup, u, ul + 기본 이미지 속성)
* 위험 태그(`<script>`, `<iframe>`, `<object>`, `<embed>`, `<style>`) 자동 제거
* 위험 속성(`onerror`, `onclick` 등 모든 on*) 자동 제거
* `javascript:` URL 스킴 자동 차단

### 적용 시점

| 시점 | 메서드 | 동작 |
|---|---|---|
| 글 작성 | `CommunityServiceImpl.writePost()` L246 | INSERT 전 `sanitizeHtml()` |
| 글 수정 | `CommunityServiceImpl.editPost()` L339 | UPDATE 전 `sanitizeHtml()` |
| 댓글 | (P0 보완 항목 — 현재 미적용) | TODO |

### Consequences

* Good
  * 알려진 XSS 벡터 거의 모두 차단
  * Summernote 표현력(굵게/이미지/링크 등) 보존
  * 저장 시점 정화로 출력 누락 위험 없음
  * jsoup 은 이미 사용 중이라 추가 의존성 없음
  * Safelist 화이트리스트 방식이라 새 공격 벡터에도 자동 안전 (블랙리스트보다 강함)
* Bad
  * **댓글 본문은 현재 미적용** — 일관성 부족 (P0 보완 필요)
  * 사용자가 입력한 HTML 이 정화로 일부 사라질 수 있음 (예: 잘못된 표 구조)
  * 정화된 HTML 이 원본과 100% 동일하지 않을 수 있음 (정상 동작이지만 디버깅 시 혼란 가능)
* Neutral
  * 정화 비용은 글 작성/수정 시점에만 발생 (조회 시점 0 비용)

## Pros and Cons of the Options

### Option A — 클라이언트 정화만

* Good, because 서버 부담 0
* Bad, because **클라이언트 우회 가능** (Postman/curl 직접 호출)
* Bad, because 신뢰 경계 원칙 위반

### Option B — 출력 시 escape 만

* Good, because 단순
* Bad, because **모든 HTML 태그 제거 → WYSIWYG 의미 상실**
* Bad, because Summernote 채택 의도와 충돌

### Option C — jsoup Safelist (선택)

* Good, because 화이트리스트 방식 (블랙리스트보다 안전)
* Good, because jsoup 이미 사용 중 → 추가 의존성 없음
* Good, because Safelist API 직관적
* Bad, because 댓글 등 일부 영역 적용 누락 위험 (코드 리뷰로 보완)

### Option D — OWASP Java HTML Sanitizer

* Good, because Google 백킹, 강력함
* Good, because 정책 표현력 더 풍부
* Bad, because 추가 의존성 증가 (jsoup 으로 충분한 상황)
* Bad, because 학습 곡선 약간 높음

## More Information

### 관련 ADR

* [ADR-0002](./0002-summernote-editor.md) — Summernote HTML 출력이 본 정화 정책의 직접 원인

### 코드 위치

* Safelist 정의: `src/main/java/org/triptogether/community/service/CommunityServiceImpl.java` L48
* sanitize 메서드: `CommunityServiceImpl.sanitizeHtml()` L60
* 적용 위치: `writePost()` L246, `editPost()` L339
* 의존성: `pom.xml` `jsoup 1.17.2`

### TODO (P0 보완)

* 댓글 작성/수정 시 `sanitizeHtml()` 적용 — `CommunityServiceImpl.addComment()` / `addReply()`
* 인라인 이미지 업로드(`uploadInlineImage()`) 의 파일 타입/크기 검증

### 참고

* [jsoup Safelist API](https://jsoup.org/apidocs/org/jsoup/safety/Safelist.html)
* [OWASP XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
