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
