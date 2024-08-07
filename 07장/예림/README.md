# 7장. 스프링 핵심 기술의 응용
- 스프링의 모든 기술은 결국 객체지향적인 언어의 장점을 적극적으로 활용해서 코드를 작성하도록 도와주는 것이다.
- 7장에서는 지금까지 살펴봤던 IoC/DI, 서비스 추상화, AOP 기술을 애플리케이션 개발에 활용해서 새로운 기능을 만들어보고 이를 통해 스프링의 개발 철학과 추구하는 가치, 스프링 사용자에게 요구되는 게 무엇인지를 살펴보겠다.


## 7.1 SQL과 DAO의 분리

- SQL을 DAO에서 분리하는 작업을 해보자.
- 데이터 액세스 로직은 바뀌지 않더라도 DB의 테이블, 필드 이름과 SQL 문장이 바뀔 수 있다.
- 그때마다 DAO 코드를 수정하고 이를 다시 컴파일해서 적용하는 건 번거로울 뿐만 아니라 위험하기도 하다.

### 7.1.1 XML 설정을 이용한 분리
- SQL은 문자열로 되어 있으니 설정 파일에 프로퍼티 값으로 정의해서 DAO에 주입해줄 수 있다.

#### 개별 SQL 프로퍼티 방식
- UserDao의 JDBC 구현 클래스인 UserDaoJdbc의 SQL 6개를 프로퍼티로 만들고 이를 XML에서 지정하도록 해보자.
- add() 메서드의 SQL을 외부로 빼는 작업을 살펴보자. 먼저 7-1과 같이 add() 메서드에서 사용할 SQL을 **프로퍼티로 정의**한다.

```java
// 7-1. add() 메서드를 위한 SQL 필드
public class UserDaoJdbc implements UserDao {
  private String sqlAdd;

  public void setSqlAdd(String sqlAdd) {
    this.sqlAdd = sqlAdd;
  }
```

- 7-2와 같이 add() 메서드의 SQL 문장을 제거하고 외부로부터 DI 받은 SQL 문장을 담은 sqlAdd를 사용하게 만든다.
```java
// 7-2. 주입받은 SQL 사용
public void add(User user) {
  this.jdbcTemplate.update(
    this.sqlAdd, // "insert into users..."를 제거하고 외부에서 주입받은 SQL 사용
    this.getId(), user.getName, ... );
}
```
- 다음은 XML 설정의 userDao 빈에 sqlAdd 프로퍼티를 추가하고 SQL을 넣어준다.
```xml
// 7-3. 설정 파일에 넣은 SQL 문장
<bean id="userDao" class="springbook.user.dao.UserDaoJdbc">
  <property name="dataSource" ref="dataSource" />
  <property name="sqlAdd" value="insert into users(id, name, password, email, level, login, recommend) values(?,?,?,?,?,?,?)" />
  ...
```
