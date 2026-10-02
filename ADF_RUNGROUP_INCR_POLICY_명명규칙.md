# ADF 수집 Run Group / INCR Policy 명명 규칙 및 신규 대상 등록 가이드

---

## 1. 대상 테이블

| 테이블 | 식별 컬럼 | 역할 |
|---|---|---|
| `adfadm.ctl_ingest_incr_policy` | `policy_cd` | 증분 조회 범위 정책 정의 |
| `adfadm.ctl_ingest_run_group` | `run_group_nm` | 실행 주기 단위의 그룹 정의 |
| `adfadm.ctl_ingest_target_master` | `target_id` | 수집 대상. `incr_policy_id`로 증분 정책 참조 |
| `adfadm.ctl_ingest_target_run_rule` | `run_rule_id` | 대상(`target_id`)과 실행 그룹(`run_group_id`) 매핑 |

```text
ctl_ingest_incr_policy ──< ctl_ingest_target_master >──< ctl_ingest_target_run_rule >── ctl_ingest_run_group
```

- 증분 정책은 `target_master.incr_policy_id`로 대상에 연결됩니다.
- FULL 대상의 `incr_policy_id`는 NULL입니다.
- 실행 그룹은 `target_run_rule`을 통해 대상에 연결됩니다.
- 정책과 그룹은 서로 독립적으로 관리됩니다.

---

## 2. Run Group과 INCR Policy의 구분

| 구분 | 의미 | 예시 |
|---|---|---|
| Run Group (`run_group_nm`) | **언제, 얼마나 자주** 실행하는가 | `MLIWP_INCR_MINUTE_10` (10분마다 실행) |
| INCR Policy (`policy_cd`) | 실행 시 **어디서부터** 조회하는가 | `BATCH_MINUTE_60` (실행 시점 기준 최근 60분부터) |

예: `MLIWP_INCR_MINUTE_10` 그룹에 `BATCH_MINUTE_60` 정책을 적용하면 10분마다 실행하며, 매 실행마다 최근 60분 데이터를 조회합니다.

---

## 3. `ctl_ingest_run_group.run_group_nm` 명명 규칙

### 3.1 형식

```text
{DB명}_{수집유형}_{실행주기단위}_{실행주기}
```

| 항목 | 설명 | 예시 |
|---|---|---|
| `DB명` | Source DB 식별명 | `MLIWP`, `MLSQP` |
| `수집유형` | `FULL`, `INCR`, `REFRESH` (3.3장의 `job_type_cd`와 일치) | `INCR` |
| `실행주기단위` | `MINUTE`, `HOUR`, `DAY`, `WEEK`, `MONTH` | `MINUTE` |
| `실행주기` | 해당 단위의 반복 간격(숫자) | `10` |

```text
MLIWP_INCR_MINUTE_10
MLIWP_INCR_HOUR_1
MLIWP_INCR_DAY_1
MLIWP_FULL_DAY_1
```

### 3.2 규칙

- 영문 대문자, 숫자, `_`만 사용합니다. 공백과 `/`, `:`, `-` 등 특수문자는 사용하지 않습니다.
- `run_group_nm`은 **UNIQUE**입니다. 동일한 이름으로 중복 등록할 수 없으므로, 등록 전에 반드시 기존 그룹을 조회합니다.
- Run Group명에는 `incr_policy`의 Lookback 범위를 포함하지 않습니다. (예: `MLIWP_INCR_MINUTE_10_LB60` ✗)
- 동일 INCR Run Group에 속한 대상은 **동일한 `incr_policy_id`를 사용**합니다.
- 같은 실행 주기인데 정책이 달라야 한다면, 임의 번호(`_G2`)보다 업무적 구분 기준이 드러나는 이름으로 별도 그룹을 만드는 것을 권장합니다. (UNIQUE 제약이 있으므로 이름은 반드시 달라야 합니다.)
- "동일 그룹 = 동일 정책" 규칙은 DB 제약이 아니라 **운영 규칙**입니다. 그룹 테이블에 정책 컬럼이 없으므로 대상 등록 시 운영자가 직접 확인해야 합니다.

