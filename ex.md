# Pastport 공유 원본 DB 적용 — 담당자 IDE 에이전트 작업지시서

작성일: 2026-11-03 (KST)
대상 저장소: `prgrms-aibe-devcourse/AIBE7_FinalProject_Team1`
실행 기준 SHA: `2c218e575f662ff5fb0c93318cef8783e5df2c8e`
작업 목적: 이미 리허설한 DB 수습 절차를 **공유 원본 Supabase DB의 public 스키마에 적용**하고 실제 사후 결과를 인계한다.

> 이 문서는 작업 범위와 실행 순서를 지정한다. 문서를 전달받았다는 사실만으로 공유 원본의 정규화·데이터 삭제를 승인받은 것은 아니다. 아래의 대상 확인 및 목적별 인간 승인이 완료되면 실제 적용까지 수행한다. 계획서나 추가 PR만 만들고 종료하지 않는다.

## 0. 이번 작업의 시작점과 종료점

이미 완료한 코드 검수·CI·합성 시험·실제 백업 복원본 리허설을 다시 수행하지 않는다. #118과 추가 인계 자료의 결과를 사용한다. 복원본 CHECK 5개 보정은 복원본 전용 처리이며, 공유 원본에서 재실행하지 않는다. 정의 가드를 완화하거나 기대 해시를 바꾸지도 않는다.

이번 작업은 다음 흐름으로 끝낸다.

```text
실행 대상·백업·쓰기 중단 확인
→ 원본 check / manifest
→ 원본용 NORMALIZE plan 생성·인간 승인·normalize
→ 별도 원본용 PURGE plan 생성·인간 승인·purge
→ migrate (내부 validate + 두 번째 migrate=0 포함)
→ 원본 사후 검사·보존 데이터 비교
→ 적용 결과 보고 및 인간 DB GO 확인
```

정상화·삭제는 현재 상태에 필요한 경우에만 실행한다. 이미 적용된 DB에 동일 삭제를 반복하지 않는다. 현재 데이터가 폐기 대상이 아니면 삭제를 중단한다.

**종료 상태:** 실제 공유 원본 적용과 사후 검사가 끝나면 `SHARED_DB_APPLIED / POSTCHECK_PASS`로 보고한다. DB 담당자가 결과를 승인한 경우에만 `DB_GO=GO`와 승인자·시각을 기록한다. 서비스 기동·공개 배포 승인은 이번 작업 밖이다.

## 1. 기준 소스와 기존 증거

### 1.1 고정 실행 소스

실제 복원본 리허설에 사용한 위 SHA의 깨끗한 worktree 또는 그 SHA를 확인할 수 있는 기존 패키지를 사용한다. 최신 main을 자동으로 당겨 실행 기준을 바꾸지 않는다. 기존 작업 폴더의 변경·미추적 파일을 덮어쓰지 않는다.

반드시 읽을 파일:

```text
docs/DB_CONVERGENCE_RUNBOOK.md
scripts/db/java/org/example/db/ConvergenceTool.java
scripts/db/verify-normalization-catalog.sql
scripts/db/normalize-ownership-version.sql
scripts/db/normalize-purchase-summary.sql
scripts/db/check-convergence.sql
src/main/resources/db/convergence/V2026110402__enforce_product_document_transfer_integrity.sql
```

실행 root는 Gradle wrapper와 위 파일들이 있는 소스 루트다. Git worktree라면 `git rev-parse HEAD`와 `git status --porcelain`로 SHA 및 무변경 상태를 확인한다. 아카이브라면 `SOURCE.txt`와 해당 manifest를 확인한다. 소스는 작업 종료 때까지 변경하지 않는다.

### 1.2 재사용할 리허설 자료

