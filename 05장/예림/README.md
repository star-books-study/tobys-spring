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
public void upgradeLevels() {
        List<User> users = userDao.getAll();

        for (User user : users) {
            Boolean changed = null;

            if (user.getLevel() == Level.BASIC && user.getLoginCount() >= 50) {
                user.setLevel(Level.SILVER);
                changed = true;
            } else if (user.getLevel() == Level.SILVER && user.getRecommendCount() >= 30) {
                user.setLevel(Level.GOLD);
                changed = true;
            } else if (user.getLevel() == Level.GOLD) {
                changed = false;
            } else {
                changed = false;
            }

            if(changed) {
                userDao.update(user);
            }
        }
    }
```
- 어쩌다가 위와 같은 메소드를 만들었다고 생각해보자. 중복된 코드는 좀 나오고 책임의 분리도 잘 안되어있지만 비즈니스 로직이 명확히 보이고 아마 제대로 동작할 것이다.

#### upgradeLevels() 테스트
```java
// 5-19. 리스트로 만든 테스트 픽스처

class UserServiceTest {

  ...
  List<User> users; // 테스트 픽스처

  @Before
  public void setUp() {
      users = Arrays.asList(
              new User("bumjin", "박범진", "p1", Level.BASIC, 49, 0)
              , new User("joytouch", "강명성", "p2", Level.BASIC, 50, 0)
              , new User("erwins", "신승한", "p3", Level.SILVER, 60, 29)
              , new User("madnite1", "이상호", "p4", Level.SILVER, 60, 30)
              , new User("green", "오민규", "p5", Level.GOLD, 100, 100)
      );
  }
```
```java
// 5-20. 사용자 레벨 업그레이드 테스트
@Test
public void upgradeLevels() {
    userDao.deleteAll();
    for (User user : users) {
        userDao.add(user);
    }

    userService.upgradeLevels();

    // 각 사용자별로 업그레이드 후의 예상 레벨 검증
    checkLevel(users.get(0), Level.BASIC);
    checkLevel(users.get(1), Level.SILVER);
    checkLevel(users.get(2), Level.SILVER);
    checkLevel(users.get(3), Level.GOLD);
    checkLevel(users.get(4), Level.GOLD);
}

// DB에서 사용자 정보를 가져와 레벨을 확인하는 코드가 중복되어 헬퍼 메서드로 분리
private void checkLevel(User user, Level expectedLevel) {
    User userUpdate = userDao.get(user.getId());
    Assertions.assertEquals(userUpdate.getLevel(), expectedLevel);
}
```

### 5.1.4 UserService.add()
- 처음 가입하는 사용자는 기본적으로 BASIC 레벨이어야 한다는 요구사항을 충족시켜보자.

- 현재는 단순히, 받은 Level을 적용시키도록 하고 있다. 그렇다면만일 레벨 정보가 null이라면, Level.BASIC을 넣도록 할까? 그건 옳지 않을 것이다. UserDao는 온전히 데이터의 CRUD를 다루는 데만 치중하는 것이 옳고, 비즈니스 로직이 섞이는 것은 바람직하지 않다.

- 차라리 User 클래스에서 level 필드를 기본 값으로 Level.BASIC으로 초기화해보자. 하지만 처음 가입할 때를 제외하면 무의미한 정보인데, 단지 이 로직을 담기 위해 클래스에서 직접 초기화하는 것은 문제가 있어 보이긴 한다.

- 그렇다면 UserService에 이 로직을 넣으면 어떨까? UserDao의 add() 메소드는 사용자 정보를 담은 User 오브젝트를 받아서 DB에 넣어주는 데 충실한 역할을 한다면, UserService에도 add()를 만들어두고 사용자가 등록될 때 적용할만한 비즈니스 로직을 담당하게 하면 될 것이다.

- UserDao와 같이 리포지토리 역할을 하는 클래스를 컨트롤러에서 바로 쓰냐 마냐에 대한 논쟁이 있는데, 바로 쓰면 아무런 비즈니스 로직이 들어가지 않은 순수한 CRUD의 의미일 것이다.

- 먼저 테스트부터 만들어보자. UserService의 add()를 호출하면 레벨이 BASIC으로 설정되는 것이다. 그런데, UserService의 add()에 전달되는 User 오브젝트에 Level 값이 미리 설정되어 있다면, 설정된 값을 이용하도록 하자.

- 그렇다면 테스트 케이스는 두가지 종류가 나올 수 있다.

- 레벨이 미리 설정된 경우
  - 설정된 레벨을 따른다.
- 레벨이 미리 설정되지 않은 경우 (레벨이 비어있는 경우)
  - BASIC 레벨을 갖는다.

- 각각 add() 메소드를 호출하고 결과를 확인하도록 만들자.

- 가장 간단한 방법은 UserService의 add() 메소드를 호출할 때 파라미터로 넘긴 User 오브젝트에 level 필드를 확인해보는 것이고, 다른 방법은 UserDao의 get() 메소드를 이용해서 DB에 저장된 User 정보를 가져와 확인하는 것이다. 두가지 다 해도 좋고, 후자만 해도 괜찮을 것 같다.

- UserService는 UserDao를 통해 DB에 사용자 정보를 저장하기 때문에 이를 확인해보는 게 가장 확실한 방법이다. UserService가 UserDao를 제대로 사용하는지도 함께 검증할 수 있고, 디폴트 레벨 설정 후에 UserDao를 호출하는지도 검증되기 때문이다.

```java
// 5-21. add() 메서드의 테스트
@Test
public void add() {
  userDao.deleteAll();

  User userWithLevel = users.get(4); // GOLD 레벨
  User useWithoutLevel = users.get(0);
  userWithoutLevel.setLevel(null);

  userService.add(userWithLevel);
  userService.add(useWithoutLevel);

  User userWithLevelRead = userDao.get(userWithLevel.getId());
  User userWithoutLevelRead = userDao.get(userWithoutLevel.getId());

  assertThat(userWithLevelRead.getLevel(), is(userWithLevel.getLevel()));
  assertThat(userWithoutLevelRead.getLevel(), is(Level.BASIC));
```

```java
// 5-22. 사용자 신규 등록 로직을 담은 add() 메서드
public void add(User user) {
  if(user.getLevel() == null) user.setLevel(Level.BASIC);
  userDao.add(user);
}
```

### 5.1.5 코드 개선
어느정도 요구사항은 맞춰놨지만, 아직 코드가 깔끔하지 않게 느껴진다. 다음 사항들을 체크해보자.

- 코드에 중복된 부분은 없는가?
- 코드가 무엇을 하는 것인지 이해하기 불편하진 않은가?
- 코드가 자신이 있어야 할 자리에 있는가
- 앞으로 변경이 일어날 수 있는 건 어떤 것이며, 그 변화에 쉽게 대응할 수 있게 작성 되었는가?

#### upgradeLevels() 메서드 코드의 문제점
- for 루프 속에 들어있는 if/else 블록이 겹쳐 읽기 불편하다.
- 레벨의 변화 단계와 업그레이드 조건, 조건이 충족됐을 때 해야 할 작업이 섞여서 로직을 이해하기 어렵다.
- 플래그를 두고 이를 변경하고 마지막에 이를 확인해서 업데이트를 진행하는 방법도 그리 깔끔해보이지 않는다.
- 코드가 깔끔해보이지 않는 이유는 이렇게 성격이 다른 여러가지 로직이 섞여있기 때문이다.

- `user.getLevel() == Level.BASIC`은 레벨이 무엇인지 파악하는 로직이다.
- `user.getLoginCount() >= 50`은 업그레이드 조건을 담은 로직이다.
- `user.setLevel(Level.SILVER);`는 다음 단계의 레벨이 무엇인지와 레벨 업그레이드를 위한 작업은 어떤 것인지가 함께 담겨있다.
- `changed = true;`는 이 자체로는 의미가 없고, 단지 멀리 떨어져 있는` userDao.update(user);`의 작업이 필요함을 알려주는 역할이다.
- 잘 살펴보면 관련이 있지만, 사실 성격이 조금 다른 것들이 섞여있거나 분리돼서 나타나는 구조다.

- 변경될만한 것 추측하기 -> 사용자 레벨, 업그레이드 조건, 업그레이드 작업
  - 사용자 레벨이 변경되면?
    - 현재 if 조건 블록이 레벨 개수만큼 반복되고 있다. 새로운 레벨이 추가되면, Level ENUM도 수정해야 하고, upgradeLevels()의 레벨 업그레이드 로직을 담은 코드에 if 조건식과 블록을 추가해줘야 한다.
  - 업그레이드 작업이 변경되면?
    - 추후에 레벨을 업그레이드 작업에서 이를테면 레벨 업그레이드 축하 알람 등 새로운 작업이 추가되면, user.setLevel(다음레벨); 뒤에 추가적인 코드를 작성해주어야 할 것이다. 그러면 점점 메소드의 if문 블록은 커진다.
  - 업그레이드 조건이 변경되면?
    - 업그레이드 조건도 문제다. 새로운 레벨이 추가되면 기존 if조건과 맞지 않으니 else로 이동하는데, 성격이 다른 두 가지 경우가 모두 한 곳에서 처리되는 것은 뭔가 이상하다.
    - 업그레이드 조건이 계속 까다로워지면 마지막엔 if() 내부에 들어갈 내용이 방대하게 커질 수 있다.
   

- 아마 upgradeLevels() 코드 자체가 너무 많은 책임을 떠안고 있어서인지 전반적으로 변화가 일어날수록 코드가 지저분해진다는 것을 추측할 수 있다. 지저분할수록 찾기 힘든 버그가 숨어들어갈 확률이 높아질 것이다.


#### upgradeLevels() 리팩토링

```java
// 5-23. 기존 작업 흐름만 남겨둔 upgradeLevels()
 public void upgradeLevels() {
  List<User> users = userDao.getAll();

  for (User user : users) {
      if(canUpgradeLevel(user)) {
          upgradeLevel(user);
      }
  }
}
```
- 위는 upgradeLevels()에서 기본 작업 흐름만 남겨둔 코드이다. 이 코드는 한 눈에 읽기에도 사용자 정보를 받아서 레벨 업그레이드를 할 수 있으면 레벨 업그레이드를 한다. 명확하다.

- 이는 구체적인 구현에서 외부에 노출할 인터페이스를 분리하는 것과 마찬가지 작업을 코드에 한 것이다.

- 이제 인터페이스화된 메소드들을 하나씩 구현해보자.

```java
// 5-24. 업그레이드 가능 확인 메서드
private boolean canUpgradeLevel(User user) {
    Level currentLevel = user.getLevel();

    return switch(currentLevel) {
        case BASIC -> user.getLoginCount() >= 50;
        case SILVER -> user.getRecommendCount() >= 30;
        case GOLD -> false;
        default -> throw new IllegalArgumentException("Unknown Level: " + currentLevel);
    };
}
```
> canUpgradeLevel()의 요구사항은 해당 사용자에 대한 레벨 업그레이드 가능 여부를 확인하고 그 결과를 반환하는 것이다.

- switch문으로 레벨을 구분하고 각 레벨에 대한 업그레이드 조건을 체크하고 업그레이드가 가능한지에 따라 true/false를 반환해준다.

- 또, 등록되지 않은 레벨에 대해 메소드를 수행할 시에는 IllegalArgumentException이 발생하기 때문에 해당 등급에 대한 로직 처리를 하지 않았음을 쉽게 알 수 있다.

```java
// 5-25. 레벨 업그레이드 작업 메서드
private void upgradeLevel(User user) {
    Level currentLevel = user.getLevel();

    switch (currentLevel) {
        case BASIC -> user.setLevel(Level.SILVER);
        case SILVER -> user.setLevel(Level.GOLD);
        default -> throw new IllegalArgumentException("Can not upgrade this level: " + currentLevel);
    }

    userDao.update(user);
}
```
> upgradeLevel()의 요구사항은 해당 사용자에 대한 레벨 업그레이드를 진행하는 것이다.

- 위와 같이 작성하여 보기엔 깔끔해 보이지만, 여기서도 무언가 맘에 안드는 점이 있다.

- 업그레이드된 다음 레벨이 무엇인지 자체를 이 메소드가 독립적으로 정하고 있다.
- 업그레이드 후의 작업이 늘어난다면? 지금은 level 필드만을 손보지만, 나중에 포인트 같은 개념이 생겨서 레벨 업그레이드 보너스 포인트 같은 것을 증정해야된다고 생각해보자. case 문 뒤의 블록 내용이 많이 늘어날 것이다.
- 업그레이드 후의 레벨이 무엇인지 결정하는 책임은 Level ENUM이 갖는 것이 맞지 않을까? 레벨의 순서에 대한 책임을 UserService에게 위임하지 말자.

```java
// 업그레이드 순서를 담고 있도록 수정한 Level
public enum Level {
    // 초기화 순서를 3, 2, 1 순서로 하지 않으면 `SILVER`의 다음 레벨에 `GOLD`를 넣는데 에러가 발생한다.
    GOLD(3, null), SILVER(2, GOLD), BASIC(1, SILVER);

    private final int value;
    private final Level next;

    Level(int value, Level next) {
        this.value = value;
        this.next = next;
    }

    public Level nextLevel() {
        return next;
    }

    public int intValue() {
        return value;
    }

    public static Level valueOf(int value) {
        return switch (value) {
            case 1 -> BASIC;
            case 2 -> SILVER;
            case 3 -> GOLD;
            default -> throw new AssertionError("Unknown value: " + value);
        };
    }
}
```
- 위와 같이 업그레이드 순서에 대한 책임을 Level enum에 맡겼다. 이제 다음 레벨이 무엇인지 알고 싶다면, .nextLevel() 메소드를 출력해보면 된다. 이제 다음 단계의 레벨이 무엇인지 일일이 if문에 담아둘 필요가 없다.

- 이제 사용자 정보가 바뀌는 부분을 UserService 메소드에서 User로 옮겨보자. User는 사용자 정보를 담고 있는 단순한 자바빈이긴 하지만 User도 엄연히 자바 오브젝트이고 내부 정보를 다루는 기능이 있을 수 있다. UserService가 일일이 레벨 업그레이드 시에 User의 어떤 필드를 수정해야 하는지에 대한 로직을 갖고 있기 보다는 User에게 레벨 업그레이드를 해야 하니 정보를 변경하라고 요청하는 편이 낫다.

```java
// 5-27. User의 레벨 업그레이드 작업용 메서드
public void upgradeLevel() {
    Level nextLevel = this.level.nextLevel();

    if (nextLevel == null) {
       throw new IllegalStateException(this.level + "은 업그레이드가 불가능합니다.");
    } else {
        this.level = nextLevel;
    }
}
```
- UserService의 canUpgradeLevel() 메소드에서 업그레이드 가능 여부를 미리 판단해주긴 하지만, User 오브젝트를 UserService만 사용한다는 보장은 없으므로, 스스로 예외상황에 대한 검증 기능을 갖고 있는 편이 안전하다.

- Level enum은 다음 레벨이 없는 경우에는 nextLevel()에서 null을 반환한다. 따라서 이 경우에는 User의 레벨 업그레이드 작업이 진행돼서는 안되므로, 예외를 던져야 한다.

- 애플리케이션의 로직을 바르게 작성하면 이런 경우는 아예 일어나지 않겠지만, User 오브젝트를 잘못 사용하는 코드가 있다면 확인해줄 수 있으니 유용하다.

- User에 업그레이드 작업을 담당하는 독립적인 메소드를 두고 사용할 경우, 업그레이드 시 기타 정보도 변경이 필요해졌을 때, 그 장점이 무엇인지 알 수 있을 것이다. 이를테면 마지막으로 업그레이드 된 시점을 기록하고 싶다면, lastUpgraded 필드를 추가하고 this.lastUpgraded = new Date();와 같은 코드를 추가함으로써, 쉽게 동작을 더할 수 있다.

```java
// 5-28. 간결해진 upgradeLevel()
private void upgradeLevel(User user) {
    user.upgradeLevel();
    userDao.update(user);
}
```
#### User 테스트
```java
 @Test
public void upgradeLevel() {
    Level[] levels = Level.values();

    for (Level level : levels) {
        if(level.nextLevel() == null) continue;

        user.setLevel(level);
        user.upgradeLevel();
        Assertions.assertEquals(user.getLevel(), level.nextLevel());
    }
}

@Test
@DisplayName("예외 테스트 - 다음 레벨이 없는 레벨을 업그레이드 하는 경우")
public void cannotUpgradeLevel() {
    Level[] levels = Level.values();

    Assertions.assertThrows(IllegalStateException.class, () -> {
        for (Level level : levels) {
            if(level.nextLevel() != null) continue;

            user.setLevel(level);
            user.upgradeLevel();
        }
    });
}
```
#### UserServiceTest 개선
```java
@Test
public void upgradeLevels() {
    for (User user : users) {
        userDao.add(user);
    }

    userService.upgradeLevels();

    checkLevelUpgraded(users.get(0), false);
    checkLevelUpgraded(users.get(1), true);
    checkLevelUpgraded(users.get(2), false);
    checkLevelUpgraded(users.get(3), true);
    checkLevelUpgraded(users.get(4), false);
}

private void checkLevelUpgraded(User userOrigin, boolean upgraded) {
    User userUpdate = userDao.get(userOrigin.getId());
    Assertions.assertEquals(
            userOrigin.getLevel().nextLevel() == userUpdate.getLevel()
            , upgraded);
}
```

- 의미 없는 상수 중복 제거 및 상수의 도입
  ```java
  public class UserService {
      UserDao userDao;
  
      public static final int MIN_LOGIN_COUNT_FOR_SILVER = 50;
      public static final int MIN_RECOMMEND_COUNT_FOR_GOLD = 30;
  
      ...
  
      private boolean canUpgradeLevel(User user) {
          Level currentLevel = user.getLevel();
  
          return switch(currentLevel) {
              case BASIC -> user.getLoginCount() >= MIN_LOGIN_COUNT_FOR_SILVER;
              case SILVER -> user.getRecommendCount() >= MIN_RECOMMEND_COUNT_FOR_GOLD;
              case GOLD -> false;
              default -> throw new IllegalArgumentException("Unknown Level: " + currentLevel);
          };
      }
  }
  ```
- 상수를 사용하도록 만든 테스트
  ```java
   @Before
  public void setUp() {
      this.userDao = this.userService.userDao;
      userDao.deleteAll();
  
      users = Arrays.asList(
              new User("bumjin", "박범진", "p1", Level.BASIC, MIN_LOGIN_COUNT_FOR_SILVER - 1, 0)
              , new User("joytouch", "강명성", "p2", Level.BASIC, MIN_LOGIN_COUNT_FOR_SILVER, 0)
              , new User("erwins", "신승한", "p3", Level.SILVER, MIN_LOGIN_COUNT_FOR_SILVER, MIN_RECOMMEND_COUNT_FOR_GOLD - 1)
              , new User("madnite1", "이상호", "p4", Level.SILVER, MIN_LOGIN_COUNT_FOR_SILVER, MIN_RECOMMEND_COUNT_FOR_GOLD)
              , new User("green", "오민규", "p5", Level.GOLD, 100, 100)
      );
  }
  ```
- 업그레이드 정책 인터페이스
  ```java
  public interface UserLevelUpgradePolicy {
    boolean canUpgradeLevel(User user);
    void upgradeLevel(User user);
  }
  ```


## 5.2 트랜잭션 서비스 추상화
> "정기 사용자 레벨 관리 작업을 수행하는 도중에 네트워크가 끊기거나 서버에 장애가 생겨서 작업을 완료할 수 없다면, 그때까지 변경된 사용자 레벨은 그대로 둘까요? 아니면 모두 초기 상태로 되돌려 놓아야 할까요?"

=> 그때까지 진행된 변경 작업도 모두 취소시키자. 일부는 조정이 됐는데 일부는 조정되지 않았다면 반발이 심할 것이기 때문
### 5.2.1 모 아니면 도
- 지금까지 만든 사용자 레벨 업그레이드 코드는 예외가 발생해 중단되면 어떨게 될까? 테스트를 만들어서 확인해보자.
- 짧은 업그레이드 작업 중간에 강제적인 장애 상황 연출이 불가능하기 때문에 예외가 던져지는 상황을 의도적으로 만들어보자.

#### 테스트용 UserService 대역
- 테스트를 위해 코드를 함부로 건드리는 건 좋은 생각이 아니다.
- 이런 경우엔 테스트용으로 특별히 만든 UserService 대역을 사용하는 방법이 좋다.
  => UserService를 상속해서 테스트에 필요한 기능을 추가하도록 일부 메서드를 오버라이딩 하자
- 현재 UserService 메서드의 대부분은 private -> 이번만 예외로 애플리케이션 코드를 수정하자
- upgradeLevel() 메서드를 오버라이딩하자
```java
protected void upgradeLevel(User user) { ... }
```
```
// 5-34. UserService의 테스트용 대역 클래스
static class TestUserService extends UserService {
  private String id;

  private TestUserService(String id) { // 예외를 발생시킬 User 오브젝트의 id를 지정할 수 있게 만든다.
    this.id = id;
  }

  protected void upgradeLevel(User user) {
    if(user.getId().equals(this.id)) throw new TestUserServiceException();
    super.upgradeLevels(user); // 지정된 id의 User 오브젝트가 발견되면 예외를 던져서 작업을 강제로 중단시킨다.
  }
}
```
- 다른 예외가 발생했을 경우 구분하기 위해 테스트 목적을 띤 예외를 다음과 같이 정의해둔다.
```java
static class TestUserServiceException extends RuntimeException {
}
```

#### 강제 예외 발생을 통한 테스트
- 레벨 업그레이드를 시도하다가 중간에 예외가 발생했을 경우 그 업그레이드 했던 사용자도 다시 원래 상태로 돌아갔는지 확인하는 테스트를 만들어보자.

```java
// 5-36. 예외 발생 시 작업 취소 여부 테스트
@Test
public void upgradeAllOrNothing() {
  // 예외를 발생시킬 네 번째 사용자의 id를 넣어서 테스트용 UserService 대역 오브젝트를 생성한다.
  UserService testUserService = new TestUserService(users.get(3).getId());
  TestUserService setUserDao(this.userDao);
  userDao.deleteAll();
  for(User user : users) userDao.add(user);

  try {
    testUserService.upgradeLevels();
    fail("TestUserServiceException expected");
  }
  catch(TestUserServiceException e) { // TestUserService가 던져주는 예외를 잡아서 계속 진행되도록 한다. 그 외의 예외라면 테스트 실패
  }

  // 예외가 발생하기 전에 레벨 변경이 있었던 사용자의 레벨이 처음 상태로 바뀌었나 확인
  checkLevelUpgraded(users.get(1), false);

  }
}
```

#### 테스트 실패의 원인
- 테스트 실패 원인 : 트랜잭션 문제
- 사용자가 레벨을 업그레이드하는 upgradeLevels() 메서드가 하나의 트랜잭션 안에서 동작하지 않았기 때문
> 트랜잭션 : 더 이상 나눌 수 없는 단위 작업
- 예외가 발생해서 작업을 완료할 수 없다면 아예 작업이 시작되지 않은 것처럼 초기 상태로 돌려놔야 한다.

### 5.2.2 트랜잭션 경계 설정
- DB는 그 자체로 완벽한 트랜잭션 지원
- 하지만 여러 개의 SQL이 사용되는 작업을 하나의 트랜잭션으로 취급해야 하는 경우도 있다. (ex. 계좌 이체, 사용자 레벨 수정 작업)
- 두 가지 작업이 하나의 트랜잭션이 되려면, 두 번째 SQL이 성공적으로 DB에서 수행되기 전에 문제가 발생할 경우에는 앞에서 처리한 SQL 작업도 취소시켜야 한다. => `트랜잭션 롤백`
- 반대로 여러 개의 SQL을 하나의 트랜잭션으로 처리하는 경우에 모든 SQL 수행 작업이 다 성공적으로 마무리 됐다고 DB에 알려줘서 작업을 확정시켜줘야 한다. => `트랜잭션 커밋`

#### JDBC 트랜잭션의 트랜잭션 경계설정
- 모든 트랜잭션은 시작하는 지점과 끝나는 지점이 있다
- 시작하는 방법은 한 가지지만 끝나는 방법은 두 가지 => 롤백과 커밋
- 애플리케이션 내에서 트랜잭션이 시작되고 끝나는 위치를 트랜잭션의 **경계**라고 부른다.

```java
// 5-37. 트랜잭션을 사용한 JDBC 코드
Connection c = dataSource.getConnection();

c.setAutoCommit(false); // 트랜잭션 시작
try {
  // 하나의 트랜잭션으로 묶인 단위 작업
  // =========================
  PreparedStatement st1 = c.prepareStatement("update users ...");
  st1.executeUpdate();

  PreparedStatement st2 = c.prepareStatement("delete users ...");
  st2.executeUpdate();
  // =========================
  c.commit(); // 트랜잭션 커밋
} catch(Exception e) {
  c.rollback(); // 트랜잭션 롤백
}
c.close();
```
- JDBC의 트랜잭션은 위처럼 `Connection` 객체를 통해 일어난다.
- `c.setAutoCommit(false)`를 호출하는 순간 트랜잭션 경계가 시작되며, `c.commit()` 혹은 `c.rollback()`을 호출하는 순간 트랜잭션 경계가 끝난다.
- autoCommit의 기본 값은 true여서 원래는 작업마다 커밋이 자동으로 이뤄지는데 이 설정값을 false로 만듦으로써 커밋을 수동으로 이뤄지게 만들어 `commit()` 혹은 `rollback()`으로 끝내는 원리이다.

- 이렇게 트랜잭션 영역을 만드는 일을 **트랜잭션 경계 설정**이라고 한다.
- 이렇게 **하나의 DB 커넥션** 안에서 만들어지는 트랜잭션을 **로컬 트랜잭션(local transaction)**이라고도 한다. **2개 이상의 DB**에서 만들어지는 트랜잭션은 **글로벌 트랜잭션(global transaction)**이라고 한다.


#### UserService와 UserDao 트랜잭션 문제
- 이제 트랜잭션 문제는 해결했지만 여러가지 새로운 문제가 발생하게 된다.

- DB커넥션을 비롯한 리소스의 깔끔한 처리를 가능하게 했던 `JdbcTemplate`을 더이상 활용할 수 없다. 결국 JDBC API를 직접 사용하는 초기 방식으로 돌아가야 한다.
  - `try/catch/finally` 블록은 이제 UserService 내에 존재하고 UserService의 코드는 JDBC 작업 코드의 전형적인 문제점을 그대로 가질 수 밖에 없다.
- DAO의 메소드와 비즈니스 로직을 담고 있는 UserService의 메소드에 Connection 파라미터가 추가돼야 한다는 점이다.
  - `upgardeLevels()`에서 사용하는 메소드의 어딘가에서 DAO를 필요로 한다면, 그 사이의 모든 메소드에 걸쳐서 Connection 오브젝트가 계속 전달돼야 한다.
  - UserService는 스프링 빈으로 선언해서 싱글톤으로 되어 있으니 UserService의 인스턴스 변수에 이 Connection을 저장해뒀다가 다른 메소드에서 사용하게 할 수도 없다.
  - 멀티 스레드 환경에서는 공유하는 인스턴스 변수에 스레드별로 생성하는 정보를 저장하다가는 서로 덮어쓰는 일이 발생하기 때문이다. 결국 트랜잭션이 필요한 작업에 참여하는 UserService의 메소드는 Connection 파라미터로 지저분해질 것이다.
- Connection 파라미터가 UserDao 인터페이스 메소드에 추가되면 UserDao는 더이상 데이터 엑세스 기술에 독립적일 수 없다는 것이다.
  - JPA나 하이버네이트로 UserDao의 구현 방식을 변경하려고 하면 Connection 대신 `EntityManage`r나 `Session` 오브젝트를 UserDao 메소드가 전달받도록 해야 한다.
  - 결국 UserDao 인터페이스는 바뀔 것이고 그에 따라 UserService 코드도 함께 수정돼야 한다. 기껏 인터페이스를 사용해 DAO를 분리하고 DI를 적용했던 수고가 물거품이 되고 말 것이다.
- DAO 메소드에 Connection 파라미터를 받게 하면 테스트코드에도 영향을 미친다.
  - 지금까지 DB 커넥션은 전혀 신경쓰지 않고 테스트에서 UserDao를 사용할 수 있었는데, 이제는 테스트 코드에서 직접 Connection 오브젝트를 일일이 만들어서 DAO 메소드를 호출하도록 모두 변경해야 한다.
 
### 5.2.3 트랜잭션 동기화
- UserService 메소드 안에서 트랜잭션 코드를 구현하며 위와 같은 문제점을 감내할 수 밖에 없을까?
- 스프링은 사실 이 문제를 해결할 수 있는 멋진 방법을 제공한다.

#### Connection 파라미터 제거
- 현재까지 문제의 핵심은 UserService에서 Connection 객체를 만들어서 해당 객체를 2번이나 전달하느라 코드가 어지럽혀졌고 그 영향이 심지어 테스트코드까지 미쳤다는 것이다.

- 이런 문제를 해결하기 위해 스프링이 제안하는 방법은 독립적인 트랜잭션 동기화(transaction synchronization) 방식이다.
- **트랜잭션 동기화**란 UserService에서 트랜잭션을 시작하기 위해 만든 `Connection` 오브젝트를 특별한 장소에 보관해두고, 이후에 호출되는 DAO의 메소드에서는 저장된 Connection을 가져다가 사용하게 하는 것이다.
- 정확히는 DAO가 사용하는 JdbcTemplate이 트랜잭션 동기화 방식을 이용하도록 하는 것이다. 그리고 트랜잭션이 모두 종료되면 그 때는 동기화를 마치면 된다.

- 다음은 트랜잭션 동기화 방식을 사용한 경우의 작업 흐름을 보여준다.

  <img width="579" alt="스크린샷 2024-06-16 오후 11 54 40" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/c9a27543-acee-4deb-8b38-5f605a727b8b">

- (1): UserService가 Connection을 생성한다.
- (2): 생성한 Connection을 트랜잭션 동기화 저장소에 저장한다. 이후에 Connection의 setAutoCommit(false)를 호출해 트랜잭션을 시작시킨다.
- (3): 첫 번째 update() 메소드를 호출한다.
- (4): update() 메소드 내부에서 이용하는 JdbcTemplate은 트랜잭션 동기화 저장소에 현재 시작된 트랜잭션을 가진 Connection 오브젝트가 존재하는지 확인한다. ((2) 단계에서 만든 Connection 오브젝트를 발견할 것이다.)
- (5): 발견한 Connection을 이용해 PreparedStatement를 만들어 SQL을 실행한다. 트랜잭션 동기화 저장소에서 DB 커넥션을 가져왔을 때는 JdbcTemplate은 Connection을 닫지 않은채로 작업을 마친다. 이렇게 첫번째 DB 작업을 마쳤고, 트랜잭션은 아직 닫히지 않았다. 여전히 Connection은 트랜잭션 동기화 저장소에 저장되어 있다.
- (6): 동일하게 userDao.update()를 호출한다.
- (7): 트랜잭션 동기화 저장소를 확인하고 Connection을 가져온다.
- (8): 발견된 Connection으로 SQL을 실행한다.
- (9): userDao.update()를 호출한다.
- (10): 트랜잭션 동기화 저장소를 확인하고 Connection을 가져온다.
- (11): 가져온 Connection으로 SQL을 실행한다.
- (12): Connection의 commit()을 호출해서 트랜잭션을 완료시킨다.
- (13): Connection을 제거한다.

위 과정 중 예외가 발생하면, commit()은 일어나지 않고 트랜잭션은 rollback()된다.

- 이렇게 트랜잭션 동기화 기법을 사용하면 파라미터를 통해 일일이 Connection 오브젝트를 전달할 필요가 없어진다. 트랜잭션의 경계설정이 필요한 upgradeLevels()에서만 Connection을 다루게 하고 여기서 생성된 Connection과 트랜잭션을 DAO의 JdbcTemplate이 사용할 수 있도록 별도의 저장소에 동기화하는 방법을 적용하기만 하면 된다.

- 더이상 로직을 담은 메소드에 Connection 타입의 파라미터가 전달될 필요도 없고, UserDao의 인터페이스에도 일일이 JDBC 인터페이스인 Connection을 사용한다고 노출할 필요도 없다.

- 문제의 핵심은 트랜잭션을 이용하기 위해 Connection이라는 파라미터를 귀찮게 2단계나 전달해야 했다는 것이다. 그리고 이 과정에서 JdbcTemplate을 이용할 수 없게 되고 기존 try/catch/finally 방식의 단점이 그대로 다시 돌아왔었다.

- 결국 Connection을 다른 저장소에 저장해두고 쓰는 방식이 필요했는데, 멀티쓰레드 환경이라는 제약 조건과 UserService가 빈이라는 제약 조건이 있었다.

- 스프링의 트랜잭션 동기화 저장소는 작업 스레드마다 독립적으로 Connection 오브젝트 저장/관리 환경을 제공함으로써 이러한 문제를 해결했다.

#### 트랜잭션 동기화 적용
- 트랜잭션 동기화의 아이디어 자체는 그냥 글로벌한 공간에 트랜잭션을 잠시 저장해둔다는 것으로 간단하지만, 멀티스레드 환경에서도 안전하게 트랜잭션 동기화를 구현하는 것이 기술적으로 간단하지는 않다.
- 다행히 스프링은 JdbcTemplate과 더불어 이런 트랜잭션 동기화 기능을 지원하는 간단한 유틸리티 메소드를 제공한다.

```java
public class UserService {
    UserDao userDao;
    DataSource dataSource;
    ...
```
```xml
<bean id="userService" class="toby_spring.user.service.UserService">
    <property name="userDao" ref="userDao" />
    <property name="dataSource" ref="dataSource" />
    <property name="userLevelUpgradePolicy" ref="userLevelUpgradePolicy" />
</bean>
```
```java
public void upgradeLevels() throws SQLException{
    // 트랜잭션 동기화 관리자를 이용해 동기화 작업을 초기화
    TransactionSynchronizationManager.initSynchronization();
    // DB 커넥션을 생성하고 트랜잭션을 시작한다.
    // 이후의 DAO 작업은 모두 여기서 시작한 트랜잭션 안에서 진행된다.
    // 아래 두 줄이 DB 커넥션 생성과 동기화를 함께 해준다.
    Connection c = DataSourceUtils.getConnection(dataSource);
    c.setAutoCommit(false);

    try {
        List<User> users = userDao.getAll();
        for (User user : users) {
            if (canUpgradeLevel(user)) {
                upgradeLevel(user);
            }
        }

        c.commit();
    }catch(Exception e) {
        c.rollback();
        throw e;
    } finally {
        // 스프링 DataSourceUtils 유틸리티 메소드를 통해 커넥션을 안전하게 닫는다.
        DataSourceUtils.releaseConnection(c, dataSource);
        // 동기화 작업 종료 및 정리
        TransactionSynchronizationManager.unbindResource(this.dataSource);
        TransactionSynchronizationManager.clearSynchronization();
    }
}
```
- upgradeLevels에 위와 같은 트랜잭션 처리를 해주었다. 스프링을 사용하지 않고 JDBC를 이용해 Connection 객체를 직접 쓸 때와 다른 점은

  - 첫째로 트랜잭션 동기화 관리(TransactionSynchronizationManager)를 이용한다는 점
  - 둘째로는 커넥션을 가져올 때나 반납할 때 DataSourceUtils라는 스프링 제공 유틸리티를 사용한다는 점
  두가지가 있다.

- 더이상 DataSource.getConnection()을 이용해 Connection을 그냥 가져오지 않는 이유는 DataSourceUtils를 이용해 커넥션을 가져오고 setAutoCommit(false) 메소드를 수행하면, DB 커넥션 생성과 트랜잭션 동기화에 사용하도록 저장소에 바인딩해주기 때문이다.

- 트랜잭션 동기화가 되어 있는 채로 JdbcTemplate을 사용하면 JdbcTemplate의 작업에서 동기화시킨 DB 커넥션을 사용하게 된다. 결국 UserDao를 통해 진행되는 모든 JDBC 작업은 upgradeLevels() 메소드에서 만든 Connection 오브젝트를 사용하고 같은 트랜잭션에 참여하게 된다.

- 작업을 정상적으로 마치면 트랜잭션을 커밋해주고, 예외가 발생하면 롤백한다. 마지막으로는 커넥션을 안전하게 반환하고, 동기화 작업에 사용됐던 부분들을 바인드 해제한다.

- JDBC의 트랜잭션 경계설정 메소드를 사용해 트랜잭션을 이용하는 전형적인 코드에 간단한 트랜잭션 동기화 작업만 붙여줌으로써, 지저분한 Connection 파라미터의 문제를 말끔히 해결했다.

한 템플릿을 제공하여, 개발자가 비즈니스 로직에 집중할 수 있고 애플리케이션 레이어를 설계하기 좋은 환경을 만들어준다.

귀찮게 Connection 파라미터를 물고다니지 않아도 된다. 또한, UserDao는 여전히 데이터 액세스 기술에 종속되지 않는 깔끔한 인터페이스 메소드를 유지한다. 그리고 테스트에서 DAO를 직접 호출해서 사용하는 것도 아무런 문제가 되지 않는다.

### 5.2.4 트랜잭션 서비스 추상화

#### 기술과 환경에 종속되는 트랜잭션 경계 설정 코드
- 새로운 문제 상황 : 이 사용자 관리 모듈을 구매해서 사용하기로 한 G 사에서 새로운 요구가 들어옴
  - G 사는 여러 개의 DB를 사용하고 있음. 그래서 하나의 트랜잭션 안에서 여러 개의 DB에 데이터를 넣는 작업 필요
  - 한 개 이상의 DB로의 작업을 하나의 트랜잭션으로 만드는 것은 JDBC의 Connection을 사용한 방식(로컬 트랜잭션)으로는 불가능
  - 로컬 트랜잭션은 하나의 DB Connection에 종속되기 때문
  => 별도의 트랜잭션 관리자를 통해 트랜잭션을 관리하는 **글로벌 트랜잭션 방식** 사용

- 자바는 글로벌 트랜잭션을 지원하는 트랜잭션 매니저를 지원하기 위한 API인 JTA(Java Transaction API)를 제공하고 있음

<img width="581" alt="스크린샷 2024-06-18 오후 1 56 16" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/139430c7-d184-4cb8-bd7d-cb238e4c9015">

- 애플리케이션에서는 기존의 방법대로 DB는 JDBC, 메시징 서버라면 JMS 같은 API를 사용해서 필요한 작업 수행
  - 단 트랜잭션은 JDBC나 JMS API를 사용해서 직접 제어하지 않고 JTA를 통해 트랜잭션 매니저가 관리하도록 위임
- 트랜잭션 매니저는 DB와 메시징 서버를 제어하고 관리하는 각각의 리소스 매니저와 XA 프로토콜을 통해 연결

- JTA를 사용한 트랜잭션의 처리코드의 전형적인 구조는 다음과 같다.
  ```java
  InitialContext ctx = new InitialContext();
  UserTransaction tx = (UserTransation)ctx.lookup(USER_TX_JNDI_NAME);

