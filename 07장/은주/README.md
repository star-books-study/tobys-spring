# 7장. 스프링 핵심 기술의 응용
- 스프링이 가장 가치를 두고 적극적으로 활용하려고 하는 것은 자바 언어가 기반을 둔 `객체지향` 기술이다.
- 스프링의 모든 기술은 결국 **객체지향적인 언어의 장점을 적극적으로 활용해서 코드를 작성하도록 도와주는 것**이다

## 7.1 SQL과 DAO의 분리
- 데이터 액세스 로직은 바뀌지 않더라도 DB 의 테이블, 필드명, SQL 문장이 변할 수 있다
- 어떤 이유든지 SQL 변경이 필요한 상황이 발생하면 SQL을 담고 있는 DAO 코드가 수정될 수밖에 없다.
  - 따라서 SQL 을 적절히 분리해 DAO 코드와 다른 파일이나 위치에 두고 관리할 수 있다면 좋을 것이다

### 7.1.1. XML 설정을 이용한 분리
- SQL 을 스프링의 XML 설정파일로 빼내고, 설정파일에 프로퍼티 값으로 정의해서 DAO 에 주입해줄 수 있다

#### 개별 SQL 프로퍼티 방식
```java
public class UserDaoJdbc implements UserDao{
	private String sqlAdd;		
    
    public void setSqlAdd(String sqlAdd) {
    	this.sqlAdd = sqlAdd;
    }
}

public void add(User user) {
    this.jdbcTemplate.update(
        this.sqlAdd,
        user.getId(), user.getName(), ...
    );
}
```

```xml
<bean id="userDao" class="springbook.user.dao.UserDaoJdbc">
    <property name="dataSource" ref="dataSource" />
    <property name="sqlAdd" value="insert into users(id, name, password, email, level, login, recommend) values(?,?,?,?,?,?,?}" />
```
- 위와 같이 add() 메소드의 SQL 문장을 제거하고, 외부로부터 DI 받은 SQL 문장을 담은 sqlAdd 를 사용하게 만든다

#### SQL 맵 프로퍼티 방식
- SQL이 많아지면 그때마다 DAO에 DI용 프로퍼티를 추가하기 귀찮으니 SQL을 컬렉션으로 담아두는 방법을 시도해보자.
  - 맵을 이용하면 프로퍼티는 하나만 만들고, 키 값을 이용하여 SQL 문장을 가져올 수 있다
```java
public class UserDaoJdbc implements UserDao{
	private Map<String, String> sqlMap;
    
    public void setSqlMap(Map<String, String> sqlMap) {
    	this.sqlMap = sqlMap;
    }
}
public void add(User user) {
	this.jdbcTemplate.update(
    	this.sqlMap.get("add"), // 프로퍼티로 제공받은 맵으로부터 키를 이용해서 필요한 SQL을 가져온다
        user.getId() ...
    )
}
```

```xml
<bean id="userDao" class="springbook.user.dao.UserDaoJdbc">
    <property name="dataSource" ref="dataSource" />
    <property name="sqlMap">
        <map>
            <entry key="add" value="sql문.." />
            ...
        </map>
    </property>
```
### 7.1.2. SQL 제공 서비스
- SQL과 DI 설정정보가 섞여 있으면 보기에도 지저분하고 관리하기에도 좋지 않다. 
- 데이터 액세스 로직의 일부인 SQL 문장을 애플리케이션의 설정정보와 함께 두는건 바람직하지 못하다. 
  - SQL 을 따로 분리해둬야 독립적으로 SQL 리뷰나 SQL 튜닝 작업을 수행 하기도 편하다
- DAO가 사용할 SQL을 제공해주는 기능을 독립시킬 필요가 있으며 독립적인 SQL 제공 서비스가 필요하다. 
- SQL 제공 기능을 분리해서 다양한 SQL 정보 소스를 사용할 수 있고, 운영 중에 동적으로 갱신도 가능한 유연하고 확장성이 뛰어난 SQL 서비스를 만들어 보자.

#### SQL 서비스 인터페이스
- DAO 가 사용할 SQL 서비스의 기능은 SQL 에 대한 키 값을 전달하면 그에 해당하는 SQL 을 돌려주는 것이다.
- 
```java
public interface SqlService {
	String getSql(String key) throws SqlRetrievalFailureException;
}

public class UserDaoJdbc implements UserDao {
	private SqlService sqlService;
    
    public void setSqlService(SqlService sqlService) {
    	this.sqlService = sqlService;
    }
}

public void add(User user) {
	this.jdbcTemplate.update(this.sqlService.getSql("userAdd"), user.getId() ...);
}
```

