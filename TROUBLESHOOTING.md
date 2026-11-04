# 트러블슈팅 정리

`pjmoo` 계정의 수정 커밋과 `docs/`의 Flyway 문서를 근거로, 2026-11-03까지 작업하면서 실제로 생긴 문제만 정리했습니다. 커밋에 기록되지 않은 재현 결과는 확인하지 않았습니다. 설계 설명과 코드 리뷰는 [시리얼 번호 보호](docs/SERIAL_NUMBER_PROTECTION.md), [인증 코드 리뷰](docs/AUTH_CODE_REVIEW.md)로 옮겼습니다.

## 1. Flyway 마이그레이션 버전 충돌

### 상황과 증상

상품 기능과 채팅 기능에서 각각 DB 테이블 생성 파일을 만들었는데 두 작업 모두 마이그레이션 버전 `V2`를 사용했습니다. 공유 DB에는 상품 생성용 `V2`가 이미 적용돼 있었습니다. 이후 같은 버전의 채팅 마이그레이션을 인식하면서 Flyway 체크섬 검증이 실패해 애플리케이션이 시작되지 않았습니다.

Flyway는 적용한 마이그레이션의 버전과 체크섬(파일 내용의 검증 값)을 `flyway_schema_history`에 기록합니다. DB에 기록된 `V2`의 내용과 코드에서 확인한 `V2`의 내용이 서로 달라 문제가 발생했습니다.

### 해결 과정

1. 공유 DB의 `V2`가 상품 생성에 이미 사용됐고, 채팅용 마이그레이션은 아직 적용되지 않았음을 확인했습니다.
2. 채팅 파일의 버전을 다른 번호로 변경했습니다.

   ```text
   변경 전: V2__create_chat_tables.sql
   변경 후: V2026101302__create_chat_tables.sql
   ```

3. 채팅 설계 문서에 적힌 파일명 참조도 새 이름으로 수정했습니다. SQL 내용은 변경하지 않았습니다.

이렇게 기존 상품 `V2`와 채팅 마이그레이션을 서로 다른 버전으로 구분했습니다. 이미 적용된 마이그레이션을 바꾼 것이 아니라 **아직 적용되지 않은 채팅 파일의 번호를 바꾼 사례**입니다.

### 확인 방법과 결과의 범위

다음 조회로 대상 DB의 적용 버전, 순서, 성공 여부를 확인할 수 있습니다.

```sql
SELECT installed_rank, version, description, script, success
FROM flyway_schema_history
ORDER BY installed_rank;
```

대상 환경에서 Flyway가 활성화된 상태로 애플리케이션을 시작하고 검증 오류가 없는지, 채팅 마이그레이션이 성공으로 기록됐는지 확인합니다. 현재 `local` 프로필은 `FLYWAY_ENABLED`의 기본값이 `false`이므로 단순 실행만으로 마이그레이션 검증이 수행됐다고 판단하지 않습니다. **이번 문서 작성에서는 실제 DB에 접속하거나 재현 테스트를 실행하지 않았습니다.**

현재 저장소 파일의 버전 순서는 판매글 `V2026101201`, 문서 `V2026101301`, 채팅 `V2026101302`, 상품 `V2026101303`입니다. 이는 파일 번호의 순서이며, 특정 DB에 실제로 어떤 순서로 적용됐는지는 위 이력에서 확인해야 합니다.

### 재발 방지

- 새 마이그레이션을 만들 때 다른 브랜치의 파일과 대상 DB의 적용 이력을 함께 확인해 버전 중복을 피합니다.
- 이미 적용된 마이그레이션은 수정하지 않고 새로운 마이그레이션으로 변경합니다.
- DB 이력 수정이나 초기화 전에 실제 적용 상태와 데이터 보존 필요성을 확인합니다.

**근거:** `pjmoo` 커밋 `cfe2229`, `src/main/resources/db/migration/`, `application-local.yaml`.

## 2. 잘못된 경로 변수에 500 응답

