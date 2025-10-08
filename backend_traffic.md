# blog 

### 백엔드 트래픽대응 최적화
Redis 캐싱, 분산락, Kafka기반 트래픽대응 및 동시성 해결 및 쇼핑몰/예약 포인트, 주문·결제, 선착순 쿠폰, 예약, 대기열 API 설계

### 쇼핑몰/예약 서비스 트래픽 대응 최적화
※ 개발환경 : SpringBoot, Java, JPA, Mysql, JUnit5
 1) 쇼핑몰 웹 솔루션 백엔드 개발- 포인트 충전·조회, 상품 주문·결제, 선착순 쿠폰, 콘서트 예약 API 설계- 대기열 토큰 예약시스템: 동시 활성화 제한, 토큰 순차 활성화- 좌석 상태 관리 3단계 전환, 결제 미완료시 자동 해제 - TDD 기반 테스트 작성 및 클린 아키텍처 적용 및 JUnit5+AssertJ 기반 단위/통합 테스트 
2) 성능 최적화 및 동시성 처리- 좌석배치도 Redis 캐싱으로 DB 부하감소- Redis 분산락적용 : 동일좌석예약, 재고상품주문, 쿠폰발급 경쟁조건 동시성제어- 다중 인스턴스 환경에서 재고 차감과 쿠폰 발급 정확성 보장
3) 이벤트 처리 및 안정성- Kafka 활용 이벤트 메시지 송수신 구현- 부하 테스트 및 장애 시나리오 대응을 통한 안정성 확보 - DB 성능 분석 및 최적화: 복합 인덱스, 테이블 파티셔닝으로 좌석 조회속도 개선, 동시성테스트로 장애 시나리오 사전 검증- Testcontainers 격리 테스트 : MySQL 8.0 컨테이너 기반환경에서 데드락, 락 충돌, 타임아웃 시나리오 테스트로 운영 환경 안정성 검증

### [백엔드 트래픽대응 최적화]
쇼핑몰 및 예약 시스템을 구현하며 포인트 충전·조회, 상품 주문·결제, 선착순 쿠폰, 좌석 예약 API를 설계하고 TDD 기반 테스트 작성과 클린 아키텍처를
적용하였습니다. 대기열 토큰 시스템을 통해 동시 활성화를 제한하고, 미결제 시 자동 해제되는 상태 전환 로직을 구현했습니다. 트래픽 대응을 위해 좌
석 배치도 등 실시간 변경 가능성이 낮은 데이터에 Redis Cache를 적용하여 DB 부하를 감소시켰고, 동시 주문 재고 처리 및 선착순 쿠폰 중복 발급 경쟁
상황 해결을 위해 Redisson 기반 분산락을 적용하였습니다. 또한 복합 인덱스 및 테이블 파티셔닝을 통해 DB 성능을 최적화하고 Testcontainer 환경에서
데드락·락충돌·타임아웃 시나리오를 포함한 부하 테스트와 장애 대응을 통해 시스템 안정성과 운영 역량을 체득했습니다.
 
  Q: 클린 아키텍처를 왜 구현했나요? 어떤 문제를 해결하기 위해서였나요?
  - 계층 간 의존성을 명확히 분리하여 테스트 가능성과 유지보수성을 향상시키기 위해
    
  Q: 대기열 토큰 시스템을 통해 동시 활성화 제한을 어떻게 했나요?
  -  스케줄러로 30초마다 최대 100명까지만 활성화하고, 나머지는 대기 상태로 유지했습니다.
```java
    // 현재 활성화된 토큰 수 확인
       List<QueueToken> activeTokens = queueTokenRepository.findActiveTokens();
       int currentActiveCount = activeTokens.size();
    // 새로운 토큰들을 활성화 (최대 100개 제한)
       int availableSlots = MAX_ACTIVE_TOKENS - currentActiveCount;  // 남은 슬롯 계산 
  상황: 1,000명이 동시 접속
  처리 방식:
  - 0초: 1,000명 토큰 발급 → 모두 WAITING 상태 (대기 순서 1~1000번)
  - 0초: 즉시 최초 100명만 ACTIVE로 전환
  - 30초: 만료된 10명 제거 + 대기 순서 1~10번 활성화 (총 100명 유지)
  - 60초: 만료된 15명 제거 + 대기 순서 1~15번 활성화 (총 100명 유지)
```
     
  Q: 미결제 시 자동 해제되는 상태 전환 로직을 어떻게 구현했나요?
  - 도메인 엔티티에서 만료 시간을 체크하고, isAvailable() 호출 시 만료된 경우 자동으로 상태를 해제하는 Lazy Evaluation
```java
  시나리오 1: 정상 결제 완료
  1. 14:00:00 - 좌석 예약 (AVAILABLE → TEMPORARY_RESERVED, 만료: 14:05:00)
  2. 14:03:00 - 결제 완료 (TEMPORARY_RESERVED → RESERVED, 만료: null)
  3. 14:10:00 - 좌석 확정 상태 유지 (RESERVED)

  시나리오 2: 미결제 자동 해제
  1. 14:00:00 - 좌석 예약 (AVAILABLE → TEMPORARY_RESERVED, 만료: 14:05:00)
  2. 14:03:00 - 결제 페이지에서 고민 중...
  3. 14:06:00 - 다른 사용자가 좌석 조회 시도
     → isAvailable() 호출
     → isExpired() = true (현재 14:06 > 만료 14:05)
     → release() 자동 호출
     → TEMPORARY_RESERVED → AVAILABLE
  4. 14:06:00 - 새로운 사용자에게 좌석 할당 가능

  시나리오 3: 결제 시도 중 만료
  // ReservationUseCase.java:96-122
  @Transactional
  public PaymentResult processPayment(ProcessPaymentCommand command) {
      // 예약 조회
      Reservation reservation = reservationRepository.findById(command.getReservationId())
              .orElseThrow();

      // 만료 체크
      if (reservation.isExpired()) {
          throw new IllegalStateException("Reservation has expired");  // 결제 불가
      }

      // ... 결제 처리
  }

  별도 스케줄러 불필요한 이유
  기존 방식 (스케줄러 사용):
  // ❌ 불필요한 스케줄러
  @Scheduled(fixedRate = 60000)  // 1분마다 실행
  public void cleanupExpiredReservations() {
      List<Seat> expired = seatRepository.findExpiredSeats();
      for (Seat seat : expired) {
          seat.release();
          seatRepository.save(seat);
      }
  }

  문제점:
  1. 주기적으로 전체 테이블 스캔
  2. 만료되지 않은 데이터도 체크
  3. 불필요한 DB 부하
  4. 최대 1분 지연 발생

  현재 방식 (Lazy Evaluation):
  // ✅ 조회 시점에 자동 해제
  public boolean isAvailable() {
      if (seatStatus == SeatStatus.TEMPORARY_RESERVED && isExpired()) {
          release();  // 만료 시 즉시 해제
          return true;
      }
      return false;
  }
  장점:
  1. 별도 스케줄러 불필요
  2. 실제로 조회할 때만 체크 (효율적)
  3. 즉시 해제 (지연 없음)
  4. DB 부하 최소화

```

  Q: Redisson 기반 분산락을 사용한 이유는 무엇인가요? (Lettuce나 다른 대안 대비 장점은?)
  - Lettuce 대비 데드락방지, 재시도 로직 (tryLock(waitTime, leaseTime)) , 락자동해제 (Watch Dog 자동갱신) 메커니즘이 내장되어 있어 안정적인 분산락 구현이 가능하기
  때문입니다.

  Q: kafka 를어떻게사용했고 어떻게확장시킬수있나요?

  Q: Testcontainer 환경에서 데드락·락충돌·타임아웃 시나리오를 포함한 부하 테스트와 장애 대응을 어떻게 처리했나요?
  - @SpringBootTest + H2 In-Memory DB + ExecutorService로 실제 동시 요청을 재현하고, 데드락/락충돌/타임아웃을 의도적으로 발생시켜 검증했습니다.

  
