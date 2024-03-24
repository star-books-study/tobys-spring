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
- 객체지향의 세계에서는 모든 것이 변한다(사용자의 비즈니스 프로세스와 그에 따른 요구사항, 기술, 운영환경 등).
- 그래서 개발자가 객체를 설계할 때 가장 염두에 둬야할 사항은 바로 미래의 변화를 어떻게 대비할 것인가다.
- 객체지향 설계와 프로그래밍이 절차적 프로그래밍 패러다임에 비해 초기에 좀 더 번거로운 작업을 요구하는 이유는 변화에 효과적으로 대처할 수 있다는 기술적인 특징 때문이다.
- **변경에 필요한 작업을 최소화하고, 그 변경이 다른 곳에 문제를 일으키지 않게 하기 위해서는 `분리와 확장`을 고려한 설계가 필요하다.**

- 먼저 분리에 대해서 생각해보자.
- 모든 변경과 발전은 한 번에 한 가지 관심사항에 집중해서 일어난다. 문제는, 그에 따른 작업은 한 곳에 집중되지 않는 경우가 많다는 점이다.
- 프로그래밍의 기초 개념 중 하나인 `관심사의 분리`를 객체지향에 적용해보면 다음과 같다.
  - 관심이 같은 것끼리는 하나의 객체 안으로 또는 친한 객체로 모이게 한다.
  - 관심이 다른 것은 가능한 따로 떨어져서 서로 영향을 주지 않도록 분리한다.
 

### 1.2.2 커넥션 만들기의 추출
UserDao에 구현된 add() 메소드에서 적어도 세 가지의 관심사항을 발견할 수 있다.
#### UserDao의 관심사항
1. DB와 연결을 위한 커넥션 가져오기
2. 사용자 등록을 위해 DB에 보낼 SQL 문장을 담을 Statement를 만들고 실행하기
3. 작업이 끝나면 사용한 리소스인 Statement와 Connection 오브젝트를 닫아주기

#### 중복 코드의 메서드 추출
커넥션을 가져오는 중복된 DB 연결 코드를 getConnection()이라는 이름의 독립적인 메서드로 만든다.
```java
// 1-4. getConnection() 메서드를 추출해서 중복을 제거한 UserDao
public void add(User user) throws ClassNotFoundException, SQLException {
  Connection c = getConnection();
  ...
}

public User get(String id) throws ClassNotFoundException, SQLException {
  Connection = getConnection();
  ...
}

private Connection getConnection() throws ClassNotFoundException, SQLException {
  Class.forName("com.mysql.jdbc.Driver");
  Connection c = DriverManager.getConnection(
    "jdbc:mysql://localhost/springbook", "spring", "book");
  return c;
}
```
- 관심의 종류에 따라 코드를 구분해놓았기 때문에 한 가지 관심에 대한 변경이 일어날 경우 그 관심이 집중되는 부분의 코드만 수정하면 된다.

#### 변경사항에 대한 검증 : 리팩토링과 테스트
- main() 메서드를 여러 번 실행하면 id 값이 중복되기 때문에 두 번째부터는 무조건 예외가 발생한다.
  => main() 메서드 재실행 전 User 테이블의 사용자 정보를 모두 삭제해주어야 한다.
- 기능에는 영향을 주지 않으면서 코드의 구조만 변경하는 작업을 `리팩토링`이라고 한다
- 공통의 기능을 담당하는 메서드로 중복된 코드를 뽑아내는 것을 `메소드 추출(extract method)`기법이라고 한다.

### 1.2.3 DB 커넥션 만들기의 독립
- 만약 발전을 거듭한 UserDao가 인기를 끌어 N사와 D사에서 UserDao를 구매하겠다는 주문이 들어왔다고 하자.
- 문제
  - N 사와 D 사가 각기 다른 종류의 DB를 사용하고 DB 커넥션을 가져오는 데 있어 독자적으로 만든 방법을 적용하고 싶어한다.
  - 고객에게 소스를 직접 공개하고 싶지는 않다. 미리 컴파일된 클래스 바이너리 파일만 제공하고 싶다.
