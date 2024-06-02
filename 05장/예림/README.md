# 05장. 서비스 추상화

- 자바에는 사용방법과 형식은 다르지만 기능과 목적이 유사한 기술이 존재
- 환경과 상황에 따라서 기술이 바뀌고, 그에 따라 다른 API를 사용하고 다른 접근 방법을 따라야 한다는 것은 피곤
- 5장에서는 지금까지 만든 DAO에 트랜잭션을 적용해보면서 스프링이 어떻게 성격이 비슷한 여러 종류의 기술을 추상화하고 일관된 방법으로 사용할 수 있도록 지원하는지 살펴본다.

## 5.1 사용자 레벨 관리 기능 추가
- UserDao에 간단한 사용자 관리 비즈니스 로직을 추가해보자. (정보 삽입, 검색, 사용자 레벨 조정)
- 구현해야 할 비즈니스 로직
  - 레벨 : BASIC, SILVER, GOLD
  - BASIC -> SILVER : 가입 후 50회 이상 로그인
  - SILVER -> GOLD : 30번 이상 추천 받기
  - 사용자 레벨 변경 작업은 일정한 주기를 가지고 일괄적으로 진행. 변경 작업 전에는 조건을 충족하더라도 레벨의 변경이 일어나지 않음
 
### 5.1.1 필드 추가

#### Level 이넘
- 레벨을 상수로 설정하면 다른 종류의 정보를 넣는 실수를 해도 컴파일러가 체크해주지 못하고, 엉뚱한 레벨이 들어갈 수도 있다.
  => 이넘 사용이 안전
```java
// 5-3. 사용자 레벨용 enum
public enum Level {
  BASIC(1), SILVER(2), GOLD(3); // 세 개의 enum 오브젝트 정의

  private final int value;

  Level(int value) { // DB에 저장할 값을 넣어줄 생성자
    this.value = value;
  }

 public static Level valueOf(int value) { // 값으로부터 타입 오브젝트를 가져오도록 만든 스태틱 메서드
  switch(value) {
    case 1: return BSIC;
    case 2: return SILVER;
    case 3: return GOLD;
    default: throw new AssertionError("Unknown value: " + value);
  }
}
```

#### User 필드 추가
- Level 타입의 변수를 User 클래스에 추가한다.
- 로그인 횟수와 추천 수도 추가하자.

```java
// 5-4. User에 추가된 필드
public class User {
  ...
  Level level;
  int login;
  int recommend;

  public Level getValue() {
    return level;
  }

  public void setLevel(Level level) {
    this.level = level;
  }
  ...
  // login, recommedn getter, setter 생략
```

#### UserDaoTest 수정
기존 코드에 새로운 기능을 추가하려면 테스트를 먼저 만드는 것이 안전

```java
// 5-5. 수정된 테스트 픽스처
public class UserDaoTest {
  ...
  @Before
  public void setUp() {
    this.user1 = new User("gyumee", "박성철", "springno1", Level.BASIC, 1, 0);
    ...
  }
```

이에 맞게 User 클래스의 생성자 파라미터도 추가해준다.

```java
// 5-6. 추가된 필드를 파라미터로 포함하는 생성자
class User {
  ...
  public User(String id, String name, String password, Level level, int login, int recommend) {
  this.id = id;
  ...
  }
}
```

UserDaoTest에서 두 개의 User 오브젝트 필드 값이 모두 같은지 비교하는 checkSameUser() 메서드를 수정한다.

```java
// 5-7. 새로운 필드를 포함하는 User 필드 값 검증 메서드
public void checkSameUser(User user1, User user2) {
  assertThat(user1.getId(), is(user2.getId()));
  ...
  assertThat(user1.getLevel(), is(user2.getLevel()));
  ... login, recommand 필드 값 검증
}
```
기존에는 테스트 메서드에서 직접 assertThat을 사용했지만 필드도 추가됐고 이제 필드가 늘어나도 User 오브젝트 비교 로직을 일정하게 유지할 수 있도록 checkSameUser()를 이용해 다음과 같이 수정한다.
```java
// 5-8. checkSameUser() 메서드를 사용하도록 addAndGet() 메서드
@Test
public void addAndGet() {
  ...
  User userget1 = dao.get(user1.getId());
  checkSameUser(userget1, user1);

  User userget2 = dao.get(user2.getId());
  checkSameUser(userget2, user2);
}
```

#### UserDaoJdbc 수정
```java
// 5-9. 추가된 필드를 위한 UserDaoJdbc의 수정 코드
class public UserDaoJdbc implements UserDao {
  ...
  private RowMapper<User> userMapper =
   new RowMapper<User>() {
      public User MapRow(ResultSet rs, int rowNum) throws SQLException {
        User user = new User();
        user.setId(rs.getString("id"));
        user.setName(rs.getString("name"));
        ...
        user.setLevel(Level.valueOf(rs.getInt("level")));
        ...
        return user;
    }
  };
  public void add(User user) {
    this.jdbcTemplate.update(
      "insert into users(id, name, password, level, login, recommend) " +
      "values(?,?,?,?,?,?)", user.getId(), user.getName(),
      user.getPassword(), user.getLevel().intValue(),
      user.getLogin(), user.getRecoomend());
}
```
- Level 타입의 level 필드를 사용할 때, Level 이넘은 DB에 저장될 수 있는 SQL 타입이 아니므로 정수형 값으로 반환함
- 반대로 조회를 했을 경우, ResultSet에서는 DB의 타입인 int로 level 정보를 가져오므로 valueOf() 메서드를 사용해 타입을 바꿔주고 setLevel() 메서드에 전달한다.
- 먼저 매핑을 위와 같이 수정해준다. 이제 DB에서 데이터를 불러왔을 때 User 객체에는 잘 반영될 것이다.

