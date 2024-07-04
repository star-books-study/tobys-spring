# 06장. AOP
- AOP를 바르게 이용하려면 OOP를 대체하려고 하는 것처럼 보이는 AOP라는 이름 뒤에 감춰진, 그 필연적인 등장배경과 스프링이 그것을 도입한 이유, 그 적용을 통해 얻을 수 있는 장점이 무엇인지에 대한 충분한 이해가 필요하다.

- 서비스 추상화를 통해 많은 근본적인 문제를 해결했던 트랜잭션 경계설정 기능을 AOP를 이용해 더욱 세련되고 깔끔한 방식으로 바꿔보자


## 6.1 트랜잭션 코드의 분리
- 트랜잭션 경계 설정을 위해 넣은 코드 때문에 UserService 코드가 깔끔하지 않다.

### 6.1.1 메서드 분리
```java
// 6-1. 트랜잭션 경계설정과 비즈니스 로직이 공존하는 메서드
public void upgradeLevels() throws Exception {
    TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
    
    try {
        List<User> users = userDao.getAll();
        for (User user : users) {
            if (canUpgradeLevel(user)) {
                upgradeLevel(user);
            }
        }
        
        this.transactionManager.commit(status);
    } catch (Exception e) {
        this.transactionManager.rollback(status);
        throw e;
    }
}
```
- UserService의 트랜잭션 경계 설정 코드와 비즈니스 로직 코드가 복잡하게 얽혀 있는 듯 보이지만, 두 코드는 성격이 다를 뿐 아니라 서로 주고받는 정보가 없는, 완벽하게 독립적인 코드다.
- 비즈니스 로직을 담당하는 코드를 메서드로 추출해서 독립시켜보자.
```java
private void upgradeLevelsInternal() { 
    List<User> users = userDao.getAll(); 
    for (User user : users) {
        if (canUpgradeLevel(user)) { 
            upgradeLevel(user);
        } 
    }
}
```

### 6.1.2 DI를 이용한 클래스의 분리
- UserService 안의 트랜잭션 코드가 버젓이 UserService 안에 자리잡고 있는 것이 마음에 안든다.
- 간단하게 트랜잭션 코드를 클래스 밖으로 뽑아내면 보이지 않게 할 수 있지 않을까?

#### DI 적용을 이용한 트랜잭션 분리
- 트랜잭션 코드를 UserService 밖으로 빼저리면 UserService의 클라이언트 코드에서는 트랜잭션 기능이 빠진 UserService를 사용하게 된다.
- 구체적인 구현 클래스를 직접 참조하는 경우의 전형적인 단점이다.
- UserService 클래스와 그 사용 클라이언트인 UserServiceTest는 현재 강한 결합도로 고정되어 있다.

<img width="354" alt="스크린샷 2024-06-24 오후 5 06 26" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/9a682bcf-bc12-41ae-942b-68692eab0936">

- 다음과 같이 UserService를 인터페이스로 만들고 UserService의 구현 클래스를 만들어넣도록 하면 클라이언트와 결합이 약해지고, 직접 구현 클래스에 의존하고 있지 않기 때문에 유연한 확장이 가능해진다.
<img width="348" alt="스크린샷 2024-06-24 오후 5 09 45" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/bdf14d95-543c-4bce-a9b0-a003779a14e9">

- 한 번에 두 개의 UserService 인터페이스 구현 클래스를 동시에 이용한다면 어떨까? UserService를 구현한 또 다른 구현 클래스를 만드는 것이다.
- 이 클래스는 단지 트랜잭션 경계 설정이라는 책임을 맡고 있을 뿐이다.
- 그리고 또 다른 UserService 구현 클래스에 실제적인 로직 처리 작업은 위임하는 것이다.

<img width="566" alt="image" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/e5d3202b-ee9f-43be-812c-08e682acc682">

