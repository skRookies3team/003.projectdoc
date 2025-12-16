# [ADR-0008] 챗봇 응답 지연 해결을 위한 SSE(Server-Sent Events) 스트리밍 도입

* **상태**: 승인됨 (Accepted) [2025-12-16]
* **결정자**: 양승준 (Backend/AI Lead)
* **관련 문서**: AWS Bedrock Runtime API Reference

## 1. 배경 및 문제 (Context)

현재 우리 프로젝트의 핵심 기능인 **'페르소나 챗봇'**과 **'일반 챗봇'**은 AWS Bedrock(Claude 3)을 기반으로 동작한다.
기존의 **Blocking HTTP (Request-Response)** 방식을 사용할 경우 다음과 같은 문제가 발생한다.

1.  **높은 체감 지연(High Latency)**: LLM이 문장을 모두 완성할 때까지 서버가 응답을 보내지 않으므로, 사용자는 짧게는 3초에서 길게는 10초 이상 '로딩 스피너'만 보며 대기해야 한다.
2.  **사용자 경험(UX) 저하**: 사용자는 서비스가 멈췄거나 오류가 발생했다고 오인하여 이탈할 가능성이 높다.
3.  **타임아웃 위험**: 답변이 매우 길어질 경우, Gateway나 Load Balancer의 HTTP Timeout 설정(기본 30~60초)에 걸려 연결이 끊길 위험이 있다.

따라서, ChatGPT나 Claude와 같은 현대적인 LLM 서비스처럼 **실시간으로 생성되는 텍스트를 사용자에게 즉시 전달**하는 기술적 대안이 필요하다.

## 2. 결정 사항 (Decision)

우리는 챗봇의 응답 처리 방식을 **Server-Sent Events (SSE)** 기반의 **스트리밍(Streaming) 아키텍처**로 전환하기로 결정했다.

* **통신 프로토콜**: HTTP 기반의 단방향 스트리밍 표준인 **SSE (MIME Type: `text/event-stream`)**를 채택한다.
* **백엔드 구현**: Spring Boot의 `SseEmitter`와 AWS SDK의 `invokeModelWithResponseStream`을 결합하여 비동기로 데이터를 전송한다.
* **프론트엔드 구현**: `EventSource` API 또는 `fetch`의 스트림 리더를 사용하여 들어오는 텍스트 조각(Chunk)을 실시간으로 화면에 렌더링한다.

## 3. 결정 근거 (Decision Drivers)

* **체감 속도 개선**: 첫 번째 토큰(글자)이 생성되는 즉시 사용자에게 보여줌으로써, 응답 대기 시간을 0.5초 이내로 단축시키는 효과를 준다.
* **구현의 용이성**: WebSocket과 달리 별도의 프로토콜 업그레이드(Handshake)가 필요 없고, 표준 HTTP를 사용하므로 기존 API Gateway 및 방화벽 설정과 호환성이 좋다.
* **리소스 효율성**: 클라이언트가 서버로 데이터를 보낼 필요가 없는 '구독(Subscribe)' 형태의 통신이므로, 양방향 통신인 WebSocket보다 서버 리소스를 적게 소모한다.

## 4. 고려했던 대안들 (Considered Options)

### [대안 1] Blocking HTTP (기존 방식)
* **설명**: 요청을 보내고 AI가 답변을 완료할 때까지 대기.
* **기각 사유**: 생성형 AI의 특성상 응답 시간이 불규칙하고 길어 UX에 치명적임.

### [대안 2] WebSocket
* **설명**: TCP 기반의 양방향 전이중 통신 채널.
* **기각 사유**: 챗봇 시나리오상 클라이언트가 중간에 서버로 데이터를 보낼 일이 거의 없으므로(질문 1회 -> 응답 스트림), 양방향 통신은 불필요한 오버엔지니어링임. 또한 Stateful한 연결 관리가 필요하여 서버 확장에 불리함.

### [대안 3] Short Polling
* **설명**: 클라이언트가 1초마다 "답변 다 됐어?"라고 서버에 요청.
* **기각 사유**: 불필요한 네트워크 트래픽을 유발하고 서버 부하를 가중시킴. 실시간성도 떨어짐.

## 5. 장단점 분석 (Pros and Cons)

### [Server-Sent Events (SSE)] (채택됨)
* **장점**:
    * **표준 HTTP 사용**: 포트 개방이나 프로토콜 변경 없이 기존 웹 인프라(LB, Proxy) 위에서 잘 동작함.
    * **자동 재연결**: 브라우저의 `EventSource`는 연결이 끊어지면 자동으로 재연결을 시도함.
    * **간결한 구현**: Spring WebMVC에서 `SseEmitter`를 통해 직관적으로 구현 가능.
* **단점**:
    * **단방향 통신**: 클라이언트에서 서버로 데이터를 보낼 때는 별도의 HTTP POST 요청을 써야 함. (현재 아키텍처에서는 문제되지 않음)
    * **인프라 타임아웃**: Nginx나 API Gateway 설정에서 SSE 연결에 대한 `read_timeout`이나 `proxy_read_timeout`을 길게(예: 300초) 설정해 주어야 함.

## 6. 후속 작업 (Action Items)
1.  **Backend**: `ChatController`에 SSE 엔드포인트 구현 및 `BedrockClient` 비동기 스트리밍 로직 적용.
2.  **Infra (DevOps)**: AWS API Gateway 및 Ingress Controller의 Timeout 설정을 점검하여 SSE 연결이 조기에 끊기지 않도록 설정 변경 (최소 60s 이상).
3.  **Frontend**: SSE 응답을 파싱하여 UI에 타이핑 효과(Typewriter Effect)를 주는 컴포넌트 구현.