### 3.3 `job_type_cd` 허용값

`ctl_ingest_run_group.job_type_cd`는 체크 제약(`ck_grp_job_type`)에 따라 아래 5개 값만 등록할 수 있습니다.

| `job_type_cd` | 이름의 `수집유형` | 비고 |
|---|---|---|
| `RAW_FULL` | `FULL` | |
| `RAW_INCR` | `INCR` | |
| `META_FULL` | `FULL` | |
| `RAW_REFRESH` | `REFRESH` | INCR 대상 중 정합성을 맞추기 위해 주기적으로 전체 데이터를 재수집하는 그룹 |
| `ADHOC` | 아래 참고 | 소스 DB별 ADHOC 대상을 묶는 단일 그룹 |

- 허용값 목록은 DDL 제약(`ck_grp_job_type`) 기준입니다.
- Run Group 이름의 `수집유형`과 `job_type_cd`가 서로 모순되지 않게 등록합니다. (예: `MLIWP_INCR_MINUTE_10` → `RAW_INCR`, `MLIWP_FULL_DAY_1` → `RAW_FULL`) 이 일치 여부는 DB가 검증하지 않으므로 운영자가 확인해야 합니다.
- `RAW_REFRESH` 그룹은 `MLIWP_REFRESH_WEEK_1`처럼 `수집유형` 자리에 `REFRESH`를 사용합니다. (이름은 예시)


**ADHOC 그룹**

- `ADHOC` 그룹은 **1개만 운영**합니다. 소스 DB별로 ADHOC 대상(`target_id`)이 1개씩 있고, 이 대상들을 해당 그룹에 묶습니다.

---

## 4. `ctl_ingest_incr_policy.policy_cd` 명명 규칙

### 4.1 컬럼 규격 (DDL 기준)

| 컬럼 | 규격 | 비고 |
|---|---|---|
| `incr_policy_id` | `int4`, PK, 자동 채번 | 직접 입력하지 않음 |
| `policy_cd` | `varchar(50)`, NOT NULL, **UNIQUE** | 중복 등록 불가 |
| `policy_nm` | `varchar(100)`, NOT NULL | 운영자가 이해하기 쉬운 명칭 |
| `policy_type_cd` | `varchar(10)`, NOT NULL | `BATCH`, `HWM`, `BIZDAY`, `CLOSE` 중 하나 |
| `range_unit_cd` | `varchar(10)`, NULL 허용 | `MINUTE`, `HOUR`, `DAY`, `MONTH` 중 하나 (`WEEK` 없음) |
| `range_interval` | `int4`, NULL 허용 | 입력 시 0보다 커야 함 |
| `enabled_yn` | `bpchar(1)`, 기본값 `'Y'` | `Y` / `N`만 허용 |
| `description` | `varchar(500)`, NULL 허용 | 용도 설명 |
| `created_by` | `varchar(100)`, NULL 허용 | 기본값 없음, 운영자가 입력 |
| `created_dt` | `timestamptz`, 기본값 `now()` | 자동 세팅 |
| `updated_by` / `updated_dt` | NULL 허용 | 기본값 없음, 수정 시 직접 입력 |

### 4.2 유형별 등록 방식

| `policy_type_cd` | `range_unit_cd` | `range_interval` | `policy_cd` 형식 | 예시 |
|---|---|---|---|---|
| `BATCH` | 필수 | 필수 | `BATCH_{UNIT}_{INTERVAL}` | `BATCH_MINUTE_60`, `BATCH_DAY_3` |
| `HWM` | 입력하지 않음(NULL) | 입력하지 않음(NULL) | `HWM` | `HWM` |
| `BIZDAY` | 입력하지 않음(NULL) | 필수 | `BIZDAY_{INTERVAL}` | `BIZDAY_1` |
| `CLOSE` | ⚠️ 확인 필요 | ⚠️ 확인 필요 | ⚠️ 확인 필요 | - |

