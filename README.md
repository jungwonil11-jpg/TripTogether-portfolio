# TripTogether ✈️

여행 커뮤니티 · AI 플래너 통합 플랫폼 — 6인 팀 프로젝트 (2026.04 ~ 2026.05, 857 commits)

여행 코스를 짜고, 후기·팁·사진을 공유하고, AI 챗봇/어시스턴트로 여행 계획을
보조받는 웹 서비스. 사용자 서비스 + 관리자/슈퍼관리자 백오피스로 구성.

## 🎬 시연 영상

담당 영역 **커뮤니티 · 신고 · 문의** 전체 흐름을 직접 시연한 영상입니다 (약 7분).

<!-- ▼ 아래 한 줄을 지우고 그 자리에 triptogether-demo.mp4 파일을 드래그&드롭하세요.
     GitHub가 https://github.com/jungwonil11-jpg/TripTogether-portfolio/assets/... 플레이어 링크로 자동 변환합니다. ▼ -->
> 🎬 **영상 업로드 예정** — 이 자리에 `triptogether-demo.mp4` 드래그&드롭

> **포트폴리오 안내** — 코드 저장소는 팀 협의에 따라 private으로 유지하고 있습니다.
> 전체 코드와 857개 커밋 히스토리(본인 304 commits)는 **면접 시 즉시 공개·시연
> 가능**합니다. 본 저장소는 프로젝트 소개와 본인 기여를 문서로 정리한 포트폴리오입니다.

## 기술 스택

| 영역 | 기술 |
|---|---|
| Backend | Spring Boot 4.x · Java 21 · MyBatis · Maven(WAR) |
| Frontend | JSP/JSTL · jQuery · Summernote · Chart.js |
| Database | MySQL 8 (AWS EC2) |
| AI | OpenAI GPT(플래너) · Google Gemini(챗봇) · Claude(문의 답변 초안) · Perspective(독성 감지) |
| External | Cloudinary · Toss Payments · 카카오/네이버/구글 OAuth · Google Maps · SMTP |
| 기반 | SSE 실시간 알림 · i18n 4개 언어(ko/en/ja/zh) · jsoup XSS sanitize · AOP 권한 체크 |

## 도메인 구성 (22개 모듈)

```
community  inquiry  report  moderation   ← 커뮤니티/신고/문의 (담당)
common(챗봇·알림·헤더/푸터)  assistant  ai  ← AI 기능
admin  superAdmin                        ← 백오피스
auth  myPage  explore  courses  detail  travelPackage
shop  reward  flight  home  cloudinary  perspective  config
```

## 나의 역할 — 304 commits

> 표기 원칙: 기획(요구사항 정의)과 설계·구현을 분리해 기록합니다.
> 모든 항목은 커밋 히스토리로 검증 가능합니다 (`git log --author=Victor`).

| 영역 | 기획 | 설계·구현 |
|---|---|---|
| **커뮤니티** 게시판 전체 (4유형·태그·댓글·이미지) | 본인 | 본인 |
| **신고 시스템** (Human-in-the-Loop 모더레이션) | 본인 | 본인 |
| **문의** 1:1 게시판 + AI 답변 초안 | 본인 | 본인 |
| **AI 모더레이션 파이프라인** (Perspective→BLUR) | 본인 | 본인 |
| **SSE 실시간 알림** (발송 연동 포함) | 지도강사 제안 → 본인 구체화 | 본인 |
| **다크모드** (사이트+어드민 전 모듈) | 본인 | 본인 |
| **모바일 반응형 표준화** (브레이크포인트 문서화+전 모듈) | 본인 | 본인 |
| **광고 캠페인** (스키마→어드민 CRUD→노출/클릭 트래킹) | 본인 | 본인 |
| **AI 챗봇** 고도화 + 챗봇/AI도우미 관리자 | 팀장 | 본인 |
| **어드민 재무(지갑) 관리** (충전한도·환불·적립률 정책) | 본인 (담당 공백 자발 보완) | 본인 |
| **슈퍼관리자** (권한체계·조직도·급여/통계) | 팀장(DB·초안) → 본인 구체화 | 본인 |
| **마이페이지** 최근 조회 내역 | 팀장 | 본인 |
| **ADR 14편** · JUnit 테스트 · 기여 문서 | 본인 | 본인 |

<!-- TODO: git shortlog -sn 캡처 넣기 -->
<!-- ![커밋 분포](assets/commits.png) -->

## 구현 하이라이트 (코드 발췌)

> 아래 코드는 본인이 단독 설계·작성한 부분에서 발췌했습니다.

### 1. 서버측 XSS 정화 — jsoup Safelist (ADR-0005)

