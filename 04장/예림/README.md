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

## 4-2.예외 전환
### 4-2-1. JDBC의 한계
- JDBC는 Connection, Statement, ResultSet 등의 표준 인터페이스로 기능을 제공한다. 따라서 개발자들은 DB 종류와 상관없이 일관된 방법으로 프로그래밍이 가능하다. 객체지향적 프로그래밍이 가능하다.

- 하지만, DB 종류에 따라 데이터 액세스 코드가 달라질 수 있다.

#### 비표준 SQL

- DB마다 SQL 비표준 문법이 제공된다. 페이지네이션이나 쿼리 조건 관련 추가적인 문법이 있을 수 있다.

- 만약 작성된 비표준 SQL이 DAO 코드에 들어간다면, 해당 DAO는 특정 DB에 종속적인 코드가 된다!

- 이를 해결하기 위해서는

  호환되는 표준 SQL만 사용 → 페이징 쿼리부터 못쓴다.
  DB 별 DAO 만들기
  SQL을 독립시켜 DB에 따라 변경 가능하게 만들기
  해당 방법은 7장에서 다뤄볼 예정이다.

#### 호환성 없는 SQLException의 DB 에러 정보

- DB마다 에러의 종류, 원인이 제각각이다.

```java
// MySQL에서 중복된 키를 가진 데이터를 입력하려고 시도했을 때
if (e.getErrorCode() == MysqlErrorNumbers.ER_DUP_ENTRY) { ...
```
- JDBC는 SQLException에 해당 에러들을 담는다.
- 그런데 이마저도 SQLException.getErrorCode()로 에러 코드를 가져왔을 때, DB 벤더마다 에러코드가 달라서 각각 처리해주어야 한다.
getSQLState()와 같은 메소드로 예외 상황에 대한 상태 정보를 가져올 수 있지만, 해당 값을 신뢰하기 힘들다. 어떤 DB는 표준 코드와 상관없는 엉뚱한 값이 들어있기도 하다.
- 결과적으로 SQL 상태 코드를 믿고 결과를 파악하도록 코드를 작성하는 것은 위험하다. 결국 호환성 없는 에러 코드와 표준을 잘 따르지 않는 상태 코드를 가진 SQLException만으로 DB에 독립적인 유연한 코드를 작성하는 것은 불가능에 가깝다.

- JDBC는 Connection, Statement, ResultSet 등의 표준 인터페이스로 기능을 제공한다. 따라서 개발자들은 DB 종류와 상관없이 일관된 방법으로 프로그래밍이 가능하다. 객체지향적 프로그래밍이 가능하다.

- 하지만, DB 종류에 따라 데이터 액세스 코드가 달라질 수 있다.


### 4-2-2. DB 에러 코드 매핑을 통한 전환
- 스프링은 DataAccessException의 서브 클래스로 세분화된 예외 클래스들을 정의하고 있다.

- SQL 문법 : BadSqlGrammerException
- DB 커넥션 : DataAcessResourceFailureException
- 데이터의 제약조건을 위배했거나 일관성을 지키지 못함 : DataIntegrityViolationException
- 그 중에서도 중복 키 때문에 발생한 경우 : DuplicatedKeyException

- 그런데 문제가 있다. DB마다 에러 코드가 다르다. => 이를 해결하기 위해, DB별 에러 코드를 참고해 발생한 예외 원인을 해석해줄 해석기가 필요하다!

#### 스프링 코드 매핑 테이블

```xml

<bean id="Oracle" class="org.springframework.jdbc.support.SQLErrorCodes">
        <property name="badSqlGrammarCodes">
            <value>900,903,904,917,936,942,17006,6550</value>
        </property>
        <property name="invalidResultSetAccessCodes">
            <value>17003</value>
        </property>
        <property name="duplicateKeyCodes">
            <value>1</value>
        </property>
        <property name="dataIntegrityViolationCodes">
            <value>1400,1722,2291,2292</value>
        </property>
        <property name="dataAccessResourceFailureCodes">
            <value>17002,17447</value>
        </property>
        <property name="cannotAcquireLockCodes">
            <value>54,30006</value>
        </property>
        <property name="cannotSerializeTransactionCodes">
            <value>8177</value>
        </property>
        <property name="deadlockLoserCodes">
            <value>60</value>
        </property>
</bean>
```
- 이를 통해, JdbcTemplate은 DB 에러 코드를 적절한 DataAccessException 서브클래스로 매핑한다.

