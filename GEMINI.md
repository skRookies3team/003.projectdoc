# 🤖 Google Antigravity Customization Rules for Healthcare AI Chatbot Service

## 📌 목적
이 파일은 Google Antigravity Agent가 **PetLog Healthcare AI Chatbot Service** 프로젝트에서 
빠르고 정확하게 작업할 수 있도록 전체 컨텍스트를 제공합니다.

---

## 🏢 프로젝트 및 팀 정보

### 프로젝트 개요
- **프로젝트명:** PetLog (반려동물 통합 플랫폼)
- **팀명:** 이음 (Team 3, SK Shieldus Rookies 4th cohort)
- **팀 구성:** 6명
  - 1명: 배포 & AWS CI/CD 담당
  - 5명: 각 MSA별 Frontend/Backend 담당
- **내 담당:** Healthcare AI Chatbot Service (Port 8085)

### 조직 및 Repository
**Organization:** https://github.com/orgs/skRookies3team/repositories

**핵심 Repository:**
- `003.projectdoc`: ADR, 아키텍처 문서 (https://github.com/skRookies3team/003.projectdoc.git)
- `api_gateway`: Spring Cloud Gateway (Port 8000) (https://github.com/skRookies3team/api_gateway.git)
- `healthcare_AIchatbot_service_backend`: Healthcare Service (Port 8085) (https://github.com/skRookies3team/healthcare_AIchatbot_service_backend.git)
- `user_service_backend`: User Service (Port 8080) (https://github.com/skRookies3team/user_service_backend.git)
- `social_service_backend`: Social Service (Port 8083) (https://github.com/skRookies3team/social_service_backend.git)
- `record_service_backend`: Diary Service (Port 8087) (https://github.com/skRookies3team/record_service_backend.git)
- `Frontend`: Next.js 기반 Frontend (https://github.com/skRookies3team/Frontend.git)
- `manifest_repo`: Kubernetes Manifest 저장소 (https://github.com/skRookies3team/manifest_repo.git)

### 배포 및 접근 정보
- **Frontend 배포 URL:** https://d3uvkb1qxxcp2y.cloudfront.net/dashboard
- **CloudFront Distribution:** d3uvkb1qxxcp2y.cloudfront.net
- **개발 환경:** Windows 10/11, IntelliJ IDEA
- **VM Option:** `-Dspring.profiles.active=dev` (IntelliJ 설정 필수)

---

## 🛠️ 기술 스택 및 아키텍처

### MSA 아키텍처 기반
- **Backend Framework:** Spring Boot 3.x + Spring Cloud Gateway
- **Frontend Framework:** Next.js (FASD 패턴)
- **API Gateway:** Spring Cloud Gateway (Port 8000)
- **인증:** JWT (User Service 발급, Gateway에서 검증)
- **메시징:** Apache Kafka
- **벡터 DB:** Milvus
- **AI 모델:** AWS Bedrock (Claude 3.5 Haiku/Sonnet)
- **데이터베이스:** PostgreSQL
- **배포:** AWS ECS / EKS
- **CI/CD:** GitHub Actions + AWS CodeDeploy

### Healthcare Service 기술 상세
**LLM:**
- Claude 3.5 Haiku: 일반 수의사 챗봇 (빠름, 저가)
- Claude 3.5 Sonnet: 페르소나 챗봇 + 건강 분석 (정확)

**벡터 DB & 임베딩:**
- Milvus: 오픈소스 벡터 검색 DB
- Claude Embeddings: AWS Bedrock 내장

**외부 API:**
- Tripo3D: 사진 → 3D 모델 변환
- Jsoup: 청진 데이터 스크래핑

---

## 📋 코드 컨벤션 및 표준

### 1️⃣ 주석 (Comments)
**모든 클래스/메서드에 주석 필수:**
```java
/**
 * Healthcare Service용 JWT 인증 필터
 * 
 * WHY: User Service에서 발급한 JWT 토큰을 검증하여 인증된 사용자만 Healthcare Service 접근 허용
 * 
 * 기능:
 * 1. JWT 토큰 추출
 * 2. 토큰 검증
 * 3. 사용자 정보 추출 및 헤더 추가
 */
```

**주석 내용:**
- **WHY:** 왜 이 클래스/메서드가 필요한지
- **HOW:** 어떻게 동작하는지
- **기능:** 구체적인 동작 항목 (번호 매김)

### 2️⃣ 네이밍 규칙
| 타입 | 규칙 | 예시 |
|------|------|------|
| 클래스 | PascalCase | `JwtAuthenticationFilter`, `ClaudeService` |
| 메서드/변수 | camelCase | `extractToken()`, `validateToken()` |
| 상수 | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT = 3` |
| 패키지 | com.petlog.{service} | `com.petlog.gateway`, `com.petlog.healthcare` |

### 3️⃣ Git Commit 규칙

**Commit 제목 형식:**
```
[Feat/Fix/Refactor/Docs/Test/Chore] : 제목 (50자 내외, 명사형)

- 본문 상세 설명 (선택)
- 여러 항목은 리스트

Issue: #{이슈번호}
```

**Commit 태그:**
| 태그 | 용도 | 예시 |
|------|------|------|
| `Feat` | 새 기능 추가 | `[Feat] Add JWT authentication filter` |
| `Fix` | 버그 수정 | `[Fix] Fix NullPointerException in TokenProvider` |
| `Refactor` | 리팩토링 | `[Refactor] Simplify filter logic` |
| `Docs` | 문서 수정 | `[Docs] Update README` |
| `Test` | 테스트 추가 | `[Test] Add integration tests` |
| `Chore` | 빌드/패키지 등 | `[Chore] Update dependencies` |

### 4️⃣ PR 템플릿
```markdown
## 관련 이슈
- Closes #<이슈번호>

## 작업 내용
- 작업1
- 작업2

## 테스트 결과
- [x] Postman 테스트 완료
- [x] 유닛 테스트 통과
- [ ] 통합 테스트

## 📸 스크린샷 (선택)

## 💬 리뷰 요구사항
```

### 5️⃣ 코드 구조 및 계층
```
healthcare_AIchatbot_service_backend/
├── src/main/java/com/petlog/healthcare/
│   ├── controller/          # REST API 엔드포인트
│   ├── service/             # 비즈니스 로직
│   ├── entity/              # JPA Entity (Domain Model)
│   ├── dto/                 # Request/Response DTO
│   ├── config/              # 설정 클래스
│   ├── filter/              # 필터
│   ├── exception/           # Exception 정의
│   ├── kafka/               # Kafka Producer/Consumer
│   └── health/              # Health Check
├── src/main/resources/
│   ├── application.yml      # 기본 설정
│   ├── application-dev.yml  # 개발 환경
│   └── application-prod.yml # 상용 환경
└── build.gradle             # Maven 의존성
```

---

## 🌐 로컬 개발 환경 설정

### 포트 정보
| 서비스 | 포트 | 설명 |
|--------|------|------|
| API Gateway | 8000 | 통합 라우팅 및 인증 |
| User Service | 8080 | JWT 발급 및 사용자 관리 |
| Healthcare Service | 8085 | AI 챗봇 서비스 |
| Social Service | 8083 | SNS/커뮤니티 |
| Diary Service | 8087 | 일기 및 이벤트 |
| Milvus | 19530 | 벡터 DB (내부) |
| PostgreSQL | 5432 | 데이터베이스 |
| Redis | 6379 | Rate Limiting / 캐시 |

### 개발 환경 실행
```bash
# 1. Healthcare Service 실행
cd healthcare_AIchatbot_service_backend
./gradlew bootRun -Dspring.profiles.active=dev
# Port 8085 확인: curl http://localhost:8085/chat/health

# 2. Redis 실행 (Rate Limiting)
docker run -d -p 6379:6379 redis:7-alpine

# 3. API Gateway 실행
cd api_gateway
./gradlew bootRun -Dspring.profiles.active=dev
# Port 8000 확인: curl http://localhost:8000/api/health

# 4. Frontend 개발 서버
cd Frontend
npm run dev
# http://localhost:3000 또는 5173
```

### IntelliJ IDE 설정
**VM Options:**
```
-Dspring.profiles.active=dev
```

**위치:**
1. Run → Edit Configurations
2. Application 선택
3. VM options: `-Dspring.profiles.active=dev` 입력
4. OK

---

## 🔐 API Gateway 연동 기준

### JWT 인증 흐름
```
1. Frontend → User Service (로그인 요청)
2. User Service → JWT 토큰 발급
3. Frontend → API Gateway (JWT 포함한 요청)
4. API Gateway → JwtAuthenticationFilter에서 검증
5. API Gateway → Healthcare Service (X-User-Id, X-User-Email 헤더 추가)
6. Healthcare Service → 비즈니스 로직 처리
```

### Public/Private Endpoint
**Public (인증 불필요):**
- `GET /api/health/**`: Health Check

**Private (JWT 필수):**
- `POST /api/chat/general`: 일반 채팅
- `POST /api/chat/persona`: 페르소나 채팅

### Rate Limiting 규칙
- **제한:** 사용자당 분당 10개 요청
- **초과 시:** 429 Too Many Requests
- **백엔드:** Redis
- **Key:** X-User-Id (JWT에서 추출)

### Circuit Breaker 규칙
- **실패 기준:** 3회 연속 실패
- **상태:** OPEN (모든 요청 차단)
- **Fallback:** 503 Service Unavailable
- **복구:** 10초 후 Half-Open 상태로 재시도

---

## 📚 Gateway 연동 구현 가이드

### Phase별 구현 순서

#### Phase 0: 사전 확인 (5분)
- Healthcare Service 8085 정상 동작 확인
- Redis 실행 확인
- Gateway 8000 실행 확인

#### Phase 1: 기본 라우팅 (30분)
- `api_gateway/application.yml` 수정
- healthcare-service route 추가
- predicates: `/api/chat/**`, `/api/health/**`
- uri: `http://localhost:8085`

**Commit:**
```
[Feat] Add basic routing for Healthcare Service
```

#### Phase 2: JWT 인증 필터 (1시간)
- `JwtAuthenticationFilter.java` 구현
- `JwtTokenProvider.java` 구현
- Public: `/api/health/**`
- Private: `/api/chat/**`

**Commit:**
```
[Feat] Add JWT authentication filter for Healthcare Service
```

#### Phase 3: 보안 & 성능 (1시간)
- `CorsConfig.java`: CloudFront origin 허용
- `RateLimitConfig.java`: 10 req/min 제한
- `FallbackController.java`: Circuit Breaker fallback
- Logging filters: 요청/응답 로깅

**Commit (3개):**
```
[Feat] Add CORS configuration
[Feat] Add Rate Limiting filter
[Feat] Add Circuit Breaker with fallback
```

#### Phase 4: 테스트 (1시간)
- Postman Collection 작성
- Bash 테스트 스크립트 작성
- 통합 테스트 수행

**Commit:**
```
[Test] Add Gateway integration test scripts
```

#### Phase 5: 문서화 (1시간)
- README.md 업데이트
- DEPLOYMENT.md 작성
- PR 생성

**Commit:**
```
[Docs] Update README and deployment guide
```

**총 소요 시간: 약 5시간**

---

## 📖 참고 자료 및 레퍼런스

### 팀 레퍼런스 서비스
- **User Service:** JWT 발급/검증 로직
  - Repository: `user_service_backend`
  - 참고 파일: `JwtTokenProvider.java`, `SecurityConfig.java`

- **Social Service:** Gateway 필터 적용 패턴
  - Repository: `social_service_backend`
  - 참고: CORS, Rate Limiting, error handling

- **Diary Service:** Kafka 이벤트 발행 패턴
  - Repository: `record_service_backend`
  - 참고: Kafka Producer 설정

### 외부 레퍼런스
- **Spring Cloud Gateway 공식 문서:** https://spring.io/projects/spring-cloud-gateway
- **JWT (JSON Web Tokens):** https://jwt.io/
- **AWS Bedrock:** https://aws.amazon.com/bedrock/
- **Milvus 벡터 DB:** https://milvus.io/
- **Apache Kafka:** https://kafka.apache.org/

### 팀 UI/UX 레퍼런스
- **Lifet (반려동물 통합 플랫폼):** https://lifet.co.kr/
- **Vdoc (AI 문서 분석):** https://ai.vdoc.kr/

---

## 🔧 Antigravity Agent 행동 규칙

### 1️⃣ 정보 수집 우선
- 항상 최신 GitHub repository 상태 확인
- 다른 팀원의 코드 패턴 분석 후 일관성 유지
- 현재 구현 상태 파악 후 대응

### 2️⃣ 주요 결정 기준 (WHY)
모든 기술 결정에 "WHY" 설명 포함:
```
WHY Spring Cloud Gateway?
→ 마이크로서비스 통합 라우팅, JWT 검증, Rate Limiting을 한곳에서 관리

WHY Circuit Breaker?
→ 장애 전파 방지, 부분 장애 격리

WHY Milvus?
→ 오픈소스, 높은 성능, 자체 관리 가능, Pinecone보다 저가
```

### 3️⃣ 코드 생성 규칙
- 모든 코드에 WHY 주석 포함
- 팀 컨벤션 준수 (네이밍, 구조, Git)
- 완벽한 구현 (TODO, 플레이스홀더 없음)
- 에러 핸들링 포함 (try-catch, validation)
- 로깅 포함 (DEBUG, INFO, WARN)

### 4️⃣ 구현 순서
1. 기존 코드 분석 (다른 서비스 참고)
2. 유사 패턴 찾기 (코드 일관성)
3. 팀 컨벤션 적용
4. 테스트 코드 작성
5. Git 커밋 가이드 제공

### 5️⃣ 피드백 스타일
- 객관적 (개인 의견 제시 금지)
- 현직 기준 (프로덕션 배포 고려)
- 근거 제시 (왜 이렇게 하는지 설명)
- 대안 제시 (여러 옵션 중 최적안)

### 6️⃣ 배포 고려사항
- dev/prod 프로파일 분리
- 환경 변수 문서화
- 배포 담당자를 위한 가이드 제공
- AWS 배포 (ECS/EKS) 고려

---

## 📋 요청 패턴 및 응답 기준

### 🎯 요청 예시 1: Gateway 연동
```
"Gateway를 Healthcare Service와 연동하고 싶어"

예상 응답:
1. Healthcare Service 현재 상태 분석
2. 기존 Gateway 패턴 학습 (User/Social Service)
3. Sequential Thinking으로 5단계 계획 수립
4. 모든 구현 코드 생성 (application.yml + 7개 Java 파일)
5. Postman Collection + 테스트 스크립트
6. Git Commit 순서 제시
7. 최종 체크리스트
```

### 🎯 요청 예시 2: 특정 버그 수정
```
"JwtAuthenticationFilter에서 401 에러 발생"

예상 응답:
1. 에러 원인 분석 (JWT Secret Key, 토큰 형식 등)
2. 로그 분석 요청
3. 수정 코드 제시
4. 테스트 방법 설명
5. Git Commit 메시지 제공
```

### 🎯 요청 예시 3: 새 기능 추가
```
"Rate Limiting을 IP 기반에서 User 기반으로 변경"

예상 응답:
1. 현재 구현 분석
2. 변경 사항 설명 (WHY, HOW)
3. 수정할 코드만 제시 (전체 아님)
4. 테스트 방법
5. 호환성 확인 (다른 부분 영향)
6. Git Commit
```

---

## 🚀 빠른 실행 명령어

### Healthcare Service 구성
```bash
# 1. Repository 클론
git clone https://github.com/skRookies3team/healthcare_AIchatbot_service_backend.git
cd healthcare_AIchatbot_service_backend

# 2. 개발 환경 실행
./gradlew bootRun -Dspring.profiles.active=dev

# 3. Health Check
curl http://localhost:8085/chat/health
```

### Gateway 연동 (기본)
```bash
# 1. api_gateway 클론
git clone https://github.com/skRookies3team/api_gateway.git
cd api_gateway

# 2. application.yml 수정 (healthcare-service route 추가)

# 3. Gateway 실행
./gradlew bootRun -Dspring.profiles.active=dev

# 4. 라우팅 테스트
curl http://localhost:8000/api/health
curl http://localhost:8000/api/chat/general (with JWT)
```

### 테스트 실행
```bash
# User Service에서 JWT 발급
JWT_TOKEN=$(curl -s -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@test.com","password":"test123"}' \
  | jq -r '.token')

# 테스트 스크립트 실행
chmod +x test-gateway-integration.sh
./test-gateway-integration.sh "$JWT_TOKEN"
```

---

## 📌 최종 체크리스트

**Antigravity에 요청할 때 이 체크리스트를 포함하면:**
- ✅ 모든 컨텍스트 이미 제공됨
- ✅ 빠르고 정확한 대응
- ✅ 팀 컨벤션 자동 준수
- ✅ 프로덕션 준비 완료

---

## 💡 주요 원칙

1. **MSA 중심:** 모든 결정이 마이크로서비스 아키텍처 기준
2. **현직 기준:** 실제 기업 수준의 코드 품질
3. **일관성:** 다른 팀원 코드와 동일한 패턴 사용
4. **문서화:** 모든 코드, 결정, 프로세스 명확히 문서화
5. **효율성:** 최소한의 대화로 최대한의 산출물 생성

---

**이 파일을 Antigravity Customizations에 등록하면, 앞으로 모든 요청에서 자동으로 이 컨텍스트가 적용됩니다!** ✨
