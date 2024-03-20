# 1장. 오브젝트와 의존관계
- 스프링의 핵심 철학 : 잃어버렸던 **객체지향** 기술의 가치를 회복시키고, 객체지향 프로그래밍이 제공하는 많은 혜택을 누릴 수 있게 기본으로 돌아가자는 것
- 따라서 스프링을 이해하려면 `오브젝트` 에 집중해야 한다
- 스프링은 **오브젝트를 어떻게 효과적으로 설계, 구현, 사용, 개선할 것인가에 대한 명쾌한 기준**을 마련해준다
- 스프링은 객체지향 기술과 설계, 구현에 대한 실용적인 전략, 검증된 best practice 를 자연스럽고 손쉽게 적용할 수 있도록 프레임워크 형태로 제공한다.

## 1.1. 초난감 DAO
- 자바빈(JavaBean)은 원래 비주얼 툴에서 조작 가능한 컴포넌트를 말한다. 
- 이제는 자바빈이라고 말하면 비주얼 컴포넌트라기보다는 다음 두 가지 관례를 따라 만들어진 오브젝트를 가리킨다. 
  - **디폴트 생성자** : 자바빈은 `파라미터가 없는 디폴트 생성자`를 갖고 있어야 한다. 툴이나 프레임워크에서 리플렉션을 이용해 오브젝트를 생성하기 때문에 필요하다.
  - **프로퍼티** : `자바빈이 노출하는 이름을 가진 속성`을 프로퍼티라고 한다. 프로퍼티는 set으로 시작하는 수정자 메소드(setter)와 get으로 시작하는 접근자 메소드(getter)를 이용해 수정 또는 조회할 수 있다.

## 1.2. DAO 의 분리
### 1.2.1. 관심사의 분리
- 개발자가 객체를 설계할 때 가장 염두에 둬야할 것은, **미래의 변화를 어떻게 대비할 것인가** 이다.
- 객체지향설계와 프로그래밍이 절차적 프로그래밍 패러다임에 비해 조금 더 많은 초기 비용을 소모되는데, 그 이유는 객체지향 기술 자체가 갖는 변화에 효과적으로 대처할 수 있다는 기술적인 특징 때문이다. 
- 변경이 일어날 때, 필요한 작업을 최소화하고 그 변경이 다른 곳에 문제를 일으키지 않게 하려면 **분리와 확장을 고려한 설계를 해야한다.**
- 변화는 대체로 집중된 `한 가지 관심` 에 대해 일어나지만, 그에 따른 작업은 한 곳에 집중되지 않는 경우가 많다. 
  - 관심이 같으면 하나의 객체 또는 친한 객체로 모이게 하고, 관심이 다르면 가능한 한 떨어지도록 분리한다

### 1.2.2. 커넥션 만들기의 추출

```java
public class UserDao {
    public void add(User user) throws ClassNotFoundException, SQLException {
		Class.forName("com.mysql.jdbc.Driver");
		Connection c = DriverManager.getConnection("jdbc:mysql://localhost/springbook?characterEncoding=UTF-8", "spring",
				"book");

		PreparedStatement ps = c.prepareStatement(
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
		Connection c = DriverManager.getConnection("jdbc:mysql://localhost/springbook?characterEncoding=UTF-8", "spring",
				"book");
		PreparedStatement ps = c
				.prepareStatement("select * from users where id = ?");
		ps.setString(1, id);

		ResultSet rs = ps.executeQuery();
		rs.next();
		User user = new User();
		user.setId(rs.getString("id"));
		user.setName(rs.getString("name"));
		user.setPassword(rs.getString("password"));

		rs.close();
		ps.close();
		c.close();

		return user;
	}
}
```

- UserDao 의 관심사항
  - DB 와의 연결을 위한 커넥션 설정
  - 파라미터로 넘어온 정보를 Statement 에 바인딩하여 실행시키기
  - Statement 와 Connectin 오브젝트 close

