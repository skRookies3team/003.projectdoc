# [ADR-0007] 프론트엔드 인증 지속성 전략 및 네트워크 계층 표준화

* **상태:** 승인됨 (Accepted) [2025-12-10]
* **결정자:** 3조 개발팀 (Frontend & Backend Lead)
* **관련 문서:** `0002-frontend-fasd-architecture.md`, `[PET-101] 초기 환경 세팅 이슈`

## 1. 배경 및 문제 (Context)
현재 React 기반 프론트엔드와 Spring Boot MSA 백엔드를 연동하는 과정에서, 인증 상태 관리와 API 통신 방식에 대한 다음과 같은 문제들이 식별되었다.

1.  **인증 상태의 휘발성 (UX 저하):**
    * `Context API`의 상태(State)는 메모리에 저장되므로, 사용자가 페이지를 **새로고침(F5)** 할 경우 로그인이 풀려버린다. 이는 사용자 경험에 치명적이다.
2.  **보안과 편의성의 충돌:**
    * 보안 권고 사항(토큰의 메모리 저장)을 따르면 UX가 저하되고, UX를 위해 브라우저 저장소를 쓰면 XSS(Cross-Site Scripting) 보안 취약점이 발생한다.
3.  **네트워크 코드의 파편화:**
    * 각 기능(Feature)별로 `axios`를 직접 호출하고 있으며, 인증 헤더(`Authorization`)를 수동으로 설정하는 중복 코드가 많아 유지보수가 어렵다.
4.  **순환 의존성(Circular Dependency):**
    * 일반 TS 파일인 `Axios Interceptor`에서 React Component인 `AuthContext`의 상태(로그아웃 함수 등)를 직접 호출할 수 없는 구조적 제약이 있다.

## 2. 결정 사항 (Decision)
우리는 **"LocalStorage 기반의 이중 토큰 전략(Dual Token Strategy)"**과 **"의존성 주입(DI)을 활용한 네트워크 모듈화"**를 채택한다.

### A. 인증 지속성 전략 (Persistence Strategy)
* **저장소 정책:** Access Token과 Refresh Token을 **`localStorage`**에 저장하여 새로고침 시에도 로그인 상태를 유지한다.
* **보안 보완책 (Short-lived Token):**
    * Access Token의 유효 기간을 **30분 이내**로 짧게 설정하여 탈취 시 피해를 최소화한다.
    * 만료 시 사용자 개입 없이 토큰을 자동 갱신하는 **Silent Refresh** 메커니즘을 구현한다.
* **상태 동기화:** 앱 초기화 시(`AuthProvider` 마운트) `localStorage`를 확인하여 로그인 상태를 즉시 복구(Hydration)한다.

### B. 네트워크 계층 아키텍처 (Network Layer)
* **단일 접점 (Single Source of Truth):** 모든 HTTP 요청은 `src/shared/api/http-client.ts`에 정의된 `httpClient` 모듈을 통해서만 수행한다.
* **Interceptor 패턴 적용:**
    * **Request:** 저장소에서 토큰을 꺼내 `Authorization: Bearer <token>` 헤더를 자동 주입한다.
    * **Response:** `401 Unauthorized` 에러 발생 시, 요청을 일시 중단(Queueing)하고 Refresh Token으로 재발급을 시도한 뒤, 실패했던 요청을 재실행(Retry)한다.
* **의존성 주입(DI):** `http-client.ts`와 `AuthContext` 간의 순환 참조를 막기 위해, `setTokenGetter`, `setTokenRemover`와 같은 함수 주입 방식을 사용한다.

## 3. 결정 근거 (Decision Drivers)
* **사용자 경험(UX) 최우선:** 웹 서비스에서 '새로고침 시 로그아웃'은 허용될 수 없는 사용자 경험이다. 보안 위험을 기술적 장치(자동 갱신)로 완화하면서 편의성을 제공하는 것이 현실적인 대안이다.
* **구현 효율성:** `HttpOnly Cookie` 방식은 백엔드 및 인프라(Proxy) 설정 비용이 높고, 모바일 앱 확장 시 처리가 복잡하다. 반면 LocalStorage 방식은 구현이 빠르고 직관적이다.
* **유지보수성:** API 호출부에서 인증 로직을 제거(`Separation of Concerns`)함으로써, 비즈니스 로직 개발자는 데이터 처리에만 집중할 수 있다.

## 4. 고려했던 대안들 (Considered Options)

| 대안 | 내용 | 장점 | 단점 |
| :--- | :--- | :--- | :--- |
| **In-Memory 저장** | 변수에만 저장 | 보안 최상 (XSS 안전) | 새로고침 시 로그아웃 (UX 최악) |
| **BFF (Proxy)** | 별도 서버에서 세션 관리 | 보안/UX 모두 우수 | 인프라 복잡도 증가 (Over-engineering) |
| **LocalStorage** | 브라우저 저장소 사용 | **구현 용이, UX 우수** | XSS 취약 (→ 자동 갱신으로 보완) |

## 5. 구현 상세 가이드 (Implementation)

### Frontend (`src/shared/api/http-client.ts`)
```typescript
// 의존성 주입 패턴 적용
let getTokenFunction: (() => string | null) | null = null;
export const setTokenGetter = (fn: () => string | null) => { getTokenFunction = fn; };

// Interceptor: 401 감지 및 재시도 로직
axios.interceptors.response.use(
  (res) => res,
  async (err) => {
    if (err.response?.status === 401 && !err.config._retry) {
      // 1. 요청 큐(Queue)에 저장
      // 2. Refresh Token으로 재발급 요청 (/auth/reissue)
      // 3. 성공 시 큐에 있는 요청 재실행, 실패 시 로그아웃
    }
    return Promise.reject(err);
  }
);
