# 최근 RDS 경보 발생 구간 분석

## 1. 요약

RDS에서 DB 부하 관련 경보가 빈번하게 발생하고 있어, 가장 최근 사례를 기준으로 관련 지표와 실행 쿼리를 확인하였다.

2026년 8월 31일 00:23에 **동시 실행 스레드 수 증가 경보**가 발생했으며, 이후 **vCPU 대비 DB Load 경보**와 **RDS 종합 부하 경보**가 연이어 발생했다.

해당 시점에 장시간 실행 중인 쿼리가 확인되었고, 해당 쿼리를 종료한 뒤 DB 부하가 정상 범위로 돌아오면서 경보가 해제되었다. 이번 사례에서는 해당 쿼리들이 부하에 영향을 준 것으로 판단된다.

다만 다른 시점에는 별도의 쿼리나 여러 쿼리의 동시 실행 등 다양한 요인으로 유사한 현상이 발생할 수 있다. 따라서 현재 모니터링 방식과 시스템 구조를 기준으로 다음 사항에 대한 검토와 자문이 필요하다.

- DB 부하 발생 원인을 효과적으로 식별하기 위해 지속적으로 확인해야 할 지표와 로그
- 현재 시스템 구조에서 우선적으로 점검해야 할 부분
- 반복 발생 시 원인 쿼리를 신속하게 특정할 수 있는 모니터링 및 대응 방안

---

## 2. 시스템 구성

```text
[사용자]
   ↓
[CloudFront / ALB]
   ↓
┌─────────────────────────────┐
│ EC2 #1 / EC2 #2             │
│                             │
│ - Next.js                   │
│ - PHP CodeIgniter3          │
│ - Socket.IO                 │
└─────────────────────────────┘
   ↓
[RDS MySQL]

[배치 / 주기 작업]

EventBridge
   ↓
Lambda
   ↓
PHP API
   ↓
RDS MySQL
```

| 구성 요소 | 역할 |
| --- | --- |
| CloudFront / ALB | 사용자 요청 전달 및 EC2 트래픽 분산 |
| EC2 | Next.js, PHP CodeIgniter3, Socket.IO 서비스 운영 |
| RDS MySQL | 서비스 주요 데이터 저장 |
| EventBridge | 정해진 주기에 Lambda 실행 |
| Lambda | PHP API 호출 등 배치 처리 |

---

## 3. 지표 및 알림 설명

### 3.1. CPU 사용량

**경보명:** `RDS CPU사용량 급증 처리바람`

- RDS 인스턴스의 CPU 사용률을 모니터링한다.
- CPU 사용량이 기준치 이상으로 증가하면 CPU 처리 부하가 높은 상태로 판단한다.
- **경보 조건:** 5분 내 1개 데이터 포인트에서 `CPUUtilization >= 98`

> 이미지 삽입 위치: CPU 사용량 경보 설정 또는 지표 화면

### 3.2. DB Connection

**경보명:** `RDS DB CONNECTION 높음`

- 현재 RDS에 연결된 전체 DB Connection 수를 모니터링한다.
- 실제 쿼리를 수행 중인 연결뿐 아니라 Sleep 상태의 연결도 포함한다.
- 단독 알림보다는 다른 부하 지표와 함께 상태를 판단하기 위한 지표로 사용한다.
- **경보 조건:** 5분 내 1개 데이터 포인트에서 `DatabaseConnections >= 100`

> 이미지 삽입 위치: DB Connection 경보 설정 또는 지표 화면

### 3.3. vCPU 대비 DB Load

**경보명:** `RDS CPU 처리능력 대비 DB 부하 높음`

- 현재 DB Load가 RDS의 최대 vCPU 처리 능력을 초과하는지 모니터링한다.
- CPU 작업뿐 아니라 I/O 및 Lock 대기 등으로 인해 DB에서 동시에 처리되거나 대기 중인 작업이 많아지는 상황을 확인하는 지표다.
- **경보 조건:** 5분 내 1개 데이터 포인트에서 `DBLoadRelativeToNumVCPUs >= 25`
- 기준치를 초과하면 개별 알림을 발송한다.

> 이미지 삽입 위치: vCPU 대비 DB Load 경보 설정 또는 지표 화면

### 3.4. 동시 실행 스레드 수

