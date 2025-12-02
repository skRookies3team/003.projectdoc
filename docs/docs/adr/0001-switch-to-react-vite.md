# [ADR-0001] Frontend 프레임워크를 Next.js에서 React + Vite로 전환

* **상태**: 승인됨 (Accepted) [2025-12-02]
* **결정자**: 프론트엔드 리드, 프로젝트 멘토
* **관련 이슈**: #12 (S3 배포 이슈), #45 (프론트엔드 리팩토링)

## 1. 배경 및 문제 (Context)
현재 프로젝트의 프론트엔드는 `Next.js (App Router)` 기반으로 구현되어 있습니다. 그러나 배포 및 운영 과정에서 다음과 같은 문제에 직면했습니다.

1.  **S3 정적 배포 호환성 문제**:
    * 우리의 배포 타겟은 AWS S3(정적 호스팅) + CloudFront입니다.
    * Next.js는 기본적으로 Node.js 서버 런타임을 필요로 하는 기능(SSR, Image Optimization, API Routes)이 많아, S3에 단순히 빌드 파일만 올릴 경우 동적 라우팅이나 서버 관련 기능이 작동하지 않는 치명적인 문제가 발생했습니다.
    * `output: 'export'` 설정을 하더라도 `next/image` 사용 불가 등 제약 사항이 많아 개발 효율이 저하됩니다.
2.  **불필요한 서버 기능의 복잡성**:
    * 우리는 이미 별도의 Spring Boot 백엔드 서버가 존재하므로, 프론트엔드에서 BFF(Backend For Frontend)나 API Routes 같은 서버 기능이 굳이 필요하지 않습니다.
    * Next.js의 미들웨어(Middleware)와 렌더링 레이어 분리 구조는 캐시 공유가 어렵고 불필요한 중복 호출을 유발하여 최적화를 어렵게 만듭니다.
3.  **러닝 커브 및 문서 품질 이슈**:
    * Next.js의 공식 문서는 내부 동작 원리나 엣지 케이스에 대한 설명이 부족하고, 일부 가이드(예: redirect 처리)는 혼란을 줄 수 있어 팀원들의 디버깅 시간을 증가시킵니다.

## 2. 결정 사항 (Decision)
우리는 프론트엔드 기술 스택을 **Next.js**에서 **React + Vite** 기반의 **SPA(Single Page Application)**로 전면 전환하기로 결정했습니다.

## 3. 결정 근거 (Decision Drivers)
* **배포 안정성**: React SPA는 빌드 결과물이 완벽한 정적 파일(HTML/CSS/JS)이므로, S3 정적 호스팅 환경에서 100% 호환되며 배포 과정이 단순하고 명확합니다.
* **아키텍처 단순화**: '프론트엔드(View) - 백엔드(Data)'의 책임 분리가 명확해집니다. 프론트엔드는 오직 데이터 렌더링과 UI 상호작용에만 집중할 수 있습니다.
* **개발 생산성 (DX)**: Vite는 Go 기반의 초고속 빌드 도구(Esbuild)를 사용하여 Next.js(Webpack/Turbopack)보다 훨씬 빠른 개발 서버 구동 속도와 HMR(Hot Module Replacement)을 제공합니다.
* **비용 효율성**: 별도의 Node.js 서버(EC2/Vercel)를 띄울 필요 없이 S3 스토리지 비용만 발생하므로 운영 비용이 획기적으로 절감됩니다.

## 4. 고려했던 대안들 (Considered Options)
* **Next.js 유지 (Static Export)**: `next.config.js`에 `output: 'export'`를 추가하여 정적 사이트로 빌드.
    * *기각 사유*: 동적 라우팅(`[id].tsx`) 처리가 번거롭고, `next/image` 등 핵심 컴포넌트를 사용할 수 없어 Next.js를 쓰는 이점이 거의 사라짐.
* **React + Webpack (CRA)**: 전통적인 Create React App 방식.
    * *기각 사유*: 설정이 복잡하고 빌드 속도가 Vite에 비해 현저히 느림. 현재 시점에서는 레거시(Legacy) 기술로 간주됨.

## 5. 장단점 분석 (Pros and Cons)

### React + Vite (채택됨)
* **장점**:
    * S3 + CloudFront 정적 배포에 최적화되어 있음 (배포 성공률 100%).
    * Vite의 압도적인 빌드 속도로 개발 경험(DX) 향상.
    * `useClient` 지시어 같은 서버/클라이언트 경계 고민 없이, 순수한 클라이언트 로직에 집중 가능.
* **단점**:
    * 초기 로딩 속도(FCP)가 SSR(Next.js)보다 다소 느릴 수 있음 (단, 우리 프로젝트 규모에서는 미미함).
    * SEO(검색 엔진 최적화)에 약점이 있으나, 현재 서비스 특성상(로그인 기반 앱) SEO가 최우선 순위는 아님.

### Next.js (기존)
* **장점**: SSR/SSG를 통한 초기 로딩 속도 및 SEO 우수.
* **단점**:
    * AWS S3 단독 배포 시 동적 기능(API, Image) 사용 불가.
    * 서버 런타임 의존성으로 인해 아키텍처 복잡도 증가.
    * 문서 품질 및 예기치 못한 동작(캐싱 등)으로 인한 디버깅 비용 발생.

## 6. 결과 및 영향 (Consequences)
* **코드 마이그레이션**: 기존 `app/` 디렉토리 구조를 `src/pages/` 및 `src/components/` 구조로 개편해야 합니다. (`useClient` 제거, 라우팅을 `react-router-dom`으로 변경)
* **배포 파이프라인**: GitHub Actions 스크립트를 수정하여 `npm run build` 후 생성된 `dist/` 폴더를 S3에 업로드하도록 변경해야 합니다.
* **백엔드 연동**: 기존 Next.js API Routes를 제거하고, 프론트엔드에서 직접 Spring Boot 서버 API를 호출하도록 `axios` 설정을 변경합니다. (CORS 설정 필요)