`BATCH` 예시는 다음과 같습니다.

| `policy_cd` | 의미 |
|---|---|
| `BATCH_MINUTE_60` | 기준 시점 - 60분부터 조회 |
| `BATCH_HOUR_2` | 기준 시점 - 2시간부터 조회 |
| `BATCH_DAY_3` | 기준 시점 - 3일부터 조회 |
| `BATCH_MONTH_1` | 기준 시점 - 1개월부터 조회 |

⚠️ `HWM`, `BIZDAY`, `CLOSE`의 `policy_cd` 형식은 `BATCH` 규칙을 확장해 **제안한 안**입니다. 기존 문서에는 `BATCH`만 정의되어 있었으므로 팀 합의가 필요합니다. 각 유형의 정확한 동작 의미(예: `HWM`은 high-water mark 기반 조회로 추정, `BIZDAY`는 영업일 기준 조회로 추정, `CLOSE`는 의미 불명)는 DDL만으로 확인할 수 없습니다. 위키에 정의를 보완해 주세요.

### 4.3 규칙

- `policy_cd`는 영문 대문자, 숫자, `_`만 사용하고 50자 이내로 합니다. 공백과 특수문자는 사용하지 않습니다.
- `policy_cd`에는 `policy_type_cd`, `range_unit_cd`, `range_interval`과 같은 값을 사용합니다. (예: `BATCH_HOUR_2` → `BATCH` / `HOUR` / `2`)
- 주 단위 범위는 `range_unit_cd`에 `WEEK`가 없으므로 `DAY`로 환산합니다. (예: 2주 → `BATCH_DAY_14`)
- 이미 사용 중인 정책의 범위값은 변경하지 않습니다. 해당 정책을 쓰는 모든 대상에 영향을 주므로, 범위가 다르면 새 정책을 추가합니다.
- 더 이상 쓰지 않는 정책은 삭제하지 않고 `enabled_yn = 'N'`으로 비활성화합니다.

### 4.4 DB 제약 한계 (운영자 유의)

`ck_pol_rule` 제약이 느슨하게 작성되어 있어, 다음 규칙은 **DB가 막아주지 않습니다.**

- 제약은 OR 조건의 집합입니다. 그중 "`range_unit_cd`와 `range_interval`이 모두 NOT NULL"이면 유형과 무관하게 통과하는 조건이 있습니다. 따라서 `HWM`에 단위와 간격을 넣어도 등록됩니다.
- `CLOSE`는 단위와 간격 값에 상관없이 항상 통과합니다.
- `policy_cd`의 형식과 컬럼 값의 일치 여부는 UNIQUE 외에는 검증하지 않습니다.

따라서 4.2장, 4.3장 규칙은 운영자가 등록 시 직접 지켜야 합니다. ⚠️ 이 제약의 의도(느슨하게 둔 것인지, 개선 대상인지)는 알 수 없습니다.

### 4.5 ⚠️ 확인 필요

- `CLOSE` 유형의 단위/간격 사용 여부와 `policy_cd` 형식

---

## 5. 수집 대상 추가 시 절차

수집 대상을 추가할 때 **필요한 증분 정책 또는 실행 그룹이 이미 있으면 재사용**하고, **없으면 먼저 추가**해야 합니다.

### 5.1 순서

```text
① 수집 요건 확인 (DB, 수집유형, 실행주기, 증분 조회 범위)
② 증분 정책 조회 → 없으면 ctl_ingest_incr_policy에 추가   (INCR 대상만)
③ Run Group 조회   → 없으면 ctl_ingest_run_group에 추가
④ ctl_ingest_target_master에 대상 등록 (incr_policy_id 지정)
⑤ ctl_ingest_target_run_rule에 대상 ↔ Run Group 매핑 등록
⑥ 등록 결과 점검
```

