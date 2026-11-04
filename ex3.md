# Pastport 공유 원본 DB 적용 — S2 정규화 완료 및 PURGE 승인 대기 보고

작성일: 2026-11-04 (KST)
실행 기준 후보 SHA: `2c218e575f662ff5fb0c93318cef8783e5df2c8e` (이후 main에서 `db/`·`scripts/db/`·Java migration 변경 없음)
실행 도구: `scripts/db` `ConvergenceTool`, lineage=`legacy`, 스키마 `public`
실행 주체: Claude Code(AI 에이전트), 사용자(pjmoo) 승인 하에 실행

## 1. 현재 상태 요약 (2026-11-04 갱신: 공유 DB 적용 완료)

| 단계 | 상태 |
|---|---|
| S1 복원본 리허설 | 완료 (PASS) |
| A0 쓰기 중단 확인 | 확인함 (§2). 앱 계정 연결 0건, PURGE 직전에도 재확인 |
| 쓰기 직전 백업 | 완료 |
| 공유 DB 사전 `check` | 완료 (리허설과 내용 동일) |
| S2 NORMALIZE (공유 DB) | **PASS** |
| A2 PURGE (공유 DB) | **PASS** (승인 plan 해시 `3c562734…7a2f` 그대로, 유효시간 내 실행) |
| 후속 migrate 10개·strict validate | **PASS** |
| 내부 재실행 신규 적용 | **0건** |
| 사후 `check`·`manifest`·보존 데이터 비교 | **PASS** (§10) |
| 앱 기동·AWS 변경·공개 배포 | 수행하지 않음 |
| DB GO | **DB 담당자 확인 대기** (에이전트는 GO를 부여하지 않음) |

## 2. A0 — 쓰기 중단 확인

공유 DB의 `pg_stat_activity`를 읽기 전용 세션으로 조회했다 (2026-11-04 00:2x KST).
- 애플리케이션 계정(`postgres.*`)의 연결: **0건**
- 존재한 연결은 Supabase 내부 프로세스뿐: PostgREST, pgbouncer(Supavisor), supabase_admin(exporter)
- 한계: 조회 시점의 스냅샷이며, 이후 팀원 IDE `bootRun`·워커·EC2가 켜지면 쓰기가 생길 수 있다.
  PURGE와 migrate 실행 전까지 팀의 쓰기 중단 유지를 확인해야 한다.

## 3. 쓰기 직전 백업

