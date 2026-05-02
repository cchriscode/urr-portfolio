# URR — 대규모 동시 접속 티켓팅 플랫폼

> KT TechUp 부트캠프 실무 프로젝트 (3단계) — 인프라 엔지니어 단독 담당
> 멀티 레포 구성: Frontend / 6 Microservices / VWR Lambda / GitOps / Macro Detection

---

## 한 줄 요약

티켓 오픈 순간 **10,000명 규모 동시 접속 폭증**을 안정적으로 처리하기 위해 **Lambda + EKS 하이브리드 아키텍처**, **VWR 2-Tier 대기열**, **5계층 보안**, **EKS 외부 모니터링**을 결합한 운영 가능한 티켓팅 시스템.

---

## 본인 담당 영역 (Infrastructure)

다른 팀원들이 프론트·백엔드를 담당하고, **저는 인프라 전 영역을 단독으로 담당**했습니다.

| 영역 | 내용 |
|---|---|
| **AWS 인프라** | EKS 1.33, Karpenter NodePool, RDS Multi-AZ, ElastiCache Redis, DynamoDB, Lambda, API Gateway, CloudFront, WAF |
| **컨테이너 운영** | EKS Pod 설계·HPA/KEDA 메트릭 기반 스케일링·Karpenter 노드 자동 프로비저닝 |
| **GitOps** | Kustomize base/overlay 구조, GitHub Actions CI → ECR → 매니페스트 업데이트 → 롤링 배포 |
| **보안** | WAF → Lambda Authorizer 5단계 → VPC Private Subnet → Security Group 체인 → IRSA |
| **모니터링** | EKS 외부 EC2에 Prometheus/Grafana/Loki 분리 배치, Grafana 대시보드 7개 표준화 |
| **비용 최적화** | DynamoDB On-Demand 자동 전환, VPC Endpoint 도입, CloudWatch 고해상도 메트릭 비용 통제 |
| **부하 테스트** | k6 기반 단계별 (100 → 500 → 1,000 → 2,000 → 8,000 → 10,000명) 시나리오 설계·실행·튜닝 |

---

## 아키텍처

```
                        Route 53
                            │
                       CloudFront + WAF
                            │
                  ┌─────────┴──────────┐
                  │                    │
                S3 (정적)          API Gateway
                                       │
                          ┌────────────┴───────────┐
                          │                        │
                  Lambda (VWR/Authorizer)    VPC Link → Internal ALB
                          │                        │
                  DynamoDB (10-shard)         EKS Cluster (Karpenter)
                                                   │
                          ┌────────┬────────┬──────┴─┬────────┬────────┐
                        Auth   Event   Ticket   Payment   Queue   Community
                          │        │        │        │       │        │
                          └────────┴───┬────┴────────┴───────┴────────┘
                                       │
                          RDS MySQL (Multi-AZ + Read Replica)
                          Redis (Primary + Replica, 2 AZ)
                          SQS FIFO + DLQ (Payment 후속 팬아웃)

                          ────────────────────────────
                          Monitoring EC2 (EKS 외부 분리)
                          Prometheus / Grafana / Loki
```

---

## 멀티 레포 구성

본 포트폴리오는 KT TechUp 조직(`KTCloud-TechUp`) 산하의 8개 레포를 submodule로 묶어 단일 진입점에서 전체 시스템을 탐색할 수 있게 구성했습니다.

