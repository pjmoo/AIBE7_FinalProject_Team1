# 회원가입 → 로그인 코드 리뷰 (한 줄씩 해설 + 데이터 흐름)

> 대상: 회원가입부터 로그인(과 로그인 직후 토큰이 쓰이는 과정)까지.
> 기준: 2026-10-30 현재 `main` 브랜치 코드를 다시 읽고 작성했다. 이전 문서는 지웠다.
> 표기: ✅ 좋은 점 / ⚠️ 확인·수정 권장 / 💡 참고·제안
> 줄 번호(`L숫자`)는 이 시점 파일 기준이다. 코드만 읽었고 **테스트·빌드는 이 문서를 쓰면서 실행하지 않았다.**
> 빈 줄과 `}` 같은 닫는 괄호는 설명하지 않았다. `import`는 클래스 이름 단위로 풀었다.

---

## 📍 이 문서의 각 섹션이 가리키는 실제 파일 위치

경로 약어: **서버** = `src/main/java/org/example/springtestci/` / **화면** = `src/main/resources/static/` / **DB** = `src/main/resources/db/migration/` / **설정** = `src/main/resources/`

| 섹션 | 리뷰 대상 파일 | 실제 위치 |
|---|---|---|
| 3-1 | `signup.html` | 화면 `pages/signup.html` |
| 3-2 | `auth.html` | 화면 `pages/auth.html` |
| 3-3 | `auth.js` | 화면 `js/auth.js` |
| 4-1 | `SecurityConfig.java` | 서버 `common/security/SecurityConfig.java` |
| 4-2 | `CsrfCookieFilter.java` | 서버 `common/security/CsrfCookieFilter.java` |
| 4-3 | `CookieBearerTokenFilter.java` | 서버 `common/security/CookieBearerTokenFilter.java` |
| 4-4 | `WebMvcConfig.java`, `RateLimitInterceptor.java` | 서버 `common/config/` |
| 4-4 | `RequestRateLimiter.java` | 서버 `common/security/RequestRateLimiter.java` |
| 5 | `AuthController.java` | 서버 `auth/controller/AuthController.java` |
| 6 | DTO 8개 (`SignupRequest`, `LoginRequest`, `EmailResendRequest`, `SignupResponse`, `UserResponse`, `LoginResponse`, `AuthTokens`, `SupabaseUserResponse`) | 서버 `auth/dto/` |
| 7, 9, 10-3 | `AuthService.java` | 서버 `auth/service/AuthService.java` |
| 8-1, 8-2 | `EmailVerificationService.java`, `VerificationStatus.java` | 서버 `auth/service/` |
| 8-3 | `SupabaseAuthClient.java` | 서버 `auth/service/SupabaseAuthClient.java` |
| 8-4 | `SupabaseConfig.java`, `SupabaseProperties.java` | 서버 `common/config/` |
| 9-1 | `LoginRateLimiter.java` | 서버 `auth/service/LoginRateLimiter.java` |
| 9-2, 9-3 | `GlobalExceptionHandler`, `CommonException`, `RateLimitedException`, `ErrorResponse`, `ErrorCode` | 서버 `common/error/` |
| 10-1 | `JwtTokenProvider.java` | 서버 `auth/service/JwtTokenProvider.java` |
| 10-2 | `RefreshTokenStore.java` | 서버 `auth/service/RefreshTokenStore.java` |
| 10-4 | `AccessTokenBlocklist`, `JwtProperties`, `ProductionSecurityGuard` | 서버 `common/security/` |
| 11-1 | `User.java`, `UserRole.java` | 서버 `auth/entity/` |
| 11-2 | `UserRepository.java` | 서버 `auth/repository/UserRepository.java` |
| 11-4 | `V5__create_users.sql`, `V2026101901__add_users_email_verified.sql` | DB |
| 12 | `UnverifiedUserCleaner.java` | 서버 `auth/service/UnverifiedUserCleaner.java` |
| 13 | `application*.yaml` | 설정 |
| **15-A** | **`SerialNumberProtector.java`** 중 지문 생성(HMAC-SHA256) | 서버 `product/service/SerialNumberProtector.java` |
| **15-B** | **`SerialNumberProtector.java`** 중 암호화·복호화(AES-GCM) | 서버 `product/service/SerialNumberProtector.java` |
| 15-A, 15-B | `ProductService.java`, `ProductRepository.java` | 서버 `product/service/`, `product/repository/` |
| 15-D | `LocalFileStorage.java` (HMAC 서명 URL) | 서버 `common/storage/LocalFileStorage.java` |

> 💡 **15번(15-A 지문 생성 HMAC, 15-B 암호문 AES-GCM)은 `auth/` 폴더가 아니라 `product/` 폴더**에 있다. 회원가입·로그인과는 별개 기능(제품 일련번호 보호)이다. 단, 같은 HMAC-SHA256 알고리즘을 로그인 토큰(JWT HS256)도 쓰므로 15-D에서 함께 비교한다.

---

## 0. 먼저 알아둘 용어

| 용어 | 쉬운 설명 |
|---|---|
| JWT | 서버가 서명해서 만든 "이 사람은 누구다"라는 문자열. 위조하면 서명이 깨져서 들통난다. |
| Access Token | 15분짜리 출입증(JWT). API를 부를 때마다 낸다. |
| Refresh Token | 14일짜리 재발급권. 무작위 문자열이며 서버(Redis)에 저장한다. 한 번 쓰면 폐기된다. |
| HttpOnly 쿠키 | 브라우저가 알아서 서버에 보내지만 JavaScript는 읽을 수 없는 쿠키. XSS로 토큰을 훔치기 어렵다. |
| SameSite=Strict | 다른 사이트에서 출발한 요청에는 이 쿠키를 붙이지 않는 설정. CSRF 1차 방어. |
| CSRF | 다른 사이트가 사용자 몰래 우리 서버에 요청을 보내게 하는 공격. 쿠키 인증을 쓰면 반드시 막아야 한다. |
| XSRF-TOKEN 쿠키 | CSRF 방어용 값. JS가 읽어서 `X-XSRF-TOKEN` 헤더로 되돌려 보낸다(HttpOnly 아님). |
| Redis | 메모리 기반 저장소. 만료 시간(TTL)을 걸어 임시 데이터를 두기 좋다. |
| BCrypt | 비밀번호를 되돌릴 수 없게 해시하는 방식. 일부러 느리다. |
| Flyway | DB 구조 변경을 SQL 파일로 버전 관리하는 도구. |
| Supabase | 외부 서비스. 이 프로젝트는 **인증 메일을 보내는 용도로만** 빌려 쓴다. 진짜 계정은 우리 `users` 테이블이다. |
| DTO | 요청·응답 전용 데이터 상자. Entity를 밖으로 내보내지 않으려고 쓴다. |
| 필터(Filter) | 요청이 컨트롤러에 닿기 전에 거치는 검문소. 순서가 중요하다. |

---

## 1. 정보가 어디로 가는가 (전체 흐름)

### 1-1. 저장 위치 한눈에

| 정보 | 어디에 | 어떤 형태로 | 얼마나 |
|---|---|---|---|
| 이메일, 닉네임, 역할, 인증 여부 | PostgreSQL `users` 테이블 | 그대로(이메일은 소문자로 정규화) | 영구 |
| 비밀번호 | `users.password_hash` | BCrypt 해시(평문은 어디에도 저장 안 함) | 영구 |
| Access Token | 브라우저 쿠키 `access_token` (HttpOnly, path `/`) | JWT. **서버는 저장하지 않음**(서명만 검증) | 15분 |
| Refresh Token | 브라우저 쿠키 `refresh_token` (HttpOnly, path `/api/auth`) + Redis `auth:refresh:<SHA-256>` | 쿠키엔 원문, Redis엔 **해시를 키로**, 값은 userId | 14일 |
| 로그아웃한 Access Token | Redis `auth:access:blocked:<jti>` | jti만 | 토큰 남은 시간만큼 |
| 로그인 실패 횟수 | Redis `auth:login:fail:email:<이메일>` / `auth:login:fail:ip:<IP>` | 숫자 | 첫 실패 후 15분 |
| 인증 메일 재발송 쿨다운 | Redis `auth:email:cooldown:<이메일>` | `"1"` | 60초 |
| Supabase 임시 계정 id | Redis `auth:email:pending:<이메일>` | Supabase 사용자 id | 1시간(메일 보낼 때마다 갱신) |
| 가입·재전송·갱신 요청 횟수 | Redis `rate:auth-signup:ip:<IP>` 등 | 숫자 | 10분(갱신은 1분) |
| CSRF 토큰 | 브라우저 쿠키 `XSRF-TOKEN` (HttpOnly 아님) | 무작위 값 | 세션 쿠키 |
| 인증 메일 발송용 임시 계정 | Supabase Auth | 이메일 + 아무도 모르는 무작위 비밀번호 | 인증 끝나면 삭제 |

### 1-2. 회원가입

```
브라우저(signup.html 폼)
  │  auth.js handleSignup: 입력 검사 → POST /api/auth/signup  (JSON + X-XSRF-TOKEN 헤더)
  ▼
[필터] CSRF 검사 → CookieBearerTokenFilter(이 요청은 쿠키 무시) → permitAll
[인터셉터] RateLimitInterceptor: IP당 10분에 10번까지 (Redis)
  ▼
AuthController.signup → @Valid 로 SignupRequest 검증 (이메일·비밀번호·닉네임 형식)
  ▼
AuthService.signup
  ├─ 이메일 소문자·공백 정리, 닉네임 공백 정리
  ├─ users 에 같은 이메일/닉네임(대소문자 무시)이 있으면 409
  ├─ 비밀번호를 BCrypt 해시 → users 에 INSERT (email_verified = false)   ← DB
  └─ EmailVerificationService.sendLink
        ├─ Redis 쿨다운 키 SET NX 60초 (이미 있으면 429)               ← Redis
        ├─ Supabase POST /auth/v1/signup (임시 계정 생성 → 확인 메일 발송) ← 외부
        └─ 임시 계정 id 를 Redis pending 키에 1시간 저장                 ← Redis
        (실패하면 방금 만든 users 행을 삭제하고 오류를 그대로 응답)
  ▼
201 Created {id, email, nickname, verificationTtlMinutes}  (비밀번호·토큰은 없음)
  ▼
브라우저: 폼 비움 → 로그인 화면으로 이동, "메일 인증 후 로그인하세요" 안내
```

### 1-3. 이메일 인증 (사용자가 메일 링크를 누를 때)

```
사용자 메일함의 링크 클릭 → Supabase 서버가 "이 이메일 확인됨"으로 표시 → /email-verified.html?result=ok 로 이동
(우리 서버는 이 순간 아무것도 모른다. 로그인 때 Supabase에 물어봐서 알게 된다.)
```

### 1-4. 로그인

```
브라우저(auth.html 폼) → auth.js handleLogin → POST /api/auth/login {email, password}
  ▼
[필터] CSRF 검사 → 쿠키 무시(POST /api/auth/*) → permitAll
  ▼
AuthController.login  (접속 IP = request.getRemoteAddr())
  ▼
AuthService.login
  ├─ 이메일 정규화
  ├─ LoginRateLimiter.checkAllowed : 이메일 5회 / IP 20회 실패 기록이 있으면 429 (비밀번호 비교도 안 함)   ← Redis
  ├─ users 에서 이메일 조회 (없으면 가짜 해시로 대신 비교 → 응답 시간 동일)                                ← DB
  ├─ BCrypt matches. 틀리면 실패 횟수 +1 후 401 "이메일 또는 비밀번호가 올바르지 않습니다"                 ← Redis
  ├─ 맞으면 이메일 실패 횟수 삭제
  ├─ 아직 미인증이면 Supabase에 확인 여부 조회 → 확인 안 됐으면 400 AUTH_008, 됐으면 users.email_verified = true  ← 외부+DB
  └─ issueTokens
        ├─ Access Token(JWT, HS256, 15분) 생성 : sub=userId, jti=UUID, scope=USER/ADMIN
        └─ Refresh Token(난수 32바이트) 생성 → Redis 에 SHA-256 해시 키로 저장                                ← Redis
  ▼
AuthController.withTokenCookies
  └─ Set-Cookie: refresh_token (path /api/auth, 14일), access_token (path /, 15분)  — 둘 다 HttpOnly·Strict
  └─ 본문: {expiresIn, user{id,email,nickname}}   (토큰은 본문에 없음)
  ▼
브라우저: setSession(user) → 헤더에 "닉네임님 / 로그아웃" 표시, 'pastport:auth-changed' 이벤트 → 홈으로 이동
```

### 1-5. 로그인 이후 API 호출이 인증되는 길

```
브라우저가 쿠키를 자동 첨부 → CsrfFilter(쓰기 요청이면 헤더 검사) → CookieBearerTokenFilter(쿠키를 Authorization 헤더로 변환)
→ BearerTokenAuthenticationFilter(JWT 서명·만료 검증 + 로그아웃 차단 목록 조회[Redis]) → 컨트롤러(@AuthenticationPrincipal Jwt)
```

---

## 2. 리뷰 요약

**꼭 확인 (실제 문제가 될 수 있음)**
1. ⚠️ **닉네임 대소문자 중복** — 서비스는 `existsByNicknameIgnoreCase`로 막지만 DB 유니크(`uk_users_nickname`)는 대소문자를 구분한다. 동시에 `Kim`/`kim`이 가입하면 DB가 못 막는다. 대소문자 무시 유니크 인덱스를 새 Flyway 파일로 추가해야 완성된다. (7번)
2. ⚠️ **이메일별 5회 잠금으로 남의 계정을 잠글 수 있다** — 남이 내 이메일로 5번 틀리면 15분 동안 나도 로그인이 안 된다. (9번)
3. ⚠️ **클라이언트 IP 판별** — `request.getRemoteAddr()` + `forward-headers-strategy: framework`. 배포 프록시 구성이 맞지 않으면 IP 제한을 우회당하거나 모든 사용자가 한 IP로 묶인다. 가입 횟수 제한도 같은 IP를 쓴다. (4-4, 5번)

**권장**
4. 💡 가입은 `DUPLICATE_EMAIL`(409)로 이메일 존재 여부를 알려 주는데 재전송 API는 숨긴다. 정책을 하나로 정하기.
5. 💡 `LoginRateLimiter`는 Redis가 죽으면 로그인이 500으로 실패(fail-closed)하는데, `RequestRateLimiter`는 통과시킨다(fail-open). 의도라면 주석으로 남기기.
6. 💡 `signup.html`의 "30분" 문구가 하드코딩이다. 서버 설정(`app.auth.unverified-user-ttl`)을 바꾸면 어긋난다. 로그인 화면 안내는 서버 값을 쓰므로 이쪽만 다르다.
7. 💡 로그인 중 `confirmIfVerified`가 Supabase 임시 계정과 대기 기록을 먼저 지운 뒤 DB `save`를 한다. 그 사이 저장이 실패하면 사용자가 "인증 안 됨" 상태에 갇혀 메일을 다시 받아야 한다.
8. 💡 `UnverifiedUserCleaner`가 `@Transactional` 안에서 사용자마다 외부 HTTP를 부른다. 미인증 계정이 많이 쌓이면 트랜잭션이 길어진다.

**사소**
9. 💡 로그인·가입 화면 문구가 아직 "내 노트북"이다(서비스는 가전 전체로 넓어짐). "이메일 찾기/비밀번호 찾기"는 눌러도 동작하지 않는 글자다.
10. 💡 `JwtProperties.refreshCookieSecure` 이름이 access 쿠키에도 쓰인다(주석으로 이유는 적혀 있음).

**이전 리뷰에서 해결된 항목 ✅** — `signup`/`login`의 `@Transactional`(외부 호출 중 DB 연결 점유)은 제거됐고, `LoginRateLimiter`는 만료 없는 키를 다시 고치는 코드가 들어갔으며, 닉네임은 대소문자 무시 검사가 추가됐고, 프런트 `handleLogin`의 `resend` null 처리도 들어갔다.

---

## 3. 프런트엔드 — 화면과 `auth.js`

