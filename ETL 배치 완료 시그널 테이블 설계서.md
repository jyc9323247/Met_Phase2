
# ETL 배치 완료 시그널 테이블 설계서

[[_TOC_]]

## 1. 개요

IBM TWS에서 수행되는 온프레미스 ETL 작업이 정상 완료되면 Azure PostgreSQL의 시그널 테이블에 수집 대상 정보를 등록한다.

Azure Data Factory(ADF)는 시그널 테이블을 주기적으로 조회하여 신규 시그널을 확인하고, 기존 ADF 수집 메타데이터와 연계하여 데이터 수집 Pipeline을 실행한다.

### 1.1 주요 설계 원칙

- 시그널 테이블은 Azure PostgreSQL에 생성한다.
- 수집 대상은 Source DB, Schema, Table로 식별한다.
- 시그널은 온프레미스 ETL 정상 완료 후 등록한다.
- 시그널 등록은 INSERT Only 방식으로 처리한다.
- 증분 수집 범위는 JSON으로 전달할 수 있다.
- JSON이 NULL이면 기존 ADF 메타데이터의 증분 정책을 사용한다.
- JSON에 증분 기준 컬럼이 없으면 Target Master의 증분 컬럼을 사용한다.
- 시그널 등록 스크립트는 Linux 서버에서 실행한다.
- Azure PostgreSQL 연결은 ODBC를 사용한다.
- 시그널 INSERT 실패 시 30초 간격으로 최대 2회 재시도한다.
- ADF는 시그널 테이블에 SELECT Only로 접근한다.
- ADF의 처리 상태 및 중복 실행 여부는 ADF Meta DB에서 관리한다.

## 2. 전체 아키텍처

### 2.1 처리 구조

```text
┌─────────────────────────────────────┐
│ IBM TWS / 온프레미스 ETL            │
│                                     │
│ ETL 수행 및 정상 완료 확인          │
│                                     │
│ Signal INSERT 스크립트 실행         │
└──────────────────┬──────────────────┘
                   │
                   │ ODBC
                   ▼
┌─────────────────────────────────────┐
│ Azure PostgreSQL                    │
│                                     │
│ adfadm.tb_etl_batch_signal          │
│                                     │
│ - Source DB                          │
│ - Source Schema                      │
│ - Source Table                       │
│ - Batch Date                         │
│ - Incremental Range JSON             │
└──────────────────┬──────────────────┘
                   │
                   │ ADF Polling
                   ▼
┌─────────────────────────────────────┐
│ Azure Data Factory                  │
│                                     │
│ 1. 신규 Signal 조회                 │
│ 2. Target Master 매핑               │
│ 3. Run Group / INCR Policy 조회     │
│ 4. JSON 수집 범위 적용              │
│ 5. 수집 Pipeline 실행               │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ ADF Meta DB                         │
│                                     │
│ - Signal 처리 이력                   │
│ - Pipeline 실행 상태                │
│ - 오류 및 재처리 관리               │
└─────────────────────────────────────┘
```

## 3. 시그널 등록 방식

시그널 등록은 다음 두 가지 방식 중 업무 환경에 적합한 방식을 선택하여 적용한다.

두 방식 모두 동일한 Linux ODBC INSERT 스크립트를 사용한다.

### 3.1 방식 A. ETL 수행 스크립트 내부 호출

IBM TWS가 실행하는 ETL 수행 스크립트 내부에서 ETL 정상 완료 여부를 확인한 후 Signal INSERT 스크립트를 호출한다.

```text
IBM TWS
   |
   v
ETL 수행 스크립트
   |
   +-- 1. ETL 수행
   |
   +-- 2. ETL 정상 완료 확인
   |
   +-- 3. Signal INSERT 스크립트 호출
   |       |
   |       +-- Source DB
   |       +-- Source Schema
   |       +-- Source Table
   |       +-- 증분 수집 범위
   |       |
   |       v
   |    Azure PostgreSQL
   |
   +-- 4. 최종 종료
```

#### 처리 기준

