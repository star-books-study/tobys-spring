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


... (생략) ...


## 7.6 스프링 3.1의 DI
- 자바 언어와 관련 기술이 다양한 변화를 겪는 동안 스프링도 꾸준히 발전하고 변신해왔다.
- 스프링이 많은 변화를 겪은 것은 사실이지만 스프링이 기본적으로 지지하는 객체지향 언어인 자바의 특징과 장점을 극대화하는 프로그래밍 스타일과 이를 지원하는 도구로서의 스프링 정체성은 변하지 않았다.
- 놀랍게도 스프링은 1.0부터 3.1까지 거의 완벽에 가까울 만큼 구 버전 호환성을 유지하고 있다.
- 스프링을 이용한 애플리케이션 코드가 DI 패턴을 이용해서 안전하게 발전하고 확장할 수 있는 것처럼 스프링 프레임워크 자체도 DI 원칙을 충실하게 따라서 만들어졌기 때문에 기존 설계와 코드에 영향을 주지 않고도 꾸준히 새로운 기능을 추가하고 확장해나가는 일이 가능했다.

#### 자바 언어의 변화와 스프링
- DI가 적용된 코드를 작성할 때 사용하는 핵심 도구인 **자바 언어**에는 그간 적지 않은 변화가 있었다.
- 이런 변화들이 스프링의 사용 방식에도 여러 영향을 줬다.
- 대표적인 두 가지 변화를 살펴보자.

- **애너테이션의 메타정보 활용**
  - 자바는 소스코드가 컴파일된 후 클래스 파일에 저장됐다가, JVM에 의해 메모리로 로딩되어 실행된다. 그런데 때로는 자바 코드가 실행이 목적이 아니라 데이터로 취급되기도 한다.
  - 자바 코드의 일부를 리플렉션 API 등을 이용해 이떻게 만들었는지 살펴보고 그에 따라 동작하는 기능이 점점 많이 사용되고 있다.
  - 원래 리플렉션 API는 자바 코드나 컴포넌트를 작성하는 데 사용되는 툴을 개발할 때 이용되도록 만들어졌는데, 언제부턴가 본래 목적보다는 자바 코드의 메타 정보를 데이터로 활용하는 스타일의 프로그래밍 방식에 더 많이 활용되고 있다. 이런 프로그래밍 방식의 절정이 자바 5에서 등장한 **애너테이션**일 것이다.
  - 애너테이션은 복잡한 리플렉션 API를 이용해 애너테이션의 메타정보를 조회하고, 애너테이션 내에 설정된 값을 가져와 참고하는 방법이 전부다.
  - 애너테이션 자체가 클래스의 타입에 영향을 주지도 못하고, 일반 코드에서 활용되지도 못하기 때문에 일반적인 객체지향 프로그래밍 스타일의 코드나 패턴 등에 적용할 수도 없다.
  - 그럼에도 애너테이션을 이용하는 표준 기술과 프레임워크는 빠르게 증가했다. 이렇게 **애너테이션의 활용이 늘어난 이유는 무엇일까?**
  - 애너테이션은 핵심 로직을 담은 자바 코드와 이를 지원하는 IoC 방식의 프레임워크, 그리고 프레임워크가 참조하는 메타정보라는 세 가지로 구성하는 방식에 잘 어울리기 때문일 것이다.
  - 애플리케이션을 구성하는 많은 오브젝트와의 관계를 설정할 때 단순 자바코드로 만들어두면 불편함
    ➡️ 그래서 xml로 전환했
    ➡️ 애노테이션 등장. 애노테이션은 자바코드의 일부로 사용되기 때문에 xml보다 유리한 점이 많다.
  - 정의하기에 따라서 타입, 필드, 메서드, 파라미터 등 여러 레벨에 적용 가능
  - 단순히 애너테이션 하나 추가하는 것만으로 애너테이션이 부여된 클래스의 패키지, 클래스 이름, 접근 제한자 등의 메타 정보를 알 수 있다.
  - 단점은 변경할 때마다 매번 클래스를 새로 컴파일해줘야 한다는 것
  - 그러나 흐름은 애너테이션으로 가고 있다. 스프링 3.1부터 xml을 완전히 배제한 설정이 가능하다.
