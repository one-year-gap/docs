---
title: '[성능 개선] Holliverse 관리자 대시보드 회원 목록 조회 속도 최적화 (Timeout 해결)'
date: 2026-03-20
domain: 백엔드
severity: Major
status: resolved
author: 허영현
thumbnail: /docs/assets/images/uploads/Screenshot 2026-03-23 at 3.03.06 PM.png
tags:
  - 인덱스
  - DB튜닝
  - 성능개선
---

# **1. 문제 상황 (Issue)**

## **1-1. 증상**

관리자 페이지의 '고객 통합 관리' 화면에서 다중 필터링 검색 시, 데이터가 로드되지 않고 에러 UI가 노출됨

## **1-2. 현상 파악**

브라우저 네트워크 탭에서 Response Header 없이 연결이 끊기는 현상(Canceled) 확인. 포스트맨으로 해당 API 직접 호출 시 HTTP 200 응답은 떨어지나 약 10초(10,000ms) 이상 소요됨

## **1-3. 원인 요약**

백엔드의 DB 쿼리 조회 속도가 프론트엔드(Axios)의 Timeout 설정 시간을 초과하여 발생한 병목 현상

# **2. 근본 원인 분석 (Root Cause)**

- `Member(회원)` - `Subscription(구독)` - `Product(상품)` 간의 다대다(N:M) 조인 쿼리 실행 계획(`EXPLAIN ANALYZE`) 분석 진행
- 매핑 테이블 역할을 하는 `Subscription`의 외래키(`member_id`, `product_id`)에 인덱스가 누락되어 있었음
- 이로 인해 회원의 요금제를 조회하거나 특정 요금제 기반으로 회원을 필터링할 때마다 수백만 건을 뒤지는 **Full Table Scan(전체 스캔)**이 발생하여 쿼리 비용(Cost)과 실행 시간이 기하급수적으로 증가함

# **3. 해결 방안 (Solution)**

조인(JOIN) 및 다중 WHERE 조건에 빈번하게 사용되는 핵심 컬럼을 식별하여 단일 및 복합 인덱스 추가

## **3-1. 적용 쿼리**

```plain
-- 1. 회원의 요금제 내역 조회(정방향) 최적화
CREATE INDEX CONCURRENTLY idx_subscription_member_id ON subscription (member_id);

-- 2. 특정 요금제 사용자 필터링(역방향) 최적화
CREATE INDEX CONCURRENTLY idx_subscription_product_id ON subscription (product_id);

-- 3. 고객 목록 기본 조회(최신순) 및 가입일 필터링 최적화
CREATE INDEX CONCURRENTLY idx_member_role_join_date ON member (role, join_date DESC);

```

## **3-2. 인덱스 설계 전략 및 선정 기준**

이번 트러블슈팅에서는 관리자 페이지의 조회 패턴(Access Pattern)과 데이터의 카디널리티(Cardinality)를 종합적으로 분석하여 총 3개의 핵심 인덱스를 도출함 

### **① 다대다(N:M) 조인 병목 해소: 외래키(FK) 인덱스**

#### **대상**

`subscription` 테이블의 `member_id`, `product_id`

#### **선정 이유**

 관리자 페이지 특성상 "특정 고객이 가입한 요금제 목록"이나 "특정 요금제를 이용 중인 고객 목록"을 교차 조회하는 경우가 매우 빈번함 

- `subscription`은 `member`와 `product`를 연결하는 핵심 매핑 테이블로, 모든 필터링 쿼리의 `JOIN ON` 조건과 `WHERE`절에 항상 포함됨 
- 특히, PostgreSQL은 제약조건으로 Foreign Key를 설정하더라도 인덱스를 자동 생성하지 않기 때문에 수동으로 단일 인덱스를 추가하여 Full Table Scan을 Index Scan으로 유도함

### **② 복합 필터링 및 정렬 최적화: 복합 인덱스 (Composite Index)**

#### **대상**

`member` 테이블의 `(role, join_date DESC)`

#### **선정 이유**

- 관리자 페이지의 '고객 통합 관리' 첫 화면은 기본적으로 **고객(CUSTOMER)들의 목록을 최신 가입일순으로 정렬**하여 보여줌 (`WHERE role = 'CUSTOMER' ORDER BY join_date DESC`)
- `role` 컬럼 자체는 ENUM 타입(GUEST, COUNSELOR, CUSTOMER, ADMIN)으로 카디널리티가 낮지만, 관리자 조회 화면에서 **항상 고정적으로 사용되는 필수 조건**이므로 복합 인덱스의 선행 컬럼으로 배치함
- 이어서 카디널리티가 높은 `join_date`를 역순(DESC)으로 결합함. 이를 통해 DB가 데이터를 찾은 후 별도의 정렬 작업(Filesort)을 수행하지 않고, 인덱스에 이미 정렬된 순서대로 데이터를 읽어오도록(Index Range Scan) 최적화함

# **4. 적용 시 고려사항 (Zero-Downtime & DB Sync)**

## **4-1. Lock 방지**

운영 DB에 일반 `CREATE INDEX` 적용 시 테이블 락(Table Lock)이 발생하여 회원가입/결제 등 필수 서비스에 장애가 발생할 위험이 있음. 이를 방지하기 위해 PostgreSQL의 `CONCURRENTLY` 옵션을 활용, **서비스 무정지 상태로 백그라운드 인덱스 생성을 진행함**

## **4-2. 형상 관리**

이후 Flyway 마이그레이션 스크립트에 `IF NOT EXISTS` 구문을 포함한 인덱스 생성 코드를 추가하여 로컬 및 테스트 DB 환경 간의 형상 동기화 문제도 사전에 방지함

# **5. 개선 결과 (Result)**

## **5-1. 성능 개선**

기존 10초(10,000ms) 이상 소요되던 무거운 복합 필터링 API 응답 속도가 인덱스 적용 후 **약 107ms (0.1초)** 수준으로 획기적으로 단축됨 **(약 100배 이상 성능 향상)**

![](/docs/assets/images/uploads/%EC%84%B1%EB%8A%A5%EA%B0%9C%EC%84%A0.png)

## **5-2. 기대 효과**

타임아웃 에러 해결을 통한 관리자 대시보드 사용성(UX) 정상화 및 전체 스캔으로 인한 DB CPU/Memory 부하 해소