- 어떻게 소스코드를 N사와 D사에 제공해주지 않고도 고객 스스로 원하는 DB 커넥션 생성 방식을 적용해가면서 UserDao를 사용하게 할 수 있을까?

#### 상속을 통한 확장
- getConnection()을 추상 메서드로 만들고 추상 클래스인 UserDao를 N사와 D사에 판매한다.
- UserDao 클래스를 상속해서 NUserDao와 DUserDao라는 서브 클래스를 만든다.
- 기존에는 같은 클래스에서 다른 메서드로 분리됐던 DB 커넥션 연결이라는 관심을 **상속을 통해 서브클래스로 분리**해버리는 것이다.
<img width="533" alt="스크린샷 2024-03-23 오후 8 11 35" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/51d8038f-6dcc-4756-898d-c0b4bb134e6c">

```java
// 1-5. 상속을 통한 확장 방법이 제공되는 UserDao
public abstract class UserDao {
  public void add(User user) throws ClassNotFoundException, SQLException {
    Connection c = getConnection();
    ...
  }
  
  public User get(String id) throws ClassNotFoundException, SQLException {
    Connection = getConnection();
    ...
  }
  
  private abstract Connection getConnection() throws ClassNotFoundException, SQLException;
}

public class NUserDao extends UserDao {
  public Connection getConnection() throws ClassNotFoundException, SQLException {
    // N사 DB Connection 생성 코드
  }
}

public class DUserDao extends UserDao {
  public Connection getConnection() throws ClassNotFoundException, SQLException {
    // D사 DB Connection 생성 코드
  }
}
```
- UserDao가 담당하는 기능 : DAO의 핵심 기능인 어떻게 데이터를 등록하고 가져올 것인가라는 관심을 담당
- NUserDao, DUserDao가 담당하는 기능 : DB 연결 방법은 어떻게 할 것인가라는 관심을 담당
- 이렇게 슈퍼클래스에 기본적인 로직의 흐름을 만들고, 그 기능의 일부를 추상 메서드나 오버라이딩이 가능한 `protected` 메서드 등으로 만든 뒤 서브클래스에서 이런 메서드를 필요에 맞게 구현해서 사용하도록 하는 방법을 `템플릿 메서드 패턴(template method pattern)`이라고 한다.
- UserDao의 서브클래스의 getConnection() 메서드처럼 서브 클래스에서 구체적인 오브젝트 생성 방법을 결정하게 하는 것을 `팩토리 메서드 페턴(factory method pattern)`이라고 부르기도 한다.
  
  <img width="605" alt="스크린샷 2024-03-24 오전 10 25 47" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/aa2eba6e-4164-40c4-8b9e-577ec847a068">
  UserDao에 적용된 팩토리 