- **증상:** 숫자 ID가 필요한 `/api/v1/products/abc` 같은 요청에 서버 오류 500이 반환됐습니다.
- **원인:** 경로 변수 변환 중 발생한 `TypeMismatchException`이 입력 오류로 처리되지 않았습니다.
- **해결:** `GlobalExceptionHandler`에서 해당 예외를 `INVALID_INPUT`으로 처리해 400을 반환하도록 변경했습니다.
- **확인:** `GlobalExceptionHandlerTest`에 잘못된 타입의 경로 값에 대한 400 응답 검증이 추가됐습니다.
- **근거:** 커밋 `c157f6a`.

## 3. 판매글 작성 요청의 401 응답

- **증상:** 로그인 후 판매글 작성 요청이 401을 반환했습니다.
- **원인:** 판매글 화면이 사용하는 `window.PastportApi.fetch`가 정의되지 않아 인증 토큰을 포함한 요청이 이루어지지 않았습니다.
- **해결:** `auth.js`에 인증 헤더를 붙이는 공통 fetch를 추가했습니다. 401이면 토큰을 한 번 갱신하고 요청을 재시도하며, 일회용 Refresh Token의 동시 갱신은 하나의 진행 중인 요청으로 합칩니다.
- **확인:** 브라우저 개발자 도구에서 요청의 인증 여부와 응답 상태를 확인하고, 세션이 만료됐다면 다시 로그인합니다. 토큰 값 자체는 공유하거나 기록하지 않습니다.
- **근거:** 커밋 `ec28fd4`, `src/main/resources/static/js/auth.js`.

## 4. 이메일 인증 조회 실패 시 사용자가 삭제될 위험

- **증상:** Supabase 인증 확인 조회가 실패하면 인증 여부를 알 수 없는데도 미인증 계정 정리 대상이 될 수 있었습니다.
- **해결:** `VerificationStatus`(`CONFIRMED`, `UNCONFIRMED`, `UNKNOWN`, `EXPIRED`)를 도입했습니다. 조회가 실패한 `UNKNOWN`은 삭제하지 않습니다. 정리 유예 시간도 조정하고 인증 메일 재전송을 보완했습니다.
- **함께 고친 문제:** 만료되거나 잘못된 쿠키가 남아 있으면 로그인이 막혔습니다. `CookieBearerTokenFilter`가 `/api/auth/` 요청에서는 쿠키 토큰을 무시하도록 했습니다.
- **확인:** `EmailVerificationServiceTest`, `SupabaseAuthClientTest`, `UnverifiedUserCleanerTest`가 추가됐습니다.
- **근거:** 커밋 `4d909f4`, `ff9d1d2`.

## 5. 화면 경로를 직접 입력하면 빈 화면(401)이 뜸

- **증상:** 주소창에 `/market`, `/passport` 같은 화면 이름 경로를 입력하거나 그 상태에서 새로고침하면 본문 없는 401 응답으로 빈 화면이 나옵니다. 존재하지 않는 경로(`/nope`)도 404가 아니라 같은 401이 반환됩니다.
- **원인:** 이 앱은 `/`(index.html) 한 곳에서 화면을 바꾸는 단일 페이지 구조라서 서버에는 `/market` 같은 경로가 없습니다. `SecurityConfig`가 허용한 경로 외에는 모두 인증을 요구해서, 화면 이름 경로와 없는 경로가 인증 문제(401)로 먼저 걸렸습니다.
- **해결:**
  - `PageController`를 추가해 화면 이름 경로(`/market`, `/passport` 등)를 `index.html`로 넘깁니다. 열리는 화면은 첫 화면이며, 해당 화면으로 바로 이동하지는 않습니다.
  - `SecurityConfig`에서 `/api/`로 시작하지 않는 GET 요청은 인증 대상에서 뺐습니다. 없는 경로는 공통 형식의 404(`WEB_404`)로 응답합니다. `/api/**`는 기존대로 인증이 필요합니다.
- **확인:** 새 코드로 서버를 띄워(포트 8081) 요청해 확인했습니다. `/market`, `/passport`는 200(HTML), `/nope`와 `/pages/ai.html`은 404(JSON), `/api/v1/products`는 401이었습니다. 이 항목에 대한 자동 테스트는 추가하지 않았습니다.
- **주의:** 화면 이름이 새로 생기면 `PageController`의 경로 목록에도 추가해야 합니다. 판매글 상세 링크는 `/?listingId=번호` 형태라서 이 경로와 무관합니다.