### API / TDD / 아키텍처 
  
  1. RESTful API 설계 원칙
  Q: 포인트 충전·조회 API를 어떻게 설계했나요? RESTful 원칙을 어떻게 적용했나요?
  A: REST 리소스 중심 설계와 HTTP 메서드를 적절히 활용했습니다.

  // BalanceController.java:14-44
  ```java 
  @RestController
  @RequestMapping("/api/v1/users/{userId}/balance")
  public class BalanceController {

      private final UserService userService;

      // 포인트 충전 - POST 메서드
      @PostMapping("/charge")
      public ResponseEntity<ApiResponse<BalanceResponse>> chargeBalance(
              @PathVariable Long userId,
              @Valid @RequestBody BalanceChargeRequest request) {

          BalanceResponse response = userService.chargeBalance(userId, request.getAmount());

          return ResponseEntity.ok(
              ApiResponse.success("잔액 충전이 완료되었습니다", response)
          );
      }

      // 포인트 조회 - GET 메서드
      @GetMapping
      public ResponseEntity<ApiResponse<BalanceResponse>> getBalance(@PathVariable Long userId) {

          BalanceResponse response = userService.getBalance(userId);

          return ResponseEntity.ok(
              ApiResponse.success("잔액 조회가 완료되었습니다", response)
          );
      }
  }
```
  RESTful 설계 원칙 적용:
  1. 리소스 중심 URL: /api/v1/users/{userId}/balance - 사용자의 잔액이라는 리소스를 명확히 표현
  2. HTTP 메서드 활용:
    - POST /charge: 충전이라는 행위 (비멱등성)
    - GET: 조회 (안전성, 멱등성)
  3. 계층 구조: users/{userId}/balance - 사용자 하위 리소스로 잔액 표현
  4. 버전 관리: /api/v1 - 향후 호환성 유지
  5. 일관된 응답 형식: ApiResponse<T> 래퍼로 통일된 응답 구조

  ---
  2. 주문·결제 API 설계
  Q: 상품 주문·결제 API는 어떻게 설계했나요?
  A: 주문 생성, 조회, 목록을 RESTful하게 구현했습니다.

  // OrderController.java:14-57
  ```java 
  @RestController
  @RequestMapping("/api/v1")
  public class OrderController {

      private final OrderUseCase orderUseCase;

      // 주문 생성 - POST
      @PostMapping("/orders")
      public ResponseEntity<ApiResponse<OrderResponse>> createOrder(
              @Valid @RequestBody OrderRequest request) {

          OrderResponse response = orderUseCase.createOrder(request);

          return ResponseEntity.ok(
              ApiResponse.success("주문이 완료되었습니다", response)
          );
      }

      // 주문 단건 조회 - GET
      @GetMapping("/orders/{orderId}")
      public ResponseEntity<ApiResponse<OrderResponse>> getOrder(@PathVariable Long orderId) {

          OrderResponse response = orderUseCase.getOrder(orderId);

          return ResponseEntity.ok(
              ApiResponse.success("주문 조회가 완료되었습니다", response)
          );
      }

      // 사용자별 주문 목록 조회 - GET with Pagination
      @GetMapping("/users/{userId}/orders")
      public ResponseEntity<ApiResponse<Page<OrderResponse>>> getUserOrders(
              @PathVariable Long userId,
              @RequestParam(defaultValue = "0") int page,
              @RequestParam(defaultValue = "10") int size) {

          Page<OrderResponse> response = orderUseCase.getUserOrders(userId, page, size);

          return ResponseEntity.ok(
              ApiResponse.success("주문 목록 조회가 완료되었습니다", response)
          );
      }
  }
```
  설계 특징:
  1. 컬렉션 리소스: /orders - 주문 목록
  2. 개별 리소스: /orders/{orderId} - 특정 주문
  3. 하위 컬렉션: /users/{userId}/orders - 사용자별 주문 목록
  4. 페이지네이션: page, size 파라미터로 대용량 데이터 처리
  5. Validation: @Valid 어노테이션으로 요청 검증

  ---
  3. 콘서트 예약 API 설계
  Q: 콘서트 예약 API의 대기열 토큰 시스템은 어떻게 설계했나요?
  A: 토큰 발급, 상태 조회, 예약/결제 시 헤더로 토큰 전달하는 방식으로 구현했습니다.

  // ConcertController.java:12-88
 ```java 
  @RestController
  @RequestMapping("/api/concerts")
  public class ConcertController {

      private final ReservationUseCase reservationUseCase;
      private final QueueManagementUseCase queueManagementUseCase;

      // 1. 대기열 토큰 발급
      @PostMapping("/queue/token")
      public ResponseEntity<QueueTokenResult> issueQueueToken(
              @RequestBody IssueTokenRequest request) {
          QueueTokenResult result = queueManagementUseCase.issueToken(request.getUserId());
          return ResponseEntity.ok(result);
      }

      // 2. 대기열 상태 조회
      @GetMapping("/queue/status")
      public ResponseEntity<QueueStatusResult> getQueueStatus(
              @RequestParam String tokenUuid) {
          QueueStatusResult result = queueManagementUseCase.getQueueStatus(tokenUuid);
          return ResponseEntity.ok(result);
      }

      // 3. 예약 가능한 스케줄 조회 (토큰 필수)
      @GetMapping("/schedules")
      public ResponseEntity<List<AvailableScheduleInfo>> getAvailableSchedules(
              @RequestHeader("Queue-Token") String tokenUuid) {
          // 토큰 유효성 검증
          queueManagementUseCase.getQueueStatus(tokenUuid);

          List<AvailableScheduleInfo> schedules = reservationUseCase.getAvailableSchedules();
          return ResponseEntity.ok(schedules);
      }

      // 4. 좌석 목록 조회 (토큰 필수)
      @GetMapping("/schedules/{scheduleId}/seats")
      public ResponseEntity<List<AvailableSeatInfo>> getAvailableSeats(
              @PathVariable Long scheduleId,
              @RequestHeader("Queue-Token") String tokenUuid) {
          queueManagementUseCase.getQueueStatus(tokenUuid);

          List<AvailableSeatInfo> seats = reservationUseCase.getAvailableSeats(scheduleId);
          return ResponseEntity.ok(seats);
      }

      // 5. 좌석 예약 (토큰 필수)
      @PostMapping("/reservations")
      public ResponseEntity<ReservationResult> reserveSeat(
              @RequestHeader("Queue-Token") String tokenUuid,
              @RequestBody ReserveSeatRequest request) {

          ReserveSeatCommand command = new ReserveSeatCommand(
                  tokenUuid,
                  request.getUserId(),
                  request.getSeatId(),
                  request.getPrice()
          );

          ReservationResult result = reservationUseCase.reserveSeat(command);
          return ResponseEntity.ok(result);
      }

      // 6. 결제 처리 (토큰 필수)
      @PostMapping("/payments")
      public ResponseEntity<PaymentResult> processPayment(
              @RequestHeader("Queue-Token") String tokenUuid,
              @RequestBody ProcessPaymentRequest request) {

          ProcessPaymentCommand command = new ProcessPaymentCommand(
                  tokenUuid,
                  request.getUserId(),
                  request.getReservationId()
          );

          PaymentResult result = reservationUseCase.processPayment(command);
          return ResponseEntity.ok(result);
      }
  }
```
  대기열 토큰 설계 특징:
  1. 토큰 발급 → 상태 폴링 → 활성화 후 예약: 명확한 플로우
  2. Header 기반 인증: Queue-Token 헤더로 토큰 전달
  3. 토큰 검증 레이어: 모든 예약/결제 API에서 토큰 유효성 검증
  4. 계층적 리소스: /schedules/{scheduleId}/seats - 스케줄 하위 좌석
  5. 명령/조회 분리: 조회(GET)와 액션(POST)을 명확히 구분

  ---
  4. TDD 기반 단위 테스트

  Q: TDD를 어떻게 적용했나요? 단위 테스트 예시를 보여주세요.

  A: Given-When-Then 패턴으로 도메인 로직을 먼저 테스트했습니다.

  // ProductTest.java:11-84
