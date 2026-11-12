# 동네수리

   집 안의 고장·수리가 필요한 **의뢰자**와 **수리자**를 지역 기반으로 연결하는 수리 매칭 플랫폼입니다.<br/>멋쟁이사자처럼 자바 백엔드 부트캠프 24기 프로젝트입니다.

- 기간: 2026.09 (1개월)
- 인원: 4명

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

마이크로서비스 구조이며, 모든 요청은 API Gateway를 거칩니다.

| 서비스 | 역할 |
|---|---|
| api-gateway | 요청 라우팅 |
| user-service | 회원·인증, 지역 |
| post-service | 게시글, 거래(FixDeal), 계약서, 후기, 프로필(거래 내역) |
| payment-service | 결제, 웹훅 처리, 정산 확정 기록 |
| chat-service | 채팅방, 메시지, 첨부파일 |
| common | 공통 예외 처리, 로그인 사용자, 서비스 간 공유 객체 |

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

`POST /reviews` (multipart)

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

## 문제 해결

| 문제 | 원인 | 해결 |
|---|---|---|
| 결제 건이 있는데 PortOne 조회가 404 | 여러 팀이 쓰는 공용 API 키라 storeId를 명시하지 않으면 기본 스토어로 조회됨 | storeId를 지정하고, 일시적 404에 대비해 대기 시간을 늘리며 최대 4회 재시도 |
| 후기 작성 시 `NonUniqueResultException` | 취소 후 재매칭된 글에 같은 postId의 취소된 거래가 남아 있음 | 취소된 거래를 제외하고 조회 |
| 프로필 거래 내역에서 NPE | 채팅방이 아직 없는 거래의 값이 null인 채로 `Set.copyOf`에 들어감 | null을 걸러낸 뒤 변환 |

## 실행 방법

필요한 것: Java 21, MySQL

```bash
cd backend/user-service && ./gradlew bootRun
cd backend/post-service && ./gradlew bootRun
cd backend/payment-service && ./gradlew bootRun
cd backend/chat-service && ./gradlew bootRun
cd backend/api-gateway && ./gradlew bootRun
```

API 문서: 각 서비스의 Swagger UI (`/swagger-ui.html`) 또는 게이트웨이의 `/{서비스명}/v3/api-docs`