```java
public void add(User user) throws DuplicateKeyException {
    this.jdbcTemplate.update("insert into users(id, name, password) values (?, ?, ?)"
            , user.getId()
            , user.getName()
            , user.getPassword()
    );
}
public void add(User user) throws DuplicateUserIdException {
    try {
        this.jdbcTemplate.update("insert into users(id, name, password) values (?, ?, ?)"
                , user.getId()
                , user.getName()
                , user.getPassword()
        );
    } catch (DuplicateKeyException e) {
        throw new DuplicateUserIdException(e);
    }
}
```
- 위와 같이 더 명확한 예외 클래스로 예외전환도 가능하다.

### 4-2-3. DAO 인터페이스와 DataAccessException 계층 구조
- DataAcessException은 의미가 같은 예외라면 데이터 액세스 기술의 종류와 상관없이 일관된 예외가 발생하도록 만들어준다.

- 데이터 액세스 기술에 독립적인 추상화된 예외를 제공하는 것이다.

#### DAO 인터페이스와 구현의 분리

- DAO를 인터페이스로 분리하면, 내부의 데이터 액세스 기술에 의존하지 않게 된다. POJO를 주고받으며 데이터 액세스 기능을 사용하기만 하면 된다. 이때 우리는 DAO 안에서 throw하지 않는다.
- 만약 throw를 한다면?

```java
public interface UserDao {
  public void add(User user) throws SQLException;
}
자바 DATA 접근 API가 바뀔 시 인터페이스도 바뀐다.

public interface UserDao {
  public void add(User user) throws PersistentException; // JPA
  public void add(User user) throws HibernateException; // Hibernate
  public void add(User user) throws JdoException; // JDO
}
```
- 이를 통해 DAO 사용 기술에 독립적인 인터페이스 선언이 가능해졌다.

#### 데이터 엑세스 예외 추상화와 DataAcessException 계층구조

- 스프링은 자바의 다양한 데이터 액세스 기술을 사용할 때 발생하는 예외들을 추상화해서 DataAcessException 계층구조 안에 정리해놓았다

- JdbcTemplate과 같이 스프링의 데이터 액세스 지원 기술을 이용해 DAO를 만들면, 사용 기술에 독립적인 일관성 있는 예외를 던질 수 있다.

- 결국 인터페이스 사용, 런타임 예외 전환과 함께 DataAccessException 예외 추상화를 적용하면 데이터 액세스 기술과 구현 방법에 독립적인 이상적인 DAO를 만들 수 있다.

### 4-2-4. 기술에 독립적인 UserDao 만들기

#### 인터페이스 적용
- setDataSource() 메소드는 인터페이스에 추가하면 안된다.
- UserDao의 구현 방법에 따라 변경될 수 있는 메소드이다.
- UserDao를 사용하는 클라이언트가 알고 있을 필요도 없다. 이후 빈 클래스를 변경하면 된다!

```java
public interface UserDao {
    void add(User user);
    User get(String id);
    User getByName(String name);
    List<User> getAll();
    void deleteAll();
    int getCount();
}
```
```xml
<bean id="dataSource" class="org.springframework.jdbc.datasource.SimpleDriverDataSource">
    <property name="username" value="postgres" />
    <property name="password" value="iwaz123!@#" />
    <property name="driverClass" value="org.postgresql.Driver" />
    <property name="url" value="jdbc:postgresql://localhost/toby_spring" />
</bean>

<bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
    <property name="dataSource" ref="dataSource" />
</bean>

<bean id="userDao" class="toby_spring.user.dao.UserDaoJdbc">
    <property name="jdbcTemplate" ref="jdbcTemplate" />
</bean>
```
- 테스트는 굳이 수정해줄 필요 없다. @Autowired 를 통해 자동으로 빈을 주입받는다.