1. TWS가 기존 ETL 수행 스크립트를 실행한다.
2. ETL 수행 스크립트가 데이터 처리를 완료한다.
3. ETL 정상 완료 여부를 확인한다.
4. 정상 완료 시 Signal INSERT 스크립트를 호출한다.
5. Signal INSERT가 성공하면 정상 종료한다.
6. Signal INSERT가 최종 실패하면 오류를 반환한다.

#### Shell 호출 예시

```bash
#!/bin/bash

# 1. ETL 수행
/etl/bin/run_etl.sh

ETL_RC=$?

if [ $ETL_RC -ne 0 ]; then
    echo "ETL FAILED"
    exit $ETL_RC
fi

# 2. Signal INSERT
/etl/bin/insert_etl_signal.sh \
    --db DB1 \
    --schema MLDADM \
    --table TB_SAMPLE \
    --from-months-ago 3 \
    --to-months-ago 1 \
    --column CLYM

SIGNAL_RC=$?

if [ $SIGNAL_RC -ne 0 ]; then
    echo "SIGNAL INSERT FAILED"
    exit 1
fi

exit 0
```

### 3.2 방식 B. TWS 후속 작업에서 호출

IBM TWS에서 ETL 수행 작업의 정상 완료를 조건으로 Signal INSERT 작업을 후속 실행한다.

```text
IBM TWS
   |
   +-- ETL 수행 작업
   |      |
   |      +-- ETL 정상 완료
   |                |
   |                v
   +-- Signal INSERT 후속 작업
                  |
                  +-- INSERT 스크립트 실행
                  |
                  v
             Azure PostgreSQL
```

#### 처리 기준

1. TWS가 ETL 작업을 실행한다.
2. ETL 작업이 정상 완료되면 후속 Signal INSERT 작업을 실행한다.
3. 후속 작업에서 Source DB, Schema, Table 및 증분 범위를 전달한다.
4. Signal INSERT 스크립트가 Azure PostgreSQL에 데이터를 등록한다.
5. INSERT 실패 시 스크립트 내부에서 재시도한다.
6. 최종 실패 시 TWS에 오류 상태를 반환한다.

#### TWS 후속 작업 실행 예시

```bash
/etl/bin/insert_etl_signal.sh \
    --db DB1 \
    --schema MLDADM \
    --table TB_SAMPLE \
    --from-months-ago 1 \
    --column CLYM
```

### 3.3 적용 기준

| 구분 | 방식 A | 방식 B |
|---|---|---|
| 실행 주체 | ETL 수행 스크립트 | IBM TWS 후속 작업 |
| ETL 수정 | 필요 | 불필요 |
| TWS 작업 추가 | 불필요 | 필요 |
| ETL 완료 확인 | ETL 스크립트 내부 | TWS 선행 작업 완료 조건 |
| Signal INSERT | ETL 스크립트에서 호출 | TWS 후속 작업에서 호출 |
| 장애 확인 | ETL 실행 로그 | TWS 작업 실행 로그 |

업무별로 한 가지 방식을 선택하여 적용하며, 동일 ETL 완료 건에 대해 두 방식이 중복 실행되지 않도록 한다.

## 4. 시그널 테이블 논리 모델

테이블명:

`adfadm.tb_etl_batch_signal`

| 컬럼명 | 데이터 타입 | 필수 | 설명 |
|---|---|---|---|
| signal_id | BIGINT | Y | 시그널 고유 ID |
| src_db_nm | VARCHAR(30) | Y | 수집 대상 DB명 |
| src_schema_nm | VARCHAR(128) | Y | 수집 대상 스키마명 |
| src_table_nm | VARCHAR(128) | Y | 수집 대상 테이블명 |
| batch_date | DATE | Y | ETL 배치 기준일 |
| batch_end_dt | TIMESTAMP | Y | ETL 완료 시각 |
| incr_range_json | JSONB | N | 증분 수집 조건 및 범위 |
| created_dt | TIMESTAMP | Y | 시그널 등록 시각 |

시간 컬럼은 KST 기준으로 관리한다.

## 5. Azure PostgreSQL DDL

