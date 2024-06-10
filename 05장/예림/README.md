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
@Test
public void upgradeAllOrNothing() {
  UserService testUserService = new TestUserService(users.get(3).getId());
  TestUserService setUserDao(this.userDao);
  userDao.deleteAll();
  for(User user : users) userDao.add(user);

  try {
    testUserService.upgradeLevels();
    fail("TestUserServiceException expected");
  }
  catch(TestUserServiceException e) {
  }

  checkLevelUpgraded(users.get(1), false);

  }
}
```
