# ADR-0007: 이미지 스토리지 — Cloudinary 외부 CDN 채택

* Status: Accepted
* Date: 2026-04-24
* Decision-Makers: Victor Jung
* Consulted: N/A
* Informed: N/A

## Context and Problem Statement

커뮤니티 게시글 본문 인라인 이미지, 문의 첨부파일 등 사용자 업로드 이미지를 어디에 저장하고 어떻게 서빙할지 결정해야 했습니다.

초기 Phase 1 에서는 Summernote 가 base64 임시 삽입을 사용했지만, 이는 다음 문제가 있습니다:

* DB 사이즈 폭증 (이미지가 본문 HTML 안에 들어감)
* 페이지 응답 무거워짐
* 이미지 변환·최적화 불가
* 캐싱 전략 부재

Phase 2 에서 정식 이미지 스토리지가 필요했습니다.

## Decision Drivers

* **개인/소규모 프로젝트 운영 부담** — 자체 호스팅 시 디스크/대역폭/백업 운영 비용
* **이미지 변환·최적화** — 모바일/데스크톱별 다른 크기 자동 변환
* **CDN 글로벌 캐싱** — 한국 외 사용자에게도 빠른 응답
* **WAR 배포 환경** — 서버 재배포/이전 시 이미지 마이그레이션 부담
* **무료 티어 활용** — 학습/포트폴리오 단계에서 비용 0
* **Java SDK 지원** — Spring 환경에서 통합 단순성

## Considered Options

* **Option A — 자체 디스크 저장 + Spring 정적 리소스 서빙**
  현재 일부 구현되어 있음 (`upload/community/`).
* **Option B — Cloudinary 외부 CDN (선택)**
  무료 티어 + 글로벌 CDN + 변환 API.
* **Option C — AWS S3 + CloudFront**
  엔터프라이즈급. 비용/설정 복잡도 큼.
* **Option D — Firebase Storage**
  Google 생태계. SDK 좋음.

## Decision Outcome

**Chosen option: "Option B — Cloudinary"**, because 무료 티어로 포트폴리오 단계에서 비용 0이며, 이미지 변환·최적화·글로벌 CDN 캐싱을 즉시 사용할 수 있고, Java SDK 가 단순해 Spring 환경에서 통합 비용이 가장 낮습니다.

### 폴더 구조

| 폴더 | 용도 |
|---|---|
| `community/` | 커뮤니티 게시글 대표 이미지 |
| `community/inline/` | Summernote 본문 인라인 이미지 |
| (문의/광고 등) | 모듈별 분리 |

### 업로드 플로우

```
[사용자] Summernote 에디터에서 이미지 삽입
  ↓
[클라이언트] onImageUpload 후크 → POST /community/inline-image
  ↓
[서버] CloudinaryService.uploadImage(file, "community/inline")
  ↓
[Cloudinary] secure_url 반환 (https://res.cloudinary.com/...)
  ↓
[클라이언트] $.summernote('insertImage', secure_url)
  ↓
[저장] 본문 HTML 안에 외부 URL 만 박힘
```

### 고아 이미지 정리

본문 HTML 안에 `<img src>` 로 박혀 있어, 글 작성 도중 업로드만 하고 글을 안 올리면 Cloudinary 에 고아 이미지 누적됩니다. 이를 정리하는 별도 스케줄러 운영:

```text
// CommunityImageScheduler
@Scheduled(cron = "0 0 3 * * *")  // 매일 새벽 3시
public void cleanupOrphanImages() {
    // 1. ACTIVE 게시글 본문에서 사용 중인 publicId 수집 (jsoup 으로 <img src> 파싱)
    // 2. Cloudinary community/inline/ 폴더 전체 publicId 조회
    // 3. 차집합 = 고아 이미지 → Cloudinary API 로 삭제
}
```

### Consequences

* Good
  * **운영 부담 0** — 디스크/대역폭/백업 모두 Cloudinary 위임
  * **자동 변환·최적화** — `w_300,h_300,c_fill,f_auto,q_auto` 같은 URL 파라미터로 즉석 변환
  * **글로벌 CDN** — Cloudinary 의 CDN 인프라로 글로벌 빠른 응답
  * **WAR 배포 친화** — 서버 재배포/이전 시 이미지 영향 없음
  * **무료 티어** — 25 monthly credits 가 포트폴리오 단계에서 충분
* Bad
  * **외부 의존성** — Cloudinary 장애/가격 변동 시 영향
  * **고아 이미지 청소 잡 필요** — 본문 HTML diff 추적 부담
  * **벤더 락인** — 마이그레이션 시 모든 URL 재발급 필요
  * **무료 티어 한계** — 트래픽 급증 시 유료 전환 필요
* Neutral
  * 클라이언트 → Cloudinary 직접 업로드(unsigned upload) 도 가능하지만, 권한 검증을 위해 서버 경유 방식 채택

## Pros and Cons of the Options

### Option A — 자체 디스크

* Good, because 외부 의존성 없음
* Good, because 비용 통제 가능
* Bad, because 디스크/대역폭/백업 운영 부담
* Bad, because 변환·최적화 직접 구현 필요
* Bad, because WAR 재배포 시 이미지 마이그레이션 부담
* Bad, because CDN 캐싱 직접 설정 필요

### Option B — Cloudinary (선택)

* Good, because 운영 부담 0, 변환·CDN 즉시 사용
* Good, because 무료 티어로 포트폴리오 단계 비용 0
* Bad, because 외부 의존성, 벤더 락인

### Option C — AWS S3 + CloudFront

* Good, because 엔터프라이즈급 안정성
* Good, because AWS 생태계 통합
* Bad, because **비용 발생** (포트폴리오 단계 부담)
* Bad, because IAM/Bucket Policy/CloudFront 설정 복잡
* Bad, because 변환은 Lambda@Edge 등 별도 구성 필요

### Option D — Firebase Storage

* Good, because Google 생태계 통합
* Good, because SDK 좋음
* Bad, because Java/Spring 보다 모바일/JS SDK 가 1순위
* Bad, because 변환 기능 약함

## More Information

### 관련 ADR

* [ADR-0002](./0002-summernote-editor.md) — Summernote 가 본 스토리지의 직접 사용처

### 코드 위치

* 서비스: `src/main/java/org/triptogether/cloudinary/CloudinaryService.java`
* 인라인 이미지 업로드: `CommunityController.uploadInlineImage()` (`POST /community/inline-image`)
* 고아 정리 스케줄러: `src/main/java/org/triptogether/community/service/CommunityImageScheduler.java`
* 설정: `application.properties` `cloudinary.cloud-name` / `cloudinary.api-key` / `cloudinary.api-secret`

### TODO (P0 보완)

* `uploadInlineImage()` 의 파일 타입/MIME/크기 검증 강화 (`jpg/jpeg/png/gif/webp`, 5MB 한계)

### 참고

* [Cloudinary Java SDK](https://cloudinary.com/documentation/java_integration)
* [Cloudinary Image Transformations](https://cloudinary.com/documentation/image_transformations)