**경보명:** `RDS 동시 실행 스레드 수 높음`

- Sleep 상태를 제외하고 현재 DB에서 작업을 수행하거나 대기 중인 스레드 수를 모니터링한다.
- 동시에 실행되는 DB 작업이 비정상적으로 증가하는 상황을 확인하는 지표다.
- **경보 조건:** 3분 내 2개 데이터 포인트에서 `runningThreads > 60`
- 기준치를 초과하면 개별 알림을 발송한다.

#### 기준치 변경 이력

| 변경일 | 변경 내용 | 변경 사유 |
| --- | --- | --- |
| 2026.08.13 | 30 → 45 | 서비스 영향 없이 알림이 빈번하게 발생하여 상향 |
| 2026.08.25 | 45 → 60 | 동일 사유로 추가 상향 |

> 이미지 삽입 위치: 동시 실행 스레드 수 경보 설정 또는 지표 화면

### 3.5. RDS 종합 부하 이상 감지

**경보명:** `RDS 종합 부하 이상 감지(DB 커넥션, CPU 급증, vCPU 대비 DB 부하)`

- 다음 세 지표 중 **2개 이상이 동시에 기준치를 초과할 때** 알림을 발송한다.
  - CPU 사용량
  - DB Connection
  - vCPU 대비 DB Load
- 단일 지표의 일시적인 증가보다 여러 부하 징후가 동시에 발생하는 상황을 감지하기 위한 복합 경보다.

> 이미지 삽입 위치: RDS 종합 부하 복합 경보 설정 화면

### 3.6. 경보 조건 요약

| 구분 | 지표 | 조건 | 평가 구간 |
| --- | --- | --- | --- |
| CPU 사용량 | `CPUUtilization` | `>= 98` | 5분 내 1개 데이터 포인트 |
| DB Connection | `DatabaseConnections` | `>= 100` | 5분 내 1개 데이터 포인트 |
| vCPU 대비 DB Load | `DBLoadRelativeToNumVCPUs` | `>= 25` | 5분 내 1개 데이터 포인트 |
| 동시 실행 스레드 수 | `runningThreads` | `> 60` | 3분 내 2개 데이터 포인트 |
| RDS 종합 부하 | 위 3개 지표의 복합 조건 | 2개 이상 동시 초과 | 복합 경보 설정 기준 |

---

## 4. 최근 발생 경보 분석

### 4.1. 분석 개요

최근 경보 발생 시점인 **2026.08.31 00:23**을 기준으로 전후 약 30분간의 지표를 분석하였다.

| 구분 | 내용 |
| --- | --- |
| 분석 기간 | 2026.08.30 23:50 ~ 2026.08.31 00:50 |
| 경보 발생 구간 | 2026.08.31 00:23:14 ~ 00:28:14 |
| 확인 대상 | CloudWatch 주요 RDS 지표 및 Database Insights |

### 4.2. 발생 경보 타임라인

| 발생 시각 | 경보 | 비고 |
| --- | --- | --- |
| 2026.08.31 00:23 | `RDS 동시 실행 스레드 수 높음` | 최초 발생 |
| 2026.08.31 00:24 | `RDS CPU 처리능력 대비 DB 부하 높음` | 최초 경보 약 1분 후 발생 |
| 2026.08.31 00:24 | `RDS 종합 부하 이상 감지` | 복합 경보 발생 |

> 이미지 삽입 위치: 발생 경보 목록 또는 경보 타임라인 화면

### 4.3. CloudWatch 지표

> 이미지 삽입 위치: CloudWatch 주요 RDS 지표 1

> 이미지 삽입 위치: CloudWatch 주요 RDS 지표 2

> 이미지 삽입 위치: CloudWatch 주요 RDS 지표 3

> 이미지 삽입 위치: CloudWatch 주요 RDS 지표 4

> 이미지 삽입 위치: CloudWatch 주요 RDS 지표 5

### 4.4. Database Insights Top SQL

#### 분석 시점 Top SQL 25

> 이미지 삽입 위치: 분석 시점 Top SQL 25 화면 1

> 이미지 삽입 위치: 분석 시점 Top SQL 25 화면 2

#### 경보 발생 직전 Top SQL 25

> 이미지 삽입 위치: 경보 발생 직전 Top SQL 25 화면