#### UserService 인터페이스 도입
- 기존의 UserService 클래스명을 UserServiceImpl로 변경하고 클라이언트가 사용할 로직을 담은 핵심 메서드만 UserService 인터페이스로 만들기
```java
// 6-3. UserService 인터페이스
public interface UserService {
    void add(User user);
    void upgradeLevels();
}
```
```java
// 7-4. 트랜잭션 코드를 제거한 UserService 구현 클래스
...
public class UserServiceImpl implements UserService {
    UserDao userDao;
    MailSender mailSender;

    public void upgradeLevels() {
        List<User> users = userDao.getAll();
        for(User user : users) {
            if(canUpdateLevel(user)) {
                upgradeLevel(user);
            }
        }
    }
}
```
#### 분리된 트랜잭션 기능
```java
// 6-5. 위임 기능을 가진 UserServiceTx 클래스
...
public UserServiceTx implements UserService {
    UserService userService;

    // UserService를 구현한 다른 오브젝트를 DI 받는다.
    public void setUserService(UserService userService) {
        this.userService = userService;
    }

    // DI 받은 UserService 오브젝트에 모든 기능을 위임한다.
    public void add(User user) {
        userService.add(user);
    }

    // DI 받은 UserService 오브젝트에 모든 기능을 위임한다.
    public void upgradeLevels() {
        userService.upgradeLevels();
    }
}
```
- 이렇게 준비된 UserServiceTex에 트랜잭션 경계 설정 작업을 부여해보자.
```java
// 6-6. 트랜잭션이 적용된 UserServiceTx
public class UserServiceTex implements UserService {
    UserService userService;
    PlatformTransactionManager transactionManager;

    public void setTransactionManager (PlatformTransationManager transactionManaer) {
            this.transactionManager = transactionManager;
    }

    public void setUserService(UserService userService) {
        ...
    }

    ...

    public void upgradeLevels() {
        TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDenition());
        try {
            userService.upgradeLevels();

            this.transactionMananager.commit(status);
        } catch (RuntimeException e) {
            this.transactionManager.rollback(status);
            throw e;
        }
    }
}
```
#### 트랜잭션 적용을 위한 DI 설정
- 클라이언트가 UserService 인터페이스를 통해 사용자 관리 로직을 이용하려고 할 때, 먼저 트랜잭션을 담당하는 오브젝트가 사용돼서 트랜잭션에 관련된 작업을 진행해주고, 사용자 관리 로직을 담은 오브젝트가 이후에 호출돼서 비즈니스 로직에 관련된 작업을 수행하도록 만들어야 한다.

<img width="551" alt="스크린샷 2024-06-27 오후 7 43 19" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/37072770-7a21-453b-baa2-1dabcafaa298">


```xml
// 6-7. 트랜잭션 오브젝트가 추가된 설정 파일
<bean id="userService" class="springbook.user.service.UserServiceTx">
    <property name="transactionManager" ref="transactionManager" />
    <property name="userService" ref="userServiceImpl" />
</bean>

<bean id="userServiceImpl" class="springbook.user.service.UserServiceImpl">
    <property name="userDao" value="userDao" />
    <property name="mailSender" ref="mailSender" />
</bean>
```
- 이제 클라이언트는 UserServiceTx 빈을 호출해서 사용하도록 만들어야 한다.
- 따라서 userService라는 대표적인 빈 아이디는 UserServiceTex 클래스로 정의된 빈에 부여해준다.
#### 트랜잭션 분리에 따른 테스트 수정
- 기존의 UserService 클래스가 인터페이스와 두 개의 클래스로 분리된 만큼 테스트도 변경해야 한다.
- 현재 수정한 스프링 설정 파일에는 UserService라는 인터페이스 타입을 가진 빈이 두 개 존재한다. **UserService 클래스 타입의 빈을 @Autowired로 가져오면 어떤 빈을 가져올까?**
- @Autowired는 타입으로 하나의 빈을 결정할 수 없는 경우에는 필드 이름을 이용해 찾는다. 따라서 다음과 같은 userService 변수를 설정해두면 **아이디가 userService인 빈이 주입**될 것이다.
  ```java
  @Autowired UserService userService;
  ```
