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
- 하지만 이 방법은 새로운 SQL이 필요할 때마다 프로퍼티를 추가하고 DI를 위한 변수와 수정자 메서드도 만들어줘야 한다는 점에서 조금 불편해 보인다.

#### SQL 맵 프로퍼티 방식
- 이번에는 SQL을 하나의 컬렉션으로 담아두는 방법을 시도해보자. 맵을 이용하면 키 값을 이용해 SQL 문장을 가져올 수 있다.
```java
// 7-4. 맵 타입의 SQL 정보 프로퍼티
public class UserDaoJdbc implements UserDao {
  ...
  private Map<String, String> sqlMap;

  public void setSqlMap(Map<String, String> sqlMap) {
    this.sqlMap = sqlMap;
  }
```
- 각 메서드는 미리 정해진 키 값을 이용해 sqlMap으로부터 SQL을 가져와 사용하도록 만든다.
```java
// 7-5. sqlMap을 사용하도록 수정한 add()
public void add(User user) {
  this.jdbcTemplate.update(
    this.sqlMap.get("add"), // 프로퍼티로 제공받은 맵으로부터 키를 이용해서 필요한 SQL을 가져온다.
    user.getId(), user.getName(), user.getPassword(), ... );
}
```
- 이제 XML 설정을 수정하자.
- Map은 스프링이 제공하는 `<map>` 태그를 사용해야 한다.
- 맵을 초기화해서 sqlMap 프로퍼티에 넣으려면 다음과 같이 `<map>`과 `<entry>` 태그를 `<property>` 태그 내부에 넣어주면 된다.
```xml
// 7-6. 맵을 이용한 SQL 설정
<bean id="userDao" class="springbook.user.dao.UserDaoJdbc">
  <property name="dataSource" ref="dataSource" />
  <property name="sqlMap">
    <map>
      <entry key="add" value="insert into users(id, name, password, email, level, login, recommend) values(?,?,?,?,?,?,?)" />
      <entry key="get" value="select * from users where id = ?" />
      <entry key="getAll" value="select * from users order by id = ?" />
      ...
    </map>
  </proprty>
</bean>
```
- 맵으로 만들어두면 새로운 SQL이 필요할 때 `<entry>`만 추가하면 된다.
- 대신 SQL을 가져올 때 문자열로 된 키 값을 가져오기 때문에 오타가 있어도 오류를 확인하기 힘들다는 단점이 있다.
- 따라서 포괄적인 테스트를 만들어서 모든 SQL을 바르게 가져오는지 미리미리 검증해볼 필요가 있다.

### 7.1.2 SQL 제공 서비스
- SQL과 DI 설정 정보가 섞여있으면 지저분하고 관리하기에도 좋지 않다. ➡️ SQL을 따로 분리하자.
- SQL을 꼭 스프링 빈 설정 방법을 사용해 XML에 담아둘 필요도 없다.
- 스프릥의 설정파일로부터 생성된 오브젝트와 정보는 애플리케이션을 다시 시작하기 전에는 변경이 매우 어렵다는 점도 문제다.
- 이런 문제점을 해결하려면 DAO가 사용할 SQL을 제공해주는 기능을 독립시킬 필요가 있다. 독립적인 SQL 제공 서비스가 필요하다는 것이다.

#### SQL 서비스 인터페이스
- SQL 서비스의 인터페이스를 설계해보자.
- DAO가 사용할 서비스의 기능은 간단하다. SQL에 대한 키 값을 전달하면 그에 해당하는 SQL을 돌려주는 것이다.
```java
// 7-7. SqlService 인터페이스
public interface SqlService {
  String getSql(String key) thorws SqlRetrievalFailureException; // 런타임 예외이므로 특별히 복구해야 할 필요가 없다면 무시해도 된다.
}
```
- SQL을 가져오다가 어떤 이유에서든 실패하는 경우에는 SqlRetrievalFailureException 예외를 던지도록 정의한다.
- 7-8과 같이 메시지와 원인이 되는 예외를 담을 수 있는 SqlRetrievalFailureException 클래스를 정의하자.
```java
// 7-8. SQL 조회 실패 시 예외
public class SqlRetrievalFailureException extends RuntimeException {
  public SqlRetrievalFailureException(String message) {
    super(message);
  }

  public SqlRetrievalFailureException(String message, Throwable cause) {
    super(message, cause);
  }
}
```

- UserDaoJdbc는 SqlService 인터페이스를 통해 SQL을 가져올 수 있도록 만든다.
- 일단 SqlService 타입의 빈을 DI 받을 수 있도록 프로퍼티를 정의해준다.

```java
// 7-9. SqlService 프로퍼티 추가
public class UserDaoJdbc implements UserDao {

  ...
  private SqlService sqlService;

  public void setSqlService(SqlService sqlService) {
    this.sqlService = sqlService;
  }
```
- 이제 모든 메서드에서 sqlService를 이용해 SQL을 가져오도록 수정한다.
- SqlService는 모든 DAO에서 서비스 빈을 사용하게 만들 것이기 때문에 키 이름이 DAO 별로 중복되지 않게 해야 한다.

```java
// 7-10. sqlService를 사용하도록 수정한 메서드
public void add(User user) {
  this.jdbcTemplate.update(this.sqlService.getSql("userAdd"), user.getId(), user.getName(), user.getPassword(), user.getEmail(), user.getLevel().intValue(), user.getLogin(), user.getRecommend());
}

public User get(String id) {
  return this.jdbcTemplate.queryForObject(this.sqlService.getSql("userGet"), new Object[] {id}, this.userMapper);
}

public List<User> getAll() {
  return this.jdbcTemplate.queryForObject(this.sqlService.getSql("userGetAll"), this.userMapper);
}

public void deleteAll() {
  this.jdbcTemplate.update(this.sqlService.getSql("userDeleteAll"));
}

public int getCount() {
  return jdbcTemplate.queryForInt(this.sqlService.getSql("userGetCount"));
}

public void update(User user) {
  this.jdbcTemplate.update(this.sqlService.getSql("userUpdate"), user.getName(), ...(생략));
}
```