```sql
CREATE TABLE adfadm.tb_etl_batch_signal
(
    signal_id BIGINT
        GENERATED ALWAYS AS IDENTITY
        PRIMARY KEY,

    src_db_nm VARCHAR(30) NOT NULL,
    src_schema_nm VARCHAR(128) NOT NULL,
    src_table_nm VARCHAR(128) NOT NULL,

    batch_date DATE NOT NULL,
    batch_end_dt TIMESTAMP NOT NULL,

    incr_range_json JSONB NULL,

    created_dt TIMESTAMP NOT NULL
        DEFAULT LOCALTIMESTAMP,

    CONSTRAINT ck_etl_signal_json
        CHECK (
            incr_range_json IS NULL
            OR (
                jsonb_typeof(incr_range_json) = 'object'
                AND incr_range_json ? 'from'
                AND jsonb_typeof(incr_range_json -> 'from') = 'string'
                AND (
                    NOT (incr_range_json ? 'to')
                    OR jsonb_typeof(incr_range_json -> 'to') = 'string'
                )
                AND (
                    NOT (incr_range_json ? 'column')
                    OR jsonb_typeof(incr_range_json -> 'column') = 'string'
                )
            )
        )
);

CREATE INDEX ix_etl_signal_created
ON adfadm.tb_etl_batch_signal
(
    created_dt,
    signal_id
);
```

## 6. 증분 수집 범위 JSON 규격

### 6.1 기본 규격

```json
{
  "from": "202607",
  "to": "202609",
  "column": "CLYM"
}
```

| 속성 | 필수 | 설명 |
|---|---|---|
| from | Y | 수집 시작월, 해당 월 포함 |
| to | N | 수집 종료월, 해당 월 포함 |
| column | N | 증분 기준 컬럼명 |

`from`은 `>=` 조건으로 적용한다.

`to`가 존재하면 `<=` 조건으로 적용한다.

`to`를 생략하면 상한 조건을 적용하지 않는다.

`column`을 생략하면 `ctl_ingest_target_master.incr_column_nm`을 사용한다.

JSON 전체가 NULL이면 기존 ADF 메타데이터의 증분 정책을 적용한다.

### 6.2 시작월만 지정

```json
{
  "from": "202607"
}
```

메타 증분 기준 컬럼이 `CLYM`인 경우:

```sql
WHERE CLYM >= '202607'
```

### 6.3 시작월과 종료월 지정

```json
{
  "from": "202607",
  "to": "202609"
}
```

적용 조건:

```sql
WHERE CLYM >= '202607'
  AND CLYM <= '202609'
```

### 6.4 증분 컬럼 지정

```json
{
  "from": "202607",
  "to": "202609",
  "column": "CLYM"
}
```

JSON에 지정된 컬럼을 기준으로 범위 조건을 적용한다.

단, JSON 컬럼명은 실제 수집 대상 테이블에 존재하는 컬럼이어야 하며 ADF에서 사전 검증한다.

### 6.5 JSON이 NULL인 경우

```sql
incr_range_json = NULL
```

기존 ADF Target Master 및 INCR Policy 설정을 사용한다.

## 7. N개월 전 파라미터 처리

온프레미스 Signal INSERT 스크립트는 N개월 전 값을 숫자로 입력받아 실제 `YYYYMM` 값으로 변환한 후 PostgreSQL에 저장한다.

### 7.1 스크립트 파라미터

| 파라미터 | 필수 | 설명 |
|---|---|---|
| --db | Y | Source DB |
| --schema | Y | Source Schema |
| --table | Y | Source Table |
| --from-months-ago | N | 시작월, N개월 전 |
| --to-months-ago | N | 종료월, N개월 전 |
| --column | N | 증분 기준 컬럼 |
| --batch-date | N | 배치 기준일, 생략 시 실행일 |

증분 범위를 지정할 경우 `--from-months-ago`를 필수로 입력한다.

### 7.2 변환 예시

배치 기준일: `2026-10-27`

| 입력값 | 실제 저장값 |
|---|---|
| from-months-ago = 1 | 202609 |
| from-months-ago = 2 | 202608 |
| from-months-ago = 3 | 202607 |
| from-months-ago = 12 | 202510 |

### 7.3 실행 예시

