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
// 추가된 필드를 위한 UserDaoJdbc의 수정 코드
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
- 먼저 매핑을 위와 같이 수정해준다. 이제 DB에서 데이터를 불러왔을 때 User 객체에는 잘 반영될 것이다.

