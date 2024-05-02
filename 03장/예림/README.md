# 3장. 템플릿

- 개방폐쇄원칙은 코드에서 어떤 부분은 변경을 통해 그 기능이 다양해지고 확장하려는 성질이 있고, 어떤 부분은 고정되어 있고 변하지 않으려는 성질이 있음을 말해준다.
- 템플릿 : 이렇게 변하는 성질이 다른 코드 중에서 변경이 거의 일어나지 않으며 일정한 패턴으로 유지되는 특성을 가진 부분을 자유롭게 변경되는 성질을 가진 부분으로부터 독립시켜서 효과적으로 활용할 수 있도록 하는 방법
        => 변하지 않는 특성을 변하는 특성을 가진 부분으로부터 독립시킨다!

## 3.1 다시보는 초난감 DAO
UserDao에는 예외 상황 처리에 관해서 심각한 문제점이 있다.

### 3.1.1 예외처리 기능을 갖춘 DAO


#### JDBC 수정 기능의 예외처리 코드

- 1장에서 만든 UserDao에 예외 처리를 추가한다. DB 커넥션 관련 코드는 리소스를 공유하기 때문에 예외가 발생하여 정상적으로 close()가 안 되면, 리소스 반환이 안 되어 나중에 리소스 부족과 관련된 에러가 발생할 수 있다. 일반적으로 서버에서는 제한된 DB 커넥션을 만들어 풀(Pool)로 관리한다.
- close()를 안 하면 다시 풀에 리소스를 넣지 못해서 반환하지 못한 커넥션이 쌓이면 다음 커넥션 요청 때 풀에 리소스가 없어서 서버 중단이 될 수 있다.
- 다음과 같이 try/catch/finally로 예외처리를 한다.

```java
 public void deleteAll() throws SQLException {
        Connection c = null;
        PreparedStatement ps = null;
        
        try {
            c = dataSource.getConnection();
            ps = c.prepareStatement("delete from users");
            ps.executeUpdate();
        } catch(SQLException e) {
        	throw e;
        } finally {
            if(ps!=null) {try {ps.close();} catch(SQLException e) {}}
            if(c!=null) {try{c.close();}catch(SQLException e) {}}
        }
    }
```

- deletaAll()에 예외처리를 적용했다. c나 ps가 null인 상태에서 close()를 호출하면 NullPointerException이 발생하기 때문에 위와 같이 null 여부를 우선 체크한 후 null이 아니면 close()를 호출하였다. close() 자체도 exception이 발생할 수 있기 때문에 try/catch문을 한 번 더 적용했다.

#### JDBC 조회 기능의 처리
- ResultSet도 반환해야 하는 리소스이기 때문에 close()가 필요
```java
public int getCount() throws SQLException {
        Connection c = null;
        PreparedStatement ps = null;
        ResultSet rs = null;

        try {
                c = dataSource.getConnection();
                ps = c.preparedStatement("select count(*) from users");

                rs = ps.executeQuery();
                rs.next();
                return rs.getInt(1);

        } catch (SQLException e) {
                throw e;
        } finally {
                if(rs != null) {
                        try {
                                rs.close();
                        } catch(SQLException e) {
                        }
                }
                ... 생략 ...
                // ps와 c를 순서대로 닫아준다. close()는 만들어진 순서의 반대로 하는 것이 원칙이다
```



## 3.2 변하는 것과 변하지 않는 것

### 3.2.1 JDBC try/catch/finally 코드의 문제점
- try/catch/finally가 계속해서 모든 메소드마다 반복된다. 중복의 문제가 발생하는 것이다.
 

### 3.2.2 분리와 재사용을 위한 디자인 패턴 적용

> UserDao 메서드를 개선해보자!

- 개선할 deleteAll() 메서드는 다음과 같다.
```java
Connection c =null;
PreparedStatement ps =null;
try (
        c = dataSource.getConnection();

        ps = c.prepareStatement("delete from users"); // 변하는 부분

        pS.executeUpdate();
} catch (5QLException e) {
        throw e;
} finally {
        if (ps != null) { try ( pS.close(); } catch (SQLException e) {} }
        if (c != null) { try (c.close(); } catch (5QLException e) {} }
}
```