| 항목 | 내용 |
|---|---|
| 파일 | `pre-write-20261104-002517.dump` (`C:\restricted-backup\`, 저장소 밖) |
| 크기·SHA-256 | 74,008바이트 / `9ca8f5cfdd379cc7bbd184dc889f64eee9bfa328dc853e9d3879913f58eda9f7` |
| 범위 | `public` 스키마 논리 덤프(`-Fc`, `--no-owner --no-privileges`), 읽기 전용 세션 |
| 검증 | `pg_restore --list` 결과 테이블 데이터 20개 |
| 앞선 백업 | `shared.dump`(리허설), `pre-apply-20261103-215025.dump` — 모두 같은 크기·내용 |

## 4. 사전 비교 — 공유 DB vs 리허설 원본 복원본

- 행 수: 11개 업무·보존 테이블 모두 동일
- 내용 해시(시간대 `UTC` 고정, `md5(string_agg(row::text))`): **11개 전부 일치**
  (users, products, documents, chat_messages, chat_rooms, chat_read_positions, laptop_details, listing_drafts, product_images, draft_images, product_field_authors)
- 고아 FK·CHECK 위반 점검: 전부 동일 (고아: products.owner 2, documents.uploader 3, chat_rooms.seller 1·inquirer 1, chat_messages.sender 4)
- `NOT_RUN`: 0
- 참고: 도구가 출력하는 테이블 digest는 서버 시간대 표기 때문에 리허설 때와 문자열이 다르다. 내용이 달라진 것이 아니며 위 UTC 직접 비교로 확인했다.
  또한 `catalog=` 해시 하나가 다른 것은 복원본에서 CHECK 5개를 보정했기 때문이며, 공유 원본은 기대 14개 제약이 모두 일치한다(인계 자료 참조).

## 5. S2 NORMALIZE 실행

| 항목 | 내용 |
|---|---|
| plan | `shared-plan-normalize.txt` |
| plan SHA-256(승인 해시) | `cda226d9baec1d7d52a028a3eab755b7c9e1f0279a6fb2e26adcf7f65cfc3305` |
| 범위 | `history version/script only: purchase 2701->2103, summary 2706->2701, ownership 2702->2803 and 2707->2804; all other history fields preserved` |
| 결과 | `normalize committed` (오류 없음) |

사후 확인 (`evidence/shared-history-after-normalize.txt`):

| installed_rank | 전 | 후 | checksum |
|---|---|---|---|
| 15 | 2026102701 | 2026102103 | -778179566 |
| 16 | 2026102702 | 2026102803 | -1318655273 |
| 19 | 2026102706 | 2026102701 | 620449880 |
| 20 | 2026102707 | 2026102804 | -1107274211 |

- 이력 행 수 22행 유지, success 전부 `t`, 위 4건 외 변경 없음, rank 17 결번은 복원하지 않음
- 업무 데이터는 이 단계에서 변경되지 않았다.

## 6. PURGE — 승인 요청 (→ 승인되어 실행 완료, §10 참조)

PURGE plan: `shared-plan-purge.txt`, SHA-256 `3c562734873815c7d68752e07dd02ed4c91c6d09e63363a2d789176ce0107a2f`
(정규화 이후 새로 생성. plan 유효시간 30분이므로 만료되면 재생성하여 범위가 같은지 다시 확인한다.)

삭제 대상은 아래 업무 테이블의 **전체 행**이다.

| 테이블 | 삭제 행 수 |
|---|---|
| products | 2 |
| documents | 3 |
| chat_messages | 4 |
| chat_rooms | 1 |
| chat_read_positions | 2 |
| product_images | 2 |
| product_field_authors | 5 |
| laptop_details | 2 |
| listing_drafts | 1 |
| draft_images | 1 |
| 그 외(audit_logs, document_extractions, event_documents, listing_images, listings, ownership_transfers, product_events, purchase_requests 등) | 0 |

보존: `users` 1행, `user_profile_images`, `flyway_schema_history` 22행, 시퀀스

주의:
- 삭제는 되돌릴 수 없고, 복구 수단은 §3의 백업뿐이다 (복원 시 제약 표기가 달라지는 알려진 문제가 있다).
- SQL 정리는 업로드 파일, Redis, 서명 URL을 지우지 않는다. 스토리지 담당자의 별도 승인·처리가 필요하다.
- 사용자 계정이 1명뿐이며 삭제 대상 행은 모두 존재하지 않는 사용자를 참조하는 고아 데이터다(요청서 기준 폐기 가능 테스트 데이터).

## 7. 승인 후 이어서 할 일

1. 승인 확인 → (plan 만료 시 재생성·범위 재확인) → PURGE 실행
2. 새 plan 없이 `migrate`: 10개 적용(2026103002 ~ 2026110402), strict validate, 재실행 0건 확인
3. 사후 `check`·`manifest`: 고아 0, CHECK 위반 0, 신규 FK·제약 5개 검증, 사용자 digest 비교
4. 결과를 §5와 같은 형식으로 보고서에 추가
5. DB 담당자 DB GO → 이후 배포 담당자가 앱 시작 (DB 정리 전 시작 금지)

## 8. 실패·중단 시

- PURGE는 단일 트랜잭션이라 오류가 나면 롤백된다. 이후 상태를 재점검한 뒤 새 plan으로 재개한다.
- migrate는 일부 migration이 이미 커밋될 수 있다(비트랜잭션 인덱스 포함). 실패 시 쓰기를 멈춘 채 전체 이력과 인덱스 유효성을 수집한다.
- 자동 repair, clean, 이력 수동 편집은 하지 않는다.

## 9. 근거 파일 (`C:\restricted-backup\evidence\`, 저장소 밖)

`shared-check-pre.txt`, `shared-plan-normalize.txt`, `shared-plan-purge.txt`, `shared-history-after-normalize.txt`,
`shared-content-hash.txt`, `clone0-content-hash.txt`, `shared-migrate.txt`, `shared-check-post.txt`, `shared-manifest-post.txt`

## 10. 실제 공유 원본 적용 결과 (A2 승인 후)

실행 소스 `2c218e575f662ff5fb0c93318cef8783e5df2c8e`, 실행 시각 2026-11-04 00:4x KST.

실행 직전 확인: PURGE plan 생성 후 약 16분 경과(30분 이내), plan 파일 해시가 승인 해시와 일치, 앱 계정 연결 0건.
도구 자체 가드(대상·소스·내용 변경, 승인 해시, 만료)를 통과했고 `purge committed`로 끝났다. 오류 없음.

| 항목 | 결과 |
|---|---|
| PURGE | PASS — 10개 테이블 23행 삭제 (products 2, documents 3, chat_messages 4, chat_rooms 1, chat_read_positions 2, product_images 2, product_field_authors 5, laptop_details 2, listing_drafts 1, draft_images 1) |
| MIGRATE_VALIDATE | PASS — 10개 적용(2026103002 ~ 2026110402), 최종 버전 2026110402, strict validate 32건 성공 |
| SECOND_MIGRATE_ZERO | YES — `Schema "public" is up to date. No migration necessary.` |
| 필수 FK·CHECK·인덱스 | 정상 — `fk_products_owner`, `fk_documents_product`, `fk_documents_uploader`, `ck_products_owner_id_required`, `ck_products_serial_ciphertext_required` 모두 검증됨(`convalidated=true`) |
| 미검증 제약 / 무효 인덱스 | 0 / 0 |
| 주요 무결성 위반 | 0 — 고아 FK 0, CHECK 위반 0, `NOT_RUN`·`MISSING_PREREQUISITE` 0 |
| 이력 | 32행 전부 success, 실패 0. manifest: SUCCESS 28, OUT_OF_ORDER 4 |
| 보존 데이터 | 일치 — users 1행(내용 해시 `530751cd…8385c6`가 사전과 동일), flyway_schema_history 22행 보존 + 신규 10행, user_profile_images 0행(테이블 신규 생성) |
| SHARED_DB_APPLIED | YES |
| DB_GO | DB 담당자 확인 대기 |

로그의 `already exists, skipping` 경고(2026110301, 2026110304)는 멱등 구문이라 발생했으며 리허설과 같다.

### DB 담당자 확인이 필요한 점
- `manifest`의 `OUT_OF_ORDER` 4건(2026102601, 2026102701, 2026102801, 2026102802)은 적용에 성공한 기존 이력의 순서 표기다.
- 파일 원본(스토리지 업로드 파일), Redis(토큰·세션), 서명 URL은 변경하지 않았다. 삭제된 문서·이미지의 파일은 스토리지 담당자의 별도 승인·처리가 필요하다.
- 롤백이 필요하면 `pre-write-20261104-002517.dump`(§3)로만 복구할 수 있다. 복원 시 제약 표기가 달라지는 알려진 문제가 있다.
- 이후 앱 시작 시 Flyway가 다시 실행되므로, 앱 기동은 DB GO 이후 배포 담당자가 한다. 기능 스위치(`TRADE_P3_ENABLED` 등)는 이 작업의 범위가 아니다.

백업 파일, 비밀번호, 접속 문자열, 프로젝트 식별자, 실제 사용자·문서 원문은 이 문서에 포함하지 않는다.
