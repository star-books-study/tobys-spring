# 3장. 템플릿
- 개방 폐쇄 원칙(OCP) 은 코드에서 어떤 부분은 변경을 통해 그 기능이 다양해지고 확장하려는 성질이 있고, 어떤 부분은 고정되어 있고 변하지 않으려는 성질이 있음을 말해준다.
- 템플릿이란 변경이 거의 일어나지 않으며 일정한 패턴으로 유지되는 특성을 가진 부분을, **자유롭게 변경되는 부분으로부터 독립시켜서** 효과적으로 활용할 수 있게 하는 방법

## 3.1. 다시 보는 초난감 DAO
### 3.1.1. 예외처리 기능을 갖춘 DAO
```java
public void deleteAll() throws SQLException {
    Connection c = dataSource.getConnection();

    PreparedStatement ps = c.preparedStatement("delete from users");
    ps.executeUpdate();     //  여기서 예외가 발생하면 바로 메소드 실행이 중단되면서 DB 커넥션이 반환되지 못한다.

    ps.close();
    c.close();
}
```
- DB 풀은 매번 getConnection()으로 가져간 **커넥션을 명시적으로 close()해서 돌려줘야지 다시 풀에 넣었다가 다음 커넥션 요청이 있을 때 재사용**할 수 있다. 
- 그런데 오류가 날 때마다 미처 반환되지 못한 Connection이 계속 쌓이면 어느 순간에 커넥션 풀에 여유가 없어지고 리소스가 모자란다는 심각한 오류를 내며 서버가 중단될 수 있다.
- 리소스 반환과 close()
  - 단순히 생각하면 만들어진 것을 종료한다고 볼 수도 있지만, 보통 `리소스를 반환` 한다는 의미로 이해하면 좋다
  - 미리 정해진 풀 안에 **제한된 수의 리소스 (Connection, Statement) 를 만들어 두고 필요할 때 이를 할당하고, 반환하면 다시 풀에 넣는 방식으로 운영** 된다.
  - 매번 새로운 리소스를 생성하는 대신, 풀에 미리 만들어둔 리소스를 돌려가며 사용하는 게 훨씬 유리하다
  - 대신, 사용한 리소스는 **빠르게 반환** 해야 한다.
  - close() 메소드는 사용한 리소스를 풀로 다시 돌려주는 역할을 한다.
- 예외사항에서도 리소스를 제대로 반환할 수 있도록 try/catch/finally 구문 사용을 권장하고 있다
```java
public void deleteAll() throws SQLException {
    Connection c = null;
    PreparedStatement ps = null;
    
    try {
        c = dataSource.getConnection();
        ps = c.prepareStatement("delete from users");
        ps.executeUpdate();     //  예외가 발생할 수 있는 코드를 모두 try 블록으로 묶어준다.
    } catch (SQLException e) {
        throw e;        
    } finally {  //  finally이므로 try 블록에서 예외가 발생했을 떄나 안 했을 때나 모두 실행된다.
        if (ps != null) {
            try {
                ps.close();
            } catch (SQLException e) {} //  ps.close() 메소드에서도 SQLException이 발생할 수 있기 때문에 잡아줘야한다.
        }
        if (c != null) {
            try {
                c.close();
            } catch (SQLException e) {}
        }
    }
}
```
- 어느 시점에서 예외가 발생했는지에 따라서 close()를 사용할 수 있는 변수가 달라질 수 있기 때문에 finally에서는 반드시 c와 ps가 null이 아닌지 먼저 확인한 후에 close() 메소드를 호출해야 한다.
- 문제는 이 close()도 SQLException이 발생할 수 있는 메소드이므로, try/catch 문으로 처리해줘야 한다.
#### JDBC 조회 기능의 예외처리
- 조회를 위한 JDBC 코드에는 Connection, PreparedStatement 외에도 ResultSet 이 추가된다.
- 만들어진 ResultSet 을 close 하는 구문이 추가되며, **close() 는 만들어진 순서의 반대로 하는 게 원칙이다**
```java
public void deleteAll() throws SQLException {
    Connection c = null;
    PreparedStatement ps = null;
    ResultSet rs = null;

    try {
        c = dataSource.getConnection();
        ps = c.prepareStatement("delete from users");
        rs.executeQuery();
        rs.next();
        return rs.getInt(1);
    } catch (SQLException e) {
        throw e;        
    } finally {  //  finally이므로 try 블록에서 예외가 발생했을 떄나 안 했을 때나 모두 실행된다.
        if (rs != null) {
            try {
                ps.close();
            } catch (SQLException e) {}
        }
        if (ps != null) {
            try {
                ps.close();
            } catch (SQLException e) {}
        }
        if (c != null) {
            try {
                c.close();
            } catch (SQLException e) {}
        }
    }
}
```

