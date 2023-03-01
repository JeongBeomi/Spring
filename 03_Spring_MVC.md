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
  - 정적 리소스만 제공하는 웹서는 잘 죽지 않음, WAS