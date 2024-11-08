# 부록 A. 스프링 모듈

## A.1 스프링 모듈의 종류와 특ㅈ

### A.1.1 스프링 모듈 이름
- 스프링 모듈은 jar 확장자를 가진 파일이다.
- 모듈 이름은 다음과 같이 구성된다.
  <img width="339" alt="스크린샷 2024-11-05 오후 11 26 06" src="https://github.com/user-attachments/assets/5f9f8b54-d91d-46e0-a556-1b4cfa2425eb">
- 스프링 모듈은 OSGi 모듈 조건을 충족하는 OSGi 번들이기도 하다. 따라서 OSGi 플랫폼에 바로 가져다 사용할 수 있다.
- Maven의 명명 규칙을 따라 만들어진 스프링 모듈 파일도 있다.
  <img width="316" alt="스크린샷 2024-11-06 오전 10 02 32" src="https://github.com/user-attachments/assets/8afac767-9d87-4707-8c15-153a22656562">

### A.1.2 스프링 모듈 추가
- 스프링 모듈을 프로젝트에 추가하는 방법 2가지
  - 배포판에 포함된 모듈을 직접 추가하는 방법
  - Maven이나 Ivy의 의존 라이브러리 관리 기능을 이용해 모듈 리포지토리에서 자동으로 추가되게 하는 방법

#### 수동 추가
- 스프링 모듈을 얻을 수 있는 가장 손쉬운 방법은 스프링 배포판을 이용하는 것
- 배포판 압축 풀고 dist 폴더 열기

#### Maven / Ivy 자동 추가
- Maven 사용 시 pom.xml 파일의 의존 정보 설정만으로 필요한 모듈을 가져올 수 있음
- Ivy도 Maven과 동일한 리포지토리 사용

#### A.1.3 스프링 모듈 목록
- 스프링 모듈의 각 목록과 각 모듈의 OSGi 이름과 Maven 이름
  <img width="535" alt="스크린샷 2024-11-06 오전 10 10 02" src="https://github.com/user-attachments/assets/8e60a939-7449-4cf7-9382-741c00bf471e">
  <img width="535" alt="스크린샷 2024-11-06 오전 10 10 19" src="https://github.com/user-attachments/assets/f10d01fa-5b86-4045-9a78-a4a58b4ce24a">

## A.1 스프링 모듈의 의존관계
<img width="564" alt="스크린샷 2024-11-06 오전 10 10 50" src="https://github.com/user-attachments/assets/2475d650-ae69-44e0-8797-543390203430">

## A.1.1 스프링 모듈 이름
