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