- UserServiceTest는 UserServcieImpl 클래스로 정의된 빈도 가져와야 한다.
- 일반적인 UserService 기능 테스트에서는 UserService 인터페이스를 통해 결과를 확인하는 것으로 충분하다. 그러나 목 오브젝트를 이용한 테스트에서는 테스트에서 직접 MailSender를 DI해줘야 할 필요가 있었다.
- MailSender를 DI해줄 대상을 구체적으로 알고 있어야 하기 때문에 UserServiceImpl 클래스의 오브젝트를 가져올 필요가 있다.
- 이렇게 목 오브젝트를 이용해 수동 DI를 적용하는 테스트라면 어떤 클래스의 오브젝트인지 분명하게 알 필요가 있다.
- 그래서 다음과 같이 UserServiceImpl 클래스 타입의 변수를 선언하고 @Autowired를 지정해서 해당 클래스로 만들어진 빈을 주입 받도록 해야 한다.
```java
@Autowired UserServiceImpl userServiceImpl;
```
- upgradeLevels() 테스트 메서드도 다음과 같이 별도로 가져온 userServiceImpl 빈에 MailSender의 목 오브젝트를 설정해줘야 한다.
```java
@Test
public void upgradeLevels() throws Exception {
    ...
    MockMailSender mockMailSender = new MockMailSender();
    userServiceImpl.setMailSender(mockMailSender);
```
- add() 테스트 메서드는 손댈 것이 없지만 upgradeAllNothing() 테스트는 트랜잭션 기술이 바르게 적용되었는지 확인하기 위한 일종의 학습 테스트이기 때문에 변경이 필요하다.
- 기존에는 TestUserService가 오브젝트를 만들어서 의존 오브젝트를 넣어주고서 테스트를 진행했다.
- 이제는 예외를 발생시킬 위치가 UserServiceImpl에 있기 때문에 TestUserService가 트랜잭션 기능은 빠진 UserServiceImpl을 상속하도록 해야 한다.
- 또한 TestUserService 오브젝트를 UserServiceTx 오브젝트에 수동 DI 시킨 후에 트랜잭션 기능까지 포함된 UserServiceTx의 메서드를 호출하면서 테스트를 수행하도록 해야 한다.
```java
// 6-9. 분리된 테스트 기능이 포함되도록 수정한 upgradeAllOrNothing()
@Test
public void upgradeAllOrNothing() throws Exception {
    TestUserService testUserService = new TestUserService(users.get(3).getId());
    testUserService.setUserDao(userDao);
    testUserService.setMailSender(mailSender);

    // 트랜잭션 기능을 분리한 UserServiceTx는 예외 발생용으로 수정할 필요가 없으니 그대로 사용
    UserServiceTx txUserService = new UserServiceTx();
    txUserService.setTransactionManager(transactionManager);
    txUserService.setUserService(testUserService);

    userDao.deleteAll();
    for(User user : users) userDao.add(user);

    try {
        txUserService.upgraeLevels(); // 트랜잭션 기능을 분리한 오브젝트를 통해 예외 발생용 TestUserService가 호출되게 해야 한다.
        fail("TestUserServiceException expected");
    }
    ...
```
- 트랜잭션 테스트용으로 특별히 정의한 TestUserService 클래스는 이제 UserServiceImpl 클래스를 상속하도록 바꿔주면 된다.
```java
static class TestUserService extends UserServiceImpl {
```

#### 트랜잭션 경계 설정 코드 분리의 장점
**1. 비즈니스 로직을 담당하는 코드를 작성할 때 트랜잭션과 같은 기술적인 내용에는 신경쓰지 않아도 된다.**

- 트랜잭션은 DI를 이용해 UserServiceTx와 같은 트랜잭션 기능을 가진 오브젝트가 먼저 실행되도록 만들기만 하면 된다.
- 스프링이나 트랜잭션 같은 로우레벨의 기술적인 지식은 부족한 개발자라고 할지라도 비즈니스 로직을 잘 이해하고 자바 언어의 기초에 충실하면 복잡한 비즈니스 로직을 담은 UserService 클래스 개발 가능
  
**2. 비즈니스 로직에 대한 테스트를 손쉽게 만들어낼 수 있다.**
- 6.2 절에서 좀 더 자세히 알아보자.



## 6.2 고립된 단위 테스트
- 가장 편하고 좋은 테스트 방법은 가능한 한 작은 단위로 쪼개서 테스트하는 것
  - 테스트 단위가 작아야 테스트 의도와 내용이 분명해지고, 만들기도 쉬워진다.
    - 테스트 대상의 단위가 커지면 충분한 테스트를 만들기도 쉽지 않다. 논리적인 오류가 발생해서 결과가 바르게 나오지 않았을 때 그 원인을 찾기도 어려워진다.
- 하지만 작은 단위로 테스트하고 싶어도 그럴 수 없는 경우가 많다.
- 테스트 대상이 다른 오브젝트와 환경에 의존하고 있다면 작은 단위의 테스트가 주는 장점을 얻기 힘들다.

### 6.2.1 복잡한 의존관계 속의 테스트
- UserService의 구현 클래스들이 동작하려면 세 가지 타입의 의존 오브젝트가 필요하다.
  - UserDao 타입의 오브젝트 : DB와 데이터를 주고받는다.
  - MailSender를 구현한 오브젝트 : 메일을 발송한다.
  - PlatformTransactionManger : 트랜잭션 처리를 위해 커뮤니케이션해야 한다.
