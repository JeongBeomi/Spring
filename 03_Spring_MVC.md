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
  