#### 메서드 추출
- 메소드마다 `preparedStatement`로 SQL을 작성하는 부분을 제외하고 대부분 같은 코드가 반복된다.
- 이 코드를 분리해야한다. 우선 메소드 추출기법을 생각해볼 수 있겠는데 이 방법은 변하는 부분을 `makeStatement()`로 빼기 때문에 문제가 있다.
- 왜냐하면 보통 메소드 추출 기법은 공통된 코드를 빼서 여러 곳에서 재사용하기 위함인데, 이는 반대로 변하는 부분을 뺐기 때문에 재사용이 어렵다.
 

#### 템플릿 메소드 패턴 적용하기

- 템플릿 메소드 패턴은 상속을 통해 기능을 확장해서 사용하는 부분이다.
- 변하지 않는 부분은 슈퍼 클래스에 두고 변하는 부분은 추상 메서드로 정의 -> 서브 클래스에서 오버라이드
- makeStatement() 메서드를 다음과 같이 추상 메서드 선언으로 변경한다. (UserDao 클래스도 추상 클래스로 변경)
        ```java
        abstract protected PreparedStatement makeStatement(Connection c) throws SQLException;
        ```
- 그리고 이를 상속하는 서브 클래스를 만들어서 거기서 이 메서드 구현

```java
public class UserDaoDeleteAll extends UserDao {
    protected PreparedStatment makeStatement(Connection c) throws SQLException {
        PreparedStatement ps = c.prepareStatement("delete from users");
        return ps;
    }
}
```
- 그러나 이런식으로 사용하면 각각의 **DAO 메소드마다 클래스를 만들어야 한다**는 문제점이 있다.
- 또한 확장구조가 클래스 설계 시점에 고정되어 OCP원칙에 위배된다.

         <img width="575" alt="스크린샷 2024-04-30 오후 8 38 49" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/d23aa863-e34d-4d73-8b97-749bda3d938e">

=> 상속을 통해 확장을 꾀하는 템플릿 메서드 패턴 OUT



#### 전략 패턴의 적용
- 오브젝트를 아예 둘로 분리하고 클래스 레벨에서는 인터페이스를 통해서만 의존하도록 만드는 **전략 패턴**을 사용해보자 (OCP 준수 / 템플릿 메서드 패턴보다 유연)
- 전략 패턴은 확장에 해당하는 변하는 부분을 별도의 클래스로 만들어 추상화된 인터페이스를 통해 위임하는 방식
<img width="538" alt="image" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/b849231c-18c1-474f-a091-203d64e726df">

- Context의 contextMethod()에서 일정한 구조를 가지고 동작하다가 확장 기능은 Strategy 인터페이스를 통해 외부의 독립된 전략 클래스에 위임
- 우선 preparedStatement를 만드는 전략을 인터페이스(StatemStrategy.java)로 다음과 같이 만든다.

```java
public interface StatementStrategy {
    PreparedStatment makePreparedStatement(Connection c) throws SQLException;
}
```

- 그리고 이것을 구현한 전략(DeleteAllStatement.java)으로 만든다.

```java
public class DeleteAllStatement implements StatementStrategy{
    public PreparedStatement makePreparedStatemnt(Connection c) throws SQLException{
        PreparedStatement ps = c.preparedStatement("delete from users");
        return ps;
    }
}
```
 
- 이제 UserDao의 deleteAll() 메소드에서 이 전략 인스턴스를 생성한 후 사용하면 된다. 다음과 같다.


```java
public void deleteAll() throws SQLException{
    //생략
    try { 
        c = dataSource.getConnection();
        StatementStrategy strategy = new DeleteAllStatement();
        ps = strategy.makePreparedStatement(c);
    
        ps.executeUpdate();
    }catch(SQLException e){
        //생략
    }
}
```
- 전략 패턴의 방식을 적용했으나 문제점이 있다. deleteAll()이 컨텍스트의 역할을 하는데 컨텍스트가 구체적인 전략클래스인 DeleteAllStatement를 사용하도록 고정되어 있다.
- 만약 DeleteAllStatement가 수정되면 컨텍스트에 직접적인 영향을 미치기 때문에 OCP원칙에 잘 맞는 코드라고 볼 수 없다.
- 실제로 구체적인 전략클래스를 생성 및 사용하는 것은 컨텍스트의 역할이 아니라 '클라이언트'의 역할이기 때문에 컨텍스트와 클라이언트의 분리작업이 필요하다.

 
#### DI적용을 위한 클라이언트와 컨텍스트 분리