## 3.2. 변하는 것과 변하지 않는 것
### 3.2.1. JDBC try/catch/finally 코드의 문제점
- 복잡한 try/catch/finally 블록이 2중 중첩인데다 모든 메소드마다 반복된다.

### 3.2.2. 분리와 재사용을 위한 디자인 패턴 적용
- 핵심은 변하지 않는, 그러나 많은 곳에서 중복되는 코드와 로직에 따라 자주 확장되고 자주 변하는 코드를 잘 분리해내는 것이다.

```java
public class UserDao {
    public void add(User user) throws ClassNotFoundException, SQLException {
		Connection c = null;

		PreparedStatement ps = null;

        try {
            c = datasource.getConnection();
            ps = c.prepareStatement("delete from users") // 이 부분만 변하는 부분 -> makeStatement 로 추출
        } catch (SQLException e) {
            throw e;
        } finally {
            if (ps != null) {
                try {
                    ps.close();
                } catch (SQLException e) {

                }
            }
            
            if (c != null) {
                try {
                    c.close();
                } catch (SQLException e) {

                }
            }
        }
	}
```
#### 템플릿 메소드 패턴의 적용
- 템플릿 메소드 패턴 : `상속` 을 통해 기능을 확장해서 사용하기
  - 변하지 않는 부분을 슈퍼클래스에, 변하는 부분을 `추상 메소드` 로 정의해서 서브클래스에서 오버라이드하여 새롭게 정의하여 쓰도록 함
```java
abstract protected PreparedStatement makeStatement(Connection c) throws SQLException;

public class UserDaoDeleteAll extends UserDao {
    protected PreparedStatement makeStatement(Connection c) throws SQLException {
        PreparedStatement ps = c.preparedStatement("delete from users");
        return ps;
    }
}
```
- UserDao 클래스의 기능을 확장하고 싶을 때마다 상속을 통해 자유롭게 확장 가능하며, 확장 때문에 기존의 상위 DAO 클래스에 불필요한 변화는 생기지 않는다.
- 따라서 객체지향 설계의 핵심 원리인 개방 폐쇄 원칙(OCP)을 그럭저럭 지키는 구조를 만들어낼 수 있다.
- 단점
  - DAO 로직마다 상속을 통해 새로운 클래스를 만들어야 하는 단점이 있다.
  - **확장 구조가 클래스 설계시점에 고정** 된다.
    - 컴파일 시점에 PreparedStatement 를 담고 있는 서브 클래스와 try/catch/finally 블록간의 관계가 이미 결정되어 있으므로 **관계에 대한 유연성이 떨어진다**

#### 전략 패턴의 적용
- 전략 패턴
  - 개방 폐쇄 원칙의 실현에도 가장 잘 들어맞는 패턴
  - 자신의 기능 맥락(context) 에서, 필요에 따라 **변경이 필요한 알고리즘 (독립적인 책임으로 분리 가능한 기능)을 인터페이스를 통해 통째로 외부로 분리시키고, 이를 구현한 구체적인 알고리즘을 필요에 따라 바꿔서 사용** 할 수 있게 하는 디자인 패턴 -> 전략을 바꾼다! 라고 생각
    - PreparedStatement 를 만들어주는 부분이 전략패턴에서 말하는 `전략`
    - 인터페이스의 메소드를 통해 PreparedStatement 생성 전략을 호출해주면 된다
```java
public interface StatementStrategy {
    PreparedStatement makePreparedStatement(Connection c) throws SQLException;
}

public class DeleteAllStatement implements StatementStrategy {
    protected PreparedStatement makeStatement(Connection c) throws SQLException {
        PreparedStatement ps = c.preparedStatement("delete from users");
        return ps;
    }
}

public void deleteAll() throws SQLException {
    ...
    try {
        c = dataSource.getConnection();

        StatementStrategy strategy = new DeleteAllStatement();  
        //  전략 클래스가 DeleteAllStatement로 고정됨으로써 OCP 개방 원칙에 맞지 않게 된다.
        ps = starategy.makePreparedStatement(c);

        ps.executeUpdate();
    } catch (SQLException e) {...}
}
```
- 하지만 전략패턴은 필요에 따라 `컨텍스트는 그대로 유지` 하면서, 전략을 `바꿔 사용` 가능한데, 위의 코드는 이미 컨텍스트 내에서 전략 클래스가 고정되어 있어서 이상하다

#### DI 적용을 위한 클라이언트/컨텍스트 분리
- 전략 패턴에 따르면 Context가 **어떤 전략을 사용하게 할 것인가**는 Context를 사용하는 **앞단의 Client가 결정**하는 게 일반적이다.
  - Client 가 구체적인 전략을 선택하고 오브젝트로 만들어 Context (변하지 않는 부분) 에 전달 
  - Context 는 전달받은 Strategy 구현 클래스의 오브젝트를 사용