#### 스프링 설정을 사용하는 단순 SQL 서비스
- SqlService를 구현하는 클래스를 만들고 Map으로 추가하고, 맵에서 SQL을 읽어서 돌려주도록 getSql() 메소드를 구현한다.

```java
public class SimpleSqlService implements SqlService {
	private Map<String, String> sqlMap;	// sql정보를 담을 Map
	
	public void setSqlMap(Map<String, String> sqlMap) {
		this.sqlMap = sqlMap;
	}
	
	@Override
	public String getSql(String key) throws SqlRetrievalFailureException {
		String sql = sqlMap.get(key);
		
		if(sql == null) {
			throw new SqlRetrievalFailureException(key + "에 대한 SQL을 찾을 수 없습니다.");
		} else return sql;
	}
}
```
- 이제 UserDao를 포함한 모든 DAO는 SQL을 어디에 저장해두고 가져오는지에 대해서는 전혀 신경 쓰지 않아도 된다. **구체적인 구현 방법과 기술에 상관없이 SqlService 인터페이스 타입의 빈을 DI 받아서 필요한 SQL을 가져다 쓰기만 하면 된다.**
- 동시에 sqlService 빈에는 DAO에 전혀 영향을 주지 않은 채로 다양한 방법으로 구현된 SqlService 타입 클래스를 적용할 수 있다. 
- 이제 DAO의 수정 없이도 편리하고 자유롭게 SQL 서비스 구현을 발전시킬수 있다

## 7.2. 인터페이스의 분리와 자기참조 빈
### 7.2.1. XML 파일 매핑
- 검색용 키와 SQL 문장 2가지를 담을 수 있는 간단한 XML 문서를 설계하고, 이 XML 파일에서 SQL 을 읽어뒀다가 DAO 에게 제공해주는 SQL 서비스 구현 클래스를 만들어보자

#### JAXB
- XML 에 담긴 정보를 파일에서 읽어오는 방법 중 하나로 JAXB (Java Architecture for XML Binding) 가 있다
- JAXB 의 장점은 XML 문서 정보를 거의 동일한 구조의 오브젝트로 직접 매핑해준다는 것이다.

## 7.6. 스프링 3.1의 DI
- 스프링 프레임워크 자체도 DI 원칙을 충실히 따라 만들어졌기에 기존 설계와 코드에 영향없이 꾸준히 새로운 기능을 추가하고 확장하는 것이 가능했다 

### 자바 언어의 변화와 스프링
#### 애노테이션의 메타정보 활용
- 리플렉션은 자바 코드의 메타정보를 데이터로 활용하는 스타일의 프로그래밍 방식에 더 많이 활용되고 있다
- 복잡한 리플렉션 API 를 이용해 애노테이션 메타정보 조회, 애노테이션 내에 설정된 값을 가져와 참고하는 방법으로 활용된다.
- 자바 개발의 흐름은 점차 **XML 같은 텍스트 형태의 메타 정보 활용에서, 자바 코드에 내장된 애노테이션으로 대체하는 쪽**으로 가고 있다
  - 스프링 3.1에 이르러서는 `핵심 로직`을 담은 자바 코드 / `DI 프레임 워크`(클라이언트) / DI 를 위한 `메타 데이터`로서의 자바코드 (클라이언트가 참고하는 일종의 메타정보 ; 애노테이션) 으로 재구성 되고 있다

#### 정책과 관례를 이용한 프로그래밍
- 스프링은 점차 애노테이션으로 메타정보를 작성하고, 미리 정해진 정책과 관례를 활용해서 간결한 코드에 많은 내용을 담을 수 있는 방식을 적극 도입하고 있다

### 7.6.1. 자바 코드를 이용한 빈 설정
- 지금까지 사용한 **XML 은 원래 자바 코드로 만든 오브젝트 팩토리의 기능을 프레임워크의 도움을 받아 간략한 방식으로 표현**한 것이다

#### 테스트 컨텍스트의 변경
- 자바 코드와 애노테이션으로 정의된 DI 정보와 @ImportResource 로 가져온 XML 의 DI 정보가 합쳐져서 최종 DI 설정 정보로 통합된다.
- 단계적으로 XML 내용을 옮기다가 XML 에 더이상 아무런 DI 정보가 남지 않으면 그 때 XML 파일과 @ImportResource 를 제거하면 XML 에서 자바 코드로의 전환 작업이 마무리된다