> **템플릿 메서드 패턴**
>
> 변하지 않는 기능은 슈퍼 클래스에 만들어두고 자주 변경되어 확장할 기능은 서브 클래스에서 만들도록 한다.
> - 슈퍼 클래스에서는 미리 추상 메서드 또는 오버라이드 가능한 메서드를 정의해두고 이를 활용해 코드의 기본 알고리즘을 담고 있는 템플릿 메서드를 만든다.
> - 슈퍼 클래스에서 디폴트 기능을 정의해두거나 비워뒀다가 서브클래스에서 선택적으로 오버라이드할 수 있도록 만들어둔 메서드를 `훅(hook) 메서드`라고 한다.
> - 서브 클래스에서는 추상 메서드를 구현하거나, 훅 메서드를 오버라이딩하는 방법을 활용해 기능의 일부를 확장한다.
> ```java
> public abstract class Super {
>   public void templateMethod() {
>     // 기본 알고리즘 코드
>     hookMethod();
>     abstractMethod();
>     ...
>   }
>   protected void hookMethod() { } // 선택적으로 오버라이드 가능한 훅 메서드
>   public abstract void abstractMethod(); // 서브 클래스에서 반드시 구현해야 하는 추상 메서드
> }
>
> public class Sub1 extends Super {
>   protected void hookMethod() {
>     ...
>   }
>   public void abstractMethod() {
>     ...
>   }
> }


> **팩토리 메서드 패턴**
>
> - 슈퍼 클래스 코드에서는 서브 클래스에서 구현할 메서드를 호출해서 필요한 타입의 오브젝트를 가져와 사용한다.
> - 이 메서드는 주로 인터페이스 타입으로 오브젝트를 리턴하므로 서브 클래스에서 정확히 어떤 클래스의 오브젝트를 만들어 리턴할지는 슈퍼 클래스에서는 알지 못한다.

- 하지만 상속을 이용한 방법은 많은 한계점이 있다.
  - 자바는 클래스의 다중상속을 허용하지 않는다. 단지, 커넥션 객체를 가져오는 방법을 분리하기 위해 상속 구조로 만들어버리면, 후에 다른 목적으로 UserDao에 상속을 적용하기 힘들다.
  - 상속을 통한 상하위 클래스의 관계는 생각보다 밀접하다. 슈퍼 클래스 내부의 변경이 있을 때 모든 서브클래스를 함께 수정하거나 다시 개발해야할 수도 있다.
  - 확장된 기능인 DB 커넥션을 생성하는 코드를 다른 DAO 클래스에 적용할 수 없다. 만약 UserDao 외의 DAO 클래스들이 계속 만들어진다면 상속을 통해 만들어진 getConnection() 구현 코드가 매 DAO 클래스마다 중복돼서 심각한 문제가 발생할 것이다.

## 1.3 DAO의 확장
- 추상 클래스를 만들고 이를 상속한 서브 클래스에서 변화가 필요한 부분을 바꿔서 쓸 수 있게 만든 이유는 이렇게 변화의 성격이 다른 것을 분리해서, 서로 영향을 주지 않은 채로 각각 필요한 시점에 독립적으로 변경할 수 있게 하기 위해서다.
- 그러나 여러가지 단점이 많은, 상속이라는 방법을 사용했다는 사실이 불편하게 느껴진다.
### 1.3.1 클래스의 분리
- 이번에는 아예 상속관계도 아닌 완전히 독립적인 클래스로 만들어보겠다.
- DB 커넥션과 관련된 부분을 서브클래스가 아니라, 아예 별도의 클래스에 담는다. 그리고 이렇게 만든 클래스를 UserDao가 이용하게 하면 된다.
  <img width="587" alt="스크린샷 2024-03-24 오전 11 37 14" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/aabb5f20-1a47-476c-8466-e6a8971aa57f">
```java
// 1-6. 독립된 SimpleConnectioinMaker를 사용하게 만든 UserDao
public class UserDao {
  private SimpleConnectionMaker simpleConnectionMaker;

  public UserDao() {
    // 상태를 관리하는 것도 아니니 한 번만 만들어 인스턴스 변수에 저장해두고 메서드에서 사용하게 한다.
    simpleConnectionMaker = new SimpleConnectionMaker();
  }

  public void add(User user) throws ClassNotFoundException, SQLException {
    Connection c = simpleConnectionMaker.makeNewConnection();
    ...
  }

