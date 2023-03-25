## 0. 목차
- 회원 등록 API
- 회원 수정 API
- 회원 조회 API


### 주의! - 스프링 부트 3.0
- 스프링 부트 3.0을 선택하게 되면 다음 부분을 꼭 확인해주세요.
1. Java 17 이상을 사용
2. javax 패키지 이름을 jakarta로 변경
오라클과 자바 라이센스 문제로 모든 javax 패키지를 jakarta로 변경
3. H2 데이터베이스를 2.1.214 버전 이상 사용해주세요.
- 패키지 이름 변경 예)
  - JPA 애노테이션
    - javax.persistence.Entity jakarta.persistence.Entity
  - 스프링에서 자주 사용하는 @PostConstruct 애노테이션
    - javax.annotation.PostConstruct jakarta.annotation.PostConstruct
  - 스프링에서 자주 사용하는 검증 애노테이션
    - javax.validation jakarta.validation
스프링 부트 3.0 관련 자세한 내용 : https://bit.ly/springboot3

## 1. 회원 등록 API

### 1-1. V1 엔티티를 Request Body에 직접 매핑
- api/MemberApiController
  ```java
  @RestController // @Controller @ResponseBody 합친것
  @RequiredArgsConstructor
  public class MemberApiController {

      private final MemberService memberService;

      @PostMapping("/api/v1/members")
      public CreateMemberResponse setMemberV1(@RequestBody @Valid Member member) {
          // @RequestBody Member member Json으로 온 바디를 맵핑해서 Member에 다 넣어줌
          /*
          @Valid가 달려 있으므로 MemberEntity에서 @NotEmpty를 달아주면 유효성검사를 해줌
          하지만 NotEmpty는 엔티티단계에서 필요한 검증 단계가 아님, presentation단계 controller단에서
          처리하는게 맞는 거다.. 다른 api에서는 필요없을 수도 있으니까
          또한  Member member를 쓰면 엔티티가 api랑 1대1 맵핑이되어버림 entity는 여러곳에서
          쓰기 때문에 변경의 가능성이 있다. -> DTO의 필요성 대두
          qpi개발에는 entity를 파라미터로 그냥 받지말자
          */
          Long id = memberService.join(member);
          return new CreateMemberResponse(id);
      }

      @Data
      static class CreateMemberResponse {
          private Long id;

          public CreateMemberResponse(Long id) {
              this.id = id;
          }
      }
  }
  ```

- domain/Member
  ```java
  @Entity
  @Getter @Setter
  public class Member {

      @Id @GeneratedValue // @Id 현재 엔티티의 PK값으로 선언 @Gener.. 기본키 할당 방법 선택
      @Column(name = "member_id") // 없으면 이름이 그냥 id가 되어버림
      private Long id;

      @NotEmpty // Controller/CreateMemberResponseV1에서 유효성 검사를 위해 달아준다
      private String name;

      @Embedded // 내장타입을 포함했다는 어노테이션으로 맵핑
      private Address address; // Alt + Enter -> create class

      @OneToMany(mappedBy = "member") // Order와 1:N 관계 Member가 1이니까 OneToMany
      // (mappedBy = "member") orders테이블에 있는 member필드에 의해 맵핑 된거야라고 표시
      private List<Order> orders = new ArrayList<>(); // Alt + Enter -> create class
  }
  ```

- 문제점
  - 엔티티에 프레젠테이션 계층을 위한 로직이 추가됨
  - 엔티티에 API 검증을 위한 로직이 들어감 (@NotEmpty 등등)
  - 실무에서는 회원 엔티티를 위한 API가 다양하게 만들어지는데, 한 엔티티에 각각의 API를 위한 모든 요청 요구사항을 담기는 어려움
  - 엔티티가 변경되면 API 스펙이 변한다.

- 결론
  - API요청 스펙에 맞추어 별도의 DTO를 파라미터로 받아야함