### 3-1. `pages/signup.html` (회원가입 화면)

> 🔎 **선택 근거 — 화면 기술**
> - **쓴 것:** 빌드 없는 순수 HTML + 바닐라 JS(SPA 라우팅). 기존 프로젝트 구조가 이미 이 방식이고, 정적 파일을 Spring이 그대로 서빙한다.
> - **대안:** React·Vue·Next.js(컴포넌트·상태 관리가 편함), Thymeleaf(서버 렌더링).
> - **안 쓴 이유:** 새 빌드 도구(Node, 번들러)와 배포 단계가 생기고, 팀이 이미 만든 화면을 전부 다시 써야 한다. AGENTS.md도 요청 없는 대규모 리팩터링을 금지한다. 화면 수가 늘면 재검토 대상.
> - **입력 검증을 두 번 하는 이유:** 화면(`pattern`, `required`)은 사용자 편의용, 서버(`@Valid`)는 보안용. 화면 검증만 믿으면 curl로 우회된다.

화면 조각 파일이다. `pages.js`가 이 HTML을 `index.html`의 `#signup` 영역에 끼워 넣는다.

- **L1** `<div class="au split">` — 로그인/가입 공통 2단 레이아웃(왼쪽 소개, 오른쪽 폼).
- **L2** `<aside class="au-side">` — 왼쪽 소개 영역. 접근성용 `aria-label`.
- **L3** 브랜드 로고(`P` 마크 + Pastport + PRODUCT PASSPORT).
- **L4-5** 제목과 부제.
- **L6-10** 가입 후 이용 순서 3단계 목록(`ol.au-steps`). 장식·안내용이라 로직 없음.
- **L11** `<div data-auth-art>` — 일러스트 자리표시자. `auth.js` L268-273이 `auth-art.html`로 바꿔 끼운다.
- **L12** 개인정보 안내 문구.
- **L15** 오른쪽 폼 영역 시작.
- **L16-18** 영문 소제목 JOIN, 제목 "회원가입", 설명. 💡 L18 "내 노트북의 기록" 문구.
- **L19** `<form data-auth-form="signup" novalidate>` — `data-auth-form` 값으로 `auth.js`가 이 폼을 가입 폼으로 인식한다. `novalidate`는 브라우저 기본 검증 팝업을 끄고 우리 코드가 메시지를 보여 주겠다는 뜻.
- **L20** `<p class="au-msg" role="alert" hidden>` — 오류·안내 메시지 자리. `role="alert"`라 스크린리더가 바로 읽는다.
- **L21** 이메일 입력. `type="email"`, `autocomplete="username"`(비밀번호 관리자 연동), `maxlength="254"`(서버 `@Size(max=254)`와 맞춤). ⚠️ 안내문 "30분"이 하드코딩.
- **L22** 비밀번호 입력. `autocomplete="new-password"`(브라우저가 새 비밀번호 생성 제안), `maxlength="64"`.
- **L23** 규칙 체크 표시 4개(`data-rule` = length/letter/digit/special). `auth.js`가 입력하는 동안 `ok` 클래스를 켠다.
- **L24** 비밀번호 확인 입력과 일치 여부 표시칸(`<small data-match>`).
- **L25** 닉네임 입력. `maxlength="12"`(서버 규칙 2~12자). 💡 한글·영문·숫자·밑줄 규칙은 화면에서 미리 안내하지 않아 서버 응답을 받아야 안다.
- **L26-30** 약관 동의. `data-agree-all`(전체 동의)과 `data-agree-required`(필수 두 개).
- **L31** 제출 버튼 `type="submit"` — 눌러야 `auth.js`의 submit 이벤트가 동작한다.
- **L32** 로그인 화면으로 가는 버튼(`data-page="auth"` → `app.js`의 화면 전환).

### 3-2. `pages/auth.html` (로그인 화면)

- **L1-8** 가입 화면과 같은 왼쪽 소개 영역. 💡 L4 문구가 "내 노트북의 모든 기록"이라 가전 확장 방향과 어긋난다.
- **L13** `<form data-auth-form="login" novalidate>` — 로그인 폼 표식.
- **L14** 메시지 영역.
- **L15** `[data-email-resend]` 버튼 "인증 메일 다시 받기" — 처음엔 `hidden`. 에러 코드 `AUTH_008`일 때만 `auth.js`가 보여 준다.
- **L16** 이메일 입력(`autocomplete="username"`).
- **L17** 비밀번호 입력(`autocomplete="current-password"`, `maxlength="64"`).
- **L18** 로그인 제출 버튼.
- **L19** `이메일 찾기/비밀번호 찾기` — 💡 지금은 글자만 있고 기능이 없다.
- **L21** 회원가입으로 가는 버튼.

### 3-3. `js/auth.js` (로그인·가입 동작)

> 🔎 **선택 근거 — 토큰을 JS가 만지지 않는 방식**
> - **쓴 것:** `fetch`로 요청만 보내고, 토큰은 서버가 `HttpOnly` 쿠키로 내려준다. JS는 토큰 값을 읽지도 저장하지도 않는다.
> - **대안:** 응답 본문의 토큰을 `localStorage`/`sessionStorage`에 저장하고 `Authorization: Bearer` 헤더로 보내는 방식(구현이 단순, 모바일·타 도메인 API에 편함).
> - **안 쓴 이유:** XSS 한 번이면 `localStorage`의 토큰이 통째로 탈취된다. HttpOnly 쿠키는 JS에서 읽을 수 없어 피해가 작다. 대신 쿠키는 CSRF 위험이 생기므로 `SameSite=Strict` + XSRF 헤더로 막는다(4-1~4-3).
> - **`fetch` vs Axios:** 브라우저 내장 `fetch`로 충분해 라이브러리를 추가하지 않았다.

- **L1-2** 주석 — 화면 HTML이 나중에 채워지므로 `document`에 이벤트를 위임한다. 토큰은 전부 HttpOnly 쿠키라 JS가 못 읽고, 쓰기 요청은 `XSRF-TOKEN` 쿠키 값을 `X-XSRF-TOKEN` 헤더로 보낸다.
- **L3** `const auth = (() => { ... })();` — 즉시 실행 함수. 안의 변수가 전역에 새지 않는다.
- **L4** `let user = null;` — 로그인한 사용자(메모리에만 존재). 새로고침하면 사라지고 L276의 `refresh()`가 복구한다.
- **L6-11** `csrfHeaders(method)`
  - **L8** 메서드가 없거나 `GET/HEAD/OPTIONS`(읽기)면 빈 객체 — CSRF 검사가 필요 없다.
  - **L9** `document.cookie`에서 `XSRF-TOKEN=값`을 정규식으로 찾는다.
  - **L10** 찾으면 `X-XSRF-TOKEN` 헤더 객체를, 없으면 빈 객체. `decodeURIComponent`로 쿠키 인코딩을 푼다.
- **L14** `goto(page)` — `data-page="..."` 버튼을 찾아 클릭해서 화면을 이동시킨다(`app.js`의 `show()`가 실행됨).
- **L17-33** `request(path, options)` — 인증 API 공통 호출 함수
  - **L18-22** `fetch` 호출. 기본 메서드 `POST`, 헤더는 JSON + CSRF 헤더 + 추가 헤더, 본문은 `JSON.stringify`.
  - **L23** `204`(내용 없음)면 `null` 반환 — 로그아웃 등.
  - **L24** 응답 JSON을 읽되 실패하면 `null`(JSON이 아닐 때 대비).
  - **L25-31** 실패 응답이면 `Error`를 만들어 `status`, `code`(예: `AUTH_008`), `fieldErrors`를 붙여 던진다. 메시지는 서버 `ErrorResponse.message`.
  - **L32** 성공 시 `payload.data`(서버 `ApiResponse`의 `data`)를 반환.
- **L36-61** `renderHeader()` — 로그인 여부에 따라 헤더 오른쪽을 다시 그린다.
  - **L37-39** `.landing-account` 영역을 찾아 비운다.
  - **L40-49** 로그인 상태: `닉네임님` 버튼(마이페이지 이동)과 `로그아웃` 버튼(`data-auth-logout`)을 만든다. ✅ `textContent`로 닉네임을 넣어 **XSS가 불가능**(innerHTML을 안 씀).
  - **L50-60** 비로그인: `로그인 | 회원가입` 버튼.
- **L65-83** `renderMyPage()` — 마이페이지에 로그인 여부별 영역을 보이고 닉네임·이메일을 채운다. 역시 `textContent`.
- **L86-91** `setSession(data)` — 로그인 응답(또는 `null`)으로 `user`를 갱신하고 화면을 다시 그린 뒤 `pastport:auth-changed` 이벤트를 쏜다(다른 화면이 로그인 변화에 반응하도록, `userId` 포함).
- **L94** `clearSession` — `setSession(null)`.
- **L97-103** `message(form, text, kind)` — 폼의 `.au-msg`에 문구를 넣고, 빈 문자열이면 숨긴다. `kind`(error/info)로 색을 바꾼다.
- **L106-109** `setBusy` — 요청 중 제출 버튼을 잠가 **이중 제출을 막는다**.
- **L112-117** `passwordRules` — 길이(8~64), 영문, 숫자, 특수문자(영문·숫자 아니면서 전체가 `\x21-\x7E`)를 각각 true/false로 계산. ✅ 서버 `SignupRequest` 정규식과 같은 규칙.
- **L120-144** `handleLogin(form)`
  - **L121-122** 이메일은 `trim()`, 비밀번호는 그대로(공백도 비밀번호의 일부일 수 있어서).
  - **L123** 비어 있으면 서버에 보내지 않고 안내.
  - **L124-125** 재발송 버튼을 일단 숨긴다(이전 오류의 흔적 제거). ✅ `if (resend)`로 null 대비.
  - **L126** 제출 버튼 잠금.
  - **L128** `POST /api/auth/login`.
  - **L129** 성공하면 비밀번호 칸 비움(화면에 오래 두지 않기).
  - **L130-131** 메시지 지우고 `setSession`.
  - **L132-135** 이동 결정 — 주소에 `listingId`가 있으면(판매글 보다가 로그인한 경우) 그 글로, `myListings=1`이면 내 판매글로, 아니면 홈(`landing`)으로.
  - **L136-140** 실패: 비밀번호 비우고 서버 메시지 표시. 오류 코드가 `AUTH_008`(미인증)일 때만 "인증 메일 다시 받기" 버튼을 보인다. ✅ 이 코드는 서버가 **비밀번호가 맞은 뒤에만** 내므로 이 버튼으로 계정 존재 여부가 드러나지 않는다.
  - **L141-143** `finally`에서 버튼 잠금 해제.
  - 💡 `'AUTH_008'` 코드가 문자열 하드코딩이다(서버 `ErrorCode.EMAIL_NOT_VERIFIED`와 짝).
- **L147-154** `refresh()` — Refresh 쿠키로 세션 복구. `refreshing ??=`로 **진행 중이면 같은 요청을 같이 기다린다**. Refresh Token은 1회용이라 동시에 두 번 보내면 한쪽이 실패하기 때문. ✅ 좋은 처리.
- **L158-174** `authFetch` — 다른 화면이 쓰는 공통 요청 함수.
  - **L159-163** 요청 보내기: CSRF 헤더 추가, `credentials: 'same-origin'`(같은 사이트 쿠키 포함).
  - **L165** 새로고침 직후 세션 복구가 진행 중이면 끝나길 기다린다.
  - **L167** 401이 아니면 그대로 반환.
  - **L168-172** 401이면 한 번 `refresh()` 후 재전송. 갱신도 실패하면 원래 401을 돌려준다.
  - **L175** `window.PastportApi = { fetch: authFetch }` — 전역에 공개.
- **L178-208** `handleSignup(form)`
  - **L179-181** 값 읽기(이메일·닉네임은 `trim`).
  - **L182** 필수 항목 검사.
  - **L183-185** 비밀번호 규칙 4개 모두 충족하는지.
  - **L186** 비밀번호 확인 일치.
  - **L187-189** 필수 약관이 모두 체크됐는지.
  - **L190** 버튼 잠금.
  - **L192** `POST /api/auth/signup` — 본문 `{email, password, nickname}`. (`passwordConfirm`은 서버에 보내지 않는다.)
  - **L193-195** 폼 초기화, 일치 표시 지우기, 메시지 지우기.
  - **L196-200** 로그인 폼에 이메일을 미리 채우고 "가입 완료 / 메일 인증 후 로그인 / N분 안에 인증 안 하면 취소" 안내를 `info`로 표시. N은 서버가 준 `verificationTtlMinutes`(없으면 30).
  - **L201** 로그인 화면으로 이동.
  - **L202-204** 실패: 항목별 오류(`fieldErrors[0].message`)가 있으면 그것을, 없으면 `error.message`.
  - **L205-207** 버튼 잠금 해제.
- **L210-216** 문서 전체 `submit` 이벤트 위임 — `data-auth-form` 폼이면 기본 제출(페이지 새로고침)을 막고 종류에 맞는 핸들러 호출. 화면이 나중에 로드돼도 동작한다.
- **L220-227** `showPasswordMatch` — 확인 칸이 비어 있으면 아무 표시 없음, 아니면 "일치해요/일치하지 않아요".
- **L229-237** `input` 이벤트 — 비밀번호 칸 입력 시 규칙 표시(`ok` 클래스)와 일치 표시를 즉시 갱신.
- **L239-243** "전체 동의" 체크 시 필수 약관 전체 체크.
- **L245-256** 인증 메일 재전송 버튼 — 이메일이 비면 안내, 아니면 `POST /api/auth/email/resend`. 성공하면 "메일을 보냈어요"(서버는 계정이 있든 없든 204라서 이 문구도 계정 존재를 알려 주지 않는다 ✅).
- **L257-264** 로그아웃 버튼 — 서버 호출이 실패해도 **화면에서는 무조건 로그아웃** 처리하고 홈으로.
- **L267-273** 로그인·가입 화면의 일러스트를 `auth-art.html`에서 한 번 받아 자리표시자와 교체. 마이페이지가 로드되면 `renderMyPage()`.
- **L276** `refresh().catch(() => {})` — 페이지가 열릴 때 Refresh 쿠키가 있으면 세션을 복구. 없으면 401이 나지만 조용히 무시한다. 💡 비로그인 방문자도 매번 `/api/auth/refresh`를 한 번 호출한다(서버 쪽 제한은 IP당 분당 30회).
- **L278** `return { get user() { return user; } }` — 읽기 전용으로 `user`만 공개.

---

## 4. 요청이 서버에 들어오는 순서 (필터·인터셉터)

### 4-1. `SecurityConfig.java` (보안 설정의 중심)

> 🔎 **선택 근거 — 인증 방식 전체**
> - **쓴 것:** Spring Security + 무상태(Stateless) JWT Access Token(15분) + Redis의 Refresh Token. 서버 메모리에 로그인 상태를 두지 않는다.
> - **대안 ① 서버 세션(HttpSession / Spring Session + Redis):** 구현이 가장 단순하고 즉시 폐기가 쉽다. → 안 쓴 이유: 요청마다 세션 조회가 필요하고, 모바일·다른 서비스(채팅 등)가 늘면 확장이 불편. AGENTS.md가 "Spring Security + JWT, Refresh Token은 Redis"로 정해 두었다.
> - **대안 ② Opaque 토큰(무작위 문자열을 매번 Redis 조회):** 즉시 폐기 가능. → 안 쓴 이유: 모든 요청이 Redis를 거친다. JWT는 서명만 검증하면 되어 빠르다.
> - **대안 ③ 외부 인증 서버(Keycloak, Auth0, Supabase Auth 로그인까지 위임):** 직접 만들 코드가 줄어든다. → 안 쓴 이유: 사용자 테이블·권한·차단이 이미 자체 DB에 있고, 외부 의존이 커진다. (Supabase는 메일 인증에만 제한적으로 사용.)
> - **CSRF를 끄지 않은 이유:** `csrf.disable()`은 Bearer 헤더만 쓰는 API에서만 안전하다. 이 프로젝트는 쿠키로 인증하므로 켜 두어야 한다.