- 다음은 UserService를 분리하기 전의 테스트가 동작하는 모습이다.
    <img width="576" alt="스크린샷 2024-06-30 오후 2 18 12" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/8e7a360c-76b3-44ac-ab57-458c8287ce8b">

- UserServiceTest의 테스트 단위는 UserService 클래스여야 한다.
- 그러나 UserDao, TransactionManager, MailSender라는 세 가지 의존관계를 갖고 있고, 이들이 테스트가 진행되는 동안에 **같이 실행된다**
- UserService를 테스트하는 것처럼 보이지만 사실은 그 뒤에 존재하는 훨씬 더 많은 오브젝트와 환경, 서비스, 서버, 심지어 네트워크까지 함께 테스트 대상이 되는 것이다.

### 6.2.2 테스트 대상 오브젝트 고립시키기
- 테스트를 의존 대상으로부터 분리해 고립시키는 방법은 MailSender를 적용해봤던 대로 테스트를 위한 대역을 사용하는 것이다. (DummyMailSender, MockMailSender)
#### 테스트를 위한 UserServiceImpl 고립
- UserServiceImpl은 트랜잭션 코드를 독립시켰기 떄문에 더이상 PlatformTransactionManager에 의존하지 않는다. ➡️ PlatformTransactionManager는 고립시키지 않아도 된다.
- 사전에 테스트를 위해 준비된 동작만 하도록 만든 두 개의 목 오브젝트에만 의존하는, 완벽하게 고립된 테스트 대상을 만들어보자.
  <img width="552" alt="스크린샷 2024-06-30 오후 2 26 24" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/64e14efb-daea-4a06-b242-bf30cddba85c">
- UserDaos는 단지 테스트 대상의 코드가 정상적으로 수행되도록 도와주기만 하는 스텁이 아니라, 부가적인 검증 기능까지 가진 목 오브젝트로 만들었다. 그 이유는 고립된 환경에서 동작하는 upgradeLevels()의 테스트 결과를 검증할 방법이 필요하기 때문이다.
- upgradeLevles() 메서드는 void 형 ➡️ 결과를 검증하는 것이 불가능하다.
- 따라서 그 동작이 바르게 됐는지 확인하려면 DB를 직접 확인해야 하는데, 의존 오브젝트나 외부 서비스에 의존하지 않는 고립된 테스트 방식으로 만든 UserServiceImpl은 아무리 기능이 수행돼도 DB 등을 통해 남지 않으니 작업 결과를 검증하기 힘들다.
- 이럴 땐 **테스트 대상인 UserServiceImpl과 그 협력 오브젝트인 UserDao에게 어떤 요청을 했는지 확인하는 작업이 필요하다.**
- UserDao와 같은 역할을 하면서 UserServiceImpl과의 사이에서 주고받은 정보를 저장해뒀다가, 테스트의 검증에 사용할 수 있게 하는 목 오브젝트를 만들 필요가 있다.