- PR #118 보고서 및 사용자가 전달한 추가 인계 자료.
- 기존 담당자 경로: `C:\restricted-backup\evidence\`.
- 기존 파일: `history-before.txt`, `history-after.txt`, `manifest.txt`, `manifest2.txt`, `check.txt`, `check2.txt`, `migrate.txt`, `plan-normalize.txt`, `plan-purge.txt`, `check-fix.sql`, `check-fix-before-after.txt`, `shared-14-match.txt`.
- 실제 복원본 결과는 CHECK 보정 후 정규화·정리·신규 migration 10건·validate 32건·재실행 0건으로 보고됐다. 공유 원본은 미변경으로 보고됐다.

필요한 기존 자료만 읽고, 보고서 문장 정리나 CHECK 예외의 일반화를 위해 전체 리허설을 재시작하지 않는다. 기존 clone의 plan·승인 해시·대상 정보는 원본 실행에 재사용하지 않는다.

## 2. 담당과 허용 범위

| 구분 | 담당 |
|---|---|
| 대상 Supabase 프로젝트·원본 host·DB 확인, 단일 실행자 지정 | 인간 DB 담당자 |
| 쓰기 중단 및 현재 업무 데이터의 폐기 범위 승인 | 인간 DB 담당자와 팀 |
| 승인된 접속 제공, 백업·복구 방법 확인 | 인간 DB 담당자 |
| 기존 runner의 조회·plan·승인 후 실제 명령 실행, 로그·전후 비교 | IDE 에이전트 |
| NORMALIZE/PURGE plan의 목적·대상·내용·해시 승인 | 인간 DB 담당자 |
| 사후 결과 확인과 DB GO | 인간 DB 담당자 |
| AWS·S3·EC2·Nginx·systemd·서비스 공개 | 우리 쪽 배포 담당, 이번 에이전트 범위 밖 |

허용 작업은 원본 조회, 승인된 백업, 목적별 승인 후 기존 runner의 `normalize`, `purge`, `migrate`, 사후 조회뿐이다.

다음은 수행하지 않는다.

- 새 수습 알고리즘·migration·기능 코드 구현, 무관한 리팩터링, 기존 CLOSED 지적의 재검수.
- 공유 원본의 CHECK 5개 DROP/ADD, 정규화 가드 우회, checksum·installed_rank 수동 수정, `repair`/`clean`.
- 직접 작성한 DELETE·TRUNCATE·임의 owner 재지정, 사용자·관리 스키마 삭제.
- S3 파일·Redis 키 삭제, 앱 실행·시딩, 운영 env/feature flag 변경, PR 생성·push·병합.

실제 기능 실행을 막는 오류, 승인 범위 밖 삭제 위험, 대상 불일치, 기존 도구의 실행 실패만 작업 중단 사유로 다룬다. 새 기능 제안·문구 개선·가정적인 엣지 케이스로 범위를 늘리지 않는다.

## 3. 최초 실행 전 확인 — 인간 승인 A0

아래 조건을 확인한 뒤 현재 상태 조회와 목적별 plan 생성을 진행한다. 이미 리허설 담당자가 확인한 항목은 기록을 재사용한다.

1. **정확한 원본 대상:** Supabase 프로젝트 ref, host, `postgres/public`, PostgreSQL 17 계열을 확인한다. `postgres/public`이라는 이름만으로 프로젝트를 판단하지 않는다. 다른 담당자가 적용 중인지 확인하고 실행자를 한 명으로 고정한다.
2. **쓰기 중단:** 해당 DB에 쓰는 팀 IDE 앱·작업자·배포 서비스를 담당자들이 중단한다. 이 에이전트가 AWS나 다른 팀 PC를 독자 조작하지 않는다. 적용 종료 후에도 자동 재기동하지 않는다.
3. **현재 복구 지점:** 기존 백업과 복원 성공 기록을 보존한다. 21:50에 만든 파일이라는 이유만으로 지금의 적용 전 백업으로 간주하지 않는다. 이후 쓰기가 있었거나 동일성을 확인할 수 없으면, 쓰기 중단 후 기존에 성공한 방식으로 현재 public을 한 번 더 백업하고 시각·크기·SHA-256·목차 확인을 남긴다. 이번 작업을 위해 동일 리허설 전체를 다시 수행할 필요는 없다.
4. **백업 범위:** 기존 dump는 public 논리 덤프이며 소유권·권한 복원의 전체 증거가 아니다. 담당자는 기존 복구 방법과 소유권·권한 보존 경로를 확인한다. 원본 public의 DROP/재생성이나 자동 복원은 승인 범위에 포함하지 않는다.
5. **실제 폐기 범위:** 과거의 제품 2개 등은 참고값이다. 현재 plan에 표시된 업무 데이터 전체를 팀이 버려도 되는지 승인해야 한다. 리허설에서 지웠다는 이유로 원본 삭제를 자동 승인하지 않는다.
6. **파일 참조 보관:** 삭제할 DB 행이 참조하는 파일의 필요한 식별자는 저장소 밖 제한된 위치에 남기거나 이미 확보한 유효한 목록을 확인해 배포 담당자에게 인계한다. 이번에는 파일·Redis 상태를 삭제하지 않는다.

A0는 대상·유지보수 시간·백업·migration 실행 범위의 확인이다. 이후 실제 정상화와 삭제에는 각 plan 해시를 포함한 별도 승인이 필요하다. 이미 명시적으로 주어진 승인을 반복해서 요청하지 않는다.

## 4. 실행 환경과 로그

### 4.1 입력

JDK 17과 체크인된 Gradle wrapper를 사용한다. 기존 `.env`를 읽거나 앱을 기동하지 않는다. 별도 유지보수 프로세스에 필요한 값만 비공개로 주입한다.

| 환경변수 | 지정 내용 |
|---|---|
| `CONVERGENCE_DB_URL` | 원본 JDBC PostgreSQL URL. 비밀번호·토큰을 URL에 포함하지 않음 |
| `CONVERGENCE_DB_USER` / `CONVERGENCE_DB_PASSWORD` | 승인된 원본 유지보수 계정·암호, 비공개 주입 |
| `CONVERGENCE_DATABASE` | 담당자가 확인한 DB명. 관측값은 `postgres` |
| `CONVERGENCE_SCHEMA` | `public` |
| `CONVERGENCE_TARGET_ID` | 인간이 확인한 원본 Supabase 프로젝트 식별자 |
| `CONVERGENCE_URL_SHA256` | 위 JDBC URL의 정확한 UTF-8 문자열 SHA-256. 대상 확인 뒤 계산 |
| `CONVERGENCE_PLAN` | 단계별로 서로 다른 제한된 로컬 파일 경로 |
| `CONVERGENCE_APPROVED_PLAN_SHA256` | 사람이 현재 목적의 plan을 승인한 후에만 주입하는 해시 |

`pg_dump`용 `pgpass`와 Java runner의 환경변수는 별도 입력이다. 기존 세션 풀러/직접 연결 중 리허설에서 확인한 유지보수 연결 방식을 재사용한다. 암호·전체 URL을 출력하거나 PR·공개 보고서·채팅에 기록하지 않는다.

원본 조회·백업은 읽기 전용으로 수행한다. 쓰기 단계는 담당자가 승인한 쓰기 가능한 유지보수 연결에서 실행한다. 기존 read-only 제한 때문에 실패하면 해당 실행 환경만 담당자와 확인한다. 서버·역할의 전역 보호 설정을 임의로 변경하지 않는다.

### 4.2 증거 보관

기존 증거를 덮어쓰지 않는 새 제한 디렉터리를 만든다. 예:

```text
C:\restricted-backup\evidence\shared-apply-<실행시각>\
```

코드 블록은 **단계별로 실행**한다. 전 단계를 이어 붙인 무승인 자동 실행 스크립트로 만들지 않는다. 명령 stdout/stderr·실제 종료 코드·실행 시각을 보관한다. 로그 수집은 리허설에서 성공한 방식을 사용하고 PowerShell 출력 처리 오류를 도구 성공으로 무시하지 않는다.

아래 명령은 Windows PowerShell 기준이다. 리허설과 같은 Unix 환경에서는 `./gradlew`로 바꾸고 동일하게 각 종료 코드를 확인한다.

## 5. 단계별 실제 적용

### S1. 원본의 현재 상태 확인

```powershell
.\gradlew.bat dbConvergence -PdbLineage=legacy -PdbAction=check --console=plain
if ($LASTEXITCODE -ne 0) { throw 'S1 check 실패: 쓰기 단계로 진행하지 말 것' }

