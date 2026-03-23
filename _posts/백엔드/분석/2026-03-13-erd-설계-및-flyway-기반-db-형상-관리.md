---
title: ERD 설계 및 Flyway 기반 DB 형상 관리
date: 2026-03-13
work_type: 기타
author: 허영현
pinned: false
thumbnail: /docs/assets/images/uploads/flyway (1).png
excerpt: ERD 설계 및 Flyway 기반 DB 형상 관리
tags:
  - '#ERD #DB #형상관리 #Flyway #마이그레이션'
---

member-product

![](/docs/assets/images/uploads/member-product.png)

member-상담-키워드

![](/docs/assets/images/uploads/member-%EC%83%81%EB%8B%B4-%ED%82%A4%EC%9B%8C%EB%93%9C.png)

member-캐릭터유형

![](/docs/assets/images/uploads/member-%EC%B6%94%EC%B2%9C-%ED%8E%98%EB%A5%B4%EC%86%8C%EB%82%98.png)

member-쿠폰, 최근 본 상품, 리프레시 토큰

![](/docs/assets/images/uploads/member-%EC%BF%A0%ED%8F%B0.png)

batch 관련

![](/docs/assets/images/uploads/batch.png)

# Spring DB Migration Tool

# **1. 배경**

현재 우리 프로젝트는 **하나의 PostgreSQL 인스턴스 내에서 4개의 스키마(Core, Reco, Analytics, Notification)를 논리적으로 분리**하여 사용하는 구조

⇒ 개발 및 운영 단계에서 다음과 같은 문제점이 예상되어, 명확한 DB 형상 관리 도구가 필요

### 1) **데이터 증발 리스크**

JPA의 `ddl-auto: update`는 편리하지만, 운영 중인 데이터의 안전성을 보장하지 못함

(예: `NOT NULL` 제약 추가 시 기존 데이터 충돌로 인한 테이블 Drop 위험)

### 2) **협업 간 스키마 불일치**

팀원 A가 로컬 DB를 수정하고 코드를 푸시했을 때, 팀원 B의 로컬 환경에서 DB 에러가 발생하여 개발 흐름이 끊기는 현상이 잦음 → "내 컴퓨터에선 되는데?" 방지

### 3) **jOOQ 코드 생성 의존성**

jOOQ는 DB 스키마를 기반으로 자바 코드를 생성하므로, 소스 코드보다 DB 스키마가 먼저 확정되고 관리되어야 함

# **2. 후보군 비교**

**JPA(Hibernate DDL)**, **Liquibase**, **Flyway** 세 가지 검토

| **구분** | **1. JPA (ddl-auto)** | **2. Liquibase** | **3. Flyway** |
| --- | --- | --- | --- |
| **방식** | Java 엔티티 코드 기반 자동 생성 | XML, YAML, JSON 등 DSL 사용 | **순수 SQL 파일 (.sql)** |
| **장점** | 초기 세팅이 빠르고 간편함 | DB 벤더가 바뀌어도 호환 가능 (추상화) | **러닝커브가 없음 (SQL만 알면 됨)** |
| **단점** | **데이터 보존 불가**, 세밀한 제어 불가능 | XML/YAML 작성 번거로움, 문법 학습 필요 | DB를 바꾸면 SQL을 다시 짜야 함 |
| **평가** | ❌ 프로토타입용, 운영 불가 | ⚠️ 7주 프로젝트에 과한 스펙 | ✅ **우리 프로젝트에 최적** |

# 3. 결정

**우리 프로젝트는 Flyway를 공식 DB 마이그레이션 도구로 채택**

# **4. 상세 선정 근거**

### **1) “운영 데이터 보호”가 최우선**

JPA의 `update` 옵션은 기존 데이터가 있는 상태에서 구조를 변경할 때 매우 취약 예를 들어 `User` 테이블에 필수값(`NOT NULL`)인 `nickname` 컬럼을 추가하는 경우

- \*\*JPA:\*\* 기존 데이터가 Null이므로 에러가 발생하거나, 해결 과정에서 데이터를 날릴 위험이 큼
- \*\*Flyway: **SQL 스크립트를 통해** [ 컬럼 추가(Null 허용) → 기존 데이터에 기본값 채우기(UPDATE) → Not Null 제약 걸기 ]\*\* 순서로 안전하게 마이그레이션이 가능

### **2) PostgreSQL 특화 기능(JSONB, Multi-Schema)의 제약 없는 활용**

우리 프로젝트는 **추천 시스템의 페르소나 태그 저장을 위해 PostgreSQL의** `JSONB` **타입** 사용, **스키마를 명시적으로 분리**

→ 이를 구현하기 위해 **Flyway**가 필수적인 이유는 다음과 같음

**2-1. JSONB 및 GIN Index 성능 최적화**

- \*\*배경:\*\* 페르소나 태그 검색 성능을 위해 `JSONB` 타입과 `GIN(Generalized Inverted Index)` 인덱싱 필수
- \*\*JPA의 한계:\*\* 표준 JPQL은 JSON 내부 필드에 대한 인덱스 스캔(Index Scan) 쿼리를 지원하지 X
결국 복잡한 Native Query를 작성해야 하며, `hypersistence-utils` 같은 별도 라이브러리 의존성 발생
- \*\*Liquibase의 한계:\*\* Liquibase의 표준 XML 문법 `<createIndex>`은 PostgreSQL 전용인 `USING GIN` 옵션이나 `jsonb_path_ops` 연산자 클래스를 지원하지 X
이를 구현하려면 결국 `<sql>` 태그 안에 Raw SQL을 적어야 하므로 추상화의 장점이 사라짐
- **Flyway의 이점: Native SQL을 그대로 사용하므로, PostgreSQL의 인덱싱 기능을 제약 없이 100% 활용 가능**

