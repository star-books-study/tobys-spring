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