- **L3-34** import — Nimbus JWT 서명 키 소스(`ImmutableSecret`), 문자 인코딩, 비밀 키 클래스, `@Bean`/`@Configuration`, 설정 바인딩, `HttpMethod`, `Customizer`, `HttpSecurity`, 세션 정책, `BCryptPasswordEncoder`, `PasswordEncoder`, OAuth2 토큰 검증 관련 클래스들, `MacAlgorithm`, `Jwt`·`JwtDecoder`·`JwtEncoder` 및 Nimbus 구현, `BearerTokenAuthenticationFilter`, `SecurityFilterChain`, `AccessDeniedHandlerImpl`, CSRF 저장소·핸들러, 보안 헤더 작성기, `SessionManagementFilter`. ✅ 와일드카드 없음.
- **L36** `@Configuration` — 설정 클래스.
- **L37** `@EnableConfigurationProperties(JwtProperties.class)` — `app.jwt.*` 설정을 `JwtProperties`로 묶는다.
- **L40** `ACCESS_TOKEN_COOKIE = "access_token"` — 쿠키 이름 상수. 컨트롤러와 필터가 함께 쓴다.
- **L43-44** `securityFilterChain` — 요청이 지나는 보안 규칙 전체.
- **L45-50** CSRF 설정.
  - **L47** `CookieCsrfTokenRepository.withHttpOnlyFalse()` — CSRF 토큰을 `XSRF-TOKEN` 쿠키로 내린다. JS가 읽어야 하므로 HttpOnly를 끈다(인증 토큰이 아니라 안전).
  - **L50** `CsrfTokenRequestAttributeHandler` — 값을 변형(마스킹)하지 않는다. JS가 쿠키 값을 그대로 헤더에 담기 때문.
  - L48-49 주석: Authorization 헤더가 있는 요청(API 클라이언트·Swagger)은 Spring이 기본으로 CSRF에서 뺀다.
- **L52** `CsrfCookieFilter`를 `SessionManagementFilter` **뒤**에 추가 — 인증이 끝난 뒤 새 CSRF 쿠키를 내리기 위해(주석 L51).
- **L54** `CookieBearerTokenFilter`를 `BearerTokenAuthenticationFilter` **앞**에 추가. ✅ 순서가 핵심: CSRF 검사가 **먼저** 끝나야 쿠키 인증 요청도 CSRF 검사를 받는다(주석 L53).
- **L56** CSRF 검사 실패 시 401이 아니라 **403**으로 응답.
- **L57-68** 보안 응답 헤더 — HSTS(HTTPS 1년, 하위 도메인 포함), Referrer-Policy `same-origin`, Permissions-Policy(위치·마이크 차단). ✅
- **L69** `cors(withDefaults())` — CORS 기본 설정(허용 출처는 `CorsConfig`가 관리).
- **L70-71** `STATELESS` — 서버 세션을 만들지 않는다(토큰 방식).
- **L72-117** 접근 규칙(위에서부터 먼저 맞는 규칙 적용)
  - **L75-92** 화면·정적 파일·WebSocket 핸드셰이크·API 문서는 누구나 접근. 💡 `/swagger-ui/**`, `/v3/api-docs/**`가 운영에서도 열려 있다. 운영 공개 여부 확인.
  - **L93-98** `/api/`로 시작하지 않는 GET은 모두 허용 — 없는 경로가 401이 아닌 404로 응답하도록(주석 L93).
  - **L99-105** 공개 GET API(매물 목록·썸네일·이전 검증·제품 이벤트).
  - **L106-115** **로그인 전에 불러야 하는 POST**: 재전송·가입·로그인·갱신·로그아웃(+AI 두 개).
  - **L116-117** 나머지는 전부 인증 필요. ✅ 기본값이 "막는다".
- **L118** `oauth2ResourceServer(jwt)` — `Authorization: Bearer ...`의 JWT를 검증해 인증 처리.
- **L119** 필터 체인 완성.
- **L122-139** `jwtDecoder` — 토큰 검증기
  - **L125-128** 비밀 키로 HS256 서명을 검증하는 디코더.
  - **L130-135** 차단 목록(로그아웃된 토큰)에 `jti`가 있으면 "로그아웃된 토큰입니다" 오류. 서명·만료가 멀쩡해도 막는다. ✅
  - **L136-137** 기본 검증(만료 등) + 차단 검증 둘 다 적용.
  - 💡 인증 요청마다 Redis를 1번 조회한다. Redis 장애 시 모든 인증 요청이 실패(fail-closed)한다 — 보안상 합리적이지만 의도를 주석으로 남기면 좋다.
- **L142-145** `jwtEncoder` — 토큰 서명기.
- **L148-150** `secretKey` — 설정의 비밀 문자열을 UTF-8 바이트로 바꿔 HS256 키 생성.
- **L153-156** `passwordEncoder` — `BCryptPasswordEncoder`(기본 강도 10). ✅ 표준.

### 4-2. `CsrfCookieFilter.java`

- **L11** 주석 — 인증 과정에서 교체된 CSRF 토큰을 응답 전에 읽어 `XSRF-TOKEN` 쿠키를 발급한다.
- **L12** 패키지 전용 클래스, `OncePerRequestFilter`(요청당 한 번 실행).
- **L15-18** `doFilterInternal`.
- **L19** 요청에 달린 `CsrfToken` 객체를 꺼낸다.
- **L20-22** 있으면 `getToken()`을 호출한다 — Spring의 CSRF 토큰은 **쓰일 때 지연 생성**되므로 이 호출이 쿠키 발급을 일으킨다. 그래서 처음 화면을 받을 때도 쿠키가 내려가 로그인 POST가 막히지 않는다.
- **L23** 다음 필터로 진행.

### 4-3. `CookieBearerTokenFilter.java`

> 🔎 **선택 근거 — 쿠키를 Bearer로 바꿔 주는 필터**
> - **쓴 것:** 쿠키의 `access_token`을 읽어 Spring Security의 표준 Bearer 인증 흐름에 태운다.
> - **대안:** 커스텀 `OncePerRequestFilter`로 토큰을 직접 파싱·인증 객체 생성 / `BearerTokenResolver`만 교체.
> - **안 쓴 이유:** 서명·만료 검증을 직접 구현하면 실수 위험이 크다. 표준 `BearerTokenAuthenticationFilter`와 `JwtDecoder`를 재사용해 검증 코드를 우리가 쓰지 않는 쪽이 안전하다.

- **L3-16** import — 서블릿 필터/요청 래퍼, `Cookie`, 컬렉션들, `HttpHeaders`, `OncePerRequestFilter`, `WebUtils`.
- **L18-21** 주석 — 반드시 CSRF 필터 뒤. Spring은 Bearer 헤더가 있는 요청을 CSRF에서 빼므로, 쿠키를 처음부터 Bearer로 보이게 하면 CSRF 검사를 건너뛰게 된다.
- **L22** 패키지 전용 클래스.
- **L25-31** `doFilterInternal` — 쿠키 토큰이 있으면 요청을 `BearerRequest`로 감싸 다음 필터에 넘긴다.
- **L34-54** `cookieToken` — 쿠키 토큰을 쓸지 결정
  - **L35-37** 이미 `Authorization` 헤더가 있으면 그것을 쓴다(쿠키 무시).
  - **L39-41** `/api/`가 아닌 경로(화면·정적 파일)는 쿠키를 쓰지 않는다 — 잘못된 쿠키가 남아도 화면이 막히지 않게.
  - **L43-47** 공개 매물 목록·썸네일 GET은 쿠키 무시 — 만료 쿠키가 있어도 비회원으로 볼 수 있다.
  - **L49-51** `POST /api/auth/*`(로그인·가입·갱신·로그아웃)도 쿠키 무시 — 만료 쿠키가 남아도 로그인·갱신이 막히지 않는다. ✅ 이 로그인 흐름에서 중요한 줄.
  - **L52-53** `access_token` 쿠키를 읽어 값이 있으면 반환, 없거나 빈 값이면 `null`.
  - 💡 공개 경로가 늘 때마다 `SecurityConfig`와 이 파일, 두 곳을 같이 고쳐야 한다. 한 곳에서 관리하면 누락을 막을 수 있다.
- **L56-89** `BearerRequest` — 요청의 헤더 조회를 가로채는 래퍼
  - **L58, 61-64** 쿠키로 만든 `"Bearer <토큰>"` 문자열 보관.
  - **L67-72** `getHeader("Authorization")` → 만든 값, 나머지는 원래대로.
  - **L75-80** `getHeaders(...)` 같은 방식.
  - **L83-88** `getHeaderNames()`에 `Authorization` 추가.

### 4-4. 가입·재전송·갱신 횟수 제한 — `WebMvcConfig`, `RateLimitInterceptor`, `RequestRateLimiter`

> 🔎 **선택 근거 — 횟수 제한 방식**
> - **쓴 것:** Redis 카운터(키 + TTL)를 인터셉터에서 확인.
> - **대안:** Bucket4j·Resilience4j RateLimiter 라이브러리, Guava/Caffeine 인메모리 카운터, Nginx·API 게이트웨이 제한, 클라우드 WAF.
> - **안 쓴 이유:** 인메모리는 서버가 2대 이상이면 횟수가 서버마다 따로 센다. Redis는 이미 Refresh Token용으로 쓰고 있어 새 의존성이 없다. 라이브러리는 이 정도 단순한 규칙(N분에 M번)에는 과하다. 인프라 단 제한은 운영 환경에 의존해 로컬·테스트에서 재현이 어렵다.
> - **한계:** Redis가 죽으면 이 제한은 통과시킨다(fail-open). 로그인 제한은 반대로 막는다(fail-closed). 정책 차이는 9-1 참고.

**WebMvcConfig.java**
- **L7-8** 설정 클래스. **L10-14** 인터셉터를 주입받는다.
- **L17-22** `/api/v1/**`, `/api/auth/signup`, `/api/auth/refresh`, `/api/auth/email/resend` 경로에만 인터셉터를 건다. 💡 `/api/auth/login`은 여기 없다 — 로그인은 `LoginRateLimiter`가 따로 제한한다.

**RateLimitInterceptor.java**
- **L16-18** 주석과 `@Component`. 컨트롤러 실행 **전에**(`preHandle`) 횟수를 센다.
- **L21-22** `Rule` — 이름(bucket), 메서드, 경로, 최대 횟수, 시간 창.
- **L50** `auth-signup`: `POST /api/auth/signup`, **10분에 10번**.
- **L51-56** `auth-email-resend`: 10분에 5번.
- **L63** `auth-refresh`: 1분에 30번.
- **L78-84** 경로 매처와 제한기 주입.
- **L88-97** `preHandle` — 규칙 중 메서드·경로가 맞는 첫 규칙으로 `check`를 호출하고 `break`. 항상 `true`(통과).
- **L101-107** `subject` — 로그인한 사용자면 `user:<id>`, 아니면 `ip:<getRemoteAddr()>`. ⚠️ 가입은 비로그인이라 항상 IP 기준. 같은 공유기 뒤 사용자들이 한도를 나눠 쓰고, 프록시 설정이 틀리면 전원이 하나로 묶인다.

**RequestRateLimiter.java**
- **L16-18** 로거, 키 접두사 `rate:`, `NO_EXPIRY = -1`.
- **L31-59** `check`
  - **L32** 키 = `rate:<버킷>:<대상>`.
  - **L36** `increment`(Redis INCR, 원자적 +1).
  - **L40-42** 첫 요청이면 만료 시간을 건다.
  - **L43-49** 한도를 넘었으면 남은 TTL 조회. TTL이 `-1`(만료 없음)이면 다시 건다 — INCR과 EXPIRE 사이 실패로 영구 차단되는 일을 막는 **자가 치유**. ✅
  - **L50-53** Redis 오류는 로그만 남기고 **통과**(fail-open). 보조 장치가 서비스를 멈추지 않게.
  - **L54-58** 한도 초과면 `RateLimitedException`(남은 시간 포함).

---

## 5. `AuthController.java` — 요청을 받는 입구

- **L1** 패키지 `auth.controller`.
- **L3** `HttpServletRequest` — 접속 IP를 꺼내려고.
- **L4** `@Valid` — 요청 본문 검증 지시.
- **L5** `Duration` — "15분/14일" 같은 시간 길이.
- **L6-12** DTO 7개: `AuthTokens`(서비스 결과), `EmailResendRequest`, `LoginRequest`, `LoginResponse`, `SignupRequest`, `SignupResponse`, `UserResponse`.
- **L13-14** `AuthService`, `EmailVerificationService`.
- **L15** `ApiResponse` — 응답을 `{success, data}`로 감싸는 공통 형식.
- **L16-17** `JwtProperties`, `SecurityConfig`(쿠키 이름 상수).
- **L18** `@Value` — 설정값 주입 표시.
- **L19-22** `HttpHeaders`(헤더 이름 상수), `HttpStatus`, `ResponseCookie`(쿠키 만들기), `ResponseEntity`(상태·헤더·본문을 직접 지정).
- **L23-24** `@AuthenticationPrincipal`, `Jwt` — 로그인한 사용자의 토큰.
- **L25-31** 웹 어노테이션들(쿠키 읽기, GET, POST, 본문, 헤더, 공통 주소, JSON 컨트롤러).
- **L33** `@RestController` — 반환값을 JSON으로 응답.
- **L34** `@RequestMapping("/api/auth")` — 아래 모든 주소 앞에 `/api/auth`.
- **L37** `REFRESH_COOKIE = "refresh_token"` — 쿠키 이름 상수.
- **L38** `REFRESH_COOKIE_PATH = "/api/auth"` — 이 쿠키는 `/api/auth` 아래 요청에만 전송된다. ✅ 노출 범위를 줄인 좋은 설계.
- **L40-43** `final` 필드 4개(서비스 둘, 설정, 미인증 유예 시간).
- **L45-55** 생성자(생성자 주입). L50 `@Value("${app.auth.unverified-user-ttl:30m}")` — 설정이 없으면 기본 30분. ✅ 프로젝트 컨벤션과 일치.
- **L57-62** `resendEmailLink` — `POST /api/auth/email/resend`. L59 `@Valid @RequestBody`로 이메일 형식 검사, L60 서비스에 위임, L61 **항상 204**. ✅ 계정이 있든 없든 같은 응답.
- **L64-71** `signup` — `POST /api/auth/signup`
  - **L67** 본문을 `SignupRequest`로 변환하며 검증(실패 시 `GlobalExceptionHandler`가 400).
  - **L68** 서비스가 가입 처리 후 `UserResponse` 반환.
  - **L69-70** `201 Created` + `SignupResponse`(회원 정보 + 인증 유예 분). ✅ Entity가 아니라 DTO, 비밀번호·토큰 없음.
- **L73-78** `login` — `POST /api/auth/login`
  - **L76** 본문과 `HttpServletRequest`를 받는다.
  - **L77** `getRemoteAddr()`로 접속 IP를 얻어 서비스에 넘기고, 결과를 `withTokenCookies`로 응답으로 만든다. ⚠️ `forward-headers-strategy: framework`(application.yaml L19-20)라서 프록시가 보낸 `X-Forwarded-For`를 Spring이 해석해 이 값에 반영한다. **신뢰할 수 없는 프록시가 앞에 있으면 헤더 위조로 IP 제한을 피할 수 있다.** 반대로 프록시 뒤인데 설정이 안 맞으면 모든 사용자가 같은 IP로 보인다.
- **L80-85** `refresh` — `POST /api/auth/refresh`. L83 `@CookieValue(required = false)`: 쿠키가 없어도 400이 아니라 서비스에서 일관된 401로 처리. ✅
- **L87-98** `logout`
  - **L90-92** Refresh 쿠키, Access 쿠키, Authorization 헤더를 모두 "없어도 됨"으로 받는다. ✅ 어떤 상태에서도 로그아웃된다.
  - **L93** 서비스에 Refresh 토큰과 Access 토큰(헤더 우선, 없으면 쿠키)을 넘긴다.
  - **L94-97** 204 응답에 두 쿠키를 `Duration.ZERO`(즉시 만료)로 덮어써 브라우저에서 지운다. ✅ 발급할 때와 같은 헬퍼를 써서 path·속성이 같아 실제로 삭제된다.