#### 고립된 테스트 활용
- 기존 upgradeLevels() 테스트는 다섯 단계의 작업으로 구성된다.
```java
// 6-10. upgradeLevels() 테스트
@Test
public void upgradeLevels() throws Exception {
  // (1) DB 테스트 데이터 준비
  userDao.deleteAll();
  for(User user : users) userDao.add(user);

  // (2) 메일 발송 여부 확인을 위한 목 오브젝트 DI
  MockMailSender mockMailSender = new MockMailSender(); 
  userService.setMailSender(mockMailSender);

  // (3) 테스트 대상 실행
  userService.upgradeLevels();

  // (4) DB에 저장된 결과 확인
  checkLevelUpgraded(users.get(0), false);
  checkLevelUpgraded(users.get(1), true);
  checkLevelUpgraded(users.get(2), false);
  checkLevelUpgraded(users.get(3), true);
  checkLevelUpgraded(users.get(4), false);

  // (5) 목 오브젝트를 이용한 결과 확인
  List<String> request = mockMailSender.getRequests();
  assertThat(request.size(), is(2));
  assertThat(request.get(0), is(users.get(1).getEmail()));
  assertThat(request.get(1), is(users.get(3).getEmail()));
}

private void checkLevelUpgraded(User user, boolean upgraded) {
  User userUpdate = userDao.get(user.getId());
  ...
}
```
- 실제 UserDao와 DB까지 직접 의존하고 있는 (1)과 (4) 테스트 방식도 목 오브젝트를 만들어 적용해보겠다.
- 목 오브젝트는 기본적으로 스텁과 같은 방식으로 테스트 대상을 통해 사용될 때 필요한 기능을 지원해줘야 한다.
- upgradeLevels() 메서드가 실행되는 중에 **UserDao와 어떤 정보를 주고받는지 입출력 내역을 먼저 확인할 필요가 있다.**
- upgradeLevels()와 그 사용 메서드에서 UserDao를 사용하는 경우는 두 가지다.
```java
// 6-1. 사용자 레벨 업그레이드 작업 중에 UserDao를 사용하는 코드
public void upgradeLevels() {
    List<User> users = userDao.getAll(); // (1) 업그레이드 후보 사용자 목록을 가져온다.
    for(User user : users) {
        if(canUpgradeLevel(user)) {
            upgradeLevel(user);
        }
    }
}

protected void upgradeLevel(User user) {
    user.upgradeLevel();
    userDao.upgrade(user); // (2) 수정된 사용자 정보를 DB에 반영한다.
    sendUpgradeEmail();
}
```
- (1) 기능을 테스트용으로 대체 ➡️ 테스트용 UserDao에서는 DB에서 읽어온 것처럼 미리 준비된 사용자 목록을 제공해줘야 한다.
- (2) 기능을 테스트용으로 대체 ➡️ 리턴 값이 없으므로 아무런 내용도 없는 빈 메서드로 만들어도 된다. 하지만 update() 메서드의 내용은 '변경'이 되었는지 검증할 수 있는 중요한 기능이기도 하다.
- 따라서 getAll()에 대해서는 스텁으로서, update()에 대해서는 목 오브젝트로서 동작하는 UserDao로서 동작하는 UserDao 타입의 테스트 대역이 필요하다.
- 이 클래스의 이름을 MockUserDao라고 하자.
```java
// 6-12. UserDao 오브젝트
static class MockUserDao implements UserDao {
    private List<User> users; // 레벨 업그레이드 후보 User 오브젝트 목록
    private List<User> updated = new ArrayList(); // 업그레이드 대상 오브젝트를 저장해둘 목록

    private MockUserDao(List<User> users) {
        this.users = users;
    }

    public List<User> getUpdated() {
        return this.updated;
    }

    // 스텁 기능 제공
    public List<User> getAll(); {
        return this.users;
    }

    // 목 오브젝트 기능 제공
    public void update(User user) {
        updated.add(user);
    }

    // 테스트에 사용되지 않는 메서드
    public void add(User user) { throw new UnsupportedOperationException(); }
    public void deleteAll() { throw new UnsupportedOperationException(); }
    public void get(String id) { throw new UnsupportedOperationException(); }
    public void getCount() { throw new UnsupportedOperationException(); }
}
```
- MockUserDao는 UserDao 구현 클래스를 대신하니 implements 해야 한다.
- 다만 이를 통해 사용하지 않을 메소드도 오버라이드 받게 되는데 이럴때는 UnsupportedOperationException을 던지도록 만드는 편이 좋다.
- 해당 클래스에는 두 개의 User 타입 리스트가 있는데, 변수명 users는 처음 생성자가 생성될 때 전달받은 사용자 목록을 저장한 뒤 getAll() 메소드가 호출되면 전달하는 용도이다.
- 다른 하나는 update() 메소드를 실행하면서 넘겨준 업그레이드 대상 User 오브젝트를 저장해뒀다가 검증을 위해 돌려주기 위한 것이다.이제 upgradeLevels() 테스트가 MockUserDao를 사용하도록 수정해보자.

