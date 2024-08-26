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
// 7-82. XML 파일을 사용하는 UserDaoTest
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

```java
// 7-84. TestApplicationContext를 테스트 컨텍스트로 사용하도록 변경한 UserDaoTest
@RunWith(SpringJUnit4ClassRunnder.class)
@ContextConfiguration(classes=TestApplicationContext.class)
public class UserDaoTest {
```

- XML에 있던 모든 빈 설정 정보를 TestApplicationContext에 담는 대신 XML의 도움을 받는 게 좋겠다.
- `@ImportResource` 애너테이션 사용
```java
// 7-85. TestApplicationContext를 테스트 컨텍스트로 사용하도록 변경한 UserDaoTest
@Configuration
@ImportResource("/test-applicationContext.xml")
public class TestApplicationContext {
}
```

#### <context:annotation-config /> 제거
- `<context:annotation-config />`은 `@PostConstruct`를 붙인 메서드가 빈이 초기화된 후에 자동으로 실행되도록 사용했다.
- @Configuration이 붙은 설정 클래스를 사용하는 컨테이너가 사용되면 더 이상 필요 없다. 컨테이너가 직접 `@PostConstruct` 애너테이션을 처리하는 빈 후처리기를 등록해주기 때문이다.

#### <baan>의 전환
- @Bean 이 붙은 public 메서드 이름은 <bean>의 id와 같다.
- 리턴값은 구현 클래스보다 인터페이스로 해야 DI에 따라 구현체를 자유롭게 변경할 수 있다.
- 그런데 메서드 내부에서는 빈의 구현 클래스에 맞는 프로퍼티 값 주입이 필요하다.
```xml
<bean id="dataSource" class="org.springboot.jdbc.datasource.SimpleDriverDataSource">
    <property name="driverClass" value="com.mysql.jdbc.mysql" />
    ...
</bean>
```

```java
@Bean
public DataSource dataSource() {
    SimpleDriverDataSource dataSource = new SimpleDriverDataSource ();

    dataSource.setDriver(Driver.class);
    dataSource.setUrl("jdbc:mysql://localhost/springbook?characterEncoding=UTF-8");
    dataSource.setUsername("spring");
    dataSource.setPassword("book");
    return dataSource;
}
```
- XML로 정의된 transcationManager 빈을 TestApplicationContext 내의 메소드로 전환하면 다음과 같다.
```xml
<bean id="transactionManager" class="org.springboot.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource" />
</bean>
```

```java
@Bean 
    public PlatformTranscationManager transcationManager() {
        DataSourceTransactionManager tm = new DataSourceTransactionManager();
        tm.setDataSource(dataSource());
        return tm;
    }
```

#### 전용 태그 전환
- Spring 3.1은 xml에서 자주 사용되는 전용 태그를 `@Enable`로 시작하는 애노테이션으로 대체할 수 있도록 애노테이션을 제공한다.
- `<tx:annotation-driven />`은 `@EnableTransactionManagement`로 대체할 수 있다.


### 7.6.2 빈 스캐닝과 자동 와이어링

