# [ADR-0004] 확장 가능한 프론트엔드 아키텍처(FASD) 도입

* **상태**: 승인됨 (Accepted) [2025-12-08]
* **결정자**: 프론트엔드 리드, 아키텍트
* **관련 문서**: [Naver D2 - 실용적인 프론트엔드 아키텍처](https://d2.naver.com/helloworld/7030870), ADR-0001 (React 전환)

## 1. 배경 및 문제 (Context)
기존 프로젝트 구조는 파일의 종류(`components`, `hooks`, `pages`)별로 폴더를 나누는 **수평적 구조**였습니다.
MSA(Microservices Architecture) 환경에서 기능이 복잡해짐에 따라 다음과 같은 문제가 발생했습니다.

1.  **낮은 응집도 (Low Cohesion):** '헬스케어' 기능을 수정하려면 `api`, `components`, `hooks` 등 여러 폴더를 오가며 파일을 찾아야 함.
2.  **불명확한 경계:** 특정 도메인(예: 쇼핑)에서만 쓰이는 컴포넌트와 전역 공통 컴포넌트가 `components` 폴더에 섞여 있어 재사용성 판단이 어려움.
3.  **협업 충돌:** 6명의 팀원이 서로 다른 기능을 개발함에도 공통 폴더를 건드려 Git Merge Conflict가 빈번함.

## 2. 결정 사항 (Decision)
우리는 프론트엔드 프로젝트에 **FASD (Feature-App-Shared-Design)** 패턴을 변형하여 적용하기로 결정했습니다.
모든 코드는 **'기능(Feature)'**을 기준으로 수직적으로 격리하며, 백엔드 마이크로서비스 구조와 프론트엔드 모듈 구조를 일치시킵니다.

## 3. 상세 구조 정의 (Implementation)

### 3.1. Layer 구분
* **Feature (`src/features`):** 비즈니스 로직의 핵심 단위입니다.
    * 원칙: 각 Feature 폴더는 독립적인 '작은 앱'처럼 동작하며, 서로 직접 참조를 지양합니다.
    * 구성: `api`, `model`, `ui`, `hooks`를 내부에 가집니다.
    * 매핑: `features/healthcare`는 백엔드의 `Healthcare Service`와 1:1로 대응됩니다.
* **App (`src/app`):** 애플리케이션의 진입점 및 전역 설정입니다.
    * 구성: `router`, `providers`, `global-styles`.
* **Shared (`src/shared`):** 도메인 지식이 없는 순수 재사용 요소입니다.
    * 구성: `shared/ui` (Design System), `shared/lib` (Axios, Utility).
* **Pages (`src/pages`):** 실제 라우팅되는 페이지 단위입니다.
    * 역할: `feature`들을 레고처럼 조립(Composition)하는 껍데기 역할만 수행합니다. 비즈니스 로직을 포함하지 않습니다.

## 4. 결정 근거 (Decision Drivers)
* **Co-location (관련 코드 집약):** 관련 파일을 한 폴더에 모아두어 개발 시 컨텍스트 스위칭 비용을 최소화함.
* **MSA Alignment:** 백엔드 서비스 경계와 프론트엔드 기능 경계를 일치시켜, 특정 서비스(예: 헬스케어) 변경 시 영향 범위를 `features/healthcare` 내부로 한정할 수 있음.
* **확장성 (Scalability):** 새로운 기능(예: 펫 장례)이 추가되면 `features` 폴더 하나만 추가하면 되므로 기존 코드에 영향을 주지 않음.

## 5. 대안 분석 (Alternatives)

* **Atomic Design Pattern:**
    * *단점:* '분자', '유기체' 등의 구분이 모호하여 팀원 간 분류 기준 논쟁으로 시간이 소요됨. 비즈니스 로직 격리보다는 UI 재사용성에 치중됨.
* **Layered Architecture (기존 방식):**
    * *단점:* 프로젝트 규모가 커질수록 파일 탐색이 어렵고, 기능 간 결합도가 높아짐.

## 6. 결과 및 영향 (Consequences)
* **Refactoring:** 기존 `components`에 있던 파일들을 `shared/ui`와 `features/**/components`로 분류하여 이동해야 합니다.
* **Import Path:** 상대 경로(`../../`) 지옥을 피하기 위해 `tsconfig.json`에 Path Alias(`@features/*`, `@shared/*`) 설정이 필수적입니다.
* **Team Rule:** "이 컴포넌트가 다른 기능에서도 쓰이나?"를 항상 고민해야 하며, 재사용된다면 `shared`로, 아니라면 `feature`로 보내는 의사결정 습관이 필요합니다.