---

## 5. 경보 발생 시 확인 및 처리 내용

2026.08.31 00:23에 동시 실행 스레드 수(`runningThreads`) 경보가 먼저 발생했고, 약 1분 뒤인 00:24에 vCPU 대비 DB Load 경보와 RDS 종합 부하 경보가 발생했다.

해당 시점에 `SHOW FULL PROCESSLIST;`를 실행하여 쿼리 상태를 확인한 결과, 장시간 실행되고 있는 쿼리 두 건이 확인되었다. 이 쿼리들은 경보 발생 직전 Top SQL 25의 2번째 및 3번째 항목에 해당한다.

해당 쿼리를 종료한 뒤 지표가 정상 범위로 돌아왔고 경보도 해제되었다.

### 5.1. 확인된 쿼리 1: 캠페인 리스트 조회(모집 현황)

> 이미지 삽입 위치: 캠페인 리스트 조회 쿼리 관련 화면

```sql
SELECT ANY_VALUE(U_name) AS U_name,
       ANY_VALUE(U_nick) AS U_nick,
       ANY_VALUE(CV_state) AS CV_state,
       C_idx,
       C_title,
       C_regi_start_date,
       C_regi_end_date,
       C_choice_start_date,
       C_choice_date,
       C_start_date,
       C_end_date,
       C_thumb_img_path,
       C_choice_count,
       C_volunteer_count,
       C_provision,
       C_provision_price,
       C_provide_reward,
       C_recruit_type,
       C_goods_name,
       C_goods_cost,
       C_purchase_url,
       C_etc_market,
       C_emergency_recruit,
       C_click_cnt,
       C_auto_regi_count,
       C_state,
       C_isDiscount_Campaign,
       C_discount_rate_level1,
       C_discount_rate_level2,
       C_discount_rate_level3,
       C_discount_rate_level4,
       C_max_discount_price,
       C_reject_reason,
       C_payment_number,
       C_uct_idx,
       CAR_regi_start_date,
       CAR_regi_end_date,
       CAR_c_idx,
       CAR_count,
       UCT_name,
       _CAMPAIGN_CONNECT.CNC_choice_method,
       IFNULL(CAR_isAutoRegi, 'N') AS CAR_isAutoRegi,
       GROUP_CONCAT(DISTINCT CS_type) AS CS_type,
       GROUP_CONCAT(DISTINCT CI_img_path ORDER BY CI_idx ASC) AS thumb_img,
       COUNT(DISTINCT CASE
           WHEN CV_state = 'HOLD' AND CV_blocked != 'Y' THEN CV_idx
       END) AS CV_recruit,
       COUNT(DISTINCT CASE
           WHEN CV_state = 'HOLD' AND DATEDIFF(NOW(), CV_regi_date) <= 3 THEN CV_idx
       END) AS CV_new,
       COUNT(DISTINCT CASE
           WHEN CV_state IN ('CHOICE', 'COMPLETE') THEN CV_idx
       END) AS CV_choice,
       COUNT(DISTINCT CASE
           WHEN CV_regi_date IS NOT NULL AND CV_visited_date IS NULL THEN CV_idx
       END) AS CV_before_visit,
       COUNT(DISTINCT CASE
           WHEN CV_visited_date IS NOT NULL AND CV_url_date IS NULL THEN CV_idx
       END) AS CV_reviewing,
       COUNT(DISTINCT CASE
           WHEN CV_url_date IS NOT NULL THEN CV_idx
       END) AS CV_complete,
       COUNT(DISTINCT CASE
           WHEN CV_state = 'CHOICE' AND (CV_url IS NULL OR CV_url = '') THEN CV_idx
       END) AS CV_unsubmit,
       COUNT(DISTINCT CASE
           WHEN CV_state IS NOT NULL THEN CV_idx
       END) AS CV_total_apply,
       COUNT(DISTINCT CASE
           WHEN CV_url_check_date IS NULL THEN CV_idx
       END) AS CV_notViewed,
       COUNT(DISTINCT CASE
           WHEN CV_state = 'COMPLETE' THEN CV_idx
       END) AS CV_submission,
       COUNT(DISTINCT CASE
           WHEN CV_state IN ('CEN', 'REQCEN', 'CENREJ') THEN CV_idx
       END) AS CV_cancel,
       ANY_VALUE(isFirstPurchase.cnt) AS isFirstPurchase,
       C_is_connect_campaign,
       CASE WHEN CNC_idx > 0 THEN 'Y' ELSE 'N' END AS is_set_connect_condition,
       PH.PH_product AS connect_plan,
       IF(PH.PH_idx, 'Y', 'N') AS isConnectCompany,
       IFNULL(CVL.CVL_count, 0) AS reachCount,
       CASE
           WHEN PH.PH_idx IS NULL THEN 'NO_PRODUCT'
           WHEN PH.PH_product_end_date < CURDATE() THEN 'EXPIRED'
           ELSE 'ACTIVE'
       END AS connect_plan_status,
       LEAST(
           ROUND(
               DATEDIFF(NOW(), C_regi_start_date)
               / NULLIF(DATEDIFF(C_regi_end_date, C_regi_start_date), 0)
               * 100,
               0
           ),
           100
       ) AS recruit_time_rate,
       ROUND(
           COUNT(DISTINCT CASE
               WHEN CV_state = 'HOLD' AND CV_blocked != 'Y' THEN CV_idx
           END) / NULLIF(C_choice_count, 0) * 100,
           0
       ) AS apply_rate
FROM _CAMPAIGN
LEFT JOIN _USER_T_CLIENT_COMPANY
       ON UCC_uct_idx = C_uct_idx
LEFT JOIN _CAMPAIGN_IMG
       ON _CAMPAIGN_IMG.campaign_idx = C_idx
LEFT JOIN _CAMPAIGN_VOLUNTEER
       ON _CAMPAIGN_VOLUNTEER.campaign_idx = C_idx
LEFT JOIN _USER_WITHDRAWAL
       ON _USER_WITHDRAWAL.user_idx = _CAMPAIGN_VOLUNTEER.user_idx
LEFT JOIN _USER_T_C_TYPE
       ON _USER_T_C_TYPE.UCT_idx = C_uct_idx
LEFT JOIN _CAMPAIGN_SNS
       ON _CAMPAIGN_SNS.CS_c_idx = C_idx
LEFT JOIN _CAMPAIGN_AUTO_REGI
       ON _CAMPAIGN_AUTO_REGI.CAR_idx = _CAMPAIGN.C_car_idx
LEFT JOIN _USER_T_Client
       ON UCC_u_idx = _USER_T_Client.user_idx
LEFT JOIN _USER
       ON _USER.U_idx = _CAMPAIGN_VOLUNTEER.user_idx
LEFT JOIN _CAMPAIGN_CONNECT
       ON CNC_c_idx = C_idx
LEFT JOIN (
    SELECT PH_idx,
           PH_product,
           PH_product_end_date,
           PH_product_target,
           PH_product_type
    FROM _PAY_HISTORY
    WHERE PH_status = 'paid'
      AND PH_root_idx = 0
      AND PH_product_type IN (
          'connect_basic',
          'connect_recruit',
          'connect_ad',
          'connect',
          'account'
      )
) AS PH
    ON (
        PH.PH_product_type IN ('connect_basic', 'connect_recruit', 'connect_ad')
        AND C_idx = PH.PH_product_target
    )
    OR (
        PH.PH_product_type = 'connect'
        AND C_uct_idx = PH.PH_product_target
    )
-- 제공된 원문이 이 지점에서 종료되어 이후 구문은 생략됨
```

