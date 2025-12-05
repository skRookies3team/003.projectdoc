# [ADR-0005] 이미지 저장소 단일화 및 외부 전송 책임 일원화

- **상태**: 승인됨 (Accepted) [2025-12-05]
- **결정자**: 김지은
- **관련 문서**: Issue #11, PR #12

## 1. 배경 및 문제 (Context)

기존 설계는 일기 이미지(`DiaryImage`)와 내부 보관함(`PhotoArchive`)을 **두 개의 독립적인 테이블**로 관리했습니다. 이는 다음과 같은 문제를 야기했습니다.

1. **데이터 중복 및 관리 비효율**: 내부 보관함(`PhotoArchive`)은 사실상 외부 스토리지 서비스에 저장될 데이터의 **메타데이터 미러링** 역할을 수행하므로, 내부 DB에 별도의 테이블을 유지하는 것은 불필요한 관리 비용과 리소스 낭비를 초래함.
2. **서비스 결합도**: `DiaryServiceImpl`이 내부 보관함 저장을 위해 `PhotoArchiveService`를 의존해야 했으므로 결합도가 발생함.
3. **복잡한 로직**: 동일한 이미지 URL을 두 테이블에서 관리하는 것이 복잡했으며, 조회 API 설계 시 혼란을 야기함.

이러한 문제를 해결하고 아키텍처를 간결하게 만들기 위해 데이터 모델 단일화가 필요합니다.

## 2. 결정 사항 (Decision)

우리는 **`PhotoArchive` 엔티티를 삭제**하고, **`DiaryImage` 엔티티에 `userId` 및 `source` 필드를 통합**하여 이미지 저장 및 전송 로직을 일원화하기로 결정했습니다.

- **저장소 단일화**: `DiaryImage`가 일기 기록 역할과 내부 보관함 역할을 모두 수행합니다.
- **전송 정책**: `DiaryServiceImpl`에서 이미지의 `source` 필드를 확인하여 외부 `Storage Service`로의 전송 여부를 결정합니다.
    - `source == GALLERY`: 외부 스토리지 전송 **(O)**
    - `source == ARCHIVE`: 외부 스토리지 전송 **(X)** (이미 외부에 존재한다고 간주)
- **구조 정리**: `PhotoArchiveService` 및 관련 Repository를 제거하고, `DiaryServiceImpl`에서 `StorageServiceClient`를 직접 사용하여 외부 전송 로직을 간결하게 만듭니다.

## 3. 결정 근거 (Decision Drivers)

- **모델 단순화**: 불필요한 테이블을 제거하여 데이터 모델의 복잡도를 낮춤.
- **결합도 해소**: `DiaryService`에서 내부 보관함 서비스(`PhotoArchiveService`) 의존성을 제거하여 서비스 간 결합도를 해소하고 코드를 간소화함.
- **효율성**: 로컬 DB는 오직 `Diary` 엔티티와의 관계(1:N)와 최소한의 출처 기록(Audit Log)만을 책임지도록 역할을 명확히 함.

## 4. 고려했던 대안들 (Considered Options)

### 대안 1: 기존 이중 테이블 구조 유지

`DiaryImage`와 `PhotoArchive`를 그대로 유지하고 `DiaryServiceImpl`에서 이중 저장을 계속하는 방식.

### 대안 2: 단일 테이블 통합 (Selected)

`PhotoArchive`의 필드를 `DiaryImage`에 통합하여 단일 테이블로 관리하는 방식.

### 대안 3: 이미지 메타데이터를 NoSQL로 분리

이미지 URL 및 메타데이터 관리를 RDBMS(PostgreSQL)가 아닌 MongoDB(현재 프로젝트에 MongoDB가 있으므로) 등으로 분리하는 방식.

## 5. 장단점 분석 (Pros and Cons)

### [대안 2: 단일 테이블 통합] (채택됨)

- **장점**:
    - **DB 효율성**: 불필요한 테이블 및 Repository 제거.
    - **로직 일원화**: 생성 및 삭제 시 로직이 `DiaryImage` 하나로 집중되어 가독성이 좋음.
    - **간결한 코드**: `DiaryServiceImpl`이 불필요한 Service를 의존하지 않게 됨.
- **단점**:
    - `DiaryImage` 엔티티가 `diary_id` 외에 `user_id` 등 **두 도메인의 속성**을 동시에 갖게 되어 책임이 소폭 증가함.
    - `Diary` 삭제 시 `DiaryImage`가 삭제되는 `Cascade` 설정을 유지해야 하므로, **내부 보관 기록도 삭제**됨. (단, 원본은 외부 스토리지에 남아있으므로 정책에 부합함.)

### [대안 1: 이중 테이블 구조 유지]

- **장점**: 역할 구분이 가장 명확함.
- **단점**: **관리 비용이 2배**로 들고, 내부 DB에 외부 스토리지와 중복되는 데이터를 불필요하게 저장함. 결합도가 높음.

### [대안 3: NoSQL로 분리]

- **장점**: 이미지 메타데이터처럼 유연한 데이터 관리가 필요할 경우 최적. RDBMS 부하를 줄임.
- **단점**: `Diary`와 `DiaryImage` 간의 JPA 관계 매핑 로직이 복잡해지며, MongoDB 사용 경험이 필요함.

## 6. 결과 및 영향 (Consequences)

- **긍정적**:
    - 아키텍처가 단순화되어 `DiaryService`는 오직 `StorageServiceClient`만 신경 쓰면 됨.
    - `DiaryServiceImpl`의 `createDiary` 메서드에서 **외부 전송/스킵 정책**이 `ImageSource` 기준으로 명확하게 분기 처리됨.
    - 향후 `PhotoArchive` 기능이 필요할 경우, `DiaryImage` 데이터를 기반으로 조회 API만 추가하면 됨.
- **부정적**:
    - `DiaryImage` 테이블에 `userId`를 추가함으로써 `DiaryImage`가 일기 종속성과 사용자 귀속성이라는 두 가지 책임을 갖게 됨.