# 🍚 밥풀 BobFull

> **혼자여도 함께 먹을 수 있도록**  
> 개인 단위로 좌석을 예약하고, 다른 사용자와 함께 식사할 수 있도록 연결하는 **합석형 좌석 예약 플랫폼**입니다.

[📚 Technical Docs](https://github.com/bobfull-project/bobfull-docs) · [🔬 Flow Lab](https://bobfull-project.github.io/bobfull-docs/flow-lab/v3/operations-flow-lab/) · [⚙️ Backend](https://github.com/bobfull-project/bobfull-backend) · [🖥️ Frontend](https://github.com/bobfull-project/bobfull-frontend)

**🍴 식당 탐색 → 👥 회차·인원 선택 → 💳 예약·결제 → ✅ 합석 성사 → 💬 채팅·식사**

> **[GIF 첨부 예정 — 서비스 핵심 흐름]**

---

## 🍽️ 서비스 소개

혼자 여행하거나 식사할 때 겪는 `2인 이상 주문`, `1인 예약 제한`, 함께 식사할 사람을 직접 구해야 하는 불편을 해결합니다.

사용자는 원하는 식당과 회차를 선택해 **개인 단위로 좌석을 예약**하고, 최소 성사 인원이 모이면 합석이 성사되어 채팅 후 함께 식사할 수 있습니다.

---

## ✨ 핵심 기능

- 🍴 식당·합석 회차 탐색
- 👤 개인 단위 좌석 예약·잔여 좌석 관리
- 💳 PortOne 예약금 결제·취소·환불
- 🔒 동시 예약 상황의 좌석 정합성 보장
- 💬 다중 App 환경 실시간 채팅
- 🛡️ AI 채팅 위험 메시지 검수
- 🍽️ AI 식당 피드백 분석·익명 집계
- 📊 성능·장애·운영 검증 기반 시스템 고도화

---

## 🏗️ Engineering Highlights

**🔒 예약·결제 정합성**  
좌석 예약은 동시성 제어로 초과 예약을 막고, 결제는 외부 검증과 내부 상태 전이를 분리해 중복·역전 전이를 방지했습니다.

**📨 비동기 신뢰성**  
ChatRoom·Email은 **Transactional Outbox**로 처리하고, AI 후속 처리는 동일 Outbox 조건에서 Async와 Kafka를 비교했습니다. Async가 더 빨랐지만(`5.394s` vs `7.210s`), 적체·복구·독립 확장을 분리할 운영 경계가 필요한 AI 작업에만 **Outbox + Kafka**를 적용했습니다.

**🤖 AI 채팅 검수 · 식당 피드백 분석**  
명확한 위험 메시지는 규칙으로 빠르게 걸러내고, 판단이 필요한 메시지만 LLM으로 분석합니다. 음식·서비스·가격·청결 관련 의견은 개인 식별정보 없이 구조화·집계해 사장님용 피드백 인사이트로 제공합니다.

**📊 측정 기반 의사결정**  
Redis Cache·Query/Index·Kafka·App 확장 여부를 k6, 통합 테스트, 실제 AWS 환경의 측정 결과를 바탕으로 판단했습니다.

---

## 🗺️ System Architecture

AWS 기반 다중 App 구조와 Blue-Green 배포를 구성하고, 데이터 저장소·메시징·모니터링·외부 결제/AI 연동의 책임을 분리했습니다.

<img width="1642" height="952" alt="시스템 아키텍처" src="https://github.com/user-attachments/assets/abe4f7a8-37cd-4365-bb0a-62620a4728bf" />


> 평시에는 Blue/Green 중 **Active 환경의 App EC2 2대만 서비스**하고, 배포 시 Inactive 환경을 기동·검증한 뒤 ALB Traffic Weight를 전환합니다.  
> [▶ 상세 System Architecture 보기](https://github.com/bobfull-project/bobfull-docs/blob/main/architecture/system-architecture.md)

---

## ✅ 구현·검증 현황

문서에 존재하거나 설계가 완료됐다는 이유만으로 **구현·배포·실측 완료로 표시하지 않습니다.** 실제 코드와 Evidence를 기준으로 상태를 구분합니다.

| 영역 | 상태 | 확인 범위 |
|---|---|---|
| 예약·환불·Outbox 동시성 방어 | ✅ 구현·검증 | 비관적 락·조건부 UPDATE·낙관락·Outbox claim 검증 |
| Redis 식당 검색 Cache | 📊 구현·실측 | Warm Cache의 DB 조회·Hikari Pool 점유 감소 실측 |
| Query / Index 최적화 | 📊 구현·실측 | 검색·예약 조회·지급 예정금 조회 Hot-path 개선 실측 |
| Transactional Outbox | ✅ 구현·검증 | ChatRoom·Email·Kafka 발행 의도 보존과 실패 재처리 검증 |
| Kafka AI 후속 처리 | ✅ 구현·검증 | Retry / DLT·Consumer 중단 복구·중복 방어 검증 |
| AI Moderation | 📊 구현·실측 | Regression·Held-out 데이터셋 기반 품질 측정 |
| App HA / Blue-Green | 📊 구현·실측 | ALB + App EC2 2대, Traffic Switch·Rollback 실측 |
| Auto Scaling | ⚪ 실측 후 미도입 | 현재 부하에서는 App CPU보다 DB Pool 대기가 먼저 발생해 미채택 |
| AI Worker / MSA 분리 | ⚪ 실측 후 미도입 | AI 지연 격리·복구를 확인한 뒤 통합 모놀리스 유지 |

> ✅ `구현·검증` · 📊 `구현·실측` · ⚪ `실측 후 미도입`  
> 상세 수치·실험 조건·한계는 [V3 Final Claim Matrix](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/evidence/v3/FINAL_CLAIM_MATRIX.md)에서 확인할 수 있습니다.

---

## 🛠️ Tech Stack

| 영역 | 기술 / 버전 |
|---|---|
| Backend | Java **17** · Spring Boot **4.1.0** · Spring AI **2.0.0** · QueryDSL **5.1.0** · Gradle **9.5.1** |
| Data | MySQL **8.4** (Local) · Amazon RDS for MySQL **8.0.46 (db.t4g.micro, Production)** · Redis **7-alpine** (Local) · Amazon ElastiCache for Valkey **9.1.0 (cache.t4g.micro, Production)** |
| Messaging | Apache Kafka **3.9.0** (Local) · KRaft Broker on EC2 **t3.small** (Production) |
| Frontend | React **19.2.0** · TypeScript **5.9.3** · Vite **7.3.1** · Tailwind CSS **3.4.17** |
| Payment | PortOne Server SDK **0.24.0** |
| Infra / CI·CD | AWS EC2 **t3.small** · ALB · RDS · S3 · CloudFront · Route 53 · Lambda · ECR · SSM · Docker **25.0.14** · GitHub Actions · Blue-Green Deployment |
| Monitoring | Prometheus **3.13.2** · Grafana **13.0.2** · AWS CloudWatch |
| Test | JUnit · Testcontainers · k6 |

> 버전은 프로젝트의 실제 빌드·실행 설정 기준이며, AWS 관리형 서비스는 실제 배포 환경을 기준으로 표기합니다.

---

## 🔬 Flow Lab

복잡한 백엔드 흐름을 문서로만 설명하지 않고 **Chapter · Scenario · Step** 단위로 시각화해, 주요 상태 전이와 비동기 후속 처리를 직접 따라가며 확인할 수 있도록 만들었습니다.

[▶ Flow Lab에서 시스템 흐름 직접 보기](https://bobfull-project.github.io/bobfull-docs/flow-lab/v3/operations-flow-lab/)

<details>
<summary><b>💳 결제 장애 격리 Flow Lab GIF</b></summary>

<br>

<img width="900" height="422" alt="결제 이후 후속처리 흐름" src="https://github.com/user-attachments/assets/62524185-b285-47d1-a0df-9e984725b754" />

</details>

---

## 🔎 더 알아보기

| 구분 | 내용 |
|---|---|
| [📚 Technical Docs](https://github.com/bobfull-project/bobfull-docs) | Architecture · API · ERD · ADR · Troubleshooting |
| [🔬 Flow Lab](https://bobfull-project.github.io/bobfull-docs/flow-lab/v3/operations-flow-lab/) | 주요 백엔드 시스템 흐름 인터랙티브 확인 |
| [⚙️ Backend](https://github.com/bobfull-project/bobfull-backend) | Spring Boot 구현 · 정책 · Evidence · 상세 기술 문서 |
| [🖥️ Frontend](https://github.com/bobfull-project/bobfull-frontend) | React 클라이언트 구현 · 실행 · 배포 |

---

## 👥 Team 밥조

| 이름 | 주요 담당 |
|---|---|
| **김현승** | 결제 · 환불 · 정산 · AI · 시스템 신뢰성 |
| **배지현** | 예약 · 좌석 동시성 · Frontend |
| **정용태** | 회원 · 인증 · 식당 · 관리자 · 조회 성능 |
| **김홍기** | 합석 · 회차 · 검색 · 배포 · 모니터링 |

**Project · 2026.07.21 ~ 2026.08.24**