- 이제 upgradeLevels() 테스트가 MockUserDao를 사용하도록 수정해보자.
```java
// 6-13. MockUserDao를 사용해서 만든 고립된 테스트
@Test
public void upgradeLevels() throws Exception{

    // 고립된 테스트에서는 테스트 대상 오브젝트를 직접 생성하면 된다.
    UserServiceImpl userServiceImpl = new UserServiceImpl();

    // 목 오브젝트로 만든 UserDao를 직접 DI 해준다.
    MockUserDao mockUserDao = new MockUserDao(this.users);
    userServiceImpl.setUserDao(mockUserDao);

    // 메일 발송 여부 확인을 위해 목 오브젝트 DI
    MockMailSender mockMailSender = new MockMailSender();
    userServiceImpl.setMailSender(mockMailSender);

    // 테스트 대상 실행
    userServiceImpl.upgradeLevels();

    List<User> updated = mockUserDao.getUpdated();
    assertThat(updated.size(), is(2));
    chekcUserAndLevel(updated.get(0), "joytouch", Level.SILVER);
    chekcUserAndLevel(updated.get(1), "madnite1", Level.GOLD);


    // 목 오브젝트를 이용한 결과 확인.
    // 목 오브젝트에 저장된 메일 수신자 목록을 가져와 업그레이드 대상과 일치하는지 확인 한다.
    List<String> request = mockMailSender.getRequests();
    assertThat(request.size(), is(2));
    assertThat(request.get(0), is(users.get(1).getEmail()));
    assertThat(request.get(1), is(users.get(3).getEmail()));

}
```
- 고립된 테스트 이전에는 @Autowired를 통해 가져온 UserService 타입의 빈이었다.
- 이제는 완전히 고립되어 테스트만을 위해서 사용할 것이기에 스프링 컨테이너에서 가져올 필요 없이 직접 생성한다.
- 이후 UserServiceImpl 오브젝트의 메소드를 실행시키면 된다.
- 이후 update()를 이용해 몇 명의 사용자 정보를 DB에 수정하려고 했는지, 그 사용자들이 누구인지, 어떤 레벨로 변경됐는지를 확인하면 된다.
- 이렇게 고립된 테스트를 하면 테스트가 다른 의존 대상에 영향을 받을 경우를 대비해 복잡하게 준비할 필요가 없을 뿐만 아니라, 테스트 수행 성능도 크게 향상된다.
- 고립된 테스트를 만들려면 목 오브젝트 작성과 같은 약간의 수고가 더 필요할 지 모르겠지만, 그 보상은 충분히 기대할 만 하다.
### 6.2.3 단위 테스트와 통합 테스트
- upgradeLevels() 테스트처럼 '테스트 대상 클래스를 목 오브젝트 등의 테스트 대역을 이용해 의존 오브젝트나 외부의 리소스를 사용하지 않도록 고립시켜서 테스트하는 것'을 단위테스트, 반면에 두개 이상의, 성격이나 계층이 다른 오브젝트가 연동하도록 만들어 테스트하거나, 또는 외부의 DB 파일, 서비스 등의 리소스가 참여하는 테스트는 통합테스트 이다.

### 6.2.4 목 프레임워크
- 단위 테스트가 많은 장점이 있고 가장 우선시해야 할 테스트 방법인 건 사실이지만 작성이 번거롭다는 것이 문제이다.
- 특히 목 오브젝트를 만드는 일이 가장 큰 점이다. 다행히도, 이런 번거로운 목 오브젝트를 편리하게 작성하도록 도와주는 다양한 목 오브젝트 지원 프레임워크가 있다.

#### Mockito 프레임워크
```java
// 6-14. Mockito를 적용한 테스트 코드
@Test
public void mockUpgradeLevels() throws Exception{
    UserServiceImpl userServiceImpl = new UserServiceImpl();

    // 다이나믹한 목 오브젝트 생성과 메소드의 리턴 값 설정, 그리고 DI까지 세 줄이면 충분하다.
    UserDao mockUserDao = mock(UserDAO.class);
    when(mockUserDao.getAll()).thenReturn(this.users);
    userServiceImpl.setUserDao(mockUserDao);

    // 리턴 값이 없는 메소드를 가진 목 오브젝트는 더욱 간단하게 만들 수 있다.
    MailSender mockMailSender = mock(MailSender.class);
    userServiceImpl.setMailSender(mockMailSender);

    userServiceImpl.upgradeLevels();

    // 목 오브젝트가 제공하는 검증 기능을 통해서 어떤 메소드가 몇 번 호출 됐는지, 파라미터는 무엇인지 확인할 수 있다.
    verify(mockUserDao, times(2)).update(any(User.class));
    verify(mockUserDao).update(users.get(1));
    assertThat(users.get(1).getLevel(), is(Level.SILVER));
    verify(mockUserDao).update(users.get(3));
    assertThat(users.get(3).getLevel(), is(Level.GOLD));

    // Mockito 프레임워크의 툴 사용법으로써 배우지 않은 부분이니 그냥 보내질 메일 주소를 확인한다고 알고 있자.
    ArgumentCaptor<SimpleMailMessage> mailMessageArg = ArgumentCaptor.forClass(SimpleMailMessage.class);
    verify(mockMailSender, times(2)).send(mailMessageArg.capture());
    List<SimpleMailMessage> mailMessages = mailMessageArg.getAllValues();
    assertThat(mailMessages.get(0).getTo()[0], is(users.get(1).getEmail()));
    assertThat(mailMessages.get(1).getTo()[0], is(users.get(3).getEmail()));
}
```
- 다만, 제대로 사용하기 위해서는 Mockito 프레임워크의 툴을 알아두어야 하며 이를 알아둔다면 단위 테스트를 만들때 유용하게 사용될 수 있다는 것을 알아두자.