```java 
  @DisplayName("Product 도메인 테스트")
  class ProductTest {

      @Nested
      @DisplayName("재고 차감 테스트")
      class DeductStockTest {

          @Test
          @DisplayName("정상적인 재고 차감")
          void deductStock_Success() {
              // given
              Product product = new Product("iPhone 15", "Apple iPhone",
                                          BigDecimal.valueOf(1000000), 10);
              Integer deductQuantity = 3;
              Integer expectedStock = 7;

              // when
              product.deductStock(deductQuantity);

              // then
              assertThat(product.getStockQuantity()).isEqualTo(expectedStock);
          }

          @Test
          @DisplayName("재고 부족 시 예외 발생")
          void deductStock_InsufficientStock_ThrowsException() {
              // given
              Product product = new Product("iPhone 15", "Apple iPhone",
                                          BigDecimal.valueOf(1000000), 3);
              Integer deductQuantity = 5;

              // when & then
              assertThatThrownBy(() -> product.deductStock(deductQuantity))
                  .isInstanceOf(IllegalStateException.class)
                  .hasMessage("재고가 부족합니다. 요청: 5개, 현재 재고: 3개");
          }

          @Test
          @DisplayName("null 수량 차감 시 예외 발생")
          void deductStock_NullQuantity_ThrowsException() {
              // given
              Product product = new Product("iPhone 15", "Apple iPhone",
                                          BigDecimal.valueOf(1000000), 10);

              // when & then
              assertThatThrownBy(() -> product.deductStock(null))
                  .isInstanceOf(IllegalArgumentException.class)
                  .hasMessage("차감할 수량은 0보다 커야 합니다.");
          }

          @Test
          @DisplayName("0 수량 차감 시 예외 발생")
          void deductStock_ZeroQuantity_ThrowsException() {
              // given
              Product product = new Product("iPhone 15", "Apple iPhone",
                                          BigDecimal.valueOf(1000000), 10);

              // when & then
              assertThatThrownBy(() -> product.deductStock(0))
                  .isInstanceOf(IllegalArgumentException.class)
                  .hasMessage("차감할 수량은 0보다 커야 합니다.");
          }
      }
  }
```
  TDD 적용 방식:
  1. 도메인 로직 먼저 테스트: Product 엔티티의 비즈니스 로직 검증
  2. Given-When-Then: 명확한 테스트 구조
  3. AssertJ 활용: 가독성 높은 assertion
  4. @Nested: 테스트 그룹화로 가독성 향상
  5. 경계 조건 테스트: 정상 케이스, 예외 케이스 모두 검증

  ---
  5. Mockito 기반 서비스 테스트
  Q: 서비스 계층 단위 테스트는 어떻게 작성했나요?
  A: Mockito로 의존성을 모킹하여 격리된 단위 테스트를 작성했습니다.

  // UserServiceTest.java:27-120
```java 
  @ExtendWith(MockitoExtension.class)
  @DisplayName("UserService 테스트")
  class UserServiceTest {

      @Mock
      private UserRepository userRepository;

      @Mock
      private BalanceHistoryRepository balanceHistoryRepository;

      @InjectMocks
      private UserService userService;

      @Nested
      @DisplayName("잔액 충전 테스트")
      class ChargeBalanceTest {

          @Test
          @DisplayName("정상적인 잔액 충전")
          void chargeBalance_Success() {
              // given
              Long userId = 1L;
              BigDecimal chargeAmount = BigDecimal.valueOf(10000);
              User user = new User("testUser", "test@example.com");

              given(userRepository.findByIdWithLock(userId)).willReturn(Optional.of(user));
              given(userRepository.save(any(User.class))).willReturn(user);
              given(balanceHistoryRepository.save(any(BalanceHistory.class)))
                  .willAnswer(invocation -> invocation.getArgument(0));

              // when
              BalanceResponse response = userService.chargeBalance(userId, chargeAmount);

              // then
              assertThat(response.getUserId()).isEqualTo(userId);
              assertThat(response.getBalance()).isEqualTo(chargeAmount);
              assertThat(response.getChargedAmount()).isEqualTo(chargeAmount);

              verify(userRepository).findByIdWithLock(userId);
              verify(userRepository).save(user);
              verify(balanceHistoryRepository).save(any(BalanceHistory.class));
          }

          @Test
          @DisplayName("존재하지 않는 사용자 충전 시 예외 발생")
          void chargeBalance_UserNotFound_ThrowsException() {
              // given
              Long userId = 1L;
              BigDecimal chargeAmount = BigDecimal.valueOf(10000);

              given(userRepository.findByIdWithLock(userId)).willReturn(Optional.empty());

              // when & then
              assertThatThrownBy(() -> userService.chargeBalance(userId, chargeAmount))
                  .isInstanceOf(BusinessException.class)
                  .satisfies(exception -> {
                      BusinessException be = (BusinessException) exception;
                      assertThat(be.getErrorCode()).isEqualTo(ErrorCode.USER_NOT_FOUND);
                  });
          }

          @Test
          @DisplayName("최소 충전 금액 미만 충전 시 예외 발생")
          void chargeBalance_InvalidAmount_ThrowsException() {
              // given
              Long userId = 1L;
              BigDecimal invalidAmount = BigDecimal.valueOf(500);

              // when & then
              assertThatThrownBy(() -> userService.chargeBalance(userId, invalidAmount))
                  .isInstanceOf(BusinessException.class)
                  .satisfies(exception -> {
                      BusinessException be = (BusinessException) exception;
                      assertThat(be.getErrorCode()).isEqualTo(ErrorCode.INVALID_AMOUNT);
                      assertThat(be.getMessage()).contains("충전 금액은 1,000원 이상이어야 합니다");
                  });
          }
      }
  }
```
  단위 테스트 전략:
  1. @Mock: Repository 의존성 모킹
  2. @InjectMocks: UserService에 모킹된 의존성 주입
  3. BDD 스타일: given().willReturn() 패턴
  4. verify(): 메서드 호출 여부 검증
  5. 예외 검증: assertThatThrownBy() + satisfies()로 상세 검증

  ---
  6. 통합 테스트
  Q: 통합 테스트는 어떻게 작성했나요?
  A: @DataJpaTest로 실제 DB를 사용한 전체 플로우 테스트를 작성했습니다.

  // ConcertReservationIntegrationTest.java:25-106
```java 
  @DataJpaTest
  @TestPropertySource(properties = {
          "spring.datasource.url=jdbc:h2:mem:testdb",
          "spring.jpa.hibernate.ddl-auto=create-drop"
  })
  class ConcertReservationIntegrationTest {

      @Autowired
      private TestEntityManager entityManager;

      @Autowired
      private JpaSeatRepository seatRepository;

      @Autowired
      private JpaReservationRepository reservationRepository;

      @Autowired
      private JpaQueueTokenRepository queueTokenRepository;

      @Test
      @DisplayName("콘서트 예약 전체 플로우 통합 테스트")
      void concertReservationFullFlow() {
          // Given: 콘서트 및 스케줄 설정
          Concert concert = new Concert("Spring Concert", "Spring Band", "Spring Hall");
          entityManager.persist(concert);

          ConcertSchedule schedule = new ConcertSchedule(
                  concert,
                  LocalDateTime.now().plusDays(7),
                  LocalDateTime.now().minusHours(1)
          );
          entityManager.persist(schedule);

          Seat seat = new Seat(schedule, 1);
          entityManager.persist(seat);

          entityManager.flush();
          entityManager.clear();

          Long userId = 1L;
          BigDecimal price = new BigDecimal("50000");

          // 1. 대기열 토큰 생성
          QueueToken token = new QueueToken(userId, 1L);
          token.activate(10);
          queueTokenRepository.save(token);

          // 2. 좌석 예약
          Seat foundSeat = seatRepository.findById(seat.getSeatId()).orElseThrow();
          assertThat(foundSeat.isAvailable()).isTrue();

          foundSeat.reserve(userId, 5);
          seatRepository.save(foundSeat);

          // 3. 예약 정보 생성
          Reservation reservation = new Reservation(userId, foundSeat, price);
          reservationRepository.save(reservation);

          // 4. 예약 확정
          reservation.confirm();
          foundSeat.confirmReservation();

          reservationRepository.save(reservation);
          seatRepository.save(foundSeat);

          // 5. 토큰 완료 처리
          token.complete();
          queueTokenRepository.save(token);

          // Then: 최종 상태 검증
          Seat finalSeat = seatRepository.findById(seat.getSeatId()).orElseThrow();
          Reservation finalReservation = reservationRepository.findById(
              reservation.getReservationId()).orElseThrow();
          QueueToken finalToken = queueTokenRepository.findById(
              token.getTokenId()).orElseThrow();

          assertThat(finalSeat.getSeatStatus()).isEqualTo(Seat.SeatStatus.RESERVED);
          assertThat(finalSeat.getReservedUserId()).isEqualTo(userId);

          assertThat(finalReservation.getReservationStatus())
              .isEqualTo(Reservation.ReservationStatus.CONFIRMED);
          assertThat(finalReservation.getConfirmedAt()).isNotNull();

          assertThat(finalToken.getTokenStatus()).isEqualTo(QueueToken.TokenStatus.COMPLETED);
      }
  }
```
  통합 테스트 특징:
  1. @DataJpaTest: JPA 레포지토리만 로드 (빠른 테스트)
  2. H2 In-Memory DB: 격리된 테스트 환경
  3. TestEntityManager: flush/clear로 영속성 컨텍스트 제어
  4. 전체 플로우 검증: 토큰 발급 → 예약 → 결제 → 완료
  5. 상태 전환 검증: 각 단계별 엔티티 상태 확인

  ---
  7. 동시성 통합 테스트
  Q: 동시성 테스트는 통합 테스트에서 어떻게 검증했나요?
  A: CountDownLatch로 실제 동시 요청을 재현하여 좌석 예약 경쟁 조건을 테스트했습니다.

  // ConcertReservationIntegrationTest.java:108-171
