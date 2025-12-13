# 포트 상세 가이드

> weAlist 프로젝트에서 사용하는 모든 포트와 역할을 정리한 문서입니다.

## 목차

- [애플리케이션 서비스](#애플리케이션-서비스)
- [인프라 서비스](#인프라-서비스)
- [LiveKit 포트 상세](#livekit-포트-상세)
- [WebSocket vs LiveKit 비교](#websocket-vs-livekit-비교)
- [네트워크 흐름](#네트워크-흐름)
- [Kubernetes 포트 설정](#kubernetes-포트-설정)
- [Kind 클러스터 설정](#kind-클러스터-설정)

---

## 애플리케이션 서비스

| 서비스 | 포트 | 프로토콜 | 용도 |
|--------|------|----------|------|
| **Frontend** | 3000:5173 | HTTP | React + Vite 개발 서버 |
| **Auth Service** | 8080 | HTTP | OAuth2, JWT 토큰 관리 (Spring Boot) |
| **User Service** | 8081 | HTTP | 사용자/워크스페이스 관리 |
| **Board Service** | 8000 | HTTP + WS | 칸반 보드 (WebSocket 실시간 동기화) |
| **Chat Service** | 8001 | HTTP + WS | 채팅 (WebSocket 메시지) |
| **Notification Service** | 8002 | HTTP + SSE | 알림 (Server-Sent Events) |
| **Storage Service** | 8003 | HTTP | 파일 저장소 관리 |
| **Video Service** | 8004 | HTTP | 비디오 룸 관리, LiveKit API 연동 |

---

## 인프라 서비스

| 서비스 | 포트 | 프로토콜 | 용도 |
|--------|------|----------|------|
| **Nginx** | 80 | HTTP | API Gateway, 리버스 프록시 |
| **PostgreSQL** | 5432 | TCP | 데이터베이스 |
| **Redis** | 6379 | TCP | 캐시/세션/Pub-Sub |
| **MinIO API** | 9000 | HTTP | S3 호환 저장소 API |
| **MinIO Console** | 9001 | HTTP | MinIO 관리 콘솔 |
| **Prometheus** | 9090 | HTTP | 메트릭 수집 |
| **Grafana** | 3001 | HTTP | 모니터링 대시보드 |
| **Loki** | 3100 | HTTP | 로그 수집 |
| **Redis Exporter** | 9121 | HTTP | Redis 메트릭 익스포터 |
| **PostgreSQL Exporter** | 9187 | HTTP | PostgreSQL 메트릭 익스포터 |

---

## LiveKit 포트 상세

### 포트 요약

| 포트 | 프로토콜 | 역할 | 필수 여부 |
|------|----------|------|-----------|
| **7880** | TCP | HTTP API + WebSocket 시그널링 | 필수 |
| **7881** | TCP | WebRTC TCP 폴백 | 권장 |
| **50000-50020** | UDP | RTP 미디어 데이터 (개발) | 필수 |
| **50000-60000** | UDP | RTP 미디어 데이터 (프로덕션) | 필수 |
| **3478** | UDP/TCP | Built-in TURN (NAT 통과) | 권장 |

### 포트 7880 (TCP) - HTTP API & WebSocket

```
┌─────────────────────────────────────────────────────────────┐
│  역할: LiveKit의 메인 통신 포트                              │
├─────────────────────────────────────────────────────────────┤
│  • REST API: 룸 생성, 참가자 관리, 토큰 검증                 │
│  • WebSocket: 시그널링 (SDP offer/answer, ICE candidate)    │
│  • 브라우저 ↔ LiveKit 서버 간 제어 메시지 전달               │
└─────────────────────────────────────────────────────────────┘

사용 예:
  - Client SDK → ws://localhost:7880 (WebSocket 연결)
  - video-service → http://livekit:7880 (API 호출)
```

### 포트 7881 (TCP) - TCP Fallback

```
┌─────────────────────────────────────────────────────────────┐
│  역할: UDP가 차단된 환경에서 미디어 전송용 TCP 폴백          │
├─────────────────────────────────────────────────────────────┤
│  • 일부 기업 네트워크/방화벽에서 UDP 차단 시 사용            │
│  • TURN-over-TCP 방식으로 미디어 릴레이                     │
│  • UDP보다 지연시간이 길지만 연결 안정성 보장                │
└─────────────────────────────────────────────────────────────┘
```

### 포트 50000-50020/UDP (개발) - RTP 미디어 포트

```
┌─────────────────────────────────────────────────────────────┐
│  역할: 실제 오디오/비디오 데이터(RTP) 전송                   │
├─────────────────────────────────────────────────────────────┤
│  • 각 미디어 스트림마다 별도 포트 할당                       │
│  • UDP: 실시간 미디어에 최적화 (낮은 지연)                   │
│  • 프로덕션: 50000-60000 (최대 10,000개 동시 스트림)         │
│  • 개발: 50000-50020 (21개 포트로 제한)                      │
└─────────────────────────────────────────────────────────────┘

포트 범위가 필요한 이유:
  - 각 참가자의 오디오/비디오는 별도 RTP 스트림
  - 참가자 1명 = 약 2-4개 포트 (오디오 + 비디오 송수신)
  - 10명 회의 = 약 40-80개 포트 필요
```

### 포트 3478 (UDP/TCP) - Built-in TURN

```
┌─────────────────────────────────────────────────────────────┐
│  역할: NAT 통과 및 미디어 릴레이 (coturn 대체)               │
├─────────────────────────────────────────────────────────────┤
│  • STUN: NAT 뒤에 있는 클라이언트의 공인 IP 발견            │
│  • TURN: P2P 연결 실패 시 미디어 릴레이                     │
│  • LiveKit에 내장되어 별도 coturn 서버 불필요                │
└─────────────────────────────────────────────────────────────┘
```

---

## WebSocket vs LiveKit 비교

### 왜 다른 서비스들은 일반 WebSocket을 쓰고, LiveKit은 별도 포트를 사용하는가?

#### 일반 WebSocket (Board, Chat 등)

```
┌─────────────────────────────────────────────────────────────┐
│                    일반 WebSocket 서비스                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Browser ───── WebSocket (TCP) ─────► Service (8000/8001)  │
│              ws://host:8000/ws                              │
│                                                             │
│   전송 데이터:                                               │
│   • JSON 메시지 (수 bytes ~ 수 KB)                          │
│   • 텍스트 기반 이벤트                                       │
│   • 단방향 또는 양방향 메시지                                │
│                                                             │
│   특징:                                                     │
│   • TCP 기반 - 순서 보장, 재전송                            │
│   • 지연 허용 (100-500ms OK)                                │
│   • 단일 포트로 충분                                        │
│   • HTTP Upgrade로 연결                                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘

예시:
  - Chat: "사용자A가 메시지를 보냄" → JSON 이벤트 전송
  - Board: "카드가 이동됨" → 상태 동기화 이벤트
```

#### LiveKit WebRTC (Video)

```
┌─────────────────────────────────────────────────────────────┐
│                      LiveKit WebRTC                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Browser ─┬─ WebSocket (7880) ──► 시그널링 (제어)          │
│            │                                                │
│            ├─ UDP (50000-50020) ─► 미디어 (오디오/비디오)    │
│            │                                                │
│            └─ TCP (7881) ────────► 폴백 (UDP 차단 시)        │
│                                                             │
│   전송 데이터:                                               │
│   • RTP 패킷 (오디오: 20ms 단위, 비디오: 프레임 단위)         │
│   • 초당 수십~수백 패킷                                      │
│   • 720p 비디오 = 약 1-2 Mbps                               │
│                                                             │
│   특징:                                                     │
│   • UDP 기반 - 순서/재전송 없음 (실시간 우선)               │
│   • 지연 민감 (50ms 이하 필요)                              │
│   • 다중 포트 필요 (스트림당 1-2개)                         │
│   • ICE/STUN/TURN으로 NAT 통과                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 핵심 차이점

| 구분 | 일반 WebSocket | LiveKit (WebRTC) |
|------|----------------|------------------|
| **전송 데이터** | JSON 텍스트 (KB) | 오디오/비디오 바이너리 (Mbps) |
| **프로토콜** | TCP only | UDP + TCP 폴백 |
| **지연 허용** | 100-500ms OK | 50ms 이하 필요 |
| **포트 수** | 1개 | 다중 (시그널링 + 미디어 범위) |
| **NAT 통과** | 불필요 (HTTP 기반) | STUN/TURN 필요 |
| **패킷 손실** | 재전송 | 무시 (다음 프레임 사용) |
| **사용 사례** | 채팅, 실시간 동기화 | 화상회의, 스트리밍 |

### 왜 LiveKit은 별도 포트가 필요한가?

```
1. UDP 미디어 전송
   ─────────────────
   일반 WebSocket은 TCP만 사용하므로 HTTP 포트(80/443)로 충분
   LiveKit은 UDP로 미디어를 전송해야 하므로 별도 포트 범위 필요

2. 동시 다중 스트림
   ─────────────────
   채팅: 모든 메시지가 하나의 TCP 연결로 처리
   비디오: 참가자마다 별도 UDP 스트림 → 포트 범위 필요

3. NAT 통과 (TURN)
   ─────────────────
   WebSocket: 표준 HTTP 포트라 NAT 문제 없음
   WebRTC: P2P 특성상 NAT 통과 메커니즘 필요 → 3478 포트

4. 품질 vs 신뢰성 트레이드오프
   ─────────────────────────────
   채팅에서 메시지 손실: 치명적 (TCP 재전송 필요)
   비디오에서 프레임 손실: 허용 (다음 프레임으로 대체)
```

---

## 네트워크 흐름

### 전체 아키텍처에서의 포트 사용

```
                                   Internet
                                      │
                                      ▼
                              ┌──────────────┐
                              │    Nginx     │
                              │   (Port 80)  │
                              └──────┬───────┘
                                     │
           ┌─────────────────────────┼─────────────────────────┐
           │                         │                         │
           ▼                         ▼                         ▼
    ┌─────────────┐          ┌─────────────┐          ┌─────────────┐
    │   /api/*    │          │  /livekit   │          │   /minio    │
    │  Services   │          │  WebSocket  │          │    S3 API   │
    │ (8000-8004) │          │   (7880)    │          │   (9000)    │
    └─────────────┘          └──────┬──────┘          └─────────────┘
                                    │
                                    ▼
                            ┌──────────────┐
                            │   LiveKit    │
                            │   Server     │
                            ├──────────────┤
                            │ HTTP:  7880  │ ◄── 시그널링
                            │ TCP:   7881  │ ◄── TCP 폴백
                            │ UDP: 50000+  │ ◄── 미디어
                            │ TURN: 3478   │ ◄── NAT 통과
                            └──────────────┘
```

### WebRTC 연결 시퀀스

```
┌────────────────────────────────────────────────────────────────────┐
│                        연결 시퀀스                                  │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  1. video-service가 LiveKit API (7880)로 토큰 생성                 │
│                                                                    │
│  2. 브라우저가 토큰으로 LiveKit WebSocket (7880) 연결              │
│                                                                    │
│  3. ICE Candidate 수집:                                            │
│     ├── Host: 로컬 IP (192.168.x.x)                               │
│     ├── Server Reflexive: STUN으로 발견한 공인 IP (3478)           │
│     └── Relay: TURN 릴레이 주소 (3478)                             │
│                                                                    │
│  4. ICE Connectivity Check:                                        │
│     ├── 우선순위 1: Host (직접 연결)                               │
│     ├── 우선순위 2: Server Reflexive (STUN)                        │
│     └── 우선순위 3: Relay (TURN) - 항상 성공 보장                   │
│                                                                    │
│  5. 미디어 전송 시작 (UDP 50000-50020)                             │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## coturn 없이 운영 (현재 설정)

LiveKit은 Built-in TURN 서버를 내장하고 있어 별도 coturn 없이 NAT 통과가 가능합니다.

### livekit.yaml 설정

```yaml
turn:
  enabled: true          # Built-in TURN 서버 활성화
  domain: localhost      # TURN 도메인
  udp_port: 3478         # STUN/TURN UDP 포트
  tls_port: 0            # 개발에서는 TLS 비활성화 (프로덕션: 5349)
```

### Built-in TURN vs 외부 coturn

| 구분 | Built-in TURN | 외부 coturn |
|------|---------------|-------------|
| **설정** | LiveKit 설정만으로 완료 | 별도 서버 구성 필요 |
| **스케일링** | LiveKit과 함께 스케일 | 별도 스케일링 필요 |
| **리소스** | LiveKit 프로세스 내 | 별도 프로세스/컨테이너 |
| **유연성** | LiveKit 기능에 한정 | 다양한 옵션 지원 |
| **권장 환경** | 소~중규모 | 대규모, 특수 요구사항 |

---

## Docker Compose 포트 매핑 요약

```yaml
# 애플리케이션 서비스
frontend-service:    "3000:5173"
auth-service:        "8080:8080"
user-service:        "8081:8081"
board-service:       "8000:8000"
chat-service:        "8001:8001"
noti-service:        "8002:8002"
storage-service:     "8003:8003"
video-service:       "8004:8004"

# LiveKit
livekit:
  - "7880:7880"                    # HTTP/WS API + 시그널링
  - "7881:7881"                    # TCP 폴백 (WebRTC)
  - "50000-50020:50000-50020/udp"  # RTP 미디어 (UDP)

# 인프라
nginx:               "80:80"
postgres:            "5432:5432"
redis:               "6379:6379"
minio:               "9000:9000", "9001:9001"

# 모니터링
prometheus:          "9090:9090"
grafana:             "3001:3000"
loki:                "3100:3100"
```

---

## 방화벽 설정 가이드

### 개발 환경 (로컬)

특별한 방화벽 설정 불필요

### 프로덕션 환경

```bash
# 필수 포트
80/tcp      # HTTP (Nginx)
443/tcp     # HTTPS (Nginx)
7880/tcp    # LiveKit API/WebSocket
7881/tcp    # LiveKit TCP 폴백

# UDP 포트 범위 (LiveKit 미디어)
50000-60000/udp

# TURN 서버 (Built-in)
3478/udp
3478/tcp
```

---

## Kubernetes 포트 설정

### Service 타입 이해

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Service Types                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ClusterIP (기본값)                                          │
│  ├── 클러스터 내부에서만 접근 가능                            │
│  ├── Pod간 통신에 사용                                       │
│  └── 예: postgres:5432, redis:6379                          │
│                                                             │
│  NodePort                                                   │
│  ├── 클러스터 외부에서 접근 가능                             │
│  ├── 모든 노드의 특정 포트로 노출 (30000-32767)              │
│  └── 예: livekit-external:30880                             │
│                                                             │
│  LoadBalancer                                               │
│  ├── 클라우드 환경에서 외부 로드밸런서 생성                   │
│  ├── AWS ALB/NLB, GCP Load Balancer 등                      │
│  └── 프로덕션 환경에서 권장                                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### port vs targetPort vs nodePort

```yaml
spec:
  type: NodePort
  ports:
    - port: 7880         # Service 포트 (클러스터 내부)
      targetPort: 7880   # Pod 포트 (컨테이너)
      nodePort: 30880    # Node 포트 (외부 접근)
```

```
외부 요청 흐름:
  Client → Node:30880 → Service:7880 → Pod:7880

내부 요청 흐름:
  다른 Pod → Service:7880 → Pod:7880
```

### LiveKit Kubernetes Services

```yaml
# 파일: infrastructure/base/livekit/service.yaml

# 1. ClusterIP Service (내부 통신용)
apiVersion: v1
kind: Service
metadata:
  name: livekit
spec:
  type: ClusterIP
  ports:
    - port: 7880      # HTTP API + WebSocket
    - port: 7881      # RTC TCP
    - port: 3478/UDP  # TURN UDP
    - port: 3478/TCP  # TURN TCP

# 2. NodePort Service (외부 WebRTC용)
apiVersion: v1
kind: Service
metadata:
  name: livekit-external
spec:
  type: NodePort
  ports:
    - port: 7880, nodePort: 30880  # HTTP
    - port: 7881, nodePort: 30881  # RTC TCP
    - port: 3478, nodePort: 30478  # TURN UDP
    - port: 3478, nodePort: 30479  # TURN TCP
```

---

## Kind 클러스터 설정

### Kind의 포트 노출 제한

Kind(Kubernetes in Docker)는 Docker 컨테이너 내에서 K8s를 실행하므로:
- 포트 범위(50000-60000) 매핑이 어려움
- 각 포트를 명시적으로 선언해야 함
- UDP 포트 범위 노출에 제한

### TCP 전용 모드 (Kind 환경)

UDP 포트 범위 제한을 우회하기 위해 TCP 전용 모드 사용:

```yaml
# 파일: infrastructure/base/livekit/configmap.yaml

rtc:
  tcp_port: 7881
  use_external_ip: true
  node_ip: "공인IP"           # 라우터 포트포워딩 대상 IP
  port_range_start: 7881      # UDP 비활성화 (TCP와 동일하게)
  port_range_end: 7881

turn:
  enabled: true
  udp_port: 3478
  tcp_port: 3478
```

### Kind extraPortMappings

```yaml
# 파일: docker/scripts/dev/kind-config.yaml

nodes:
  - role: control-plane
    extraPortMappings:
      # Ingress (Nginx)
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP

      # LiveKit WebRTC TCP
      - containerPort: 7881
        hostPort: 7881
        protocol: TCP

      # LiveKit TURN UDP
      - containerPort: 3478
        hostPort: 3478
        protocol: UDP

      # LiveKit TURN TCP (UDP 차단 환경용)
      - containerPort: 3478
        hostPort: 3478
        protocol: TCP
```

### 외부 접근 흐름 (Kind + WSL)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    외부 접근 흐름 (Kind + WSL)                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Internet                                                           │
│      │                                                              │
│      ▼                                                              │
│  공유기 (포트포워딩)                                                 │
│  ├── 7880 → 192.168.x.x:7880 (HTTP/WS 시그널링)                    │
│  ├── 7881 → 192.168.x.x:7881 (RTC TCP)                             │
│  └── 3478 → 192.168.x.x:3478 (TURN UDP/TCP)                        │
│      │                                                              │
│      ▼                                                              │
│  호스트 PC (Windows)                                                │
│      │                                                              │
│      ▼                                                              │
│  WSL2 (포트 자동 포워딩)                                            │
│      │                                                              │
│      ▼                                                              │
│  Kind Docker Container                                              │
│  └── extraPortMappings                                              │
│      │                                                              │
│      ▼                                                              │
│  Kubernetes Cluster                                                 │
│  └── NodePort Service (livekit-external)                            │
│      │                                                              │
│      ▼                                                              │
│  LiveKit Pod                                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 클러스터 재생성 방법

```bash
# 1. 기존 클러스터 삭제
make kind-delete

# 2. 새 클러스터 생성 (kind-config.yaml의 extraPortMappings 적용)
make kind-setup

# 3. 이미지 로드
make kind-load-images

# 4. 배포 (ConfigMap 포함)
make kind-apply

# 상태 확인
make status
kubectl get configmap livekit-config -n wealist-dev -o yaml
```

### TCP vs UDP 성능 차이

| 환경 | 모드 | 지연시간 | 품질 | 권장 |
|------|------|----------|------|------|
| Kind (개발) | TCP only | 50-100ms | 보통 | ✓ |
| EKS (프로덕션) | UDP + TCP fallback | 20-50ms | 최상 | ✓ |

개발 환경에서는 TCP 전용 모드로도 충분하며, 프로덕션(EKS)에서는 UDP 포트 범위를 열어 최적 성능을 확보합니다.
