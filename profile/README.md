# 🍚 밥풀 BobFull

> **제주에서 혼자 온 손님과 비어 있는 좌석을 연결하는 좌석 단위 합석 예약 플랫폼**

BobFull은 1인 방문이 어려운 제주 식당에서 사용자가 좌석 단위로 합석을 모집하고, 예약금 결제 후 함께 식사할 참여자와 연결되도록 돕는 서비스입니다.

[📚 Technical Docs](https://github.com/bobfull-project/bobfull-docs) · [🔬 Flow Lab](https://bobfull-project.github.io/bobfull-docs/flow-lab/v3/operations-flow-lab/) · [⚙️ Backend](https://github.com/bobfull-project/bobfull-backend) · [🖥️ Frontend](https://github.com/bobfull-project/bobfull-frontend)

```text
식당·회차 탐색 → 모임 생성·참여 → 예약금 결제 → 합석 확정 → 참여자 채팅
```

---

## 🍽️ 문제 정의와 해결

| 대상 | 문제 | BobFull의 접근 |
|---|---|---|
| 사용자 | 혼자 방문하기 부담스러운 식당, 2인 이상 주문 제약, 함께 식사할 사람을 직접 찾아야 하는 불편 | 좌석 단위로 합석을 모집하고 예약금 결제 후 참여자 채팅으로 연결 |
| 식당 | 테이블 단위 운영으로 남는 좌석, 취소·노쇼, 낮은 좌석 활용도 | 빈 좌석을 개인 단위 합석 수요와 연결해 좌석 활용도를 높임 |

---

## ✨ 핵심 기능

- **좌석 단위 합석 예약**: 식당 회차와 테이블 정원 안에서 개인 단위로 모임을 만들거나 참여합니다.
- **모집 · 결제 · 환불**: PortOne 예약금 결제, 모집 마감, 취소·환불 상태를 서버 검증 흐름으로 관리합니다.
- **다중 App 실시간 그룹 채팅**: 결제 후 생성된 채팅방에서 여러 App EC2에 붙은 참여자도 같은 방 메시지를 주고받습니다.
- **AI 안전 관리 · 식당 피드백 분석**: 위험 메시지를 검수하고, 식당 운영자가 볼 수 있는 익명 피드백 인사이트를 구조화합니다.

---

## 🧠 핵심 기술과 의사결정

| 문제 | 선택 | 검증 |
|---|---|---|
| 좌석 경쟁으로 초과 예약이 발생할 수 있음 | `TimeSlot`/`Reservation` 비관적 락 + 최신 상태 재검증 | 예약 동시성 MySQL 3/3 PASS, 보호장치 제거 시 실패 재현 |
| 결제 완료 뒤 후속 작업 실패가 핵심 거래를 흔들 수 있음 | 핵심 트랜잭션에 `OutboxEvent(PENDING)` 함께 저장 | ChatRoom·Email 재처리, 중복 방어, 장시간 처리 중 상태 회수 검증 |
| AI 지연·실패가 채팅 저장 경로에 전파될 수 있음 | AI 후속 처리에만 Outbox + Kafka Consumer / Retry / DLT 적용 | 발행 실패 복구, 반복 실패 DLT, 중복 이벤트 멱등 처리 검증 |
| 다중 App에서 로컬 STOMP만으로는 원격 인스턴스 세션에 닿지 않음 | DB 원본 저장 + WebSocket/STOMP + Redis Pub/Sub 전파 | App A/B 양방향 전달, ChatMessage 단일 저장, DB 기반 누락 메시지 복구 검증 |
| AI 검수의 비용·지연과 경계 오탐을 함께 관리해야 함 | 규칙 기반 빠른 경로 + LLM Structured Output, 경계 사례는 범위 한계로 기록 | 미사용 검증 데이터 74/80(92.5%), 경계 사례에서 과탐 확인 |

---

## 📊 대표 실측 결과

| 검증 | 실측 결과 | 판단 |
|---|---|---|
| 인기 회차 핵심 조회 경로 | 쿼리 `83 → 7`, 고부하 단계 p95 `13.14s → 1.34s` | 반복 쿼리를 배치 조회로 줄여 병목 임계점을 뒤로 이동. 최고 부하 단계에서는 CPU/HikariCP 포화가 남음 |
| 정산 조회 | p95 `6.5s → 30.32ms`, Hikari 대기 `92 → 0` | Batch/Snapshot 저장 없이 쿼리·인덱스 개선 유지 |
| 반복 식당 검색 Redis Cache | p95 `43ms → 14ms`, DB 쿼리 `2 → 0` | 정합성 영향이 낮은 반복 검색에 제한 적용, Redis 장애 시 DB 조회로 대체 |
| AI 검수 Kafka 파티션 키 | 전체 처리 완료 시간 `15.616s → 7.271s`, 활성 파티션 `1 → 3` | 방 순서 계약이 없는 AI 검수는 `messageId` 키로 병렬성 확보 |
| Blue-Green 배포 | 외부 요청 검증 `2,787 / 2,787` HTTP 200, 실패 `0`, 관측 다운타임 `0s` | 애플리케이션 계층 기준 ALB + Active App EC2 2대와 트래픽 전환 검증 |
| AI Consumer 중단·복구 | Consumer 중단 후 `15 / 15` 복구, 유실 `0` | 별도 Worker 분리 없이 통합 Spring Boot 내부 Consumer 유지 |
| Outbox + Async / Kafka 비교 | Async 전체 처리 완료 시간 `5.394s`, Kafka 전체 처리 완료 시간 `7.210s`, 유실/중복 `0 / 0` | Kafka는 더 빨라서가 아니라 운영·복구·격리 경계 때문에 AI에 한정 유지 |

상세 조건과 한계는 [V3 Final Claim Matrix](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/evidence/v3/FINAL_CLAIM_MATRIX.md)와 각 Evidence 문서가 기준입니다.

---

## 🏗️ 시스템 아키텍처

AWS 기반으로 프론트엔드 전달 경로, Blue-Green App, 데이터 저장소, Kafka, 모니터링, CI/CD, 외부 결제·AI 연동의 책임을 분리했습니다.

<img width="1642" height="952" alt="시스템 아키텍처" src="https://github.com/user-attachments/assets/abe4f7a8-37cd-4365-bb0a-62620a4728bf" />

- **애플리케이션 계층**: ALB 뒤 Blue/Green Target Group을 두고, 평시에는 Active App EC2 2대만 서비스합니다.
- **RDS**: 현재 Single-AZ MySQL 구성입니다.
- **Kafka**: 전용 EC2의 Single KRaft Broker 구성이며, Kafka 계층 HA까지 보장하지 않습니다.
- **Redis / Valkey**: 인증 상태, 검색 Cache, 다중 App 채팅 Pub/Sub의 공용 Redis 호환 저장소입니다.

[▶ 상세 시스템 아키텍처 보기](https://github.com/bobfull-project/bobfull-docs/blob/main/architecture/system-architecture.md)

---

## 🚀 배포 및 운영

백엔드 배포는 새 버전을 운영 트래픽에 넣기 전에 비활성 환경에서 먼저 검증하고, 실패하면 기존 Listener 상태로 되돌리는 흐름으로 구성했습니다.

```text
GitHub Actions
→ ECR
→ SSM
→ 비활성 App EC2 2대 배포
→ Readiness / Target Group Health Check
→ ALB 트래픽 전환
→ 외부 요청 검증
  ├─ 성공 → 모니터링 대상 갱신 → Prometheus 신규 Active 2대 UP 확인
  └─ 실패 → 기존 환경으로 롤백
```

운영 관측은 `/actuator/prometheus`를 Prometheus가 수집하고, Grafana Alerting이 Slack으로 알림을 보내며, CloudWatch Logs와 애플리케이션 구조화 로그로 원인을 추적하는 흐름입니다.

- 배포 실패는 SSM, Docker, Target Group Health, 외부 요청 검증 단계에서 중단하거나 롤백합니다.
- App 장애 원인은 Docker/CloudWatch Logs와 Health Check / Readiness 결과로 확인합니다.
- 결제 상태 불일치나 보상 필요 상태는 `PAYMENT_COMPENSATION_REQUIRED` 같은 구조화 로그로 추적할 수 있게 했습니다.
- 비정상 요청과 API 처리 시간, 오류 코드는 운영 로그와 Prometheus/Grafana 지표를 함께 보며 감지 → 알림 → 로그/지표 확인 → 원인 추적으로 연결합니다.

---

## ✅ 구현·검증 현황

현재 구현과 검증 범위를 실제 코드와 Evidence 기준으로 요약합니다.

| 영역 | 상태 | 확인 범위 |
|---|---|---|
| 예약·환불·Outbox 동시성 방어 | ✅ 구현·검증 | 비관적 락·조건부 UPDATE·낙관락·Outbox claim 검증 |
| Redis 식당 검색 Cache | 📊 구현·실측 | Warm Cache의 DB 조회·Hikari Pool 점유 감소 실측 |
| 쿼리 / 인덱스 최적화 | 📊 구현·실측 | 인기 회차 조회·검색·정산 조회 핵심 조회 경로 개선 실측 |
| Transactional Outbox | ✅ 구현·검증 | ChatRoom·Email·Kafka 발행 의도 보존과 실패 재처리 검증 |
| Kafka AI 후속 처리 | ✅ 구현·검증 | Retry / DLT·Consumer 중단 복구·중복 방어 검증 |
| WebSocket/STOMP + Redis Pub/Sub 채팅 | ✅ 구현·검증 | 다중 App 인스턴스 간 전달, 방별 local STOMP 격리, DB cursor 복구 경계 |
| AI 검수 | 📊 구현·실측 | 규칙 기반 빠른 경로·LLM Structured Output·미사용 검증 데이터 기반 품질 측정 |
| 애플리케이션 계층 이중화 / Blue-Green | 📊 구현·실측 | ALB + Active App EC2 2대, 트래픽 전환·롤백 실측 |
| Auto Scaling | ⚪ 실측 후 미도입 | 단일 App 고부하의 CPU/Pool 포화와 Active App 2대 조건의 Hikari 대기를 구분해 ASG/스케일링 정책 미채택 |
| AI Worker / MSA 분리 | ⚪ 실측 후 미도입 | AI 지연·Consumer 중단 복구를 확인한 뒤 통합 모놀리스 내부 Consumer 유지 |

> ✅ `구현·검증` · 📊 `구현·실측` · ⚪ `실측 후 미도입`

---

## 🤝 AI-Native 개발 방식

BobFull은 AI를 Issue 정리, 구현 보조, 테스트·리뷰 체크리스트 생성에 활용했지만, **AI 제안이 곧 최종 결정은 아니었습니다.**

```text
Issue → AI Implementation → Human Review → PR Checklist → Feedback → Merge
```

최종 의도, 데이터 정합성, 권한·보안, 트랜잭션 경계, Trade-off는 사람이 검증하고 문서화했습니다.

[AI Workflow 보기](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/AI_WORKFLOW.md)

---

## 🛠️ 기술 스택

| 영역 | 기술 / 버전 |
|---|---|
| Backend | Java **17** · Spring Boot **4.1.0** · Spring AI **2.0.0** · QueryDSL **5.1.0** · Gradle **9.5.1** |
| Data | MySQL **8.4** (Local) · Amazon RDS for MySQL **8.0.46 (db.t4g.micro, Production)** · Redis **7-alpine** (Local) · Amazon ElastiCache for Valkey **9.1.0 (cache.t4g.micro, Production)** |
| Messaging | Apache Kafka **3.9.0** (Local) · KRaft Broker on EC2 **t3.small** (Production) |
| Frontend | React **19.2.0** · TypeScript **5.9.3** · Vite **7.3.1** · Tailwind CSS **3.4.17** |
| Payment | PortOne Server SDK **0.24.0** |
| Infra / CI·CD | AWS EC2 **t3.small** · ALB · ACM · RDS · S3 · CloudFront · Route 53 · Lambda · ECR · SSM · Docker **25.0.14** · GitHub Actions · Blue-Green Deployment |
| Monitoring | Prometheus **3.13.2** · Grafana **13.0.2** · Grafana Alerting · Slack Alert · AWS CloudWatch |
| Test | JUnit · Testcontainers · k6 |

> 버전은 프로젝트의 실제 빌드·실행 설정 기준이며, AWS 관리형 서비스와 외부 알림 채널에는 임의 버전을 붙이지 않습니다.

---

## 📚 기술 문서 & Flow Lab

세부 구현 조건, 장애 시나리오, 실험 원본 결과는 아래 문서에서 확인할 수 있습니다.

| 구분 | 내용 |
|---|---|
| [📚 Technical Docs](https://github.com/bobfull-project/bobfull-docs) | Architecture · API · ERD · ADR · Performance · Troubleshooting |
| [🔬 Flow Lab](https://bobfull-project.github.io/bobfull-docs/flow-lab/v3/operations-flow-lab/) | 정상 / 실패 / 복구 흐름을 Chapter · Scenario · Step 단위로 확인 |
| [⚙️ Backend](https://github.com/bobfull-project/bobfull-backend) | Spring Boot 구현 · 정책 · 검증 근거 |
| [🖥️ Frontend](https://github.com/bobfull-project/bobfull-frontend) | React 클라이언트 구현 · 실행 · 배포 |

<details>
<summary><b>💳 결제 이후 후속 처리 Flow Lab GIF</b></summary>

<br>

<img width="900" height="422" alt="결제 이후 후속처리 흐름" src="https://github.com/user-attachments/assets/62524185-b285-47d1-a0df-9e984725b754" />

</details>

---

## 👥 Team 밥조

| 이름 | 주요 담당 |
|---|---|
| **김현승** | 결제 · 환불 · 정산 조회 · AI · 채팅 시스템 |
| **배지현** | 예약 · 좌석 동시성 · Frontend |
| **정용태** | 회원 · 인증 · 식당 · 관리자 · 조회 성능 |
| **김홍기** | 합석 · 회차 · 검색 · AWS 인프라 · 배포 · 모니터링 |

**Project · 2026.07.21 ~ 2026.08.24**