```java 
  @Test
  @DisplayName("동시성 테스트 - 여러 사용자가 동시에 같은 좌석 예약 시도")
  void concurrentSeatReservationTest() throws InterruptedException {
      // Given
      Concert concert = new Concert("Concurrent Test Concert", "Test Band", "Test Hall");
      entityManager.persist(concert);

      ConcertSchedule schedule = new ConcertSchedule(
              concert,
              LocalDateTime.now().plusDays(7),
              LocalDateTime.now().minusHours(1)
      );
      entityManager.persist(schedule);

      Seat seat = new Seat(schedule, 1);
      entityManager.persist(seat);

      entityManager.flush();
      entityManager.clear();

      int threadCount = 10;
      ExecutorService executorService = Executors.newFixedThreadPool(threadCount);
      CountDownLatch latch = new CountDownLatch(threadCount);
      AtomicInteger successCount = new AtomicInteger(0);
      AtomicInteger failCount = new AtomicInteger(0);

      // When: 동시에 좌석 예약 시도
      for (int i = 0; i < threadCount; i++) {
          final long userId = i + 1;
          executorService.submit(() -> {
              try {
                  Seat foundSeat = seatRepository.findById(seat.getSeatId()).orElseThrow();

                  if (foundSeat.isAvailable()) {
                      foundSeat.reserve(userId, 5);
                      seatRepository.save(foundSeat);

                      Reservation reservation = new Reservation(userId, foundSeat,
                                                               new BigDecimal("50000"));
                      reservationRepository.save(reservation);

                      successCount.incrementAndGet();
                  } else {
                      failCount.incrementAndGet();
                  }
              } catch (Exception e) {
                  failCount.incrementAndGet();
              } finally {
                  latch.countDown();
              }
          });
      }

      latch.await();
      executorService.shutdown();

      // Then: 오직 한 명만 성공해야 함
      assertThat(successCount.get()).isEqualTo(1);
      assertThat(failCount.get()).isEqualTo(threadCount - 1);

      // 최종 좌석 상태 확인
      Seat finalSeat = seatRepository.findById(seat.getSeatId()).orElseThrow();
      assertThat(finalSeat.getSeatStatus()).isEqualTo(Seat.SeatStatus.TEMPORARY_RESERVED);
      assertThat(finalSeat.getReservedUserId()).isNotNull();
  }
```
  동시성 통합 테스트 포인트:
  1. 실제 DB 사용: JPA + H2로 실제 트랜잭션 검증
  2. 10개 스레드 동시 실행: ExecutorService + CountDownLatch
  3. 1개 좌석에 10명 요청: 경쟁 조건 재현
  4. 정확히 1명만 성공: 동시성 제어 검증
  5. 최종 상태 검증: DB에 저장된 실제 상태 확인

  ---
  8. 테스트 커버리지
  Q: 어떤 종류의 테스트를 작성했나요?
  A: 도메인, 서비스, 통합, 동시성 테스트를 계층별로 작성했습니다.
  프로젝트 테스트 구조 (총 15개 테스트 파일, 132개 테스트 메서드)

  1. 도메인 단위 테스트
     - ProductTest.java: 재고 차감, 소계 계산, 활성화 검증
     - UserTest.java: 포인트 충전, 결제 처리 검증

  2. 서비스 단위 테스트 (Mockito)
     - UserServiceTest.java: 충전/조회 로직 검증
     - OrderServiceTest.java: 주문 생성 로직 검증
     - CouponServiceTest.java: 쿠폰 발급 로직 검증

  3. UseCase 테스트
     - ReservationUseCaseTest.java: 예약 유스케이스 검증
     - QueueManagementUseCaseTest.java: 대기열 관리 검증
     - OrderUseCaseTest.java: 주문 유스케이스 검증

  4. 통합 테스트 (@DataJpaTest)
     - ConcertReservationIntegrationTest.java: 콘서트 예약 전체 플로우
     - ECommerceIntegrationTest.java: 쇼핑몰 전체 플로우
     - OrderIntegrationTest.java: 주문 플로우

  5. 동시성 테스트 (@SpringBootTest)
     - ConcurrencyTestSuite.java: 재고/쿠폰/좌석 경쟁 조건 (6개 테스트)
     - DeadlockAndLockContentionTest.java: 데드락/타임아웃 시나리오

  6. 인프라 테스트
     - RedisIntegrationTest.java: Redis 캐싱/분산락 검증

  7. 컨트롤러 테스트
     - BalanceControllerTest.java: API 레이어 검증

  테스트 전략:
  - 단위 테스트: 도메인 로직, 서비스 로직 격리 검증
  - 통합 테스트: 전체 플로우, DB 상태 전환 검증
  - 동시성 테스트: 경쟁 조건, 데드락, 타임아웃 검증
  - Given-When-Then: 모든 테스트에서 일관된 구조
  - AssertJ: 가독성 높은 assertion

  ---
  9. 클린 아키텍처 적용

  Q: 클린 아키텍처를 어떻게 적용했나요?

  A: 계층을 Domain, Application, Infrastructure로 분리하고 의존성 규칙을 준수했습니다.

프로젝트 구조:

```java 
  src/main/java/kr/hhplus/be/server/
  ├── domain/                         # Domain Layer (순수 비즈니스 로직)
  │   ├── Product.java               # 엔티티 + 비즈니스 로직
  │   ├── User.java
  │   ├── Coupon.java
  │   ├── concert/
  │   │   ├── Seat.java              # 좌석 상태 3단계 로직
  │   │   ├── Reservation.java
  │   │   └── repository/            # 인터페이스 (의존성 역전)
  │   │       ├── SeatRepository.java
  │   │       └── ReservationRepository.java
  │   └── queue/
  │       └── QueueToken.java        # 대기열 토큰 로직
  │
  ├── application/                    # Application Layer (유스케이스)
  │   ├── concert/
  │   │   └── ReservationUseCase.java  # 예약 유스케이스
  │   ├── queue/
  │   │   └── QueueManagementUseCase.java  # 대기열 관리
  │   └── ecommerce/
  │       └── OrderUseCase.java        # 주문 유스케이스
  │
  ├── infrastructure/                  # Infrastructure Layer (구현체)
  │   ├── concert/
  │   │   ├── JpaSeatRepository.java   # JPA 구현체
  │   │   └── JpaReservationRepository.java
  │   ├── redis/
  │   │   ├── RedisDistributedLock.java  # 분산락 구현
  │   │   └── SeatCacheService.java      # 캐싱 구현
  │   └── ecommerce/
  │       └── JpaOrderRepository.java
  │
  ├── controller/                      # Presentation Layer (API)
  │   ├── BalanceController.java
  │   ├── OrderController.java
  │   └── concert/
  │       └── ConcertController.java
  │
  └── service/                         # Service Layer (레거시 구조)
      ├── UserService.java
      └── CouponService.java
```
  의존성 규칙:
  1. Domain ← Application ← Infrastructure: 안쪽에서 바깥쪽으로만 의존
  2. 인터페이스 기반: Domain에 Repository 인터페이스, Infrastructure에 구현체
  3. 순수 비즈니스 로직: Domain은 외부 의존성 없음 (JPA 제외)
  4. 유스케이스 중심: Application 레이어에 비즈니스 플로우 구현

  ---
  10. 결제 미완료 시 자동 해제
  Q: 좌석 예약 후 결제 미완료 시 자동 해제는 어떻게 구현했나요?
  A: 도메인 엔티티에서 만료 시간 체크 + isAvailable() 호출 시 자동 해제하는 방식으로 구현했습니다.

  // Seat.java:57-100 (앞서 설명한 코드 재활용)