.\gradlew.bat dbConvergence -PdbLineage=legacy -PdbAction=manifest --console=plain
if ($LASTEXITCODE -ne 0) { throw 'S1 manifest 실패: 쓰기 단계로 진행하지 말 것' }
```

`check`의 exit=0은 조회 완료라는 의미이지 위반 0이라는 뜻이 아니다. 출력 내용도 읽는다.

현재 알려진 출발 상태는 이력 22행, 마지막 `2026103001`, 네 충돌 번호, 제품/문서/채팅 고아 참조다. 후속 컬럼·테이블·제약의 미존재와 이 단계의 예상된 고아 참조는 **수습 대상**이지 자동 중단 사유가 아니다. 적용 전 상태를 기록한다.

- 알려진 legacy 상태이면 S2로 진행한다.
- 다른 담당자가 이미 수습을 완료했다면 정상화·삭제를 다시 실행하지 말고 S5의 사후 결과를 수집한다.
- 알려지지 않은 이력·부분 적용·다른 대상·보존해야 할 데이터가 확인되면 그 상태만 보고하고 쓰기를 멈춘다. 임의 history 수정으로 맞추지 않는다.

원본의 14개 제약 확인은 기존 읽기 전용 조회를 재사용할 수 있고, `normalize`도 기존 catalog guard를 그대로 실행한다. 원본에서 가드가 실패하면 `check-fix.sql`을 실행하지 말고 실제 오류에서 중단한다.

### S2. NORMALIZE plan 생성 → 인간 승인 A1 → 실행

`CONVERGENCE_PLAN`을 이번 원본 실행 디렉터리의 새 `plan-normalize.txt`로 지정하고, 이전 승인 변수를 비운다.

```powershell
Remove-Item Env:CONVERGENCE_APPROVED_PLAN_SHA256 -ErrorAction SilentlyContinue
.\gradlew.bat dbConvergence -PdbLineage=legacy -PdbAction=plan -PdbPlanOperation=NORMALIZE --console=plain
if ($LASTEXITCODE -ne 0) { throw 'S2 NORMALIZE plan 생성 실패' }
```

사람에게 대상·소스 SHA·`operation=NORMALIZE`·변경 범위·plan SHA-256을 제시한다. 정확한 현재 plan에 대한 승인을 받은 뒤 승인 해시를 `CONVERGENCE_APPROVED_PLAN_SHA256`에 입력한다. plan에서 해시를 얻는 즉시 에이전트가 스스로 승인 변수까지 채우지 않는다.

승인 문구 예:

```text
원본 NORMALIZE 승인. 대상=<확인한 원본>, 소스=2c218e575f662ff5fb0c93318cef8783e5df2c8e,
plan_sha256=<현재 해시>. 네 이력의 version/script 변경만 승인한다.
```

```powershell
.\gradlew.bat dbConvergence -PdbLineage=legacy -PdbAction=normalize --console=plain
if ($LASTEXITCODE -ne 0) { throw 'S2 normalize 실패: 후속 쓰기 중단' }
```

직후 history를 읽어 아래 결과와 다른 필드 보존을 확인한다. 기존 `check` 출력을 사용해도 된다.

| 변경 전 | 변경 후 | 유지할 checksum |
|---|---|---:|
| `2026102701` 구매 요청 | `2026102103` 구매 요청 | -778179566 |
| `2026102706` summary | `2026102701` summary | 620449880 |
| `2026102702` ownership | `2026102803` ownership | -1318655273 |
| `2026102707` pending slot | `2026102804` pending slot | -1107274211 |

이 단계에서 업무 행·installed_rank·description·type·checksum·success·기존 실행 metadata를 임의로 변경하지 않는다. 결번 rank 17도 채우지 않는다.

### S3. PURGE plan 새로 생성 → 인간 승인 A2 → 실행

이 단계는 **현재 plan의 업무 데이터 전체가 폐기 가능하다고 승인된 경우에만** 수행한다. 일부를 보존해야 하면 임의 부분 삭제 코드를 만들지 말고 현재 실행을 보류한다. 이미 업무 데이터가 모두 비어 있고 삭제가 불필요하다면 `PURGE=NOT_REQUIRED`로 기록한다.

`CONVERGENCE_PLAN`을 S2와 다른 새 `plan-purge.txt`로 변경한다.

```powershell
Remove-Item Env:CONVERGENCE_APPROVED_PLAN_SHA256 -ErrorAction SilentlyContinue
.\gradlew.bat dbConvergence -PdbLineage=legacy -PdbAction=plan -PdbPlanOperation=PURGE --console=plain
if ($LASTEXITCODE -ne 0) { throw 'S3 PURGE plan 생성 실패' }
```

고정 삭제 allowlist는 다음과 같다. **고아 행만 삭제하는 것이 아니라, 존재하는 대상 업무 테이블의 모든 행을 삭제한다.** 테이블을 추가하거나 삭제 범위를 변경하지 않는다.

```text
transaction_change_reviews, event_documents, document_extractions,
document_replacements, audit_logs, product_events, ownership_transfers,
purchase_requests, chat_read_positions, chat_messages, chat_rooms,
listing_images, listings, draft_images, listing_drafts,
product_field_authors, product_images, laptop_details, documents, products
```

보존 대상은 `users`, 기존 `user_profile_images`, `flyway_schema_history`, sequence/identity다. Supabase 관리 스키마·Auth·Storage, 실제 파일과 Redis는 범위 밖이다.

현재 테이블별 삭제 건수·보존 범위·`operation=PURGE`·plan SHA-256을 사람이 확인한 뒤 승인한다.

```text
원본 PURGE 승인. 대상=<확인한 원본>, plan_sha256=<현재 PURGE 해시>.
현재 plan에 표시된 allowlist 업무 행 전체의 폐기를 승인한다.
users·프로필·Flyway 이력·sequence와 관리 스키마는 보존한다.
```

그 승인에 해당하는 해시만 `CONVERGENCE_APPROVED_PLAN_SHA256`에 넣고 실행한다.

```powershell
.\gradlew.bat dbConvergence -PdbLineage=legacy -PdbAction=purge --console=plain
if ($LASTEXITCODE -ne 0) { throw 'S3 purge 실패: migration으로 진행하지 말 것' }
```

삭제 대상 잔존 행과 보존 대상 digest를 **purge 직전 plan 대비** 확인한다. users의 과거 리허설 digest나 고정 ID를 현재 정상값으로 강제하지 않는다. 정상화 전후에는 history의 네 변경을, purge 전후에는 history 전체의 불변을 각각 확인한다.

plan은 30분 만료 및 내용·대상·소스 변경 검사를 유지한다. 만료·drift 시 현재 상태로 다시 생성하고 재승인받는다. 목적별 승인을 서로 재사용하지 않는다.

### S4. 후속 migration 적용

S2/S3가 필요한 경우 완료됐고, A0에서 고정 후보의 전체 migration 적용 범위까지 승인받은 상태에서 실행한다. 이 명령은 조회가 아니라 실제 쓰기다.

```powershell
.\gradlew.bat dbConvergence -PdbLineage=legacy -PdbAction=migrate --console=plain
if ($LASTEXITCODE -ne 0) { throw 'S4 migrate/validate 실패: 쓰기 중단 유지 및 현재 상태 수집' }
```

기존 도구가 `migrate → validate → 두 번째 migrate의 신규 적용 0건`을 이미 수행한다. 이를 확인하려고 같은 전체 명령을 무조건 다시 실행하지 않는다.

관측했던 원본 상태가 유지됐다면 적용될 10개 버전:

```text
2026103002
2026110201, 2026110202
2026110301, 2026110302, 2026110303, 2026110304, 2026110305
2026110401, 2026110402
```

최종 기대 버전은 `2026110402`, 이력은 기존 22행에 신규 10행이 더해진 32행이다. 이 수치는 현재 상태 대조용이다. 숫자를 맞추려고 history를 추가·삭제하지 않는다. 기대와 다른 적용 목록이면 실제 원인을 기록하고 멈춘다.

### S5. 원본 사후 검사와 인계

```powershell
.\gradlew.bat dbConvergence -PdbLineage=legacy -PdbAction=check --console=plain
if ($LASTEXITCODE -ne 0) { throw 'S5 사후 check 실패' }