- 컨텍스트가 어떤 전략을 사용할지에 대해서는 **클라이언트가 결정**하도록 해야한다.
	- 클라이언트가 구체적인 전략의 하나를 선택하고 오브젝트로 만들어서 Context에 전달 -> Context는 전달받은 Strategy 구현 클래스의 오브젝트 사용
<img width="603" alt="image" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/4dbcee18-53d7-43b4-b67a-dd360c7bae3b">

- 위의 그림은 UserDao의 ConnectionMaker DI 방식이 따랐던 흐름과 유사하다. add()메소드가 필요한 커넥션 객체를 UserDaoTest 클라이언트에서 전략적으로 생성하여 UserDao에 넘겨준 후 add() 메소드가 이를 사용했다. 지금 상황에서도 이런 흐름에 맞게 개선할 필요가 있다.

- 여기서는 try/catch/finally 코드가 유사하게 반복되기 때문에 이 코드를 따로 빼서 컨텍스트로 만든다. 그리고 기존의 deleteAll()은 클라이언트 역할을 하도록 DeleteAllStatement()를 만든 후 컨텍스트의 인자로 넘겨 컨텍스트를 호출하는 역할로 변경한다. 다음과 같다.

```java
public void jdbcContextWithStatementStrategy(StatementStrategy stmt) throws SQLException{
    Connection c = null;
    PreparedStatement ps = null;
         
    try {
        c = dataSource.getConnection();
        ps = stmt.makePreparedStatement(c); // 주목
        ps.executeUpdate();
    } catch(SQLException e) {
        throw e;
    } finally {
        if(ps!=null) {try {ps.close();} catch(SQLException e) {}}
        if(c!=null) {try {ps.close();} catch(SQLException e) {}}
    }
}
```
- StatementStrategy stmt : 클라이언트가 컨텍스트를 호출할 때 넘겨줄 전략 파라미터
```java
// 클라이언트 역할
public void deleteAll() throws SQLException {
    StatementStrategy st = new DeleteAllStatement(); // 선정한 전략 클래스의 오브젝트 생성
    jdbcContextWithStatementStrategy(st);   // 컨텍스트 호출. 전략 오브젝트 전달
}
```
- 이제 클라이언트와 컨텍스트가 DI를 이용하여 분리된 코드로 개선되었다.

## 3.3 JDBC 전략 패턴의 최적화
- 

### 3.4 컨텍스트와 DI

#### 3.4.1 컨텍스트를 JdbcContext 클래스로 분리하기
- `jdbcContextWithStateStrategy()`는 JDBC의 기본적 흐름을 담고 있는 컨텍스트로, 다른 DAO에서도 사용 가능하다.
- 그렇기 때문에 UserDao에서 따로 클래스로 분리하는 작업을 해본다.

#### 클래스로 분리하기

- 일단 UserDao에 있는 컨테스트 코드를 `JdbcContext`라는 클래스를 만든 후에 `workWithStatementStrategy()` 라는 이름으로 변경하여 작성한다. 그리고 DataSource에 의존하기 때문에 DataSource도 setter로 DI받을 수 있도록 만든다. 
```java
public class JdbcContext {
    private DataSource dataSource;
	
    public void setDataSource(DataSource dataSource) {
        this.dataSource = dataSource;
    }
	
    // 전략패턴 컨텍스트 역할
    public void workWithStatementStrategy(StatementStrategy stmt) throws SQLException{
    	 Connection c = null;
         PreparedStatement ps = null;
         
         try {
             c = dataSource.getConnection();
             ps = stmt.makePreparedStatement(c);
             ps.executeUpdate();
         } catch(SQLException e) {
         	throw e;
         } finally {
             if(ps!=null) {try {ps.close();} catch(SQLException e) {}}
             if(c!=null) {try {ps.close();} catch(SQLException e) {}}
         }
    }
}
```
- 이제 UserDao의 코드를 수정한다. 기존의 컨텍스트는 삭제한 후, JdbcContext를 DI받도록 만든다. 기존의 DataSource 코드는 아직 사용하고 있는 메소드들이 있기 때문에 그대로 냅둔다.
