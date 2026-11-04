# 인증(AUTH) 코드 리뷰: 로그인 요청은 어떻게 처리되는가

> `TROUBLESHOOTING.md`에 있던 코드 리뷰 내용을 옮겼습니다. 트러블이 아니라 코드 설명입니다.

로그인 폼에서 엔터를 누른 순간부터 응답이 화면에 반영될 때까지 코드를 따라가며 정리했습니다. 파일과 메서드 이름을 그대로 적었으니 코드를 열어 놓고 순서대로 대조할 수 있습니다. 이 절은 코드를 읽어서 작성했고, 로컬에서 브라우저로 전체 흐름을 직접 실행해 확인한 것은 아닙니다. 동작은 아래 [확인 방법](#확인-방법과-범위)의 테스트로 검증했습니다.

### 파일 지도

| 계층 | 파일 | 역할 |
| --- | --- | --- |
| 화면 | `static/pages/auth.html`, `signup.html` | 로그인·가입 폼(`data-auth-form`) |
| 화면 | `static/js/auth.js` | 폼 이벤트, API 호출, 세션 상태, 공통 요청 `PastportApi.fetch` |
| 보안 필터 | `common/security/SecurityConfig` | CSRF 검증, 접근 규칙, JWT 디코더(차단 목록 확인) |
| 보안 필터 | `CsrfCookieFilter`, `CookieBearerTokenFilter` | CSRF 쿠키 발급, 쿠키 토큰을 인증 헤더로 변환 |
| 컨트롤러 | `auth/controller/AuthController` | 요청 검증, 서비스 호출, 응답과 쿠키 생성 |
| 서비스 | `auth/service/AuthService` | 가입·로그인·갱신·로그아웃 업무 흐름 |
| 서비스 | `EmailVerificationService`, `SupabaseAuthClient` | 이메일 인증 메일 발송과 확인 조회 |
| 서비스 | `JwtTokenProvider`, `RefreshTokenStore`, `AccessTokenBlocklist` | 토큰 발급, Refresh Token 저장, Access Token 차단 |
| 서비스 | `LoginRateLimiter`, `UnverifiedUserCleaner` | 로그인 시도 제한, 미인증 계정 정리 |
| 저장소 | `UserRepository`(JPA/PostgreSQL), Redis, Supabase Auth | 회원 / 토큰·제한·대기 기록 / 이메일 확인 |

Controller → Service → Repository → Entity 구조를 따르며, 컨트롤러는 Repository를 직접 호출하지 않습니다.

### 서비스 메서드 한눈에 보기

| 메서드 | 하는 일 | 사용하는 저장소 |
| --- | --- | --- |
| `AuthService.signup` | 이메일·닉네임 중복 검사, 비밀번호 해시 저장, 인증 메일 발송 | DB, Redis, Supabase |
| `AuthService.login` | 시도 제한 확인, 비밀번호 비교, 인증 확인, 토큰 발급 | DB, Redis, Supabase |
| `AuthService.refresh` | Refresh Token을 한 번 쓰고 새 토큰 쌍 발급 | Redis, DB |
| `AuthService.logout` | Refresh Token 폐기, Access Token 차단 | Redis |
| `AuthService.getMe` | 회원 id로 내 정보 조회 | DB |
| `LoginRateLimiter.checkAllowed / recordFailure / reset` | 이메일 5회, IP 20회 실패 시 15분 제한 | Redis |
| `EmailVerificationService.sendLink / resend` | 인증 메일 발송(60초 쿨다운), 기존 임시 계정이면 메일만 재발송 | Redis, Supabase |
| `EmailVerificationService.checkVerification` | 대기 기록과 Supabase로 인증 여부 조회(`CONFIRMED / UNCONFIRMED / UNKNOWN / EXPIRED`) | Redis, Supabase |
| `JwtTokenProvider.createAccessToken` | HS256 토큰 생성(`sub`=회원 id, `jti`, `scope`=역할, 15분) | 없음 |
| `RefreshTokenStore.issue / consume / revoke` | 임의 토큰 발급, 조회와 동시에 삭제(재사용 방지), 폐기 | Redis |
| `AccessTokenBlocklist.block / isBlocked` | 로그아웃한 토큰의 `jti`를 남은 유효 기간 동안 차단 | Redis |
| `UnverifiedUserCleaner.deleteExpired` | 5분마다 인증 유예(30분)가 지난 미인증 계정 정리 | DB, Redis, Supabase |

### 로그인 흐름: 엔터를 누른 순간부터

**0. 화면을 열 때 (엔터 이전에 이미 일어나는 일)**

- `GET /` 응답에 `XSRF-TOKEN` 쿠키가 실립니다. `CsrfCookieFilter`가 매 요청마다 토큰을 읽어 쿠키를 내려주기 때문입니다.
- `pages.js`가 `/pages/auth.html` 같은 화면 조각을 가져와 채웁니다.
- `auth.js`가 로드되면 `refresh()`를 한 번 호출해 `POST /api/auth/refresh`로 세션 복구를 시도합니다. 로그인 전이면 Refresh Token이 없어 401을 받고 조용히 무시합니다.

**1. 엔터 입력 → 폼 제출 이벤트**

- 로그인 버튼이 `type="submit"`이라 입력칸에서 엔터를 눌러도 `submit` 이벤트가 발생합니다.
- `auth.js`는 `document`에 `submit` 리스너 하나를 두고(화면 HTML이 나중에 채워지기 때문), `data-auth-form="login"` 폼이면 `preventDefault()`로 기본 제출을 막은 뒤 `handleLogin(form)`을 호출합니다.

**2. `handleLogin`: 입력 확인과 첫 요청**

- 이메일과 비밀번호가 비어 있으면 요청 없이 안내 문구를 보여주고 끝냅니다.
- 중복 제출을 막으려고 제출 버튼을 잠근 뒤(`setBusy`) `request('/api/auth/login', ...)`를 호출합니다.
- **엔터 이후 첫 요청은 `POST /api/auth/login`입니다.**

| 항목 | 값 |
| --- | --- |
| 메서드·경로 | `POST /api/auth/login` |
| 헤더 | `Content-Type: application/json`, `X-XSRF-TOKEN: (XSRF-TOKEN 쿠키 값)` |
| 쿠키 | 브라우저가 `XSRF-TOKEN`을 자동으로 함께 보냄 |
| 본문(JSON) | `{ "email": "...", "password": "..." }` |

**3. 서버 필터 체인 (컨트롤러 도착 전)**

1. `RequestIdFilter`: Spring Security보다 먼저 실행되며, 요청 추적 id(`X-Request-Id`)를 정해 로그와 응답 헤더에 연결합니다. 클라이언트가 보낸 값이 있으면 그 값을, 없으면 새로 만듭니다.
2. CORS 검사: 허용된 출처인지 확인합니다.
3. `CsrfFilter`: `XSRF-TOKEN` 쿠키 값과 `X-XSRF-TOKEN` 헤더 값이 같은지 비교합니다. 다르거나 없으면 403으로 끝납니다.
4. `CsrfCookieFilter`: 응답에 CSRF 쿠키가 실리게 합니다.
5. `CookieBearerTokenFilter`: `POST /api/auth/`로 시작하는 요청은 쿠키 토큰을 무시합니다. 만료되거나 잘못된 쿠키가 남아 있어도 로그인이 막히지 않게 하려는 것입니다.
6. `BearerTokenAuthenticationFilter`: 토큰이 없으므로 익명으로 통과합니다.
7. 접근 규칙: `POST /api/auth/login`은 `permitAll`이라 인증 없이 허용됩니다.
8. `DispatcherServlet`이 `AuthController.login`을 찾아 호출합니다.

**4. `AuthController.login`: 요청을 받아 어디로 넘기는가**

- 본문이 `LoginRequest`로 변환되고 `@Valid`로 검증됩니다(`email`, `password` 모두 `@NotBlank`, 각각 최대 254, 64자). 실패하면 `GlobalExceptionHandler`가 400 `COMMON_001`로 응답합니다.
- 통과하면 컨트롤러는 업무 로직 없이 `authService.login(request, httpRequest.getRemoteAddr())`만 호출합니다. 두 번째 인자는 로그인 시도 제한에 쓸 접속 IP입니다.

**5. `AuthService.login`: 업무 흐름 (`@Transactional`)**

| 순서 | 처리 | 실패하면 |
| --- | --- | --- |
| 1 | `normalizeEmail`: 공백 제거, 소문자 변환 | - |
| 2 | `LoginRateLimiter.checkAllowed`: Redis `auth:login:fail:email:{이메일}`, `...:ip:{IP}` 확인 | 한도 도달 시 **429** `AUTH_012` + `Retry-After`. 비밀번호는 비교하지 않음 |
| 3 | `userRepository.findByEmail`: 회원 조회 | 없으면 가짜 해시(`dummyHash`)로 비교를 계속함 |
| 4 | `passwordEncoder.matches`: BCrypt 비교 | 결과는 5번에서 판정 |
| 5 | 회원이 없거나 비밀번호가 틀리면 `recordFailure`로 실패 기록 | **401** `AUTH_005` (이메일과 비밀번호 중 무엇이 틀렸는지 알리지 않음) |
| 6 | 성공하면 `reset`으로 이메일 실패 횟수 삭제 | - |
| 7 | 미인증 회원이면 `EmailVerificationService.confirmIfVerified` → `checkVerification`: Redis `auth:email:pending:{이메일}`로 Supabase 임시 계정 id를 찾아 `confirmationStatus` 조회 | 확인 안 됨: **400** `AUTH_008` |
| 8 | 확인됐으면 임시 계정 정리(`discard`) 후 `user.markEmailVerified()` | - |
| 9 | `issueTokens`: `JwtTokenProvider.createAccessToken` + `RefreshTokenStore.issue` | - |

- Access Token은 서명된 JWT(15분)이고, Refresh Token은 임의의 32바이트 값입니다. Redis에는 원문이 아니라 SHA-256 해시를 키(`auth:refresh:{해시}`)로, 값은 회원 id로 저장하며 유효 기간은 14일입니다.
- 계정 존재 여부를 숨기려고 (a) 없는 이메일에도 해시 비교 비용을 치르게 하고, (b) 실패 응답을 하나로 통일하고, (c) 비밀번호가 맞은 뒤에만 인증 여부를 알립니다.

**6. 응답 생성: `AuthController.withTokenCookies`**

- 응답 헤더에 `Set-Cookie`가 두 개 붙습니다.

| 쿠키 | 경로 | 만료 | 속성 |
| --- | --- | --- | --- |
| `access_token` | `/` | 15분 | `HttpOnly`, `SameSite=Strict`, `Secure`(설정) |
| `refresh_token` | `/api/auth` | 14일 | 위와 같음 |

- 본문은 `ApiResponse`로 감싼 `{ "success": true, "data": { "expiresIn": 900, "user": { "id", "email", "nickname" } } }`입니다. **토큰은 본문에 넣지 않습니다.**

**7. 브라우저: 응답 처리**

- 쿠키는 `HttpOnly`라 JS가 읽을 수 없고, 브라우저가 알아서 저장합니다.
- `handleLogin`이 "이메일 저장" 체크에 따라 이메일을 `localStorage`에 저장하거나 지우고, 비밀번호 칸을 비웁니다.
- `setSession(data)`가 회원 정보를 메모리에 두고 헤더(`renderHeader`)와 마이페이지(`renderMyPage`)를 다시 그린 뒤 `goto('landing')`으로 홈으로 이동합니다.

### 실패했을 때 화면에 어떻게 나타나는가

| 상황 | 상태 | 코드 | 화면 |
| --- | --- | --- | --- |
| 입력 형식 오류 | 400 | `COMMON_001` | 오류 메시지 표시 |
| 이메일 또는 비밀번호 불일치 | 401 | `AUTH_005` | 오류 메시지, 비밀번호 칸 비움 |
| 이메일 인증 전 | 400 | `AUTH_008` | 오류 메시지와 "인증 메일 다시 받기" 버튼 표시 |
| 실패 횟수 초과 | 429 | `AUTH_012` | "N분 뒤에 다시 시도해 주세요" |
| CSRF 토큰 불일치 | 403 | - | 오류 메시지 |
| 서버·Redis 오류 | 500 | `COMMON_999` | 내부 정보 없이 공통 오류 메시지 |

### 로그인 이후 인증이 필요한 요청은 어떻게 처리되는가

판매글 저장 같은 요청은 `window.PastportApi.fetch`(`auth.js`의 `authFetch`)를 거칩니다.

1. 브라우저가 `access_token` 쿠키를 자동으로 보내고, `authFetch`가 쓰기 요청에 `X-XSRF-TOKEN` 헤더를 붙입니다.
2. `CsrfFilter`가 쓰기 요청의 CSRF를 검사합니다.
3. `CookieBearerTokenFilter`가 `/api/`로 시작하는 요청에 한해 쿠키 값을 `Authorization: Bearer ...` 헤더가 있는 요청으로 바꿉니다. 화면과 정적 파일에는 적용하지 않아, 잘못된 쿠키가 화면 로딩을 막지 않습니다.
4. `BearerTokenAuthenticationFilter`가 `JwtDecoder`로 서명과 만료를 검사하고, `AccessTokenBlocklist.isBlocked(jti)`로 로그아웃된 토큰인지 확인합니다.
5. 통과하면 컨트롤러에서 `@AuthenticationPrincipal Jwt`로 회원 id(`sub`)를 얻습니다. 회원 id는 클라이언트가 보낸 값이 아니라 토큰에서 꺼내므로, 다른 회원의 id를 넣어 요청할 수 없습니다.
6. 401이 오면 `authFetch`가 `refresh()`를 한 번 호출해 세션을 갱신하고 같은 요청을 다시 보냅니다. 갱신도 실패하면 원래 401을 그대로 돌려줍니다.

### 갱신·로그아웃·가입

- **갱신 `POST /api/auth/refresh`:** `refresh_token` 쿠키를 `RefreshTokenStore.consume`이 조회와 동시에 삭제하므로 같은 토큰을 두 번 쓸 수 없습니다. 이후 회원을 조회해 새 토큰 쌍을 발급합니다. 동시에 두 번 갱신하면 한쪽이 실패하기 때문에 `auth.js`가 진행 중인 갱신을 공유합니다.
- **로그아웃 `POST /api/auth/logout`:** Refresh Token을 Redis에서 지우고, Access Token은 `jti`를 남은 유효 시간만큼 차단 목록에 올립니다. 응답으로 두 쿠키를 즉시 만료시킵니다. 이미 만료되거나 위조된 토큰은 차단하지 않고 넘어갑니다.
- **가입 `POST /api/auth/signup`:** `AuthService.signup`(`@Transactional`)이 중복을 검사하고 비밀번호를 BCrypt로 저장한 뒤 `EmailVerificationService.sendLink`로 인증 메일을 보냅니다. 메일을 보내지 못하면 예외로 가입 전체가 되돌아가 인증할 수 없는 계정이 남지 않습니다. 인증 메일은 같은 이메일에 60초에 한 번만 보낼 수 있습니다.
- **인증 유예와 정리:** `UnverifiedUserCleaner`가 5분마다 가입 후 30분이 지난 미인증 계정을 점검합니다. 인증됐으면 계정을 살리고, 조회에 실패해 알 수 없으면 건드리지 않으며, 최근에 메일을 다시 보냈다면 그 시점부터 다시 기다립니다.

### 코드 리뷰 관점

**잘 된 점**

- 컨트롤러가 얇고 업무 로직은 서비스에 있으며, 응답에 엔티티를 직접 쓰지 않고 DTO(`UserResponse`, `LoginResponse`)만 씁니다.
- 토큰이 응답 본문과 JS에 노출되지 않고 `HttpOnly` 쿠키로만 오갑니다.
- Refresh Token은 1회용이고 Redis에는 해시로만 저장합니다. Access Token은 로그아웃 후에도 차단됩니다.
- 로그인 실패 응답이 계정 존재 여부를 알려 주지 않고, 시도 제한 중에는 비밀번호 비교 비용도 쓰지 않습니다.

**알고 있어야 할 한계와 후속 작업**

- Redis가 멈추면 인증이 필요한 모든 요청이 500으로 실패합니다. Access Token 검증에 차단 목록 조회가 들어갔기 때문입니다. 503과 안내 메시지로 바꾸거나 장애 시 동작(차단 목록 확인을 건너뛸지)을 정해야 합니다.
- 인증 메일 재발송 응답(204 또는 429)으로 미인증 계정의 존재 여부를 추측할 수 있습니다. 모든 이메일에 같은 응답을 주도록 바꾸는 것이 좋습니다.
- 이메일 기준 시도 제한은 다른 사람이 일부러 틀린 비밀번호를 입력해 계정을 15분간 잠그는 데 쓰일 수 있습니다. 횟수 증가와 만료 설정도 한 번에 처리되지 않아, 만료 설정이 실패하면 기록이 남을 수 있습니다.
- 로그인 중 인증이 확인되면 Supabase 임시 계정을 즉시 지우는데, 같은 트랜잭션이 이후 롤백되면 인증 근거를 잃을 수 있습니다.
- `getRemoteAddr()`를 접속 IP로 쓰므로 프록시나 로드밸런서 뒤에서는 모든 요청이 같은 IP로 보일 수 있습니다. 배포 시 `server.forward-headers-strategy` 설정이 필요합니다.
- 채팅 실시간(`ChatStompAuthorizationInterceptor`)은 `Authorization: Bearer` 헤더만 읽습니다. Access Token이 `HttpOnly` 쿠키가 됐으므로 채팅 화면을 붙일 때 쿠키를 읽도록 맞춰야 합니다.
- 내 정보의 보유 제품 수(AUTH-05)와 관리자 권한 검사(ADM-01)는 아직 구현되지 않았습니다.
- 3절에 적힌 "인증 헤더를 붙이는 공통 fetch"는 당시 구현입니다. 지금은 쿠키 인증으로 바뀌어 `authFetch`가 헤더 대신 쿠키와 CSRF 헤더를 사용합니다.

### 확인 방법과 범위

- 자동 테스트: `AuthServiceTest`(가입·로그인·갱신·로그아웃·제한), `LoginRateLimiterTest`, `AccessTokenBlocklistTest`, `SecurityConfigTest`(차단된 토큰 거부), `EmailVerificationServiceTest`, `SupabaseAuthClientTest`, `UnverifiedUserCleanerTest`, `JwtTokenProviderTest`, `ListingDraftHttpTest`(쿠키 인증과 CSRF 403, 잘못된 쿠키가 화면을 막지 않는지).
- 직접 확인하려면 브라우저 개발자 도구에서 다음을 봅니다. Application → Cookies에서 `XSRF-TOKEN`이 페이지 로드 직후 생기는지, 로그인 후 `access_token`과 `refresh_token`이 `HttpOnly`로 생기는지, 로그아웃 후 둘 다 사라지는지 확인합니다. Network 탭에서 `POST /api/auth/login`의 요청 헤더에 `X-XSRF-TOKEN`이 있고 응답에 `Set-Cookie`가 두 개 있는지, 응답 본문에 토큰이 없는지 확인합니다.
- Redis, Supabase와 실제로 연결한 통합 실행은 이 문서를 쓰는 시점에 하지 않았습니다. 해당 저장소는 테스트에서 가짜 객체로 대체했습니다.

**근거:** `auth/controller/AuthController`, `auth/service/*`, `common/security/*`, `common/error/GlobalExceptionHandler`, `static/js/auth.js`, `static/pages/auth.html` 및 관련 테스트.

