# 4장. 예외
## 4.1. 사라진 SQL EXCEPTION
### 4.1.1. 초난감 예외처리
#### 예외 블랙홀
```java
try {
    ...
} catch(SQLException e) {

}
```
- 예외를 잡고 아무것도 하지 않는 코드로, 굉장히 위험하다.
```java
} catch (Exception e) {
  System.out.println(e);
}

} catch (Exception e) {
  e.printStackTrace();
}
```
- 예외를 단순히 출력만 하는 것도 안된다. 
- 모든 예외는 적절히 `복구` 되든지, 작업을 `중단` 시키고 운영자/개발자에게 분명히 통보되어야 한다
- 굳이 예외를 잡아서 뭔가 조치를 취할 방법이 없다면 잡지 말아야 한다.
- 메소드에 throws SQLException 을 선언해서 메소드 밖으로 던지고, 자신을 호출한 코드에 **예외처리 책임을 전가해버리자**
  
#### 무의미하고 무책임한 throws
```java
public void method1() throws Exception {
  method2();
  ...
}

public void method2() throws Exception {
  method3();
  ...
}

public void method3() throws Exception ...
```
- 예외블랙홀 보단 낫지만 메소드 선언에 throws Exception을 기계적으로 붙이는 게 되면, 결과적으로 **적절한 처리를 통해 복구될 수 있는 예외상황도 제대로 다룰 수 있는 기회를 박탈당한다.**

### 4.1.2. 예외의 종류와 특징
- Error
  - 시스템에 비정상적인 상황이 발생했을 경우
  - 주로 자바 VM 에서 발생시키는 것이고, 애플리케이션 코드에서 잡으려고 하면 안된다.
  - OutOfMemoryError 나 ThreadDeath 같은 에러는 catch 로 잡아봤자 대응 방법이 없다.
- Exception 과 체크 예외
  - 애플리케이션 코드 작업 중에 예외상황 발생했을 경우
  - `체크 예외` : Exception 클래스의 서브 클래스, **RuntimeException 클래스 상속하지 않은 것**
  - `언체크 예외` : Exception 클래스의 서브 클래스, **RuntimeException 클래스 상속한 것**
  - 일반적으로 예외라 함은, 체크 예외라고 생각해도 된다
  - 체크 예외가 발생할 수 있는 메소드 사용 시 **반드시 예외를 처리하는 코드를 함께 작성**해야 한다
    - 그렇지 않으면 **컴파일 에러** 가 발생한다
