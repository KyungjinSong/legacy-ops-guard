# Legacy Ops Guard

> PG 결제 운영 리스크를 줄이기 위한 대상자 검증·중복 방지·결제 실행 감시 시스템

![Java](https://img.shields.io/badge/Java-17-orange)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.3-brightgreen)
![H2](https://img.shields.io/badge/DB-H2-blue)
![Swagger](https://img.shields.io/badge/API-Swagger-green)

---

## 프로젝트 소개

Legacy Ops Guard는 레거시 운영 환경에서 작업자가 수동 SQL 검증과 직접 감시에 의존하던 PG 결제 운영 업무를 **대상자 정규화 → 중복 결제 사전 검증 → 결제 실행 감시 → 실패 기록 → 자동 중단 조건 → 재처리 후보 관리** 구조로 바꾸기 위한 Java/Spring 기반 백엔드 프로젝트입니다.

---

## 배경

### 기존 수동 업무 흐름

```
1. 결제 대상자 식별자 복사 → 수동 정규화 (공백·개행·특수문자 제거)
2. 당월 결제 건 존재 여부 직접 SQL 조회
3. 이미 결제된 대상자 수동 제외
4. 결제 실행
5. 실패·통신 실패가 나지 않는지 화면 직접 감시
6. 연속 실패·통신 실패 발생 시 수동 중단 또는 확인
7. 실패 건 재처리 여부 사람이 직접 판단
```

### 이 방식의 리스크

| 문제 | 내용 |
|------|------|
| 대상자 누락·중복 | 수동 정규화 과정의 오타·복사 실수 |
| 이중결제 위험 | 당월 결제 여부 미확인 시 발생 가능 |
| 감시 의존 | 대량 결제 실행 중 자리를 비우기 어려움 |
| 재처리 기준 불명확 | 통신 실패는 결제 여부 불분명, 즉시 재시도 시 이중결제 위험 |
| 이력 부재 | 어떤 실행에서 어떤 대상이 실패했는지 추적 어려움 |

---

## 핵심 기능

### 1. 결제 대상자 정규화
- 공백·개행·특수문자 자동 정리
- 식별자 형식 검증 (정규식 기반)
- 중복 대상자 자동 제거
- 검증 결과: `VALID` / `DUPLICATED` / `INVALID_FORMAT`

### 2. 당월 중복 결제 사전 검증
- 결제 실행 전 당월 결제 이력 자동 조회
- 분류: `READY_TO_PAY` / `ALREADY_PAID` / `NEED_MANUAL_REVIEW`

### 3. 결제 실행 단위 관리 및 상태 추적
- 결제 실행을 단위(PaymentRun)로 분리 관리
- 상태: `READY` → `RUNNING` → `COMPLETED` / `STOPPED_BY_RULE` / `COMPLETED_WITH_FAILURES`

### 4. 자동 중단 조건
- 연속 결제 실패 2건 이상 시 자동 중단
- 통신 실패(PG_COMMUNICATION_FAILED) 발생 시 즉시 중단
- 실패율 기준 초과 시 중단

### 5. 실패 유형 기록
- 실패 단계·사유·에러 코드·재처리 가능 여부 기록
- 유형: `VALIDATION_FAILED` / `ALREADY_PAID` / `PG_COMMUNICATION_FAILED` / `PAYMENT_REJECTED` / `DB_UPDATE_FAILED`

### 6. 재처리 후보 관리
- 실패 유형별 재처리 가능 여부 분류
- 통신 실패 → `NEED_MANUAL_CONFIRM` (자동 재시도 차단, 이중결제 방지)
- 중복 재처리 방지 로직 포함

### 7. 운영 리포트
- 총 대상·성공·실패·자동 중단 여부·수동 검토 필요 건수 요약

---

## 도메인 모델

```
PaymentTarget      결제 대상자
PaymentPreCheck    당월 중복 결제 사전 검증 결과
PaymentRun         결제 실행 단위
PaymentResult      개별 결제 결과
FailureRecord      실패 기록
RetryRequest       재처리 요청
OperationEventLog  운영 이벤트 이력
```

### 관계

```
PaymentRun    1 ──── N  PaymentResult
PaymentRun    1 ──── N  FailureRecord
PaymentRun    1 ──── N  OperationEventLog
FailureRecord 1 ──── N  RetryRequest
PaymentTarget 1 ──── N  PaymentResult
PaymentTarget 1 ──── N  PaymentPreCheck
```

---

## 상태 전이도

### PaymentRun

```
READY
  └─▶ RUNNING
        ├─▶ COMPLETED
        ├─▶ COMPLETED_WITH_FAILURES
        ├─▶ STOPPED_BY_RULE          ← 자동 중단 조건 발동
        └─▶ FAILED
```

### PaymentTarget

```
RAW_INPUT
  └─▶ VALID / DUPLICATED / INVALID_FORMAT
          └─▶ READY_TO_PAY / ALREADY_PAID / NEED_MANUAL_REVIEW
```

### RetryRequest

```
RETRY_AVAILABLE
  └─▶ RETRY_REQUESTED
        └─▶ RETRY_IN_PROGRESS
              ├─▶ RETRY_SUCCEEDED
              └─▶ RETRY_FAILED

※ 통신 실패 건: NEED_MANUAL_CONFIRM → 수동 확인 후 RETRY_AVAILABLE 전환
```

---

## API 목록

| Method | Endpoint | 설명 |
|--------|----------|------|
| POST | `/api/payment-targets/normalize` | 대상자 정규화 |
| POST | `/api/payment-targets/pre-check` | 당월 중복 결제 사전 검증 |
| POST | `/api/payment-runs` | 결제 실행 단위 생성 |
| GET | `/api/payment-runs/{runId}` | 결제 실행 상태 조회 |
| PATCH | `/api/payment-runs/{runId}/results` | 결제 결과 기록 |
| GET | `/api/payment-runs/{runId}/stop-rules` | 자동 중단 조건 조회 |
| GET | `/api/failures` | 실패 목록 조회 |
| GET | `/api/failures/{failureId}` | 실패 상세 조회 |
| POST | `/api/failures/{failureId}/retry-requests` | 재처리 요청 |
| GET | `/api/payment-runs/{runId}/report` | 운영 리포트 조회 |

---

## 자동 중단 설계 원칙

결제는 금전과 직결되기 때문에, 실패를 감지한 뒤 사람이 판단하기 전까지 실행을 멈추는 것이 자동 계속보다 안전합니다.

```
중단 조건 1: 연속 결제 실패 2건 이상
중단 조건 2: PG 통신 실패 1건 이상
중단 조건 3: 전체 대비 실패율 기준 초과
중단 조건 4: 중복 결제 위험 대상 발견
```

중단 시 `PaymentRun` 상태 → `STOPPED_BY_RULE`, 중단 사유 기록 후 운영자 수동 확인 필요

---

## 재처리 설계 원칙

재처리는 단순히 실패한 건을 다시 실행하는 기능이 아닙니다. 특히 통신 실패는 결제가 실제로 됐는지 알 수 없기 때문에, 바로 재시도하면 이중결제가 발생할 수 있습니다.

```
PAYMENT_REJECTED        → RETRY_AVAILABLE      재시도 가능
VALIDATION_FAILED       → RETRY_AVAILABLE      대상 수정 후 가능
PG_COMMUNICATION_FAILED → NEED_MANUAL_CONFIRM  수동 확인 필수
ALREADY_PAID            → RETRY_BLOCKED        재시도 불가
```

---

## 기술 스택

| 분류 | 기술 |
|------|------|
| Language | Java 17 |
| Framework | Spring Boot 3.3, Spring Data JPA |
| Database | H2 (개발/테스트), MySQL (운영 전환 가능) |
| API 문서 | springdoc-openapi (Swagger UI) |
| 테스트 | JUnit 5 |
| 빌드 | Gradle |

---

## 실행 방법

```bash
# 클론
git clone https://github.com/{계정명}/legacy-ops-guard.git
cd legacy-ops-guard

# 실행
./gradlew bootRun

# API 문서
open http://localhost:8080/swagger-ui/index.html

# H2 콘솔 (JDBC URL: jdbc:h2:mem:testdb)
open http://localhost:8080/h2-console
```

---

## AI 활용 범위

| 영역 | 여부 |
|------|------|
| 엔티티·Repository 보일러플레이트 | ✅ Claude Code 활용 |
| Swagger 설정 초안 | ✅ Claude Code 활용 |
| README 작성 보조 | ✅ Claude Code 활용 |
| 자동 중단 조건 판단 로직 | ❌ 직접 설계·구현 |
| 재처리 가능 여부 분류 로직 | ❌ 직접 설계·구현 |
| PaymentRun 상태 전이 설계 | ❌ 직접 설계·구현 |
| 당월 중복 결제 사전 검증 서비스 | ❌ 직접 설계·구현 |
| 핵심 비즈니스 로직 테스트 | ❌ 직접 설계·구현 |

> AI는 운영자를 대체하는 자동 실행자가 아니라, 운영자가 더 빨리 판단하도록 돕는 보조 도구로 제한했습니다. 결제·정산처럼 금전 리스크가 큰 영역은 자동화보다 검증·중단·기록·리포트 구조가 먼저입니다.

---

## 한계와 개선 예정

- [ ] 실제 PG 연동 없음 (Mock 기반 시뮬레이션)
- [ ] 인증·권한 미구현
- [ ] 프론트엔드 없음 (API 전용)
- [ ] Spring Batch 도입으로 대량 처리 개선 예정
- [ ] Redis 기반 중복 실행 방지 개선 예정