클라이언트 escape는 우회 가능하므로 신뢰하지 않고, 저장 직전 서버에서 화이트리스트
기반으로 정화합니다. Summernote 본문은 서식·이미지를 허용하는 커스텀 Safelist,
plain textarea 입력인 댓글은 `Safelist.none()`으로 전체 태그를 제거합니다.

```java
// Summernote 본문용 화이트리스트 — basicWithImages + 서식/인라인스타일 허용
private static final Safelist COMMUNITY_SAFELIST = Safelist.basicWithImages()
        .addTags("h1", "h2", "h3", "h4", "h5", "h6", "u", "s", "strike", "font", "span", "div", "hr")
        .addAttributes("span",  "style")
        .addAttributes("img",   "src", "alt", "width", "height", "style")
        .addProtocols("img", "src", "http", "https", "data");

// <script>, on* 핸들러, javascript: URL 등 실행 가능 요소 제거
private String sanitizeHtml(String html) {
    if (html == null || html.isBlank()) return "";
    return Jsoup.clean(html, "", COMMUNITY_SAFELIST,
            new Document.OutputSettings().prettyPrint(false));
}

// 댓글은 plain textarea 입력 — 모든 HTML 제거, 개행만 보존
private String sanitizeCommentText(String text) {
    if (text == null || text.isBlank()) return "";
    return Jsoup.clean(text, "", Safelist.none(),
            new Document.OutputSettings().prettyPrint(false));
}
```

### 2. SSE 실시간 알림 — 멀티탭 커넥션 관리

유저 1명이 여러 탭을 열 수 있어 `userIdx → List<SseEmitter>` 구조로 관리하고,
완료/타임아웃/에러 3개 콜백에서 동일한 정리 로직으로 누수를 방지합니다.
30초 주기 하트비트(주석 라인)로 프록시·방화벽 idle timeout을 회피합니다.

```java
private final Map<Long, List<SseEmitter>> emitters = new ConcurrentHashMap<>();

public SseEmitter subscribe(Long userIdx) {
    SseEmitter emitter = new SseEmitter(TIMEOUT_MS);
    emitters.computeIfAbsent(userIdx, k -> new CopyOnWriteArrayList<>()).add(emitter);

    Runnable remove = () -> {
        List<SseEmitter> list = emitters.get(userIdx);
        if (list != null) {
            list.remove(emitter);
            if (list.isEmpty()) emitters.remove(userIdx);
        }
    };
    emitter.onCompletion(remove);
    emitter.onTimeout(remove);
    emitter.onError(e -> remove.run());
    // ...초기 connect 이벤트 전송 (프록시 버퍼링 방지)
    return emitter;
}

public void sendTo(Long userIdx, FeedNotificationDto noti) {
    List<SseEmitter> list = emitters.get(userIdx);
    if (list == null || list.isEmpty()) return;
    for (SseEmitter emitter : list) {
        try {
            emitter.send(SseEmitter.event().name("notification").data(noti));
        } catch (IOException e) {
            emitter.complete(); // onCompletion 콜백이 리스트에서 제거
        }
    }
}
```

### 3. AI 독성 감지 — 비동기 파이프라인 + fail-safe (ADR-0010)

Perspective API는 1~5초 지연이 있어 글 등록 응답을 막지 않도록 `@Async`로
분리했습니다. API 장애 시 `false`를 반환해 검사를 건너뜁니다 — 모더레이션
실패가 사용자 글쓰기를 막아서는 안 된다는 fail-safe 원칙입니다.
임계값은 하드코딩하지 않고 어드민이 조정 가능한 정책 테이블에서 읽습니다.

```java
public boolean isToxic(String text) {
    Double score = getToxicityScore(text);
    if (score == null) return false;  // API 실패 → 글쓰기 차단하지 않음 (fail-safe)
    double threshold = moderationPolicyService.getPolicy().getToxicityThreshold();
    return score >= threshold;        // 임계값은 어드민 정책 테이블에서 동적 조회
}

/** 글 등록 응답을 막지 않도록 @Async 로 분리 (Perspective API 1~5초 지연 회피) */
@Async
public void checkAndFlagPostAsync(Long postId, String text) {
    try {
        if (isToxic(text)) {
            communityService.flagPostAsToxic(postId);  // ai_flagged=1 → BLUR 처리
        }
    } catch (Exception e) {
        log.warn("비동기 게시글 독성 검사 실패 (postId={}): {}", postId, e.getMessage());
    }
}
```

### 4. 중복 신고 3중 방어 — 동시성 커버 (ADR-0004)

사전 SELECT만으로는 동시 요청 사이의 race condition을 막을 수 없어
DB UNIQUE 제약을 최종 방어선으로 두고, 취소된 신고는 새 행 INSERT 대신
재활성화해 UNIQUE 제약과 충돌하지 않도록 설계했습니다.

