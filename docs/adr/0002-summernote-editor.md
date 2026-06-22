# ADR-0002: 커뮤니티 글쓰기 WYSIWYG — Summernote 채택

* Status: Accepted
* Date: 2026-04-23
* Decision-Makers: Victor Jung
* Consulted: N/A
* Informed: N/A

## Context and Problem Statement

커뮤니티 글쓰기 페이지에서 사용자(특히 후기/사진 글 작성자)가 본문에 텍스트와 이미지를 자연스럽게 섞을 수 있는 WYSIWYG 에디터가 필요했습니다. 단순 textarea + 별도 이미지 업로드 폼 조합은 다음 문제가 있었습니다:

* 본문에 이미지를 원하는 위치에 삽입 불가
* 사진 후기 글의 UX 한계
* 글 본문과 이미지의 분리된 관리

스택은 JSP + jQuery 기반이며 npm/번들러 파이프라인이 없는 상태였습니다. 또한 이미지는 Phase 1(base64 임시) → Phase 2(Cloudinary 업로드) 단계적 도입 계획이 있었습니다.

## Decision Drivers

* **JSP + jQuery 스택 적합성** — 빌드 파이프라인 없이 CDN 한 줄 도입이 가능한가
* **인라인 이미지 후크 단순성** — `onImageUpload` 같은 명확한 확장점
* **한국어 lang 팩 공식 지원** — 1순위 사용자 언어
* **단계적 도입 친화** — Phase 1/2 전환이 코드 한 군데 수정으로 가능한지
* **HTML 출력 제어** — photo 유형 `<img>` 개수 검증 등 비즈니스 룰 적용 용이성
* **유지보수 부담 최소화** — 추가 학습 비용/번들러 도입 비용 회피

## Considered Options

* **Option A — Summernote (선택)**
  jQuery + Bootstrap 기반 WYSIWYG. CDN 한 줄 도입.
* **Option B — Toast UI Editor**
  NHN 한국산. 마크다운+WYSIWYG 토글.
* **Option C — Quill**
  자체 Delta 포맷 출력. 가벼움.
* **Option D — TipTap**
  ProseMirror 기반 모던 에디터. React/Vue 친화.
* **Option E — CKEditor 5**
  엔터프라이즈급. 빌더 사이트 빌드 산출물 사용.

## Decision Outcome

**Chosen option: "Option A — Summernote"**, because JSP+jQuery 스택에 가장 적합하고 (이미 로드되는 jQuery 위에 CDN 한 줄), 인라인 이미지 후크가 단순해 Phase 1→2 전환 비용이 낮으며, 한국어 lang 팩이 공식 지원되어 사용자 UX 검증이 빠르기 때문입니다.

### Consequences

* Good
  * CDN 한 줄로 즉시 도입, 빌드 파이프라인 변경 없음
  * `onImageUpload` 후크 한 곳만 갈아끼우면 base64 → Cloudinary 전환 가능
  * 본문이 HTML 그대로라 photo 유형 `<img>` 개수 검증 같은 비즈니스 룰 적용 단순
  * 한국어 lang 팩 공식 지원
  * 풀스크린·코드뷰 등 부가 기능 기본 제공
* Bad
  * **HTML 출력으로 인한 XSS 위험** → ADR-0005 의 jsoup Safelist sanitize 로 보완
  * jQuery 의존 + ~140KB 번들 (이미 jQuery 로딩되므로 추가 비용은 적음)
  * Summernote 0.9.1 은 유지보수 정체 — 장기적으로 마이그레이션 검토 필요
  * 모바일 UX 가 모던 에디터(TipTap 등) 대비 약함
* Neutral
  * 본문 HTML 저장으로 검색·요약·번역 시 plainText 변환 레이어 필요 (서버에서 jsoup 으로 처리 가능)

## Pros and Cons of the Options

### Option A — Summernote

* Good, because JSP+jQuery 스택에 천생연분 (CDN 한 줄)
* Good, because `onImageUpload` 후크가 단순해 외부 스토리지 연동 용이
* Good, because 한국어 lang 팩 공식 지원
* Good, because 본문 HTML 직접 노출 → 비즈니스 룰 적용 쉬움
* Bad, because 유지보수 정체, 모바일 UX 약함

### Option B — Toast UI Editor

* Good, because 한국산이라 한국어 UX 가장 자연스러움
* Good, because 마크다운+WYSIWYG 토글로 파워유저 편의성
* Bad, because 마크다운 토글이 일반 사용자(후기 작성자)에게는 혼란 유발
* Bad, because 번들 크기 더 큼 (~250KB)

### Option C — Quill

* Good, because 가볍고 API 깔끔
* Bad, because **출력이 자체 Delta 포맷** — 서버 HTML 저장 시 변환 레이어 필요
* Bad, because 한국어 lang 팩 비공식
* Bad, because 인라인 이미지 핸들러 직접 작성

### Option D — TipTap

* Good, because 모던 에디터 중 가장 확장성 강함 (ProseMirror 기반)
* Bad, because **JSP+jQuery 환경에 부적합** — 사실상 npm + React/Vue 전제
* Bad, because 익스텐션 조립식이라 단순 도입에는 오버엔지니어링

### Option E — CKEditor 5

* Good, because 엔터프라이즈급 안정성과 접근성
* Bad, because 빌더 사이트 빌드 산출물 워크플로우가 CDN 방식과 어색
* Bad, because 협업 편집 등 핵심 기능 유료
* Bad, because 학습곡선 있음

## More Information

### 관련 ADR

* [ADR-0005](./0005-xss-server-sanitize.md) — XSS 방지 jsoup Safelist (Summernote HTML 출력 보완)
* [ADR-0007](./0007-cloudinary-image-storage.md) — 인라인 이미지 Cloudinary 업로드 (Phase 2)

### 코드 위치

* 에디터 초기화: `src/main/webapp/WEB-INF/views/community/write.jsp` L660~723
* CDN 로드: write.jsp L31~35 (`summernote-lite@0.9.1` + jQuery 3.7.1 + ko-KR lang)
* 인라인 이미지 후크: write.jsp `onImageUpload` 콜백 → `POST /community/inline-image`
* 서버 sanitize: `CommunityServiceImpl.sanitizeHtml()` (L60)

### 참고

* [Summernote 공식](https://summernote.org/)
* [Toast UI Editor](https://ui.toast.com/tui-editor)
* [Quill](https://quilljs.com/)
* [TipTap](https://tiptap.dev/)
