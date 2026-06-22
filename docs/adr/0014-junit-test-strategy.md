# ADR-0014: JUnit 테스트 전략 — Service 단위 테스트 + ADR 정책 검증

* Status: Accepted
* Date: 2026-04-26
* Decision-Makers: Victor Jung
* Consulted: N/A
* Informed: 팀원 전체

## Context and Problem Statement

본 모듈(`community / report / inquiry`) 에는 ADR-0001 ~ ADR-0013 까지 12개의 의식적 설계 결정이 있지만, **이 결정들이 실제 코드로 정확히 동작하는지 검증할 자동화 테스트가 부족**한 상태였습니다. 기존 테스트는 다음 한계가 있었습니다:

* `CommunityServiceTest` 는 일부 도배 방지/좋아요 케이스만 커버
* 의존성 변경 후 Mock 누락으로 깨진 채 방치되어 있었음
* `ReportServiceImpl` / `InquiryServiceImpl` 에 단위 테스트 자체가 없었음
* ADR 정책(3중 방어 / 자동 BLUR 임계값 / 답변 이력 / Soft Delete 등) 이 코드로 검증된 흔적 없음

→ "정책은 ADR 에 있지만, 그 정책이 코드로 동작한다는 증거는 없는" 상태.

## Decision Drivers

* **신입 포트폴리오 약점 1순위**: 테스트 커버리지 부재
* **ADR ↔ 코드 매핑** — 정책이 단순 문서가 아니라 검증 가능한 동작임을 증명
* **회귀 방지** — 의존성 추가/리팩터링 시 정책 위반을 즉시 감지
* **시간 효율** — Service 단위 테스트가 가장 가성비 좋음 (DB/HTTP 부팅 비용 0)

## Considered Options

* **Option A — 풀 통합 테스트** (`@SpringBootTest`)
  높은 신뢰성 / 매우 느림 / 검증 단위 모호
* **Option B — Service 단위 + Mockito (선택)**
  빠름 / ADR 정책 검증에 충분 / Mock 부담만 있음
* **Option C — Mapper 통합 테스트** (`@MybatisTest` + H2)
  SQL 검증에 좋음 / H2 와 MySQL 방언 차이 위험
* **Option D — Controller 슬라이스** (`@WebMvcTest`)
  HTTP 계층 검증 / Spring 컨텍스트 부팅 부담

## Decision Outcome

**Chosen option: "Option B — Service 단위 + Mockito"** 를 메인으로, 향후 필요 시 Option D 를 보조적으로 결합. 이유:

* 본 모듈의 **핵심 정책은 모두 Service 레이어**에 있음 (sanitize / BLUR 임계값 / 답변 이력 / Soft Delete / 3중 방어 / 도배 방지)
* Mockito 로 Mapper / 외부 서비스 격리 → 정책 로직만 정확히 검증
* 빠른 실행 (전체 31개 테스트 ~12초)

### 작성 패턴 (표준)

```text
@ExtendWith(MockitoExtension.class)
class XxxServiceTest {
    @Mock XxxMapper xxxMapper;
    @Mock OtherDependency other;
    @InjectMocks XxxServiceImpl service;

    @BeforeEach
    void setUp() {
        // 정책 객체 stub (ADR-0009 의 ContentModerationPolicyVO)
        // i18n 메시지 stub (ADR-0013 의 MessageUtil)
        lenient().when(...).thenReturn(...);
    }

    @Test
    @DisplayName("정책 X - 조건 Y 일 때 Z 동작")
    void method_condition_expectedBehavior() {
        given(...).willReturn(...);
        // when
        service.method(...);
        // then
        verify(...);
        assertThat(...);
    }
}
```

* **JUnit 5 (Jupiter)** + **Mockito 5** + **AssertJ**
* **BDD style** (`given().willReturn()`, `assertThatThrownBy`)
* **`lenient()`** 로 stub strict mode 회피 (필요한 케이스만)
* **DisplayName 한국어** — 정책-동작 매핑이 한눈에 보이게

## ADR ↔ 테스트 매핑 (현재 적용된 검증)

| ADR | 검증 테스트 | 위치 |
|---|---|---|
| **ADR-0001** (자동 제재 BLUR 까지) | `updatePostReportCache_atThreshold_sendsBlurNotification` / `updateReportStatus_resolved_sendsNotification` / `submitReport_*` | Community + Report |
| **ADR-0003** (BLUR vs BLOCKED) | `updatePostReportCache_*` / `clearPostBlur_callsMapperAndNotifies` | Community |
| **ADR-0004** (중복 신고 3중 방어) | `submitReport_duplicateInReview_returnsFalse` / `submitReport_cancelledReactivation_returnsTrue` / `submitReport_dataIntegrityViolation_returnsFalse` | Report |
| **ADR-0005** (XSS sanitize) | `addComment_sanitize_removesScriptTag` | Community |
| **ADR-0006** (캐시 컬럼 동기화) | `deleteComment_softDeleteAndDecrement` (comment_count 감소 검증) | Community |
| **ADR-0008** (Soft Delete) | `deletePost_softDelete_setsStatusDeleted` / `deleteComment_softDeleteAndDecrement` | Community |
| **ADR-0009** (정책 외부화) | `@BeforeEach` 의 `ContentModerationPolicyVO` stub 자체가 외부화 검증 | Community + Inquiry |
| **답변 이력 (Phase 1)** | `updateAnswer_archivesPreviousThenUpdates` / `deleteAnswer_archivesAsDeleteThenRemoves` / `updateAnswer_noExisting_skipsHistory` | Inquiry |
| **P0 보안: 첨부파일 검증** | `addAttachment_invalidExtension_skipped` / `addAttachment_oversize_skipped` | Inquiry |
| **도배 방지** | `*_floodLimit_throwsException` 다수 | Community + Inquiry |

