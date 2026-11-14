# 지역 기반 수리 매칭 플랫폼 (동네수리)

가전제품 및 전자기기 수리가 필요한 의뢰자와 수리자를 지역 기반으로 연결하는 매칭 플랫폼입니다.
멋쟁이사자처럼 자바 백엔드 부트캠프 24기 파이널 프로젝트입니다.

- 기간: 2026.09 (1개월)
- 인원: 4명

> 이 저장소는 팀 프로젝트를 fork하였습니다.

## 서비스 흐름

1. 의뢰자가 수리 글을 올린다
2. 수리자와 채팅으로 협의하고 견적에 합의한다
3. 계약서에 서명하고 결제한다
4. 수리가 끝나면(거래 완료) 의뢰자가 후기를 남긴다

## 기술 스택

| 구분 | 기술 |
|---|---|
| Language / Framework | Java 21, Spring Boot 3.4.2 |
| 데이터 | Spring Data JPA, MySQL, Flyway |
| 서비스 간 통신 | Spring Cloud OpenFeign, REST Client |
| 결제 | PortOne |
| 테스트 | JUnit 5, Spring Boot Test, Testcontainers(MySQL) |
| 문서 | springdoc-openapi (Swagger UI) |
| 배포 | Docker |

## 아키텍처

| 서비스 | 역할 |
|---|---|
| api-gateway | 요청 라우팅 |
| user-service | 회원·인증, 지역 |
| post-service | 게시글, 거래, 계약서, **후기**, **프로필** |
| payment-service | **결제**, 웹훅 처리, |
| chat-service | 채팅방, 메시지, 첨부파일 |
| common | 공통 예외 처리, 로그인 사용자, 서비스 간 공유 객체 |

## 내가 맡은 부분

결제(payment-service), 후기 작성과 프로필 조회(post-service)를 구현했습니다.

### 결제 (payment-service)

PortOne 결제 연동입니다. 클라이언트가 보내는 값을 믿지 않는 것을 기준으로 설계했습니다.

- **서버 주문 선생성**: 결제창을 열기 전에 서버가 postId로 금액·수신자를 조회해 주문을 만든다.
- **결제 후 재조회 검증**: 결제 상태(PAID), 금액, 통화(KRW)를 PortOne 서버에서 다시 조회해 주문과 대조한 뒤 저장한다. 확정 시점에 견적·수신자가 바뀌지 않았는지도 다시 확인한다.
- **멱등성**: 프론트 확인 요청과 웹훅이 같은 결제를 동시에 저장하지 않도록 `portone_payment_id` unique 제약과 `DataIntegrityViolationException` 처리를 사용한다.
- **웹훅 보안**: HMAC-SHA256 서명 검증, 5분을 넘은 타임스탬프 거부, 비교는 `MessageDigest.isEqual`로 상수 시간 비교.
- **취소 후 재결제**: Flyway 마이그레이션(`V2`)으로 `post_id` unique 제약을 일반 인덱스로 바꿔 한 거래에 여러 결제 이력을 허용하고, 현재 유효한 결제는 최신 행으로 판단한다.
- **수수료**: 견적 금액의 10%를 플랫폼 수수료로 계산해 수리자 수령액과 함께 저장한다.
- 실제 계좌 송금은 범위 밖이고, 정산은 확정 시각(`settledAt`)만 기록한다.

### 후기 작성 (post-service)

완료된 거래에 대한 후기를 작성합니다.

- 거래가 완료(COMPLETED)된 건만 작성 가능
- 의뢰자 본인만 작성 가능
- 거래 완료 후 3일 이내만 작성 가능
- 거래당 후기 1건
- 별점 1~5, 이미지 최대 5장
- 취소 후 재매칭된 글은 취소된 거래를 제외하고 조회

### 프로필 조회 (post-service)

내 거래 내역과 내가 쓴 후기를 페이징으로 조회합니다.

- 거래 내역마다 chat-service를 호출하지 않고, 한 페이지의 거래 ID를 모아 채팅방 정보를 한 번에 조회
- 같은 작성자·대상의 닉네임은 한 번 조회한 값을 재사용
- 채팅방이 없는 거래, 삭제된 게시글에서도 예외 없이 동작하도록 처리하고 테스트(`ProfileServiceTest`)로 검증

## 실행 방법

필요한 것: Java 21, MySQL

1. 각 서비스의 `src/main/resources`에 `application-secret.yaml`을 만들어 DB 접속 정보와 PortOne 설정값을 넣습니다. (`application-secret.example.yaml` 참고 — 실제 값은 저장소에 올리지 않습니다)
2. 서비스별로 실행합니다.

```bash
cd backend/user-service && ./gradlew bootRun
cd backend/post-service && ./gradlew bootRun
cd backend/payment-service && ./gradlew bootRun
cd backend/chat-service && ./gradlew bootRun
cd backend/api-gateway && ./gradlew bootRun
```

3. API 문서: 각 서비스의 Swagger UI (`/swagger-ui.html`) 또는 게이트웨이의 `/{서비스명}/v3/api-docs`

## 테스트

```bash
./gradlew test
```

Flyway 스키마 검증은 Testcontainers(MySQL)로 실행하므로 Docker가 필요합니다.