## 6. 알려진 한계: `/error` 직접 접속 시 500

- **증상:** 주소창에서 `/error`를 직접 열면 `{"status":999,"error":"None"}`과 함께 500이 반환됩니다.
- **원인:** `/error`는 서버 내부에서 오류가 났을 때 Spring이 오류 정보를 넘겨 호출하는 경로입니다. 오류 정보 없이 직접 열면 정상적인 오류 상태가 없어 이런 응답이 나옵니다. `SecurityConfig`가 이 경로를 허용(`permitAll`)해 두었기 때문에 직접 접속이 가능합니다.
- **현재 처리:** 수정하지 않았습니다. 일반 사용자가 열 일이 거의 없고, 실제 오류 응답이 `/error`로 전달되는 흐름에도 영향이 없습니다. 허용 목록에서 빼면 오류 응답이 401로 바뀔 수 있어 별도 검증 없이 건드리지 않았습니다.
- **필요하면:** `/error` 직접 접속은 404로 응답하도록 처리하되, 실제 오류 전달 흐름(오류 응답이 JSON으로 나가는지)을 테스트로 확인하고 진행합니다.

## 7. 마켓 AI 요약에서 세션 만료 시 실패

- **증상:** 홈 상품 카드 레이아웃이 어긋났고, 마켓 AI 요약이 세션 갱신을 처리하지 못했습니다.
- **해결:** CSS를 보정하고 `market.js`가 세션을 갱신하도록 고쳤습니다.
- **근거:** 커밋 `1969dc9`(11/02).

## 8. 병합 충돌을 해결하며 계약이 어긋난 문제

`upstream/main`을 병합할 때마다 팀의 기준을 따르면서 내 변경을 함께 유지해야 했습니다.

| 날짜 | 충돌 지점 | 처리 | 커밋 |
| --- | --- | --- | --- |
| 10/20 | `SecurityConfig` 허용 경로 | 양쪽 경로를 병합, 내 제품 화면은 upstream 기준 | `2018dc3` |
| 10/28 | 이력 마이그레이션 번호 | 팀의 `V2026102801`을 따름 | `f29163f` |
| 11/02 | AI·이전 검증 인증 필수화와 로그인 이어가기 | 두 동작을 모두 유지 | `dd6a093` |
| 11/03 | `index.html`, `DeployMigrationPostgresTest` | 스타일시트는 양쪽 유지, `auth.js` 캐시 버전 갱신, 팀의 `migratePreDeploy` 방식 채택 후 중복된 `migrateBeforeDeploy` 제거 | `74ca1c5` |
| 11/03 | `mypage.js` | 제품 목록 API의 페이지 응답(`items`)과 검증은 팀 hotfix를 따르고, 문서 목록·개수 집계는 `data.items` 기준으로 유지 | `d6a180a` |

- **병합 직후 문제:** 병합한 마이페이지가 페이지 응답 계약과 팀 테스트를 만족하지 않았습니다.
- **해결:** 목록 요소가 없으면 건너뛰고, 문서 조회 실패는 문서 영역에만 알리며, 제품이 없을 때 버튼 문구를 계약에 맞췄습니다(`cb5af2f`).

## 9. 배포용 Flyway가 공유 DB와 어긋남

- **증상:** AWS 첫 배포 전 점검에서 공유 Supabase DB와 `main`의 Flyway 이력이 맞지 않았습니다.
- **원인:**
  1. `ignore-migration-patterns: "*:*"` 때문에 이미 적용된 번호의 새 SQL이 오류 없이 건너뛰어집니다.
  2. `purchase_requests`와 `transaction_change_reviews` DDL이 자동 적용되지 않는 `db/pending`에만 있었습니다.
  3. 이력 공개·검토 컬럼 3개(`product_events.visibility`, `product_events.public_text_reviewed`, `audit_logs.public_reason_reviewed`)가 이미 적용된 번호의 DB에는 없었습니다.
