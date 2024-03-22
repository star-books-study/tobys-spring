# 01장. 오브젝트와 의존 관계

## 들어가며
- 스프링의 핵심철학은 자바 엔터프라이즈 기술의 혼란 속에서 잃어버렸던 객체지향 기술의 진정한 가치를 회복시키고, 그로부터 객체지향 프로그래밍이 제공하는 폭넓은 혜택을 누릴 수 있도록 기본으로 돌아가자는 것이다.
- 그래서 스프링이 가장 관심을 많이 두는 대상이 오브젝트이며, 스프링을 이해하려면 먼저 오브젝트에 깊은 관심을 가져야 한다.
- 1장에서는 스프링이 관심을 갖는 대상인 오브젝트의 설계와 구현, 동작원리에 더 집중하기를 바란다.

## 1.1 초난감 DAO
사용자 정보를 JDBC API를 통해 DB에 저장하고 조회할 수 있는 간단한 DAO를 하나 만들어보자.
> DAO : DB를 사용해 데이터를 조회하거나 조작하는 기능을 전담하도록 만든 오브젝트

### 1.1.1 User
사용자 정보를 저장할 때는 자바빈 규약을 따르는 오브젝트를 이용하면 편하다.


> **자바빈**
> 
> 원래 비주얼 툴에서 조작 가능한 컴포넌트. 자바의 주력 개발 플랫폼이 웹 기반의 엔터프라이즈 방식으로 바뀌면서 비주얼 컴포넌트로서 자바빈은 인기를 잃어핬지만, 자바빈의 몇 가지 코딩 관례는 JSP 빈, ELB와 같은 표준 기술과 자바빈 스타일의 오브젝트를 사용하는 오픈소스 기술을 통해 계속 이어져 왔다. 이제는 자바빈이라고 말하면 비주얼 컴포넌트라기보다는 다음 두 가지 관례를 따라 만들어진 오브젝트를 가리킨다. 간단히 빈이라고 부르기도 한다.
>
> - 디폴트 생성자 : 자바빈은 파라미터가 없는 디폴트 생성자를 갖고 있어야 한다.
> - 프로퍼티 : 자바빈이 노출하는 이름을 가진 속성. setter와 getter를 통해 수정 또는 조회할 수 있다.

```java
// 1-1. 사용자 정보 저장용 자바빈 User 클래스
package springbook.user.domain;

public class User {
  String id;
  String name;
  String password;

  public String getId() {
    return id;
  }
  public setId() {
    this.id = id;
  }
  public String getName() {
    return name;
  }
  public String getPassword() {
    return password;
  }
  public void setPassword(String password) {
    this.password = password;
  }
}

```

이제 User 오브젝트가 담긴 정보가 실제로 보관될 DB의 테이블을 하나 만들어보자. 테이블 이름은 USER로 User 클래스의 프로퍼티와 동일하게 구성한다.


- 테이블 필드 내역
  | 필드명 | 타입 | 설정 |
  |------|-----|-----|
  | id | VARCHAR(10) | Primary Key |
  | Name | VARCHAR(20) | Not Null |
  | Password | VARCHAR(20) | Not Null |
- SQL문
  ```sql
  create table users (
    id varchar(10) primary key,
    name varchar(20) not null,
    password varchar(10) not null
  )
  ```

### 1.1.2 UserDao
사용자 정보를 DB에 넣고 관리할 수 있는 DAO 클래스를 만들어보자.
JDBC를 이용하는 작업의 일반적인 순서는 다음과 같다.
- DB 연결을 위한 Connection을 가져온다.
- SQL을 담은 Statement(또는 PreparedStatement)를 만든다.
- 만들어진 Statement를 실행한다.
- 조회의 경우 SQL 쿼리의 실행결과를 ResultSet으로 받아서 정보를 저장할 오브젝트(여기는 User)에 옮겨준다.
- 작업 중에 생성된 Connection, Statement, ResultSet 같은 리소스는 작업을 마친 후 반드시 닫아준다.
- JDBC API가 만드는 예외를 잡아서 직접 처리하거나, 메소드에 throws를 선언해서 예외가 발생하면 메소드 밖으로 던지게 한다.
```java
// 1-2. JDBC를 이용한 등록과 조회 기능이 있는 UserDao 클래스
package springbook.user.dao;
...
public class UserDao {
  public void add(User user) throws ClassNotFoundException, SQLException {
    Class.forName("com.mysql.jdbc.Driver");
    Connection c = DriverManager.getConnection(
      "jdbc:mysql://localhost/springbook", "spring", "book");

    PreparedStatement ps = c.preparedStatement(
      "insert into users(id, name, password) values(?,?,?)");
    ps.setString(1, user.getId());
    ps.setString(2, user.getName());
    ps.setString(3, user.getPassword());

    ps.executeUpdate();

    ps.close();
    c.close();
  }

  public User get(String id) throws ClassNotFoundException, SQLException {
    Class.forName("com.mysql.jdbc.Driver");
    Connection c = DriverManager.getConnection(
      "jdbc:mysql://localhost/springbook", "spring", "book");

    PreparedStatement ps = c.preparedStatement(
      "select * from users where id = ?");

    ps.setString(1, id);

    ResultSet rs = ps.executeQuery();
    rs.next();
    User user = new User();
    user.setId(rs.getString("id"));
    user.setId(rs.getString("name"));
    user.setId(rs.getString("password"));

    rs.close();
    ps.close();
    c.close();

    return user;
  }
}
```

이 클래스가 제대로 동작하는지 어떻게 확인할 수 있을까? 웹 어플리케이션을 만들어 서버에 배치하고, 웹 브라우저를 통해 DAO 기능을 사용하기에는 배보다 배꼽이 더 크다.

### 1.1.3 main()을 이용한 DAO 테스트 코드
```java
// 1-3. 테스트용 main() 메서드
public static void main(String[] args) thorws ClassNotFoundException, SQLException {
  UserDao dao = new UserDao();

  User user = new User();
  user.setId("whiteship");
  user.setName("백기선");
  user.setPassword("married");

  dao.add(user);

  System.out.println(user.getId() + " 등록 성공");

  User user2 = dao.get(user.getId());
  System.out.println(user2.getName());
  System.out.println(user2.getPassword());

  System.out.println(user2.getId() + " 조회 성공");
}
```
main() 클래스를 실행하면 다음과 같은 테스트 성공 메시지를 얻을 수 있다.
```
whiteship 등록 성공
백기선
married
whiteship 조회 성공
```

그런데 UserDao 클래스 코드에는 사실 여러가지 문제가 있다. 정말 한심한 코드다.

## 1.2 DAO의 분리
### 1.2.1 관심사의 분리