```java 
  public void reserve(Long userId, int temporaryReservationMinutes) {
      if (!isAvailable()) {
          throw new IllegalStateException("Seat is not available for reservation");
      }

      this.seatStatus = SeatStatus.TEMPORARY_RESERVED;
      this.reservedUserId = userId;
      this.reservedAt = LocalDateTime.now();
      this.expiresAt = LocalDateTime.now().plusMinutes(temporaryReservationMinutes);  // 5분 설정
  }

  public boolean isAvailable() {
      if (seatStatus == SeatStatus.AVAILABLE) {
          return true;
      }

      // 임시 예약이 만료된 경우 자동으로 해제 (핵심 로직)
      if (seatStatus == SeatStatus.TEMPORARY_RESERVED && isExpired()) {
          release();  // 자동 해제
          return true;
      }

      return false;
  }

  public boolean isExpired() {
      return expiresAt != null && LocalDateTime.now().isAfter(expiresAt);
  }

  public void release() {
      this.seatStatus = SeatStatus.AVAILABLE;
      this.reservedUserId = null;
      this.reservedAt = null;
      this.expiresAt = null;
  }
```
  자동 해제 구현 전략:
  1. 임시 예약 시 만료 시간 설정: 5분 (설정 가능)
  2. Lazy Evaluation: isAvailable() 호출 시 만료 체크
  3. 자동 해제: 만료된 경우 release() 호출로 상태 초기화
  4. 별도 스케줄러 불필요: 조회 시점에 자동 처리
  5. 통합 테스트로 검증: ConcertReservationIntegrationTest.java:210-251

  ---
  요약
  프로젝트의 API 설계 및 TDD 적용 핵심:
  API 설계
  1. RESTful 원칙: 리소스 중심, HTTP 메서드 활용, 계층 구조
  2. 대기열 토큰: Header 기반 인증으로 순서 제어
  3. 페이지네이션: 대용량 데이터 처리
  4. 버전 관리: /api/v1 구조

  TDD/테스트

  1. 도메인 우선 테스트: Product, User 등 엔티티 단위 테스트
  2. Mockito 활용: 서비스 계층 격리 테스트
  3. 통합 테스트: @DataJpaTest로 전체 플로우 검증
  4. 동시성 테스트: CountDownLatch로 경쟁 조건 재현
  5. Given-When-Then: 일관된 테스트 구조

  클린 아키텍처

  1. 계층 분리: Domain → Application → Infrastructure
  2. 의존성 역전: Repository 인터페이스로 결합도 낮춤
  3. 유스케이스 중심: 비즈니스 플로우를 UseCase로 표현


### 코드질문
  1. Redis 분산락 구현 방식
  Q: Redisson을 사용한 분산락을 어떻게 구현했나요?
  A: Redisson의 RLock을 사용하여 분산락을 구현했습니다.

  // RedisDistributedLock.java:32-56
```java 
  public <T> T executeWithLock(String lockKey, long waitTime, long leaseTime, Supplier<T> supplier) {
      RLock lock = redissonClient.getLock(lockKey);

      try {
          boolean isLockAcquired = lock.tryLock(waitTime, leaseTime, TimeUnit.SECONDS);

          if (!isLockAcquired) {
              log.warn("Failed to acquire lock for key: {}", lockKey);
              throw new IllegalStateException("Could not acquire lock for key: " + lockKey);
          }

          log.debug("Lock acquired for key: {}", lockKey);
          return supplier.get();

      } catch (InterruptedException e) {
          Thread.currentThread().interrupt();
          throw new IllegalStateException("Lock acquisition was interrupted", e);
      } finally {
          if (lock.isLocked() && lock.isHeldByCurrentThread()) {
              lock.unlock();
          }
      }
  }
```
  핵심:
  - tryLock(waitTime, leaseTime): 락 획득 대기 시간과 보유 시간 설정
  - 기본값: 대기 5초, 보유 10초
  - isHeldByCurrentThread() 체크로 다른 스레드의 락 해제 방지
  - InterruptedException 처리로 스레드 안전성 보장

  ---
  2. 좌석 예약 분산락 적용
  Q: 동일 좌석 중복 예약 방지를 위해 분산락을 어떻게 적용했나요?
  A: 좌석 ID 기반 락 키로 동시 예약을 제어했습니다.

  // ReservationUseCase.java:44-77
```java 
  @Transactional
  public ReservationResult reserveSeat(ReserveSeatCommand command) {
      String lockKey = "seat:reserve:" + command.getSeatId();

      return distributedLock.executeWithLock(lockKey, 3, 10, () -> {
          // 1. 토큰 검증
          QueueToken token = queueTokenRepository.findByTokenUuid(command.getTokenUuid())
                  .orElseThrow(() -> new IllegalArgumentException("Invalid token"));

          if (!token.isActive()) {
              throw new IllegalStateException("Token is not active");
          }

          // 2. 좌석 조회 및 예약 가능성 확인
          Seat seat = seatRepository.findById(command.getSeatId())
                  .orElseThrow(() -> new IllegalArgumentException("Seat not found"));

          if (!seat.isAvailable()) {
              throw new IllegalStateException("Seat is not available");
          }

          // 3. 좌석 임시 예약 (5분)
          seat.reserve(command.getUserId(), 5);
          seatRepository.save(seat);

          // 4. 좌석 배치도 캐시 무효화
          seatCacheService.invalidateSeatLayout(seat.getSchedule().getScheduleId());

          // 5. 예약 정보 저장
          Reservation reservation = new Reservation(command.getUserId(), seat, command.getPrice());
          reservationRepository.save(reservation);

          return new ReservationResult(...);
      });
  }
```
  핵심:
  - 좌석별 독립적인 락: "seat:reserve:{seatId}"
  - Lock Granularity를 좌석 단위로 설정하여 성능 최적화
  - 대기 3초, 보유 10초로 적절한 타임아웃 설정

  ---
  3. 좌석 상태 3단계 전환
  Q: 좌석 상태를 3단계로 관리한 이유와 구현 방법은?
  A: AVAILABLE → TEMPORARY_RESERVED → RESERVED 3단계로 관리하여 결제 미완료 시 자동 해제를 구현했습니다.

  // Seat.java:142-146
```java 
  public enum SeatStatus {
      AVAILABLE,           // 예약 가능
      TEMPORARY_RESERVED,  // 임시 예약 (5분간)
      RESERVED             // 확정 예약
  }
```
  // Seat.java:57-66 - 임시 예약
```java 
  public void reserve(Long userId, int temporaryReservationMinutes) {
      if (!isAvailable()) {
          throw new IllegalStateException("Seat is not available for reservation");
      }

      this.seatStatus = SeatStatus.TEMPORARY_RESERVED;
      this.reservedUserId = userId;
      this.reservedAt = LocalDateTime.now();
      this.expiresAt = LocalDateTime.now().plusMinutes(temporaryReservationMinutes);
  }
```
  // Seat.java:68-75 - 확정 예약

```java 
  public void confirmReservation() {
      if (seatStatus != SeatStatus.TEMPORARY_RESERVED) {
          throw new IllegalStateException("Seat is not temporarily reserved");
      }

      this.seatStatus = SeatStatus.RESERVED;
      this.expiresAt = null; // 확정 예약 시 만료 시간 제거
  }
```

  // Seat.java:84-96 - 자동 해제 로직
```java 
  public boolean isAvailable() {
      if (seatStatus == SeatStatus.AVAILABLE) {
          return true;
      }

      // 임시 예약이 만료된 경우 자동으로 해제
      if (seatStatus == SeatStatus.TEMPORARY_RESERVED && isExpired()) {
          release();
          return true;
      }

      return false;
  }
```
  핵심:
  - 임시 예약(5분): 결제 진행 시간 확보
  - 만료 시간 체크로 자동 해제 (별도 스케줄러 불필요)
  - isAvailable() 호출 시 만료 체크로 lazy evaluation

  ---
  4. 선착순 쿠폰 발급 동시성 제어
  Q: 선착순 쿠폰 발급에서 오버 이슈를 어떻게 방지했나요?
  A: 비관적 락으로 쿠폰 테이블을 잠그고 발급 수량을 원자적으로 증가시켰습니다.

  // CouponRepository.java:16-18 - 비관적 락
```java 
  @Lock(LockModeType.PESSIMISTIC_WRITE)
  @Query("SELECT c FROM Coupon c WHERE c.couponId = :couponId")
  Optional<Coupon> findByIdWithLock(@Param("couponId") Long couponId);
```
  // CouponService.java:40-64 - 쿠폰 발급 로직
```java 
  @Transactional
  public UserCoupon issueCoupon(Long userId, Long couponId) {
      // 1. 사용자 존재 확인
      User user = userService.getUser(userId);

      // 2. 이미 발급받은 쿠폰인지 확인
      if (userCouponRepository.existsByUserIdAndCouponId(userId, couponId)) {
          throw new BusinessException(ErrorCode.COUPON_ALREADY_ISSUED);
      }

      // 3. 쿠폰 조회 및 비관적 락 적용
      Coupon coupon = couponRepository.findByIdWithLock(couponId)
              .orElseThrow(() -> new BusinessException(ErrorCode.COUPON_NOT_FOUND));

      // 4. 쿠폰 발급 가능 여부 검증 및 발급 수량 증가
      coupon.issueCoupon();

      // 5. 사용자-쿠폰 관계 생성
      UserCoupon userCoupon = new UserCoupon(userId, couponId);
      return userCouponRepository.save(userCoupon);
  }
```
  // Coupon.java:60-79 - 도메인 로직에서 검증