| 레포 | 역할 | 기술 스택 |
|---|---|---|
| **[urr-frontend](./urr-frontend)** | 사용자 UI | Next.js 16, React 19, TypeScript, Tailwind, TanStack Query, Zustand |
| **[urr-authService](./urr-authService)** | 인증·JWT·OAuth (Kakao) | Spring Boot 4, Java 21, Spring Security, JPA |
| **[urr-eventService](./urr-eventService)** | 이벤트·공연·좌석 카탈로그·멤버십 | Spring Boot 4, JPA, Caffeine 캐시 |
| **[urr-ticketService](./urr-ticketService)** | 좌석 LOCK·예약 상태 머신 | Spring Boot 4, Redis SETNX, JPA Optimistic Lock |
| **[urr-paymentService](./urr-paymentService)** | Toss Payments 연동·결제 후처리 | Spring Boot 4, SQS FIFO, @TransactionalEventListener |
| **[urr-queueService](./urr-queueService)** | Tier2 대기열 (Redis ZSET) | Spring Boot 4, Lua Script, ShedLock |
| **[urr-communityService](./urr-communityService)** | 티켓 양도 마켓 | Spring Boot 4, Saga 패턴 |
| **[urr-vwr](./urr-vwr)** | Tier1 VWR (Lambda) | Node.js 24, AWS SDK v3, DynamoDB 10-shard |
| **[urr-gitops](./urr-gitops)** | K8s 매니페스트·Terraform·운영 스크립트 | Kustomize, Terraform, GitHub Actions |
| **[urr-macroService](./urr-macroService)** | 매크로 탐지 (FastAPI) | Python 3.10, FastAPI, QMacroDetector |

---

## 핵심 설계 결정 (인프라 관점)

### 1) Lambda + EKS 하이브리드
- **Lambda (VWR)**: 0 → 대규모 burst를 무제한 스케일로 흡수, 유휴 시 비용 0
- **EKS (비즈니스 로직)**: DB 트랜잭션·커넥션 풀·복잡한 상태 관리에 적합
- **분리 기준**: "burst 흡수" vs "안정 운영" 두 책임을 각자 잘하는 런타임에 위임

### 2) VWR 2-Tier 대기열
| 티어 | 구현 | 역할 |
|---|---|---|
| Tier 1 (VWR) | Lambda + DynamoDB 10-shard | 순간 폭주 흡수, 봇/매크로 1차 차단 |
| Tier 2 (Queue) | Spring Boot + Redis ZSET | EKS 유입 속도 제어 (admissionRate) |

### 3) 5계층 보안 (Defense in Depth)
```
WAF (Rate Limit/SQLi 차단)
  → Lambda Authorizer 5단계 (IP·Bot·JWT·UserId·EntryToken)
    → VPC Private Subnet (백엔드 인터넷 노출 0)
      → Security Group 체인 (계층별 직전 SG만 허용, 0.0.0.0/0 인바운드 0)
        → IRSA (Pod별 최소 권한 IAM)
```

### 4) 모니터링 EC2 분리
EKS 자체가 장애 나도 대시보드는 살아있어야 원인 추적이 가능 → Prometheus/Grafana/Loki를 EKS 외부 EC2(t3.small)에서 운영.

---

## 트러블슈팅 하이라이트

### TS-1. Lambda Throttle 134건 → 0건
- **원인**: Authorizer + VWR Lambda가 같은 요청에서 동시 실행되어 quota 두 배 소진
- **해결**: VWR `assign` 경로를 `Auth:NONE`으로 분리, Lambda 내부에서 JWT 직접 검증
- **결과**: 동시 실행 절반, throttle 안정화

### TS-2. EKS AMI v1.33.8 kubelet 크래시
- **원인**: AWS 측 알려진 AMI 버그
- **해결**: Node Group v1.33.10 업데이트
- **시간**: 약 4시간 디버깅 → 5분 내 정상화
- **개선**: 장애 발생 5분 안에 코드/인프라 양쪽 의심 체크리스트 도입

### TS-3. NODE_ENV dimension 불일치
- **원인**: counter-advancer Lambda는 `NODE_ENV=production`으로 메트릭 publish, CloudWatch Adapter는 `value: dev` 하드코딩
- **해결**: 양쪽 dimension 일치시켜 메트릭 수집 정상화
- **의의**: 알람도 안 울리는 미세 버그 → 메트릭 흐름 직접 추적의 중요성

### TS-4. CloudWatch 메트릭 파이프라인 약 2~3분 → 약 36초
- **조치**: 고해상도 메트릭(StorageResolution: 1) + Adapter 폴링 60→10초 + HPA stabilizationWindow 30→10초 동시 조정
- **추가 비용**: 설계 단계 사전 검증으로 월 $2~3 수준에 통제

