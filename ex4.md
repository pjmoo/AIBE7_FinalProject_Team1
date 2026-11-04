# Pastport 공유 원본 DB 적용 — A2 PURGE · migrate · 사후 검사 결과 보고

작성일: 2026-11-04 (KST)
실행 소스: `2c218e575f662ff5fb0c93318cef8783e5df2c8e`
대상: 공유 원본(Supabase) `public` 스키마
도구: `scripts/db` `ConvergenceTool`, lineage=`legacy`
실행 주체: Claude Code(AI 에이전트), 사용자 전달 승인(A2) 하에 실행
앞 단계: S2 NORMALIZE 완료 (`ex3.md` 참조)

## 1. 판정

| 항목 | 결과 |
|---|---|
| PURGE | **PASS** |
| MIGRATE_VALIDATE | **PASS** |
| SECOND_MIGRATE_ZERO | **YES** |
| 필수 FK·CHECK·인덱스 | 정상 |
| 주요 무결성 위반 | **0** |
| 보존 데이터 | 일치 |
| SHARED_DB_APPLIED | **YES** |
| DB_GO | DB 담당자 확인 대기 (에이전트는 부여하지 않음) |
| 앱 기동·AWS 변경·공개 배포 | 수행하지 않음 |

## 2. 승인 내용 (전달받은 A2)

- 승인 plan_sha256: `3c562734873815c7d68752e07dd02ed4c91c6d09e63363a2d789176ce0107a2f`
- 승인 범위: 고정 allowlist 중 현재 plan에 기록된 비어 있지 않은 10개 테이블 23행
- 보존: `users`, `user_profile_images`, `flyway_schema_history`, 시퀀스, 관리 스키마
- 비변경: S3, 로컬 파일, Redis
- 조건: 실행 직전 쓰기 중단 유지·plan 유효시간 확인, 만료·변경 시 실행하지 않고 재승인, 기존 해시 재사용 금지
- 성공 시 후속 migration 10개, strict validate, 재실행 0건, 사후 check·manifest, 보존 데이터 비교까지 이어서 수행

## 3. 실행 직전 확인

| 확인 | 결과 |
|---|---|
| plan 생성 시각 | 2026-11-03T15:27:21Z |
| 실행 시점 | 2026-11-03T15:43Z — 약 16분 경과, 유효시간(30분) 이내 |
| plan 파일 SHA-256 | 승인 해시와 **일치** (`3c562734…7a2f`) |
| 쓰기 중단 | 앱 계정(`postgres.*`) 연결 **0건**. Supabase 내부 프로세스(PostgREST, pgbouncer, supabase_admin)만 존재 |
| 백업 | `pre-write-20261104-002517.dump` (74,008바이트, SHA-256 `9ca8f5cf…a9f7`), PURGE 전에 받았고 그 사이 데이터 변화 없음 |
| 도구 가드 | 대상·소스·내용 변경, 승인 해시, 만료 검사 통과 |

한계: 쓰기 중단은 조회 시점의 연결 스냅샷으로 확인했다. 실행 중 신규 쓰기는 도구의 잠금(5초 lock, 60초 statement timeout)과 내용 재검사로 막혔다.

## 4. 단계별 실행

### 4-1. PURGE
- 결과: `purge committed` (오류 없음, 단일 트랜잭션)
- 삭제 행 수

| 테이블 | 삭제 | 테이블 | 삭제 |
|---|---:|---|---:|
| products | 2 | product_images | 2 |
| documents | 3 | product_field_authors | 5 |
| chat_messages | 4 | laptop_details | 2 |
| chat_rooms | 1 | listing_drafts | 1 |
| chat_read_positions | 2 | draft_images | 1 |
| **합계** | | | **23** |

- 나머지 allowlist 테이블(audit_logs, document_extractions, event_documents, listing_images, listings, ownership_transfers, product_events, purchase_requests 등)은 실행 전 0행이었다.

### 4-2. 후속 migrate (새 plan 없이 `migrate`)
- 현재 버전 2026103001 → **2026110402**, `Successfully applied 10 migrations`, 실행 시간 약 0.7초