```java 
  public void issueCoupon() {
      if (!isActive) {
          throw new BusinessException(ErrorCode.COUPON_NOT_ACTIVE);
      }

      if (issuedQuantity >= totalQuantity) {
          throw new BusinessException(ErrorCode.COUPON_SOLD_OUT);
      }

      this.issuedQuantity++;  // 원자적 증가
  }
```
  핵심:
  - PESSIMISTIC_WRITE 락으로 행 단위 잠금
  - 도메인 엔티티에서 비즈니스 로직 검증
  - 트랜잭션 단위로 수량 증가 원자성 보장

  ---
  5. Redis 캐싱 전략
  Q: 좌석 배치도를 Redis에 캐싱한 이유와 무효화 전략은?
  A: 좌석 배치는 조회가 빈번하지만 변경이 적어 Cache-Aside 패턴을 적용했습니다.

  // SeatCacheService.java:19-22 - TTL 설정
```java 
  private static final String SEAT_LAYOUT_PREFIX = "seat:layout:";
  private static final Duration LAYOUT_CACHE_EXPIRY = Duration.ofHours(2); // 2시간
```
  // SeatCacheService.java:35-43 - 캐시 저장
```java 
  public void cacheSeatLayout(Long scheduleId, List<SeatLayoutDto> seatLayout) {
      try {
          String key = SEAT_LAYOUT_PREFIX + scheduleId;
          redisTemplate.opsForValue().set(key, seatLayout, LAYOUT_CACHE_EXPIRY);
          log.debug("Cached seat layout for schedule: {}", scheduleId);
      } catch (Exception e) {
          log.error("Failed to cache seat layout for schedule: {}", scheduleId, e);
      }
  }
```
  // SeatCacheService.java:48-65 - 캐시 조회
```java 
  public List<SeatLayoutDto> getCachedSeatLayout(Long scheduleId) {
      try {
          String key = SEAT_LAYOUT_PREFIX + scheduleId;
          Object cached = redisTemplate.opsForValue().get(key);

          if (cached instanceof List) {
              log.debug("Cache hit for seat layout: {}", scheduleId);
              return (List<SeatLayoutDto>) cached;
          }

          log.debug("Cache miss for seat layout: {}", scheduleId);
          return null;
      } catch (Exception e) {
          log.error("Failed to get cached seat layout", e);
          return null; // Redis 장애 시 DB Fallback
      }
  }
```
  // ReservationUseCase.java:167-180 - Cache-Aside 패턴
```java 
  public List<SeatCacheService.SeatLayoutDto> getSeatLayout(Long scheduleId) {
      // 캐시에서 먼저 조회
      List<SeatCacheService.SeatLayoutDto> cachedLayout =
          seatCacheService.getCachedSeatLayout(scheduleId);

      if (cachedLayout != null) {
          return cachedLayout;
      }

      // 캐시 미스인 경우 DB 조회 후 캐시 저장
      List<SeatCacheService.SeatLayoutDto> seatLayout =
          seatRepository.findSeatLayoutByScheduleId(scheduleId);
      seatCacheService.cacheSeatLayout(scheduleId, seatLayout);

      return seatLayout;
  }
```
  // SeatCacheService.java:103-111 - 캐시 무효화
```java 
  public void invalidateSeatLayout(Long scheduleId) {
      try {
          String key = SEAT_LAYOUT_PREFIX + scheduleId;
          redisTemplate.delete(key);
          log.debug("Invalidated seat layout cache for schedule: {}", scheduleId);
      } catch (Exception e) {
          log.error("Failed to invalidate seat layout cache", e);
      }
  }
```

  핵심:
  - TTL 2시간: 좌석 배치는 자주 변경되지 않음
  - Cache-Aside 패턴: 애플리케이션에서 캐시 관리
  - 예약/결제 시 캐시 무효화로 정합성 보장
  - Redis 장애 시 null 반환으로 DB Fallback

  ---
  6. 재고 차감 동시성 제어
  Q: 동시 주문 시 재고 차감 정확성을 어떻게 보장했나요?
  A: 비관적 락과 도메인 로직 검증을 조합했습니다.

  // OrderService.java:46-82 - 주문 처리
```java 
  @Transactional
  public OrderResponse createOrder(OrderRequest orderRequest) {
      Long userId = orderRequest.getUserId();

      // 1. 사용자 조회 (비관적 락)
      User user = userService.getUserWithLock(userId);

      // 2. 상품 조회 및 재고 확인 (비관적 락)
      Map<Long, Product> productMap = new HashMap<>();
      BigDecimal totalAmount = BigDecimal.ZERO;

      for (OrderRequest.OrderItemRequest itemRequest : orderRequest.getOrderItems()) {
          Product product = productService.getProductWithLock(itemRequest.getProductId());
          productService.validateStock(product, itemRequest.getQuantity());

          productMap.put(product.getProductId(), product);
          totalAmount = totalAmount.add(product.calculateSubtotal(itemRequest.getQuantity()));
      }

      // 3. 잔액 확인
      if (!user.hasEnoughBalance(totalAmount)) {
          throw new BusinessException(ErrorCode.INSUFFICIENT_BALANCE);
      }

      // 4. 주문 생성 및 재고 차감
      Order order = new Order(userId, totalAmount);

      for (OrderRequest.OrderItemRequest itemRequest : orderRequest.getOrderItems()) {
          Product product = productMap.get(itemRequest.getProductId());

          OrderItem orderItem = OrderItem.create(product, itemRequest.getQuantity());
          order.addOrderItem(orderItem);

          productService.deductStock(product, itemRequest.getQuantity());
      }

      // 5. 결제 및 주문 완료
      userService.processPayment(user, totalAmount, null);
      order.complete();

      return new OrderResponse(orderRepository.save(order), user.getBalance());
  }
```
  // Product.java:59-67 - 재고 차감 검증
    ```java 
  public void deductStock(Integer quantity) {
      if (quantity == null || quantity <= 0) {
          throw new IllegalArgumentException("차감할 수량은 0보다 커야 합니다.");
      }
      if (this.stockQuantity < quantity) {
          throw new IllegalStateException(
              "재고가 부족합니다. 요청: " + quantity + "개, 현재 재고: " + this.stockQuantity + "개"
          );
      }
      this.stockQuantity -= quantity;
  }
```
  핵심:
  - getProductWithLock(): 비관적 락으로 상품 조회
  - 도메인 엔티티에서 재고 검증 로직 캡슐화
  - 트랜잭션 격리로 원자적 재고 차감 보장

  ---
  7. 대기열 토큰 시스템
  Q: 대기열 토큰의 순차 활성화를 어떻게 구현했나요?
  A: 스케줄러로 30초마다 대기 큐에서 최대 100개씩 활성화합니다.

  // QueueManagementUseCase.java:16-17 - 상수 정의
```java 
  private static final int MAX_ACTIVE_TOKENS = 100;
  private static final int ACTIVE_DURATION_MINUTES = 10;
```

  // QueueManagementUseCase.java:71-102 - 스케줄러
```java 
  @Scheduled(fixedRate = 30000) // 30초마다 실행
  @Transactional
  public void processQueue() {
      // 1. 만료된 토큰들 정리
      List<QueueToken> expiredTokens = queueTokenRepository.findExpiredTokens();
      for (QueueToken token : expiredTokens) {
          if (!token.isExpired()) {
              token.expire();
              queueTokenRepository.save(token);
          }
      }

      // 2. 현재 활성화된 토큰 수 확인
      List<QueueToken> activeTokens = queueTokenRepository.findActiveTokens();
      int currentActiveCount = activeTokens.size();

      // 3. 새로운 토큰들을 활성화
      int availableSlots = MAX_ACTIVE_TOKENS - currentActiveCount;
      if (availableSlots > 0) {
          List<QueueToken> waitingTokens = queueTokenRepository.findWaitingTokens();
          int tokensToActivate = Math.min(availableSlots, waitingTokens.size());

          for (int i = 0; i < tokensToActivate; i++) {
              QueueToken token = waitingTokens.get(i);
              token.activate(ACTIVE_DURATION_MINUTES);
              queueTokenRepository.save(token);
          }

          // 4. 대기 중인 토큰들의 순서 업데이트
          updateQueuePositions();
      }
  }
```
  // QueueManagementUseCase.java:113-120 - 순서 업데이트