#### DataAccessException 활용 시 주의사항

- 이렇게 스프링을 활용하면 DB 종류나 데이터 액세스 기술에 상관없이 키 값이 중복이 되는 상황에서는 동일한 예외가 발생하리라고 기대할 것이다.
- 하지만 안타깝게도 DuplicateKeyException은 JDBC를 이용하는 경우에만 발생한다!

- 데이터 액세스 기술을 하이버네이트나 JPA를 사용했을 때도 동일한 예외가 발생할 것으로 기대하지만 실제로 다른 예외가 던져진다.
- 예를 들어 하이버네이트는 중복 키가 발생하는 경우에 하이버네이트의 ConstraintViolationException을 발생시킨다.
- 따라서, DataAccessException을 잡아서 처리하는 코드를 만들려고 한다면 미리 학습 테스트를 만들어서 실제로 전환되는 예외의 종류를 확인해 둘 필요가 있다.

🙋‍♀️ 만약 DAO에서 사용하는 기술의 종류와 상관없이 동일한 예외를 얻고 싶다면?

- DuplicatedUserIdException처럼 직접 예외를 정의해두고, 각 DAO의 add() 메소드에서 좀 더 상세한 예외 전환을 해주면 된다.

#### Jdbc에서 SQLException 해석해보기

- SQLErrorCodeSQLExceptionTranslator 를 통해 SQLException를 DB, 데이터 접근 기술에 따라 해석할 수 있다!
```java
@Test
@DisplayName("SQLException DB 에러코드 해석기로 DataAccessException 해석해보기")
public void sqlExceptionTranslate() {
    try {
        userDao.add(user1);
        userDao.add(user1);
    }catch(DataAccessException ex) {
        SQLException sqlEx = (SQLException) ex.getRootCause();
        SQLExceptionTranslator set =
                new SQLErrorCodeSQLExceptionTranslator(this.dataSource);

        DataAccessException translate = set.translate(null, null, sqlEx);
        Assertions.assertEquals(DuplicateKeyException.class, translate.getClass());
    }
}
```

단, 이때 DataSource를 인자로 주지 않을 시, 더욱 포괄적인 에러인 DataIntegrityViolationException 가 결과로 나온다!

```java
@Test
public void save() {
    try {
        Item item = new Item("A");
        Item item2 = new Item("A");
        itemRepository.save(item);
        itemRepository.save(item2);
    } catch(DataAccessException e) {
        SQLException sqlException = (SQLException) e.getRootCause();
        SQLExceptionTranslator set = new SQLErrorCodeSQLExceptionTranslator(dataSource);
        DataAccessException translate = set.translate(null, null, sqlException);
        assertThat(translate.getClass()).isEqualTo(DuplicateKeyException.class);
    }
}
```
## 4-3. 정리
- 예외를 잡아서 아무런 조취를 취하지 않거나 의미 없는 throws 선언을 남발하는 것은 위험 하다.
- 예외는 복구하거나 예외처리 오브젝트로 의도적으로 전달하거나 적절한 예외로 전환해야 한다.
- 좀 더 의미 있는 예외로 변경하거나, 불필요한 catch/throws를 피하기 위해 런타임 예외로 포장하는 두 가지 방법의 예외 전환이 있다.
- 복구할 수 없는 예외는 가능한 한 빨리 런타임 예외로 전환하는 것이 바람직하다.
- 애플리케이션의 로직을 담기 위한 예외는 체크 예외로 만든다.
- JDBC의 SQLException은 대부분 복구할 수 없는 예외이므로 런타임 예외로 포장해야 한다.
- SQLException의 에러 코드는 DB에 종속되기 때문에 DB에 독립적인 예외로 전환될 필요가 있다.
- 스프링은 DataAccessException을 통해 DB에 독립적으로 적용 가능한 추상화된 런타임 예외 계층을 제공한다.
- DAO를 데이터 액세스 기술에서 독립시키려면 인터페이스 도입과 런타임 예외 전환, 기술에 독립적인 추상화된 예외로 전환이 필요하다.