②, ③이 ④, ⑤보다 먼저 와야 합니다. 대상은 정책 ID를, 매핑은 그룹 ID와 대상 ID를 참조하기 때문입니다.

### 5.2 기존 정책 / 그룹 조회

```sql
-- 증분 정책 조회
SELECT incr_policy_id, policy_cd, policy_type_cd, range_unit_cd, range_interval, enabled_yn
FROM   adfadm.ctl_ingest_incr_policy
WHERE  policy_cd = 'BATCH_MINUTE_60';

-- Run Group 조회
SELECT run_group_id, run_group_nm, job_type_cd, parallel_cnt, enabled_yn
FROM   adfadm.ctl_ingest_run_group
WHERE  run_group_nm = 'MLSQP_FULL_DAILY_3';
```

- 조회 결과가 **있으면** 해당 ID를 그대로 사용합니다.
- 조회 결과가 **없으면** 3장, 4장의 명명 규칙에 맞춰 신규 등록합니다.
- 조회 결과가 있는데 `enabled_yn = 'N'`이면 임의로 활성화하지 말고, 사용 가능 여부를 담당자에게 확인합니다.
- `policy_cd`와 `run_group_nm`은 모두 UNIQUE이므로 같은 이름으로 중복 등록하면 오류가 발생합니다. 반드시 조회 후 등록합니다.

### 5.3 증분 정책 신규 등록

```sql
-- BATCH 예시: 10분 주기 수집 시 최근 60분 조회
INSERT INTO adfadm.ctl_ingest_incr_policy
       (policy_cd, policy_nm, policy_type_cd, range_unit_cd, range_interval, description, created_by)
VALUES ('BATCH_MINUTE_60', '배치 최근 60분', 'BATCH', 'MINUTE', 60,
        '실행 시점 기준 최근 60분부터 증분 조회', '{운영자ID}');
```

| 컬럼 | 입력 기준 |
|---|---|
| `incr_policy_id` | 입력하지 않음 (자동 채번) |
| `policy_cd` | 4장 규칙에 따른 코드 |
| `policy_nm` | 운영자가 이해하기 쉬운 명칭 (필수) |
| `policy_type_cd` / `range_unit_cd` / `range_interval` | 4.2장 유형별 방식에 따라 `policy_cd`와 일치하는 값 |
| `enabled_yn` | 생략 시 `'Y'` |
| `description` | 용도 설명 |
| `created_by` | 운영자 ID 입력 (기본값 없음) |
| `created_dt` | 입력하지 않음 (자동 세팅) |

### 5.4 Run Group 신규 등록

| 컬럼 | 입력 기준 |
|---|---|
| `run_group_nm` | 3장 규칙에 따른 이름 (UNIQUE) |
| `job_type_cd` | 3.3장 허용값 중 수집유형에 맞는 코드 (`META_FULL`, `RAW_FULL`, `RAW_REFRESH`, `RAW_INCR`, `ADHOC`) |
| `enabled_yn` | 사용 여부 |
| `description` | 용도 설명 |
| `created_by` 등 이력 컬럼 |  |

---

## 6. 정리

| 상황 | 조치 |
|---|---|
| 기존 정책과 그룹이 모두 있음 | 그대로 재사용하여 대상과 매핑만 등록 |
| 정책만 없음 | `ctl_ingest_incr_policy`에 추가 후 등록 |
| 그룹만 없음 | `ctl_ingest_run_group`에 추가 후 등록 |
| 둘 다 없음 | 정책 → 그룹 → 대상 → 매핑 순으로 등록 |
| 같은 그룹에 다른 정책이 필요함 | 그룹을 분리하여 별도 그룹 생성 |
| 기존 정책의 범위를 바꿔야 함 | 기존 정책은 변경하지 않고 새 정책 추가 |
| 정책을 더 이상 쓰지 않음 | 삭제하지 않고 `enabled_yn = 'N'` |

---