**2-2. 멀티 스키마(Multi-Schema) 및 권한 관리(DCL)의 용이성**

- \*\*배경:\*\* 4개의 스키마(`core`, `reco`, `analytics`, `noti`)를 분리하고, 각 Application User에게 최소 권한만 부여
- **JPA의 한계: `ddl-auto`는 테이블 생성(DDL)만 수행할 뿐, 스키마 자체 생성(**`CREATE SCHEMA`**)이나 권한 부여(**`GRANT/REVOKE`**) 같은 DCL은 수행하지 못함**
- \*\*Liquibase의 한계:\*\* 스키마별로 ChangeLog 파일을 쪼개서 관리해야 하며, 권한 부여 구문은 표준 태그가 없어 관리가 번거로움
- **Flyway의 이점: 하나의 SQL 파일(`V1__init.sql`) 내에서 스키마 생성 → 테이블 생성 → 권한 부여**의 흐름을 절차적으로 완벽하게 제어할 수 있어 인프라 프로비저닝이 간결해짐

### 3) **짧은 프로젝트 기간과 낮은 러닝커브**

총 7주라는 짧은 개발 기간 동안, 팀원들이 Liquibase의 XML/YAML 문법을 새로 익히는 것은 비효율적 Flyway는 백엔드 개발자라면 누구나 아는 **표준 SQL**을 사용하므로, 도입 즉시 모든 팀원이 스크립트를 작성하고 리뷰 가능

### **4) jOOQ와의 파이프라인 일치**

분석/추천 모듈은 복잡한 통계 쿼리를 위해 **jOOQ**를 사용 

- **Workflow:** `Flyway SQL 실행` → `DB 형상 변경` → `jOOQ Code Generator` → `Java 객체 생성` 
- 이 흐름을 구축하기 위해서는 DB 상태를 확실하게 보장해주는 SQL 기반의 도구가 필수적

# **5. 리퀴베이스 (+ 버린 이유)**

### **1. 리퀴베이스(Liquibase)란?**

**Flyway가 SQL**을 그대로 쓰는 거라면, 리퀴베이스는 **XML, YAML, JSON** 같은 **별도의 문법**으로 DB를 정의

🆚 코드 비교

**상황:** `member` **테이블 만들기**

**Flyway (SQL 그대로)**

```plain
-- V1__create_member.sql
CREATE TABLE member (
    id BIGINT PRIMARY KEY,
    name VARCHAR(255)
);
```

**Liquibase (XML 방식)**

```plain
<changeSet id="1" author="dba">
    <createTable tableName="member">
        <column name="id" type="BIGINT">
            <constraints primaryKey="true"/>
        </column>
        <column name="name" type="VARCHAR(255)"/>
    </createTable>
</changeSet>
```

리퀴베이스는 코드가 **훨씬 길고 복잡**

### 2. 리퀴베이스의 장점 (근데 왜 쓰는 거야?)

이렇게 복잡한데도 쓰는 이유는 **DB 독립성** 때문

- **만능 통역기:** 위 XML 코드를 한 번만 짜두면, 리퀴베이스가 알아서 상황에 맞춰 번역해줌
    - Oracle에 연결하면? -> Oracle 문법 SQL로 변환해서 실행
    - MySQL에 연결하면? -> MySQL 문법 SQL로 변환해서 실행
- **자동 롤백:** `createTable` 같은 건 자동으로 취소(Drop)하는 기능도 만들어줌

> **즉, "우리 회사는 고객사마다 DB를 다르게 써요(어디는 오라클, 어디는 MySQL)" 하는 솔루션 회사에서는 리퀴베이스 추천**

### 3. 우리 프로젝트에서 리퀴베이스를 버린 이유 (탈락 사유)

#### ❌ 이유 1: 우리는 'PostgreSQL' 하나만 판다.

우리는 지금 돈 아끼려고 AWS RDS(PostgreSQL) 딱 하나만 씀, 나중에 오라클로 바꿀 계획 없음

#### ❌ 이유 2: PostgreSQL의 '특수 기능(JSONB)' 쓰기 힘들다.

우리는 추천 시스템 때문에 `JSONB` 타입이나 `Partial Index` 같은 **PostgreSQL 전용 기능**을 써야함 리퀴베이스는 "모든 DB에서 되는 기능"을 위주로 만들어져 있어서, 이런 특수 기능을 쓰려면 설정이 엄청 복잡하거나 결국 SQL을 직접 써야함

> _Flyway는 그냥 SQL에 `jsonb`라고 적으면 끝_

#### ❌ 이유 3: 7주라는 시간 제한 (생산성)

- **Flyway:** 러닝커브가 없음 (SQL만 알면 됨)
- **Liquibase:** XML/YAML 작성 번거로움, 문법 학습 필요

### 4. 결론

> 리퀴베이스는 **여러 종류의 DB를 동시에 지원해야 할 때** 좋은 도구 하지만 우리는 **PostgreSQL의 기능을 100% 활용**해야 하고, **개발 속도**가 중요 굳이 XML 문법을 새로 배우는 비용을 치르지 말고, 익숙한 **SQL(Flyway)**로 가자

📎 **참고 자료** (참고한 블로그 등)

[데이터 베이스 형상관리(Migration)툴 비교 Flyway vs Liquibase](https://velog.io/@kjy0302014/%EB%8D%B0%EC%9D%B4%ED%84%B0-%EB%B2%A0%EC%9D%B4%EC%8A%A4-%ED%98%95%EC%83%81%EA%B4%80%EB%A6%ACMigration%ED%88%B4-%EB%B9%84%EA%B5%90)

[[SpringBoot] 데이터베이스 마이그레이션 툴 Flyway 도입기](https://ywoosang.tistory.com/18)

***