  public User get(String id) throws ClassNotFoundException, SQLException {
    Connection c = simpleConnectionMaker.makeNewConnection();
    ...
  }
}
```
DB 커넥션 생성 기능을 독립시킨 SimpleConnectionMaker는 리스트 1-7과 같이 만든다.
```java
// 1-7. 독립시킨 DB 연결 기능인 SimpleConnectionMaker
package springbook.user.dao;
...
public class SimpleConnectionMaker {
  public Connection makeNewConnection() throws ClassNotFoundException, SQLException {
    Class.forName("com.mysql.jdbc.Driver");
    Connection c = DriverManager.getConnection(
      "jdbc:mysql://localhost/springbook", "spring", "book");
    return c;
  }
}
```
- 이번에는 다른 문제가 발생했다. N 사와 D 사에 UserDao 클래스만 공급하고 상속을 통해 DB 커넥션 기능을 확장해서 사용하게 했던 게 다시 불가능해졌다.
  - `UserDao`의 코드가 `SimpleConnectionMaker`라는 특정 클래스에 종속되어 있기 때문에 상속을 사용했을 때처럼 UserDao 코드의 수정 없이 DB 커넥션 생성 기능을 변경할 방법이 없다.
- 다른 방식으로 DB 커넥션을 제공하는 클래스를 사용하기 위해서는 UserDao 소스코드의 다음 줄을 직접 수정해야 한다.
  ```java
  simpleConnectionMaker = new SimpleConnectionMaker();
  ```
- 이렇게 클래스를 분리한 경우에도 상속을 이용했을 때와 마찬가지로 자유로운 확장이 가능하게 하려면 두 가지 문제를 해결해야 한다.
  1. SimpleConnectionMaker의 메서드 문제
     만약 D 사에서 만든 DB 커넥션 제공 클래스는 openConnection()이라는 메서드 이름을 사용했다면 UserDao에 있는 add(), get() 메서드의 커넥션을 가져오는 코드를 다음과 같이 일일히 변경해야 한다.
     ```java
     Connection c = simpleConnectionMaker.openConnection();
     ```
 2. DB 커넥션을 제공하는 클래스가 어떤 것인지 UserDao가 구체적으로 알고 있어야 한다
    UserDao에서 SimpleConnectionMaker라는 클래스 타입의 인스턴스 변수까지 정의해놓고 있으니, N 사에서 다른 클래스를 구현하면 어쩔 수 없이 UserDao 자체를 다시 수정해야 한다.

- 이런 문제의 근본적인 원인은 UserDao가 바뀔 수 있는 정보, 즉 DB 커넥션을 가져오는 클래스에 대해 너무 많이 알고 있기 때문이다.

### 1.3.2 인터페이스의 도입
- 이런 문제를 해결할 가장 좋은 해결책은 두 개의 클래스가 서로 긴밀하게 연결되어 있지 않도록 중간에 추상적인 느슨한 연결고리를 만들어주는 것이다.
- 인터페이스를 도입하면 UserDao는 자신이 사용할 클래스가 어떤 것인지 몰라도 된다. 단지 인터페이스를 통해 원하는 기능을 사용하면 된다.
  <img width="552" alt="스크린샷 2024-03-24 오전 11 59 50" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/74273027-5137-486a-b1f6-61cac910bf92">

```java
// 1-8. ConnectionMaker 인터페이스
package springbook.user.dao;
...
public interface ConnectionMaker {
  public Connection makeConnection() throws ClassNotFoundException, SQLException;
}
```
- 고객에게 납품을 할 떄는 UserDao 클래스와 함께 ConnectionMaker 인터페이스도 전달한다.
- 그리고 D 사의 개발자라면 다음과 같이 ConnectionMaker를 구현한 클래스를 만들고, 자신들의 DB 연결 기술을 이용해 DB 커넥션을 가져오도록 메서드를 작성해주면 된다.
```java
// 1-9. ConnectionMaker 구현 클래스
package springbook.user.dao;
...
public class DConnectionMaker implements ConnectionMaker {
  ...
  public Connection makeConnection() throws ClassNotFoundException, SQLException {
    // D 사의 독자적인 방법으로 Connection을 생성하는 코드
  }
}
```
- 리스트 1-10은 특정 클래스 대신 인터페이스를 사용해서 DB 커넥션을 가져와 사용하도록 수정한 UserDao 코드다.
```java
// 1-10. ConnectionMaker 인터페이스를 사용하도록 개선한 UserDao
public class UserDao {

  // 인터페이스를 통해 오브젝트에 접근하므로 구체적인 클래스 정보를 알 필요가 없다.
  private ConnectionMaker connectionMaker;

  public UserDao() {
    // 앗! 그런데 여기에는 클래스 이름이 나오네!
    connectionMaker = new DConnectionMaker();
  }