![alt text](image.png)
- 컨텍스트에 해당하는 부분을 별도 메소드로 독립시킨다.
- 클라이언트는 전략클래스의 오브젝트를 컨텍스트 메소드 (jdbcContextWithStatementStrategy) 호출하며 전달해야 하므로, 전략 인터페이스 (StatementStrategy) 를 컨텍스트 메소드 파라미터로 지정한다.
```java
public class DeleteAllStatement implements StatementStrategy {
    protected PreparedStatement makeStatement(Connection c) throws SQLException {
        PreparedStatement ps = c.preparedStatement("delete from users");
        return ps;
    }
}

public void jdbcContextWithStatementStrategy(StatementStrategy stmt) throws SQLException {
    Connection c = null;
    PreparedStatement ps = null;

    try {
        c = dataSource.getConnection();
        ps = stmt.makePreparedStatement(c);
        ps.executeUpdate();
    } catch (SQLException e) {
        throw e;
    } finally {
        if (ps != null) { try { ps.close(); } catch (SQLException e) }
        if (c != null) { try { c.close(); } catch (SQLException e) }
    }
}
```
- 클라이언트로부터 StatementStrategy 타입의 전략 오브젝트를 제공받고 try/catch/finally 구조로 만들어진 컨텍스트 내에서 작업을 수행한다.
- **클라이언트에서는 전략 오브젝트를 생성하고, 컨텍스트를 호출한다**
```java
public void deleteAll() throws SQLException {
    StatementStrategy st = new DeleteAllStatement();    //  선정한 전략 클래스의 오브젝트 생성
    jdbcContextWithStatementStrategy(st);               //  컨텍스트 호출. 전략 오브젝트 주입
}
```
- 클라이언트가 컨텍스트가 사용할 전략을 정해서 결정한다는 점에서 DI 구조로 이해할 수도 있다.
- DI 의 가장 중요한 개념은 제 3자 (Client) 의 도움을 통해 두 오브젝트 (전략과 컨텍스트 메소드) 사이의 유연한 관계가 설정되도록 만드는 것이다.

## 3.3. JDBC 전략 패턴의 최적화
- 컨텍스트는 PreparedStatement 를 실행하는 `JDBC 작업 흐름` 이고, 전략은 `PreparedStatement 를 생성`하는 것이다.

### 3.3.2. 전략과 클라이언트의 동거
- 이전의 개선된 코드가 가진 2가지 문제점
  - DAO 메소드마다 새로운 StatementStrategy 구현 클래스를 만들어야 한다.
  - DAO 메소드에서 StatementStrategy에 전달할 User와 같은 부가적인 정보가 있는 경우, 이를 전달하고 저장해 둘 생성자와 인스턴스 변수를 번거롭게 만들어야 한다.
```java
public class UserDao {
	private ConnectionMaker connectionMaker;
	
	public UserDao() {
		this.connectionMaker = new DConnectionMaker();
	}

	public void add(User user) throws ClassNotFoundException, SQLException {
		Connection c = this.connectionMaker.getConnection();
	}
```
- StatementStrategy 전략 클래스를 매번 독립된 파일로 만들지 말고 **UserDao 클래스 안에 내부 클래스로 정의**해버리면 클래스 파일이 많아지는 문제는 해결할 수 있다.
```java
public class UserDao {
	private ConnectionMaker connectionMaker;
	
	public UserDao() {
		this.connectionMaker = new DConnectionMaker();
	}

	public void add(User user) throws ClassNotFoundException, SQLException {
        class AddStatement implements StatementStrategy {   //  add() 메소드 내부에 선언된 로컬 클래스
            User user;

            public AddStatement(User user) {
                this.user = user;
            }

            public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
                PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
                ps.setString(1, user.getId());
                ps.setString(2, user.getName();
                ...
                return ps;
            }
        }
        StatementStrategy st = new AddStatement(user);
        jdbcContextWithStatementStrategy(st);
	}  
}
```
- 로컬 클래스로 만들어두니, 내부 클래스의 특징을 이용해 **자신이 정의된 메소드의 로컬 변수에 직접 접근 가능**하다.
  - 즉, add() 메소드 내에 AddStatement 클래스를 정의하면 번거롭게 생성자를 통해 User 오브젝트를 전달해줄 필요가 없다.
  - 따라서 AddStatement 클래스가 정의된 add 메소드의 user 라는 메소드 파라미터 (일종의 로컬변수) 에 접근할 수 있다.
  - 다만, **내부 클래스에서 외부 변수 사용할 때는 외부 변수를 반드시 final 로 선언** 해줘야 한다.