### TS-5. DynamoDB 4월 청구서 약 $92 비용 누수
- **원인**: 안전 마진으로 Provisioned WCU 상시 할당
- **해결**: On-Demand 기본 + EventBridge로 이벤트 직전만 Provisioned 자동 전환
- **결과**: 평시 과금을 사용량 기반으로 전환

---

## 부하 테스트 결과

k6 기반 단계별 부하 테스트 (100 → 500 → 1,000 → 2,000 → 5,000 → 8,000 → 10,000)

| 단계 | 결과 |
|---|---|
| 100명 | 성공률 100%, p95 1.97초 |
| 500명 | 성공률 100%, p95 181ms (4-Layer 캐싱 적용 후 30초 → 21ms) |
| 1,000명 (1인 1회) | 성공률 100%, p95 116ms |
| 2,000명 (1인 1회) | 성공률 100%, p95 181ms, throttle 0 |
| 8,000명 burst | 7,998/8,000 iter, 에러 0건, throttle 안정화 |
| **10,000명 순간 폭증** | **VWR 흡수 + Tier2 admissionRate 제어로 EKS 정상 처리** |

---

## 기술 스택 (인프라 관점)

| 카테고리 | 기술 |
|---|---|
| **AWS** | EKS, Karpenter, Lambda, API Gateway, RDS, ElastiCache, DynamoDB, S3, SQS, CloudFront, WAFv2, ACM, Route 53, VPC Endpoint, Secrets Manager, SSM Parameter Store |
| **Container/Orchestration** | Docker, Kubernetes 1.33, Kustomize, External Secrets Operator |
| **CI/CD** | GitHub Actions, OIDC, ECR, GitOps |
| **IaC** | Terraform, Kustomize |
| **Observability** | Prometheus, Grafana, Loki, CloudWatch |
| **Load Testing** | k6 |

---

## 기간·역할

- **기간**: 2026.02 ~ 2026.04 (약 3개월)
- **팀 규모**: 10명 이상 (Frontend / Backend / Infra 분담)
- **본인 역할**: **인프라 단독 담당** (위 표 참조)

---

## 부트캠프 3단계 진행 흐름

| 단계 | 프로젝트 | 본인 인프라 담당 |
|---|---|---|
| 1단계 (기본) | [tiketi](https://github.com/cchriscode/tiketi) | Docker 기반 운영 환경, Redis FIFO 대기열, WebSocket 멀티 인스턴스 동기화 |
| 2단계 (심화) | (urr 초기 버전) | kind 클러스터 K8s 매니페스트, GitOps 기초, 마이크로서비스 네트워크 격리 |
| 3단계 (실무) | **URR (본 레포)** | EKS·Lambda·GitOps·보안·모니터링·비용 최적화 전 영역 |

---

## 사이드 프로젝트

부트캠프 기간 중 별도로 만든 학습용 프로젝트들:

- **[aws-architect-flow](https://github.com/cchriscode/aws-architect-flow)** — AWS Well-Architected Framework 학습 + 14단계 아키텍처 설계 위저드
- **[hiveops](https://github.com/cchriscode/hiveops)** — MCP 기반 AI Agent Kanban Board
- **[prom-market](https://github.com/cchriscode/prom-market)** — AI 프롬프트 마켓플레이스 (Next.js + Prisma)
- **[k8s-challenges](https://github.com/cchriscode/k8s-challenges) / [k8s-gitops-demo](https://github.com/cchriscode/k8s-gitops-demo)** — Kubernetes/GitOps 학습용

---

## 클론 방법

```bash
git clone --recurse-submodules https://github.com/cchriscode/urr-portfolio.git
cd urr-portfolio
git submodule update --init --recursive
```

---

## 참고 문서

- [docs/architecture.md](./docs/architecture.md) — 상세 아키텍처
- [docs/troubleshooting.md](./docs/troubleshooting.md) — 트러블슈팅 모음
- [docs/cost-optimization.md](./docs/cost-optimization.md) — 비용 최적화 사례

---

## License

본 포트폴리오 자체는 학습/이력서용 정리 목적으로 공개합니다.
원본 코드는 KT TechUp 조직(`KTCloud-TechUp`) 산하의 각 레포 라이선스를 따릅니다.