#### 스프링 설정을 사용하는 단순 SQL 서비스

- SqlService 인터페이스에는 어떤 기술적인 조건이나 제약사항도 담겨 있지 않다. 어떤 방법이든 상관없이 DAO가 요구하는 SQL을 돌려주기만 하면 된다.

```java
public class SimpleSqlService implements SqlService {

    private final Map<String, String> sqlMap;

    public SimpleSqlService(Map<String, String> sqlMap) {
        this.sqlMap = sqlMap;
    }

    @Override
    public String getSql(String key) throws SqlRetrievalFailureException {
        String sql = sqlMap.get(key);
        if (sql == null) {
            throw new SqlRetrievalFailureException(key + "에 대한 SQL을 찾을수 없습니다");
        }
        return sql;
    }
}
```
- SimpleSqlService 클래스를 빈으로 등록하고 UserDao가 DI받도록 설정해준다. SQL 정보는 이 빈 프로퍼티에 map 태그를 이용하여 등록하면 된다. 


## 7.2 인터페이스의 분리와 자기참조 빈

- 이제 SqlService 인터페이스의 구현 방법을 고민해보자.
- 인터페이스로 대표되는 기능을 구현 방법과 확장 가능성에 따라 유연한 방법으로 재구성할 수 있도록 설계할 필요가 있다.

### 7.2.1 XML 파일 매핑
- `<bean>` 태그 안에 SQL을 넣는 것보다 SQL을 저장해두는 전용 포맷을 가진 독립적인 파일을 이용하는 편이 바람직하다.
- 검색용 키와 SQL 문장 두 가지를 담을 수 있는 XML 문서를 설계해보고, XML에서 SQL을 읽어뒀다가 DAO에게 제공해주는 SQL 서비스 구현 클래스를 만들어보자.

#### JAXB
- XML에 담긴 정보를 파일에서 읽어오는 방법 중 하나인 JAXB는 JDK 6라면 `java.xml.bind` 패키지 안에서 구현 클래스를 찾을 수 있다.
- DOM과 비교했을 때 JAXB의 장점은 XML 문서 정보를 거의 동일한 구조의 오브젝트로 직접 매핑해준다는 것이다. ➡️ XML 정보를 오브젝트처럼 다룰 수 있어 편리하다.
- JAXB는 XML 문서의 구조를 정의한 스키마를 이용해서 매핑할 오브젝트의 클래스까지 자동으로 만들어주는 컴파일러도 제공해준다.
- 스키마 컴파일러를 통해 자동생성된 오브젝트에는 매핑정보가 **애너테이션**으로 담겨있다.

<img width="615" alt="스크린샷 2024-08-12 오전 10 19 00" src="https://github.com/user-attachments/assets/f4dc4bb3-a9f9-4844-a671-a4f9fa7a156c">

#### SQL 맵을 위한 스키마 작성과 컴파일
```xml
// SQL 맵 XML 문서
<sqlmap>
  <sql key="userAdd">insert into users(...) ...</sql>
  <sql key="userGet">select * from users ...</sql>
  ...
</sqlmap>
```
이 XML 문서의 구조를 정의하는 스키마를 만들어보자.
```xml
// SQL 맵 문서에 대한 스키마
<?xml version="1.0" encoding="UTF-8"?>
<schema xmls="http://www.w3.org/2001/XMLSchema" targetNamespace="http://www.epril.com/sqlmap" xmls:tns="http://www.epril.com/sqlmap" elementFormDefault="qualified">

  <element name="sqlmap"> <!-- <sqlmap> 엘리멘트를 정의한다 -->
    <complexType>
      <sequence>
        <element name="sql" maxOccurs="unbounded" type="tns:sqlType" />
      </sequence>
    </complexType>
  </element>
  <complexType name="sqlType"> <!-- <sql>에 대한 정의를 시작한다. -->
    <simpleContent>
      <extension base="string">
        <attribute name="key" use="required" type="string" /> <!-- 검색을 위한 키 값은 <sql>의 key 애트리뷰트에 넣는다. 반드시 입력해야 하는 필수값이다.
      </extension>
    </simpleContent>
  </complexType>
</schema>
```
- 다음 명령을 사용해 컴파일한다.
```
xjc -p springbook.user.sqlservice.jaxb.sqlmap.xsd -d src
```
- src는 생성된 파일이 저장될 위치이다.
- 명령을 실행하면 두 개의 바인딩용 자바 클래스와 팩토리 클래스가 만들어진다. (SqlType.java, Sqlmap.java)

<img width="536" alt="스크린샷 2024-08-13 오후 4 40 23" src="https://github.com/user-attachments/assets/af4fbf74-7ce7-4c47-ba5a-c28d6caa3f9b">


#### 언마샬링
- 생성된 매핑 클래스 적용 전 먼저 JAXB API의 사용법을 익힐 수 있도록 간단한 학습 테스트를 만들어보자.
```xml
// 7-16. 테스트용 SQL 맵 XML 문서
<?xml version="1.0" encoding="UTF-8"?>
<schema xmls="http://www.w3.org/2001/XMLSchema" targetNamespace="http://www.epril.com/sqlmap" xmls:tns="http://www.epril.com/sqlmap" elementFormDefault="qualified">

  <sql key="add">insert</sql>
  <sql key="get">select</sql>
  <sql key="delete">delete</sql>
</sqlmap>
```

