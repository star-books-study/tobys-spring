# 4장. 예외
- 예외와 관련된 코드는 자주 엉망이 되거나 무성의하게 만들어지기 쉽다.
- 때론 잘못된 예외처리 코드 때문에 찾기 힘든 버그를 낳을 수도 있고, 생각지 않았던 예외상황이 발생했을 때 상상 이상으로 난처해질 수도 있다.

## 4.1.사라진 SQLException
```java
public void deleteAll() throws SQLException {
  this.jdbcContext...
}
public void deleteAll() {
  this.jdbcTemplate...
}
```
- jdbcTemplate을 사용할 시 throws SQLException 가 사라졌다. 어디로 갔을까???

- `SQLException`은 JDBC API 메소드들이 던져주는 것이므로 당연히 있어야 한다.
- 어째서 내부적으로 JDBC API를 쓰는 `jdbcTemplate`가 SQLException을 바깥으로 던지지 않는가?
### 4.1.1 초난감 예외 처리
- 왜 SQLException이 사라졌는지 알아보기 전에 초난감 예외처리들의 대표 선수🤾들을 살펴보자
#### 예외 블랙홀
```java
// 초난감 예외처리 코드
try {
  ...
} catch (Exception e) {}
```
- 예외를 잡고는 아무 것도 하지 않으면 위험하다.
- 위험한 이유 : 프로그램 실행 중에 어디선가 오류가 있어서 예외가 발생했는데 그것을 무시하고 계속 진행해버리기 때문이다.
- 어떤 기능이 비정상적으로 동작하거나, 메모리나 리소스가 소진될 수 있다!

```java
// 초난감 예외처리 코드2
} catch (SQLException e) {
  System.out.println(e);
}
```
```java
// 초난감 예외처리 코드3
} catch (SQLException e) {
  e.printStackTrace();
}
```
- 콘솔 로그를 누군가 계속 모니터링하지 않는 이상 위험하다
- catch 블록을 이용해 화면에 메시지를 출력한 것은 예외를 처리한 게 아니다.

```java
// 그나마 나은 예외처리
} catch (SQLException e) {
  e.printStackTrace();
  System.exit(1);
}
```
- 차라리 시스템을 종료해라
- 굳이 예외를 잡아서 조치를 취할 방법이 없다면 잡지 말아야 한다.
- 메서드에 throws SQLException을 선언해서 예외 처리 책임을 전가해버려라.

#### 무의미, 무책임한 throws
```java
// 초난감 예외처리4
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
- 메소드 선언에 throws Exception을 기계적으로 붙였다
- catch 블록으로 예외를 잡아봐야 해결할 방법도 없고 JDK API나 라이브러리가 던지는 각종 이름도 긴 예외들을 처리하는 코드를 매번 throws로 선언하기도 귀찮아진 개발자의 임시방편
- 메소드 선언에서 의미 있는 정보를 얻을 수 없다. (정말 무엇인가 실행 중에 예외적인 상황이 발생할 수 있다는 것인지, 아니면 그냥 습관적으로 복사해서 붙여놓은 것인지 알 수가 없다.)
- 결국 이런 메소드를 사용하는 메소드 역시 throws Exception을 따라서 붙여야만 한다…!
- 결과적으로 적절한 처리를 통해 복구될 수 있는 예외상황도 제대로 다룰 수 있는 기회를 박탈당한다.
- 예외 처리에 대한 나쁜 습관은 어떤 경우에도 용납하지 않아야 한다.

### 4.1.2 예외의 종류와 특징
- 자바 개발자들 사이에서도 오랫동안 논쟁이 있었던 예외처리. 가장 큰 이슈는 checked exception
- 자바에서 throw를 통해 발생시킬 수 있는 예외는 크게 세 가지가 있다.

#### Error

- java.lang.Error 클래스의 서브 클래스
- 시스템에 비정상적 상황이 발생했을 경우 JVM 단에서 발생한다.
- 어플리케이션에서 대응할 필요 없다!

#### Exception

<img width="383" alt="스크린샷 2024-05-19 오후 10 18 41" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/ee53bfb0-208b-4d71-b4b3-0d30a828ab54">

- java.lang.Exception 클래스와 그 서브 클래스로 정의되는 예외들
- 어플리케이션 코드 작업 중 예외상황이 발생했을 경우 사용
- Exception 클래스는 다시 체크 예외와 언체크 예외로 구분
  - 체크 예외 : Exception 클래스의 서브 클래스이면서 RuntimeException을 상속하지 않은 것들
  - 언체크 예외 : RuntimeException을 상속한 클래스
- 체크 예외를 던지면 catch로 잡던지 메서드 밖으로 던져야 한다.(throws)
#### RuntimeException과 언체크/런타임 예외
- RuntimeException 클래스를 상속한 예외 -> 언체크 예외
- catch로 잡거나 throws로 선언하지 않아도 되고, 선언해줘도 상관없다.
- ex) NullPointerException, IllegalArgumentException

### 4.1.3 예외처리 방법
- 예외를 처리하는 일반적인 방법에 대해 알아보자.
#### 예외 복구
- 예외 상황을 파악하고, 문제를 해결해서 정상 상태로 돌려놓는 것을 말한다.
```java
// 재시도를 통해 예외를 복구하는 코드
int maxRetry = MAX_RETRY;

