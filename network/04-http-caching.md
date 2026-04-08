# HTTP Caching Strategy

CDN과 브라우저 캐시 정책을 설계하기 위한 HTTP 캐싱 전략 총정리.

## Cache-Control Header

캐시 동작을 제어하는 핵심 헤더. 여러 directive를 조합해서 사용한다.

### 저장 여부 제어

| Directive | 설명 |
|-----------|------|
| `no-store` | 캐시 자체를 하지 않음. 민감한 데이터(결제, 개인정보)에 사용 |
| `no-cache` | 캐시는 하되, 매번 서버에 재검증 요청. `no-store`와 혼동 주의 |
| `private` | 브라우저만 캐시 가능 (CDN 캐시 불가). 사용자별 데이터에 사용 |
| `public` | CDN 포함 모든 중간 캐시 허용 |

### no-store vs no-cache 상세 비교 (Detailed Comparison)

이름 때문에 가장 많이 혼동되는 두 directive.

- `no-store` = 아예 저장하지 마 (Don't store at all)
- `no-cache` = 저장은 해도 되지만, 쓰기 전에 매번 서버에 물어봐 (Store, but always revalidate before use)

**no-cache — 캐시하되 매번 재검증 (Cache, but Always Revalidate):**
```
브라우저: "이 리소스 캐시에 있긴 한데, 써도 돼?"
  → 서버에 조건부 요청 (Conditional Request)
  → If-None-Match: "etag-abc"

서버: "변경 없어" → 304 Not Modified (body 없음, 빠름)
서버: "변경됐어" → 200 + 새 body
```
- 브라우저 캐시에 저장됨 (O)
- 사용 전 서버에 재검증 (O, 매번)
- 변경 없으면 304 (body 전송 안 함 → 빠름)

`no-cache`라는 이름이 "캐시하지 마"처럼 보이지만, 실제 의미는 "캐시는 하되, 재검증 없이 사용하지 마 (Do not use without revalidation)"다.

**no-store — 어디에도 저장하지 마 (Don't Store Anywhere):**
```
브라우저: 이 응답을 어디에도 저장하지 않음
  → 메모리 캐시 X, 디스크 캐시 X
  → 매번 서버에 완전한 요청 (Full Request)

서버: 항상 200 + 전체 body 반환
  → 304 불가 (비교할 캐시가 없으니까)
```

**비교표 (Comparison):**

| | `no-cache` | `no-store` |
|---|---|---|
| 캐시 저장 (Store) | O (저장함) | X (저장 안 함) |
| 사용 전 재검증 (Revalidate) | O (매번) | 해당 없음 |
| 304 응답 가능 (304 Possible) | O (변경 없으면) | X (비교 대상 없음) |
| 네트워크 요청 크기 (Request Size) | 작음 (재검증용) | 큼 (전체 body) |
| 보안 민감 데이터 (Sensitive Data) | 부적합 (Not suitable) | 적합 (Suitable) |

**언제 쓰는가 (When to Use):**
```
no-cache — "최신 여부만 확인하면 되는" 리소스:
  Cache-Control: no-cache
  ETag: "v1.2.3"
  → SPA의 index.html, API 응답 등

no-store — "흔적도 남기면 안 되는" 리소스:
  Cache-Control: no-store
  → 결제 정보, 인증 토큰, 개인정보
  → 공용 PC에서 민감 데이터가 캐시에 남는 걸 방지
```

### no-store는 웹뷰에서도 완벽하게 동작하는가? (Does no-store Work in WebViews?)

스펙상으로는 완전 차단이지만, 현실은 다르다.

**일반 브라우저 (Chrome, Safari, Firefox):**
- `no-store`를 대체로 잘 지킴
- 하지만 bfcache (Back/Forward Cache)에서 예외 발생 가능

```
bfcache 문제:
  사용자가 결제 페이지 → 뒤로가기
  → 브라우저가 bfcache에서 페이지를 복원
  → no-store인데도 이전 페이지가 보임

  Chrome: no-store 페이지는 bfcache에서 제외 (2022년~)
  Safari: 여전히 bfcache에 남을 수 있음 (일부 버전)
  Firefox: no-store 페이지는 bfcache에서 제외
```

**웹뷰 (WebView) — 여기가 문제:**
```
Android WebView:
  - WebSettings.setCacheMode() 설정에 따라 달라짐:
    LOAD_DEFAULT              → no-store 존중 (정상)
    LOAD_CACHE_ELSE_NETWORK   → no-store 무시할 수 있음!
    LOAD_CACHE_ONLY           → 캐시에 있으면 no-store 무시
  - 앱 개발자가 캐시 모드를 잘못 설정하면 no-store가 무력화

iOS WKWebView:
  - 기본적으로 no-store를 존중
  - 하지만 URLCache가 디스크에 기록하는 경우가 보고됨

커스텀 WebView / 인앱 브라우저 (카카오톡, 네이버 등):
  - HTTP 캐시 스펙을 완벽히 구현하지 않을 수 있음
  - 자체 캐시 레이어가 no-store를 무시할 가능성 있음
```

**방어적 전략 — Belt and Suspenders Approach:**
```
Cache-Control: no-store, no-cache, must-revalidate, private, max-age=0
Pragma: no-cache          ← HTTP/1.0 하위 호환 (Legacy Compatibility)
Expires: 0                ← HTTP/1.0 하위 호환

각 헤더의 역할:
  no-store          → 저장하지 마 (스펙 준수 브라우저용)
  no-cache          → 저장했더라도 재검증 없이 쓰지 마 (fallback)
  must-revalidate   → 만료 후 반드시 재검증 (stale 응답 방지)
  private           → CDN/프록시 캐시 금지 (중간 캐시 차단)
  max-age=0         → 즉시 만료 (혹시 저장됐더라도)
  Pragma: no-cache  → HTTP/1.0 프록시 대응
  Expires: 0        → HTTP/1.0 프록시 대응
```

**실무 권장 (Practical Recommendation):**
```
일반 웹 (Modern Browsers):
  Cache-Control: no-store
  → 충분함 (Sufficient)

웹뷰 / 인앱 브라우저 / 레거시 대응:
  Cache-Control: no-store, no-cache, must-revalidate, private, max-age=0
  Pragma: no-cache
  Expires: 0
  → 방어적 조합 (Defense in Depth)
```

### 만료 제어

| Directive | 설명 |
|-----------|------|
| `max-age=<seconds>` | 브라우저 캐시 유효 시간 |
| `s-maxage=<seconds>` | CDN(shared cache) 전용 유효 시간. `max-age`보다 우선 |

### 재검증 관련

| Directive | 설명 |
|-----------|------|
| `must-revalidate` | 만료 후 반드시 서버 재검증 (오프라인 시 504 반환) |
| `stale-while-revalidate=<seconds>` | 만료 후에도 이 시간 동안은 stale 응답을 먼저 주고, 백그라운드에서 재검증 |
| `stale-if-error=<seconds>` | 서버 에러 시 stale 응답을 이 시간 동안 허용 |
| `immutable` | 콘텐츠가 절대 변하지 않음을 선언. 브라우저 새로고침 시에도 재검증 안 함 |

## Validation (조건부 요청)

캐시가 만료됐을 때 전체 리소스를 다시 받을 필요 없이, 변경 여부만 확인하는 메커니즘.

### ETag (Entity Tag)

```
# 서버 응답
ETag: "abc123"

# 브라우저 재검증 요청
If-None-Match: "abc123"

# 변경 없으면 → 304 Not Modified (body 없음)
# 변경 있으면 → 200 + 새 body
```

### Last-Modified

```
# 서버 응답
Last-Modified: Mon, 07 Apr 2026 08:00:00 GMT

# 브라우저 재검증 요청
If-Modified-Since: Mon, 07 Apr 2026 08:00:00 GMT
```

ETag가 Last-Modified보다 정밀함. 1초 이내 변경이나 내용 동일한 경우도 구분 가능.

## 실전 캐시 정책 설계

### 1. 해시된 정적 자산 (JS/CSS/이미지 번들)

```
Cache-Control: public, max-age=31536000, immutable
```

- 파일명에 content hash 포함 (`app.a1b2c3.js`)
- 내용 바뀌면 파일명이 바뀌므로 1년 캐시 + immutable 안전
- CDN과 브라우저 모두 캐시

### 2. HTML (SPA의 index.html)

```
Cache-Control: no-cache
ETag: "v1.2.3"
```

- 항상 재검증하되, 변경 없으면 304로 빠르게 응답
- `no-store`가 아닌 `no-cache`인 이유: 캐시 자체는 유지하면서 최신 여부만 확인
- HTML이 새 JS 번들을 참조하므로, HTML만 최신이면 나머지는 hash로 해결

### 3. API 응답 (공개 데이터)

```
Cache-Control: public, s-maxage=60, stale-while-revalidate=300
```

- CDN에서 60초 캐시
- 만료 후 5분간은 stale 응답 즉시 반환 + 백그라운드 갱신
- 사용자는 항상 빠른 응답을 받고, 데이터는 점진적으로 갱신

### 4. API 응답 (사용자별 데이터)

```
Cache-Control: private, no-cache
ETag: "user-data-hash"
```

- CDN 캐시 금지 (`private`)
- 브라우저에서 매번 재검증

### 5. 민감한 데이터 (결제, 인증)

```
Cache-Control: no-store
```

- 어디에도 캐시하지 않음

## stale-while-revalidate 동작 흐름

```
요청 → [max-age 이내?]
         ├─ Yes → 캐시 즉시 반환 (fresh)
         └─ No  → [swr 이내?]
                    ├─ Yes → stale 캐시 즉시 반환 + 백그라운드 revalidate
                    └─ No  → 서버 요청 대기 (slow)
```

`max-age=60, stale-while-revalidate=300` 예시:
- 0~60초: 캐시 그대로 사용 (fresh)
- 60~360초: 일단 stale 캐시 반환하고 뒤에서 갱신
- 360초 이후: 서버 응답 기다림

## CDN vs Browser Cache 전략 분리

`s-maxage`와 `max-age`를 다르게 설정해서 CDN과 브라우저 캐시를 독립적으로 제어할 수 있다.

```
Cache-Control: public, max-age=0, s-maxage=3600, stale-while-revalidate=86400
```

- 브라우저: 매번 CDN에 요청 (`max-age=0`)
- CDN: 1시간 캐시, 만료 후 24시간 동안 swr
- 결과: CDN invalidation만으로 전체 사용자에게 즉시 반영 가능

## Cache Busting 전략

| 방식 | 예시 | 장점 | 단점 |
|------|------|------|------|
| Content Hash | `app.a1b2c3.js` | 내용 기반, 정확함 | 빌드 도구 필요 |
| Version Query | `app.js?v=1.2.3` | 간단 | 일부 CDN이 query 무시 |
| Path Versioning | `/v2/app.js` | CDN 호환성 좋음 | 경로 관리 복잡 |

Content Hash가 가장 권장되는 방식. Webpack/Vite 등 번들러가 자동 처리.

## Browser Cache Storage Layers

브라우저가 캐시된 리소스를 반환할 때, DevTools Network 탭의 Size 컬럼에 저장 위치가 표시된다.

### Memory Cache vs Disk Cache

| | Memory Cache | Disk Cache |
|---|---|---|
| 저장 위치 | RAM | 디스크 (파일 시스템) |
| 속도 | 매우 빠름 (~0ms) | 빠름 (~1-10ms) |
| 생존 범위 | 탭/프로세스 종료 시 소멸 | 브라우저 종료 후에도 유지 |
| DevTools 표시 | `(memory cache)` | `(disk cache)` |
| 대상 | 작은 리소스, 자주 접근하는 리소스 | 큰 리소스, 장기 캐시 대상 |

### 브라우저의 캐시 탐색 순서

```
요청 발생
  → Service Worker Cache (있으면)
    → Memory Cache
      → Disk Cache
        → HTTP/2 Push Cache (세션 내)
          → Network 요청
```

1. **Service Worker** — `caches.match()`로 직접 제어하는 캐시
2. **Memory Cache** — 현재 탭의 RAM에 있는 리소스. preload된 리소스도 여기
3. **Disk Cache** — `Cache-Control` 헤더 기반으로 디스크에 저장된 리소스
4. **Push Cache** — HTTP/2 서버 푸시로 받은 리소스 (세션 종료 시 소멸)
5. **Network** — 위 모든 캐시에 없으면 실제 네트워크 요청

### Memory Cache에 들어가는 조건

브라우저가 자체적으로 판단하며, 명시적 API는 없다. 일반적으로:

- `<link rel="preload">`로 로드된 리소스
- 같은 페이지에서 이미 로드된 리소스 (이미지, 스크립트 등)
- 작은 크기의 리소스 우선
- 탭이 살아있는 동안만 유지

```html
<!-- preload된 리소스는 memory cache에 저장됨 -->
<link rel="preload" href="/fonts/main.woff2" as="font" crossorigin>
<link rel="preload" href="/critical.css" as="style">
```

### DevTools Network 탭에서 확인하는 법

Size 컬럼 표시:
- `(memory cache)` — RAM에서 즉시 반환, 0ms
- `(disk cache)` — 디스크에서 반환, 수 ms
- `(ServiceWorker)` — SW가 처리
- `(prefetch cache)` — prefetch로 미리 받아둔 리소스
- 실제 바이트 수 — 네트워크에서 새로 받음

### 304 Not Modified vs Cache Hit

```
(memory cache) / (disk cache)  →  서버에 요청 자체를 안 보냄 (Cache-Control 유효)
304 Not Modified               →  서버에 재검증 요청을 보냈고, 변경 없음 확인
200 (실제 크기)                 →  서버에서 새로 받음
```

이 구분이 중요한 이유: `no-cache`는 매번 304 확인을 하고, `max-age`가 유효하면 아예 요청을 안 보낸다.

## 실전 배포 캐시 전략 — S3 + CloudFront (Deployment Caching Strategy)

### 구조 (Architecture)

```
브라우저 → CloudFront (CDN) → S3 (Origin)

배포 시:
  1. S3에 새 파일 업로드
  2. CloudFront Invalidation 실행 → CDN 캐시 삭제
  3. 다음 요청 시 CDN이 S3에서 새 파일을 가져옴
```

### no-cache + Invalidation으로 충분한가? (Is no-cache + Invalidation Enough?)

**HTML (index.html) — no-cache로 충분하다:**
```
Cache-Control: no-cache
ETag: "abc123"

동작:
  브라우저 → CloudFront에 요청 (매번 재검증, Always Revalidate)
  CloudFront → S3에서 ETag 비교
    → 변경 없으면: 304 (빠름)
    → 변경 있으면: 200 + 새 HTML

Invalidation 후:
  CloudFront 캐시가 비워짐
  → 다음 요청 시 S3에서 새 HTML을 가져옴
  → 브라우저도 재검증하므로 새 버전 즉시 반영
```

**정적 자산 (JS/CSS)은 다른 전략이 필요하다:**
```
no-cache를 JS/CSS에도 쓰면?
  → 매 요청마다 재검증 → 불필요한 네트워크 요청 (성능 낭비)
  → 304라도 RTT는 발생

더 나은 전략:
  JS/CSS → content hash + immutable + 장기 캐시 (Long-Term Cache)
  HTML → no-cache + ETag

  app.a1b2c3.js → Cache-Control: public, max-age=31536000, immutable
  index.html    → Cache-Control: no-cache, ETag: "v1.2.3"

  배포 시:
    새 JS 파일명: app.d4e5f6.js (hash 변경)
    새 HTML: <script src="app.d4e5f6.js"> (참조 변경)
    → JS는 Invalidation 필요 없음 (파일명 자체가 다르니까)
    → HTML만 Invalidation하면 됨
```

### Invalidation이 필요한 경우 vs 불필요한 경우 (When to Invalidate)

```
필요한 경우 (Need Invalidation):
  - HTML (no-cache) — CDN이 캐시한 HTML을 즉시 갱신
  - 파일명이 바뀌지 않는 리소스 — favicon.ico, robots.txt 등
  - CDN에 s-maxage를 설정한 경우 — CDN 캐시를 강제로 비움

불필요한 경우 (No Invalidation Needed):
  - content hash가 포함된 정적 자산 — 파일명이 다르므로 새 URL
  - no-store 리소스 — 캐시 자체가 없으므로
```

### 배포 순서가 중요하다 (Deployment Order Matters)

```
올바른 순서 (Correct Order):
  ① 새 정적 자산을 S3에 업로드 (app.d4e5f6.js)
  ② 새 HTML을 S3에 업로드 (새 JS를 참조하는 index.html)
  ③ CloudFront Invalidation 실행 (/index.html)

왜?
  HTML을 먼저 올리면 → 아직 없는 JS를 참조 → 404 에러
  JS를 먼저 올리면 → 기존 HTML이 기존 JS를 참조 → 안전 (Safe)
```

### no-cache vs max-age=0 + s-maxage (Subtle Difference)

```
Cache-Control: no-cache
  → 브라우저: 매번 재검증 (항상)
  → CDN: 매번 Origin에 재검증 (항상)

Cache-Control: max-age=0, s-maxage=3600
  → 브라우저: 매번 CDN에 재검증 (max-age=0)
  → CDN: 1시간 캐시 (s-maxage=3600), Invalidation으로 제어
  → CDN 부하 감소, Origin 요청 줄임 (Reduced Origin Load)
```

실무에서는 후자가 더 많이 쓰인다:
- CDN이 Origin 앞에서 버퍼 역할 (Buffer)
- Invalidation으로 즉시 갱신 가능 (Instant Purge)
- Origin(S3) 요청 비용 절감 (Cost Reduction)