- 중복 코드의 메소드 추출
  - DB 연결코드를 getConnection() 이라는 독립 메소드로 만든다
  - `메소드 추출` : 공통의 기능을 담당하는 메소드로 중복된 코드를 뽑아내는 것 
    ```java
    private Connection getConnection() throws ClassNotFoundException, SQLException {
        Class.forName("com.mysql.jdbc.Driver");
        Connection c = DriverManager.getConnection("jdbc:mysql://localhost/springbook?characterEncoding=UTF-8", "spring","book");
        return c;
    }
    ```

### 1.2.3. DB 커넥션 만들기의 독립
- 다른 종류의 DB 를 사용하며, DB 커넥션을 가져오는 방법도 다를 경우
  - getConnection() 의 구현부를 제거하고 추상메소드로 만든다.
  - UserDao 도 추상클래스로 만들고, UserDao 클래스를 상속하여 서브클래스를 만든 후, 추상메소드로 선언했던 getConnection() 메소드를 구현한다. 
    ```java
    public abstract class UserDao {
        public void add(User user) throws ClassNotFoundException, SQLException {...}

        public User get(String id) throws ClassNotFoundException, SQLException {...}

        abstract protected Connection getConnection() throws ClassNotFoundException, SQLException;
    ```
    ```java
    public class NUserDao extends UserDao {
        protected Connection getConnection() throws ClassNotFoundException,
                SQLException {
            Class.forName("com.mysql.jdbc.Driver");
            Connection c = DriverManager.getConnection(
                    "jdbc:mysql://localhost/springbook?characterEncoding=UTF-8",
                    "spring", "book");
            return c;
        }
    }
    ```
- `템플릿 메소드 패턴` : 위와 같은 예시처럼 **슈퍼클래스에 기본 로직 흐름을 만들고, 기능의 일부를 추상메서드나 오버라이딩 가능한 protected 메소드로 만든 뒤, 서브클래스에서 이런 메소드를 필요에 맞게 구현해서 사용하도록 하는 방법**
  - 스프링에서 애용되는 디자인 패턴
- `팩토리 메소드 패턴` : 서브클래스에서 **구체적인 오브젝트 생성 방법을 결정하게 하는 것**
  - UserDao 의 서브클래스의 getConnection() 메소드 == 어떤 Connection 클래스의 오브젝트를, 어떻게 생성할 것인지를 결정하는 방법
- 결국, UserDao 는 Connection 오브젝트가 만들어지는 방법 + 내부 동작과는 상관없이, 자신이 필요한 기능을 Connection 인터페이스를 통해 사용하기만 할 뿐!
- 디자인 패턴 : 주로 객체지향 설계에 관한 것, 대부분 객체지향적 설계 원칙을 이용해서 문제를 해결한다
  - 패턴의 설계 구조는 대부분 비슷한데, 객체지향적 설계로부터 문제 해결을 위해 적용할 수 있는 확장성 추구 방법이 대부분 2가지로 정리되기 때문이다.
  - **상속과 오브젝트 합성!**
  - 핵심은 `각 패턴의 핵심이 담긴 목적 또는 의도` 이며, **패턴을 적용할 상황, 해결해야 할 문제, 솔루션 구조, 각 요소의 역할, 핵심의도가 무엇인가** 를 기억해둬야 한다.
- 템플릿 메소드 패턴
  - 상속을 통해 슈퍼클래스 기능 확장 시 사용하는 가장 대표적인 방법
  - 변하지 않는 기능은 슈퍼클래스에, 자주 변경되며 확장할 기능은 서브클래스에 둔다
  - `훅 메소드` : 슈퍼클래스에서 디폴트 기능을 정의해두거나 비워두었다가, 서브클래스에서 선택적으로 오버라이드할 수 있게 만들어둔 메소드
    - 서브클래스에서는 `추상 메소드를 구현` 하거나, `훅 메소드를 오버라이드` 하는 방법으로 기능의 일부를 확장한다
  