  public void add(User user) throws ClassNotFoundException, SQLException {
    // 인터페이스에 정의된 메서드를 사용하므로 클래스가 바뀐다고 해도 메서드 이름이 변경될 걱정은 없다.
    Connection c = connectionMaker.makeConnection();
    ...
  }

  public User get(String id) throws ClassNotFoundException, SQLException {
    Connection c = connectionMaker.makeConnection();
    ...
  }
}
```
- DB 커넥션을 제공하는 클래스에 대한 구체적인 정보는 모두 제거가 가능했지만, 초기에 한 번 어떤 클래스의 오브젝트를 사용할지 결정하는 생성자의 코드는 제거되지 않고 남아있다.

### 1.3.3 관계설정 책임의 분리
- 인터페이스를 이용한 분리에도 불구하고 여전히 UserDao 변경 없이는 DB 커넥션 기능의 확장이 자유롭지 못하다.
- 그 이유는 UserDao 안에 분리되지 않은, **또 다른 관심사항이 존재하고 있기 때문이다.**
- `new DConnectionMaker()`라는 코드는 그 자체로 충분히 독립적인 관심사를 담고 있다.
  - UserDao가 어떤 ConnectionMaker 구현 클래스의 오브젝트를 이용하게 할지를 결정하는 것이다.
  - UserDao와 UserDao가 사용할 ConnectionMaker의 특정 구현 클래스 사이의 관계를 설정해주는 것에 관한 관심이다.

- 두 개의 오브젝트가 있고 한 오브젝트가 다른 오브젝트의 기능을 사용한다면, 사용되는 오브젝트를 서비스, 사용하는 오브젝트를 클라이언트라고 부를 수 있다. (UserDao - 서비스 / UserDao를 사용하는 오브젝트 - 클라이언트)
- 바로 이 UserDao의 클라이언트 오브젝트가 제3의 관심사항인 UserDao와 ConnectionMaker 구현 클래스의 관계를 결정해주는 기능을 분리해서 두기에 적절한 곳이다.

- 먼저 UserDao가 어떤 ConnectionMaker의 구현 클래스를 사용할지 결정하도록 만들어보자. 즉 UserDao 오브젝트와 특정 클래스로부터 만들어진 ConnectionMaker 오브젝트 사이에 관계를 설정해주는 것이다.
- 오브젝트 사이의 관계는 런타임시에 한쪽이 다른 오브젝트의 레퍼런스를 갖고 있는 방식으로 만들어진다.
  - 이는 클래스 사이의 관계를 설정해주는 건 아니다. 클래스 사이의 관계가 만들어진다는 것은 한 클래스가 인터페이스 없이 다른 클래스를 직접 사용한다는 뜻이다.
  - 예를 들면, 다음 코드는 DConnectionMaker의 오브젝트 레퍼런스를 UserDao의 connectionMaker 변수에 넣어서 사용하게 함으로써, 두 개의 오브젝트가 '사용'이라는 관계를 맺게 해준다.
  ```connectionMaker = new DConnectionMaker();
  ```
- 오브젝트 사이의 관계가 만들어지려면 일단 오브젝트가 있어야 하직접 생성자를 호출해서 직접 오브젝트를 만드는 방법도 있지만 외부에서 준 것을 가져오는 방법도 있다.
-   외부에서 만든 오브젝트를 전달받으려면 메서드 파라미터나 생성자 파라미터를 이용하면 된다.
- UserDao의 모든 코드는 ConnectionMaker 인터페이스 외에는 어떤 클래스와도 관계를 가져서는 안되게 해야 한다.
- UserDao 오브젝트가 DConnectionManager 오브젝트르 사용하게 하려면 두 클래스의 오브젝트 사이에 런타임 사용관계, 링크, 또는 의존관계라고 불리는 관계를 맺어주면 된다. 아래 그림은 런타임 시점의 오브젝트 간 관계를 나타내는 오브젝트 다이어그램이다. 모델링 싱[는 없었던, 그래서 코드에는 보이지 않던 관계가 오브젝트로 만들어진 후에 생성되는 것이다.
  <img width="415" alt="스크린샷 2024-03-24 오후 12 40 40" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/d76f06a2-d5e5-4715-bc91-adb05685f468">

- UserDao의 클라이언트의 책임은 아래와 같은 구조의 클래스들을 이용해 위와 같이 런타임 오브젝트 관계를 갖는 구조로 만들어주는 것이다.
  <img width="552" alt="스크린샷 2024-03-24 오전 11 59 50" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/74273027-5137-486a-b1f6-61cac910bf92">
- 책임을 떠넘겨보자.
  - 현재는 UserDao()의 main() 메서드가 UserDao의 클라이언트라고 볼 수 있다. 아예 UserDaoTest라는 이름의 클래스를 만들어 main() 메서드를 UserDaoTest로 옮겨보자.
  - 그리고 리스트 1-11과 같이 생성자를 수정해주면 UserDao에는 DConnectionMaker가 사라진다.
    ```java
    // 1-11. 수정한 생성자
    public UserDao(ConnectionMaker connectionMaker) {
      this.connectionMaker = connectionMaker;
    }
    ```
    - UserDaoTest는 UserDao와 ConnectionMaker 구현 클래스와의 런타임 오브젝트 관계를 설정하는 책임을 담당한다.
    ```java
    public class UserDaoTest {
      public static void main(String[] args) throws ClassNotFoundExceptoin, SQLException {
      ConnectionMaker connectionMaker = new DConnectionMaker();

      UserDao dao = new UserDao(connectionMaker);
      ...
      }
    }