```bash
./insert_etl_signal.sh \
    --db DB1 \
    --schema MLDADM \
    --table TB_SAMPLE \
    --from-months-ago 3 \
    --to-months-ago 1 \
    --column CLYM \
    --batch-date 2026-10-27
```

PostgreSQL에 저장되는 JSON:

```json
{
  "from": "202607",
  "to": "202609",
  "column": "CLYM"
}
```

### 7.4 기존 메타 정책 사용

```bash
./insert_etl_signal.sh \
    --db DB1 \
    --schema MLDADM \
    --table TB_SAMPLE
```

저장값:

```text
incr_range_json = NULL
```

## 8. ODBC INSERT 처리

### 8.1 연결 방식

Linux 서버에서 unixODBC Driver Manager와 PostgreSQL ODBC 드라이버를 사용하여 Azure PostgreSQL에 연결한다.

```text
Linux ETL Server
      |
      v
Signal INSERT Shell
      |
      v
unixODBC Driver Manager
      |
      v
PostgreSQL ODBC Driver
      |
      v
Azure PostgreSQL
```

ODBC DSN은 사전에 등록한다.

예시:

```bash
export ETL_SIGNAL_DSN="AZURE_PG"
```

### 8.2 INSERT SQL

```sql
INSERT INTO adfadm.tb_etl_batch_signal
(
    src_db_nm,
    src_schema_nm,
    src_table_nm,
    batch_date,
    batch_end_dt,
    incr_range_json
)
VALUES
(
    'DB1',
    'MLDADM',
    'TB_SAMPLE',
    DATE '2026-10-27',
    LOCALTIMESTAMP,
    '{"from":"202607","to":"202609","column":"CLYM"}'::jsonb
);
```

SQL 실행 시 입력 문자열은 ODBC 파라미터 바인딩 또는 검증된 SQL 리터럴 처리 방식으로 전달한다.

## 9. INSERT 재시도 정책

Signal INSERT 실패 시 30초 간격으로 최대 2회 재시도한다.

| 항목 | 설정 |
|---|---|
| 최초 실행 | 즉시 |
| 재시도 간격 | 30초 |
| 최대 재시도 | 2회 |
| 최대 실행 | 총 3회 |
| 성공 | 즉시 정상 종료 |
| 최종 실패 | 오류 코드 반환 |

### 9.1 처리 흐름

```text
Signal INSERT 최초 실행
         |
         +-- 성공 --> 종료(0)
         |
         +-- 실패
               |
               v
           30초 대기
               |
               v
         1차 재시도
               |
               +-- 성공 --> 종료(0)
               |
               +-- 실패
                     |
                     v
                 30초 대기
                     |
                     v
               2차 재시도
                     |
                     +-- 성공 --> 종료(0)
                     |
                     +-- 실패 --> 종료(1)
```

### 9.2 운영 원칙

- 재시도는 Signal INSERT 단계에 적용한다.
- INSERT 실패 시 온프레미스 ETL 전체 재실행을 기본으로 하지 않는다.
- 최종 실패 시 호출 스크립트 또는 TWS에 오류를 반환한다.
- INSERT 결과가 불명확한 통신 장애에서는 중복 등록 가능성을 고려한다.
- 동일 ETL 완료에 대한 중복 시그널 처리 여부는 ADF 운영 정책에 따라 관리한다.

## 10. 기존 ADF 메타데이터 연계

시그널 테이블에 저장된 Source DB, Schema, Table을 기준으로 ADF Target Master와 연결한다.

### 10.1 연계 테이블

| 메타 테이블 | 역할 |
|---|---|
| ctl_ingest_target_master | 수집 대상 및 증분 컬럼 정보 |
| ctl_ingest_incr_policy | 증분 수집 정책 |
| ctl_ingest_run_group | 실행 그룹 정보 |
| ctl_ingest_target_run_rule | 대상과 실행 그룹 연결 |
| ctl_ingest_target_incr_state | 증분 HWM 상태 |
| ctl_ingest_pipeline_run | ADF 수집 실행 이력 |

### 10.2 시그널과 Target Master 매핑

