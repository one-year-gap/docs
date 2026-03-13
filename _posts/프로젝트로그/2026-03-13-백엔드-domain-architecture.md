---
title: 백엔드 - Domain Architecture
date: 2026-02-12
stage: 완료
task: 기타
domain: ''
decision_type: 아키텍처 결정
considerations: ''
decision_reason: ''
author: 김도연
pinned: true
thumbnail: /docs/assets/images/uploads/Billing Cycle Gate Flow-2026-02-12-013948.svg
excerpt: ''
tags:
  - 설계
  - 도메인 설계
---

# 1. 서론

> [ 항상 고려해야 하는 것 ]
> - 새로운 기술/방향 도입에 대한 팀 모두의 러닝커브 (팀 전체 학습 비용이 실제 구현 일정에 반영되는가)
> - 트래픽 부하로 인한 장애가 발생했을 때, 장애 범위를 기능 단위로 분리할 필요가 있는가
> - 아직 구체화하지 못한 추천/분석 시스템이 이후에 변경될 가능성이 높은가

이전 [백엔드 - System Architecture](https://one-year-gap.github.io/docs/2026/02/10/%EB%B0%B1%EC%97%94%EB%93%9C-system-architecture/) 페이지에서 작성한대로 DDD를 도입하는 방식은 팀 모두의 러닝 커브가 크다고 판단했다. 그 대신 트래픽 분산을 위해 **core-server**를 **멀티모듈**로 분리하여 **admin**과 **customer**의 기능을 모듈 경계로 강하게 분리하여 각 경계가 서로 다른 리소스(트랜잭션, 데이터베이스 커넥션, 스레드 풀)를 사용하도록 만들었다.

다만 멀티 모듈로 분리하는 과정에서 아래와 같은 문제가 현실적으로 다가왔다.

- 모듈 경계를 나누었다고 해서 런타임 자원이 분리되는 것은 아니다. 같은 프로세스에서 동작하면 결국 스레드 풀, 커넥션 풀, 메모리, CPU를 공유한다. 따라서, 리소스 고갈에 대한 장애는 전파된다.
- 모듈 간의 의존관계를 올바르게 설계하고 유지하는 작업 자체는 비용이 크다. 공통 모듈에 대해서는 버전 관리를 해주어야 하고, 공통 모듈 범위를 조금만 잘못 잡아도 공통 모듈 자체가 비대해지거나, 반대 경우에는 중복 코드가 증가한다.
- 멀티 모듈을 도입하면 Gradle 설정, 모듈별 설정 분기, Bean 스캔 범위, 테스트 스코프등 추가 관리 포인트가 증가한다.

그래서 방향을 바꿨다. 러닝커브를 최소화하면서도 **장애/트래픽** 분리를 강하게 만들기 위해, **코드 구조로 모든 문제를 해결하려고 하지 않고 인프라에서 런타임을 먼저 분리**하기로 했다. 그리고 이후 지표를 통해 병목과 위험 신호를 확인하면서 승격하는 방식으로 접근했다.

다만 검색 엔진 도입(예: 상품 검색)이나 외부 추천 시스템 도입(예: Python 기반 추천 서비스)처럼 **변화 가능성이 큰 지점**은 **결합도를 낮게** 유지해야 한다고 보았다. 그래서 그 부분은 **Port/Adapter**로 분리하여 교체 가능성을 확보했다.

# 2. 설계 방향성

우선 해당 설계방향에 대해서는 고객 추천/분석은 아직 구체화되지 않았기 때문에 대략적인 아래와 같은 흐름으로 가정하고 유연성만 고려하여 설계하였다.

```plain
일정 주기를 두고 Batch를 통해서 100,000명 규모의 고객들의 데이터 정보를
분석(복잡한 집계 쿼리) -> 개인/그룹 단위로 추천 알고리즘 수행 -> DB 추천 스키마에 적재
```

## 1) 핵심 문제 정의 : customer 트래픽 폭주가 admin server까지 죽일 수 있다.

레이어드 아키텍처를 사용하더라도 **customer/admin** 기능이 동일 런타임에서 처리되면 아래 문제가 발생할 수 있다.

- 고객 트래픽 폭주로 인해 단일 서버의 **스레드/커넥션 풀**이 고갈되면 **admin** 요청도 타임아웃이 발생한다.
- 고객 트래픽 폭주로 **데이터베이스 병목**이 커지면 admin 쪽 복잡 집계 쿼리가 더 느려지고, 운영 대응 속도가 함께 떨어진다.

그래서 해결 방안을 아래처럼 정리했다.

1. 코드 베이스는 하나로 유지한다. 팀원들의 학습 비용을 최소화한다.
2. 런타임은 분리한다. ECS Service를 두 개로 분리하여 물리 자원을 분리한다.
3. 라우팅도 분리한다. 도메인 기반 라우팅으로 서로 다른 타겟 그룹으로 보낸다.

## 2) 코드 베이스는 하나 + 런타임은 둘

우선 우리 서비스의 라우팅 전략은 다음과 같다.

- [api.holliverse.site](http://api.holliverse.site) → **customer-api-service**로 **Routing**
- [**admin-api.holliverse.site**](http://admin-api.holliverse.site) → **admin-api-service**로 **Routing**

customer/admin은 서로 다른 컨테이너로 떠 있으므로 스레드/메모리/히카리 풀이 물리적으로 분리된다. 여기서 중요한 점은 **Spring의 profile 분리가 곧 런타임 분리인 것은 아니다.** 런타임 분리는 인프라(ECS 서비스 2개)가 만들고, **profile은 분리된 런타임이 어떤 기능/설정만 켤지 선택하는 용도**로 사용한다.

- **customer**의 **runtime**은 **_SPRING_PROFILES_ACTIVE=customer_**
- **admin**의 **runtime**은 **_SPRING_PROFILES_ACTIVE=admin_**

## 3) 강제성 부여 - code/runtime/CI/Review 레벨

##

설계 의도를 팀원들에게 강하게 이해시키더라도 실제 코드를 개발하면서 설계 주도자조차 방향성을 잃고 실수가 발생할 수 있다.

 이러한 실수는 개발자는 깨닫지 못하며 주로 팀원들간의 코드 리뷰를 통해서 인지하게 된다.

하지만, 팀원들도 알아차리지 못하는 경우도 종종 발생한다.(시간이 없어서 코드리뷰를 빡빡하게 하지 못하는 경우/코드의 의도를 알 수 없거나 알아차리지 못하는 경우) 따라서, 아래와 같은 5가지 단계로 강제성을 부여하여 실수를 최소화한다.

### **1) 코드 레벨 강제**

1. admin 컨트롤러는 admin 프로파일에서만 로딩한다.
2. customer runtime에서 admin API가 절대 뜰 수 없도록 한다.

```plain
@Profile("customer")
@RestController
@RequestMapping("/api/customer")
public class CustomerController {
  // customer 전용
}

@Profile("admin")
@RestController
@RequestMapping("/api/admin")
public class AdminController {
  // admin 전용
}
```

| 기능 | 실행 주체 | 읽는 데이터 | 쓰는 데이터 | 비고 |
| --- | --- | --- | --- | --- |
| 회원가입/로그인/상담/요금제 조회 | customer-api | core.member/core.contract/core.plan | core.\* (필요한 최소) | 온라인 트래픽 핵심 |
| 관리자 제재/정책 변경 | admin-api | core.\* | core.\* (최소 write) | 무조건 host/IP 제한 |
| 대시보드 통계 조회 | admin-api(jOOQ) | analysis.\* (사전 집계), 필요시 core read | (원칙상 write 없음) | heavy query 금지, 결과 테이블 조회 |
| 분석 배치(집계/피처 생성) | worker-analysis | core read-only | analysis write-only | 온라인 영향 최소화 |
| 추천 배치(추천 결과 생성) | worker-reco | analysis read-only | reco write-only | 서빙은 core가 reco read |

### 2) Runtime 레벨 강제

- **`/api/amdin/**`** 호출이 들어오면 요청 Host를 검사하여 \*\*`admin-api.holliverse.site`\*\*가 아니면 무조건 **`403`** error를 반환한다.

```plain
@Component
@Profile("admin") // admin 런타임에서만 활성화
public class AdminHostGuardFilter extends OncePerRequestFilter {

	@Value("${spring.admin.url.host}")
  private String ADMIN_HOST;

  @Override
  protected void doFilterInternal(
      HttpServletRequest req, HttpServletResponse res, FilterChain chain)
      throws ServletException, IOException {

    if (req.getRequestURI().startsWith("/api/admin/")) {
      if (!ADMIN_HOST.equalsIgnoreCase(req.getServerName())) {
        res.sendError(HttpServletResponse.SC_FORBIDDEN);
        return;
      }
    }
    chain.doFilter(req, res);
  }
}
```

### 3) CI 레벨 강제: ArchUnit으로 경계

아래 규칙을 테스트로 고정한다.

1. customer가 admin 코드를 의존하면 **테스트/빌드** 실패
2. controller가 repository를 직접 호출하면 **테스트/빌드** 실패
3. customer 영역에서 jOOQ import하면 **테스트/빌드** 실패( = admin 통계만 jOOQ 허용)

```plain
@AnalyzeClasses(packages = "com.holliverse")
class ArchitectureRulesTest {

  @ArchTest
  static final ArchRule customer_must_not_depend_on_admin =
      noClasses()
          .that().resideInAnyPackage("com.holliverse..customer..")
          .should().dependOnClassesThat().resideInAnyPackage("com.holliverse..admin..");

  @ArchTest
  static final ArchRule controllers_must_not_access_repository =
      noClasses()
          .that().resideInAnyPackage("com.holliverse.web.controller..")
          .should().accessClassesThat().resideInAnyPackage("com.holliverse.repository..");

  @ArchTest
  static final ArchRule customer_must_not_use_jooq =
      noClasses()
          .that().resideInAnyPackage("com.holliverse..customer..")
          .should().dependOnClassesThat().resideInAnyPackage("org.jooq..");

  @ArchTest
  static final ArchRule jooq_only_in_admin_query =
      noClasses()
          .that().resideOutsideOfPackage("com.holliverse.admin.query..")
          .should().dependOnClassesThat().resideInAnyPackage("org.jooq..");
}
```

  **→ Schema 변경은 Flyway로만 하고, JPA는 ddl-auto = validate로 고정해서 엔티티 수정**

```plain
spring:
  flyway:
    enabled: true
    baseline-on-migrate: true
  jpa:
    hibernate:
      ddl-auto: validate
```

### 4) Bean 강제: customer와 admin의 Bean이 섞이지 않도록 강제

코드베이스가 하나이고 customer/admin 런타임이 같은 jar를 사용하면, 가장 위험한 실수는 “원하지 않는 빈이 함께 로딩되는 것”이다. 예를 들어 admin 전용 Adapter가 customer 런타임에서도 자동 스캔되어 등록되면, 구조적으로 분리했다고 해도 실제 런타임에서는 섞이게 된다.

그래서, 아래와 같은 규칙으로 강제한다.

- **Infra/adpapters** 영역은 **@Component**로 자동 등록시키지 않는다.
- 인프라 구성은 **@Configuration + @Bean**으로만 등록한다.
- 런타임별로 어떤 @Configuration을 쓸 지 Import 목록을 명시한다.
- Import 목록은 Enum으로 관리한다.

```plain
public enum CustomerInfraImports {
  CORE_PERSISTENCE(CorePersistenceConfig.class),
  CUSTOMER_WEB(CustomerWebConfig.class),
  REDIS(RedisConfig.class);

  private final Class<?> configClass;

  CustomerInfraImports(Class<?> configClass) {
    this.configClass = configClass;
  }

  public Class<?> configClass() {
    return configClass;
  }
}
```

1. customer 런타임에서 로딩 가능한 인프라 구성의 범위를 “화이트리스트”로 만든다.
2. 새로운 인프라 구성(Config)을 추가하더라도, 이 enum에 올리지 않으면 customer 런타임에 들어가지 않는다.
3. 결과적으로 “자동 스캔으로 인해 뜨는 빈”을 구조적으로 제거한다.

- Enum은 단순히 Bean 목록이다. 목록을 실제로 Spring 컨텍스트에 적용하려면 \*\*`ImportSelector`\*\*가 필요

```plain
public class CustomerImportsSelector implements ImportSelector {
  @Override
  public String[] selectImports(AnnotationMetadata importingClassMetadata) {
    return Arrays.stream(CustomerInfraImports.values())
        .map(i -> i.configClass().getName())
        .toArray(String[]::new);
  }
}

public enum CustomerInfraImports {
  CORE_PERSISTENCE(CorePersistenceConfig.class),
  CUSTOMER_WEB(CustomerWebConfig.class),
  REDIS(RedisConfig.class);

  private final Class<?> configClass;
  CustomerInfraImports(Class<?> configClass) { this.configClass = configClass; }
  public Class<?> configClass() { return configClass; }
}
```

1. **CustomerInfraImports**에 등록된 **configClass**들을 전부 Import 대상으로 반환.
2. Spring은 반환된 클래스들을 **@Import(…)** 로딩
3. customer Runtime에서 뜨는 인프라 **Bean**은 enum에 적힌 구성으로만 한정된다.

### 5) DataBase 강제: RDS 1개 + 권한/스키마 경계 강제

- MVP에서 DB를 여러 개로 쪼개면 비용이 급증한다. RDS는 1개로 두되, 논리 경계를 강제했다.
        - **Schema** : `core`, `analysis`, `reco`
        - **Role**: `app_customer`, `app_admin`, `worker_analysis`, `worker_reco`

| 주체(프로세스/계정) | core 스키마 | analysis 스키마 | reco 스키마 | 핵심 의도 |
| --- | --- | --- | --- | --- |
| **core-server (customer role)** `app_customer` | Read/Write(본인 데이터 범위는 앱 로직으로 제한) | No access | Read(본인 추천만 조회) | 고객 기능이 추천 결과를 조회만 하게 만든다 |
| **core-server (admin role)** `app_admin` | 제한적 Write(회원 제재/정책 변경 등 최소) + Read | Read(대시보드 조회용) | Read(추천 결과/집계 조회) | admin은 운영에 필요한 최소 write만 허용 |
| **worker-analysis** `worker_analysis` | Read-only | Write(분석 결과 테이블) | No access(또는 필요 시 Read-only) | core 데이터는 읽기만, 분석 결과만 기록 |
| **worker-reco** `worker_reco` | No access(원칙) 또는 최소 Read-only | Read(분석 결과 읽기) | Write(추천 결과 테이블) | reco는 분석 결과로 추천을 계산하고 reco에만 기록 |

### **6) Gemini Code Review - 마지막 점검**

1. 위의 1\~4 강제성을 모두 지키되 코드 레벨에서 실수가 발생할 수 있기 때문에 아래와 같은 영문 프롬프트로 코드리뷰를 받고 모든 코드 리뷰에 대해서 resolved를 진행해야 PR이 병합할 수 있도록 github ruleset 설정을 한다.

- **Prompt**

```plain
You are the lead architect and reviewer for our project. Review this PR strictly against the rules below.
The goal is to prevent architecture/permission boundary violations even with beginner developers.

[Project context]
- Holliverse API server (single repo) + Worker (separate repo)
- API server uses layered architecture.
- Customer/Admin runtimes are separated (two ECS services) but share the same codebase/artifact.
- Feature/config separation is done via Spring profiles:
    - customer runtime: SPRING_PROFILES_ACTIVE=customer
    - admin runtime: SPRING_PROFILES_ACTIVE=admin
- Admin API must be reachable only via admin-api.holliverse.site (host guard + WAF IP allowlist).

[Global rules]
1. Request/Response DTOs should be implemented as Java record by default.
	Allowed exceptions (use class) only when:
    - Many optional fields and backward-compatible evolution is expected
    - Builder is needed due to complex construction
    - Heavy validation is required (e.g., Sign-up)
    - Inheritance/polymorphism is required (rare)
  2. Customer code must not import/depended on admin code. (CI will enforce, still verify in review)
3. Admin code must not import/depend on customer code. (CI will enforce, still verify in review)
4. Shared util-like code goes only to shared or shared.domain (avoid dumping ground).

[Layering & call direction]
- Allowed call flow only:
	Controller (web) -> UseCase (application/service) -> Repository or Port
- Ports are interfaces in domain; Adapters are implementations in infra only.
- “infra” means non-POJO external dependencies (SMS/Email/S3/Redis/jOOQ/OpenSearch etc.), not AWS infrastructure.

[Web layer rules]
1. Web layer responsibilities:
- Receive HTTP requests as DTOs
- Call UseCase
- Return response DTOs
- DTO conversion must be done via Mapper/Assembler in the web layer (next to controller)
  2. Web layer must NOT:
- Use @Transactional
- Access repositories directly
- Call external APIs
  3. Profile separation is mandatory:
- Admin endpoints must be in admin profile only:
	@Profile("admin") + @RequestMapping("/api/admin")
- Customer endpoints must be in customer profile only:
	@Profile("customer") + @RequestMapping("/api/customer")
- Any admin endpoint accidentally loaded in customer runtime is a blocker.

[Mapper/Assembler rules (web layer)]
1. Mapper/Assembler is located in the web layer and only does conversions:
- Entity/Domain -> Response DTO
- Request DTO -> Command (optional)
  2. Mapper must NOT:
- Call repositories
- Trigger deep lazy traversal carelessly (avoid accidental N+1 / LazyInitialization issues)
  3.Mapping must be explicit and safe. If mapping requires nested fields, ensure the UseCase loads required data properly (fetch join/projection) rather than relying on lazy loads.

[Application layer (UseCase) rules]
1. UseCase responsibilities:
- Define transaction boundaries
- Load/modify/persist entities
- Enforce domain rules via Policy

Call external integrations via Ports only
  2. Transaction rules:
- @Transactional is allowed only at UseCase layer (NOT repositories/adapters/controllers)
- Reads should prefer @Transactional(readOnly=true)
- No external calls (SMS/S3 upload/etc.) inside transactions; split into post-commit side effects

[Domain layer rules]
- Domain contains business rules (Model/Policy/Port).
- Domain must not depend on web DTOs or repositories.
- Keep Spring dependency minimal.

[Infra layer rules]
1. Infra contains adapter implementations for external dependencies:
e.g., SmsSenderAdapter, S3UploaderAdapter, RedisCacheAdapter, OpenSearchRetrievalAdapter
2. UseCase depends on Port interfaces only.
3. Infra implementations must NOT be registered via @Component.
    - They must be registered only via @Configuration + @Bean.
4. Infra beans must be enabled only via a RuntimeModule ENUM-based mechanism:
    - If an adapter exists but is not listed in RuntimeModule, it must not be reachable.
    - Runtime configs (customer/admin) must explicitly enable required modules.

[Query rules]
Customer queries:
- Prefer JPA for customer flows.
- For complex reads, projection/fetch join is allowed; Querydsl is recommended.

Admin queries:
- Heavy analytics must use jOOQ only (Querydsl is NOT allowed for heavy admin analytics).
- jOOQ usage is allowed only inside admin.query.dao package.
- Admin analytics must avoid scanning/group-by/join-heavy queries directly on core OLTP tables.
- Prefer pre-aggregated read-model tables in analysis schema.

[Checklist-based enforcement]
During review, explicitly verify:
- Is this code customer or admin? Does it stay within its boundary?
- Any controller directly calling repository? (BLOCKER)
- Any @Transactional outside UseCase? (BLOCKER)
- Any DTO not implemented as record without justification? (HIGH PRIORITY)
- Any jOOQ imported outside admin.query.dao? (BLOCKER)
- Any customer code importing jOOQ? (BLOCKER)
- Any infra adapter annotated with @Component? (BLOCKER)
- Any infra adapter not wired through @Configuration + @Bean and RuntimeModule enum? (BLOCKER)
- Any admin endpoint accessible/loaded in customer profile? (BLOCKER)
- Any heavy admin query hitting core OLTP tables instead of analysis read-model? (HIGH PRIORITY)
- Any suspicious lazy traversal / N+1 risk introduced by mappers or web layer? (HIGH PRIORITY)

[Output format]
  1. Summary (<= 3 lines)
2. Blocking issues (must fix): list each with:
    - Rule violated
  - Why it matters (impact)
  - How to fix (with code-level suggestion)
  3. High priority improvements
4. Medium/Low suggestions
5. Architecture checklist (PASS/FAIL per rule group)
6. Next PR watchlist (top 3 recurring risks to watch)

Review only based on the given diff. If you infer anything, label it explicitly as an assumption.

[Extra focus: transaction boundaries & performance]
  - Identify every @Transactional boundary introduced/modified.
- For each transaction, list included operations (DB reads/writes, loops, external calls).
- Flag anything that could cause long transactions, lock contention, pool exhaustion.
- Verify read paths use readOnly and do not accidentally trigger writes (dirty checking).
- Flag N+1, risky fetch joins, offset pagination scans; propose safer alternatives.
```

- PR 검증 ruleset![](/docs/assets/images/uploads/%E3%85%8C%E3%85%8A%E3%85%8D%E3%85%8C%E3%85%8A%E3%85%8D%E3%85%8C%E3%85%8A%E3%85%8D.png)

### 3. 전체 아키텍처 모습

![](/docs/assets/images/uploads/Billing%20Cycle%20Gate%20Flow-2026-02-12-013948.svg)

| **서비스 이름** | **유형** | **역할/설명** | **DB 접근 권한 (RDS)** |
| --- | --- | --- | --- |
| **customer/admin API(core-server)** | **Online** | **항상 가동** |  |
| 고객 및 관리자용 API 서빙 | • **R/W**: Core 스키마 |  |  |
| • **Read**: Reco 스키마 |  |  |  |
| **worker-analysis** | Worker | **집계/피처 생성** |  |
| 데이터 분석 및 가공 작업 | • **Read**: Core 스키마 |  |  |
| • **Write**: Analysis 스키마 |  |  |  |
| **worker-reco** | Worker | **추천 결과 생성** |  |
| 분석된 데이터를 기반으로 추천 산출 | • **Read**: Analysis 스키마 |  |  |
| • **Write**: Reco 스키마 |  |  |  |
| **worker-notification** | Worker | **알림 발송** |  |
| 큐에서 메시지를 소비하여 외부 발송 | (DB 직접 접근 없음) |  |  |

## 3. 관측/확장 시나리오

| 관측 지표(무조건 모니터링) | 위험 신호(지속 조건) | 의미 | 대응 리팩토링/승격 |
| --- | --- | --- | --- |
| **Customer API p95/p99 latency** | p95가 평소 대비 2배 이상, 또는 SLO 초과가 10분 이상 반복 | 고객 트래픽/DB 병목이 발생했다 | 1) customer-api 오토스케일 상향 2) 캐시 도입/확대 3) 느린 쿼리 튜닝(인덱스/쿼리) |
| **Admin API p95 latency** | admin 대시보드가 자주 타임아웃(5xx) | admin 통계 쿼리가 DB를 흔든다 | 1) 통계는 “사전 집계 테이블”로 전환 2) admin read 캐시(Redis) 3) admin read replica 고려 |
| **Hikari pool usage / wait time** | active ≈ max, connection acquire timeout 발생 | DB 커넥션 풀이 병목이다(가장 흔한 다운) | 1) 서비스별 풀 크기 조정 2) 트랜잭션 길이/쿼리 최적화 3) read/write 분리(리드 레플리카) |
| **DB CPU/IOPS/Read latency** | CPU 70%↑가 15분 이상 지속, read latency 상승 | DB가 전체 병목이다(공유의 한계) | 1) read-heavy는 replica로 분리 2) 분석/추천 결과 테이블 분리 DB 고려 3) 파티셔닝/인덱스/쿼리 리라이트 |
| **DB Lock/Deadlock** | lock wait 증가, deadlock 발생 | 쓰기 경합/트랜잭션 설계 문제 | 1) 트랜잭션 범위 축소 2) 인덱스/쿼리 수정 3) write 분리 또는 이벤트 기반 비동기화 |
| **ALB 5xx / target unhealthy** | unhealthy 반복, 5xx 급증 | 런타임 자원 고갈(스레드/메모리) | 1) 태스크 CPU/메모리 상향 2) Tomcat thread/queue 튜닝 3) 폭주 방어(WAF rate limit 강화) |
| **Admin이 고객 폭주에 같이 죽음** | 고객 폭주 시 admin도 동일하게 5xx/timeout | 런타임/DB 격리가 아직 부족 | 1) admin WAF allowlist 강화 2) admin read replica 3) admin-api 완전 분리 배포(독립 릴리즈) |
| **추천/검색 응답 p95** | 추천 호출만 유독 느려짐, 검색/벡터 부하 | reco가 핫스팟이 됐다 | 1) reco-service 분리 배포 승격 2) OpenSearch/Vector 도입 3) retrieval/ranking 캐시 |
| **worker 배치 수행 시간** | 배치 시간이 매번 늘거나, 운영 시간과 겹침 | 분석/추천이 온라인에 영향을 주기 시작 | 1) 배치 시간대 분리/스케줄 관리 2) 배치 DB read 전용 계정 강화 3) 배치 결과를 “쓰기 전용 테이블”로만 적재 |