.\gradlew.bat dbConvergence -PdbLineage=legacy -PdbAction=manifest --console=plain
if ($LASTEXITCODE -ne 0) { throw 'S5 사후 manifest 실패' }
```

다음이 충족되면 이번 적용 작업을 마감한다.

| 항목 | 완료 기준 |
|---|---|
| 이력 | 네 identity 수렴, 기존 checksum 보존, 실패·중복·미해결·미적용 항목 없음 |
| migration | 고정 후보 전체 적용, strict validate 성공, 내부 재실행 신규 적용 0건 |
| 구조 | 문서 교체·거래 변경 검토·프로필 이미지 테이블, 공개/검토·수신자 동의 컬럼 존재 |
| 신규 FK/CHECK | `fk_products_owner`, `fk_documents_product`, `fk_documents_uploader`, `ck_products_convergence_required`, `ck_transfers_type` 존재·유효 |
| 기존 CHECK | `ck_products_owner_id_required`, `ck_products_serial_ciphertext_required` 검증 완료 또는 후보가 허용하는 동등 NOT NULL 계약 |
| 인덱스 | `ix_products_owner_archive_created` 정의 일치, valid/ready 정상 |
| 데이터 | 주요 고아·필수값·CHECK 위반 0, 필요한 검사에 오류·NOT_RUN 없음 |
| 보존 | users 및 기존 프로필 행 보존, 승인 범위 외 데이터 변경 없음. history는 정상화·신규 적용만 반영 |

알려진 `OUT_OF_ORDER` 4건(`2026102601`, `2026102701`, `2026102801`, `2026102802`)은 기존 rank를 보존한 정상화 결과로 기록한다. 그 표시만으로 실패 처리하거나 `outOfOrder=true`로 바꾸지 않는다. 알 수 없는 다른 상태는 별도로 보고한다.

`user_profile_images`가 적용 전에 없었다면 `미존재 → 생성/0행`으로 기록한다. 이를 `0행 보존`이라고 쓰지 않는다. 빈 업무 테이블에서 위반 0건인 결과를 실제 기능 인수 완료로 확대하지 않는다.

## 6. 오류·승인 대기 시 처리

- **승인·접속이 없음:** 완료한 준비와 다음에 필요한 입력 한 가지를 구체적으로 보고하고 `AWAITING_HUMAN_APPROVAL` 또는 `AWAITING_ACCESS`로 종료한다. 필요한 승인을 확보한 뒤에는 계획서만 다시 쓰지 말고 해당 실행 단계부터 재개한다.
- **명령/가드/timeout 실패:** 다음 쓰기를 중단하고 실패 단계·종료 코드·비밀값을 제거한 오류와 현재 history/catalog를 보존한다. 같은 쓰기 명령을 반복 실행해 통과를 시도하지 않는다.
- **normalize/purge 실패:** 해당 트랜잭션의 롤백 여부를 확인한다. 그 전에 완료·커밋된 단계까지 되돌아갔다고 가정하지 않는다.
- **migrate 실패:** 이전 migration은 이미 커밋됐을 수 있다. 실제 적용 이력·인덱스 유효성을 수집하고 기존 복구 절차로 담당자 판단을 받는다. 정규화·purge·Flyway 전체를 하나의 외부 BEGIN으로 묶거나 자동 repair/복원하지 않는다.

유지보수 JDBC의 `SET LOCAL` timeout을 별도 Flyway 연결 전체의 timeout이라고 주장하지 않는다. 리허설에서 정한 migration 연결·취소 방침을 재사용한다. 알려진 실제 실행 장애가 없다면 이 항목을 새 설계 작업으로 확대하지 않는다.

오류가 코드 결함으로 확정된 경우에만 최소 수정 범위와 재개 지점을 보고한다. 이번 원본 적용 도중 코드를 즉흥 변경해 계속하지 않는다.

## 7. 필수 제출물과 완료 보고

새 PR은 기본 제출물이 아니다. 원본 적용 결과 보고서와 필요한 비민감 증거를 제출한다. 기존 로컬 비공개 로그는 접근 제한 상태로 보관하고, dump·암호·접속 문자열·실제 업무 원문은 PR에 올리지 않는다.

최소 증거:

```text
실행 source SHA와 대상 확인 기록
적용 직전 backup 식별자·시각·크기·해시·복구 담당자
before-check / before-manifest
NORMALIZE plan·인간 승인 기록·normalize 로그
PURGE plan·인간 승인 기록·purge 로그 (실행한 경우)
migrate 로그 (validate 및 내부 재실행 포함)
after-check / after-manifest
보존 데이터 비교·최종 결과 보고
```

권장 보고 형식:

```text
STATUS=SHARED_DB_APPLIED / POSTCHECK_PASS
SOURCE_SHA=2c218e575f662ff5fb0c93318cef8783e5df2c8e
EVIDENCE_SOURCE=ACTUAL_SHARED_DB
TARGET_CONFIRMED=YES
EXECUTOR=<이름 또는 역할>
STARTED_AT=<KST>
COMPLETED_AT=<KST>
BACKUP=<비민감 식별자·시각·SHA-256>
WRITERS_STOPPED=YES