```sql
SELECT
    s.signal_id,
    s.batch_date,
    s.batch_end_dt,
    s.incr_range_json,

    m.target_id,
    m.src_type,
    m.src_db_nm,
    m.src_schema_nm,
    m.src_table_nm,

    m.incr_column_nm,
    m.incr_column_type,
    m.incr_column_format,
    m.incr_load_method_cd,

    p.incr_policy_id,
    p.policy_type_cd,
    p.range_unit_cd,
    p.range_interval

FROM adfadm.tb_etl_batch_signal s

JOIN adfadm.ctl_ingest_target_master m
  ON m.src_db_nm = s.src_db_nm
 AND m.src_schema_nm = s.src_schema_nm
 AND m.src_table_nm = s.src_table_nm
 AND m.data_class = 'RAW'
 AND m.delete_yn = 'N'

LEFT JOIN adfadm.ctl_ingest_incr_policy p
  ON p.incr_policy_id = m.incr_policy_id

WHERE s.created_dt >= :polling_start
  AND s.created_dt < :polling_end

ORDER BY s.signal_id;
```

`:polling_start`, `:polling_end`는 조회 구간을 나타내는 예시 파라미터이며 실제 ADF 쿼리 구성 시 실행 환경에 맞게 바인딩한다.

동일 DB·스키마·테이블 조합이 여러 `src_type`으로 등록될 수 있는 환경에서는 대상 식별 규칙을 추가로 정의해야 한다.

## 11. ADF 증분 범위 적용 규칙

### 11.1 처리 기준

| JSON 상태 | 처리 |
|---|---|
| NULL | 기존 ADF 메타 정책 적용 |
| from 존재 | 시작월 이상 |
| from, to 존재 | 시작월 이상, 종료월 이하 |
| column 존재 | JSON 지정 컬럼 적용 |
| column 없음 | Target Master 증분 컬럼 적용 |

### 11.2 JSON 범위 우선 적용

```text
Signal 조회
    |
    v
Target Master 매핑
    |
    v
incr_range_json 존재?
    |
    +-- NO
    |     |
    |     v
    |  기존 메타 증분 정책 적용
    |
    +-- YES
          |
          v
       JSON from / to 추출
          |
          v
       column 존재?
          |
          +-- YES --> JSON column 사용
          |
          +-- NO  --> 메타 incr_column_nm 사용
          |
          v
       수집 범위 조건 생성
          |
          v
       ADF Pipeline 실행
```

### 11.3 수집 범위 예시

JSON:

```json
{
  "from": "202607",
  "to": "202609"
}
```

Target Master:

```text
incr_column_nm   = CLYM
incr_column_type = STRING
```

최종 조건:

```sql
WHERE CLYM >= '202607'
  AND CLYM <= '202609'
```

## 12. ADF Polling

ADF는 시그널 테이블을 주기적으로 조회한다.

### 12.1 조회 SQL

```sql
SELECT
    signal_id,
    src_db_nm,
    src_schema_nm,
    src_table_nm,
    batch_date,
    batch_end_dt,
    incr_range_json,
    created_dt
FROM adfadm.tb_etl_batch_signal
WHERE created_dt >= :polling_start
  AND created_dt < :polling_end
ORDER BY created_dt, signal_id;
```

### 12.2 Safety Window

Polling 시 일정 시간의 Safety Window를 적용한다.

예시:

```text
마지막 정상 조회 시각 : 14:00
Safety Window         : 5분
다음 조회 시작 시각   : 13:55
```

Safety Window로 인해 동일 시그널이 중복 조회될 수 있으므로 ADF 처리 이력에서 `signal_id` 기준으로 중복 실행 여부를 확인한다.

장시간 미커밋 트랜잭션이나 장애 복구 시점의 누락을 방지하기 위해 미처리 시그널 재조회 절차를 별도로 운영한다.

## 13. ADF 처리 이력

ADF는 시그널 테이블의 데이터를 수정하지 않는다.

시그널 처리 결과는 ADF Meta DB에서 관리한다.

### 13.1 관리 항목