- 인터페이스를 도입하는 방법은 다른 DAO 클래스에서도 ConnectionMaker의 구현 클래스들은 그대로 적용할 수 있기 때문에 상속에 비해 훨씬 더 유연한다.
- 아래 그림은 UserDao가 사용할 ConnectionMaker 클래스를 선정하는 책임을 UserDaoTest가 담당하고 있는 구조를 보여준다.
  <img width="541" alt="스크린샷 2024-03-24 오후 12 58 45" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/e6fe221f-a7d9-4081-a5f4-7aa2bc518e28">

### 1.3.4 원칙과 패턴
지금까지 초난감 DAO 코드를 개선해온 결과를 객체지향 기술의 여러가지 이론을 통해 설명하려고 한다.
#### 개방 폐쇄 원칙
깔끔한 설계를 위해 적용 가능한 객체지향 설계 원칙 중 하나다.
> '클래스의 모듈은 확장에는 열려 있어야 하고 변경에는 닫혀 있어야 한다'
- UserDao는 DB 연결 방법이라는 기능을 확장하는 데는 열려 있다. UserDao에 전혀 영향을 주지 않고도 얼마든지 기능 확장 가능하다.
- UserDao 자신의 핵심 기능을 구현한 코드는 그에 변화에 받지 않고 유지할 수 있으므로 변경에는 닫혀 있다.

#### 높은 응집도와 낮은 결합도
개방 폐쇄 원칙은 높은 응집도와 낮은 결합도(high coherence and low coupling)라는 소프트웨어 개발의 고전적인 원리로도 설명이 가능하다.
- 높은 응집도
  - 응집도가 높다는 것은 변화가 일어날 때 해당 모듈에서 변하는 부분이 크다는 것으로 설명할 수 있다.
  - 즉, 변경이 일어날 때 모듈의 많은 부분이 함께 바뀐다면 응집도가 높다고 말할 수 있다.
- 낮은 결합도
  - 높은 응집도보다 더 민감한 원칙
  - 낮은 결합도, 즉 느슨한 연결은 관계를 유지하는 데 꼭 필요한 최소한의 방법만 간접적인 형태로 제공하고, 나머지는 서로 독립적이고 알 필요도 없게 만들어주는 것이다.
  - 결합도가 낮아지면 변화에 대응하는 속도도 높아지고, 구성이 깔끔해진다. 또한 확장하기에도 매우 편리하다.
  - 여기서 결합도란 '하나의 오브젝트가 변경이 일어날 때에 관계를 맺고 있는 다른 오브젝트에게 변화를 요구하는 정도'라고 설명할 수 있다.

