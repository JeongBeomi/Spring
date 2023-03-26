## 0. 목차
- 조회용 샘플 데이터 입력
- 주문 조회 V1:엔티티를 직접 노출
- 주문 조회 V2:엔티티를 DTO로 변환
- 주문 조회 V3:엔티티를 DTO로 변환 - 페치 조인 최적화
- 주문 조회 V4:JPA에서 DTO로 바로 조회


> 주문 + 배송정보 + 회원을 조회하는 API를 만들기
> 지연 로딩 때문에 발생하는 성능 문제를 단계적으로 해결

## 1. 조회용 샘플 데이터 입력

### 1-1. 샘플 데이터 자동입력
- jpabook.jpashop/InitDb.class
  ```java
  @Component
  @RequiredArgsConstructor
  public class InitDb {
      private final InitService initService;
      
      @PostConstruct
      public void init() {
          initService.dbInit1();
          initService.dbInit2();
      }

      @Component
      @Transactional
      @RequiredArgsConstructor
      static class InitService {

          private final EntityManager em;

          public void dbInit1() {

              Member member = createMember("userA", "서울", "1", "1111");
              em.persist(member);

              Book book1 = createBook("JPA1 BOOK", 10000, 100);
              em.persist(book1);

              Book book2 = createBook("JPA2 BOOK", 20000, 100);
              em.persist(book2);

              OrderItem orderItem1 = OrderItem.createOrderItem(book1, 10000, 1);
              OrderItem orderItem2 = OrderItem.createOrderItem(book2, 20000, 2);
              Order order = Order.createOrder(member, createDelivery(member),
              orderItem1, orderItem2);
              em.persist(order);
          }

          public void dbInit2() {

              Member member = createMember("userB", "진주", "2", "2222");
              em.persist(member);

              Book book1 = createBook("SPRING1 BOOK", 20000, 200);
              em.persist(book1);

              Book book2 = createBook("SPRING2 BOOK", 40000, 300);
              em.persist(book2);

              Delivery delivery = createDelivery(member);
              OrderItem orderItem1 = OrderItem.createOrderItem(book1, 20000, 3);
              OrderItem orderItem2 = OrderItem.createOrderItem(book2, 40000, 4);
              Order order = Order.createOrder(member, delivery, orderItem1,
              orderItem2);
              em.persist(order);

          }

          private Member createMember(String name, String city, String street, String zipcode) {
              Member member = new Member();
              member.setName(name);
              member.setAddress(new Address(city, street, zipcode));
              return member;
          }

          private Book createBook(String name, int price, int stockQuantity) {
              Book book = new Book();
              book.setName(name);
              book.setPrice(price);
              book.setStockQuantity(stockQuantity);
              return book;
          }

          private Delivery createDelivery(Member member) {
              Delivery delivery = new Delivery();
              delivery.setAddress(member.getAddress());
              return delivery;
          }
      }
  }
  ```

## 2. 주문 조회 V1:엔티티를 직접 노출
> 주문 + 배송정보 + 회원을 조회하는 API를 만들자
> 지연 로딩 때문에 발생하는 성능 문제를 단계적으로 해결해보자

### 2-1. OrderSimpleApiController
- api/OrderSimpleApiController.class
  ```java
  /**
  * xToOne
  * Order
  * Order(Many) -> Member(One)
  * Order(One) -> Delivery(One)
  */
  @RestController
  @RequiredArgsConstructor
  public class OrderSimpleApiController {

      private final OrderRepository orderRepository;

      @GetMapping("/api/v1/simple-orders")
      public List<Order> ordersV1() {
          List<Order> all = orderRepository.findAllByCriteria(new OrderSearch());
          for (Order order : all) {
              order.getMember().getName(); // Lazy 강제 초기화
              order.getDelivery().getAddress();
          }
          return all;
          /*
          그냥  return all 을 하면 처음에 Order의 member조회를 위해 Member로 이동
          Member의 orders 조회를 위해 Order로 이동 다시 Order의 member조회를 위해 Member 이동
          무한 루프에 빠져 객체를 계속 불러옴
          해결을 위해 @JsonIgnore을 해줘야한다 다시 Order로 돌아오지않게 하기 위해
          entity/Member/orders 와 entity/OrderItem/order 와 entity/Delivery/order @JsonIgnore처리
          하지만 다른 에러가 발생 -> Order를 불러올 때 member 필드가 fetch = Lazy 되어있다
          Member 객체를 바로 불러오는게 아니라 프록시로 만든 임시 객체를 넣어둔 상태로 불러옴 Member DB 조회 X
          Member 객체의 데이터를 꺼내오는 등 손을 대면 그때서야 sql날려서 값을 가져옴(프록시 초기화)
          Json을 만들때 Member 객체가아닌 다른객체가 있어서 에러발생
          해결 방법 -> Hibernate5Modul을 스프링 빈으로 등록하여 해결 자료 참고 여기선 진행 X
          */
      }
  }
  ```