### 1-2. V2 엔티티 대신에 DTO를 RequestBody에 매핑
- api/CreateApiController
  ```java
  @RestController // @Controller @ResponseBody 합친것
  @RequiredArgsConstructor
  public class MemberApiController {

      private final MemberService memberService;

      @PostMapping("/api/v2/members")
      public CreateMemberResponse setMemberV2(@RequestBody @Valid CreateMemberRequest request) {

          Member member = new Member();
          member.setName(request.getName());

          Long id = memberService.join(member);
          return new CreateMemberResponse(id);
      }

      @Data
      static class CreateMemberRequest {
          @NotEmpty //entity대신 여기에서 적용
          private String name;
      }

      @Data
      static class CreateMemberResponse {
          private Long id;

          public CreateMemberResponse(Long id) {
              this.id = id;
          }
      }
  }
  ```

- domain/Member
  ```java
  @Entity
  @Getter @Setter
  public class Member {

      @Id @GeneratedValue // @Id 현재 엔티티의 PK값으로 선언 @Gener.. 기본키 할당 방법 선택
      @Column(name = "member_id") // 없으면 이름이 그냥 id가 되어버림
      private Long id;

      //@NotEmpty // Controller/CreateMemberResponseV1에서 유효성 검사를 위해 달아준다
      //MemberApiController/CreateMemberResponseV2를 적용하면 여기에 @NotEmpty 안붙여도된다
      private String name;

      @Embedded // 내장타입을 포함했다는 어노테이션으로 맵핑
      private Address address; // Alt + Enter -> create class

      @OneToMany(mappedBy = "member") // Order와 1:N 관계 Member가 1이니까 OneToMany
      // (mappedBy = "member") orders테이블에 있는 member필드에 의해 맵핑 된거야라고 표시
      private List<Order> orders = new ArrayList<>(); // Alt + Enter -> create class
  }
  ```

- CreateMemberRequest 를 Member 엔티티 대신에 RequestBody와 매핑
  - 엔티티와 프레젠테이션 계층을 위한 로직을 분리
  - 엔티티와 API 스펙을 명확하게 분리
  - 엔티티가 변해도 API 스펙이 변하지 않음
  > 참고: 실무에서는 엔티티를 API 스펙에 노출하면 안됨!

## 2. 회원 수정 API

### 2-1. 회원 수정 코드
- api/MemberApiController
  ```java
  @RestController // @Controller @ResponseBody 합친것
  @RequiredArgsConstructor
  public class MemberApiController {

      private final MemberService memberService;

      //...

      @PutMapping("/api/v2/members/{id}")
      public UpdateMemberResponse updateMemberV2(
              @PathVariable("id") Long id,
              @RequestBody @Valid UpdateMemberRequest request) {

          //수정할 때는 변경감지 사용
          memberService.update(id, request.getName());
          Member findMember = memberService.findOne(id);
          return new UpdateMemberResponse(findMember.getId(), findMember.getName()); // 전체생성자 -> 파라미터 다필요
      }
      
      @Data
      static class UpdateMemberRequest {
          private String name;
      }

      @Data
      @AllArgsConstructor //전체 생성자를 만들어준다 -> 파라미터 다필요
      static class UpdateMemberResponse {
          private Long id;
          private String name;
      }

      //...
  }
  ```
- service/MemberService
  ```java
  @Service // 컨포넌트 스캔에 의해 자동으로 스프링 빈으로 관리됨
  //JPA의 어떤 모든 데이터 변경이나 로직들은 가급적이면 트랜잭션안에서 처리되야한다.
  @Transactional(readOnly = true) // 쓸수 있는 옵션이 많기때문에 spring의 어노테이션 사용 지향
  @RequiredArgsConstructor
  public class MemberService {

      //...

      @Transactional
      public void update(Long id, String name) {
          Member member = memberRepository.findOne(id);
          member.setName(name);
      }
  }
  ```
- 회원 수정 API updateMemberV2 은 회원 정보를 부분 업데이트
- PUT은 전체 업데이트를 할 때 사용하는 것이 맞음
- 부분 업데이트를 하려면 PATCH를 사용 or POST를 사용하는 것이 REST 스타일