### 5.2. 확인된 쿼리 2: 리뷰 제출 기간 수정 요청 관련

> 이미지 삽입 위치: 리뷰 제출 기간 수정 요청 쿼리 관련 화면

```sql
SELECT SUM(
           CASE WHEN NTC_title = '리뷰 제출 기간 수정요청' THEN 1 ELSE 0 END
       ) AS requestReviewDateEditCount,
       MAX(
           CASE WHEN NTC_title = '리뷰 제출 기간 수정요청' THEN UNTC_idx END
       ) AS requestReviewDateEditUNTC_idx,
       MAX(
           CASE
               WHEN NTC_title = '리뷰 제출 기간 수정요청'
               THEN SUBSTRING_INDEX(SUBSTRING_INDEX(NTC_url, 'cidx=', -1), '&', 1)
           END
       ) AS requestReviewDateEditCidx,
       (
           SELECT C_title
           FROM _CAMPAIGN
           WHERE C_idx = (
               SELECT MAX(
                   CASE
                       WHEN NTC_title = '리뷰 제출 기간 수정요청'
                       THEN SUBSTRING_INDEX(SUBSTRING_INDEX(NTC_url, 'cidx=', -1), '&', 1)
                   END
               )
           )
       ) AS requestReviewDateEditCtitle,
       (
           SELECT CV_idx
           FROM _CAMPAIGN
           LEFT JOIN _CAMPAIGN_VOLUNTEER ON C_idx = campaign_idx
           WHERE C_idx = (
               SELECT MAX(
                   CASE
                       WHEN NTC_title = '리뷰 제출 기간 수정요청'
                       THEN SUBSTRING_INDEX(SUBSTRING_INDEX(NTC_url, 'cidx=', -1), '&', 1)
                   END
               )
           )
             AND CV_idx = (
                 SELECT MAX(
                     CASE
                         WHEN NTC_title = '리뷰 제출 기간 수정요청'
                         THEN SUBSTRING_INDEX(SUBSTRING_INDEX(NTC_url, 'uidx=', -1), '&', 1)
                     END
                 )
             )
           LIMIT 1
       ) AS requestReviewDateEditCV_idx,
       (
           SELECT CV_request_deadline_date
           FROM _CAMPAIGN
           LEFT JOIN _CAMPAIGN_VOLUNTEER ON C_idx = campaign_idx
           WHERE C_idx = (
               SELECT MAX(
                   CASE
                       WHEN NTC_title = '리뷰 제출 기간 수정요청'
                       THEN SUBSTRING_INDEX(SUBSTRING_INDEX(NTC_url, 'cidx=', -1), '&', 1)
                   END
               )
           )
             AND CV_idx = (
                 SELECT MAX(
                     CASE
                         WHEN NTC_title = '리뷰 제출 기간 수정요청'
                         THEN SUBSTRING_INDEX(SUBSTRING_INDEX(NTC_url, 'uidx=', -1), '&', 1)
                     END
                 )
             )
           LIMIT 1
       ) AS requestReviewDateEditCV_request_deadline_date,
       (
           SELECT CV_request_deadline
           FROM _CAMPAIGN
           LEFT JOIN _CAMPAIGN_VOLUNTEER ON C_idx = campaign_idx
           WHERE C_idx = (
               SELECT MAX(
                   CASE
                       WHEN NTC_title = '리뷰 제출 기간 수정요청'
                       THEN SUBSTRING_INDEX(SUBSTRING_INDEX(NTC_url, 'cidx=', -1), '&', 1)
                   END
               )
           )
             AND CV_idx = (
                 SELECT MAX(
                     CASE
                         WHEN NTC_title = '리뷰 제출 기간 수정요청'
                         THEN SUBSTRING_INDEX(SUBSTRING_INDEX(NTC_url, 'uidx=', -1), '&', 1)
                     END
                 )
             )
           LIMIT 1
       ) AS requestReviewDateEditCV_request_deadline,
       (
           SELECT U_nick
           FROM _USER
           WHERE U_idx = (
               SELECT user_idx
               FROM _CAMPAIGN_VOLUNTEER
               WHERE CV_idx = (
                   SELECT MAX(
                       CASE
                           WHEN NTC_title = '리뷰 제출 기간 수정요청'
                           THEN SUBSTRING_INDEX(SUBSTRING_INDEX(NTC_url, 'uidx=', -1), '&', 1)
                       END
                   )
               )
               LIMIT 1
           )
       ) AS requestReviewDateEditNick,
       SUM(
           CASE WHEN NTC_title = '캠페인 선정 취소 요청' THEN 1 ELSE 0 END
       ) AS requestChoiceCancleCount,
       MAX(
           CASE WHEN NTC_title = '캠페인 선정 취소 요청' THEN UNTC_idx END
       ) AS requestChoiceCancleUNTC_idx,
       MAX(
           CASE
               WHEN NTC_title = '캠페인 선정 취소 요청'
               THEN SUBSTRING_INDEX(SUBSTRING_INDEX(NTC_url, 'cidx=', -1), '&', 1)
           END
       ) AS requestChoiceCancleCidx,
       (
           SELECT CV_idx
           FROM _CAMPAIGN
           LEFT JOIN _CAMPAIGN_VOLUNTEER ON C_idx = campaign_idx
           WHERE C_idx = (
               SELECT MAX(
                   CASE
                       WHEN NTC_title = '캠페인 선정 취소 요청'
                       THEN SUBSTRING_INDEX(SUBSTRING_INDEX(NTC_url, 'cidx=', -1), '&', 1)
                   END
               )
           )
             AND _CAMPAIGN_VOLUNTEER.user_idx = (
                 SELECT MAX(
                     CASE
                         WHEN NTC_title = '리뷰 제출 기간 수정요청'
                         THEN SUBSTRING_INDEX(SUBSTRING_INDEX(NTC_url, 'uidx=', -1), '&', 1)
                     END
                 )
             )
           LIMIT 1
       ) AS requestChoiceCancleCV_idx,
       (
           SELECT C_title
           FROM _CAMPAIGN
           WHERE C_idx = (
               SELECT MAX(
                   CASE
                       WHEN NTC_title = '캠페인 선정 취소 요청'
                       THEN SUBSTRING_INDEX(SUBSTRING_INDEX(NTC_url, 'cidx=', -1), '&', 1)
                   END
               )
           )
       ) AS requestChoiceCancleCtitle,
       SUM(
           CASE WHEN NTC_title = '1:1 협찬 제안 수락' THEN 1 ELSE 0 END
       ) AS acceptProposalCount,
       MAX(
           CASE WHEN NTC_title = '1:1 협찬 제안 수락' THEN UNTC_idx END
       ) AS acceptProposalUNTC_idx,
       MAX(
           CASE
               WHEN NTC_title = '1:1 협찬 제안 수락'
               THEN SUBSTRING_INDEX(NTC_message, '님이', 1)
           END
       ) AS acceptProposalNick
FROM _USER_NOTIFICATION
LEFT JOIN _NOTIFICATION
       ON NTC_idx = UNTC_ntc_idx
WHERE UNTC_u_idx =
-- 제공된 원문이 이 지점에서 종료되어 조건값 및 이후 구문은 생략됨
```

