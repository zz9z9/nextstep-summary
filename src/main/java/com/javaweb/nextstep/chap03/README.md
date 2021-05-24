## Chapter3 : 개발 환경 구축 및 웹 서버 실습 요구사항

### 어려웠던 부분
1. p.83 요구사항에 '로컬 개발 환경에 설치한 HTTP 웹 서버를 물리적으로 떨어져 있는 원격 서버에 배포해 정상적으로 동작하는지 테스트' 하라는 요구사항이 있었다.

> 방법 1 : EC2 서버에서 직접 repo clone 한 후 메이븐 통해 jar 파일 생성하여 실행 
- 놀고 있는 pc가 있는 것도 아니고 AWS도 사용해보고 싶어서 원격 서버로 AWS EC2를 선택했다. 
- 로컬에 있는 것을 원격 서버로 배포하는 건 도커를 공부해야 할 것 같아서 일단 EC2 서버에서 웹 서버 소스코드가 있는 repo를 git clone 한 후에<br>
 서버에서 메이븐을 통해 jar 파일을 만들어서 실행하는 방식으로 접근해봤다.
- mvn compile, mvn package를 통해 jar 파일을 생성한 뒤 java -jar '파일명' 으로 실행했더니 no main manifest attribute, in web-application-server.jar 라는 에러가 발생했다.
- 위 에러를 해결했더니 import하는 외부 라이브러리에 대해 java.lang.NoClassDefFoundError가 발생했다. 
- 최종적으로 아래 부분을 pom.xml에 추가하여 메인 클래스와 의존하는 외부 라이브러리의 경로를 명시해줌으로써 해결할 수 있었다. 
- pom.xml에 추가한 부분 <br>
![image](https://user-images.githubusercontent.com/64415489/119369148-d06d6280-bcee-11eb-91fc-ed09d68870a6.png)
- target 디렉토리 내부 
![image](https://user-images.githubusercontent.com/64415489/119370039-cac44c80-bcef-11eb-945d-6c7e99343d5d.png)
- 브라우저에서 EC2 8080 포트 접속 결과
 ![image](https://user-images.githubusercontent.com/64415489/119369800-89cc3800-bcef-11eb-9bff-2bf68efff2f8.png)
 
 > 방법 2 : 방법1 에서 pom.xml 변경하지 않기 (책에 나온 방법)
- 위 방법처럼 pom.xml 변경시키지 않고 기존 것 그대로 사용할 수 있다.
- mvn package 까지는 같으나 jar 파일을 실행시키는게 아니라 다음과 같은 명령어 사용 (-cp : classpath)
- java -cp target/classes:target/dependency/* webserver.WebServer 8080 & (bash 기준)
- java -cp target/classes:target/dependency/\* webserver.WebServer 8080 & (zsh 기준)
 
 > 방법 3 : 쉘 스크립트 작성 (추후 추가 예정)

 > 방법 4 : 도커 사용 (추후 추가 예정)

2. p.83 요구사항에 'HTTP 웹 서버 배포 작업은 root 계정이 아닌 배포를 담당할 새로운 계정을 만들어 진행한다'


### 배운 것 
1. AWS EC2 인스턴스 생성하기 
2. SSH 활용해서 리눅스 서버 접속하기 
2. 원격 서버에 배포하기 
3. 리눅스 APT란 ? 
4. 기본적인 리눅스 명령어 

### 인상 깊었던 말