- **L100-104** `me` — `GET /api/auth/me`. L102 `@AuthenticationPrincipal Jwt jwt` — 인증된 토큰만 여기까지 온다(`anyRequest().authenticated()`). L103 `Long.parseLong(jwt.getSubject())` — 토큰의 `sub`(사용자 id)를 숫자로. 💡 다른 컨트롤러는 `CurrentUserIdResolver`를 쓰므로 여기만 스타일이 다르다.
- **L106-112** `bearerOrCookie`
  - **L108** `regionMatches(true, 0, "Bearer ", 0, 7)` — 헤더가 `Bearer `로 시작하는지 대소문자 무시하고 확인.
  - **L109** `substring(7).trim()` — 접두사를 떼고 토큰만.
  - **L111** 헤더가 없으면 쿠키 값.
- **L114-125** `withTokenCookies` — 로그인·갱신 응답 공통
  - **L116** 본문에는 만료 초와 회원 정보만. ✅ 토큰이 본문에 없어서 JS(와 XSS)가 읽을 수 없다.
  - **L117-124** `Set-Cookie` 헤더를 두 번 `.header()`로 추가 — Refresh 쿠키(14일), Access 쿠키(15분). 같은 이름의 헤더를 **덮어쓰지 않고 둘 다** 내려보낸다.
- **L127-130** `refreshCookie` — 경로 `/api/auth`.
- **L132-135** `accessCookie` — 경로 `/`(모든 요청에 전송).
- **L137-146** `httpOnlyCookie` — 공통 쿠키 생성
  - **L140** `httpOnly(true)` ✅ JS가 못 읽는다.
  - **L141** `secure(properties.refreshCookieSecure())` — HTTPS에서만 전송할지. 💡 이름이 refresh 전용처럼 보이지만 access 쿠키에도 쓴다(설정 키 호환 때문, `JwtProperties` 주석에 이유 있음).
  - **L142** `sameSite("Strict")` ✅ 다른 사이트에서 출발한 요청엔 쿠키를 안 보낸다. 같은 사이트 JS의 `fetch`는 정상 전송된다.
  - **L143-144** 경로와 유효 시간(`maxAge`).

---

## 6. DTO — 요청·응답 데이터 상자

> 🔎 **선택 근거 — DTO 분리**
> - **쓴 것:** 요청/응답마다 별도 DTO(Java record) + Bean Validation.
> - **대안:** `User` Entity를 그대로 요청·응답에 사용, `Map<String,Object>` 사용.
> - **안 쓴 이유:** Entity를 직접 받으면 `role`·`emailVerified` 같은 필드를 요청자가 마음대로 채우는 **Mass Assignment** 취약점이 생기고, 응답에는 `passwordHash`가 새어 나갈 수 있다. AGENTS.md도 Entity를 API로 직접 반환하는 것을 금지한다. record는 불변이라 중간에 값이 바뀌지 않는다.

### 6-1. `SignupRequest.java`
- **L3-6** 검증 어노테이션 import(`Email`, `NotBlank`, `Pattern`, `Size`).
- **L8** `record SignupRequest(` — 값을 담기만 하는 불변 클래스(Java record).
- **L9** `@NotBlank @Email @Size(max = 254) String email` — 필수, 이메일 형식, 최대 254자(이메일 표준 최대 길이). DB 컬럼도 254.
- **L10-14** `password`
  - **L12** `^(?=.*[A-Za-z])(?=.*\d)(?=.*[^A-Za-z\d])[\x21-\x7E]{8,64}$` — 영문 포함 / 숫자 포함 / 영문·숫자 아닌 문자 포함 / 전체가 눈에 보이는 ASCII(`!`~`~`)만 / 8~64자. 공백·한글은 불가.
  - **L13** 틀렸을 때 보여줄 메시지(프런트가 `fieldErrors[0].message`로 표시).
  - ✅ 최대 64자는 BCrypt의 72바이트 한도 안쪽, 그리고 ASCII만 허용하므로 글자 수 = 바이트 수라 안전.
- **L15-19** `nickname` — `^[가-힣A-Za-z0-9_]{2,12}$` 한글·영문·숫자·밑줄만 2~12자. DB `VARCHAR(12)`와 맞다.

### 6-2. `LoginRequest.java`
- **L6-7** `email`(필수, 최대 254), `password`(필수, 최대 64). 로그인엔 **형식 검사를 하지 않는다.** ✅ 가입 규칙이 바뀌어도 기존 사용자가 로그인할 수 있다. 최대 길이만 둬 아주 긴 입력으로 BCrypt를 괴롭히는 것을 막는다.

### 6-3. `EmailResendRequest.java`
- **L7** 이메일 하나. 필수 + 이메일 형식 + 254자 이하.

### 6-4. `SignupResponse.java`
- **L3** 가입 결과. 인증하지 않으면 계정이 정리되기까지 남은 유예 분을 안내용으로 함께 준다.
- **L4** `id, email, nickname, verificationTtlMinutes`.
- **L6-8** `of(UserResponse, long)` — `UserResponse`에 유예 분을 붙여 만든다.

### 6-5. `UserResponse.java`
- **L5** `id, email, nickname` — 밖으로 보여줄 최소 정보. ✅ 비밀번호 해시·역할은 제외.
- **L7-9** `from(User)` — Entity → DTO 변환 정적 메서드.

### 6-6. `LoginResponse.java`
- **L3** 주석 — 토큰은 쿠키로만 간다. **L4** `expiresIn`(Access 토큰 유효 초), `user`.

### 6-7. `AuthTokens.java`
- **L3** 서비스 내부 결과. Refresh Token은 응답 본문이 아니라 쿠키로만 나간다.
- **L4-5** `accessToken, expiresIn, refreshToken, user`. 💡 record의 `toString()`에 토큰이 그대로 찍히므로 **로그에 객체째 남기지 말 것**.

### 6-8. `SupabaseUserResponse.java` (Supabase 응답 수신용)
- **L3-4** Jackson(JSON 변환) 어노테이션 import.
- **L6** 가입 응답은 사용자 객체이거나 세션과 함께 `user` 안에 들어 있다는 설명.
- **L7** `@JsonIgnoreProperties(ignoreUnknown = true)` — 모르는 필드는 무시. ✅ 외부 API가 필드를 늘려도 깨지지 않는다.
- **L8-12** `id`, `email_confirmed_at`(이메일 확인 시각), `access_token`, `user`(중첩).
- **L15-17** `userId()` — `id`가 있으면 그것, 없으면 `user.id`, 둘 다 없으면 `null`.
- **L20-24** `confirmedWithoutMail()` — 토큰이 있거나 확인 시각이 있으면 "확인 메일 없이 바로 가입된 것"(Supabase의 Confirm email 설정이 꺼진 상태)으로 판단.

---

## 7. `AuthService.signup` — 회원가입의 핵심

> 🔎 **선택 근거 — 비밀번호 저장과 이메일 인증**
> - **비밀번호: BCrypt.** 일부러 느리게 만든 해시 + 매번 다른 salt가 자동 포함된다. Spring Security 기본 제공이라 추가 의존성이 없다.
>   - **대안:** Argon2(메모리 비용까지 요구해 더 강력, 최신 권장), scrypt, PBKDF2(표준·FIPS 요구 환경), SHA-256/MD5 단독.
>   - **안 쓴 이유:** SHA-256 계열은 너무 빨라서 GPU로 초당 수십억 번 대입된다 → 절대 비밀번호에 쓰지 않는다. Argon2는 더 좋지만 별도 라이브러리(Bouncy Castle) 추가가 필요해 현재 규모에서는 BCrypt로 충분하다고 판단. 이후 업그레이드 대상으로 기록.
> - **이메일 인증: Supabase 임시 계정으로 메일 발송.**
>   - **대안:** 직접 SMTP(`JavaMailSender`) + 자체 인증 토큰, AWS SES·SendGrid 등 메일 API, 인증 없이 가입.
>   - **안 쓴 이유:** 자체 SMTP는 메일 도달률(SPF·DKIM), 템플릿, 토큰 만료·재발급을 전부 직접 만들어야 한다. 이미 쓰는 Supabase의 인증 메일 기능을 재사용하면 구현이 짧다. 다만 외부 의존이 생겨서 장애 시 가입 흐름이 영향을 받는다(8번 참고).
> - **트랜잭션을 걸지 않은 이유:** 외부 HTTP 호출(Supabase)을 DB 트랜잭션 안에 두면 응답이 느릴 때 DB 커넥션을 오래 점유한다. 그래서 `save` 단건만 짧은 트랜잭션으로 둔다.

- **L3** `Locale` — 대소문자 변환 시 언어 규칙 고정용.
- **L4-7** 입출력 DTO. **L8-9** `User`, `UserRole`. **L10** `UserRepository`.
- **L11-12** `CommonException`, `ErrorCode` — 업무 오류와 오류 종류.
- **L13** `AccessTokenBlocklist` — 로그아웃 토큰 차단 목록.
- **L14-15** SLF4J 로거. **L16** `DataIntegrityViolationException` — DB 제약 위반 예외.
- **L17** `PasswordEncoder`. **L18-19** `JwtDecoder`, `JwtException`.
- **L20-21** `@Service`, `@Transactional`.
- **L23** `@Service` — 스프링이 관리하는 서비스 객체.
- **L26** 로거. **L29** `DUMMY_PASSWORD` — 없는 이메일로 로그인할 때도 해시 비교를 한 번 하게 하려는 가짜 비밀번호(주석 L28). ✅ 응답 시간 차이로 "이 이메일은 가입돼 있다"를 알아내는 **타이밍 공격**을 막는 정석.
- **L31-39** 필드 9개 — 회원 DB, 비밀번호 도구, 토큰 발급기, Refresh 저장소, 이메일 인증, 로그인 제한, 토큰 해석기, 차단 목록, `dummyHash`.
- **L41-60** 생성자(의존성 주입). **L59** `dummyHash = passwordEncoder.encode(DUMMY_PASSWORD)` — 시작할 때 BCrypt를 한 번 돌려 가짜 해시를 만들어 둔다. ✅ 요청마다 하지 않는다. 💡 테스트에서 `PasswordEncoder`를 mock으로 쓰면 이 값이 `null`이 된다.
- **L62-65** 주석 — 외부(Supabase) 호출이 DB 연결을 붙잡지 않도록 **트랜잭션으로 묶지 않고**, 저장소 호출마다 짧게 커밋한다. ✅ (이전에 지적된 DB 연결 풀 점유 문제의 해결.)
- **L66** `public UserResponse signup(SignupRequest request)`.
- **L67** `normalizeEmail` — 공백 제거 + 소문자. 같은 이메일이 다르게 취급되는 것을 막는다.
- **L68** `nickname.trim()` — 앞뒤 공백 제거(정규식이 공백을 허용하지 않으므로 사실상 방어용).
- **L69-71** 이미 있는 이메일이면 409 `DUPLICATE_EMAIL`. 💡 이메일 존재를 알려 주는 응답이다(재전송 API는 숨기는 것과 정책이 다름).
- **L72-74** `existsByNicknameIgnoreCase` — 대소문자 구분 없이 닉네임 중복이면 409 `DUPLICATE_NICKNAME`. ✅ 사칭 방지. ⚠️ 이 검사는 **앱 수준**일 뿐이다. DB 유니크 `uk_users_nickname`(V5)은 대소문자를 구분하므로, 두 사람이 `Kim`과 `kim`으로 **동시에** 가입하면 두 검사를 모두 통과하고 DB도 막지 못한다. 새 Flyway 파일로 `CREATE UNIQUE INDEX ... ON users (lower(nickname))`를 추가해야 완성된다(기존 `V5`는 수정 금지).
- **L76-77** 새 `User` 생성. `passwordEncoder.encode(...)`로 **해시해서** 넣는다. ✅ 평문을 저장하지 않는다. 역할은 `USER`, 인증 여부는 기본 `false`.
- **L78** `User saved;` — 저장 결과 변수.
- **L79-80** `saveAndFlush` — 저장하고 즉시 DB에 반영해 제약 위반을 **바로** 알 수 있게 한다. 이 호출 하나가 자체 트랜잭션으로 커밋된다.
- **L81** 저장 중 DB 제약 위반을 잡는다.
- **L82** 주석 — 동시 가입으로 L69-74 검사를 같이 통과해도 DB 유니크가 걸러 준다(경쟁 상태 대비). ✅
- **L83** 가장 구체적인 원인 메시지를 문자열로(`String.valueOf`로 `null` 안전).
- **L84-87** 메시지에 `uk_users_nickname`이 있으면 닉네임 중복, 아니면 이메일 중복으로 판정. 💡 **제약 이름을 문자열로 비교**하므로 Flyway에서 제약 이름을 바꾸면 판정이 어긋난다.
- **L89** 주석 — 메일을 못 보내면 방금 만든 계정을 지운다.
- **L90-91** `emailVerificationService.sendLink(email)` — 인증 메일 발송(아래 8번). 이제 트랜잭션 밖이라 Supabase가 느려도 DB 연결을 붙잡지 않는다.
- **L92-95** 어떤 `RuntimeException`이든 → `discardUnverified(saved)`로 users 행 삭제 후 **원래 예외를 다시 던진다**. 사용자는 메일 발송 실패(503)·너무 잦은 요청(429)을 그대로 본다.
- **L96** `UserResponse.from(saved)` — Entity → DTO.
- **L163-170** `discardUnverified`
  - **L166** `userRepository.delete(user)`.
  - **L167-169** 삭제가 실패해도 원래 오류를 가리지 않고 경고 로그만 남긴다. 남은 계정은 `UnverifiedUserCleaner`가 지운다. ✅ 로그에 이메일이 아닌 `userId`만 남긴다.
- **L181-184** `normalizeEmail` — `trim()` + `toLowerCase(Locale.ROOT)`. ✅ `Locale.ROOT`로 터키어 `I` 같은 지역 규칙 영향을 없앴다. `package-private static`이라 `EmailVerificationService`도 같은 규칙을 쓴다.

⚠️/💡 생각해 볼 지점
- 가입 중간에 서버가 죽어 users 행만 남으면(메일 미발송) → 30분 뒤 정리 작업이 지운다. 그 사이 해당 이메일·닉네임은 다른 사람이 쓸 수 없다.
- 가입 직전 60초 이내에 같은 이메일로 `sendLink`가 이미 불렸다면(예: 방금 가입 취소 후 재가입) 쿨다운에 걸려 429가 나고 새 계정도 삭제된다. 사용자는 1분 뒤 다시 하면 된다.

---

## 8. 이메일 인증 — `EmailVerificationService`와 `SupabaseAuthClient`

> 🔎 **선택 근거 — 상태를 Redis에 두는 이유, HTTP 클라이언트**
> - **쿨다운(60초)·대기(1시간) 상태: Redis(TTL 키).**
>   - **대안:** DB 컬럼(`last_sent_at`), 서버 메모리 Map.
>   - **안 쓴 이유:** DB는 만료 정리를 위해 스케줄러가 필요하다. 메모리는 서버 재시작·다중 서버에서 사라지거나 어긋난다. Redis는 TTL로 자동 삭제된다.
> - **Supabase 호출: 스프링 `RestClient`(동기 HTTP).**
>   - **대안:** `WebClient`(비동기), OpenFeign, Supabase 공식 Java SDK.
>   - **안 쓴 이유:** 서비스가 동기(Web MVC)라서 비동기 클라이언트의 이점이 없다. Feign·SDK는 호출이 몇 개 안 되어 의존성 추가 대비 이득이 작다.

### 8-1. `EmailVerificationService.java`