- 엔티티를 직접 노출하는 것은 좋지 않음
- order member 와 order address 는 지연 로딩 -> 실제 엔티티 대신에 프록시 존재
  - jackson 라이브러리는 기본적으로 이 프록시 객체를 json으로 어떻게 생성해야 하는지 모름 예외 발생
  - Hibernate5Module 을 스프링 빈으로 등록하면 해결

### 2-2. 하이버네이트 모듈 등록
- 스프링 부트 버전에 따라서 모듈 등록 방법이 다름. 스프링 부트 3.0 부터는 javax -> jakarta 로 변경되어 지원 모듈도 다른 모듈을 등록해야 함

- JpashopApplication 에 추가
  ```java
  @Bean
  Hibernate5Module hibernate5Module() {
      return new Hibernate5Module();
  }
  ```
- 기본적으로 초기화 된 프록시 객체만 노출, 초기화 되지 않은 프록시 객체는 노출 안함
> 스프링 부트 3.0 이상: Hibernate5JakartaModule 등록

- build.gradle 에 다음 라이브러리를 추가
  - implementation 'com.fasterxml.jackson.datatype:jackson-datatype-hibernate5-jakarta'

- JpashopApplication 에 다음과 같이 설정하면 강제 지연 로딩 가능
  ```java
  @Bean
  Hibernate5Module hibernate5Module() {
      Hibernate5Module hibernate5Module = new Hibernate5Module();
      //강제 지연 로딩 설정
      hibernate5Module.configure(Hibernate5Module.Feature.FORCE_LAZY_LOADING,true);
      return hibernate5Module;
  }
  ```
  -  옵션을 키면 order -> member , member -> orders 양방향 연관관계를 계속 로딩하게 됨
  -   따라서 @JsonIgnore 옵션을 한곳에 주어야 한다.

- 엔티티를 직접 노출할 때는 양방향 연관관계가 걸린 곳은 꼭! 한곳을 @JsonIgnore 처리 해야 함 안그러면 양쪽을 서로 호출하면서 무한 루프 
- 앞에서 계속 강조했듯이 정말 간단한 애플리케이션이 아니면 엔티티를 API 응답으로 외부로 노출 X 
- 따라서 Hibernate5Module 를 사용하기 보다는 DTO로 변환해서 반환하는 것이 더 좋은 방법

> 주의: 지연 로딩(LAZY)을 피하기 위해 즉시 로딩(EARGR)으로 설정 X 
> 즉시 로딩 때문에 연관관계가 필요 없는 경우에도 데이터를 항상 조회해서 성능 문제가 발생할 수 있음
> 즉시 로딩으로 설정하면 성능 튜닝이 매우 어려움
> 항상 지연 로딩을 기본으로 하고, 성능 최적화가 필요한 경우에는 페치 조인(fetch join)을 사용(V3에서 설명)

## 3. 주문 조회 V2:엔티티를 DTO로 변환

### 3-1. OrderSimpleApiCotroller
- api/OrderSimpleApiController.class
  ```java
  /**
  * xToOne
  * Order
  * Order(Many) -> Member(One)
  * Order(One) -> Delivery(One)
  */
  @RestController
  @RequiredArgsConstructor
  public class OrderSimpleApiController {

      private final OrderRepository orderRepository;

      @GetMapping("/api/v2/simple-orders")
      public List<SimpleOrderDto> ordersV2() {
          List<Order> orders = orderRepository.findAllByCriteria(new OrderSearch());
          /*
          Orders : 2개
          N + 1 문제 -> 1 + 회원 N + 배송 N 쿼리가 날아간다..
          Orders 가 100 개라면  1+ 100 + 100 = 201번의 쿼리 망..
          */
          List<SimpleOrderDto> result = orders.stream()
                  .map(o -> new SimpleOrderDto(o))
                  .collect(Collectors.toList());

          return result;
      }

      @Data
      static class SimpleOrderDto {
          private Long orderId;
          private String name;
          private LocalDateTime orderDate; //주문시간
          private OrderStatus orderStatus;
          private Address address;

          public SimpleOrderDto(Order order) {
              orderId = order.getId();
              name = order.getMember().getName(); //Lazy 초기화 쿼리문 날아감
              orderDate = order.getOrderDate();
              orderStatus = order.getStatus();
              address = order.getDelivery().getAddress(); //Lazy 초기화 쿼리문 날아감
          }
      }
  }
  ```
- 엔티티를 DTO로 변환하는 일반적인 방법이다.

### 3-2. N + 1 문제
- 쿼리가 총 1 + N + N번 실행 (v1과 쿼리수 결과는 같음)
  - order 조회 1번(order 조회 결과 수가 N)
  - order -> member 지연 로딩 조회 N 번
  - order -> delivery 지연 로딩 조회 N 번
  - ex) order의 결과가 4개면 최악의 경우 1 + 4 + 4번 실행(최악의 경우)
- 지연로딩은 영속성 컨텍스트에서 조회하므로, 이미 조회된 경우 쿼리를 생략


## 4. 주문 조회 V3:엔티티를 DTO로 변환 - 페치 조인 최적화