## 6.3 다이내믹 프록시와 팩토리 빈
### 6.3.1 프록시와 프록시 패턴, 데코레이터 패턴
> 트랜잭션 경계설정 코드를 비즈니스 로직 코드에서 분리해낼 때 적용했던 기법을 다시 검토해보자.

- 다음은 전략 패턴으로 트랜잭션 구현 내용을 분리해냈을 때(트랜잭션과 같은 부가적인 기능을 **위임**을 통해 외부로 분리했을 때)의 결과이다.
- 구체적인 구현 코드는 제거했을지라도 위임을 통해 기능을 사용하는 코드는 핵심 코드와 함께 남아있다.
<img width="544" alt="스크린샷 2024-07-01 오후 11 58 23" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/606a492c-1a95-4e44-9f87-84416e7222c2">

- 트랜잭션 기능은 비즈니스 로직과는 성격이 달라 아예 밖으로 분리할 수 있다. 부가기능 전부를 핵심 코드가 담긴 클래스에서 독립 시킬 수 있다. ➡️ UserServiceTx, 트랜잭션 관련 코드가 하나도 없게 된 UserServiceImpl
<img width="557" alt="스크린샷 2024-07-02 오후 7 53 58" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/3a84b78b-d78c-489b-9883-5bdf7c14a16a">
- 이렇게 **분리된 부가기능**을 담은 클래스는 **나머지 기능은 원래 핵심 기능을 가진 클래스로 위임해줘야 한다**
- 핵심 기능은 부가기능을 가진 클래스의 존재를 모른다. 따라서 부가기능이 핵심 기능을 사용하는 구조가 되는 것이다.
- 문제는 이렇게 구성했더라도 클라이언트가 핵심 기능을 가진 클래스를 직접 사용해버리면 부가기능이 적용될 기회가 없다는 점이다.
- 그래서 **부가기능은 마치 자신이 핵심 기능을 가진 클래스인 것처럼 꾸며서, 클라이언트가 자신을 거쳐서 핵심 기능을 사용하도록 만들어야 한다.**
- 그러기 위해서는 클라이언트는 인터페이스를 통해서만 핵심 기능을 사용하게 해야 하고, 부가기능 자신도 같은 인터페이스를 구현한 뒤에 자신이 그 사이에 끼어들어야 한다.

<img width="551" alt="스크린샷 2024-07-02 오후 8 00 56" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/5504fb75-cc5b-47f7-9bbe-a2b4a030717c">

- 부가기능 코드에서는 핵심 기능으로 요청을 위임해주는 과정에서 자신이 가진 부가기능을 적용해줄 수 있다. (ex. 비즈니스 로직 코드에 트랜잭션 기능 부여)
- 이렇게 마치 자신이 클라이언트가 사용하려고 하는 실제 대상인 것처럼 위장해서 클라이언트의 요청을 받아주는 것을 **대리자**, **대리인**과 같은 역할을 한다고 해서 **프록시**라고 부른다.
- 그리고 프록시를 통해 최종적으로 요청을 위임받아 처리하는 실제 오브젝트를 **타깃(target)** 또는 **실체(real object)**라고 부른다.
- 다음은 클라이언트가 프록시를 통해 타깃을 사용하는 구조를 보여준다.
  <img width="552" alt="스크린샷 2024-07-02 오후 8 05 18" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/edcdb0d3-4bf3-4719-8878-eb18fd3c3564">

- 프록시의 특징은 **타깃과 같은 인터페이스를 구현**했다는 것과 **프록시가 타깃을 제어할 수 있는 위치에 있다**는 것이다.
- 프록시는 사용 목적에 따라 두 가지로 구분할 수 있다.
  1. 클라이언트가 타깃에 접근하는 방법 제어
  2. 타깃에 부가적인 기능을 부여
