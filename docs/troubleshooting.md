# 트러블슈팅 모음 (인프라 영역)

## TS-1. Lambda Throttle 134건 → 0건

**증상**: 2,000명 부하 테스트 중 Lambda 동시 실행 quota(1,000) 초과로 throttle 134건 발생

**분석**: CloudWatch Concurrent Executions 메트릭을 호출 경로별로 펼쳐보니, **API Gateway Authorizer Lambda + VWR-API Lambda**가 동일 요청에서 동시 실행되어 카운트가 두 배가 됨

**해결**:
- VWR `assign` 경로만 `Auth:NONE`으로 분리
- VWR-API Lambda 내부에서 JWT 직접 검증

**결과**: 동시 실행 절반 → throttle 안정화, p95 181ms 유지

---

## TS-2. EKS AMI v1.33.8 kubelet 크래시

**증상**: 부하 테스트 중 노드의 kubelet이 알 수 없이 죽음 → Pod NodeStatusUnknown 반복

**분석**: 워크로드 문제로 의심해 Pod 리소스 요청량·probe 설정·HPA 등을 한참 조정했지만 미해결 (약 4시간 허비)

**해결**: AWS 측 v1.33.8 AMI의 알려진 kubelet 버그 확인 → `aws eks update-nodegroup-version` 으로 v1.33.10 업데이트 → 5분 내 정상화

**개선**: 장애 발생 5분 안에 **코드와 인프라 양쪽 모두 의심 리스트에 올리는 체크리스트** 도입

---

## TS-3. NODE_ENV dimension 불일치 (silent bug)

**증상**: 프로덕션 환경에서 CloudWatch Adapter가 메트릭을 못 가져옴. 알람은 안 울림

**분석**:
- counter-advancer Lambda: `NODE_ENV = "production"` 으로 메트릭 publish (Terraform 설정)
- CloudWatch Adapter `external-metric.yaml`: `value: dev` 하드코딩
- Dimension 불일치 → 메트릭 매칭 실패 → 그러나 200 OK라 알람도 안 울림

**해결**: 양쪽 dimension을 `production`으로 일치시킴

**의의**: 메트릭 흐름을 직접 손으로 따라가지 않으면 절대 못 찾는 종류의 silent bug. 이런 미세 버그가 운영에서 가장 위험함

---

## TS-4. 좌석 조회 30초 타임아웃 → 21ms

**증상**: 500명 동시 좌석 조회 시 응답 30초 초과로 다수 실패

**원인**: 14,693석 × 500명 = 단일 응답 ~2MB × 500 동시 요청 → DB·네트워크·렌더링 모두 병목

**해결**: 4-Layer 캐싱 도입
| Layer | 대상 | 기술 |
|---|---|---|
| L1 | 좌석 카탈로그 (정적) | @Cacheable 인메모리 |
| L2 | 좌석 조회 (섹션별) | ConcurrentMapCacheManager |
| L3 | 좌석 상태 (실시간) | Redis Hash, TTL 24h |
| L4 | VWR 카운터 | Lambda 메모리, TTL 10초 |

+ **섹션별 필터링 API** (응답 1.5MB → 72KB)

**결과**: 평균 457ms → 21ms (약 22배 개선), 성공률 100%

---

## TS-5. RDS HikariCP 풀 고갈

**증상**: 500명 동시 조회 시 HikariCP 커넥션 부족으로 timeout

**분석**: Pod마다 풀을 크게 잡으면서 RDS Proxy의 멀티플렉싱 역할을 제대로 활용 못 함

**해결**: Pod 풀을 10으로 축소 → RDS Proxy가 커넥션 공유 담당

**결과**: 동일 RDS 인스턴스로 2,000명까지 커버

---

## TS-6. CloudWatch 메트릭 파이프라인 ~2~3분 → ~36초

**문제**: 오토스케일링 반응이 너무 느려 burst를 놓침

**분석**: 메트릭 파이프라인 단계별 지연 측정
- CloudWatch publish: ~10초 (counter-advancer 주기)
- Adapter 폴링: ~60초 (default)
- HPA stabilizationWindow: ~30초 (default)
- Pod ready: ~30~60초

**해결 (4가지 동시 조정)**:
1. CloudWatch 고해상도 메트릭 (StorageResolution: 1) — 60초 → 1초 해상도
2. Adapter 폴링 주기 60 → 10초
3. HPA stabilizationWindow 30 → 10초
4. (추가 비용 검증) 월 $2~3 수준 확인 후 도입

**결과**: 메트릭 → HPA 결정까지 약 36초로 단축

---

## TS-7. DynamoDB 4월 청구서 약 $92

**증상**: 4월 비용 청구서 분석 중 DynamoDB 항목이 예상보다 큼

**원인**: Provisioned WCU 상시 할당 → 평시엔 거의 idle인데 계속 과금

**해결**:
- 평시: On-Demand 모드로 전환 (사용량 기반 과금)
- 이벤트 직전: EventBridge → Lambda → `UpdateTable` API로 Provisioned 자동 전환
- 이벤트 종료 후: 다시 Lambda로 On-Demand 복원

**결과**: 평시 과금을 사용량 기반으로 전환

---

## TS-8. NAT Gateway 트래픽 비용 절감

**문제**: AWS API 호출 (ECR 이미지 Pull, S3 업로드, STS, SSM)이 전부 NAT Gateway 경유 → 트래픽 비용 누적

**해결**: VPC Interface Endpoint 도입
- ECR API / ECR DKR (이미지 Pull 트래픽 큼)
- STS (IRSA 토큰 발급)
- SSM (Parameter Store 조회)
- S3 Gateway Endpoint (무료)

**결과**: 큰 트래픽이 VPC 내부 경로로 전환되어 NAT 비용 절감

---

## TS-9. 양도 등급 차단 메시지가 토스트로 안 떴음

**증상**: 미스트 등급 사용자가 양도 등록 시 백엔드는 "TIER_CANNOT_SELL" 반환하지만, 프론트는 막연히 "양도 등록 실패"만 표시

**원인**: ApiError가 백엔드 message 필드를 보유하지 않음 → 프론트에서 백엔드 메시지 노출 불가

**해결 (프론트팀 협업)**: ApiError에 `message`, `body` 추가 → mutation `onError`에서 toast.error로 백엔드 메시지 그대로 노출

**결과**: 사용자가 "미스트 등급은 양도 게시글 등록이 불가능합니다. 클라우드 이상만 이용 가능합니다." 메시지를 직접 볼 수 있음
