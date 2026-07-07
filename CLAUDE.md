# CLAUDE.md — Legacy Ops Guard 프로젝트 가이드

이 파일은 Claude Code가 이 프로젝트를 이해하고 올바른 코드를 생성하기 위한 지침입니다.

---

## 프로젝트 한 줄 설명

PG 결제 운영 리스크를 줄이기 위한 대상자 검증·중복 방지·결제 실행 감시 백엔드 시스템.
실제 운영 환경의 수동 검증 구조를 상태 추적·자동 중단·재처리 흐름으로 시스템화한 포트폴리오 프로젝트.

---

## 기술 스택

- Java 17
- Spring Boot 3.3
- Spring Data JPA
- H2 (개발/테스트)
- springdoc-openapi (Swagger UI)
- JUnit 5
- Gradle

---

## 패키지 구조

```
com.legacyopsguard
 ├── common
 │   ├── exception      전역 예외 처리
 │   ├── response       공통 응답 포맷 (ApiResponse<T>)
 │   └── config         Swagger 등 설정
 ├── paymenttarget      결제 대상자 정규화 및 사전 검증
 ├── paymentrun         결제 실행 단위 관리
 ├── failure            실패 기록 관리
 ├── retry              재처리 요청 관리
 └── report             운영 리포트
```

각 도메인 패키지 안에는 controller / service / domain / repository / dto 를 둔다.

---

## 도메인 모델 요약

| 클래스 | 역할 |
|--------|------|
| PaymentTarget | 결제 대상자 (입력 → 정규화 → 사전검증) |
| PaymentPreCheck | 당월 중복 결제 사전 검증 결과 |
| PaymentRun | 결제 실행 단위 (성공/실패/중단 집계) |
| PaymentResult | 개별 결제 결과 |
| FailureRecord | 실패 기록 (유형·재처리 가능 여부) |
| RetryRequest | 재처리 요청 |
| OperationEventLog | 운영 이벤트 이력 |

---

## Enum 목록 (이미 정의됨 — 임의 변경 금지)

### ValidationStatus (PaymentTarget)
- VALID, DUPLICATED, INVALID_FORMAT, EMPTY_VALUE

### PreCheckStatus (PaymentPreCheck)
- READY_TO_PAY, ALREADY_PAID, NEED_MANUAL_REVIEW, INVALID_TARGET

### PaymentRunStatus (PaymentRun)
- READY, RUNNING, SUCCEEDED, FAILED, COMMUNICATION_FAILED,
  STOPPED_BY_RULE, COMPLETED_WITH_FAILURES

### PaymentResultStatus (PaymentResult)
- SUCCESS, FAILED, COMMUNICATION_FAILED, SKIPPED

### FailureType (FailureRecord)
- VALIDATION_FAILED, ALREADY_PAID, PG_COMMUNICATION_FAILED,
  PAYMENT_REJECTED, DB_UPDATE_FAILED, STOPPED_BY_RULE, UNKNOWN_ERROR

### RetryStatus (RetryRequest)
- RETRY_AVAILABLE, RETRY_BLOCKED, RETRY_REQUESTED,
  RETRY_SUCCEEDED, RETRY_FAILED, NEED_MANUAL_CONFIRM

---

## 코드 생성 규칙

### 반드시 따를 것
- 모든 Entity는 `@Entity`, `@Table`, `@Id`, `@GeneratedValue` 포함
- ID 전략: `GenerationType.IDENTITY`
- 생성 시각은 모든 Entity에 `createdAt` (`LocalDateTime`) 포함
- PaymentRun은 `startedAt`, `endedAt` 추가
- 연관관계는 단방향 `@ManyToOne` 기준으로만 설정
- DTO는 Request / Response 분리
- 공통 응답은 `ApiResponse<T>` 래퍼 사용

### 하지 말 것
- Enum 값 임의 추가·변경 금지 (위 목록이 확정본)
- 양방향 연관관계 설정 금지
- Service에 직접 Entity 반환 금지 (DTO 변환 필수)
- Lombok `@Data` 사용 금지 (`@Getter`, `@Builder` 분리 사용)

---

## Claude Code에게 맡겨도 되는 작업

- Entity 클래스 생성 (필드, 어노테이션)
- JpaRepository 인터페이스 생성
- DTO Request/Response 클래스 생성
- Controller 라우팅 껍데기
- Swagger 어노테이션 추가
- application.yml 기본 설정
- 테스트 코드 구조 (given/when/then 틀)

## Claude Code에게 맡기면 안 되는 작업 (직접 작성)

- 자동 중단 조건 판단 로직 (연속 실패 2건, 통신 실패)
- 재처리 가능 여부 분류 로직
- PaymentRun 상태 전이 로직
- 당월 중복 결제 사전 검증 서비스 핵심 로직
- 핵심 비즈니스 로직 테스트

---

## API 응답 형식

모든 API는 아래 공통 포맷을 사용한다.

```json
{
  "success": true,
  "data": { },
  "message": null
}
```

오류 시:
```json
{
  "success": false,
  "data": null,
  "message": "오류 메시지"
}
```

---

## 현재 진행 상태

- [x] Spring Boot 프로젝트 초기화
- [x] README 작성
- [ ] CLAUDE.md 작성 (이 파일)
- [ ] Enum 정의
- [ ] Entity 생성
- [ ] Repository 생성
- [ ] 핵심 서비스 로직 구현
- [ ] Controller / DTO 생성
- [ ] Swagger 설정
- [ ] 테스트 코드