while(maxRetry --> 0) {
  try {
    ... // 예외가 발생할 수 있는 시도
    return; // 작업 성공
  }
  catch(SomeException e) {
    // 로그 출력, 정해진 시간만큼 대기
  }
  finally {
    // 리소스 반납, 정리 작업
  }
}
throw new RetryFailedException(); // 최대 재시도 횟수를 넘기면 직접 예외 발생
```
- 예를 들어, 사용자가 요청한 파일을 읽으려고 시도했는데, 해당 파일이 없거나 다른 문제가 있어서 읽어지지 않는 경우(IOException) 다른 파일을 이용해보라고 안내하면 된다.
- IOException 에러 메세지가 사용자에게 그냥 던져진다면 예외 복구라고 볼 수 없다.
- 체크 예외들은 예외가 복구될 가능성이 있는 예외이므로, 예외 복구로 처리되기 위해 사용한다.

#### 예외 처리 회피

```java
// 예외처리 회피1
public void add() throws SQLException {
  // JDBC API
}
```

```
public void add throws SQLException {
  try {
    // JDBC API
  }
  catch(SQLException e) {
    // 로그 출력
    throw e;
  }
}
```
- 예외 처리를 자신이 담당하지 않고, 자신을 호출한 쪽으로 예외를 던져버리는 방식
- 콜백과 템플릿의 관계와 같은 명확한 역할 분담이 없는 상황에서 무작정 예외를 던지면 무책임한 책임회피일 수 있다.
만약 DAO가 SQLException을 던지면, 이 예외는 처리할 곳이 없어서 서비스 레이어로 갔다가 컨트롤러로 가고 결국 그냥 서버로 갈 것이다.
- 예외를 회피하는 것은 예외를 복구하는 것처럼 의도가 분명해야 한다.

- 콜백/템플릿처럼 긴밀한 다른 오브젝트에게 예외처리 책임을 지게 하거나
- 자신을 사용하는 쪽에서 예외를 다루는 게 최선이라는 확신이 있어야 한다.
#### 예외 전환
- 예외를 메소드 밖으로 던지지만, 예외 회피와 달리 발생한 예외를 적절한 예외로 전환해서 던진다는 차이가 있다.
```java
// 예외 전환 기능을 가진 DAO 메서드
public void add(User user) throws DuplicateUserIdException, SQLException {
  try {
    // JDBC를 이용해 user 정보를 DB에 추가하는 코드 또는
    // 그런 기능을 가진 다른 SQLException을 던지는 메소드를 호출하는 코드
  }
  catch(SQLException e) {
    // ErrorCode가 MySQL의 "Duplicate Entry(1062)"이면 예외 전환
    if (e.getERrorCode() == MysqlErrorNumbers.ER_DUP_ENTRY)
      throw DuplicateUserException();
    else
      throw e; // 그 외의 경우는 SQLException 그대로
  }
}
```
- 예외 자체가 예외상황에 대해 설명해주지 못할 때, 의미를 분명히 할 예외로 전환해 던질 수 있다.
- 예외 전환을 이용하여 SQLException을 DuplicateUserIdException과 같은 예외로 변경하여 던져준다면?
- 예외가 일어난 이유가 한 층 명확해지고 사용자는 다른 아이디를 사용하는 것으로 적절한 복구 작업을 수행할 수 있다.


- 예외 전환 방법은 두 가지가 있다.
  1. 중첩 예외
  2. 포장 예외
 
- 보통 전환하는 예외에 원래 발생한 예외를 담아서 **중첩 예외**로 만드는 것이 좋다.
- 중첩 예외는 getCause() 메서드를 사용해서 처음 발생한 예외가 무엇인지 확인할 수 있다.
- 생성자나 initCuase() 메서드로 근본 원인이 되는 예외를 넣어주면 된다.
```java
// 중첩 예외1
catch(SQLException e) {
  ...
  throw DuplicateUserIdException(e);
}
```

```java
// 중첩 예외2
catch(SQLException e) {
  ...
  throw DuplicateUserIdException(e).initCause(e); 
}
```

- 포장 예외는 주로 예외 처리르 강제하는 체크 예외를 런차임 예외로 바꾸는 경우에 사용한다.
- 대표적으로 EJBException을 들 수 있다.

```java
// 예외 포장
try {
  ...
} catch (NamingException ne) {
  throw new EJBException(ne);
} catch (SQLException se) {
  throw new EJBException(se);
} catch (RemoteException re) {
  throw new EJBException(re);
}
```
- 이렇게 런타임 예외로 만들어서 전달하면 EJB는 이를 시스템 익셉션으로 인식하고 트랜잭션을 자동으로 롤백
- 비즈니스적으로 의미도 없고, 복구 가능하지도 않은 예외에 대해서는 런타임 예외(언체크드 예외)로 포장해서 던지는 편이 낫다.
- 반대로 어플리케이션 로직으로 인한 예외는 체크 예외를 사용하는게 적절하다. 이에 대한 적절한 대응이나 복구전략이 필요하기 때문이다!
- 어차피 복구가 불가능한 예외라면 가능한 한 빨리 런타임 예외로 포장해 던지게 해서 다른 계층의 메소드를 작성할 때 불필요한 throws 선언이 들어가지 않도록 해줘야 한다.

### 4.1.4 예외 처리 전략
#### 런타임 예외의 보편화

- 자바 환경이 서버 환경으로 넘어가면서, 체크 예외의 활용도와 가치는 떨어지고 있다.

- 체크 예외는 복구할 가능성이 조금이라도 있는, 말 그대로 예외적인 상황이기 때문에 자바는 이를 처리하는 catch 블록이나 throws 선언을 강제하고 있다.

- 예외가 발생할 가능성이 있는 API 메소드를 사용하는 개발자의 실수를 방지하기 위한 배려이다.
- 하지만 실제로는 예외를 제대로 다루고 싶지 않을 만큼 짜증나게 만드는 원인이 되기도 한다.
- 서버 환경은 일반적인 애플리케이션 환경과 다르다.

> 자바 환경이 서버 환경으로 넘어가면서, 체크 예외의 활용도와 가치는 떨어지고 있다.

- 체크 예외는 복구할 가능성이 조금이라도 있는, 말 그대로 예외적인 상황이기 때문에 자바는 이를 처리하는 catch 블록이나 throws 선언을 강제하고 있다.

- 예외가 발생할 가능성이 있는 API 메소드를 사용하는 개발자의 실수를 방지하기 위한 배려이다.
- 하지만 실제로는 예외를 제대로 다루고 싶지 않을 만큼 짜증나게 만드는 원인이 되기도 한다.
- 서버 환경은 일반적인 애플리케이션 환경과 다르다.

#### 어플리케이션 환경

- 자바 초기 AWT, 스윙 에서는 파일을 처리한다고 했을 때, 해당 파일이 처리 불가능하면 복구가 필요했다.
- 워드에서 특정이름의 파일을 검색할 수 없다고 프로그램이 종료되어선 안된다!
#### 서버 환경

- 한 번에 다수의 사용자가 접근하여 해당 서비스를 이용하기 때문에 작업을 중지하고 예외 상황을 복구할 수 없다!
- 어플리케이션 차원에서 예외상황을 파악하고 요청에 대한 작업을 취소하는 편이 좋다.

#### 서버 환경

- 한 번에 다수의 사용자가 접근하여 해당 서비스를 이용하기 때문에 작업을 중지하고 예외 상황을 복구할 수 없다!
- 어플리케이션 차원에서 예외상황을 파악하고 요청에 대한 작업을 취소하는 편이 좋다.

### add() 메소드의 예외처리

```java
public class DuplicateUserIdException extends RuntimeException{
    public DuplicateUserIdException(Throwable cause) {
        super(cause);
    }
}
```
```java
public void add() throws DuplicateUserIdException {
  try {
    // 
  }
  catch (SQLException e) {
    if (e.getErrorCode() == MysqlErrorNumbers.ER_DUP_ENTRY)
      throw new DuplicateUserIdException(e); // 예외 전환
    else
      throw new RuntimeException(e); // 예외 포장
  }
}
```
- DuplicatedUserIdException도 굳이 체크 예외로 둬야 하는 것은 아니다.

- DuplicatedUserIdException처럼 의미 있는 예외는 add() 메소드를 바로 호출한 오브젝트 대신 더 앞단의 오브젝트에서 다룰 수도 있다.
- 어디에서든 DuplicatedUserIdException을 잡아서 처리할 수 있다면 굳이 체크 예외로 만들지 않고 런타임 예외로 만드는 게 낫다.
- 대신 add() 메소드가 DuplicatedUserIdException 을 던진다고 명시적으로 선언해야 한다. 런타임 예외도 throws로 선언 가능하다!

### 런타임 예외를 일반화 시 단점

- 컴파일러가 예외처리를 강제하지 않으므로 신경쓰지 않으면 예외 상황을 충분히 고려하지 않을 수 있다.
- 런타임 예외를 사용하는 경우엔 API 문서, 레퍼런스 문서 등을 통해 메소드를 사용할 때 발생할 수 있는 예외의 종류와 원인, 활용 방법을 자세히 설명해두자.

### 어플리케이션 예외

- 런타임 예외 중심 전략은 굳이 이름을 붙이자면 낙관적 예외처리 기법이다.

- 복구할 수 있는 예외는 없다고 가정한다.
- 예외가 생겨도 어차피 런타임 예외 이므로 시스템에서 알아서 처리해줄 것이고, 꼭 필요한 경우는 런타임 예외라도 잡아서 복구하거나 대응할 수 있으니 문제될 것이 없다는 낙관적 태도를 기반으로 한다.
- 혹시 놓치는 예외가 있을까 처리를 강제하는 체크 예외의 비관적인 접근 방법과 대비된다.
- 반면 어플리케이션 예외도 있다.

- 시스템 혹은 외부 상황이 원인이 아닌 애플리케이션 자체의 로직에 의한 예외이다.
- 반드시 catch 해서 무엇인가 조치를 취하도록 요구한다.

```java
try {
  BigDecimal balance = account.withdraw(amount);
  ...
  // 정상적인 처리 결과를 출력하도록 진행
}
catch(InsufficientBalanceException e) { // 체크 예외
  // InsufficientBalanceException에 담긴 인출 가능한 잔고 금액 정보를 가져옴
  BigDecimal availFunds = e.getAvailFunds();
  ...
  // 잔고 부족 안내 메세지를 준비하고 이를 출력하도록 진행
}
```
## 1-5. SQLException 은 어떻게 됐나?

- 앞서 체크 예외와 언체크 예외를 배워보았다.

- 코드 레벨에서 복구 방법이 없는 경우 예외 전환을 통해 언체크 예외를 던져버리는 편이 낫다는 것을 배웠다.
- 런타임 예외의 보편화와 함께 만일 비즈니스적으로 더 명확한 의미를 줄 수 있는 경우에는 의미를 분명하게 전달할 수 있는 예외를 만들고 중첩 예외로 던져버리는 편이 낫다는 결론을 얻었다.
- 복구 불가능한 예외를 괜히 체크 예외로 만들면 나쁜 예외처리 습관을 가진 개발자에 의해 더 최악의 시나리오가 발생할 수도 있다.

### SQLException은 복구 불가능

- 일반적으로 해당 예외가 발생하는 이유는 SQL 문법이 틀렸거나, 제약조건을 위반했거나, DB 서버가 다운됐거나, 네트워크가 불안정하거나, DB 커넥션 풀이 꽉 찬 경우 등이다.
- 따라서 언체크/런타임 에러로 전환해야 한다!
### 스프링 런타임 예외 보편화 전략

- 스프링 API 메소드에 정의되어 있는 대부분의 예외는 런타임 예외이다.

- SQLException이 사라진 이유는 스프링의 JdbcTemplate은 런타임 예외의 보편화 전략을 따르고 있기 때문이다.

- JdbcTemplate 템플릿과 콜백 안에서 발생하는 모든 SQLException을 런타임 예외인 DataAccessException으로 포장해서 던져준다.

- JdbcTemplate의 update(), queryForInt(), query() 메소드 선언을 잘 살펴보면 모두 throws DataAccessException이라고 되어 있음을 발견할 수 있다.

```java
public int update(final String sql) throws DataAccessException {
    //...
}
```
- throws로 선언되어 있긴 하지만 DataAccessException이 런타임 예외이므로 update()를 사용하는 메소드에서 이를 잡거나 다시 던질 이유는 없다.