- 두 가지 프록시를 두고 사용한다는 점은 동일하지만, 목적에 따라 디자인 패턴에서는 다른 패턴으로 구분한다. 

#### 데코레이터 패턴
- 타깃에 부가적인 기능을 런타임 시 다이내믹하게 부여해주기 위해 프록시를 사용하는 패턴
  - 다이내믹하게 기능을 부가한다 : 컴파일 시점, 즉 코드 상에는 어떤 방법과 순서로 프록시와 타깃에 연결되어 사용되는지 정해져 있지 않다.
- 실제 내용물은 동일하지만 부가적인 효과를 부여해줄 수 있기 때문에 데코레이터 패턴이라고 불린다.
- 프록시가 꼭 한 개로 제한되지 않는다.
- 프록시가 직접 타깃을 사용하도록 고정시킬 필요도 없다.
- 이를 위해 데코레이터 패턴에서는 같은 인터페이스를 구현한 타겟과 여러 개의 프록시를 사용할 수 있다.
- 프록시가 여러 개인만큼 순서를 정해서 단계적으로 위임하는 구조로 만들면 된다.

- 예를 들어 소스코드를 출력하는 기능을 가진 핵심 기능이 있다고 하자. 이 클래스에 데코레이터 개념을 부여해서 타깃과 같은 인터페이스를 구현하는 프록시를 만들 수 있다.
  
  <img width="667" alt="스크린샷 2024-07-03 오후 10 05 50" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/d5c67f4c-dd74-450d-956a-ad0af9ac31dc">
- 프록시로서 동작하는 데코레이터는 위임하는 대상에도 인터페이스로 접근하기 때문에 자신이 최종 타깃으로 위임하는지, 아니면 다음 단계의 데코레이터 프록시로 위임하는지 알지 못한다.
- 그래서 다음 위임 대상은 인터페이스로 선언하고 생성자나 수정자 메서드를 통해 **위임 대상을 외부에서 런타임 시에 주입 받을 수 있도록 만들어야 한다.**
- 데코레이터 패턴의 대표적인 예 : InputStream과 OutputStream
  ```java
  InputStream is = new BufferedInputream(new FileInputStream("a.txt"));
  ```
- UserService 인터페이스를 구현한 타깃인 UserServiceImpl에 트랜잭션 부가 기능을 제공해주는 UserServiceTx를 추가한 것도 데코레이터 패턴을 적용한 것이라 볼 수 있다. (수정자로 UserServiceTx에 위임할 타깃인 UserServiceImpl 주입)
- 인터페이스를 통한 데코레이터 정의와 런타임 시의 다이내믹한 구성 방법은 **스프링의 DI**를 이용하면 편하다. 데코레이터 빈의 프로퍼티로 같은 인퍼에시를 구현한 다른 데코레이터 또는 타깃 빈을 설정하면 된다.

```java
// 6-15. 데코레이터 패턴을 위한 DI 설정
<!-- 데코레이터 -->
<bean id="userService" class="springbook.user.service.UserServiceTx">
    <property name="transactionManager" ref="transactionManager" />
    <property name="userService" ref="userServiceImpl" />
</bean>

<!-- 타깃 -->
<bean id="userServiceImpl" class="springbook.user.service.UserServiceImpl">
    <property name="userDao" ref="userDao" />
    <property name="mailSender" ref="mailSender" />
</bean>
```

- 데코레이터 패턴은 인터페이스를 통해 위임하는 방식 ➡️ 어느 데코레이터에서 타깃으로 연결될지 코드 레벨에서는 미리 알 수 없음.
- 구성하기에 따라 여러 데코레이터 적용 가능
- 데코레이터 패턴은 타깃의 코드를 손대지 않고, 클라이언트가 호출하는 방법도 변경하지 않은 채로 새로운 기능을 추가할 때 유용한 방법이다.

#### 프록시 패턴
- 일반적응로 사용하는 **프록시**라는 용어는 클라이언트와 사용 대상 사이에서 대리 역할을 맡은 오브젝트를 두는 방법을 총칭한다면, 디자인 패턴에서 말하는 프록시 패턴은 프록시를 사용하는 방법 중에서 타깃에 대한 접근 방법을 제어하려는 목적을 가진 경우를 가리킨다.
- 프록시 패턴의 프록시는 타깃의 기능을 확장하거나 추가하지 않는다. 대신 클라이언트가 **타깃에 접근하는 방식**을 변경해준다.
