# 공유 원본 DB 적용 — S1 상태 확인 및 S2 NORMALIZE 승인 요청서

작성일시: 2026-11-04 00:01 KST
상태: `AWAITING_HUMAN_APPROVAL` (S2 `NORMALIZE` 승인 A1 대기)
실행 기준 SHA: `2c218e575f662ff5fb0c93318cef8783e5df2c8e`

---

## 1. 완료한 준비 및 S1(원본 현재 상태) 결과 요약

### 1.1 고정 실행 소스 (`worktree`)
- **경로**: `C:\restricted-backup\worktree-2c218e5`
- **`HEAD`**: `2c218e575f662ff5fb0c93318cef8783e5df2c8e` (`git status --porcelain` 무변경 확인)
- **`source` digest**: `6f86f38cad45a0b94dd5bb1860d7376fd1e33a3a7bb5af37409a3401498fe16f` (리허설 소스 47개 파일 해시와 100% 일치)

### 1.2 증거 보관 디렉터리 및 파일 참조 보관
- **증거 경로**: `C:\restricted-backup\evidence\shared-apply-20261103-2353\`
- **생성된 읽기 전용 증거**: `before-check.txt`, `before-manifest.txt`, `plan-normalize.txt`, `plan-normalize-stdout.txt`, `backup-toc.txt`
- **파일 참조 식별자 목록**: `file-references-inventory.txt` (총 6건: `documents` 3, `product_images` 2, `draft_images` 1 — 저장소 밖 제한 경로 보관 완료)

### 1.3 적용 직전 신규 백업 (읽기 전용 `pg_dump -Fc`)
- **백업 파일**: `C:\restricted-backup\pre-apply-20261103-2353.dump`
- **생성 시각**: `2026-11-03 23:53:29 +09:00 (KST)`
- **파일 크기**: `74,008` bytes (`pg_restore --list` 목차 검증 결과 `TABLE DATA` 20개, 이전 덤프와 데이터 변동 없음)
- **SHA-256**: `21d3020a5ecc578dbcfe873f2a37ad0a4dafab2f1675592d8dedae5b99416141`

### 1.4 S1 원본 `check` / `manifest` 결과 (`exit=0`)
- **실행 시각**:
  - `check`: `2026-11-03 23:59:12 +09:00` (`exit=0`)
  - `manifest`: `2026-11-03 23:59:22 +09:00` (`exit=0`)
- **대상 확인**: 공유 원본 Supabase (`PostgreSQL 17.6`, `database=postgres`, `schema=public`, `transaction_read_only=on`)
- **`target` fingerprint**: `2836cd4516d4977eaa6599039b29c7a8fce6de36c19023efed7ecb85c53f3f6d`
- **`URL_SHA256`**: `b653d19bb300dcf45c00a34f6551739155d88992beeeee4219a78464b95b7ee2`
- **Flyway 이력**: **22행** (마지막 `2026103001`, 결번 rank 17, 구버전 `2026102701`/`2026102702`/`2026102706`/`2026102707` 4건 존재, 후속 `PENDING` 10건)
- **정규화 가드 대상 14개 제약 MD5**: **14/14 모두 일치** (복원본 전용 `check-fix.sql` 불필요·미실행)
- **현재 테이블 건수 및 상태**:
  - **보존 대상(`PRESERVE`)**:
    - `users`: 1행 (`digest=2a251d9cd3cd429633706d368bcbc88d`)
    - `flyway_schema_history`: 22행 (`digest=ee87b65ec3d24712890c4ef84897b855`)
    - `user_profile_images`: 미존재 (S4 `2026110401`에서 생성 예정)
  - **수습 대상 업무 테이블(`DELETE` 후보)**:
    - `products=2`, `laptop_details=2`, `product_images=2`, `product_field_authors=5`, `documents=3`, `listing_drafts=1`, `draft_images=1`, `chat_rooms=1`, `chat_messages=4`, `chat_read_positions=2` (그 외 allowlist 테이블 0건)
  - **예상된 수습 전 고아 참조**:
    - `products.owner_id=2`, `documents.uploader_id=3`, `chat_rooms.seller_id=1`, `chat_rooms.inquirer_id=1`, `chat_messages.sender_id=4`
    - `CHECK` 위반: 0건

---

## 2. S2 원본 `NORMALIZE plan` 정보 (승인 요청 대상)

- **Plan 파일**: `C:\restricted-backup\evidence\shared-apply-20261103-2353\plan-normalize.txt`
- **생성 시각**: `2026-11-03T14:59:27.074200100Z` (`2026-11-03 23:59:27 KST`, 만료 30분)
- **Operation**: `NORMALIZE`
- **Target**: `2836cd4516d4977eaa6599039b29c7a8fce6de36c19023efed7ecb85c53f3f6d` (`postgres/public`)
- **Source SHA**: `2c218e575f662ff5fb0c93318cef8783e5df2c8e` (`source=6f86f38cad45a0b94dd5bb1860d7376fd1e33a3a7bb5af37409a3401498fe16f`)
- **Scope (변경 범위)**:
  - `history version/script only: purchase 2701->2103, summary 2706->2701, ownership 2702->2803 and 2707->2804; all other history fields preserved`
  - (`installed_rank`, `description`, `type`, `checksum`, `success` 및 업무 데이터 무변경)
- **`plan_sha256`**:
  `473af201b70c44a10faaaa34a89eab7ca66fcae859ad785b3b1ed33164cc31ef`

| 변경 전 | 변경 후 | 유지할 checksum |
|---|---|---:|
| `2026102701` 구매 요청 (`V2026102701__create_purchase_requests.sql`) | `2026102103` 구매 요청 (`V2026102103__create_purchase_requests.sql`) | -778179566 |
| `2026102706` summary (`V2026102706__support_listing_summary_refresh.sql`) | `2026102701` summary (`V2026102701__support_listing_summary_refresh.sql`) | 620449880 |
| `2026102702` ownership (`V2026102702__create_ownership_tables.sql`) | `2026102803` ownership (`V2026102803__create_ownership_tables.sql`) | -1318655273 |
| `2026102707` pending slot (`V2026102707__add_ownership_pending_slot.sql`) | `2026102804` pending slot (`V2026102804__add_ownership_pending_slot.sql`) | -1107274211 |

---

## 3. 다음에 필요한 입력 (인간 승인 A0 확인 + A1 승인)

쓰기 중단 완료 및 백업·후속 migration 범위(A0)와 위 `NORMALIZE plan`(A1)을 승인하시는 경우, 아래 문구를 복사하여 전달해 주세요.
(승인 시점이 plan 생성 시각 `23:59:27 KST`로부터 30분을 초과한 경우, 즉시 새 plan을 다시 생성하여 해시를 안내해 드립니다.)

```text
A0 확인(원본 대상·쓰기 중단·백업 pre-apply-20261103-2353.dump·후속 migration 범위) 및 원본 NORMALIZE 승인.
대상=2836cd4516d4977eaa6599039b29c7a8fce6de36c19023efed7ecb85c53f3f6d (postgres/public),
소스=2c218e575f662ff5fb0c93318cef8783e5df2c8e,
plan_sha256=473af201b70c44a10faaaa34a89eab7ca66fcae859ad785b3b1ed33164cc31ef.
네 이력의 version/script 변경만 승인한다.
```
