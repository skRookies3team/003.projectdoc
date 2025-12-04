# [ADR-0003] 다이어리 조회 로직 분리를 위한 논리적 CQRS 패턴 적용

- **상태**: 승인됨 (Accepted) [2025-12-03]
- **결정자**: 김지은
- **관련 문서**: Issue #7, PR #8

## 1. 배경 및 문제 (Context)

현재 `DiaryService`는 일기의 생성, 수정, 삭제(CRUD) 로직을 담당하고 있습니다. 최근 캘린더 뷰(날짜별 조회)와 AI 아카이브(조건부 필터링) 기능이 추가되면서 다음과 같은 문제에 직면했습니다.

1. **서비스 비대화**: 단일 서비스 클래스에 비즈니스 로직(검증, 상태 변경)과 단순 조회 로직이 혼재되어 가독성이 떨어짐.
2. **관심사 분리 부재**: 트랜잭션 성격이 다른 쓰기 작업(Command)과 읽기 작업(Query)이 묶여 있어, 조회 성능 최적화 시 쓰기 로직에 영향을 줄 우려가 있음.
3. **확장성 제한**: 향후 조회 전용 DB(Replica)나 캐시 도입 시 리팩토링 비용이 증가할 구조임.

따라서, 기존 단일 테이블 구조를 유지하면서도 코드 레벨에서 조회와 명령의 책임을 명확히 분리할 필요가 있습니다.

## 2. 결정 사항 (Decision)

우리는 **논리적 CQRS(Command and Query Responsibility Segregation) 패턴**을 도입하기로 결정했습니다.

- **구조 분리**: 조회 전용 계층(`DiaryQueryController`, `DiaryQueryService`, `DiaryQueryRepository`)을 신설하여 기존 CRUD(`DiaryService`, `DiaryRepository`)와 분리합니다.
- **단일 데이터 소스 유지**: 물리적인 DB 분리(Physical CQRS) 단계까지는 가지 않고, 기존 `DIARIES` 테이블(Entity)을 공유하는 형태를 유지합니다.
- **날짜 조회 표준화**: 캘린더 조회를 위한 날짜 범위 쿼리는 `LocalDate`를 입력받아 `atStartOfDay()`와 `atTime(LocalTime.MAX)`를 사용하여 `BETWEEN` 검색을 수행하는 방식을 표준으로 채택합니다.

## 3. 결정 근거 (Decision Drivers)

- **유지보수성 향상**: SRP(단일 책임 원칙)를 준수하여 코드의 응집도를 높이고 결합도를 낮춤.
- **확장 용이성**: 추후 조회 트래픽이 급증할 경우, Query Service 구현체만 변경하여 Read DB나 Elasticsearch 등으로 전환하기 용이함.
- **개발 생산성**: 복잡한 조회 쿼리 작성 시 엔티티의 비즈니스 로직(Dirty Checking 등)을 신경 쓰지 않고 DTO 변환에만 집중 가능.

## 4. 고려했던 대안들 (Considered Options)

### 대안 1: 기존 DiaryService 확장 (Monolithic Service)

조회 메서드를 기존 서비스와 레포지토리에 계속 추가하는 방식.

### 대안 2: 논리적 CQRS (Selected)

DB는 공유하되, 애플리케이션 코드 레벨에서 Command와 Query의 패키지/클래스를 분리하는 방식.

### 대안 3: 물리적 CQRS (Separate Database)

쓰기 DB(Master)와 읽기 DB(Slave)를 분리하고, 데이터 동기화 파이프라인을 구축하는 방식.

## 5. 장단점 분석 (Pros and Cons)

### [대안 2: 논리적 CQRS] (채택됨)

- **장점**:
    - 아키텍처 복잡도를 크게 높이지 않으면서 코드 복잡도를 효과적으로 낮출 수 있음.
    - 데이터 동기화(Replication lag) 이슈가 없어 데이터 정합성이 즉시 보장됨.
    - 기존 JPA Entity를 재사용하여 개발 속도가 빠름.
- **단점**:
    - 클래스 파일 수가 늘어남 (Controller, Service, Repository가 2배가 됨).
    - 물리적 성능 분리는 되지 않아, 조회 트래픽이 쓰기 DB 부하에 영향을 줌.

### [대안 1: 기존 DiaryService 확장]

- **장점**: 구현이 가장 빠르고 파일 수가 적음.
- **단점**: 서비스 클래스가 거대해짐(God Class). 조회 로직 변경이 핵심 비즈니스 로직에 사이드 이펙트를 줄 위험이 커짐.

### [대안 3: 물리적 CQRS]

- **장점**: 읽기/쓰기 성능을 독립적으로 스케일링 가능.
- **단점**: 인프라 비용 및 관리 포인트 증가. 데이터 최종 일관성(Eventual Consistency) 문제 해결을 위한 추가 구현 필요. 현재 트래픽 규모에는 오버엔지니어링임.

## 6. 결과 및 영향 (Consequences)

- **긍정적**:
    - `DiaryService`는 도메인 로직에만 집중하게 되어 코드가 간결해짐.
    - 조회 API(`calendar`, `ai-archive`)의 요구사항 변경 시 `DiaryQueryService`만 수정하면 되므로 변경 범위가 격리됨.
    - 타임존 이슈를 고려한 명확한 날짜 범위 검색 로직이 확립됨.
- **부정적**:
    - 프로젝트 내 클래스 파일 개수가 증가함.
    - 신규 입사자에게 Command와 Query를 나누는 기준(단순 ID 조회는 어디에 둘 것인가 등)에 대한 가이드가 필요함.