  tx.begin();
  Connection c = dataSource.getConnection(); // JNDI로 가져온 dataSource를 사용해야 한다.
  try {
    // 데이터 액세스 코드
    tx.commit();
  } catch (Exception e) {
    tx.rollback();
    throw e;
  } finally {
    c.close();
  }
  ```
  - 트랜잭션 경계 설정을 위한 구조는 JDBC를 사용했을 때와 비슷
    - `Connection` 메서드 대신에 `UserTransaction`의 메서드를 사용한다는 점을 제외하면 트랜잭션 처리 방법은 별로 달라진 게 없음
  - 문제는 JDBC 로컬 트랜잭션을 JTA를 이용하는 글로벌 트랜잭션으로 바꾸려면 UserService의 코드를 수정해야 함
  - 그런데 로컬 트랜잭션을 사용하면 충분한 고객을 위해서는 JDBC를 이용한 트랜잭션 관리코드를, G 사처럼 다중 DB를 위한 글로벌 트랜잭션을 필요로 하는 곳을 위해서는 JTA를 이용한 코드를 적용해야 함 => UserService 로직이 기술 환경에 따라서 바뀐다
- 그러던 와중... Y 사에서는 하이버네이트를 이용해 UserDato를 직접 구현했다고 알려줬다. UserService와 UserDao는 DI를 통해 연결되어 있기 때문에 UserService를 수정하지 않고도 UserDao의 데이터 액세스 기술은 얼마든지 변경이 가능하다는 점을 영리하게 활용한 것
- 그런데 문제는 하이버네이트를 이용한 트랜잭션 관리코드는 JDBC와 JTA의 코드와는 또 다르다. (Connection 대신 Session 사용, 독자적인 트랜잭션 관리 API 사용) => UserService 로직을 변경해야 함

#### 트랜잭션 API의 의존관계 문제와 해결책
- UserService에 트랜잭션 경계설정 코드를 도임한 후의 클래스 의존관계
  <img width="566" alt="스크린샷 2024-06-18 오후 2 13 19" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/ef23f830-890c-4d1a-af30-c860e1f032f2">
- 원래 UserService는 UserDao 인터페이스에만 의존하는 구조였는데, 트랜잭션 코드가 UserService에 등장하면서부터 간접적으로 의존하는 코드가 되어버림
- 어떻게 해야할까?
  - 트랜잭션 경계 코드를 제거할 수는 없음
  - 하지만 특정 기술에 의존적인 Connection, UserTransaction, Session/Transation API 등에 종속도ㅚ지 않게 하는 방법은 있음
  - 트랜잭션의 경계 설정을 담당하는 코드는 일정한 패턴을 갖는 유사한 구조 -> 추상화 가능
- 트랜잭션 처리 코드에서 추상화를 도입해보자

#### 스프링과 트랜잭션 서비스 추상화
- 스프링은 트랜잭션 기술의 공통점을 담은 트랜잭션 추상화 기술을 제공하고 있음
- 스프링이 제공하는 트랜잭션 추상화 계층 구조
  <img width="574" alt="스크린샷 2024-06-18 오후 2 20 44" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/ff4167ef-7b7d-4237-9cc6-9e0345a64ccb">

- 트랜잭션 추상화 방법을 UserService에 적용한 코드
  ```java
  public void upgradeLevels() {
    PlatformTransactionManager transactionManager = new DataSourceTransactionManager(dataSource); // JDBC 트랜잭션 추상 오브젝트 생성

    // 트랜잭션 시작
    TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());
  try {
    // 트랜잭션 안에서 진행되는 작업
    // =================================
    List<User> users = userDao.getAll();
    for(User user : users) {
      if(canUpgradeLevel(user)) {
        upgradeLevel(user);
      }
    }
    // =================================
    transactionManager.commmit(status); // 트랜잭션 커밋
  } catch (RuntimException e) {
    transactionManager.rollback(status); // 트랜잭션 커밋
    throw e;
  }
}
```
- PlatformTransactionManager에서는 트랜잭션을 가져오는 요청인 getTransaction() 메서드를 호출하여 트랜잭션을 시작할 수 있다.(일단은 트랜잭션을 가져온다는 것을 시작한다는 의미라고 생각하자.)
- DefaultTransactionDefinition 오브젝트는 트랜잭션에 대한 속성을 담고 있다.
- 이렇게 시작된 트랜잭션은 TransactionStatus 타입의 변수에 저장됨
- 트랜잭션에 대한 조작이 필요할 때 PlatformTransactionManager 메서드의 파라미터로 전달해주면 된다.

#### 트랜잭션 기술 설정의 분리
- 트랜잭션 추상화 API를 적용한 UserService 코드를 글로벌 트랜잭션으로 변경해보자.
- `PlatformTransactionManager` 구현 클래스를 `DataSourceTransactionManager`에서 `JTATransactionManager`로 바꿔주기만 하면 된다.
 - `JTATransactionManager` : 주요 자바 서버에서 제공하는 JTA 정보를 JNDI를 통해 자동으로 인식하는 기능을 갖고 있다. 따라서 별다른 설정 없이 이를 사용하기만 해도 서버의 트랜잭션 매니저/서비스와 연동해서 동작한다.

```java
PlatformTransactionManager txManager = new JTATransactionManager();
```

- 하디만 어떤 트랜잭션 매니저 구현 클래스를 사용할지 UserService가 알고 있음 -> DI 원칙 위배
=> 컨테이너를 통해 외부에서 제공받게 하는 스프링 DI 방식으로 바꾸자.

```java
// 5-46. 트랜잭션 매니저를 빈으로 분리시킨 UserService
public class UserService {
  ...
  private PlatformTransactionManager transactionManager;

