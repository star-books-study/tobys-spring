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