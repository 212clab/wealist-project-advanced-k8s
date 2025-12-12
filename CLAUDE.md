# weAlist Project Context

> 이 파일은 Claude Code 세션에서 자동으로 로드되어 프로젝트 컨텍스트로 사용됩니다.

## 프로젝트 개요

**weAlist**는 팀 협업을 위한 클라우드 네이티브 마이크로서비스 플랫폼입니다.

### 핵심 기능
- 워크스페이스 기반 팀 관리
- 칸반 보드 기반 프로젝트/태스크 관리
- 실시간 채팅 (WebSocket)
- 실시간 알림 (SSE)
- 클라우드 파일 스토리지 (S3/MinIO)
- 영상/음성 통화 (LiveKit WebRTC)

---

## 아키텍처

```
┌─────────────────────────────────────────────────────┐
│              Frontend (React + Vite)                 │
│                    :3000                             │
└────────────────────┬────────────────────────────────┘
                     │
         ┌───────────▼───────────┐
         │  NGINX API Gateway    │
         │      :80              │
         └───────┬───────────────┘
                 │
    ┌────────────┼────────────────────────────┐
    │            │                            │
    ▼            ▼            ▼               ▼
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
│ Auth   │ │ User   │ │ Board  │ │ Chat   │ │ Noti   │ ...
│:8080   │ │:8081   │ │:8000   │ │:8001   │ │:8002   │
│Spring  │ │Go/Gin  │ │Go/Gin  │ │Go/Gin  │ │Go/Gin  │
└────────┘ └────────┘ └────────┘ └────────┘ └────────┘
```

---

## 마이크로서비스

| 서비스 | 포트 | 기술 | 역할 |
|--------|------|------|------|
| **Auth Service** | 8080 | Spring Boot 3 | JWT/OAuth2 인증 |
| **User Service** | 8081 | Go + Gin | 사용자/워크스페이스 관리 |
| **Board Service** | 8000 | Go + Gin | 프로젝트/보드/댓글 관리 |
| **Chat Service** | 8001 | Go + Gin + WebSocket | 실시간 채팅 |
| **Noti Service** | 8002 | Go + Gin + SSE | 실시간 알림 |
| **Storage Service** | 8003 | Go + Gin + S3 | 파일 스토리지 |
| **Video Service** | 8004 | Go + Gin + LiveKit | 영상통화 |
| **Frontend** | 3000 | React + Vite + TypeScript | 웹 UI |

---

## 기술 스택

- **Backend**: Go 1.21+ (Gin), Spring Boot 3 (Auth만)
- **Frontend**: React 18, Vite, TypeScript
- **Database**: PostgreSQL 17 (서비스별 독립 DB)
- **Cache**: Redis 7.2
- **Storage**: MinIO (S3 호환)
- **WebRTC**: LiveKit + Coturn
- **Container**: Docker, Kubernetes
- **CI/CD**: GitHub Actions, ArgoCD

---

## 디렉토리 구조

```
wealist-project-advanced-k8s/
├── services/
│   ├── auth-service/        # Spring Boot (Java)
│   ├── user-service/        # Go
│   ├── board-service/       # Go
│   ├── chat-service/        # Go
│   ├── noti-service/        # Go
│   ├── storage-service/     # Go
│   ├── video-service/       # Go
│   └── frontend/            # React
├── docker/
│   ├── compose/             # docker-compose 파일
│   └── env/                 # 환경변수 템플릿
├── k8s/
│   ├── base/                # K8s 기본 매니페스트
│   └── overlays/            # 환경별 오버레이 (local, eks)
├── team_A_diagrams/         # 팀 다이어그램
│   └── 05_business_logic.md # 비즈니스 로직 다이어그램
└── docs/                    # 문서
```

---

## Go 서비스 구조 (공통)

```
services/{service-name}/
├── cmd/main.go              # 엔트리포인트
├── internal/
│   ├── config/              # 설정
│   ├── router/              # API 라우터
│   ├── handler/             # HTTP 핸들러
│   ├── service/             # 비즈니스 로직
│   ├── repository/          # DB 접근
│   ├── model/               # 도메인 모델
│   ├── dto/                 # DTO
│   └── middleware/          # 미들웨어
├── migrations/              # DB 마이그레이션
└── Dockerfile
```

---

