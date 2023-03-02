## 0. 목차

## 1. 웹 서버 & 웹 애플리케이션 서버
### 1-1. 웹 서버(Web Server)
- HTTP 기반으로 동작
- 정적 리소스 제공, 기타 부가기능
- 정적(파일) HTML, CSS, JS, 이미지, 영상
- 예) NGINX, APACHE
![image](https://user-images.githubusercontent.com/109258397/220804527-b56b5e07-56e5-43da-8026-aae599422c97.png)

### 1-2. 웹 애플리케이션 서버(WAS-Web Application Server)
- HTTP기반으로 동작
- 웹 서버 기능 포함 + (정적 리소스 제공 가능)
- 프로그램 코드를 실행해서 애플리케이션 로직 수행
  - 동적 HTML, HTTP API(JSON)
  - 서블릿, JSP, 스프링 MVC
- 예) 톰캣, Jetty, Undertow
![image](https://user-images.githubusercontent.com/109258397/220804930-fdfa1578-9066-4337-88cf-3dbd2c235b4d.png)

### 1-3. 차이
- 웹 서버는 정적 리소스(파일), WAS는 애플리케이션 로직
- 사실은 둘의 용어도 경계도 모호함
  - 웹 서버도 프로그램을 실행하는 기능을 포함하기도 함
  - 웹 애플리케이션 서버도 웹서버의 기능을 제공함
- 자바는 서블릿 컨테이너 기능을 제공하면 WAS(서블릿 없이 자바코드를 실행하는 서버 프레임워크도 있음)
- WAS는 애플리케이션 코드를 실행하는데 더 특화

### 1-4. 웹 시스템 구성
-  WAS, DB 만으로 시스템 구성 가능
  -  WAS는 정적 리소스, 애플리케이션 로직 모두 제공 가능
  -  WAS가 너무 많은 역할을 담당, 서버 과부하 우려
  -  가장 비싼 애플리케이션 로직이 정적 리소스 때문에 수행이 어려울 수 있음
  -  WAS장애시 오류화면도 노출 불가능
    ![image](https://user-images.githubusercontent.com/109258397/222216708-f448ef1d-3609-4edb-8d64-d778f026ca25.png)

- WEB, WAS, DB 구성
  - 정적 리소스는 웹 서버가 처리
  - 웹 서버는 애플리케이션 로직같은 동적인 처리가 필요하면 WAS에 요청을 위임
  - WAS 는 중요한 애플리케이션 로직 처리 전담
  ![image](https://user-images.githubusercontent.com/109258397/222217466-1ddfc14f-6481-4985-9ddf-8b3bfe5e5a95.png)
  - 효율적인 리소스 관리
    - 정적 리소스가 많이 사용 -> Web 서버 증설 / 애플리케이션 리소스 -> WAS 증설
  - 정적 리소스만 제공하는 웹서는 잘 죽지 않음, 애플리케이션 로직이 동작하는 WAS서버는 잘죽음
  - WAS, DB 장애시 WEB 서버가 오류 화면 제공 가능

## 2. 서블릿

### 2-1. HTML Form 데이터 전송
- POST전송 - 저장
  ![image](https://user-images.githubusercontent.com/109258397/222297876-c9287bfc-e096-464a-a3e7-87f2172cd2a9.png)
- 의미있는 비즈니스 로직 전에 과정이 너무 중복, 길다 ->  서블릿의 등장
  ![image](https://user-images.githubusercontent.com/109258397/222298308-d5f6b611-0d15-4177-a1a7-34524d516920.png)
- 서블릿 특징
  ![image](https://user-images.githubusercontent.com/109258397/222298559-f7dcc2b8-d00b-47f4-b5e2-420ecf008cf6.png)
- HTTP 요청, 응답 흐름
  - HTTP 요청시
    - WAS는 Request, Response 객체를 새로 만들어 서블릿 객체 호출
    - 개발자는 Request 객체에서 HTTP 요청 정보를 편리하게 꺼내서 사용
    - 개발자는 Response 객체에 HTTP 응답 정보를 편리하게 입력
    - WAS는 Response 객체에 담겨있는 내용으로 HTTP 응답 정보를 생성
  ![image](https://user-images.githubusercontent.com/109258397/222299466-45a87fd9-1132-4248-b9d9-a6d58d5e96cd.png)

### 2-2. 서블릿 컨테이너
- 톰캣처럼 서블릿을 지원하는 WAS를 서블릿 컨테이너라고함
- 서블릿 컨테이너는 서블릿 객체를 생성, 초기화, 호출, 종료 생명주기 관리
- 서블릿 객체는 싱글톤으로 관리(하나의 객체만 만들어 재사용)
  - 고객의 요청시 계속 객체 생성은 비효율
  - 최초 로딩시 서블릿 객체를 미리 만들어두고 재활용
  - **공유 변수 사용 주의**
  - 서블릿 컨테이너 종료시 함께 종료
- JSP도 서블릿으로 변환되어 사용
- 동시 요청을 위한 멀티 쓰레드 처리 지원