- **정책과 관례를 이용한 프로그래밍**
  - 애너테이션과 같은 메타 정보를 활용하는 프로그래밍 방식은 코드 없이도 약속한 규칙 또는 관례를 따라 프로그램이 동작하도록 만드는 프로그래밍 스타일을 적극적으로 포용하게 만들었다.
    - 예) 스프링의 XML
  - 반면 미리 정의된 많은 규칙과 관례를 기억해야 하고, 메타정보를 보고 프로그램이 어떻게 동작할지 이해해야 하는 부담을 주기도 한다.
  - 적지 않은 학습 비용이 들고, 자칫 잘못 이해하고 있을 경우 찾기 힘든 버그를 만들어내기도 한다.
  - RoR 프레임워크의 인기에 영향을 받아 이런 스타일의 프로그래밍 방식은 스프링에도 많은 영향을 주었다.
    - 같은 XML을 작성하더라도 메타 정보의 양을 최소화
    - 애너테이션은 일정한 패턴을 따르는 경우 관례를 부여
  - 코드로 직접 모든 내용을 작성하는 것보다 간결하고 빠른 개발이 가능하기 때문에 이런 스타일의 프로그래밍 방식은 지속적으로 인기를 끌고 있다.
  - 가능한 한 명시적으로 메타정보를 작성하는 것이 안전하고 쉽다고 보는 사람도 있고, 최대한 관례와 지능적인 디폴트 등을 이용해서 작성할 코드와 메타정보를 최소화하는 방법에 매력을 느끼기도 한다.
  - 어쨌든 스프링은 점차 애너테이션으로 메타정보를 작성하고, 미리 정해진 정책과 관례를 활용해 간결한 코드에 많은 내용을 담을 수 있는 방식을 적극 도입하고 있다.
- 지금까지 발전시켜온 사용자 DAO와 서비스 기능을 스프링 3.1의 최신 DI 스타일로 바꿔보자.


### 7.6.1 자바 코드를 이용한 빈 설정
- **첫 번째 작업 : XML 없애기**
- 지금까지는 자바 코드보다 간결하고 편하게 DI 정보를 담을 수 있어 XML을 사용해왔지만 이제는 **애너테이션과 새로운 스타일의 자바 코드**로 바꿀 것
- XML에 담긴 DI 정보는 스프링 테스트 컨텍스트를 이용해 테스트를 작성할 때 사용
  ➡️ 스프링 컨텍스트의 도움이 필요 없는 UserTest 같은 단위 테스트에서는 필요 없음
  ➡️ UserDaoTest & UserServiceTest 두 가지만 신경 쓰면 된다.
- 이제부터 진행할 작업에는 DI 관련 정보를 스프링 3.1로 변경하는 일과 함께 테스트용으로 만들어진 기존 XML에서 애플리케이션이 **운영환경**에서 동작할 때 필요로 하는 DI 정보를 분리해내는 일도 포함된다.

#### 테스트 컨텍스트의 변경
- UserDaoTest와 UserServiceTest에는 다음과 같이 XML 위치를 지정하는 코드가 들어가 있다.
```java
// 8-82. XML 파일을 사용하는 UserDaoTest
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations="/test-applicatioinContext.xml")
public class UserDaoTest {
```

- 먼저 DI 정보로 사용될 자바 클래스를 만들어야 한다. DI 설정 정보를 담은 클래스는 평범한 자바 클래스에 `@Configuration` 애너테이션을 달아주면 만들 수 있다.

```java
// 7-83. DI 메타 정보로 사용될 TestApplicationContext 클래스
@Configuration
public class TestApplicationContext {
}
```