## 핵심 비즈니스 로직

### 1. 인증 플로우
- Google OAuth2 → Auth Service → JWT 발급
- User Service에서 사용자 자동 생성
- Access Token (15분) + Refresh Token (7일)

### 2. 워크스페이스 & 프로젝트
- 워크스페이스 생성 시 기본 프로젝트 자동 생성
- RBAC: OWNER > ADMIN > MEMBER

### 3. 칸반 보드
- Fractional Indexing으로 보드 위치 O(1) 이동
- Custom Fields (JSONB) + Field Options
- WebSocket으로 실시간 동기화

### 4. 실시간 시스템
- Chat: WebSocket + Redis Pub/Sub
- Noti: SSE (Server-Sent Events)
- Video: LiveKit SFU + Coturn TURN

---

## 개발 가이드

### 로컬 실행
```bash
# Docker Compose로 전체 실행
docker compose -f docker/compose/docker-compose.yml up -d

# 개별 서비스 실행 (Go)
cd services/board-service
go run cmd/main.go
```

### Swagger UI
- User: http://localhost:8081/swagger/index.html
- Board: http://localhost:8000/swagger/index.html
- Chat: http://localhost:8001/swagger/index.html

### K8s 배포
```bash
make k8s-apply-local   # 로컬 (Minikube/Kind)
make k8s-apply-eks     # AWS EKS
```

---

## 서비스 간 통신

### 내부 API (Service-to-Service)
| 호출 서비스 | 대상 | 엔드포인트 |
|------------|------|-----------|
| Auth → User | OAuth 로그인 | `POST /api/internal/oauth/login` |
| Board → User | 사용자 확인 | `GET /api/internal/users/{id}/exists` |
| Board → Noti | 알림 생성 | `POST /api/internal/notifications` |
| Chat → Noti | 알림 생성 | `POST /api/internal/notifications` |

---

## 다이어그램 위치

상세한 비즈니스 로직 다이어그램:
- `team_A_diagrams/05_business_logic.md` (Mermaid 기반)

---

## 환경 설정

### 핵심 참조 파일
| 용도 | 파일 경로 |
|------|-----------|
| Docker-compose 환경변수 | `docker/env/.env.dev.example` |
| Docker-compose 메인 설정 | `docker/compose/docker-compose.yml` |
| K8s 서비스 설정 | `services/{service}/k8s/base/` |
| 상세 설정 문서 | `docs/CONFIGURATION.md` |

### 인프라 포트
| 서비스 | 포트 | 비고 |
|--------|------|------|
| nginx | 80 | API Gateway |
| postgres | 5432 | PostgreSQL 17 |
| redis | 6379 | Redis 7.2 |
| minio | 9000/9001 | S3 API / 콘솔 |
| livekit | 7880 | WebSocket |
| prometheus | 9090 | 메트릭 |
| grafana | 3001 | 대시보드 |

### 데이터베이스 패턴
- **DB 이름**: `wealist_{service}_db` (docker) / `{service}_db` (K8s)
- **유저**: `{service}_service`
- **비밀번호**: `{service}_service_password`
- **Auth Service**: DB 미사용, Redis만 사용

### 공통 환경변수
```bash
JWT_SECRET=your-super-secret-jwt-key-change-this-in-production-min-32-chars
JWT_ACCESS_TOKEN_EXPIRATION_MS=1800000      # 30분
JWT_REFRESH_TOKEN_EXPIRATION_MS=604800000   # 7일
REDIS_HOST=redis
REDIS_PORT=6379
S3_BUCKET=wealist-local-files
S3_ENDPOINT=http://minio:9000
```

---

## 주의사항

- 각 서비스는 **독립된 PostgreSQL DB** 사용
- JWT Secret은 모든 Go 서비스에서 공유
- 파일 업로드는 **Presigned URL** 방식 (클라이언트 → S3 직접)
- WebSocket 연결은 Redis Pub/Sub으로 스케일 아웃 지원
- **포트 변경 금지**: 모든 환경에서 고정
- K8s vs Docker-compose DB 이름: K8s는 prefix 없음

### Health Check 엔드포인트
| 서비스 타입 | Liveness | Readiness |
|-------------|----------|-----------|
| Go 서비스 | `/health` | `/ready` |
| Spring 서비스 | `/health` | `/ready` |