| 항목 | 설명 |
|---|---|
| signal_id | 시그널 식별자 |
| target_id | ADF 수집 대상 ID |
| master_run_id | ADF 마스터 Pipeline 실행 ID |
| ingest_pipeline_id | 수집 Pipeline 실행 식별자 |
| process_status | 처리 상태 |
| start_dt | 처리 시작 시각 |
| end_dt | 처리 완료 시각 |
| error_log | 오류 내용 |

시그널 처리 이력의 저장 방식은 기존 ADF 실행 이력 테이블 활용 여부를 검토하여 결정한다.

동일 시그널을 여러 Polling 실행이 동시에 처리하지 않도록 원자적인 중복 선점 처리가 필요하다.

## 14. 데이터 처리 예시

### 14.1 시그널 등록 데이터

| signal_id | DB | Schema | Table | incr_range_json |
|---|---|---|---|---|
| 1001 | DB1 | MLDADM | TB_A | NULL |
| 1002 | DB1 | MLDADM | TB_B | {"from":"202609"} |
| 1003 | DB1 | MLDADM | TB_C | {"from":"202607","to":"202609"} |
| 1004 | DB1 | MLDADM | TB_D | {"from":"202609","column":"CLYM"} |

### 14.2 ADF 처리

**Signal 1001**

기존 ADF 메타데이터에 등록된 정책을 적용한다.

**Signal 1002**

메타 증분 컬럼을 기준으로 `202609` 이상 범위를 수집한다.

**Signal 1003**

메타 증분 컬럼을 기준으로 `202607` 이상, `202609` 이하 범위를 수집한다.

**Signal 1004**

`CLYM` 컬럼을 기준으로 `202609` 이상 범위를 수집한다.

## 15. 운영 및 구현 기준

### 15.1 시그널 등록

- 시그널은 ETL 정상 완료 이후 등록한다.
- 업무별로 ETL 내부 호출 또는 TWS 후속 호출 방식을 적용한다.
- 동일 완료 건에 대한 중복 호출을 방지한다.
- INSERT는 신규 Row를 추가하는 방식으로 수행한다.
- INSERT 실패 시 최대 2회 재시도한다.

### 15.2 증분 범위

- 증분 수집 범위는 `YYYYMM` 형식을 기본으로 사용한다.
- N개월 전 계산은 INSERT Shell 스크립트에서 수행한다.
- `from`은 시작월을 포함한다.
- `to`는 선택값이며 지정 시 종료월을 포함한다.
- JSON에 지정된 범위는 해당 시그널 실행에만 적용한다.
- JSON이 NULL이면 기존 ADF 메타 정책을 사용한다.
- JSON 컬럼명과 추출 조건은 ADF에서 검증한다.

### 15.3 ADF 실행

- ADF는 시그널을 주기적으로 조회한다.
- Target Master와 매핑되지 않는 시그널은 실행하지 않고 오류 대상으로 관리한다.
- 실행 대상의 Run Group 및 증분 정책을 조회한다.
- JSON 범위가 있으면 해당 범위를 우선 적용한다.
- 실행 상태와 재처리 정보는 ADF Meta DB에서 관리한다.
- `signal_id`를 기준으로 중복 처리를 방지한다.

### 15.4 데이터 보관

시그널 테이블은 누적 INSERT 방식으로 운영한다.

보관기간은 운영 정책에 따라 정의한다.

예시:

- 최근 3개월
- 최근 6개월
- 최근 1년

보관기간이 지난 데이터는 삭제하거나 별도 이력 테이블로 보관한다.

## 16. 추가 협의 항목

- [ ] 시그널 등록 방식별 적용 업무 확정
- [ ] Azure PostgreSQL ODBC DSN 및 접속 계정 확정
- [ ] 온프레미스 ETL 서버에서 Azure PG 연결 확인
- [ ] 증분 JSON의 `YYYYMM` 유효성 검증 방식 확정
- [ ] JSON 지정 컬럼에 대한 검증 기준 확정
- [ ] 시그널별 Run Group 선택 기준 확정
- [ ] ADF Polling 주기 및 Safety Window 확정
- [ ] 동일 ETL 완료 건 중복 INSERT 처리 기준 확정
- [ ] ADF 시그널 처리 이력 저장 방식 확정
- [ ] 시그널 데이터 보관기간 확정