| # | 버전 | 설명 |
|---|---|---|
| 1 | 2026103002 | add history publication state |
| 2 | 2026110201 | track document replacements |
| 3 | 2026110202 | index owned product list (Java, 비트랜잭션) |
| 4 | 2026110301 | create purchase requests |
| 5 | 2026110302 | add purchase request cancellation metadata |
| 6 | 2026110303 | create transaction change reviews |
| 7 | 2026110304 | ensure history publication state |
| 8 | 2026110305 | direct transfer recipient consent |
| 9 | 2026110401 | create user profile images |
| 10 | 2026110402 | enforce product document transfer integrity |

- strict validate: `Successfully validated 32 migrations` (적용 전·후 모두)
- 재실행: `Schema "public" is up to date. No migration necessary.` → **신규 적용 0건**
- `already exists, skipping` 경고(2026110301의 purchase_requests·인덱스 3개, 2026110304의 컬럼 3개·인덱스 1개)는 멱등 구문이라 발생했고, 리허설과 같은 결과다.

### 4-3. 사후 check · manifest
- 두 실행 모두 종료 코드 0, `NOT_RUN`·`MISSING_PREREQUISITE` 0

## 5. 사후 검증 결과

| 항목 | 적용 전 | 적용 후 |
|---|---|---|
| products.owner_id 고아 | 2 | **0** |
| documents.uploader_id 고아 | 3 | **0** |
| chat_rooms.seller_id / inquirer_id 고아 | 1 / 1 | **0 / 0** |
| chat_messages.sender_id 고아 | 4 | **0** |
| CHECK 위반 합계 | 0 | 0 |
| 미검증 제약 | 2 | **0** |
| 무효 인덱스 | 0 | 0 |
| 업무 테이블 행 수 (products, documents, chat_messages 등) | 2 / 3 / 4 | **0 / 0 / 0** |

신규 제약 (모두 `convalidated=true`):
`fk_products_owner`, `fk_documents_product`, `fk_documents_uploader`, `ck_products_owner_id_required`, `ck_products_serial_ciphertext_required`

### 보존 데이터 비교
| 대상 | 적용 전 | 적용 후 |
|---|---|---|
| users | 1행, 내용 해시(UTC) `530751cdaf9655e6f29343111e8385c6` | 1행, **해시 동일** |
| flyway_schema_history | 22행 | 32행 (기존 22행 보존 + 신규 10행), 실패 0 |
| user_profile_images | 테이블 없음 | 신규 생성, 0행 |

### Flyway manifest
- SUCCESS 28, OUT_OF_ORDER 4 (2026102601, 2026102701, 2026102801, 2026102802)
- OUT_OF_ORDER는 적용에 성공한 기존 이력의 순서 표기이며 실패나 미적용이 아니다.

## 6. DB 담당자 확인 요청

1. `OUT_OF_ORDER` 4건이 정상 기록으로 인정되는지
2. 결번(`installed_rank=17`)은 런북대로 복원하지 않았음을 확인
3. DB GO 부여 여부 (이 문서는 DB GO가 아니다)

## 7. 이번 작업에서 변경하지 않은 것 / 후속

| 구분 | 내용 |
|---|---|
| 변경하지 않음 | S3·로컬 업로드 파일, Redis(세션·토큰), 서명 URL, 앱 기동, AWS, 공개 배포 |
| 스토리지 담당자 | 삭제된 문서·이미지의 업로드 파일은 별도 승인 후 정리 |
| 배포 담당자 | DB GO 이후에만 앱 시작(시작 시 Flyway 실행). 거래 기능은 `TRADE_PURCHASE_REQUESTS_ENABLED`, `TRADE_P3_ENABLED`, `TRADE_P4_ENABLED`가 켜져 있어야 동작 |
| 인수 시험 | 실제 가입·로그인·제품 등록·문서·채팅·거래·소유권 이전 흐름은 앱 기동 후 별도 확인 (이번 범위 아님) |
| 롤백 | `pre-write-20261104-002517.dump`로만 복구 가능. 복원 시 CHECK 표기가 달라지는 알려진 문제가 있어 복구 절차는 담당자와 협의 |

## 8. 근거 파일 (`C:\restricted-backup\evidence\`, 저장소 밖)

`shared-plan-purge.txt`, `shared-migrate.txt`, `shared-check-post.txt`, `shared-manifest-post.txt`,
`shared-check-pre.txt`, `shared-content-hash.txt`

백업 파일, 비밀번호, 접속 문자열, 프로젝트 식별자, 실제 사용자·문서 원문은 이 문서에 포함하지 않는다.