- **L10-13** 주석 — Supabase가 확인 메일을 보내고, 사용자가 링크를 누르면 Supabase 쪽 이메일이 확인된다. 우리는 로그인 때 그 결과를 조회해 `users`를 인증 완료로 바꾼다. 임시 계정 id는 Redis에 TTL로 둔다.
- **L14** `@Service`.
- **L17** `COOLDOWN_KEY` — "메일을 방금 보냈다" 표시 키. **L18** `PENDING_KEY` — 이메일별 Supabase 임시 계정 id 키. **L19** `COOLDOWN = 60초`.
- **L22** `PENDING_TTL = 1시간` — 마지막 발송 후 임시 계정 id를 기억하는 시간(`UnverifiedUserCleaner`가 참조하려고 package-private).
- **L24-26** Redis, 회원 저장소, Supabase 클라이언트.
- **L28-36** 생성자.
- **L38-58** `sendLink(normalizedEmail)` — 확인 메일 발송
  - **L40-43** `setIfAbsent(키, "1", 60초)` — 키가 **없을 때만** 만들고 성공(true). 즉 같은 이메일로 60초에 한 번만 통과한다. 이미 있으면 `TOO_MANY_REQUESTS`(429). ✅ 원자적이라 동시 요청에도 안전.
  - **L44** 아래 외부 호출이 실패할 수 있어 `try`.
  - **L45** 이미 만든 임시 계정 id가 Redis에 있는지 조회.
  - **L46-48** 없으면 `supabase.signUp`으로 임시 계정을 만들어(이때 Supabase가 메일 발송) id를 받고 Redis에 1시간 저장.
  - **L49-53** 있으면 가입을 다시 하지 않고 `supabase.resend`로 메일만 다시 보낸 뒤 유효 시간을 갱신한다(기존 id 유지, 주석 L50).
  - **L54-57** `CommonException`이면 쿨다운 키를 지워 바로 재시도할 수 있게 하고 예외를 다시 던진다. ✅ 메일이 안 갔는데 60초를 기다리게 하지 않는다. 💡 Redis 오류 같은 다른 예외는 쿨다운이 남는다.
  - 💡 Supabase `signUp`은 성공했는데 L48 Redis 저장이 실패하면 임시 계정 id를 잃어 Supabase에 고아 계정이 남는다. 드문 경우.
- **L60-67** `resend(rawEmail)` — 사용자가 "다시 받기"를 눌렀을 때
  - **L62** 이메일 정규화. **L63-66** 회원이 있고 **미인증일 때만** `sendLink`. 없거나 이미 인증됐으면 아무 일도 하지 않고 조용히 끝난다. ✅ 계정 존재 여부 비노출(컨트롤러가 항상 204).
  - 💡 단, 존재하는 미인증 계정은 쿨다운에 걸리면 429를 내므로 응답 차이(204 vs 429)로 존재 여부를 **추측**할 여지는 있다. 재전송 한도(IP당 10분 5회)가 이를 완화한다.
- **L69-72** `confirmIfVerified` — 확인 결과가 `CONFIRMED`면 true. **로그인에서 호출**한다.
- **L74-88** `checkVerification` — 상태 조회
  - **L79** Redis에서 임시 계정 id 조회.
  - **L80-82** 없으면 `EXPIRED`(대기 기록이 사라짐).
  - **L83** Supabase에 확인 여부 조회.
  - **L84-86** 확인됐으면 쓰임이 끝난 임시 계정과 대기 기록을 정리(`discard`).
  - **L87** 상태 반환.
- **L90-96** `linkSentWithin(email, window)` — 마지막 발송이 `window` 안인지. 대기 기록 TTL이 발송마다 1시간으로 리셋되는 성질을 이용한다(남은 TTL이 `1시간 - window`보다 크면 최근에 보낸 것). 💡 영리하지만 이해하기 어렵다.
- **L98-101** `discard` — 대기 기록을 가져오며 지우고(`getAndDelete`), 그 id의 Supabase 임시 계정을 삭제한다.

### 8-2. `VerificationStatus.java`
- **L3** 주석 — `UNKNOWN`은 조회에 실패했거나 근거가 사라져 판단할 수 없는 경우라 이 상태의 계정은 **지우지 않는다**.
- **L5-9** `CONFIRMED`(확인됨), `UNCONFIRMED`(아직), `UNKNOWN`(모름), `EXPIRED`(대기 기록 만료).

### 8-3. `SupabaseAuthClient.java`

- **L3-5** 난수, Base64, Map. **L6-9** 응답 DTO, 설정, 오류. **L10-11** 로거. **L12-16** HTTP(`MediaType`, `RestClient` 및 예외 2종).
- **L18-21** 주석 — Supabase Auth는 확인 메일만 빌려 쓴다. 진짜 계정은 `users` 테이블이고 Supabase 계정은 임시라 지운다. secret 키는 서버에서만 쓰고 로그에 남기지 않는다. ✅
- **L25** 로거. **L26** `PASSWORD_BYTES = 24` — 임시 계정 비밀번호용 난수 길이.
- **L28-30** HTTP 클라이언트, 설정 여부, 난수 생성기. **L33-36** 생성자 — 설정이 올바른지 미리 저장.
- **L39-68** `signUp(email)` — 임시 계정 생성 + 메일 발송
  - **L40-44** 설정이 없으면 로그를 남기고 `EMAIL_SEND_FAILED`(503). 로그에 키 값은 없다. ✅
  - **L45-47** 무작위 24바이트 비밀번호 생성. 아무도 모르는 값이라 이 계정으로는 로그인할 수 없다. ✅
  - **L49-56** `POST /auth/v1/signup`(기준 주소는 `SupabaseConfig`)에 `{email, password}`를 JSON으로 보내고 응답을 `SupabaseUserResponse`로 받는다.
  - **L57** 응답에서 사용자 id 추출.
  - **L58-63** id가 없거나 "메일 없이 바로 가입됨"이면 Confirm email이 꺼진 잘못된 설정 → 소유 확인이 안 되므로 임시 계정을 지우고 실패 처리. ✅ 보안 설정 오류를 서버가 방어.
  - **L64** id 반환. **L65-66** 네트워크·HTTP 오류는 `sendFailure`로 변환.
- **L74-90** `resend(email)` — 이미 만든 임시 계정으로 메일 재발송. 주석(L71-73): 가입 API를 다시 부르면 이메일 열거 방지 설정 때문에 다른 id가 돌아와 기존 임시 계정을 잃을 수 있어 쓰지 않는다.
  - **L75-78** 설정 없으면 실패. **L80-86** `POST /resend`, 본문 `{type: "signup", email}`, 응답 본문은 필요 없어 `toBodilessEntity()`.
- **L93-103** `sendFailure` — 오류 변환
  - **L94-100** HTTP 응답 오류면 **상태 코드만** 로그. 429는 `TOO_MANY_REQUESTS`, 나머지는 `EMAIL_SEND_FAILED`.
  - **L101-102** 연결 자체 실패는 예외 **종류 이름만** 로그. ✅ 메시지에 URL·키가 섞일 수 있어 일부러 뺀 것.
- **L106-127** `confirmationStatus(userId)` — 이메일 확인 여부 조회
  - **L107-109** 설정 없으면 `UNKNOWN`. **L111-116** `GET /admin/users/{id}`(관리자 키 필요). **L117-119** 응답이 비면 `UNKNOWN`. **L120-122** `email_confirmed_at`이 있으면 `CONFIRMED`, 없으면 `UNCONFIRMED`. **L123-126** 실패하면 `UNKNOWN`. ✅ 조회 실패를 "미인증"으로 오해해 계정을 지우는 일을 막는다.
- **L130-143** `deleteUser(userId)` — 임시 계정 삭제. 설정이 없거나 id가 없으면 아무것도 안 한다. 실패해도 경고 로그만 남기고 흐름을 막지 않는다. 💡 삭제 실패한 임시 계정이 Supabase에 쌓일 수 있다.

### 8-4. `SupabaseConfig.java`, `SupabaseProperties.java`
- **SupabaseProperties L6-8** `app.supabase.url/secret` 설정을 받는 record. **L11-15** `configured()` — 주소·키가 모두 있고 `http(s)://`로 시작해야 true. **L18-20** `baseUrl()` — 끝의 `/`와 `/rest/v1`을 떼어 프로젝트 주소만 남긴다.
- **SupabaseConfig L17-35** `RestClient` 빈
  - **L19-22** JDK HTTP 클라이언트, **연결 5초·읽기 10초 타임아웃**. ✅ 외부 호출이 무한정 매달리지 않는다.
  - **L25-27** 설정이 올바르면 기준 주소 `<프로젝트>/auth/v1`과 `apikey` 헤더(비밀 키)를 기본으로 단다.
  - **L29-32** 옛 형식 키(`eyJ...` JWT)일 때만 `Authorization: Bearer` 헤더도 단다. 새 형식 키(`sb_secret_...`)는 JWT가 아니라 보내지 않는다.

---

## 9. `AuthService.login` — 로그인의 핵심

> 🔎 **선택 근거 — 로그인 방어**
> - **더미 BCrypt 비교:** 없는 이메일이어도 BCrypt를 한 번 돌려 응답 시간 차이로 가입 여부를 알아내는 **타이밍 공격**을 막는다. 대안은 "없으면 즉시 실패"(빠르고 단순하지만 계정 존재가 노출).
> - **이메일 5회 / IP 20회 제한(15분):** 무차별 대입을 막는다.
>   - **대안:** CAPTCHA(봇 차단에 강하나 사용성 저하, 외부 서비스 필요), DB `failed_count`·`locked_until` 컬럼(영속적이지만 쓰기 부하), 2단계 인증(가장 강력하나 개발 범위가 큼).
>   - **안 쓴 이유:** 현 단계에서는 Redis 카운터가 가장 가볍다. 단 이메일 기준 잠금은 남이 일부러 틀려서 잠글 수 있는 문제가 있다(리뷰 요약 참고).
> - **오류 문구를 하나로 통일:** "이메일 없음"과 "비밀번호 틀림"을 구분해 알려 주면 계정 열거 공격이 쉬워진다.

- **L99** 주석 — 제한 중이면 거부하고, 실패는 기록하며, 미인증 계정은 인증 여부를 확인한 뒤에만 허용. 인증 여부 확인이 외부 호출이라 **트랜잭션으로 묶지 않는다**.
- **L100** `login(LoginRequest request, String clientIp)`.
- **L101** 이메일 정규화(가입 때와 같은 규칙이라 같은 키로 조회된다).
- **L102-103** 주석 + `loginRateLimiter.checkAllowed(email, clientIp)` — 이미 제한(이메일 5회·IP 20회 실패) 중이면 **비밀번호를 보기 전에** 429로 거부. ✅ 제한 중에는 비밀번호가 맞아도 로그인되지 않는다.
- **L104** `findByEmail(email).orElse(null)` — 회원 조회, 없으면 `null`.
- **L105** 회원이 없으면 `dummyHash`, 있으면 진짜 해시.
- **L106** `passwordEncoder.matches(입력, 해시)` — ✅ 회원이 없어도 비교를 해서 걸리는 시간이 비슷해진다.
- **L107-110** 회원이 없거나 비밀번호가 틀리면 실패를 기록하고(`recordFailure`) 401 `INVALID_CREDENTIALS`. 어느 쪽이 틀렸는지 알리지 않는다. ✅ 메시지 "이메일 또는 비밀번호가 올바르지 않습니다".
- **L111** `reset(email)` — 비밀번호가 맞았으니 이 이메일의 실패 횟수 삭제. (IP 횟수는 남긴다.)
- **L112** 주석 — 비밀번호가 맞은 뒤에만 인증 여부를 알려 미인증 계정의 존재를 함부로 드러내지 않는다. ✅
- **L113-119** 미인증 계정이면
  - **L114-116** `confirmIfVerified`가 false면(링크를 안 눌렀거나 조회 실패/기록 만료) 400 `EMAIL_NOT_VERIFIED`(`AUTH_008`) → 프런트가 "인증 메일 다시 받기" 버튼을 띄운다.
  - **L117-118** true면 `markEmailVerified()` + `save` — `users.email_verified = true`로 영구 반영. 이후 로그인은 Supabase를 부르지 않는다. 💡 `open-in-view: false`라 `user`는 이미 분리된 상태여서 `save`가 병합(merge)을 한다. 동작은 정상이다. 💡 `confirmIfVerified` 안에서 Supabase 임시 계정과 Redis 기록이 이미 지워졌으므로, 그 직후 `save`가 실패하면 다음 로그인은 `EXPIRED` → AUTH_008이 되어 메일 재전송이 필요하다.
- **L120** `issueTokens(user)` — 통과했으니 토큰 발급.
- **L172-179** `issueTokens`
  - **L175** `jwtTokenProvider.createAccessToken(user)` — Access Token.
  - **L176** 만료 초(15분 = 900).
  - **L177** `refreshTokenStore.issue(user.getId())` — Refresh Token 발급 + Redis 저장.
  - **L178** 회원 정보 DTO.

⚠️ 이 메서드에 대한 리뷰 의견
- **계정 잠금 악용(L103, `LoginRateLimiter`)**: 이메일 단위 5회 실패 잠금 때문에, **남이 내 이메일로 5번 틀리게 입력하면 나는 15분간 로그인할 수 없다.** 이메일+IP 조합으로 세거나 단계적 지연(점점 늘어나는 대기)을 검토. 현재 방식은 무차별 대입을 확실히 막는 대신 가용성을 희생한 것이라 정책 결정이 필요하다.
- **응답 시간**: 잠금 확인(L103)은 BCrypt 비교보다 먼저라 잠긴 계정은 빠르게 429가 나온다. 의도된 동작이다. 실패 기록은 존재하지 않는 이메일에도 쌓이므로 계정 존재 여부와는 무관하다.

### 9-1. `LoginRateLimiter.java`

- **L3-7** import(시간, 오류 코드, `RateLimitedException`, Redis, `@Component`).
- **L9-12** 주석 — 이메일별 5회, IP별 20회까지 실패 가능, 첫 실패 후 15분 뒤 해제. 제한 중엔 비밀번호가 맞아도 로그인 불가.
- **L16-18** `MAX_EMAIL_FAILURES = 5`, `MAX_IP_FAILURES = 20`, `WINDOW = 15분`.
- **L20-21** Redis 키 접두사. **L22** `NO_EXPIRY = -1` — Redis가 "만료 없음"을 알려 주는 값.
- **L24-29** 생성자.
- **L32-35** `checkAllowed` — 이메일 키와 IP 키 둘 다 검사.
- **L38-41** `recordFailure` — 이메일·IP 카운터를 각각 +1.
- **L44-46** `reset` — 성공 시 이메일 카운터만 삭제(IP 카운터는 다른 계정 시도와 합산되므로 남김).
- **L49-61** `checkKey(key, max)`
  - **L50** 현재 횟수 조회. **L51-53** 없거나 한도 미만이면 통과(`return`).
  - **L54** 남은 TTL 조회.
  - **L55-58** ✅ TTL이 `-1`(만료 없는 키)이면 윈도우를 **다시 걸어** 영구 잠금을 막는다. (이전 리뷰의 "꼭 확인" 항목이 이 코드로 완화됨.)
  - **L59-60** 한도에 이르렀으면 `RateLimitedException(LOGIN_RATE_LIMITED, 남은 시간)`. 남은 시간을 모르면(`null`/0 이하) 15분.
  - 💡 `Long.parseLong(count)`는 Redis 값이 숫자가 아니면 예외가 난다. 이 키는 이 클래스만 쓰므로 현재는 안전.
- **L64-69** `increment`
  - **L65** Redis `INCR` — 원자적 +1. **L66-68** 처음(`count == 1`)이면 15분 만료를 건다.
  - 💡 `INCR`과 `EXPIRE`는 여전히 두 명령이다. 사이에 실패하면 한도 **미만** 구간(1~4회)에서는 만료 없는 키가 남을 수 있다(한도에 닿을 때야 L55-58이 고친다). 영구 잠금은 아니지만 실패 횟수가 오래 누적된다. Lua 스크립트 등으로 원자화하면 깔끔하다.
  - 💡 Redis가 죽으면 이 클래스는 예외를 그대로 던져 **로그인이 500으로 실패**한다(`RequestRateLimiter`는 통과시킴). 보안상 의도라면 주석으로 남기자.