#### 전략 패턴
- 개선한 UserDaoTest-UserDao-ConnectionMaker 구조를 디자인 패턴의 시각으로 보면 전략 패턴(Strategy Pattern)에 해당한다고 볼 수 있다.
- 자신의 기능 맥락(context)에서, 필요에 따라 변경이 필요한 알고리즘을 인터페이스를 통해 통째로 외부로 분리시키고, 이를 구현한 구체적인 알고리즘 클래스를 바꿔서 사용할 수 있게 하는 디자인 패턴이다.
- UserDao는 전략 패턴의 컨텍스트에 해당한다.
  - 컨텍스트는 자신의 기능을 수행하는 데 필요한 기능 중에서 변경 가능한, DB 연결 방식이라는 알고리즘을 ConnectionMaker라는 인터페이스로 정의하고, 이를 구현한 클래스, 즉 번략을 바꿔가면서 사용할 수 있게 분리했다.
 
## 1.4 제어의 역전

### 1.4.1 오브젝트 팩토리
- 현재 UserDaoTest는 UserDao의 기능이 잘 작동하는지 테스트하려고 만들었지만 엉겹결에 어떤 ConnectionMaker 구현 클래스를 사용할지 결정하는 기능도 떠맡게 되었다.
- 관심사를 분리하자.
  - UserDao와 ConnectionMaker 구현 클래스의 오브젝트를 만드는 역할
  - 그렇게 만들어진 두 개의 오브젝트가 연결돼서 사용할 수 있도록 관계를 맺어주는 역할
#### 팩토리
- 객체의 생성 방법을 결정하고 그렇게 만들어진 오브젝트를 오브젝트를 돌려주는 클래스를 만들어보겠다.
- 이런 일을 하는 오브젝트를 `팩토리(factory)`라고 부른다.
```java
// 1-14. UserDao의 생성 책임을 맡은 팩토리 클래스
package springbook.user.dao;
...
public class DaoFactory {
  public UserDao userDao() {
    ConnectionMaker connectionMaker = new DConnectionMaker();
    UserDao userDao = new UserDao(connectionMaker);
    return userDao;
  }
}
```
```java
// 1-15. 팩토리를 사용하도록 수정한 UserDaoTest
public class UserDaoTest {
  public static void main(String[] args) throws ClassNotFoundException, SQLException {
    UserDao dao = new DaoFactory.userDao();
    ...
  }
}
```

#### 설계도로서의 팩토리
- UserDao와 ConnetionMaker가 실질적인 로직을 담당하는 컴포넌트라면, DaoFactory는 컴포넌트의 구조와 관계를 정의한 설계도와 같은 역할을 한다.
- 이런 작업이 애플리케이션 전체에 걸쳐서 일어난다면 컴포넌트의 의존관계에 대한 설계도와 같은 역할을 하게될 것이다.
  - 그림 첨부
- 이제 N사와 D사에 UserDao를 공급할 때, DaoFactory도 같이 제공하되, ConnetionMaker의 구현 클래스를 변경할 수 있도록 소스코드 형태로 제공한다. 
- 팩토리를 통해 애플리케이션의 컴포넌트 역할을 하는 오브젝트와 애플리케이션의 구조를 결정하는 오브젝트를 분리할 수 있다.

### 1.4.2 오브젝트 팩토리의 이용
- DaoFactory에 UserDao가 아닌 다른 DAO의 생성 기능을 넣게 되면 ConnectionMaker 구현 클래스의 오브젝트를 선정하고 생성하는 코드가 메서드마다 반복된다.
```java
// 1-16 DAO 생성 메서드 추가로 인해 발생하는 중복
public class DaoFactory {
  public UserDao userDao() {
    return new UserDao(new DConnectionMaker());
  }

  public AccountDao accountDao() {
    return new AccountDao(new DConnectionMaker());
  }

  public MessageDao messageDao() {
    return new MessageDao(new DConnectionMaker());
  }
}
```