### 5.2.1 사용자 수정 기능 추가
- 사용자 정보는 여러 번 수정될 수 있음. 성능을 극대화하기 위해 각각 여러 개 수정용 DAO를 만들어야 할 때도 있음.
- 아직은 사용자 정보가 단순하므로 수정할 정보가 담긴 User 오브젝트를 전달하면 필드 정보를 UPDATE 문을 이용해 변경해주는 메서드를 만들어보자.

#### 수정 기능 테스트 추가
```java
// 5-10. 사용자 정보 수정 메서트 테스트
@Test
public void update() {
  dao.deleteAll();

  dao.add(user1);

  user1.setName("정예림");
  user1.setPassword("1234");
  user1.setLevel(Level.GOLD);
  user1.setLogin(1000);
  user1.setRecommend(999);
  dao.update(user1);

  User user1update = dao.get(user1.getId());
  checkSameUser(user1, user1update);
}
```
#### UserDao와 UserDaoJdbc 수정
```java
// 5-11. update() 메서드 추가
public interface UserDao {
  ...
  public void update(User user1);
}
```

```java
// 5-12. 사용자 정보 수정용 update() 메서드
public void update(User user) {
  this.jdbcTemplate.update(
    "update users set name = ?, password = ?, level = ?, login = ?, " +
    "remommend = ? where id = ? ", user.getName(), user.getPassword(),
    user.getLevel().intValue(), user.getLogin(), user.getRecommend(),
    user.getId());
}
```

#### 수정 테스트 보완
- 필드 이름이나 SQL 키워드를 잘못 넣은 거라면 에러가 나니 쉽게 확인할 수 있지만,
- UPDATE 문장에서 WHERE 절을 빼먹는 경우 정상적으로 작동하게 된다.
- 해결 방법
  1. JdbcTemplate의 update()가 돌려주는 리턴 값을 확인하는 것
  2. 테스트를 보강해서 원하는 사용자 외의 정보는 변경되지 않았음을 직접 확인

- 다음은 두 번째 방법으로 테스트를 보완한 것이다.
```java
// 5-13. 보완된 update() 테스트
@Test
public void update() {
  dao.deleteAll();

  dao.add(user1); // 수정할 사용자
  dao.add(user1); // 수정하지 않을 사용자

  user1.setName("정예림");
  ...
  user1.setRecommend(999);

  dao.update(user1);

  User user1update = dao.get(user1.getId());
  checkSameUser(user1, user1update);
  User user2same = dao.get(user2.getId());
  checkSameUser(user2, user2same);
}
```

### 5.1.3 UserService.upgradeLevels()
- 사용자 관리 로직을 담을 클래스 UserService를 추가하자.
- UserService는 UserDao 인터페이스 타입으로 userDao 빈을 DI 받아 사용하게 만든다.
- UserService는 UserDao의 구현 클래스가 바뀌어도 영향을 받지 않아야 한다.
  - 따라서 DAO 인터페이스를 사용하고 DI를 적용해야 한다. -> UserService도 스프링의 빈으로 등록돼야 한다.
  - UserService를 위한 테스트 클래스도 하나 추가하자.
- UserService의 클래스 레벨 의존 관계
  <img width="489" alt="스크린샷 2024-06-02 오전 10 23 30" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/d00cfe64-4a2c-40c7-aa31-e44b0d161d4a">

#### UserService 클래스와 빈 등록
- UserDao 오브젝트가 DI 가능하도록 수정자 메서드 추가
  ```java
  // 5-14. UserService 클래스
  ...
  public class UserService {
    UserDao userDao;
  
    public void setUserDao(UserDao userDao) {
      this.userDao = userDao;
    }
  }
  ```
- 스프링 설정 파일에 userService 아이디로 빈을 추가 & userDao 빈을 DI 받도록 프로퍼티 추가
  ```xml
  <bean id="userService" class="springbook.user.service.UserService">
    <property name="userDao" ref="userDao" />
  </bean>
  
  <bean id="userDao" class="springbook.dao.UserDaoJdbc">
    <property name="dataSource" ref="dataSource" />
  </bean>
  ```
#### UserServiceTest 테스트 클래스
```java
// 5-16. UserServiceTest 클래스
...
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations="/test-applicationContext.xml")
public class UserServiceTest {
  @Autowired
  UserService userService; // 테스트 대상인 UserService 빈을 제공받음
}
```
#### upgradeLevels() 메서드
- 사용자 레벨 관리 기능을 먼저 만들고 테스트를 만들어보자.

```java
// 5-18. 사용자 레벨 업그레이드 메서드