```java
// 3중 방어: ① DB UNIQUE 제약(최종 방어선) ② 사전 SELECT ③ CANCELLED 재활성화
@Override
@Transactional
public boolean submitReport(String targetType, Long targetId, Long userIdx,
                            String reason, String description,
                            String sourceType, Long sourceId) {
    ReportDto existing = reportMapper.selectReportByUserAndTarget(userIdx, targetType, targetId);
    if (existing != null) {
        if ("CANCELLED".equals(existing.getStatus())) {
            // 취소된 신고는 새 행 INSERT가 아니라 재활성화 → UNIQUE 제약과 공존
            reportMapper.reactivateCancelledReport(existing.getReportId(),
                    reason, description, sourceType, sourceId);
            return true;
        }
        return false; // IN_REVIEW / RESOLVED / DISMISSED — 중복 신고 거부
    }
    // ...신규 INSERT (UNIQUE 제약이 동시 요청 race 최종 차단)
}
```

## 그 외 구현

### 커뮤니티 + AI 모더레이션
- Summernote WYSIWYG + Cloudinary 인라인 이미지 업로드, 본문 이미지로 대표/갤러리 자동 구성
- 카운터 캐시(like/comment/report) + **일일 reconcile 스케줄러**로 정합성 안전망 (ADR-0006)
- 인라인 이미지 **orphan 정리 스케줄러** — 본문 HTML 파싱으로 사용 중 이미지 추적
- AI 감지 콘텐츠는 SYSTEM 봇이 자동 신고 등록 → 어드민 검토 큐 합류

### 신고 — Human-in-the-Loop 설계
- 자동 제재는 BLUR까지, 삭제/차단은 어드민 수동 결정 (ADR-0001 — false positive 비용 고려)
- 누적 BLUR 임계값 동적 조절, 처리 결과 신고자 알림

### AI 챗봇 (기획: 팀장 / 구현: 본인)
- ChatGPT 스타일 UI(대화 목록·드래그앤드롭 정렬), Gemini 기반 엔진 전면 교체
- **1차 LLM 의도 분류** → 부적절 입력은 본 호출 생략, 단순 네비게이션은 **fast-path로 LLM 우회** (비용·지연 절감)
- 실데이터 프롬프트 주입 (여행지/패키지/코스/공개 게시글 후보) + URL 화이트리스트 검증
- 등급별 쿼터(주기·리셋 시각 정책화) + 대화 삭제 시 쿼터 환급, 비로그인은 IP 기준
- **링크 클릭 로깅 인프라** (테이블+API+beacon) → 어드민에서 URL별 클릭 집계·클릭자 역추적

### 백오피스 (어드민 / 슈퍼관리자)
- 재무 관리자 페이지: 사용자단 재무는 타 팀원 담당, 관리자단 공백을 발견하고
  read-only MVP → 충전한도(AOP, 기존 결제 코드 무수정) → 토스 cancel 환불 + audit 로그
  → 적립률 정책 → 권한 세분화(FINANCE_OPERATOR/FINANCE_POLICY_ADMIN) 순으로
  이틀간 단계적 자발 구축 (이후 팀장이 UX 정리 참여)
- 슈퍼관리자: 권한 그룹/템플릿 체계 재편, 조직도 트리 UI, 급여/역량 엑셀 업로드, Chart.js 대시보드
- 광고: 캠페인 CRUD, 노출 트래킹, 링크 타입별 내부 컨텐츠 매핑(게시글/여행지/코스)

### 품질·문서
- **ADR 14편** (MADR 표준) — 설계 결정과 근거 기록,
  "코드만 보면 결함으로 오인되는 정책" FAQ 포함
- JUnit Service 단위 테스트 (ADR 검증 전략, Phase 1~3)
- Spring Boot 4 / Security 7 호환 마이그레이션, Jackson 3 호환, JDBC 타임존 세션 강제

## 핵심 기능 하이라이트

> 전체 흐름은 **상단 🎬 시연 영상** 참고. 아래는 핵심 기능별 짧은 GIF (준비 중).

<!-- TODO: 핵심 기능 GIF 3~5개 캡처해서 채우기 -->
<!-- ![커뮤니티](assets/demo-community.gif) — 글 작성 → 이미지 업로드 → 독성 감지 BLUR -->
<!-- ![챗봇](assets/demo-chatbot.gif) — 의도 분류 → 실데이터 추천 → 링크 이동 -->
<!-- ![알림](assets/demo-sse.gif) — 댓글 작성 → 상대방 화면 실시간 토스트 -->

*(전체 기능은 면접 시 라이브 시연 가능합니다)*
