# CompensaTX – 보상 트랜잭션 기반 분산 환전 시스템

## 소개

**CompensaTX**는 완전히 분리된 원화/외화 계좌 시스템(각기 다른 DB, 서버) 환경에서  
안전하고 강인하게 환전·이체를 처리할 수 있는 **Saga(보상 트랜잭션) 패턴 기반 분산 트랜잭션 시스템**입니다.

- 장애/네트워크 오류 발생 시에도 데이터 정합성을 보장
- 자동 복구 및 수동 개입 모두 지원
- 실제 토스뱅크의 분산 환전 구조를 벤치마킹하여 설계

---

## **👥 팀원 소개**

| <img width="150px" src="https://avatars.githubusercontent.com/u/52108628?v=4"/> | <img width="150px" src="https://avatars.githubusercontent.com/u/76148385?v=4"/> | <img width="150px" src="https://avatars.githubusercontent.com/u/56614731?v=4"/> | <img width="150px" src="https://avatars.githubusercontent.com/u/87272634?v=4"/> |
| --- | --- | --- | --- |
| **고태우** | **이제현** | **이용훈** | **장송화** |
| [@kohtaewoo](https://github.com/kohtaewoo) | [@lyjh98](https://github.com/lyjh98) | [@dldydgns](https://github.com/dldydgns) | [@songhajang](https://github.com/songhajang) |
| Kafka consumer | 외화(OEHWA) 계좌 API 서버 | 오케스트레이션 API 서버 | 프론트엔드 UI <br> 원화(KRW) 계좌 API 서버 |


## 📂 레포지토리 구조

| 레포 | 설명 | 링크 |
|------|------|------|
| **compensaTX-BE** | Backend API 서버 (오케스트레이션, 트랜잭션 흐름 제어) | [🔗 바로가기](https://github.com/Woori-compensaTX/compensaTX-BE) |
| **compensaTX-FE** | Frontend (Vue.js 기반 UI) | [🔗 바로가기](https://github.com/Woori-compensaTX/compensaTX-FE) |
| **compensaTX-api-krw** | 원화(KRW) 계좌 API 서버 (Oracle 기반) | [🔗 바로가기](https://github.com/Woori-compensaTX/compensaTX-api-krw) |
| **compensaTX-api-oehwa** | 외화(OEHW) 계좌 API 서버 (MySQL 기반) | [🔗 바로가기](https://github.com/Woori-compensaTX/compensaTX-api-oehwa) |
| **compensaTX-consumer-krw** | 원화(KRW) Kafka 컨슈머 (고가용성 인스턴스 2개) | [🔗 바로가기](https://github.com/Woori-compensaTX/compensaTX-consumer-krw) |
| **compensaTX-consumer-oehwa** | 외화(OEHW) Kafka 컨슈머 (고가용성 인스턴스 2개) | [🔗 바로가기](https://github.com/Woori-compensaTX/compensaTX-consumer-oehwa) |
| **compensaTX-consumer-dlq** | DLQ 전용 Kafka 컨슈머 (보상 트랜잭션 재시도/복구) | [🔗 바로가기](https://github.com/Woori-compensaTX/compensaTX-consumer-dlq) |
| **.github** | GitHub Actions 및 워크플로 설정 저장소 | [🔗 바로가기](https://github.com/Woori-compensaTX/.github) |

---

## 아키텍처

```
[사용자]
│
▼
[환전 서버 (Orchestrator)]
├─(HTTP 동기)→ [원화 계좌 서버/DB]
└─(HTTP 동기)→ [외화 계좌 서버/DB]
│
└─(Kafka 메시지, DLQ, Scheduler등: 보상 트랜잭션/재시도/지연처리)
```
- **환전 서버**: 전체 환전 상태 관리, 오케스트레이터 역할  
- **원화/외화 계좌 서버**: 각 계좌별 입출금 및 DB 관리  
- **Kafka/DLQ/Scheduler**: 장애 시 보상 트랜잭션, 자동 재시도, 딜레이 메시지, 배치 복구 담당  

---

## 🏗️ 분산환경에서의 트랜잭션

### 기존 모놀리식 방식

- 단일 서버, 단일 DB → DB 트랜잭션만으로 원자성 보장

### MSA/분산 환경 문제

- 원화/외화 계좌 서버와 DB가 분리 → **단일 트랜잭션 적용 불가**
- 입출금 중 한쪽만 성공할 경우 **데이터 불일치(정합성 문제) 발생**

---

## 💡 분산 트랜잭션 관리 방식

### 1. **2PC (Two-Phase Commit)**
- 코디네이터가 각 서버의 커밋 가능여부를 질의, 모두 OK 시 커밋, 아니면 롤백
- **문제점**:  
  - 느린 참여자에 의해 전체 지연  
  - 가용성과 확장성 저하  
  - 단일 장애점 위험

### 2. **Saga(보상 트랜잭션) 패턴**
- 각 서비스가 **로컬 트랜잭션**을 개별 커밋, 실패 시 이미 커밋된 부분을 **보상 트랜잭션**으로 롤백
- **장점**:  
  - 높은 확장성, 가용성  
  - 새로운 서비스(카드, 회계 등)도 참여 용이
- **단점**:  
  - 중간상태 노출  
  - 보상 트랜잭션 직접 구현 필요

---

## 🎵 Saga 방식 선택과 구조

- **Saga 패턴 중 ‘오케스트레이션 방식’** 선택
  - 오케스트레이터(환전 서버)가 전체 플로우와 상태 추적
  - 환전 한도, 현재 진행 상태 등 관리 용이


- **환전 서버**: 전체 환전 상태 관리, 오케스트레이터 역할  
- **원화/외화 계좌 서버**: 각 계좌별 입출금 및 DB 관리  
- **Kafka/DLQ/Scheduler/Batch**: 장애 시 보상 트랜잭션, 자동 재시도, 딜레이 메시지, 배치 복구 담당  

---

## 주요 특징

- **Saga(Orchestration) 패턴 기반** 분산 트랜잭션 구현  
- **출금 → 입금 순**의 원자적 환전 및 보상 트랜잭션 설계  
- **Kafka DLQ 및 Scheduler**로 장애/지연 상황 자동 복구  
- **State Table(상태 로그)**로 전체 이력 및 상태 추적  
- **Batch/Alert**를 통한 미처리 환전 자동/수동 복구  
- **Loose Coupling**: 신규 서비스/계좌/참여자 손쉬운 확장  

---

## 트랜잭션/보상 트랜잭션 플로우

### 1. 정상 플로우

1. 환전 서버 → **원화 출금 요청** (HTTP)
2. 출금 성공 → **외화 입금 요청** (HTTP)
3. 입금 성공 → **환전 성공 처리**

### 2. 실패 플로우

- 출금 실패 → **즉시 환전 실패 처리**
- 입금 실패 → **보상 트랜잭션(출금 취소) 메시지** Kafka로 발행

### 3. 비정상/네트워크 에러 처리

- HTTP 타임아웃/네트워크 오류 발생시 **Kafka Scheduler**로 딜레이 재시도
- 여러 차례 실패 시 **Batch**에서 중단 상태 탐지, Alert 후 수동/자동 복구

### 4. DLQ/데드레터

- 보상 트랜잭션 메시지(출금 취소)가 컨슈머에서 실패 → **DLQ 서버**에서 자동 재시도
- 일정 횟수 초과 시 운영 개입(코드/서버 복구 후 재발행)

### 5. 트랜잭셔널 메시징

- **환전 실패 처리 + 보상 메시지 발행**을 원자적으로 처리 (Kafka Producer DLQ 활용)

---

## 🛠️ Trouble Shooting

분산 환전 시스템 개발 과정에서 실제로 겪었던 주요 이슈와 해결 방안을 기록합니다.

---

### 1. JPA 엔티티 간 참조 시 ID 값 매핑 문제

- **문제**  
  JPA에서 한 엔티티가 다른 엔티티를 참조(연관관계 매핑)할 때, 참조 엔티티의 객체 자체가 아니라 **ID 값만 이용해 조회**하거나 쿼리를 작성해야 할 경우가 있었음.  
  익숙하지 않은 JPA 문법(예: `entity.field.id`, `@ManyToOne(fetch = FetchType.LAZY)` 등)으로 인해 적절한 JPQL, Criteria, 네이티브 쿼리 작성에 어려움을 겪음.

- **해결**  
  공식 문서 및 여러 예제들을 참고하여,  
  연관 엔티티의 ID 값만 이용한 조건 검색/조인 방법을 학습하고,  
  `join fetch` 또는 `where e.otherEntity.id = :id`와 같이 JPQL에서 엔티티 객체가 아닌 **ID 값 기준의 검색** 방식으로 개선.

---

### 2. HTTP 응답 코드와 비즈니스/시스템 예외 구분 문제

- **문제**  
  HTTP 통신에서 비즈니스 로직 상 실패(예: 잔액 부족 등)를 599 등의 **5xx 서버 에러**로 반환했더니,  
  시스템 예외와 비즈니스 예외가 동일하게 처리되어  
  재시도/장애 감지/알림 등에서 혼선 발생.

- **해결**  
  명확히 **비즈니스 실패(예: 200 OK + 실패 사유 메시지)**와  
  **시스템 장애(5xx)**를 구분하여  
  비즈니스 실패는 2xx 상태코드(주로 200/202 등)로 내려주고,  
  실제 에러 정보는 본문(body)에 담아 컨슈머에서 분기 처리하도록 수정.

---

### 3. HTTP 실패 요청 재처리 시 영속성 및 ID 할당 문제

- **문제**  
  HTTP 실패 요청을 Kafka로 넘겨 일정 시간 후 재실행하는 구조에서,  
  이전 요청의 DTO를 이용해 엔티티를 새로 만들면 JPA에서 아직 저장하지 않은 영속 상태라 **ID 값이 없음**.  
  엔티티 자체를 Kafka로 넘기는 것은 비효율적이며,  
  save() 호출 시에는 ID가 새로 부여되어 원래 요청과 매칭되지 않는 문제 발생.

- **해결**  
  요청 DTO에 **UUID 등 고유 식별자**를 필수로 포함시켜  
  Kafka Consumer에서는 이 UUID를 이용해  
  기존 DB에서 엔티티를 조회 및 재처리하도록 구조 개선.

---

### 4. Kafka DTO ↔️ JSON 변환 예외 관리

- **문제**  
  Kafka 프로듀서/컨슈머 간 DTO와 JSON 변환 과정에서  
  매핑 오류, 필드 누락 등 파싱 예외가 발생했으나  
  비즈니스 트랜잭션과는 직접 관련이 없어 시스템 로직과 혼재됨.

- **해결**  
  Kafka의 **Serializer/Deserializer** 설정을 통일하고  
  (ex: Jackson, Gson 등 명확히 일치시키고, 불일치 시 예외 발생),  
  JSON 파싱 예외를 Kafka 설정에서 처리(즉, 개별 비즈니스 로직에서 try-catch로 잡지 않음)  
  → **카프카 시스템 레벨**에서 예외 처리 및 DLQ로 위임.

---

### 5. Kafka 토픽 파서 불일치로 인한 변환 오류

- **문제**  
  Kafka 토픽 발행 시 사용하는 파서(직렬화/역직렬화 방식)가 서버/서비스별로 일치하지 않아  
  DTO ↔️ JSON 변환에 실패, 메시지 소비 불가

- **해결**  
  **Kafka 프로듀서/컨슈머의 직렬화/역직렬화 파서**를 **통일**  
  (동일한 라이브러리 및 설정 사용, DTO 스키마/버전 관리)  
  → 메시지 포맷 일관성 확보

---

### 6. Kafka 장애시 보상 트랜잭션 중단(단일 장애점) 문제

- **문제**  
  Kafka가 다운되면 보상 트랜잭션이 동작하지 않아  
  **사용자 자금 손실, 데이터 불일치** 등 치명적 리스크 존재

- **해결**  
  **Kafka 브로커 복제(Replication) 및 클러스터링** 적용  
  - 최소 2대 이상의 브로커로 구성,  
  - 파티션/리더/팔로워 구조로 한쪽 서버가 장애 발생 시에도  
    다른 브로커가 **Failover**하여 메시지 손실/누락 없이 처리  
  → 시스템의 **고가용성(HA)** 및 신뢰성 확보

---

> 실서비스 환경에서 발생했던 트러블을 기반으로,  
> 장애/에러 대응력과 분산 환경에서의 일관성 있는 트랜잭션 관리 경험을 축적하였습니다.