- **해결:** 배포 프로필(`supabase`, `prod`)만 `db/deploy`를 추가로 읽게 했습니다. 이미 있으면 건너뛰는(멱등) 마이그레이션 `V2026110301~04`를 추가했습니다. 기존 마이그레이션, `db/pending`, `local` 프로필은 수정하지 않았습니다.
- **확인:** `DeployMigrationPostgresTest`가 빈 DB, 기존 방식으로 만든 DB, 컬럼이 빠진 DB, 재적용을 검증합니다. 이 테스트는 `DEPLOY_DDL_TEST_ENABLED=true`와 DB 접속 환경 변수가 있을 때만 실행됩니다.
- **남은 확인:** 공유 DB의 `flyway_schema_history`를 직접 확인하지 못했습니다. 같은 번호가 다른 SQL로 적용된 마이그레이션이 더 있는지, 취소 메타데이터 제약(`V2026110302`)이 기존 `purchase_requests` 데이터에 걸리는지 확인이 필요합니다.
- **근거:** 커밋 `b10300b`, `docs/FLYWAY_DEPLOY_REPAIR.md`.

## 10. 구버전 소유권 마이그레이션(2702·2707)이 적용된 DB의 기동 실패

- **증상:** 구버전이 적용된 DB에서 `V2026102803`이 `ownership_transfers`를 다시 만들려다 `relation "ownership_transfers" already exists`로 실패해 서버가 뜨지 않습니다. 무시 패턴으로도 막을 수 없습니다.
- **원인:** 옛 `2702`와 `2707`은 현재 `2803`과 `2804`의 SQL과 바이트 단위로 같고 체크섬도 같습니다. 구조와 데이터는 맞고 이력의 번호만 달랐습니다.
- **해결:** `scripts/db/normalize-ownership-version.psql`을 추가했습니다. 이력의 `version`과 `script`만 `2702→2803`, `2707→2804`로 바꾸고, 조건이 하나라도 어긋나면 롤백합니다. 두 번 실행해도 결과가 같습니다.
- **확인:** `NormalizeOwnershipVersionPostgresTest` 9개를 격리 스키마에서 검증했습니다.
- **주의:** 서버를 멈추고 전체 백업을 받은 뒤 대상 DB 담당자가 실행합니다. 소유권 정규화를 먼저, 요약 정규화(`normalize-summary-version`)를 그다음에 실행합니다.
- **근거:** 커밋 `040a4a7`, `docs/FLYWAY_DEPLOY_REPAIR.md`, `docs/FLYWAY_VERSION_ALIGNMENT.md`.

## 11. 단일 세션 도입 후 단위 테스트와 포맷 검사 실패

- **증상:** 중복 로그인 제어와 프로필 사진 기능을 넣은 뒤 `AuthService`, `RefreshTokenStore` 단위 테스트가 새 동작과 맞지 않았고 Spotless 검사에도 걸렸습니다.
- **해결:** 단일 세션 기준으로 테스트를 갱신하고 예외 시그니처를 정리했으며 Spotless 포맷을 적용했습니다.
- **근거:** 커밋 `cab98a7`, `2edcfc7`.

## 12. HMAC 지문·AES 암호문에서 생길 수 있는 오류 (예상 가이드)

> **이 항목은 실제로 발생한 트러블이 아닙니다.** 커밋 기록에는 HMAC·AES 관련 오류 수정이 없습니다. `SerialNumberProtector`, `application-*.yaml`, 마이그레이션을 읽고 정리한 **발생 가능한 오류와 대처**입니다. 실제로 겪으면 날짜와 원인을 채워 위 항목처럼 바꿉니다.

### 구조 요약

- 마스터 비밀값 `app.product.serial-secret`(`PRODUCT_SERIAL_SECRET`)에서 지문용 키와 암호화용 키를 따로 만듭니다.
- 지문(`serial_number_fingerprint`, HMAC-SHA256, 64자 hex)은 중복 확인용입니다. 유니크 제약은 `(manufacturer_normalized, serial_number_fingerprint)`입니다.
- 암호문(`serial_number_ciphertext`, AES-256-GCM, Base64)은 `IV 12바이트 + 암호문 + 인증 태그`를 한 문자열로 저장합니다.

