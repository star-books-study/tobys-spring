# 3장. 템플릿

**개방 폐쇄 원칙 (OCP)**

어떤 부분은 변경을 통해 그 기능이 다양해지고 확장하려는 성질이 있고, 어떤 부분은 고정되어 있고 변하지 않으려는 성질이 있다. 변화의 특성이 다른 부분을 구분해주고, 각각 다른 목적과 다른 이유에 의해 다른 시점에 독립적으로 변경될 수 있는 효율적인 구조를 만들어주는 것이 개방 폐쇄 원칙이다.

## 3.1 다시보는 초난감 DAO

### 3.1.1 예외처리 기능을 갖춘 DAO
- JDBC 수정 기능의 예외처리 코드
        - 1장에서 만든 UserDao에 예외 처리를 추가한다. DB 커넥션 관련 코드는 리소스를 공유하기 때문에 예외가 발생하여 정상적으로 close()가 안 되면, 리소스 반환이 안 되어 나중에 리소스 부족과 관련된 에러가 발생할 수 있다. 일반적으로 서버에서는 제한된 DB 커넥션을 만들어 풀(Pool)로 관리한다. close()를 안 하면 다시 풀에 리소스를 넣지 못해서 반환하지 못한 커넥션이 쌓이면 다음 커넥션 요청 때 풀에 리소스가 없어서 서버 중단이 될 수 있다. 다음과 같이 try/catch/finally로 예외처리를 한다.

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

- deletaAll()에 예외처리를 적용했다. c나 ps가 null인 상태에서 close()를 호출하면 NullPointerException이 발생하기 때문에 위와 같이 null여부를 우선 체크한 후 null이 아니면 close()를 호출하였다. close() 자체도 exception이 발생할 수 있기 때문에 try/catch문을 한 번 더 적용했다.

## 3.2 변하는 것과 변하지 않는 것

### 3.2.1 JDBC try/catch/finally 코드의 문제점
 try/catch/finally가 계속해서 모든 메소드마다 반복된다. 중복의 문제가 발생하는 것이다.

 

### 3.2.2 분리와 재사용을 위한 디자인 패턴 적용
- 메소드마다 preparedStatement로 SQL을 작성하는 부분을 제외하고 대부분 같은 코드가 반복된다. 이 코드를 분리해야한다. 우선 메소드 추출기법을 생각해볼 수 있겠는데 이 방법은 변하는 부분을 makeStatement()로 빼기 때문에 문제가 있다. 왜냐하면 보통 메소드 추출 기법은 공통된 코드를 빼서 여러 곳에서 재사용하기 위함인데, 이는 반대로 변하는 부분을 뺐기 때문에 재사용이 어렵다.

 

**방법1.템플릿 메소드 패턴 적용하기**

- 템플릿 메소드 패턴은 공통된 알고리즘 흐름을 호출하는 템플릿 메소드를 정의하고 변하는 부분은 따로 서브클래스에서 오버라이드 재정의하여 사용하는 디자인패턴 기법이다.
- deleteAll()을 예로 든다면, preparedStatement() 호출 부분이 다른 dao 메소드에서도 변하는 부분이기 때문에 이부분을 따로 추상 메소드 선언을 하고 이를 따로 구현한 코드를 사용하는 것이다. UserDao는 당연히 추상 클래스로 정의해야 한다. 상속해서 만든 클래스는 다음과 같다.
 