  public void setTransactionManger(PlatformTransactionManager transactionManager) {
    // 프로퍼티 이름은 관례를 따라 transactionManager라고 만드는 것이 편리
    this.transactionManager = transactionManager;
  }

  public void upgradeLevels() {
    TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
    try {
      List<User> users = userDao.getAll();
      ... 생략 ...
    }
    this.transactionManager.commit(status);
  } catch (RuntimeException e) {
    this.transactionManager.rollback(status);
    throw e;
  }
}
```
```xml
// 5-47. 트랜잭션 매니저 빈을 등록한 설정 파일
<bean id="userService" class="springbook.user.service.UserService">
  <property name="userDao" ref="userDao" />
  <property name="transactionManager ref="transactionManager" />
</bean>

<bean id="transactionManager"
  class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
  <property name="dataSource" ref="dataSource" />
</bean>
```
```java
// 5-48. 트랜잭션 매니저를 수동 DI 하도록 수정한 테스트
public class UserServiceTest {
  @Autowired
  PlatformTransactionManager transactionManager;

  @Test
  public void upgradeAllOrNothing() throws Exception {
    ...
    // userService 빈의 프로퍼티 설정과 동일한 수동 DI
    testUserService.setUserDao(userDao);
    testUserService.setTransactionManager(transactionManager);
    ...
  }
}
```
- 트랜잭션을 JTA를 이용하는 것으로 바꾸고 싶다면 설정 파일의 transactionManager 빈의 설정만 다음과 같이 고치면 된다.
  ```xml
  <bean id="transactionManager"
    class="org.springframework.transaction.jta.JtaTransactionManager" />
  ```

## 5.3 서비스 추상화와 단일 책임 원칙
- 이렇게 기술과 서비스에 대한 추상화 기법을 이용하면 특정 기술환경에 종속되지 않는 포터블한 코드를 만들 수 있다.
### 수직, 수평 계층 구조와 의존관계
- UserDao와 UserService는 같은 애플리케이션 로직을 담은 코드지만 내용에 따라 분리 -> 같은 계층에서 수평적인 분리
- 트랜잭션 추상화는 애플리케이션의 비즈니스 로직과 그 하위에서 동작하는 로우 레벨의 트랜잭션 기술이라는 아예 다른 계층의 특성을 갖는 코드를 분리 -> 수직적인 분리
<img width="592" alt="스크린샷 2024-06-19 오전 11 14 52" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/902c2547-56f4-461f-b1ee-b9b3b5d71823">
- 애플리케이션 로직의 종류에 따른 수평적인 구분이든, 로직과 기술이라는 수직적인 구분이든 모두 결합도가 낮으며, 서로 영향을 주지 않고 자유롭게 확장될 수 있는 구조를 만들 수 있는 데는 스프링의 DI가 중요한 역할을 하고 있다.

#### 단일 책임 원칙
- 이러한 적절한 분리가 가져오는 특징은 SRP(단일 책임 원칙)으로 설명 가능
  - 하나의 모듈이 바뀌는 이유는 한 가지여야 한다.
- 어떻게 사용자 레벨을 관리할 것인가와 어떻게 트랜잭션을 관리할 것인가라는 두 가지 책임을 갖고 있었던 UserService 코드에 트랜잭션 서비스의 추상화 방식을 도입하고, 이를 DI를 통해 외부에서 제어하도록 만들 나서는 UserService가 바뀔 이유는 한 가지가 됨.
=> 단일 책임 원칙을 충실하게 지키고 있다.

#### 단일 책임 원칙의 장점
- 어떤 변경이 필요할 때 수정 대상이 명확해진다.
- 적절하게 책임과 관심이 다른 코드를 분리하고, 서로 영향을 주지 않도록 다양한 추상화 기법을 도입하고 애플리케이션 로직과 기술 / 환경을 분리하는 등의 작업은 갈수록 복잡해지는 엔터프라이즈 애플리케이션에는 반드시 필요

> 패턴이나 설계 원칙을 공부하는 이유는 폼나는 용어를 외우고 기계적인 지식을 습득하면 저절로 깔끔하고 유연한 코드가 나오기 때문이 아니다. 좋은 코드를 만들기 위한 개발자 스스로의 노력과 고민이 있을 때 도움을 주기 때문이다.

## 5.4 메일 서비스 추상화