### 오류 유형별 정리

| # | 증상 | 원인 | 확인 방법 | 대처 |
| --- | --- | --- | --- | --- |
| 1 | 앱이 시작되지 않음: `app.product.serial-secret must be at least 32 characters` | 비밀값이 비었거나 32자 미만 | 환경 변수 `PRODUCT_SERIAL_SECRET`의 존재와 길이 | 32자 이상으로 설정. 값 자체를 로그나 채팅에 남기지 않음 |
| 2 | 앱이 시작되지 않음: placeholder를 해석할 수 없다는 오류 | `prod` 프로필은 기본값 없이 `${PRODUCT_SERIAL_SECRET}`만 참조하는데 배포 환경에 변수가 없음 | 배포 환경 변수와 `.env`(`.env.example`에는 키만 있고 값이 비어 있음) | 배포 전 변수 등록을 점검 항목에 포함 |
| 3 | 같은 노트북인데 중복 검사에 걸리지 않음 | 환경(local, 배포, 테스트)마다 비밀값이 달라 같은 번호의 지문이 다름 | 같은 번호를 두 환경에서 `fingerprint`로 계산해 비교 | 같은 DB를 쓰는 환경은 같은 비밀값을 사용 |
| 4 | 시리얼 번호 복호화 시 `Failed to decrypt serial number` | 비밀값이 바뀌었거나 암호문이 변조됨. GCM은 키·내용이 맞지 않으면 인증 태그 검증에 실패 | 최근 비밀값 변경 이력, 해당 행의 암호문 | 이전 비밀값으로 복구. 키 교체는 아래 "키 교체" 참고 |
| 5 | 복호화 시 `IllegalArgumentException`(Base64 오류) 등 예상 밖 예외 | 암호문 값이 잘리거나 손상됨. `decrypt`는 `GeneralSecurityException`만 감싸서, Base64 해석 오류와 너무 짧은 값은 다른 예외로 나올 수 있음 | 컬럼 길이(`VARCHAR(512)`)와 저장된 값의 길이 | 손상된 행을 백업에서 복구. 복호화 호출부를 만들 때 예외 처리를 함께 확인 |
| 6 | 같은 제품이 `AB-12`와 `AB12`로 각각 등록됨 | 정규화가 공백 제거와 대문자 변환만 함. 하이픈은 시리얼 번호에서는 제거하지 않음(제조사명은 하이픈도 제거) | 두 값의 `normalize` 결과 비교 | 규칙을 바꾸려면 아래 "정규화 규칙 변경" 참고 |
| 7 | 동시에 같은 제품을 등록하면 한 요청이 DB 오류 | 사전 중복 검사를 둘 다 통과하고 유니크 제약 `uk_products_manufacturer_serial`에서 막힘 | 로그의 제약 위반 메시지 | 제약이 막는 것은 정상 동작. 사용자에게는 중복 안내(409 계열)로 응답되는지 `ProductService` 테스트로 확인 |
| 8 | 지문 컬럼 저장 실패 | 지문은 항상 64자 hex이므로 컬럼(`VARCHAR(64)`)과 맞음. 다른 형식의 값을 직접 넣었을 때만 발생 | 직접 INSERT 여부 | 지문은 `SerialNumberProtector.fingerprint`로만 생성 |
| 9 | 테스트에서 컨텍스트 로딩 실패 | 테스트 설정의 비밀값이 32자 미만 | `application-test.yaml`의 `serial-secret` | 32자 이상으로 유지(현재 설정은 충족) |

### 키 교체 시 주의 (가장 큰 운영 위험)

- 비밀값을 바꾸면 **지문과 암호문이 모두 기존 값과 호환되지 않습니다.** 기존 제품의 중복 검사가 깨지고 복호화도 실패합니다.
- 교체가 필요하면 이전 비밀값으로 기존 암호문을 복호화한 뒤 새 비밀값으로 지문과 암호문을 다시 만들어 갱신하는 이전 작업이 필요합니다. 이 프로젝트에는 이런 재처리 코드가 없습니다.
- 비밀값을 잃으면 기존 암호문은 복구할 수 없습니다. 비밀값은 별도 안전한 곳에 백업합니다.