NORMALIZE=PASS / NOT_REQUIRED / NOT_RUN / FAIL
NORMALIZE_APPROVAL=<승인자·시각·원본 plan 해시>
PURGE=PASS / NOT_REQUIRED / NOT_APPROVED_NOT_RUN / FAIL
PURGE_APPROVAL=<승인자·시각·원본 plan 해시>
MIGRATE_VALIDATE=PASS / NOT_RUN / FAIL
NEW_MIGRATIONS=<실제 목록>
SECOND_MIGRATE_ZERO=YES / NOT_RUN
HISTORY_COUNT_BEFORE=<실제 값>
HISTORY_COUNT_AFTER=<실제 값>
KNOWN_OUT_OF_ORDER=<실제 알려진 목록>
CONSTRAINTS_AND_INDEXES=PASS / FAIL
INTEGRITY_COUNTS=<사후 검사별 수치>
PRESERVED_DATA=PASS / FAIL
SHARED_DB_CHANGED=YES / NO

AGENT_RECOMMENDATION=DB_GO / HOLD
HUMAN_DB_GO=GO / HOLD / PENDING
APPROVER_AND_TIME=<실제 승인자·시각>
AWS_AND_REDIS_AND_STORAGE_CHANGES=NOT_PERFORMED
APP_START_AND_PUBLIC_DEPLOY=NOT_PERFORMED
```

값은 실제 실행 결과로 채운다. `SHARED_DB_CHANGED=NO`인 상태를 원본 적용 완료라고 보고하지 않는다. 인간 승인 전에는 에이전트가 DB GO를 대신 선언하지 않는다.

DB 담당자가 승인한 후 배포 담당자에게 **원본 적용 결과, 고정 DB 도구 SHA, 적용 버전 목록, 남은 실제 기능 인수 항목**을 넘긴다. 정상화된 legacy DB의 런타임 프로필은 `prod,supabase,db-legacy-convergence`로 안내하되, 이 에이전트가 EC2 env를 직접 바꾸거나 서비스를 시작하지 않는다. 팀의 DB writer 재개 시점도 담당자가 결정한다.

## 8. 근거와 범위

이 작업서는 다음을 기초로 한 실행 지시다.

- 실제 복원본 리허설에 사용한 SHA `2c218e575f662ff5fb0c93318cef8783e5df2c8e`의 runbook, ConvergenceTool, `build.gradle`의 `dbConvergence` task.
- PR #118의 복원본 실행 결과와 사용자가 전달한 추가 인계 자료. 로컬 증거 파일의 실제 내용은 실행 담당자가 보유한 원문을 사용한다.
- 이전 검수에서 종료한 승인 목적 분리·bootstrap 안전 보정은 재개하지 않는다.

주요 고정 원문:

```text
https://github.com/prgrms-aibe-devcourse/AIBE7_FinalProject_Team1/blob/2c218e575f662ff5fb0c93318cef8783e5df2c8e/docs/DB_CONVERGENCE_RUNBOOK.md
https://github.com/prgrms-aibe-devcourse/AIBE7_FinalProject_Team1/blob/2c218e575f662ff5fb0c93318cef8783e5df2c8e/scripts/db/java/org/example/db/ConvergenceTool.java
https://github.com/prgrms-aibe-devcourse/AIBE7_FinalProject_Team1/blob/2c218e575f662ff5fb0c93318cef8783e5df2c8e/build.gradle
```

문서 작성 자체로 실제 공유 DB에 접속·변경한 것은 아니다. 이번 작업의 성공은 **새 PR 생성이 아니라 실제 공유 원본의 정상화와 사후 결과 제출**이다.