### 4-1. OrderSimpleApiController
- api/OrderSimpleApiController.class
  ```java
  /**
  * xToOne
  * Order
  * Order(Many) -> Member(One)
  * Order(One) -> Delivery(One)
  */
  @RestController
  @RequiredArgsConstructor
  public class OrderSimpleApiController {

      private final OrderRepository orderRepository;

      @Data
      static class SimpleOrderDto {
          private Long orderId;
          private String name;
          private LocalDateTime orderDate; //주문시간
          private OrderStatus orderStatus;
          private Address address;

          public SimpleOrderDto(Order order) {
              orderId = order.getId();
              name = order.getMember().getName(); //Lazy 초기화 쿼리문 날아감
              orderDate = order.getOrderDate();
              orderStatus = order.getStatus();
              address = order.getDelivery().getAddress(); //Lazy 초기화 쿼리문 날아감
          }
      }

      @GetMapping("/api/v3/simple-orders")
      public List<SimpleOrderDto> ordersV3() {
          List<Order> orders = orderRepository.findAllWithMemberDelivery(); //리포지토리에 fetch join 메서드 생성
          List<SimpleOrderDto> result = orders.stream()
                  .map(o -> new SimpleOrderDto(o))
                  .collect(Collectors.toList());

          return result;
      }

  }
  ```

### 4-2. OrderRepository 추가 코드
- repository/OrderRepository.class
  ```java
  @Repository
  @RequiredArgsConstructor
  public class OrderRepository {

      private final EntityManager em;
      
      //...

      public List<Order> findAllWithMemberDelivery() {
          return em.createQuery(
                          "select o from Order o" +
                                  " join fetch o.member m" +
                                  " join fetch o.delivery d", Order.class)
                  .getResultList();
      }
  }
  ```

- 엔티티를 페치 조인(fetch join)을 사용해서 쿼리 1번에 조회
- 페치 조인으로 order -> member , order -> delivery 는 이미 조회 된 상태 이므로 지연로딩X

## 5. 주문 조회 V4:JPA에서 DTO로 바로 조회

### 5-1. OrderSimpleApiController
- api/OrderSimpleApiController.class
```java
/**
* xToOne
* Order
* Order(Many) -> Member(One)
* Order(One) -> Delivery(One)
*/
@RestController
@RequiredArgsConstructor
public class OrderSimpleApiController {

    private final OrderRepository orderRepository;

    @GetMapping("/api/v4/simple-orders")
    public List<OrderSimpleQueryDto> ordersV4() {
        return orderRepository.findOrderDtos();
    }

}
```

### 5-2. OrderSimpleQueryDto
- repository/OrderSimpleQueryDto.class
```java
@Data
public class OrderSimpleQueryDto {
    private Long orderId;
    private String name;
    private LocalDateTime orderDate; //주문시간
    private OrderStatus orderStatus;
    private Address address;

    public OrderSimpleQueryDto(Long orderId, String name, LocalDateTime
            orderDate, OrderStatus orderStatus, Address address) {
        this.orderId = orderId;
        this.name = name;
        this.orderDate = orderDate;
        this.orderStatus = orderStatus;
        this.address = address;
    }
}
```

### 5-3. OrderRepository
- repository/OrderRepository
```java
@Repository
@RequiredArgsConstructor
public class OrderRepository {

    private final EntityManager em;

    //...

    public List<OrderSimpleQueryDto> findOrderDtos() {
        return em.createQuery(
                        "select new jpabook.jpashop.repository.OrderSimpleQueryDto(o.id, m.name, o.orderDate, o.status, d.address)" +
                                // o.id .. 이런식으로까서넣는 이유 엔티티로 주고받으면 오류날수 있음
                                " from Order o" +
                                " join o.member m" +
                                " join o.delivery d", OrderSimpleQueryDto.class)
                .getResultList();
    }
}
```

### 5-4. 정리
- 일반적인 SQL을 사용할 때 처럼 원하는 값을 선택해서 조회
- new 명령어를 사용해서 JPQL의 결과를 DTO로 즉시 변환
- SELECT 절에서 원하는 데이터를 직접 선택하므로 DB 애플리케이션 네트웍 용량 최적화 (생각보다미비)
- 리포지토리 재사용성 떨어짐, API 스펙에 맞춘 코드가 리포지토리에 들어가게 됨
- 엔티티를 DTO로 변환하거나, DTO로 바로 조회하는 두가지 방법은 각각 장단점이 존재
- 엔티티로 조회하면 리포지토리 재사용성도 좋고, 개발도 단순해진다. 

- 쿼리 방식 선택 권장 순서
  1. 우선 엔티티를 DTO로 변환하는 방법을 선택
  2. 필요하면 페치 조인으로 성능을 최적화. 대부분의 성능 이슈가 해결됨
  3. 그래도 안되면 DTO로 직접 조회하는 방법을 사용
  4. 최후의 방법은 JPA가 제공하는 네이티브 SQL이나 스프링 JDBC Template을 사용해서 SQL을 직접 사용