### 9-2. `GlobalExceptionHandler`, `CommonException`, `RateLimitedException`, `ErrorResponse`

- **CommonException L3-14** `RuntimeException`을 상속 — 그래서 서비스에서 던지면 호출 쪽에서 `catch (RuntimeException)`으로 잡을 수 있다(`signup` L92). `errorCode()`로 오류 종류를 꺼낸다.
- **RateLimitedException L6-18** `CommonException` + 다시 시도할 수 있기까지 남은 시간(`retryAfter`).
- **GlobalExceptionHandler**
  - **L35-41** `@Valid` 실패(`MethodArgumentNotValidException`) → 400 `COMMON_001` + 항목별 `fieldErrors`(프런트가 `fieldErrors[0].message`로 표시).
  - **L58-76** `RateLimitedException` → 429 + `Retry-After` 헤더. 로그인 제한이면 "N분 뒤에 다시 시도해 주세요"로 올림 계산(`(seconds + 59) / 60`).
  - **L79-83** `CommonException` → 오류 코드의 상태·코드·메시지.
  - **L86-103** 예상 못 한 예외는 내부 정보 없이 공통 오류로(로그에는 클래스 이름과 requestId만). ✅ 스택 트레이스·메시지가 응답으로 새지 않는다.
- **ErrorResponse L6-13** `timestamp, status, code, message, path, requestId, fieldErrors` — 프런트 `request()`가 `payload.message`, `payload.code`, `payload.fieldErrors`를 쓰는 근거.

### 9-3. 관련 오류 코드 (`ErrorCode`)

| 코드 | HTTP | 언제 |
|---|---|---|
| `AUTH_003` DUPLICATE_EMAIL | 409 | 가입 시 이메일 중복 |
| `AUTH_004` DUPLICATE_NICKNAME | 409 | 가입 시 닉네임 중복 |
| `AUTH_005` INVALID_CREDENTIALS | 401 | 로그인 실패(이메일·비밀번호 어느 쪽인지 안 알림) |
| `AUTH_006` INVALID_REFRESH_TOKEN | 401 | Refresh 토큰이 없거나 사용됨/만료 |
| `AUTH_008` EMAIL_NOT_VERIFIED | 400 | 비밀번호는 맞지만 이메일 미인증 |
| `AUTH_010` TOO_MANY_REQUESTS | 429 | 인증 메일 60초 쿨다운 / Supabase 429 |
| `AUTH_011` EMAIL_SEND_FAILED | 503 | 인증 메일 발송 실패(설정 오류 포함) |
| `AUTH_012` LOGIN_RATE_LIMITED | 429 | 로그인 실패 한도 초과 |
| `COMMON_003` REQUEST_RATE_LIMITED | 429 | 가입·재전송·갱신 요청 한도 초과 |
| `COMMON_001` INVALID_INPUT | 400 | 입력 검증 실패 |

---

## 10. 토큰 발급 — `JwtTokenProvider`, `RefreshTokenStore`

> 🔎 **선택 근거 — 토큰 종류와 저장**
> - **Access Token: JWT(HS256), 15분.**
>   - **대안:** RS256/ES256(비대칭 키: 공개키만 다른 서비스에 나눠 검증 가능), 서버 세션, Opaque 토큰.
>   - **안 쓴 이유:** 발급·검증 서버가 같은 애플리케이션 하나라 비대칭 키가 줄 이점이 없고, HS256이 더 빠르고 설정이 단순하다. 서비스가 여러 개로 쪼개지면 RS256으로 바꾼다.
> - **Refresh Token: 무작위 32바이트 + Redis, 해시로 저장, 1회용 회전(`getAndDelete`).**
>   - **대안:** DB 테이블, Refresh도 JWT로 발급(상태 저장 없음).
>   - **안 쓴 이유:** JWT형 Refresh는 탈취돼도 서버가 폐기할 수 없다. DB는 만료 정리가 필요하고 느리다. Redis는 TTL이 자동이고 AGENTS.md가 "Refresh Token은 Redis"로 정했다. 해시로 저장하면 Redis가 유출돼도 원본 토큰을 못 쓴다. 1회용으로 바꿔 쓰면 탈취된 토큰의 재사용을 감지한다.
> - **로그아웃 jti 블록리스트:** JWT는 만료 전 강제 무효화가 안 되므로 `jti`를 Redis에 남은 수명만큼 기록한다. 대안은 "그냥 15분 기다림"(단순하지만 로그아웃 후에도 토큰 유효)이나 사용자별 토큰 버전 컬럼(요청마다 DB 조회).

### 10-1. `JwtTokenProvider.java` (Access Token 만들기)

- **L3-5** `Clock`(현재 시각), `Instant`, `UUID`(무작위 id). **L6-7** `User`, `JwtProperties`. **L8-12** Spring Security JWT 클래스(알고리즘, 헤더, 클레임 묶음, 인코더, 인코더 파라미터).
- **L15** `@Component`. **L18-20** 인코더·설정·시계. `Clock`을 주입받아 테스트에서 시간을 고정할 수 있다. ✅
- **L22-27** 생성자.
- **L30-42** `createAccessToken(user)`
  - **L31** 현재 시각. **L33** 클레임 빌더 시작.
  - **L34** `.id(UUID)` — 토큰마다 고유한 `jti`. 로그아웃 차단 목록의 열쇠.
  - **L35** `.subject(userId)` — 토큰 주인.
  - **L36** `.issuedAt(now)`, **L37** `.expiresAt(now + 15분)`.
  - **L38** `.claim("scope", role)` — `"USER"`/`"ADMIN"`. 💡 Spring은 `scope` 클레임을 `SCOPE_USER` 권한으로 읽는다. 나중에 `hasRole("ADMIN")`을 쓰려면 권한 이름 매핑을 확인해야 한다. 지금은 역할 기반 접근 제어가 없어 문제 없음.
  - **L40** 헤더에 HS256(HMAC-SHA256) 지정. **L41** 서명해서 `xxxxx.yyyyy.zzzzz` 문자열로 반환.
- **L45-47** 유효 시간을 초로 반환(응답의 `expiresIn`).

### 10-2. `RefreshTokenStore.java` (Refresh Token을 Redis에 보관)

- **L3-9** 문자 인코딩, 해시, 난수, Base64, 16진수 변환, `Optional`. **L10-12** `JwtProperties`, Redis 도구, `@Component`.
- **L14** 주석 — 원문 토큰은 저장하지 않고 SHA-256 **해시를 키로** 쓴다. ✅ Redis가 유출돼도 토큰 원문을 알 수 없다.
- **L18** 키 접두사 `auth:refresh:`. **L19** `TOKEN_BYTES = 32`(256비트).
- **L21-23** Redis, 설정, `SecureRandom`(암호용 난수). ✅ 일반 `Random`이 아님.
- **L26-29** 생성자.
- **L32-40** `issue(userId)`
  - **L33-34** 32바이트 난수 생성.
  - **L35** URL에 안전한 Base64(패딩 없음) 문자열로 변환 — 이것이 쿠키에 담길 원문 토큰.
  - **L36-38** Redis에 `key(token)` → `userId`를 14일 TTL로 저장.
  - **L39** 원문 토큰 반환(저장은 해시, 쿠키는 원문).
- **L43-53** `consume(token)` — **한 번만 쓰게** 조회+삭제
  - **L44** `getAndDelete` — 가져오면서 동시에 삭제. ✅ 한 명령이라 같은 토큰으로 동시에 두 요청이 와도 한쪽만 성공한다(재사용 방지, rotation).
  - **L45-47** 없으면 빈 값(만료·이미 사용·위조). **L48-52** 숫자로 변환, 이상한 값이면 빈 값.
- **L56-58** `revoke` — 로그아웃 때 토큰 삭제.
- **L61-69** `key(token)` — 토큰을 SHA-256으로 해시해 16진수 문자열 앞에 접두사를 붙인다. `NoSuchAlgorithmException`은 사실상 일어나지 않아 `IllegalStateException`으로 바꿔 던진다.
- ✅ 갱신할 때마다 **새 토큰**이 나오고 이전 것은 폐기된다.

### 10-3. 갱신·로그아웃 (`AuthService.refresh/logout/getMe`)
- **L123-138** `refresh` — `@Transactional(readOnly = true)`(DB는 읽기만). L126-128 쿠키가 없거나 빈 문자열이면 `INVALID_REFRESH_TOKEN`. L129-132 `consume`으로 한 번 쓰고 폐기하며 userId 획득. L133-136 그 회원이 아직 있는지 확인(탈퇴·삭제됐으면 오류). L137 새 토큰 쌍 발급.
- **L140-152** `logout` — L142-144 Refresh 토큰이 있으면 Redis에서 삭제. L145-151 Access 토큰이 있으면 해석해서 차단 목록에 올린다(남은 유효 시간 동안). 이미 만료·위조된 토큰(`JwtException`)은 무시. ✅ 어떤 토큰이 와도 로그아웃이 실패하지 않는다.
- **L154-161** `getMe` — id로 조회해 DTO로 변환, 없으면 401.

### 10-4. `AccessTokenBlocklist.java`, `JwtProperties.java`, `ProductionSecurityGuard.java`
- **AccessTokenBlocklist L10** 토큰 원문 대신 `jti`만 저장한다. **L14** 접두사 `auth:access:blocked:`. **L26-36** `block` — 만료 시각·`jti`가 없으면 무시, 남은 시간이 0 이하면 무시, 아니면 남은 시간만큼만 Redis에 저장. ✅ 토큰이 어차피 만료되는 시점에 키도 사라져 Redis가 쌓이지 않는다. **L39-41** `isBlocked`.
- **JwtProperties L10-11** `@Validated`, `app.jwt` 접두사. **L13** `secret` 필수·**32자 이상**(HS256 키가 너무 짧은 사고를 서버 시작 때 막음). **L14** 기본 15분. **L15** 기본 14일. **L19** `refreshCookieSecure` 기본 `true`(HTTPS 전용). L16-18 주석: access 쿠키에도 적용되며 이름은 기존 설정 키와 맞추려고 그대로 둠. 로컬은 `application-local.yaml`에서 `false`.
- **ProductionSecurityGuard L6-21** `prod` 프로필에서 JWT 비밀 키가 `change-this`로 시작하거나 Secure 쿠키가 꺼져 있으면 **서버 시작을 막는다**. ✅ 로컬용 설정이 운영에 섞이는 사고 방지.

---

## 11. 저장소 — `User`, `UserRepository`, Flyway

> 🔎 **선택 근거 — 데이터 접근과 스키마 관리**
> - **쓴 것:** Spring Data JPA + PostgreSQL + Flyway.
> - **대안:** MyBatis·jOOQ(SQL을 직접 통제), Liquibase(스키마 변경 도구), `ddl-auto=update`.
> - **안 쓴 이유:** 팀의 기술 스택이 JPA로 정해져 있고, 단순 조회·저장이 대부분이다. `ddl-auto=update`는 운영 DB 변경 이력이 남지 않고 예기치 않은 변경이 일어나 AGENTS.md가 금지한다. Flyway는 변경이 파일로 남아 팀원 간 DB를 맞추기 쉽다. Liquibase도 가능하지만 이미 Flyway로 시작했다.

### 11-1. `User.java` (회원 Entity)
- **L3-10** JPA 어노테이션 import. **L11** `BaseTimeEntity` — `created_at`/`updated_at` 자동 관리 부모 클래스.
- **L13** `@Entity`. **L14** `@Table(name = "users")` — `user`는 PostgreSQL 예약어라 `users` 사용. ✅
- **L17-19** `id` — 기본키, `IDENTITY`(DB가 번호를 자동 증가).
- **L21-22** `email` — 필수, 254자, 유니크.
- **L24-25** `passwordHash` — 컬럼 `password_hash`, 필수, 100자(BCrypt 결과는 60자).
- **L27-28** `nickname` — 필수, 12자, 유니크.
- **L30-32** `role` — `EnumType.STRING`이라 `"USER"` 문자열로 저장. ✅ enum 순서가 바뀌어도 안전.
- **L34-35** `emailVerified` — 컬럼 `email_verified`, 필수.
- **L38** `protected User() {}` — JPA 전용 빈 생성자(외부에서 못 쓰게 `protected`).
- **L41-46** 생성자 — 이메일·해시·닉네임·역할로 인증 전 회원 생성. `emailVerified`는 기본 `false`.
- **L49-51** `isEmailVerified()`. **L54-56** `markEmailVerified()` — setter 대신 의도가 드러나는 메서드. ✅
- **L59-81** getter들(id, 이메일, 해시, 닉네임, 역할). 💡 `getPasswordHash()`가 `public`이지만 DTO로는 절대 나가지 않는다(`UserResponse`에 없음).

### 11-2. `UserRepository.java`
- **L9** `JpaRepository<User, Long>` — 기본 저장·조회 기능 자동 제공.
- **L11** `findByEmail` — 이메일로 한 명. **L13** `existsByEmail`. **L15** `existsByNicknameIgnoreCase` — 대소문자 무시 중복 확인. 💡 대소문자 변환 비교라 일반 유니크 인덱스를 타지 못할 수 있다. 회원이 많아지면 `lower(nickname)` 인덱스가 조회 성능에도 도움이 된다(7번의 유니크 인덱스와 같이 해결). **L17** `findByEmailVerifiedFalseAndCreatedAtBefore` — 미인증 + 특정 시각 이전 가입자(정리 작업용).
- 메서드 이름만으로 쿼리가 만들어진다. 모두 파라미터 바인딩이라 SQL 인젝션 위험이 없다. ✅

### 11-3. `UserRole.java`
- **L3-6** `USER`, `ADMIN`.

### 11-4. Flyway
**V5__create_users.sql**
- **L1** 주석 — `products.owner_id`가 참조할 사용자 식별자.
- **L2-13** `users` 테이블: `id`(자동 증가 PK), `email VARCHAR(254)`, `password_hash VARCHAR(100)`, `nickname VARCHAR(12)`, `role VARCHAR(20)`, `created_at`/`updated_at`(시간대 포함).
- **L10** `uk_users_email UNIQUE (email)` — 이메일 중복 금지. **L11** `uk_users_nickname UNIQUE (nickname)` — 닉네임 중복 금지(**대소문자 구분**). 💡 `AuthService` L85가 이 이름을 문자열로 비교하므로 이름을 바꾸면 중복 판별이 깨진다.
- **L12** `ck_users_role CHECK (role IN ('USER','ADMIN'))` — 역할 값을 DB에서도 제한. ✅

**V2026101901__add_users_email_verified.sql**
- **L2** `email_verified BOOLEAN NOT NULL DEFAULT TRUE` — 이미 가입한 회원은 인증한 것으로 본다. **L3** 그 뒤 기본값을 `FALSE`로 바꿔 **새 가입자는 미인증으로 시작**. ✅ 기존 사용자가 갑자기 로그인 못 하는 사고를 피한 순서.

---

## 12. `UnverifiedUserCleaner.java` — 인증 안 한 계정 자동 정리

> 🔎 **선택 근거 — 정리 작업 방식**
> - **쓴 것:** `@Scheduled` 주기 작업으로 기한 지난 미인증 계정 삭제.
> - **대안:** Quartz(작업 영속·클러스터 지원), ShedLock(여러 서버 중 한 곳만 실행), DB 이벤트/pg_cron, 삭제하지 않고 방치.
> - **안 쓴 이유:** 현재 서버 1대 기준이라 `@Scheduled`로 충분. 서버를 늘리면 같은 작업이 동시에 돌므로 ShedLock이 필요해진다(남은 과제). 방치하면 가입 이메일·닉네임이 영원히 묶인다.