```java 
  private void updateQueuePositions() {
      List<QueueToken> waitingTokens = queueTokenRepository.findWaitingTokens();
      for (int i = 0; i < waitingTokens.size(); i++) {
          QueueToken token = waitingTokens.get(i);
          token.updatePosition((long) (i + 1));
          queueTokenRepository.save(token);
      }
  }
```

  핵심:
  - 동시 활성화 제한: 최대 100명
  - 토큰 활성 시간: 10분
  - 30초마다 자동으로 다음 대기자 활성화
  - 만료 토큰 자동 정리로 슬롯 확보

  ---
  8. Redis 설정
  Q: Redisson 커넥션 풀을 어떻게 설정했나요?
  A: 커넥션 풀 크기, 타임아웃, 재시도 전략을 세밀하게 설정했습니다.

  // RedisConfig.java:59-78
```java 
  @Bean
  public RedissonClient redissonClient() {
      Config config = new Config();

      String redisAddress = String.format("redis://%s:%d", host, port);
      config.useSingleServer()
              .setAddress(redisAddress)
              .setConnectionPoolSize(10)           // 최대 10개 연결
              .setConnectionMinimumIdleSize(2)     // 최소 유지 2개
              .setIdleConnectionTimeout(10000)     // 유휴 연결 10초 후 해제
              .setConnectTimeout(10000)            // 연결 타임아웃 10초
              .setTimeout(3000)                    // 명령 타임아웃 3초
              .setRetryAttempts(3)                 // 재시도 3회
              .setRetryInterval(1500);             // 재시도 간격 1.5초

      if (password != null && !password.isEmpty()) {
          config.useSingleServer().setPassword(password);
      }

      return Redisson.create(config);
  }
```
  핵심:
  - 커넥션 풀: 최대 10개로 다중 인스턴스 대응
  - 타임아웃 전략: 연결 10초, 명령 3초
  - 재시도: 3회 시도 + 1.5초 간격으로 일시적 장애 대응

  ---
  9. 동시성 테스트
  Q: 동시성 테스트는 어떻게 작성했나요?
  A: CountDownLatch와 ExecutorService로 실제 동시 요청을 재현했습니다.

  // ConcurrencyTestSuite.java:53-103 - 재고 차감 테스트
```java 
  @Test
  @DisplayName("동시성 테스트 1: 재고 차감 경쟁 조건")
  void concurrentStockDecrementTest() throws InterruptedException {
      // Given: 재고 10개인 상품
      Product product = new Product("Limited Product", new BigDecimal("10000"), 10);
      productRepository.save(product);

      // 충분한 잔액을 가진 사용자들 생성
      int userCount = 15;
      for (int i = 1; i <= userCount; i++) {
          User user = new User("user" + i + "@test.com", "User " + i, new BigDecimal("50000"));
          userRepository.save(user);
      }

      int threadCount = 15;
      ExecutorService executorService = Executors.newFixedThreadPool(threadCount);
      CountDownLatch latch = new CountDownLatch(threadCount);
      AtomicInteger successCount = new AtomicInteger(0);
      AtomicInteger failCount = new AtomicInteger(0);

      // When: 15명이 동시에 1개씩 주문 (재고는 10개)
      for (int i = 0; i < threadCount; i++) {
          final long userId = i + 1;
          executorService.submit(() -> {
              try {
                  OrderUseCase.OrderCommand command = new OrderUseCase.OrderCommand(
                          userId,
                          Arrays.asList(new OrderUseCase.OrderItemRequest(product.getProductId(), 1)),
                          UUID.randomUUID().toString()
                  );

                  orderUseCase.processOrder(command);
                  successCount.incrementAndGet();
              } catch (Exception e) {
                  failCount.incrementAndGet();
              } finally {
                  latch.countDown();
              }
          });
      }

      latch.await();
      executorService.shutdown();

      // Then: 정확히 10명만 성공해야 함
      assertThat(successCount.get()).isEqualTo(10);
      assertThat(failCount.get()).isEqualTo(5);

      // 최종 재고 확인
      Product finalProduct = productRepository.findById(product.getProductId()).orElseThrow();
      assertThat(finalProduct.getStock()).isEqualTo(0);
  }
```
  핵심:
  - CountDownLatch: 모든 스레드 동시 실행 보장
  - AtomicInteger: 스레드 안전한 카운터
  - ExecutorService: 고정 크기 스레드 풀로 동시성 제어
  - 재고 10개에 15명 요청 → 정확히 10명 성공, 5명 실패 검증

  ---
  10. HikariCP 설정
  Q: HikariCP 커넥션 풀을 최대 3개로 설정한 이유는?
  A: 로컬 개발 환경 리소스 절약과 커넥션 부족 시나리오 테스트를 위해 설정했습니다.

  # application.yml:12-16
```java 
  datasource:
    hikari:
      maximum-pool-size: 3      # 최대 커넥션 3개
      minimum-idle: 1           # 최소 유지 1개
      connection-timeout: 10000 # 연결 대기 10초
      max-lifetime: 60000       # 최대 생명 주기 60초
```
  핵심:
  - 로컬 환경: MySQL 기본 max_connections 대비 적절한 수
  - 부하 테스트: 커넥션 풀 고갈 시나리오 재현
  - 실무 환경: 인스턴스 수와 DB 성능에 맞게 조정 필요 (보통 10~20)
  ---
  요약
  이 프로젝트에서는 다음과 같은 핵심 기술을 구현했습니다:
  1. Redisson 분산락: 좌석 예약, 재고 차감, 쿠폰 발급의 동시성 제어
  2. Redis 캐싱: Cache-Aside 패턴으로 좌석 배치도 조회 최적화
  3. 비관적 락: 선착순 쿠폰 발급에서 PESSIMISTIC_WRITE 적용
  4. 3단계 좌석 상태: 임시 예약 → 확정 예약으로 결제 프로세스 관리
  5. 대기열 시스템: 스케줄러 기반 토큰 순차 활성화
  6. 동시성 테스트: CountDownLatch로 실제 경쟁 조건 재현