## 3. 회원 조회 API
### 3-1. V1 회원조회 : 응답 값으로 엔티티를 직접 외부에 노출
- api/MemberApiController
  ```java
  @RestController // @Controller @ResponseBody 합친것
  @RequiredArgsConstructor
  public class MemberApiController {

      private final MemberService memberService;

      //...

      @GetMapping("/api/v1/members")
      public List<Member> membersV1() {
          return memberService.findMembers();
          /*
          entity 자체를 반환하면 orders 필드도 출력됨 (불필요한 필드 유출)
          -> entity/Member orders필드에 @JsonIgnore를 달아주면 막을 수 있낀함
          하지만 다른 api에서 해당 필드를 원하면? 문제발생
          또한 반환 값이 array(list) -> Json스펙이 깨진다. 추가 데이터에 대한 확장 힘들다
          */
      }

      // ...
  }
  ```
- domain/Member
  ```java
  @Entity
  @Getter @Setter
  public class Member {

      //...
      
      @JsonIgnore
      /*
      엔티티 반환할때 안보여줄 수 있다
      하지만 다른 api에서 해당 필드를 원하면? 문제발생
      */
      @OneToMany(mappedBy = "member") // Order와 1:N 관계 Member가 1이니까 OneToMany
      // (mappedBy = "member") orders테이블에 있는 member필드에 의해 맵핑 된거야라고 표시
      private List<Order> orders = new ArrayList<>(); // Alt + Enter -> create class
  }
  ```
 - 문제점
  - 엔티티에 프레젠테이션 계층을 위한 로직이 추가됨
  - 기본적으로 엔티티의 모든 값이 노출
  - 응답 스펙을 맞추기 위해 로직이 추가됨 (@JsonIgnore, 별도의 뷰 로직 등등)
  - 실무에서는 같은 엔티티에 대해 API가 용도에 따라 다양하게 만들어짐, 한 엔티티에 각각의 API를 위한 프레젠테이션 응답 로직을 담기는 어려움
  - 엔티티가 변경되면 API 스펙이 변함
  - 추가로 컬렉션을 직접 반환하면 항후 API 스펙을 변경하기 어려움 (별도의 Result 클래스 생성으로 해결)
  - 또한 반환 값이 array(list) -> Json스펙이 깨진다. 추가 데이터에 대한 확장 힘들다

 - 결론
  - API 응답 스펙에 맞추어 별도의 DTO를 반환해야함
  
- 엔티티를 외부에 노출하지 마세요!
  - 실무에서는 member 엔티티의 데이터가 필요한 API가 계속 증가하게 됨
  - 어떤 API는 name 필드가 필요하지만, 어떤 API는 name 필드가 필요없을 수 있음
  - 결론적으로 엔티티 대신에 API 스펙에 맞는 별도의 DTO를 노출해야 함

### 3-2. V2 회원 조회 : 응답 값으로 엔티티가 아닌 별도의 DTO 사용
- api/MemberApiController
  ```java
  @RestController // @Controller @ResponseBody 합친것
  @RequiredArgsConstructor
  public class MemberApiController {

      private final MemberService memberService;

      //...

      @GetMapping("/api/v2/members")
      public Result memberV2() {
          List<Member> findMembers = memberService.findMembers();
          List<MemberDto> collect = findMembers.stream()
                  .map(m -> new MemberDto(m.getName()))
                  .collect(Collectors.toList());

          return new Result(collect.size(), collect);
      }

      @Data
      @AllArgsConstructor
      static class Result<T> {
          // array로 반환하는 것을 막기 위해 가공해준다
          private int count; //필드 추가하기도 간편하다
          private T data;
      }

      @Data
      @AllArgsConstructor
      static class MemberDto {
          private String name;
      }

      //...
  }
  ```
- 추가로 Result 클래스로 컬렉션을 감싸서 향후 필요한 필드를 추가할 수 있음