- **L3-13** import(시계, 시간, `List`, 엔티티, 저장소, 로그, 설정값 주입, 스케줄, `@Transactional`).
- **L15** 주석 — 정해진 시간 안에 인증하지 않은 계정을 지워 이메일·닉네임을 다시 쓸 수 있게 한다.
- **L19** 로거. **L21-24** 저장소, 이메일 인증 서비스, 시계, 유예 시간(`ttl`).
- **L27-36** 생성자. **L31** `@Value("${app.auth.unverified-user-ttl:30m}")` — 설정값, 없으면 30분.
- **L39** `@Scheduled(fixedDelayString = "PT5M", initialDelayString = "PT1M")` — 시작 1분 뒤부터, 끝난 뒤 5분 간격.
- **L40** `@Transactional` — 삭제·수정을 한 묶음으로.
- **L42-43** "인증 안 됨 + 가입한 지 `ttl` 지남" 회원을 모두 조회. **L44** 삭제 개수 카운터. **L45** 한 명씩 처리.
- **L46** `checkVerification` — Redis + Supabase로 확인 상태 조회.
- **L47-51** `CONFIRMED` — 링크는 눌렀지만 아직 로그인 안 한 사용자는 **지우지 않고** 인증 완료로 바꾼다. ✅ 인증한 사용자를 지우는 사고 방지.
- **L52-56** `UNKNOWN` — 확인 여부를 모르면 경고 로그(userId만) 후 건너뜀. 다음 주기에 재확인.
- **L57-61** `UNCONFIRMED`인데 메일을 최근에 다시 보냈다면 그 시점부터 다시 기다림.
- **L62-67** `EXPIRED`여도 가입한 지 1시간(`PENDING_TTL`)이 안 됐다면 지우지 않음(확인할 기회가 남았을 수 있음).
- **L68** 임시 계정(Supabase) 정리. **L69** 우리 DB에서 회원 삭제. **L70** 개수 +1.
- **L72-74** 하나라도 지웠으면 개수만 로그. ✅ 이메일 같은 개인정보는 로그에 안 남긴다.
- 💡 서버가 여러 대면 스케줄이 중복 실행된다(분산 락 없음). 지금은 단일 서버라 문제 없다.
- 💡 반복문 안에서 `checkVerification`이 건마다 외부 HTTP를 부르는데 이 메서드가 `@Transactional`이다. 미인증 계정이 많이 쌓이면 DB 트랜잭션이 길어진다.
- 💡 가입 후 최대 30분(+5분 주기 지연) 동안 같은 이메일·닉네임을 다른 사람이 쓸 수 없다(선점 가능성). 정책상 감수하는 부분.

---

## 13. 설정 파일에서 이 흐름에 영향을 주는 값

- `application.yaml`
  - **`server.forward-headers-strategy: framework`** — 프록시가 보낸 `X-Forwarded-*` 헤더를 해석해 `getRemoteAddr()`에 반영. 로그인·가입 제한이 **IP 기준**이라 배포 환경에서 꼭 확인할 값.
  - **`spring.jpa.open-in-view: false`**, **`ddl-auto: validate`** — 화면 렌더 동안 DB 연결을 잡지 않고, DB 구조는 Flyway가 관리한다. ✅
  - **`app.supabase.url/secret`** — 환경 변수(`SUPABASE_URL`, `SUPABASE_SECRET`)로만 주입. 코드에 키 없음 ✅
  - **`app.jwt.secret: ${JWT_SECRET}`** — 환경 변수로만. 기본값 없음.
- `application-local.yaml` — `refresh-cookie-secure: false`(HTTP 개발용), JWT 기본값은 `change-this-...`(로컬 전용). `ProductionSecurityGuard`가 prod에서 이 값을 막는다.
- `application-prod.yaml` — `refresh-cookie-secure: true`.
- `application-supabase.yaml` — DB 연결 풀 최대 5개(기본). `signup`/`login`이 트랜잭션 안에서 외부 호출을 하지 않게 바뀌어 이전만큼 병목 위험이 크지 않지만, `UnverifiedUserCleaner`는 아직 긴 트랜잭션이다.

---

## 14. 보안 체크리스트 (현재 코드 기준)

| 항목 | 결과 |
|---|---|
| 비밀번호 평문 저장/로그/응답 | ✅ 없음(BCrypt 해시만 저장, 응답 DTO에 없음) |
| 토큰이 응답 본문·JS에 노출 | ✅ 없음(HttpOnly 쿠키만) |
| Refresh Token 재사용 | ✅ 1회용(rotation) + Redis 해시 키 |
| 로그아웃 후 Access Token | ✅ `jti` 차단 목록으로 즉시 무효화 |
| CSRF | ✅ 쿠키 요청도 검사(필터 순서 정확), 실패 시 403 |
| XSS로 DOM에 닉네임 삽입 | ✅ `textContent` 사용 |
| 계정 존재 여부 노출 | ⚠️ 가입 409로 노출(의도된 UX지만 재전송 정책과 다름), 로그인·재전송은 숨김 |
| 무차별 대입 | ✅ 이메일 5회/IP 20회 제한, 타이밍 공격 방어. ⚠️ 이메일 잠금 악용 가능성 |
| 외부 호출 타임아웃 | ✅ 연결 5초·읽기 10초 |
| 로그에 비밀 노출 | ✅ 상태 코드·예외 종류·userId만 기록 |
| 시크릿 환경 변수 | ✅ 코드에 하드코딩 없음, prod에서 로컬 키 차단 |
| 닉네임 동시성 | ⚠️ DB 유니크가 대소문자를 구분(7번) |

---

## 15. 일련번호 보호 — 지문 생성(HMAC)과 암호문(AES-GCM)을 따로 정리

> 위치: 서버 `product/service/SerialNumberProtector.java`(핵심), `product/service/ProductService.java`(호출), `product/repository/ProductRepository.java`(저장·조회), DB `V2026101303__create_product_tables.sql`·`V2026102001__align_product_p1_requirements.sql`, 테스트 `src/test/java/.../product/service/SerialNumberProtectorTest.java`.
> 회원가입·로그인에는 쓰이지 않는다. 사용자가 **제품을 등록할 때** 입력한 일련번호(시리얼)를 보호하는 코드이며, **같은 입력값을 두 가지 방식으로 따로 가공해 DB에 둘 다 저장**한다.
> 아래 15-A(지문)와 15-B(암호문)는 서로 독립적이라 따로 읽어도 된다.

### 15-0. 두 방식 한눈에 비교

| | 15-A 지문 (HMAC-SHA256) | 15-B 암호문 (AES-256-GCM) |
|---|---|---|
| 목적 | **같은 기기 중복 등록 방지** | 소유자 본인에게 **원래 번호를 다시 보여 주기** |
| 되돌릴 수 있나 | ❌ 불가능 | ✅ 키가 있으면 복호화 |
| 같은 입력 → 결과 | **항상 같다**(그래야 중복을 찾는다) | **매번 다르다**(무작위 IV) |
| 결과 길이 | 16진수 64글자 | Base64 약 124글자(최대 입력 64자 기준) |
| DB 컬럼 | `serial_number_fingerprint VARCHAR(64)` | `serial_number_ciphertext VARCHAR(512)` |
| 쓰는 키 | 마스터에서 파생한 **지문 전용 키** | 마스터에서 파생한 **암호화 전용 키** |
| 메서드 | `fingerprint()` | `encrypt()` / `decrypt()` |

두 키는 같은 마스터 시크릿(`app.product.serial-secret`)에서 **용도 이름을 다르게** 해서 따로 만든다(15-C). DB에는 **원문 일련번호가 어디에도 저장되지 않는다.**

---

### 15-A. 지문(Fingerprint) 생성 — HMAC-SHA256

> 🔎 **선택 근거 — 왜 HMAC-SHA256인가, 무엇을 안 썼나**
> - **쓴 것:** 비밀 키를 섞은 해시(HMAC-SHA256) → 항상 같은 입력이면 같은 64자리 값. 이것으로 DB에서 "같은 일련번호가 이미 등록됐나"를 **복호화 없이** 검사한다.
> - **대안 ① 일반 SHA-256(키 없음):** 구현은 가장 쉬움. → **안 쓴 이유:** 일련번호는 형식이 정해져 경우의 수가 작아서, DB가 유출되면 후보를 전부 해시해 비교하는 **사전 대입으로 역산**된다. 비밀 키가 있어야 계산되는 HMAC은 이를 막는다.
> - **대안 ② BCrypt/Argon2:** 매번 salt가 달라 결과가 달라진다. → **안 쓴 이유:** 같은 값끼리 비교·유니크 제약·인덱스 검색이 불가능하다. 지문은 "결정적(같은 입력→같은 출력)"이어야 한다.
> - **대안 ③ 암호문(AES) 자체로 중복 검사:** → **안 쓴 이유:** 매번 IV가 달라 암호문이 매번 다르다. 중복 비교용으로 못 쓴다(그래서 지문과 암호문을 따로 둠).
> - **대안 ④ 평문 저장 + unique:** 가장 단순하지만 DB 유출 시 일련번호가 그대로 노출 → 서비스 신뢰 문제.
> - **대안 ⑤ SHA-512 / HMAC-SHA512:** 가능하나 컬럼이 128자로 늘고, 보안 이득이 의미 있는 수준은 아니다(SHA-256으로 충분).

#### 15-A-1. 무엇이고 왜 필요한가
- **지문** = 일련번호를 비밀 키로 섞어 만든 고정 길이 문자열(64글자). 원문을 알아낼 수 없지만, **같은 번호는 항상 같은 지문**이 나온다.
- 그래서 DB에서 `지문이 같은 제품이 이미 있나?`만 확인하면 **원문을 몰라도 중복 등록을 막을 수 있다.**
- 단순 SHA-256이 아니라 **HMAC(키가 있는 해시)**를 쓰는 이유: 일련번호는 형식이 한정적이라 키 없는 해시는 가능한 값을 전부 계산해 두고 대조(사전 공격)할 수 있다. 비밀 키가 있어야 지문을 만들 수 있으면 DB만 유출돼도 대조할 수 없다.

#### 15-A-2. 데이터 흐름
```
브라우저 → POST /api/v1/products { manufacturer: "Hewlett-Packard", serialNumber: "ab12 cd34", ... }
  ▼
ProductService.create (@Transactional)
  ├─ normalizeManufacturer("Hewlett-Packard") → "HEWLETTPACKARD"
  ├─ fingerprint("ab12 cd34")
  │     ├─ normalize → "AB12CD34"   (공백 제거 + 대문자)
  │     └─ HMAC-SHA256(지문 전용 키, "AB12CD34") → 32바이트 → 16진수 64글자
  ├─ DB 조회: manufacturer_normalized = 'HEWLETTPACKARD' AND serial_number_fingerprint = '<64글자>'
  │     └ 있으면 409 PRODUCT_DUPLICATE_SERIAL
  └─ INSERT products (..., serial_number_fingerprint = '<64글자>')
        └ (제조사, 지문) 유니크 제약이 동시 등록도 막는다
```

#### 15-A-3. `SerialNumberProtector.java` 중 지문 관련 줄

- **L10** `import javax.crypto.Mac` — HMAC 계산기. **L12** `SecretKeySpec` — 바이트를 키 객체로. **L7** `HexFormat` — 바이트를 16진수 문자열로.
- **L32** `fingerprintKey` — 지문 전용 키를 담는 필드.
- **L36-39** 생성자에서 `app.product.serial-secret`이 없거나 **32자 미만이면 서버 시작 실패**. ✅
- **L40** 비밀 문자열을 UTF-8 바이트(`master`)로.
- **L41** `fingerprintKey = new SecretKeySpec(derive(master, "serial-fingerprint"), "HmacSHA256")` — 마스터에서 `"serial-fingerprint"`라는 이름으로 지문 전용 키를 만든다.
- **L45-48** `normalize(serialNumber)` — `replaceAll("\\s+", "")`로 공백(탭·줄바꿈 포함) 전부 제거, `toUpperCase(Locale.ROOT)`로 대문자화. `"ab12 cd34"`, `" AB12CD34 "`가 모두 `"AB12CD34"`가 된다. ✅ 입력 표기만 바꿔 중복 검사를 피하는 우회를 막는다.
- **L50-53** `normalizeManufacturer` — 공백·하이픈 제거 + 대문자. 중복 검사는 `(제조사, 지문)` 조합이므로 제조사 표기 차이로 우회되지 않게 같은 방식으로 정리한다.
- **L55-57** `fingerprint(serialNumber)` — `hmac(fingerprintKey, normalize(serialNumber))`의 결과 32바이트를 `HexFormat.of().formatHex(...)`로 **64글자 16진수**로 바꿔 반환. 컬럼 `VARCHAR(64)`와 길이가 정확히 맞는다.
- **L94-102** `hmac(key, message)` — 공통 HMAC 함수. `Mac.getInstance("HmacSHA256")` → `mac.init(key)` → `mac.doFinal(message의 UTF-8 바이트)`. 알고리즘이 없을 때만 `IllegalStateException`(사실상 발생 안 함).

#### 15-A-4. 지문이 쓰이는 곳

**ProductService.java** (`product/service/`)
- **L66** `String fingerprint = serialNumberProtector.fingerprint(request.serialNumber());`
- **L67-69** `existsByManufacturerAndSerial(제조사 정규화, 지문)`이 true면 `CommonException(PRODUCT_DUPLICATE_SERIAL)` → 409.
- **L73-78** `insertProduct(...)`에 지문을 넘겨 저장.
- **L79-81** 두 사람이 동시에 같은 번호를 등록해 L67 검사를 같이 통과해도 DB 유니크 제약 위반(`DataIntegrityViolationException`)을 같은 409로 바꾼다. ✅ 경쟁 상태 대비.

**ProductRepository.java** (`product/repository/`)
- **L37-42** `existsByManufacturerAndSerial` — `WHERE manufacturer_normalized = ? AND serial_number_fingerprint = ?`. 파라미터 바인딩이라 SQL 인젝션 없음.
- **L63-64** insert 컬럼에 `serial_number_fingerprint` 포함.

**DB**
- `V2026101303__create_product_tables.sql` L11 `serial_number_fingerprint VARCHAR(64) NOT NULL`, L21 지문 유니크 제약.
- `V2026102001__align_product_p1_requirements.sql` L13-16 — 유니크를 "지문만"에서 **(제조사 정규화, 지문)** 조합으로 변경. 서로 다른 제조사가 우연히 같은 시리얼이어도 막지 않는다.

#### 15-A-5. 지문 관련 테스트 (`SerialNumberProtectorTest`)
- `fingerprintIgnoresCaseAndWhitespace` — 대소문자·공백이 달라도 같은 지문.
- `manufacturerNormalizationIgnoresCaseWhitespaceAndHyphens` — `" Hewlett-Packard "` → `HEWLETTPACKARD`.
- `fingerprintDiffersForDifferentSerialNumbers` — 번호가 다르면 지문도 다름.
- `fingerprintFitsTheColumnAndHidesTheSerialNumber` — 길이 64, 원문이 지문 안에 보이지 않음.
- `fingerprintDependsOnTheSecret` — 비밀 값이 다르면 같은 번호도 다른 지문(키가 있어야 만들 수 있다는 증거).
- (이 문서를 쓰면서 테스트를 실행하지는 않았다.)

#### 15-A-6. 지문 리뷰 의견
✅ 키 있는 HMAC 사용(사전 공격 방어), 정규화로 우회 차단, 길이가 컬럼과 정확히 일치, 동시 등록까지 DB 제약으로 방어.
- ⚠️ **키가 바뀌면 지문이 전부 달라진다** — `PRODUCT_SERIAL_SECRET`을 교체하면 기존 제품의 지문과 새로 만드는 지문이 어긋나 **중복 검사가 무력화**된다. 지문은 키 버전 표시가 없어 재계산 마이그레이션이 필요하다.
- 💡 **지문은 결정적**이라 "이 시리얼이 이미 등록됨"(409)이 곧 존재 여부 힌트다. 같은 제조사·같은 번호일 때만 드러나는 UX 트레이드오프.
- 💡 정규화는 공백·대소문자만 처리한다. `0`/`O` 같은 혼동 문자는 다른 번호로 취급한다(의도된 범위로 보임).
- 💡 `fingerprint()`에 설명 주석(Javadoc)이 없다. 클래스 주석은 있다.

---

### 15-B. 암호문 생성·복호화 — AES-256-GCM