#### @Autowired를 이용한 자동와이어링
- 빈으로 사용되는 UserServiceImpl이나 UserDaoJdbc 같은 클래스에서는 `@Autowired`를 사용할 수 없을까? 물론 사용할 수 있다.
  ![image](https://github.com/user-attachments/assets/387badbc-c878-4aa1-9cdc-1d1d31a4f7ac)
- 자동와이어링을 사용하면 컨테이너가 이름 / 타입 기준으로 주입될 빈을 찾아준다. ➡️ 자바 코드나 XML의 양을 대폭 줄일 수 있다.
- setter에 `@Autowired`를 붙이면 파라미터 타입을 보고 주입 가능한 타입의 빈을 모두 찾는다. 주입 가능한 빈이 1개일 땐 스프링이 setter를 호출해서 넣고, 2개 이상일 때는 그 중에서 프로퍼티와 동일한 이름의 빈을 찾고 없으면 에러
```java
// 7-100. dataSource 수정자에 @Autowired 적용
public class UserDaoJdbc implements UserDao {

  @Autowired
  public void setDataSource(DataSource dataSource) {
    this.jdbcTemplate = new JdbcTemplate(dataSource);
  }
```
- setter에서 필드 그대로 넣는다면 필드에 직접 @Autowired 를 적용할 수 있다.
  - 위 예시코드의 경우에는 필드에 직접 하면 안된다.(setter에 JdbcTemplate을 생성하는 코드가 있으므로 필드로 대체가 되지 않는다.)
```java
// 7-101. sqlService 필드에 @Autowired 적용
public class UserDaoJdbc implements UserDao {
  ...

  @Autowired
  private SqlService sqlService;

  public void setSqlService(SqlService sqlService) {
    this.sqlService = sqlService;
  }
```
- 원래 자바에서 private 필드에는 클래스 외부에서 값을 넣을 수 없게 되어 있지만 스프링은 리플렉션 API를 이용해 제약 조건을 우회해서 값을 넣어준다.
- 필드에 직접 값을 넣을 수 있다면 수정자 멤서드는 없어도 된다.
- 반면에 setDataSource() 수정자 메서드를 없애고 필드에 `@Autowired`를 적용하는 건 불가능
    - 여타 수정자 메서드처럼 주어진 오브젝트를 그대로 필드에 저장하는 대신 JdbcTemplate을 생성해서 저장해주기 때문
- 단순히 필드에 값을 수정하는 수정자 메서드라도 `@Autowired`를 필드에 직접 부여했다고 메서드를 생략하면 안되는 경우가 있으니 주의하자.
- `@Autowired`와 같은 자동와이어링은 편리하지만 빈 설정 정보를 보고 의존관계를 한눈에 파악하기 힘들다는 단점도 있다.

#### @Component를 이용한 자동 빈 등록
- 클래스에 부여
- `@Component`가 붙은 클래스는 빈 스캐너를 통해 자동으로 빈으로 등록된다.
- 스캔에 사용되는 애너테이션은 `@ComponentScan`이고 basePackage를 기준으로 한다.
  - 프로젝트 내의 모든 클래스패스를 다 뒤져서 @Component 애너테이션이 달린 클래스를 찾는 것은 부담되는 작업
  - 특정 패키지 아래서만 찾도록 기준이 되는 패키지를 지정할 때 사용
  ```
  @ComponentScan(basePackages="springbook.user")
  ```
  - basePackage 엘리먼트는 기준 패키지를 설정할 때 사용
- `@Component`로 추가되는 빈의 id는 별도 지정 없으면 클래스 이름의 첫 글자를 소문자로 바꿔서 사용
  - 별도 지정은 `@Component("userDao")`와 같이 한다.

#### 메타 애너테이션
- 애너테이션의 정의에 부여된 애너테이션을 의미
- 빈 스캔을 통해 자동 등록 대상으로 인식하게 하려면 `@Component`을 붙여주면 된다.
```java
@Component
public @interface SnsConnector { // annotation은 @interface 키워드로 정의
  ...
}
```
- bean 스캔 검색 대상 + 부가적인 용도의 마커로 사용하기 위한 `@Repository`(Dao기능 제공 클래스), `@Service`(비즈니스 로직을 담은 빈) 와 같은 빈 자동등록용 애노테이션이 사용된다.

### 7.6.3 컨텍스트 분리와 @import
- 이번에는 성격이 다른 DI 정보를 분리해보자.
#### 테스트용 컨텍스트 방법
- 테스트용 DI 정보는 분리하자.
- **DI 설정 정보 분리하는 방법은 DI 설정 클래스를 추가하고 관련 애너테이션, 필드, 메서드를 옮기면 된다.**
<img width="654" alt="스크린샷 2024-08-22 오후 9 01 47" src="https://github.com/user-attachments/assets/96c2c8df-5b49-487a-aa38-0c8a12b594ef">
- 그리고 DI 설정 클래스에는 @Configuration을 붙여준다.
- userDao, mailSender 프로퍼티는 자동와이어링 대상이므로 다음과 같이 간단하게 바꿀 수 있다.
<img width="679" alt="스크린샷 2024-08-22 오후 9 03 36" src="https://github.com/user-attachments/assets/e367eaf0-53c1-4643-9fc6-8e2ca78a3ff0">
- 테스트용 컨텍스트를 만들 때 이제는 두 개의 클래스가 필요하므로 다음과 같이 테스트 컨텍스트의 DI 설정 클래스 정보를 수정할 수 있다.
<img width="643" alt="스크린샷 2024-08-22 오후 9 06 21" src="https://github.com/user-attachments/assets/8bb9f839-4421-46c7-8014-9ded897094ba">

#### @Import
- SQL 서비스용 빈도 독립적인 모듈로 취급하는 게 나아 보인다.
  - 다른 애플리케이션에서도 사용 가능
- `@Configuration` 클래스를 하나 더 만들자.
```java
@Configuration
public class SqlServiceContext {
  ...
}
```
- DI 설정 정보를 담은 클래스가 세 개가 됐다(두 개는 애플리케이션 핵심 빈 정보, 한 개는 테스트) 
- `@ContextConfiguration`의 `classes` 내용을 수정할 필요 없이 `@Import`를 사용하자.
- `@Import` : 자바 클래스로 된 설정 정보를 가져올 때 사용한다.
```java
// AppContext가 메인 설정 정보, SqlServiceContext는 보조 설정 정보로 사용
@Import(SqlServiceContext.class)
public class AppContext {
```

### 7.6.4 프로파일
- 지금까지는 DummyMailSender라는 테스트용 클래스를 만들어 사용했지만 운영 시스템에서는 실제 동작하는 메일 서버를 통해 메일을 발송하는 기능이 있는 메일 발송 서비스 빈이 필요하다.
- AppContext에 실제 애플리케이션이 동작될 때 사용되는 MailSender 타입 빈 설정을 넣어주면 테스트용과 운영용 mailSender 빈이 테스트에 사용되는 문제가 발생한다.
- mailSender 빈처럼 양쪽 모두 필요하면서 빈의 내용이 달라져야 하는 경우에는 빈 설정정보 작성이 곤란해진다.
- 이 문제를 해결하려면 **운영환경에는 반드시 필요하지만 테스트 실행 중에는 배제되어야 하는 빈 설정을 별도의 설정 클래스를 만들어 따로 관리할 필요가 있다.
  ➡️ ProductAppContext라는 새로운 클래스 생성 후 운영환경에서는 AppContext와 ProductAppContext가 DI 정보로 사용되게 설정

<img width="601" alt="스크린샷 2024-08-24 오후 4 57 55" src="https://github.com/user-attachments/assets/85f86c62-d90b-450b-9664-3daf7702457f">

- 그러나 이런 식으로 설정 파일이 여러 개로 쪼개지고 몇 개를 선택해서 동작하는 구성은 번거롭다.

#### @Profile과 @ActiveProfiles
- 실행환경에 따라 빈 구성이 달라지는 내용을 프로파일로 정의해서 만들어두고, 실행 시점에 어떤 프로파일의 빈 설정을 사용할지 지정하자.
- 프로파일을 적용하면 하나의 설정 클래스만 가지고 환경에 따라 다른 빈 설정 조합을 만들어낼 수 있다.
- 프로파일은 설정 클래스 단위로 지정한다. 다음과 같이 @Profile 애너테이션을 클래스 레벨에 부여하고 프로파일 이름을 넣어주면 된다.
```java
@Configuration
@Profile("test")
public class TestAppContext {
```
>  ProductionAppContext는 production 프로파일 지정
- 이제 TestAppContext는 test 프로파일의 빈 설정 정보를 담은 클래스가 됐다.
- 프로파일이 지정되어 있지 않은 빈 설정은 default 프로파일로 취급
- 프로파일을 적용하면 모든 설정 클래스를 부담 없이 메인 설정 클래스에서 @Import해도 된다는 장점이 있다.


```java
@Configuration
...
@Import({SqlServiceContext.class, TestAppContext.class, ProductionContext.class})
public class AppContext {
```
- 그러나 `@Profile`이 붙은 설정 클래스는 @Import로 가져오든 `@ContextConfiguration`에 직접 명시하든 상관 없이 현재 컨테이너의 활성 프로파일 목록에 자신이 프로파일 이름이 들어 있지 않으면 무시된다.
  - 활성 프로파일 : 스프링 컨테이너를 실행할 때 추가로 지정해주는 속성
- 테스트가 실행될 때 활성 프로파일로 test 프로파일을 지정하려면 `@ActiveProfiles`를 사용하면 된다.
#### 컨테이너의 빈 등록 정보 확인
- 스프링 컨테이너는 모두 `BeanFactory`라는 인터페이스를 구현하고 있다.
- 대부분은 `DefaultListableBeanFactory`에서 관리 된다.

```java
@Autowired
DefaultListableBeanFactory bf;

public void beans() {
  for(String n : bf.getBeanDefinitionNames()) {
    System.out.println("bf.getBean(n).getClass().getName()");
  }
}
```

- 테스트를 실행해보면 테스트 컨텍스트에 등록된 빈 이름과 빈의 클래스를 모두 얻을 수 있다.

#### 중첩 클래스를 이용한 프로파일 적용
- 프로파일마다 빈 구성이나 구현 클래스에 어떤 차이가 있는지 한눈에 비교하기 불편할 수 있다.
- 이번에는 프로파일에 따라 분리했던 설정 정보를 하나의 파일로 모아보자. (스태틱 중첩 클래스 이용)
```java
@Configuration
...
@Import({SqlServiceContext.class, TestAppContext.class, ProductionContext.class})
public class AppContext {

  @Bean
  public DataSource dataSource() {
    ...
  }

  @Bean
  public PlatformTransactionManager transactionManager() {
    ...
  }

  @Configuration
  @Profile("production")
  public static class ProductionAppContext {
    ...
  }

  @Configuration
  @Profile("test")
  public static class TestAppContext {
    ...
  }
}
```

### 7.6.5 프로퍼티 소스
- 적어도 DB 연결 정보는 환경에 따라서 다르게 설정될 수 있어야 함 ➡️ 빌드 작업이 따로 필요하는 XML이나 프로퍼티 파일 같은 텍스트 파일에 저장하자.
#### @PropertySource
- 프로퍼티 파일의 확장자는 보통 properties이고, 내부에 키=값 형태로 프로퍼티를 정의
- `@PropertySource`로 프로퍼티 소스를 등록할 수 있다.
```java
...
@PropertySource("/database.properties")
public class AppContext {
```
- `@PropertySource`로 등록한 리소스로부터 가져오는 프로퍼티 값은 컨테이너가 관리하는 Environment 타입의 환경 오브젝트에 저장된다.
- Environment 오브젝트의 getProperty() 메서드를 이용하면 프로퍼티 값을 가져올 수 있다.
```java
@Autowired Environment env;

@Bean
public DataSource dataSource() {
  ...
  (생략) ... env.getProperty("db.driverClass");
}
```
#### PropertySourcesPlaceholderConfigurer
- `@Value`를 이용하여 `${프로퍼티 속성}`으로 주입 가능
<img width="634" alt="스크린샷 2024-08-25 오후 10 56 54" src="https://github.com/user-attachments/assets/078f39ec-3ba0-4a6a-9605-6c521ca45397">


### 7.6.6 빈 설정의 재사용과 @Enable
- 이제는 SQL 서비스 관련 빈 등록을 @Import 한 줄만 추가해주면 한 번에 끝낼 수 있다.
```java
@Import(SqlServiceContext.class)
```

#### 빈 설정자
- SqlServiecContext에서 SQL 매핑파일의 위치를 지정하는 작업을 분리하기 위해 다음과 같은 인터페이스를 정의할 수 있다.