```java
public class UserDao {
	private ConnectionMaker connectionMaker;
	
	public UserDao() {
		this.connectionMaker = new DConnectionMaker();
	}

	public void add(final User user) throws ClassNotFoundException, SQLException {
        class AddStatement implements StatementStrategy {   
            public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
                PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
                ps.setString(1, user.getId()); // 내부 클래스에서 외부의 메소드 로컬 변수에 직접 접근 가능
                ps.setString(2, user.getName();
                ...
                return ps;
            }
        }
        StatementStrategy st = new AddStatement(); // 생성자 파라미터로 user 전달 불필요
        jdbcContextWithStatementStrategy(st);
	}  
}
```

#### 익명 내부 클래스
- AddStatement 클래스는 add() 메소드에서만 사용할 용도로 만들어져서, 간결하게 클래스명도 제거할 수 있다.
- 클래스 선언과 동시에 오브젝트를 생성한다.
- 클래스를 **재사용할 필요가 없고**, 구현한 인터페이스 타입으로만 사용할 경우에 유용하다
`new 인터페이스 이름() { 클래스 본문 };`
```java
// AS-IS
class AddStatement implements StatementStrategy {   
    public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
        PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
        ps.setString(1, user.getId()); // 내부 클래스에서 외부의 메소드 로컬 변수에 직접 접근 가능
        ps.setString(2, user.getName();
        ...
        return ps;
    }
}
StatementStrategy st = new AddStatement(); // 생성자 파라미터로 user 전달 불필요
jdbcContextWithStatementStrategy(st);

// TO-BE
StatementStrategy st = new StatementStrategy() {
    public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
        PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
        ps.setString(1, user.getId()); 
        ps.setString(2, user.getName();
        ...
        return ps;
    }
}
```
- 위와 같이 만들어진 익명 내부 클래스 오브젝트는 딱 1번만 사용하기에 변수에 담지말고, jdbcContextWithStatementStrategy() 메소드 파라미터에서 바로 생성하는 게 낫다.
```java
public void add(final User user) throws SQLException {
    jdbcContextWithStatementStrategy(
        new StatementStrategy() {
            public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
                PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
    
                ps.setString(1, user.getId());
                ps.setString(2, user.getName();
                ...
                return ps;
            }
        }
    );
}
```

## 3.4. 컨텍스트와 DI
### 3.4.1. JdbcContext 의 분리
- 전략 패턴의 구조로 보았을 때,
  - UserDao 의 메소드 (add 메소드) : `클라이언트`
  - 익명 내부 클래스로 만들어진 StatementStrategy : `전략`
  - UserDao 내의 PreparedStatement 를 실행하는 기능을 수행하는 jdbcContextWithStatementStrategy : `컨텍스트 메소드`
  - 컨텍스트 메소드는 JDBC 일반적인 흐름을 담고 있으므로 다른 DAO 에서도 사용할 수 있게 분리해보자

#### 클래스 분리
- JdbcContext 라는 클래스를 만들고, jdbcContextWithStatementStrategy 메소드를 이름을 변경하여 옮겨 둔다
```java
// AS-IS 
public void jdbcContextWithStatementStrategy(StatementStrategy stmt) throws SQLException {
    Connection c = null;
    PreparedStatement ps = null;

    try {
        c = dataSource.getConnection();
        ps = stmt.makePreparedStatement(c);
        ps.executeUpdate();
    } catch (SQLException e) {
        throw e;
    } finally {
        if (ps != null) { try { ps.close(); } catch (SQLException e) }
        if (c != null) { try { c.close(); } catch (SQLException e) }
    }
}

// TO-BE
public class JdbcContext {
    private DataSource dataSource;

    public void setDataSource(DataSource dataSource) {  //  DataSource 타입 빈을 DI 받을 수 있게 준비
        this.dataSource = dataSource;
    }

    public void workWithStatementStrategy(StatementStrategy stmt) throws SQLException {
        Connection c = null;
        PreparedStatement ps = null;

        try {...} 
        catch (SQLException e) {...}
        finally {...}
    }
}
```
- UserDao 는 JdbcContext 를 DI 받아서 사용해야 하므로 아래와 같이 코드를 수정한다.
```java
// AS-IS : add 메소드
public void add(final User user) throws SQLException {
    jdbcContextWithStatementStrategy(
        new StatementStrategy() {
            public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
                ...
            }
        }
    );
}

// TO-BE
public class UserDao {
    ...
    private JdbcContext jdbcContext;

    public void setJdbcContext(JdbcContext jdbcContext) {
        this.jdbcContext = jdbcContext;            
         // jdbcContext를 DI받도록 만든다.
    }

    public void add(final User user) throws SQLException {
        this.jdbcContext.workWithStatementStrategy(     
            // DI 받은 JdbcContext의 컨텍스트 메소드를 사용하도록 변경한다.
            new StatementStrategy() {...}
        );
    }
}
```