#### <context:annotation-config /> 제거
- `XML 에 담긴 DI 정보를 이용하는 스프링 컨테이너`를 사용하는 경우, **@PostConstruct 와 같은 애노테이션 기능이 필요하면 반드시 <context:annotation-config /> 를 포함시켜 필요한 빈 후처리기가 등록되게 만들어야 한다.**
- 반면 `@Configuration 이 붙은 설정 클래스를 사용하는 컨테이너`가 사용되면 **컨테이너가 직접 @PostConstruct 애노테이션을 처리하는 빈 후처리기를 등록**해주기에 <context:annotation-config /> 가 필요하지 않다.

#### <bean> 의 전환
- <bean> 으로 정의된 DI 정보는 @Bean 이 붙은 메소드와 거의 1:1 로 매핑된다
- 빈의 의존관계가 인터페이스를 통해 안전하게 맺어지도록 빈의 리턴값은 `인터페이스` 로 하는 것이 좋다
- @Bean 메소드 내부에서는 빈의 구현 클래스에 맞는 프로퍼티 값 주입이 필요한데, **프로퍼티는 구현 클래스에 의존적**인 경우가 대부분이다.
- XML 과 자바 클래스를 동시에 DI 정보로 사용하는 경우 자바 코드로 정의한 빈은 XML 에서 <property> 를 이용해 참조할 수 있다
- 자바 코드에서 XML 에서 정의한 빈을 참조하려면 @Autowired 를 붙여서 XML 에 정의된 빈을 컨테이너가 주입해주게 해야한다.
- @Resource 는 @Autowired 와 유사하게 필드에 빈 주입받을 때 사용하는데, **@Autowired 는 필드 타입 기준으로 빈을 찾고, @Resource 는 필드 이름 기준**으로 한다

#### 전용 태그 전환
- <tx: annotation-driven /> 과 같이 특별한 목적을 위해 만들어진, 내부적으로 복잡한 로우 레벨의 빈을 등록해주는 전용 태그에 대응되는 애노테이션을 제공해준다 @EnableTransactionManagement
- 스프링 3.1 은 **XML에서 자주 사용되는 전용 태그를 @Enable 로 시작하는 애노테이션으로 대체할 수 있게** 다양한 애노테이션을 제공한다

### 7.6.2. 빈 스캐닝와 자동와이어링
#### @Autowired 를 이용한 자동와이어링
- **@Autowired 는 자동와이어링 기법을 이용해 조건에 맞는 빈을 찾아 자동으로 수정자 메소드나 필드에 넣어준다**
  - 자동 와이어링 이용 시 컨테이너가 이름, 타입을 기준으로 주입될 빈을 찾아주기 때문에 빈의 프로퍼티 설정 코드 양을 대폭 줄일 수 있다
- 원래 자바에서 private 필드에는 클래스 외부에서 값을 넣을 수 없게 되어있지만, **스프링은 리플렉션 API 를 이용해 제약조건을 우회해서 값을 넣어준다.**
- 스프링 컨테이너에서 의존관계를 맺어주는 방식으로만 코드가 사용되면, @Autowired 를 필드에 직접 부여하고 수정자 메소드를 삭제해도 된다.
  - 하지만 스프링과 무관하게 직접 오브젝트 생성하고 다른 오브젝트를 주입하여 사용할 경우에는 수정자 메소드가 필요하다.

#### @Component 를 이용한 자동 빈 등록
- **@Component 또는 @Component 를 메타 애노테이션으로 갖고 있는 애노테이션이 붙은 클래스가 자동 빈 등록 대상이 된다**
  - @Component 는 빈으로 등록될 후보 클래스에 붙여주는 일종의 마커
- @Component 애노테이션이 달린 클래스를 자동으로 찾아 빈을 등록해주게 하려면 빈 스캔 기능을 사용하겠다는 애노테이션 정의가 필요하다.
  - 빈 자동등록이 컨테이너가 디폴트로 제공하는 기능은 아니기 때문
  - @ComponentScan 애노테이션에 basePackages 엘리먼트로 @Component 가 붙은 클래스를 스캔할 기준 패키지를 지정한다
- 애노테이션 포인트컷을 이용하면 패키지나 클래스 이름 패턴 대신 애노테이션을 기준으로 어드바이스 적용 대상을 선별할 수 있다