> 🔎 **선택 근거 — 왜 AES-256-GCM인가, 무엇을 안 썼나**
> - **쓴 것:** AES-256 + GCM 모드. 암호화와 동시에 위·변조 검출(인증 태그)까지 해 준다. IV는 매번 무작위 12바이트. JDK 표준(`javax.crypto`)이라 외부 라이브러리가 필요 없다.
> - **대안 ① AES-CBC(+ 별도 HMAC):** 널리 쓰이지만 무결성이 없어 직접 HMAC을 덧붙여야 하고, 패딩 오류로 **Padding Oracle 공격**에 노출될 수 있다. → GCM이 암호화+인증을 한 번에 해결.
> - **대안 ② AES-ECB:** 같은 평문이 같은 암호문이 되어 패턴이 보인다. 절대 사용 금지.
> - **대안 ③ RSA:** 비대칭이라 느리고 한 번에 암호화할 수 있는 길이가 짧다. 이 용도(서버만 암·복호화)는 대칭키가 적합하다.
> - **대안 ④ ChaCha20-Poly1305:** GCM과 같은 인증 암호화로 안전하고 하드웨어 가속이 없는 환경에서 빠르다. → 서버(AES-NI 지원)에서는 AES-GCM이 충분히 빠르고 더 보편적이라 선택.
> - **대안 ⑤ Jasypt·JPA `AttributeConverter` 자동 암호화:** 코드가 깔끔해진다. → 안 쓴 이유: 조회할 때마다 자동 복호화되어 "소유자만 복호화" 같은 통제가 흐려진다. 현재는 `decrypt()`를 부르는 곳이 명시적이다.
> - **대안 ⑥ DB 암호화(pgcrypto, 디스크 암호화)·KMS/Vault 봉투 암호화:** 더 안전하고 키 교체도 쉽다. → 안 쓴 이유: 운영 부담과 비용이 커서 현 단계에서는 앱 레벨 AES-GCM을 선택. **서비스가 커지면 KMS로 이전하는 것이 정석**이다.
> - **IV 재사용 금지:** 같은 키로 IV가 반복되면 GCM 보안이 무너진다. 그래서 `SecureRandom`으로 매번 새로 만든다.

#### 15-B-1. 무엇이고 왜 필요한가
- **암호문** = 일련번호를 비밀 키로 잠근 값. 키가 있어야만 원문으로 되돌릴 수 있다.
- 지문은 되돌릴 수 없으므로, **소유자에게 "내 제품의 일련번호는 이거예요"라고 다시 보여 주려면** 되돌릴 수 있는 보관본이 필요하다. 그 용도가 이 암호문이다.
- **GCM** 모드는 암호화와 함께 **인증 태그**를 만들어, 누군가 DB의 암호문을 조금이라도 바꾸면 복호화 단계에서 바로 실패한다(위변조 탐지).

#### 15-B-2. 저장 형식
```
DB에 저장되는 문자열 = Base64( IV 12바이트 | 암호문 | 인증 태그 16바이트 )
                                  ↑ 비밀 아님, 복호화에 꼭 필요해서 같이 저장
```
일련번호 최대 64자 → 12 + 64 + 16 = 92바이트 → Base64 약 124글자. 컬럼은 `VARCHAR(512)`라 충분하다(테스트 `longestSerialNumberStillFitsTheColumn`).

#### 15-B-3. 데이터 흐름

**암호화 (제품 등록 시)**
```
ProductService.create
  └─ SerialNumberProtector.encrypt("ab12 cd34")
        ├─ normalize → "AB12CD34"
        ├─ SecureRandom 으로 IV 12바이트 새로 생성  (매번 다름)
        ├─ AES/GCM/NoPadding, 암호화 전용 키, 태그 128비트로 암호화  → 암호문+태그
        ├─ IV + (암호문+태그) 이어 붙이기
        └─ Base64 문자열 → INSERT products.serial_number_ciphertext
```

**복호화 (소유자가 제품을 볼 때)**
```
ProductService.get(ownerId, productId)
  ├─ productRepository.findSerialCiphertextOwned(productId, ownerId)   ← 소유자 조건 포함
  │     └ 남의 제품이면 결과 없음 → 404 RESOURCE_NOT_FOUND
  └─ SerialNumberProtector.decrypt(ciphertext)
        ├─ Base64 해제 → 앞 12바이트 = IV, 나머지 = 암호문+태그
        ├─ 같은 키·같은 IV로 복호화 모드 초기화
        ├─ doFinal : 태그 검증 + 복호화    (틀리면 예외)
        └─ "AB12CD34" → 응답 ProductResponse.serialNumber 에만 포함
```
목록(`listMine`)과 판매글(`getForListing`)은 **복호화하지 않는다.** 일련번호가 판매글에 노출되지 않는다.

#### 15-B-4. `SerialNumberProtector.java` 중 암호문 관련 줄

- **L4** `GeneralSecurityException` — 암호 관련 오류의 부모. **L5** `SecureRandom`(암호용 난수). **L6** `Base64`.
- **L9** `Cipher` — 암호화 엔진. **L11** `GCMParameterSpec` — GCM의 IV·태그 길이 지정.
- **L29** `IV_BYTES = 12` — GCM 표준 IV 길이(96비트). **L30** `TAG_BITS = 128` — 인증 태그 길이(가장 강한 값). ✅
- **L33** `encryptionKey` — 암호화 전용 키 필드. **L34** `secureRandom` — IV 생성용.
- **L42** `encryptionKey = new SecretKeySpec(derive(master, "serial-encryption"), "AES")` — 마스터에서 `"serial-encryption"`이라는 이름으로 파생한 **32바이트(AES-256) 키**.
- **L59-73** `encrypt(serialNumber)`
  - **L61-62** 12바이트 배열을 만들고 `secureRandom.nextBytes(iv)`로 채운다. ✅ **암호화할 때마다 새 IV** → 같은 번호도 암호문이 매번 다르다.
  - **L63** `Cipher.getInstance("AES/GCM/NoPadding")` — GCM은 패딩이 필요 없다.
  - **L64** `cipher.init(ENCRYPT_MODE, encryptionKey, new GCMParameterSpec(128, iv))`.
  - **L65** `normalize(serialNumber)`를 UTF-8 바이트로 바꿔 `doFinal` — 결과는 **암호문 + 16바이트 태그**.
  - **L66-68** `iv.length + encrypted.length` 크기 배열에 IV, 암호문+태그를 차례로 복사.
  - **L69** `Base64.getEncoder().encodeToString(combined)` — DB에 넣을 문자열.
  - **L70-71** 암호 오류는 `IllegalStateException("Failed to encrypt serial number")`. 메시지에 원문이 들어 있지 않다. ✅
- **L75-88** `decrypt(ciphertext)`
  - **L77** Base64를 풀어 `combined` 배열 복원.
  - **L78** 같은 알고리즘 `"AES/GCM/NoPadding"`.
  - **L79-82** `cipher.init(DECRYPT_MODE, encryptionKey, new GCMParameterSpec(128, combined, 0, IV_BYTES))` — 배열 **0번째부터 12바이트를 IV로** 사용.
  - **L83-84** `cipher.doFinal(combined, IV_BYTES, combined.length - IV_BYTES)` — 12바이트 뒤부터 끝까지(암호문+태그)를 복호화하며 **태그를 검증**. 결과를 UTF-8 문자열로.
  - **L85-86** 키가 다르거나 데이터가 변조되면 `AEADBadTagException`(= `GeneralSecurityException`) → `IllegalStateException("Failed to decrypt serial number")`.
  - ✅ 복호화 결과는 **정규화된 값**(`"AB12CD34"`)이다. 사용자가 입력한 원래 표기(소문자·공백)는 복원되지 않는다.

#### 15-B-5. 암호문이 쓰이는 곳

**ProductService.java**
- **L78** `serialNumberProtector.encrypt(request.serialNumber())` — `insertProduct`의 마지막 인자로 넘겨 저장.
- **L93-99** `get(ownerId, productId)` — L95-98 `findSerialCiphertextOwned(productId, ownerId)`로 **소유자 조건**이 걸린 암호문을 가져오고(없으면 404), L99 `decrypt` 후 `product.withSerialNumber(...)`로 응답에 넣는다. ✅
- **L147-149** `update` 응답에서도 같은 방식으로 복호화.
- **L104-109** `getForListing` — 복호화 없음(판매 기능은 일련번호가 필요 없다).

**ProductRepository.java**
- **L63-64** insert 컬럼 `serial_number_ciphertext`. **L157-161** `findSerialCiphertextOwned` — `SELECT serial_number_ciphertext FROM products WHERE ... owner_id = ?` 형태로 소유자 확인.

**DB**
- `V2026101303__create_product_tables.sql` L17 `serial_number_ciphertext VARCHAR(512) NOT NULL`.

**로그·응답 보호**
- `CreateProductRequest.java` L63 `toString()`이 일련번호를 `<redacted>`로 출력 ✅ 요청 객체를 로그에 찍어도 원문이 남지 않는다.

#### 15-B-6. 암호문 관련 테스트 (`SerialNumberProtectorTest`)
- `encryptedSerialNumberRoundTripsToTheNormalizedValue` — `"ab12 cd34"` 암호화 → 복호화하면 `"AB12CD34"`.
- `ciphertextIsDifferentEveryTimeAndDoesNotContainThePlaintext` — 같은 번호를 두 번 암호화하면 결과가 다르고, 원문이 들어 있지 않음.
- `longestSerialNumberStillFitsTheColumn` — 64자 번호의 암호문이 512자 이하.
- `ciphertextCannotBeDecryptedWithAnotherSecret` — 다른 비밀 값으로는 복호화 실패(`IllegalStateException`).
- (이 문서를 쓰면서 테스트를 실행하지는 않았다.)

#### 15-B-7. 암호문 리뷰 의견
✅ 좋은 점
- **무작위 IV를 매번 새로 생성**(12바이트, `SecureRandom`), 태그 128비트 — 표준 권장 설정.
- GCM **인증 태그**로 변조 탐지, 다른 키로는 복호화 실패.
- 지문과 **키가 분리**되어 한쪽 키가 노출돼도 다른 쪽은 안전.
- 복호화는 **소유자 조건이 걸린 조회**로만 일어나며 판매·목록 응답에는 일련번호가 없다.
- 예외 메시지·`toString`에 원문이 없다.

⚠️/💡 확인·제안
1. ⚠️ **키 교체(rotation) 불가** — 암호문에 키 버전 표시가 없다. `PRODUCT_SERIAL_SECRET`을 바꾸면 **기존 모든 제품의 복호화가 실패**한다. 운영 값은 안전하게 백업하고, 바꿔야 한다면 `v1:` 같은 버전 접두사 + 재암호화 마이그레이션이 필요하다.
2. 💡 **AAD(추가 인증 데이터) 미사용** — 암호문이 어느 제품 행의 것인지 묶어 두지 않아, DB 쓰기 권한이 있는 사람이 두 제품의 암호문을 맞바꿔도 복호화가 성공한다. `productId`를 AAD로 넣으면 막을 수 있다(위협 모델상 낮음).
3. 💡 **복호화 입력 검증 없음** — Base64가 깨졌거나 12바이트보다 짧은 값이면 `GeneralSecurityException`이 아닌 `IllegalArgumentException`이 날 수 있어 `IllegalStateException` 변환 밖이다. 값은 DB에서만 오므로 현실적 위험은 낮다.
4. 💡 **IV가 랜덤 96비트** — 한 키로 아주 많은 수(약 40억 건 이상)를 암호화하면 IV 충돌 가능성이 생기지만, 이 서비스 규모에서는 문제없다.
5. 💡 `encrypt()`/`decrypt()`에 설명 주석(Javadoc)이 없다.

---

### 15-C. 공통 — 마스터 시크릿과 키 파생

> 🔎 **선택 근거 — 키 하나에서 둘을 만드는 이유**
> - **쓴 것:** 마스터 비밀 하나(`PRODUCT_SERIAL_SECRET`)에서 용도 이름("serial-fingerprint", "serial-encryption")으로 HMAC 키와 AES 키를 각각 파생.
> - **대안:** 같은 키를 HMAC과 AES에 그대로 사용 / 환경 변수를 용도별로 두 개 관리 / 정식 HKDF.
> - **안 쓴 이유:** 한 키를 두 알고리즘에 겹쳐 쓰면 한쪽 취약점이 다른 쪽으로 번질 수 있다(키 분리 원칙). 환경 변수 2개는 운영 실수 가능성이 늘어난다. HKDF는 표준 방식이라 더 엄밀하다 → 개선 후보로 기록.
> - **환경 변수로 주입:** 키를 코드·저장소에 두지 않는다(AGENTS.md 보안 규칙). 대안은 하드코딩이나 설정 파일 커밋인데 둘 다 유출 위험이 크다.

- **L90-92** `derive(master, purpose)` = `HMAC-SHA256(키=마스터, 메시지=용도 이름)`. `"serial-fingerprint"`와 `"serial-encryption"`이라는 서로 다른 이름 덕에 **두 키가 완전히 다른 값**이 된다(키 분리). 결과 32바이트는 HMAC 키로도, AES-256 키로도 길이가 맞다.
- 💡 표준 HKDF(RFC 5869)가 더 정석이지만, 마스터가 충분히 무작위라면 현재 방식도 키 분리 목적을 달성한다.
- **설정 `app.product.serial-secret`**
  - 로컬 `application-local.yaml` L27: `${PRODUCT_SERIAL_SECRET:change-this-local-product-serial-secret-32-bytes}` (로컬 전용 기본값)
  - 운영 `application-prod.yaml` L35: `${PRODUCT_SERIAL_SECRET}` — **기본값 없음**, 환경 변수 필수.
  - 테스트 `application-test.yaml` L22: 테스트용 값.
- 💡 마스터 검사는 "32자 이상"뿐이라 `aaaa…`도 통과한다. 운영에는 `openssl rand -base64 48` 같은 무작위 값을 쓴다.
- ⚠️ 마스터 시크릿이 바뀌면 **지문과 암호문이 동시에** 못 쓰게 된다(15-A-6, 15-B-7의 1번).

### 15-D. 같은 HMAC-SHA256을 쓰는 다른 곳 (참고)

| 위치 | 무엇을 | 키 | 용도 |
|---|---|---|---|
| `SerialNumberProtector` L55, L94 | 일련번호 → 64글자 지문 | `serial-secret`에서 파생한 지문 키 | 중복 검사 |
| `SerialNumberProtector` L90 | 용도 이름 → 키 | `serial-secret` 마스터 | 키 파생 |
| `SecurityConfig` L149, `JwtTokenProvider` L40 | 로그인 Access Token **서명(HS256 = HMAC-SHA256)** | `app.jwt.secret` | 토큰 위조 방지 |
| `LocalFileStorage` L90-97 `sign` | 파일 임시 URL `key\nexpires` 서명 | `signing-secret` | 임시 URL 위조 방지 |

- `LocalFileStorage.isValidTemporaryUrl`은 서명 비교에 `MessageDigest.isEqual`(시간이 일정한 비교)을 쓴다. ✅
- ⚠️ `STORAGE_SIGNING_SECRET` 기본값이 `${app.jwt.secret}`(JWT 비밀 키와 **같은 값**, `application.yaml`)이다. 용도별로 키를 나누는 `SerialNumberProtector`의 원칙과 다르므로 운영에서는 별도 값을 주입하는 것이 좋다.

---

## 16. 이 문서에서 하지 않은 것

- 테스트 실행(`./gradlew test`, `spotlessCheck`, `build`)은 하지 않았다. 코드 읽기만으로 작성했다.
- 브라우저에서 실제 가입·로그인을 눌러 보지 않았다. Supabase 설정(Confirm email 켜짐 여부, 메일 템플릿, 리다이렉트 주소)은 이 저장소 밖이라 확인하지 못했다.
- 가입·로그인 이후 마이페이지, 판매글 등 다른 기능은 범위에서 제외했다. (제품 일련번호 보호는 15번에서만 다뤘다.)