### 정규화 규칙 변경 시 주의

- `normalize`의 규칙을 바꾸면 같은 번호의 새 지문이 기존 지문과 달라집니다. 기존 행의 지문을 다시 계산하는 마이그레이션이 필요합니다. 이미 적용된 Flyway 마이그레이션 파일은 수정하지 않고 새 파일로 추가합니다.

### 로그와 응답

- 시리얼 번호 원문, 비밀값, 복호화 결과를 로그나 API 응답에 남기지 않습니다. 현재 등록 응답에는 시리얼 번호가 포함되지 않습니다.
- 예외 메시지(`Failed to decrypt serial number`)에는 값이 들어 있지 않습니다. 디버깅하려고 값을 출력하는 코드를 추가하지 않습니다.

**근거:** `product/service/SerialNumberProtector.java`, `application-local.yaml`, `application-prod.yaml`, `application-test.yaml`, `.env.example`, `V2026101303__create_product_tables.sql`, `V2026102001__align_product_p1_requirements.sql`.

## 13. 낮은 번호의 마이그레이션이 건너뛰어져 로컬 앱이 시작되지 않음 (11/03)

- **증상:** 로컬 실행이 `Schema validation: missing column [recipient_accepted_at] in table [ownership_transfers]`로 실패했습니다. 같은 로그에 `Successfully validated 23 migrations`와 `Schema "public" is up to date`가 찍혔습니다.
- **원인:**
  - 로컬 DB에는 내 `V2026110401`(프로필 사진)이 11/02 23:57에 먼저 적용돼 있었습니다.
  - 이후 upstream을 병합하며 더 낮은 번호인 `V2026110305`(받는 사람 동의 컬럼)와 `V2026110202`가 들어왔습니다.
  - Flyway는 "이미 적용된 최대 버전보다 낮은데 적용되지 않은" 마이그레이션을 실행하지 않습니다. `ignore-migration-patterns: "*:*"` 때문에 이것이 오류로 보고되지 않았습니다.
  - 컬럼이 없는 DB와 컬럼을 기대하는 코드(`ddl-auto: validate`)가 어긋나 시작이 막혔습니다.
- **해결:** 환경 변수 `FLYWAY_OUT_OF_ORDER=true`(`application-local.yaml`에 있는 설정)로 앱을 한 번 실행했습니다. `V2026110202`, `V2026110305` 2개가 순서를 어기고 적용됐고(`success=true`), 앱이 정상 기동했습니다. 적용된 뒤에는 환경 변수가 필요 없습니다.
- **확인:** `flyway_schema_history`에서 두 버전이 `success=t`이고 `recipient_accepted_at` 컬럼이 존재함을 조회로 확인했습니다. 적용 대상 `ownership_transfers`는 0건이었습니다.
- **배포 영향:** 공유 DB의 최대 버전이 `2026103001`이라면 미적용 마이그레이션이 모두 더 높은 번호라서 순서대로 적용되고 건너뛰는 것이 없습니다. 이 값은 직접 확인하지 못한 가정이며, `2026110401` 이상이 이미 적용돼 있으면 같은 문제가 납니다. 배포 환경에서는 시작 단계의 스키마 검증이 실패로 드러납니다.
- **재발 방지:**
  - 새 마이그레이션 번호는 `upstream/main`의 최대 번호 바로 다음으로 정하고, 날짜를 앞질러 쓰지 않습니다. 병합 순서가 바뀌면 번호를 다시 붙입니다.
  - 배포 전에 `flyway_schema_history`의 최대 버전과 저장소 파일을 비교해 "최대 버전보다 낮은데 미적용" 파일이 없는지 확인합니다.
  - 로컬에서 병합 후 비슷한 오류가 나면 `FLYWAY_OUT_OF_ORDER=true`로 한 번 실행합니다.
- **근거:** 로컬 시작 로그, `flyway_schema_history` 조회, `application-local.yaml`, 커밋 `5441a22`(`V2026110305`), `1eea4eb`(`V2026110401`).