- ConnectionMaker의 구현 클래스를 결정하고 오브젝트를 만드는 코드를 별도의 메서드 뽑아내자.
반복된다.
```java
// 1-17. 생성 오브젝트 코드 수정
public class DaoFactory {
  public UserDao userDao() {
    return new UserDao(new ConnectionMaker());
  }

  public AccountDao accountDao() {
    return new AccountDao(new ConnectionMaker());
  }

  public MessageDao messageDao() {
    return new MessageDao(new ConnectionMaker());
  }
  
  public ConnectionMaker connectionMaker() {
    // 분리해서 중복을 제거한 ConnectionMaker 타입 오브젝트 생성 코드
    return new DConnectionMaker();
  }
}
```

### 제어권 이전을 통한 제어관계 역전
- 제어의 역전이란, 간단히 프로그램의 제어 구조가 뒤바뀌는 것
- 제어의 역전에서는 오브젝트가 자신이 사용할 오브젝트를 스스로 선택하지도 생성하지도 않는다. 또 자신이 어떻게 만들어지고 어디서 사용되는지를 알 수 없다.
- 모든 제어 권한을 자신이 아닌 다른 대상에게 위임하기 때문이다.
- main()과 같은 엔트리 포인트를 제외하면 모든 오브젝트는 이렇게 위임받은 제어권한을 갖는 특별한 오브젝트에 의해 결정되고 만들어진다.
- 서블릿이나 JSP, EJB처럼 컨테이너 안에서 동작하는 구조는 간단한 방식이긴 하지만 제어의 역전 개념이 적용되어 있다.
- 템플릿 메서드 패턴에서도, 추상 메서드를 상속한 서브 클래스는 자신의 메서드가 언제 어떻게 사용될지는 모른다. 
  - 즉, 제어권을 상위 템플릿 메서드에 넘기고 자신은 필요할 때 호출되어 사용하게 하도록 한다는 제어의 역전의 개념을 발견할 수 있다.
- 프레임워크도 제어의 역전이 적용된 대표적인 기술이다. 
- 자연스럽게 관심을 분리하고 책임을 나누고 유연하게 확장 가능한 구조로 만들기 위해 DaoFactory를 도입했던 과정이 IoC를 적용하는 과정이었다.
- IoC를 애플리케이션 전반에 걸쳐 본격적으로 사용하려면 스프링 같은 IoC 프레임워크의 도움을 받는 편이 유리하다.
- 스프링은 IoC를 모든 기능의 기초가 되는 기반기술로 삼고 있으며, IoC를 극한으로 적용하고 있는 프레임워크다.

## 1.5 스프링의 IoC
- 스프링의 핵심을 담당하는 건 바로 빈 팩토리 또는 애플리케이션 컨텍스트라고 불리는 것이다.
- 이 두가지는 DaoFactory가 하는 일을 좀 더 일반화한 것이라 설명할 수 있다.
### 1.5.1 오브젝트 팩토리를 이용한 스프링 IoC
#### 애플리케이션 컨텍스트와 설정 정보
- 스프링에서는 스프링이 제어권을 가지고 직접 만들고 관계를 부여하는 오브젝트를 빈(bean)이라고 부른다.
  - 자바빈 또는 엔터프라이즈 자바빈(EJB)에서 말하는 빈과 비슷한 오브젝트 단위의 애플리케이션 컴포넌트
  - 스프링 컨테이너가 생성과 관계 설정, 사용 등을 제어해주는 제어의 역전이 적용된 오브젝트를 가리킨다.
- 스프링애서는 빈의 설정과 관계 설정 같은 제어를 담당하는 IoC 오브젝트를 빈 팩토리(bean factory)라고 부른다.
- 보통 빈 팩토라보다는 이를 확장한 애플리케이션 컨텍스트(application context)를 더 많이 사용한다.
  - IoC 방식을 따라 만들어진 일종의 빈 팩토리
  - 별도의 정보를 참고해서 빈(오브젝트)의 생성, 관계 설정 등의 제어 작업을 총괄한다.
-  