### 심화질문
  1. 트래픽 대응 및 성능 최적화
  Redis 캐싱
  - Q1: 좌석 배치도를 Redis에 캐싱한 이유와 캐시 무효화 전략은?
  - Q2: Redis 캐시의 TTL은 어떻게 설정했나요? 데이터 정합성은 어떻게 보장했나요?
  - Q3: Cache Aside vs Write Through 패턴 중 어떤 것을 선택했고 그 이유는?
  - Q4: Redis 장애 시 fallback 전략은?
  Redis 분산락
  - Q5: Redisson을 선택한 이유는? (Lettuce나 다른 대안 대비)
  - Q6: 분산락의 타임아웃은 어떻게 설정했나요? 락 획득 실패 시 재시도 로직은?
  - Q7: 락의 granularity(좌석 단위? 주문 단위?)는 어떻게 결정했나요?
  - Q8: 데드락 상황을 어떻게 테스트하고 해결했나요?
  - Q9: Redis 싱글 스레드 특성 때문에 발생할 수 있는 병목은 없었나요?
  Kafka 이벤트 처리
  - Q10: Kafka를 도입한 이유는? (동기 vs 비동기 처리 선택 기준)
  - Q11: 어떤 이벤트를 Kafka로 처리했나요? (주문 완료, 결제, 쿠폰 발급 등)
  - Q12: 메시지 유실을 방어하기 위한 방법은? (acks, retries, idempotent)
  - Q13: Consumer의 병렬 처리는 어떻게 구성했나요? (파티션 개수, Consumer Group)
  - Q14: 이벤트 순서 보장이 필요한 경우는 어떻게 처리했나요?

  2. 동시성 제어
  재고 관리
  - Q15: 동시 주문 시 재고 차감 정확성을 어떻게 보장했나요?
  - Q16: 낙관적 락 vs 비관적 락 vs 분산락 중 왜 분산락을 선택했나요?
  - Q17: 재고 복구 시나리오(결제 실패, 주문 취소)는 어떻게 처리했나요?
  - Q18: 재고 부족 시 사용자 경험은 어떻게 개선했나요?
  선착순 쿠폰
  - Q19: 선착순 쿠폰 발급에서 오버 이슈(Over-Issuing)를 어떻게 방지했나요?
  - Q20: 100명 한정 쿠폰에 1000명이 동시 요청 시 처리 방식은?
  - Q21: 쿠폰 발급 실패 시 사용자에게 어떤 응답을 주나요?
  좌석 예약
  - Q22: 좌석 상태 3단계 전환(예약 가능→임시 배정→확정)의 이유는?
  - Q23: 결제 미완료 시 자동 해제 구현 방법은? (스케줄러? 이벤트?)
  - Q24: 동일 좌석 중복 예약 방지를 분산락으로 해결한 과정은?

  3. API 설계
  REST API 설계
  - Q25: RESTful API 설계 원칙을 어떻게 적용했나요? (리소스 명명, HTTP 메서드)
  - Q26: API 버전 관리 전략은?
  - Q27: 멱등성(Idempotency)이 필요한 API는 어떻게 처리했나요?
  - Q28: 포인트 충전/조회 API의 동시성 이슈는?
  대기열 시스템
  - Q29: 대기열 토큰 시스템의 동작 원리는?
  - Q30: 토큰 순차 활성화는 어떻게 구현했나요? (Redis Sorted Set?)
  - Q31: 동시 활성화 제한을 어떻게 제어했나요?
  - Q32: 대기 시간 예측 기능은 어떻게 구현했나요?
  - Q33: 토큰 만료 정책은?

  4. 아키텍처 및 설계
  클린 아키텍처
  - Q34: 클린 아키텍처를 적용한 이유와 장점은?
  - Q35: 계층 간(Domain, Application, Infrastructure) 의존성 규칙은?
  - Q36: Entity vs DTO vs Domain Model 분리 기준은?

  다중 인스턴스 환경
  - Q37: 다중 인스턴스에서 분산락 없이 발생할 수 있는 문제는?
  - Q38: 세션 관리는 어떻게 했나요? (Redis Session?)
  - Q39: 로드 밸런서 설정은?

  5. DB 최적화
  인덱싱
  - Q40: 어떤 컬럼에 복합 인덱스를 적용했고 그 이유는?
  - Q41: 인덱스 적용 전후 성능 차이는?
  - Q42: 카디널리티가 낮은 컬럼(예: 좌석 상태)의 인덱스 전략은?
  테이블 파티셔닝
  - Q43: 어떤 테이블을 파티셔닝했고 기준은? (Range? Hash?)
  - Q44: 파티셔닝 전후 쿼리 성능 개선 수치는?
  - Q45: 파티션 프루닝(Pruning) 효과를 확인한 방법은?
  커넥션 풀
  - Q46: HikariCP 최대 연결 수를 3으로 설정한 이유는?
  - Q47: 트래픽 급증 시 커넥션 풀 부족 문제는 없었나요?

  6. 테스트 및 검증
  TDD 및 테스트
  - Q48: TDD를 어떻게 적용했나요? (Red-Green-Refactor)
  - Q49: 단위 테스트 vs 통합 테스트 범위는 어떻게 나눴나요?
  - Q50: Testcontainers를 사용한 이유는?
  - Q51: 동시성 테스트는 어떻게 작성했나요? (CountDownLatch? ExecutorService?)
  부하 테스트
  - Q52: 부하 테스트 도구는? (JMeter? nGrinder? K6?)
  - Q53: TPS 목표치와 실제 달성한 수치는?
  - Q54: 데드락 시나리오를 어떻게 재현하고 해결했나요?
  - Q55: 타임아웃 설정값은 어떻게 결정했나요?

  7. 장애 대응 및 모니터링
  장애 대응
  - Q56: Redis 장애 시 시스템 동작 방식은?
  - Q57: Kafka Consumer lag 증가 시 대응 방법은?
  - Q58: 데이터베이스 락 타임아웃 발생 시 처리 방식은?
  - Q59: 롤백 전략은? (보상 트랜잭션? Saga 패턴?)
  모니터링
  - Q60: 어떤 메트릭을 모니터링했나요? (응답 시간, 에러율, TPS)
  - Q61: 로깅 전략은? (ELK? CloudWatch?)
  - Q62: 알림(Alert) 기준은?
  8. 트레이드오프 및 의사결정
  - Q63: 성능과 데이터 정합성 사이에서 어떤 선택을 했나요?
  - Q64: 동기 처리 vs 비동기 처리 선택 기준은?
  - Q65: Eventually Consistent를 허용한 부분은?
  - Q66: 가장 어려웠던 기술적 의사결정은?

  9. 개선 사항 및 회고
  - Q67: 프로젝트에서 가장 개선하고 싶은 부분은?
  - Q68: 더 큰 트래픽(10배)을 처리하려면 어떤 개선이 필요할까요?
  - Q69: 이 프로젝트를 통해 배운 가장 중요한 교훈은?
  - Q70: 실무에서 다시 구현한다면 어떻게 다르게 할 건가요?



========================================
  // 1. DTO 객체 생성 (메모리)
  SeatLayoutDto dto = new SeatLayoutDto(...);
  // 2. RedisTemplate이 Redis 서버(localhost:6379)에 저장
  redisTemplate.opsForValue().set("key", dto, Duration.ofHours(2));
  // → dto를 JSON으로 변환 → TCP/IP로 Redis 서버 전송 → Redis 메모리에 저장
  // 3. Redis 서버에서 가져오기
  Object cached = redisTemplate.opsForValue().get("key");
  // → Redis 서버에서 JSON 조회 → DTO 객체로 변환 → 반환
  RedisTemplate = Redis 서버와 통신하는 도구
  1. 첫 요청:
     클라이언트 → Spring → Redis(없음) → DB 조회 → DTO 변환 → Redis 저장 → 응답
  2. 두 번째 요청 (2시간 내):
     클라이언트 → Spring → Redis(있음) → 바로 응답 (DB 접근 X)

  // SeatLayoutDto 구조 (SeatCacheService.java:126-169):
  ```java 
  {
    "seatId": 456,
    "seatNumber": 12,
    "seatGrade": "VIP",
    "price": 150000,
    "rowNumber": 3,
    "columnNumber": 4,
    "isAvailable": true
  }
```
 이 JSON 데이터가 Redis 메모리에 2시간 동안 저장되어, DB 부하를 줄이고 응답 속도를 높입니다.
 캐시 무효화는 예약/결제 시 seatCacheService.invalidateSeatLayout()로 삭제합니다.

 // RedisTemplate 설정 (RedisConfig.java:30-57):
   ```java 
  @Bean
  public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory connectionFactory) {
      RedisTemplate<String, Object> template = new RedisTemplate<>();
      template.setConnectionFactory(connectionFactory);  // Redis 서버 연결
      // JSON 직렬화 설정
      GenericJackson2JsonRedisSerializer jackson2JsonRedisSerializer =
          new GenericJackson2JsonRedisSerializer(objectMapper);
      template.setKeySerializer(new StringRedisSerializer());
      template.setValueSerializer(jackson2JsonRedisSerializer);  // 객체→JSON 변환
      return template;
```
============================
 // 분산락 함수  (RedisDistributedLock.java:32-56):
   ```java 
  public <T> T executeWithLock(String lockKey, long waitTime, long leaseTime, Supplier<T> supplier) {
      RLock lock = redissonClient.getLock(lockKey);  // Redis에 락 생성
      boolean isLockAcquired = lock.tryLock(waitTime, leaseTime, TimeUnit.SECONDS);
      // waitTime: 락 획득 대기 시간
      // leaseTime: 락 보유 시간
      if (!isLockAcquired) {
          throw new IllegalStateException("Could not acquire lock");
      }
      return supplier.get();  // 비즈니스 로직 실행
      // finally 블록에서 자동으로 lock.unlock()

  // 실제 사용 - 좌석 예약 (ReservationUseCase.java:44-93):
  public ReservationResult reserveSeat(ReserveSeatCommand command) {
      String lockKey = "seat:reserve:" + command.getSeatId();  // 좌석별 락
      return distributedLock.executeWithLock(lockKey, 3, 10, () -> {    //####################### 
          // 이 블록은 한번에 한 스레드만 실행 가능
          // 1. 좌석 조회
          Seat seat = seatRepository.findById(command.getSeatId());
          // 2. 예약 가능 확인
          if (!seat.isAvailable()) {
              throw new IllegalStateException("Seat is not available");
          }
          // 3. 좌석 예약 처리
          seat.reserve(command.getUserId(), 5);
          seatRepository.save(seat);
          return new ReservationResult(...);
      });
  }

  // 분산락 동작 시나리오:
  사용자 A, B가 동시에 좌석 #12 예약 시도:
  1. A 요청 → "seat:reserve:12" 락 획득 ✓ → 예약 진행
  2. B 요청 → "seat:reserve:12" 락 대기 (3초까지)
  3. A 완료 → 락 해제
  4. B 진입 → 좌석 이미 예약됨 확인 → 예외 발생
  결제 처리 분산락 (ReservationUseCase.java:96-155):
  String lockKey = "payment:" + command.getReservationId();  // 예약건별 락
  distributedLock.executeWithLock(lockKey, 3, 10, () -> {    //###################### 
      // 중복 결제 방지
      userBalanceService.deductBalance(...);  // 잔액 차감
      paymentService.processPayment(...);     // 결제 처리
      reservation.confirm();                   // 예약 확정
  });
  핵심: 동일한 좌석/예약에 대한 동시 요청을 순차 처리하여 데이터 정합성 보장
```
  

      





  

