# 아키텍처 상세

## 트래픽 흐름 (정상 케이스)

```
사용자
  ↓
Route 53 → CloudFront (WAF 적용)
  ↓
API Gateway (HTTP API v2)
  ↓ Lambda Authorizer 5단계 검증
  ├─ /vwr/* → Lambda (VWR-API) → DynamoDB (10-shard)
  └─ /api/v1/* → VPC Link → Internal ALB → EKS Service
                                              ├─ auth (8084)
                                              ├─ event (8085)
                                              ├─ ticket (8086)
                                              ├─ payment (8081)
                                              ├─ queue (8083)
                                              └─ community (8082)
                                                  ↓
                                          RDS MySQL / Redis / DynamoDB
```

## VWR 2-Tier 대기열 흐름

```
[Tier 1 — VWR (Lambda)]
  사용자 → POST /vwr/assign/{eventId}
        ← 번호표 발급 (DynamoDB position 저장)
  사용자 → GET /vwr/check/{requestId}  ← 적응형 폴링 (3~15초)
        ← admitted=true + entry-token 발급

[counter-advancer Lambda]
  EventBridge (1분 주기) → 내부 10초 × 6회 루프
  → DynamoDB servingCounter += BATCH_SIZE
  → CloudWatch QueueDepth/TotalAssigned 메트릭 발행

[Tier 2 — Queue Service (Spring Boot)]
  KEDA가 TotalAssigned 메트릭 기반 Pod 스케일링
  Redis ZSET (queue:{showId}:waiting / active / heartbeat)
  Lua Script로 원자적 enter/poll/move/cleanup
  ACTIVE 진입 시 booking-token 발급
```

## 결제 비동기 팬아웃

```
ticketService → Toss 결제 승인
  ↓
paymentService.confirmPayment()
  ↓ @TransactionalEventListener(AFTER_COMMIT)
SQS FIFO (payment-events.fifo) — DedupId로 중복 방지
  ↓
payment-worker Lambda
  ├─ ticketService.confirm (RESERVATION 확정)
  ├─ eventService.activateMembership (MEMBERSHIP 활성)
  └─ communityService.confirmTransfer (양도 확정)

실패 시 → payment-events-dlq.fifo 격리
```

## Multi-AZ HA 구성

- **EKS**: 2 AZ (ap-northeast-2a/2b) Karpenter NodePool
- **RDS MySQL**: Primary (2a) + Standby (2b) 동기 복제 + Read Replica (비동기)
- **ElastiCache Redis**: Primary (2a) + Replica (2b)
- **ALB**: Multi-AZ
- **NAT Gateway**: 외부 PG 호출용

## 5계층 보안

| 계층 | 도구 | 차단 대상 |
|---|---|---|
| 1. WAF | AWS WAFv2 + AWS Managed Rules | SQLi, XSS, Rate Limit, Bot Patterns |
| 2. Lambda Authorizer | 5단계 검증 | IP 블록, 봇 점수, JWT, UserId 추출, EntryToken |
| 3. VPC | Private Subnet | 백엔드 인터넷 노출 0 |
| 4. Security Group | 계층 체인 | 0.0.0.0/0 인바운드 단 한 줄도 없음 |
| 5. IRSA | Pod별 IAM Role | 최소 권한 (lambda invoke, sqs send 등 단위) |

## 관찰 가능성 스택

```
[수집]
EKS Pod (Spring Actuator) → Prometheus
Pod stdout              → Loki
AWS 리소스               → CloudWatch
                          ↓
[저장·시각화]
Monitoring EC2 (EKS 외부, t3.small)
  Prometheus + Grafana + Loki
                          ↓
[알림]
Discord Webhook
```

### 7개 대시보드

| 대시보드 | 핵심 패널 |
|---|---|
| Service Overview | 6개 서비스 UP/DOWN, 에러율, 응답시간, JVM Heap, DB Connection Pool |
| VWR Queue Operations | QueueDepth, TotalAssigned, VWR Lambda 헬스, DynamoDB R/W |
| Lambda Operations | Invocations, Duration, Throttles, Concurrent, Authorizer 차단 사유 |
| Business Metrics | API 트래픽, VWR 활동, Infrastructure Health |
| AWS Infrastructure | RDS, Redis, SQS, CloudFront, WAF |
| EKS Cluster | Pod 상태, 재시작, HPA 스케일링 |
| Logs | 에러 로그 실시간 검색 |