→ **거의 모든 ADR/P0 정책이 최소 1개 테스트로 검증됨**.

## 테스트 통계

| 클래스 | 개수 | 주요 검증 |
|---|---|---|
| `ReportServiceTest` | 5 | ADR-0001 / 0004 |
| `CommunityServiceTest` | 14 (기존 8 + 신규 6) | ADR-0001/0003/0005/0006/0008 + 도배 방지 |
| `InquiryServiceTest` | 8 (전부 신규) | 답변 이력 + 도배 방지 + P0 첨부 검증 |
| `SuperAdminServiceTest` | 3 (기존) | 본 ADR 적용 외 |
| `TripTogetherApplicationTests` | 1 | 컨텍스트 부팅 |
| **합계** | **31** (전부 통과) | |

## Consequences

* Good
  * **ADR 정책의 코드 동작 증명** — 면접관이 ADR + 테스트 페어 보면 신뢰도 큼
  * **회귀 방지** — 의존성/시그니처 변경 시 테스트가 즉시 실패해 알림
  * **빠른 실행** (~12초) — CI/CD 빈번 실행 가능
  * **신입 포트폴리오 1순위 약점 해소**
* Bad
  * **Mapper SQL 자체는 미검증** (단위 테스트라 쿼리 정확성은 못 봄) → 향후 `@MybatisTest` 도입 검토
  * **컨트롤러 계층 미검증** (ADR-0011 AOP 권한 / ADR-0012 CSRF) → `@WebMvcTest` 1~2개 추가 검토
  * **Mock 설정의 일치성 부담** — 시그니처 변경 시 Mock 도 같이 수정 필요
* Neutral
  * `@BeforeEach` 의 lenient stub 은 strict mode 회피 — 의도된 절충

## Pros and Cons of the Options

### Option A — 풀 통합 테스트

* Good, because 가장 신뢰성 높음
* Bad, because 매우 느림 (Spring 컨텍스트 부팅 + DB)
* Bad, because 단위 검증 모호 (어디서 깨졌는지 추적 어려움)

### Option B — Service 단위 + Mockito (선택)

* Good, because 빠르고 검증 단위 명확
* Good, because ADR 정책의 비즈니스 로직 검증에 최적
* Bad, because Mapper SQL / HTTP 계층 미검증

### Option C — Mapper 통합 테스트

* Good, because SQL 정확성 검증
* Bad, because H2 와 MySQL 의 방언 차이 (UPSERT, LIMIT 서브쿼리 등)
* Bad, because 별도 인프라 부담

### Option D — Controller 슬라이스

* Good, because HTTP/권한 계층 검증
* Good, because ADR-0011 (AOP) / 0012 (CSRF) 검증 가능
* Bad, because Spring 컨텍스트 부팅 비용
* Bad, because 본 ADR 의 핵심 정책은 Service 레이어라 우선순위 낮음

## More Information

### 향후 확장 계획

| 단계 | 추가 범위 | 시점 |
|---|---|---|
| Phase 1~3 (현재) | Service 단위 테스트 19개 추가 | 2026-04-26 |
| Phase 4 (예정) | Controller 슬라이스 (`@WebMvcTest`) — ADR-0011/0012 검증 | 추후 |
| Phase 5 (예정) | Mapper 통합 (`@MybatisTest`) — SQL 방언 정확성 | 추후 |
| Phase 6 (예정) | E2E 통합 (`@SpringBootTest` + TestContainers) | 매우 추후 |

### 코드 위치

* `src/test/java/org/triptogether/community/CommunityServiceTest.java`
* `src/test/java/org/triptogether/report/ReportServiceTest.java`
* `src/test/java/org/triptogether/inquiry/InquiryServiceTest.java`

### 면접 어필 포인트

* **ADR + 테스트 페어** — "정책이 단순 문서가 아니라 자동화 검증된다" 증명
* **신입 약점 처리** — 테스트 커버리지 부재라는 일반적 약점을 능동적으로 해소
* **점진적 확장 계획** — Service → Controller → Mapper → E2E 단계 명시
* **회귀 방지 의식** — 의존성 변경 시 즉시 실패하도록 설계