---

## 6. 분석 결과

이번 경보 발생 구간에서는 다음과 같은 흐름이 확인되었다.

1. 동시 실행 스레드 수가 먼저 증가했다.
2. 약 1분 뒤 vCPU 대비 DB Load와 종합 부하 경보가 발생했다.
3. `SHOW FULL PROCESSLIST;`에서 장시간 실행 중인 특정 쿼리들이 확인되었다.
4. 해당 쿼리를 종료한 뒤 DB 부하가 정상 범위로 돌아왔다.
5. 이후 관련 경보가 해제되었다.

위 시간적 연관성을 기준으로 볼 때, 이번 사례에서는 장시간 실행된 쿼리들이 DB 부하 상승에 영향을 준 것으로 판단된다.

다만 이번 결과만으로 모든 경보의 원인이 동일하다고 단정하기는 어렵다. 다른 시점에는 별도의 단일 쿼리, 여러 쿼리의 동시 실행, Lock 또는 I/O 대기 등 다양한 요인으로 유사한 현상이 발생할 가능성이 있다.

---

## 7. 검토 및 자문 요청 사항

DB 부하가 반복적으로 발생할 때 원인 쿼리를 효과적으로 식별하고 대응할 수 있도록 다음 사항에 대한 의견을 요청한다.

1. DB 부하 발생 시 원인을 식별하기 위해 지속적으로 수집하고 확인해야 할 CloudWatch 및 Database Insights 지표
2. 실행 쿼리, 대기 이벤트, Lock, Slow Query 등과 관련해 보존하거나 추가로 활성화해야 할 로그
3. 현재 시스템 구조에서 우선적으로 점검해야 할 애플리케이션, 배치 작업 및 DB Connection 관리 항목
4. 경보 발생 시점의 실행 쿼리와 호출 주체(EC2, Lambda, PHP API 등)를 연계하여 추적하는 방안
5. 장시간 실행 쿼리 또는 동시 실행 쿼리 증가를 조기에 탐지하고 대응하기 위한 경보 기준과 운영 절차

