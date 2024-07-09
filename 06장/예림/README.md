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
- 클라이언트에게 타깃에 대한 레퍼런스를 넘겨야 하는데, 실제 타깃 오브젝트를 만드는 대신 프록시를 넘겨줄 수 있다.
    - 프록시의 메서드를 통해 타깃을 사용하려고 시도하면, 그 때 프록시가 타깃 오브젝트를 생성하고 요청을 위임해주는 식이다.
    - 만약 레퍼런스는 갖고 있지만 끝까지 사용하지 않거나, 많은 작업이 진행된 후에 사용되는 경우라면, 이렇게 프록시를 통해 **생성을 최대한 늦춤**으로써 얻는 장점이 많다.
- 또는 원격 오브젝트를 이용하는 경우에도 프록시를 사용하면 편하다.
  - RMI나 EJB, 또는 각종 리모팅 기술을 이용해 다른 서버에 존재하는 오브젝트를 사용해야 한다면, 원격 오브젝트에 대한 프록시를 만들어두고, 클라이언트느 마치 로컬에 존재한느 오브젝트를 쓰는 것처럼 프록시를 사용하게 할 수 있다.
  - 프록시는 요청을 받으면 네트워크를 통해 원격 오브젝트를 실행하고 결과를 받아 클라이언트에게 돌려준다.
- 또는 특별한 상황에서 타깃에 대한 접근 권한을 제어하기 위해 프록시 패턴을 사용할 수 있다.
  - 만약 수정 가능한 오브젝트가 있는데, 특정 레이어로 넘어가서는 읽기 전용으로만 동작하게 강제해야 한다고 하자. 이럴 때는 오브젝트의 프록시를 만들어서 사용할 수 있다.
  - 프록시의 특정 메서드를 사용하려고 하면 접근이 불가능하다고 예외를 발생시키면 된다.
  - ex. Collections의 unmodifiableCollection()을 통해 만들어지는 오브젝트
      - 파라미터로 전달된 Collection 오브젝트의 프록시를 만들어서, add()나 remove()와 같이 정보를 수정하는 메서드를 호출할 경우 UnSupportedOperationException 발생
- 구조적으로 프록시와 데코레이터 패턴은 유사하지만, 프록시는 코드에서 자신이 만들거나 접근할 타깃 클래스 정보를 알고 있는 경우가 많다.
- 물론 프록시 패턴이더라도 인터페이스를 통해 위임하도록 만들 수 있다. 인터페이스를 통해 다음 호출 대상으로 접근하게 하면 그 사이에 다른 프록시나 데코레이터가 계속 추가될 수 있기 때문이다.
- 다음은 접근 제어를 위한 프록시를 두는 프록시 패턴과 컬러, 페이징 기능을 추가하기 위한 프록시를 두는 데코레이터 패턴을 함께 적용한 예이다. 두 가지 모두 프록시 기본 원리대로 타깃과 같은 인터페이스의 기본 원리대로 타깃과 같은 인터페이스를 구현해두고 위임하는 방식으로 만들어져 있다.
  <img width="594" alt="스크린샷 2024-07-05 오후 12 32 46" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/52599015-843f-4a2f-9556-f9c795301d5d">

### 6.3.2 다이내믹 프록시
- 프록시는 유용한 방법이지만 그럼에도 불구하고 개발자는 타깃 코드를 직접 고치고 말지 번거롭게 프록시는 만들지 않겠다고 생각한다.
- 목 오브젝트를 만드는 불편함과 목 프레임워크를 사용해 편리하게 바꿨던 것처럼 프록시도 일일히 모든 인터페이스를 구현해서 클래스를 새로 정의하지 않고도 편리하게 만들어서 사용할 방법은 없을까?
- 물론 있다. 자바에는 `java.lang.reflect` 패키지 안에 프록시를 손쉽게 만들 수 있도록 지원해주는 클래스들이 있다. 원리는 목 프레임워크와 비슷하게 몇 가지 API를 이용해 프록시처럼 동작하는 오브젝트를 다이내믹하게 생성하는 것이다.

#### 프록시의 구성과 프록시 작성의 문베점
- 프록시는 다음의 두 가지 기능으로 구성된다.
  - 타깃과 같은 메서드를 구현하고 있다가 메서드가 호출되면 타깃 오브젝트로 **위임**한다.
  - 지정된 요청에 대해서는 **부가기능**을 수행한다.
- `UserServiceTx`는 기능 부가를 위한 프록시다. `UserServiceTx` 코드에서 이 두가지 기능을 구분해보자.

```java
// 6-16. UserServiceTx 프록시의 기능 구분
public class UserServiceTx implements UserService {
    UserService userService; // 타깃 오브젝트
    ...

    // 메서드 구현과 위임
    public void add(User user) {
        this.userService.add(user);
    }

    public void upgradeLevels() { // 메서드 구현
        // 부가 기능 수행
        TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
        try {
            // 위임
            userService upgradeLevels(); 

            // 부가 기능 수행 이어서
            this.transactionManager.commit(status);
        } catch(RuntimeException e) {
            this.transactionManager.rollback(status);
            throw e;
            // 부가 기능 수행 끝
        }
    }
}
```


- 그렇다면 프록시를 만들기 번거로운 이유는 무엇일까?

  - 타깃 인터페이스를 구현하고 위임하는 코드를 **작성하기 번거롭다.** 부가기능이 필요 없는 메서드도 구현해서 타깃으로 위임하는 코드를 일일히 만들어줘야 한다. 또 타깃 인터페이스의 메서드가 추가되거나 변경될 때마다 함꼐 수정해줘야 한다는 부담도 있다.

  - 부가 기능 **코드가 중복**될 가능성이 많다.

- 트랜잭션 외의 프록시를 활용할만한 부가기능, 접근제어 기능은 일반적인 성격을 띤 것이 많아 다양한 타깃 클래스와 메서드에 중복돼서 나타날 가능성이 높다.
- 두 번째 문제는 코드를 분리해 어떻게든 해결할 수 있지만 첫 번째 문제는 간단해 보이지 않는다. 바로 이런 문제를 해결하는 데 유용한 것이 JDK의 **다이내믹 프록시**다.

#### 리플렉션
- 다이내믹 프록시는 리플렉션 기능을 이용해 프록시를 만들어준다.
  - 리플랙션 : 자바의 코드 자체를 추상화해서 접근하도록 만든 것
- 리플렉션 API 중에서 메서드에 대한 정의를 담은 Method라는 인터페이스를 이용해 메서드를 호출하는 방법을 알아보자.
  - String 클래스의 정보를 담은 Class 타입의 정보는 `String.class`라고 하면 가져올 수 있다.
  - 또는 스트링 오브젝트 name이 있으면 `name.getClass()`라고 해도 된다.
  - 그리고 이 클래스 정보에서 특정 이름을 가진 메서드 정보를 가져올 수 있다. String의 length() 메서드라면 다음과 같이 하면 된다.
    ```java
    Method lengthMethod = String.class.getMethod("length");
    ```
  - 스트링이 가진 메서드 중에서 "length"의 이름을 갖고 있고, 파라미터는 없는 메서드의 정보를 가져오는 것이다.
- java.lang.reflect.Method 인터페이스는 메서드에 대한 자세한 정보를 담고 있을 뿐만 아니라 이를 이용해 특정 오브젝트의 메서드를 실행시킬 수도 있다. Method 인터페이스에 정의된 `invoke()` 메서드를 사용하면 된다.
- `invoke()` 메서드는 메서드를 실행시킬 대상 오브젝트(`obj`)와 파라미터 목록(`args`)를 받아서 메서드를 호출한 뒤에 그 결과를 `Object` 타입으로 돌려준다.
```java
public Object invoke(Object obj, Object ... args)
```
- 이를 이용해 length() 메서드를 다음과 같이 실행할 수 있다.
```java
int length = lengthMethod.invoke(name); // int length = name.length();
```

```java
// 6-17. 리플렉션 학습 테스트
...
public class ReflectionTest {
    @Test
    public void invokeMethod() throws Exception {
        String name = "Spring";

        // length()
        assertThat(name.length(), is(6));

        Method lengthMethod = String.class.getMethod("length");
        assertThat((Integer)lengthMethod.invoke(name), is(6));

        // charAt()
        assertThat(name.length(0), is('S'));

        Method charAtMethod = String.class.getMethod("charAt", int.class);
        assertThat((Character)charAtMethod.invoke(name, 0), is('S'));
    }
}
```

#### 프록시 클래스
- 다이내믹 프록시를 이용한 프록시를 만들어보자.
- 프록시를 적용할 간단한 타깃 클래스와 인터페이스를 다음과 같이 정의한다.

```java
// 6-18. Hello 인터페이스
interface Hello {
    String sayHello(String name);
    String sayHi(String name);
    String sayThankYou(String name);
}
```

```java
// 6-19. 타깃 클래스
public class HelloTarget implements Hello {
    public String sayHello(String name) {
        return "Hello " + name;
    }

    public String sayHi(String name) {
        return "Hi " + name;
    }

    public String sayThankYou(String name) {
        return "Thank You " +name;
    }
}
```
- 이제 Hello 인터페이스를 통해 `HelloTarget` 오브젝트를 사용하는 **클라이언트 역할을 하는 간단한 테스트**를 다음과 같이 만든다.

```java
// 6-20. 클라이언트 역할의 테스트
@Test
public void simpleProxy() {
    Hello hello = new HelloTarget(); // 타깃은 인터페이스를 통해 접근하는 습관을 들이자.
    assertThat(hello.sayHello("Toby"), is("Hello Toby"));
    assertThat(hello.sayHi("Toby"), is("Hi Toby"));
    assertThat(hello.sayThankYou("Toby"), is("Thank You Toby"));
}
```

- 이제 `Hello` 인터페이스를 구현한 프록시를 만들어보자. 프록시에는 데코레이터 패턴을 적용해서 타깃인 `HelloTarget`에 부가기능을 추가하겠다. 프록시의 이름은 `HelloUppercase`이다.
- 추가하는 기능은 리턴하는 문자를 모두 대문자로 바꿔주는 것이다.
- `SimpleTarget`이라는 원본 클래스는 그대로 두고, 경우에 따라 대문자로 출력이 필요한 경우를 위해서 `HelloUppercase` 프록시를 통해 문자를 바꿔준다. `HelloUppercase` 프록시는 `Hello` 인터페이스를 구현하고, `Hello` 타입의 타깃 오브젝트를 받아서 저장해둔다.
- `Hello` 인터페이스는 구현 메서드에서는 타깃 오브젝트의 메서드를 호출한 뒤에 결과를 대문자로 바꿔주는 부가기능을 적용하고 리턴한다.
- 위임과 기능 부가라는 두 가지 프록시 기능을 모두 처리하는 전형적인 프록시 클래스
```java
// 6-21. 프록시 클래스
public class HelloUpperCase implements Hello {
    Hello hello; // 위임할 타깃 오브젝트. 여기서는 타깃 클래스의 오브젝트인 것은 알지만 다른 프록시를 추가할 수도 있으므로 인터페이스로 접근한다.

    public HelloUppercase(Hello hello) {
        this.hello = hello;
    }

    public String sayHello(String name) {
        return hello.sayHello(name).toUpperCase(); // 위임과 부가기능 적용
    }

    public String sayHi(String name) {
        return hello.sayHi(name).toUpperCase();
    }

    public String sayThankYou(String name) {
        return hello.sayThankYou(name).toUpperCase();
    }

}
```

- 다음과 같이 테스트 코드를 추가해서 프록시가 동작하는지 확인해보자.

```java
// 6-22. HelloUpperCase 프록시 테스트
Hello proxiedHello = new HelloUpperCase(new HelloTarget()); // 프록시를 통해 타깃 오브젝트에 접근하도록 구성한다.
asserThat(proxiedHello.sayHello("Toby"), is("HEELO TOBY"));
asserThat(proxiedHello.sayHi("Toby"), is("HI TOBY"));
asserThat(proxiedHello.sayThankYou("Toby"), is("THANK YOU TOBY"));
```
- 이 프록시는 프록시 적용의 일반적인 문제점 두 가지를 갖고 있다.
  - 인터페이스의 모든 메서드를 구현해 위임하도록 코드를 만들어야 한다.
  - 부가기능인 리턴 값을 대문자로 바꾸는 기능이 모든 메서드에 중복돼서 나타난다.

#### 다이내믹 프록시 적용
- 클래스로 만든 프록시인 HelloUpperCase를 **다이내믹 프록시**를 이용해 만들어보자.

<img width="552" alt="스크린샷 2024-07-07 오후 5 04 40" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/6629c78d-e53f-4559-a091-8e4c343ae20e">

- 다이내믹 프록시는 프록시 팩토리에 의해 런타임 시 다이내믹하게 만들어지는 오브젝트다.
- 다이내믹 프록시 오브젝트는 타깃의 인터페이스와 같은 타입으로 만들어진다.
- 클라이언트는 다이내믹 프록시 오브젝트를 타깃 인터페이스를 통해 사용할 수 있다.
- 프록시 팩토리에게 인터페이스 정보만 제공해주면 해당 인터페이스를 구현한 클래스의 오브젝트를 자동으로 만들어주기 때문에 편리하다.
- 그러나 프록시로서 필요한 부가기능 제공 코드는 직접 작성해야 한다. 부가기능은 프록시 오브젝트와 독립적으로 `InvocationHandler`를 구현한 오브젝트에 담는다.

```java
public Object invoke(Object proxy, Method method, Object[] args)
```
- `invoke()` 메서드는 리플렉션의 `Method` 인터페이스를 파라미터로 받는다. 메서드를 호출할 때 전달되는 파라미터도 `args`로 받는다.
- 다이내믹 프록시 오브젝트는 클라이언트의 모든 요청을 리플렉션 정보로 변환해서 `InvocationHandler` 구현 오브젝트의 `invoke()` 메서드로 넘기는 것이다.
- `InvokeHandler` 구현 오브젝트가 타깃 오브젝트 레퍼런스를 갖고 있다면 리플렉션을 이용해 간단한 위임 코드를 만들어낼 수 있다.
- `InvocationHandler` 인터페이스를 구현한 오브젝트를 제공해주면 다이내믹 프록시가 받는 요청을 `InvocationHandler`의 `invoke()` 메서드로 보내준다.
<img width="523" alt="스크린샷 2024-07-07 오후 5 29 04" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/22556acf-1deb-4b80-a6de-cdc6a5a8b180">

```java
// 6-23. InvocationHandler 구현 클래스
public class UppercaseHandler implements InvocationHandler {
    Hello target;

    // 다이내믹 프록시로부터 전달받은 요청을 다시 타깃 오브젝트에 위임해야하기 때문에 타깃 오브젝트를 주입받아 둔다.
    public UppercaseHandler(Hello target) {
        this.target = target;
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        String ret = (String)method.invoke(target, args); // 타깃으로 위임, 인터페이스의 메서드 호출에 모두 적용한다.
        return ret.toUpperCase(); // 부가기능 제공
    }
}
```

- 다이내믹 프록시를 통해 요청이 전달되면 리플렉션 API를 이용해 타깃 오브젝트의 메서드를 호출한다.
- Hello 인터페이스의 모든 메서드는 결과가 String 타입이므로 메서드 호출의 결과를 String 타입으로 변환해도 안전하다.
- 이제 `InvocationHandler`를 사용하고 Hello 인터페이스를 구현하는 프록시를 만들어보자.
- 다이내믹 프록시의 생성은 Proxy 클래스의 newProxyInstance() 스태틱 팩토리 메서드를 이용하면 된다.

```java
// 6-24. 프록시 생성
Hello proxiedHello = (Hello)Proxy.newProxyInstance( // 생성된 다이내믹 프록시 오브젝트는 Hello 인터페이스를 구현하고 있으므로 Hello 타입으로 캐스팅해도 안전하다.
                getClass.getClassLoader(), // 동적으로 생성되는 다이내믹 프록시 클래스의 로딩에 사용할 클래스로더 
                new Class[] { Hello.class }, // 구현할 인터페이스
                new UpperCaseHandler(new HelloTarget())); // 부가기능과 위임 코드를 담은 invocationHandler
```

- 그런데 원래 만들었던 HelloUpperCase 프록시 클래스보다 그다지 코드 양이 줄어들지 않은 것 같고, 코드 작성도 까다로워진 것 같다.
- 다이내믹 프록시를 적용했을 때 장점은 있는 걸까?

#### 다이내믹 프록시의 확장
- Hello 인터페이스 메서드가 늘어나도 UppercaseHandler와 다이내믹 프록시를 생성해서 사용하는 코드는 전혀 손댈 게 없다.
- UpperCaseHandler에서는 모든 메서드의 리턴 타입이 스트링이라 가정한다. 그런데 스트링 외의 리턴 타입을 갖게 되는 메서드가 추가되면 어떨까?
  - Method를 이용한 타깃 오브젝트의 메서드 호출 후 리턴 타입을 확인해서 스트링인 경우만 대문자로 바꿔주고 나머지는 그대로 넘겨주는 방식으로 수정하는 것이 좋겠다.
- InvocationHandler 방식의 또 한 가지 장점은 타깃의 종류와 상관없이 적용이 가능하다는 점이다.
- 어떤 종류의 인터페이스를 구현한 타깃이든 상관없이 재사용할 수 있고, 메서드의 리턴 타입이 스트링인 경우만 대문자로 결과를 바꿔주도록 UpperCaseHandler를 만들 수 있다.
```java
// 6-25. 확장된 UpperCaseHandler
 public class UppercaseHandler implements InvocationHandler {
    // 어떤 종류의 인터페이스를 구현한 타깃에도 적용 가능하도록 Object 타입으로 수정
    Object target;
    private UppercaseHandler(Object target) {
        this.target = target;
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 호출한 메서드의 리턴 타입이 String인 경우만 대문자 변경 기능을 적용하도록 수정
        Object ret = method.invoke(target, args);
        if (ret instanceof String) {
            return ((String ret).toUpperCase();
        }
        else {
            return ret;
        }
    }
}
```
- 리턴 타입 뿐 아니라 메서드의 이름도 조건으로 걸 수 있다.
- 메서드의 이름이 say로 시작하는 경우에만 대문자로 바꾸는 기능을 적용하고 싶다면 다음과 같이 Method 파라미터에서 메서드 이름을 가져와 확인하는 방법을 사용하면 된다.

```java
// 메서드를 선별해서 부가기능을 적용하는 invoke()

public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    Object ret = method.invoke(target, args);
    if (ret instanceof String && method.getName().startsWith("say")) { // 리턴 타입과 메서드 이름이 일치하는 경우에만 부가기능 적용
        return ((String ret).toUpperCase();
    }
    else {
        return ret; // 조건이 일치하지 않으면 타깃 오브젝트의 호출결과를 그대로 리턴
    }
}
```

### 6.3.3 다이내믹 프록시를 이용한 트랜잭션 부가기능
- UserServiceTx를 다이내믹 프록시 방식으로 변경해보자.

#### 트랜잭션 InvocationHandler
```java
// 6-27. 다이내믹 프록시를 위한 트랜잭션 부가기능
public class TransactionHandler implements InvocatioinHandler {
    private Object target; // 부가기능을 제공할 타깃 오브젝트. 어떤 타입의 오브젝트에도 적용 가능하다.
    private PlatformTransactionManager transactionManager; // 트랜잭션 기능을 제공하는 데 필요한 트랜잭션 매니저
    private String pattern; // 트랜잭션에 적용할 메서드 이름 패턴

    public void setTarget(Object target) {
        this.target = target;
    }

    public void setTransactionManager(PlatformTransactionManager transactionManager) {
        ...
    }

    public void setPattern(String pattern) {
        ...
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 트랜잭션 적용 대상 메서드를 선별해서 트랜잭션 경계설정 기능을 부여해준다.
        if (method.getName().startsWith(pattern)) {
            return invokeTransation(method, args);
        } else {
            return method.invoke(target, args);
        }
    }

    private Object invokeTransation(Method method, Object[] args) throws Throwable {
        TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
        
        try {
            // 트랜잭션을 시작하고 타깃 오브젝트의 메서드를 호출한다. 예외가 발생하지 않았다면 커밋한다.
            Object ret = method.invoke(target, args);
            this.transactionManager.commit(status);
            return ret;
        } catch (InvocationTargetException e) { // 예외가 발생하면 트랜잭션을 롤백한다.
            this.transactionManager.rollback(status);
            throw e.getTargetException;
        }
    }
}
```
- 트랜잭션을 적용하면서 타깃 오브젝틩 메서드를 호출하는 것은 UserServiceTx에서와 동일하다.
- 한 가지 차이점은 롤백을 적용하기 위한 예외는 RunTimeException 대신에 InvocationTargetException을 잡도록 해야 한다는 점이다.
  - 리플렉션 메서드인 Method.invoke()를 이용해 타깃 오브젝트의 메서드를 호출할 때는 타깃 오브젝트에서 발생하는 예외가 InvocationTargetException으로 한 번 포장돼서 전달된다.
  - 따라서 일단 InvocationTargetException으로 받은 후 getTargetException() 메서드로 중첩되어 있는 예외를 가져와야 한다.

#### TransactionHandler와 다이내믹 프록시를 이용하는 테스트
```java
// 6-28. 다이내믹 프록시를 이용한 트랜잭션 테스트
@Test
public void upgradeAllOrNothing() throws Exception {
    ...
    // 트랜잭션 헨들러가 필요한 정보와 오브젝트를 DI 해준다.
    TransactionHandler txHandler = new TransactionHander();
    txHandler.setTarget(testUserService);
    txHandler.setTransactionManager(transactionManager);
    txHandler.setPattern("upgradeLevels");

    // UserService 인터페이스 타입의 다이내믹 프록시 생성
    UserService txUserService = (UserService)Proxy.newProxyInstance(getClass.getClassLoader(), new Class[] { UserService.class }, txHandler);
    ...
}
```

### 6.3.4 다이내믹 프록시를 위한 팩토리 빈
- 문제 : DI의 대상이 되는 다이내믹 프록시 오브젝트는 일반적인 스프링 빈으로는 등록할 방법이 없다.
    - 사전에 프록시 오브젝트 클래스 정보를 미리 알아내서 스프링의 빈에 정의할 방법이 없다.
- 다이내믹 프록시는 Proxy 클래스의 newProxyInstance()라는 스태틱 팩토리 메서드를 통해서만 만들 수 있다.


#### 팩토리 빈
- 스프링은 빈을 만들 수 있는 다른 여러가지 방법을 제공한다. 대표적으로 팩토리 빈을 이용한 빈 생성 방법을 들 수 있다.
- 팩토리 빈이란 스프링을 대신해서 오브젝트의 생성로직을 담당하도록 만들어진 특별한 빈을 말한다.
- 팩토리 빈을 만드는 방법에는 여러가지가 있는데, 가장 간단한 방법은 스프링의 FactoryBean이라는 인터페이스를 구현하는 것이다.

```java
// 6-29. FactoryBean 인터페이스
public interface FactoryBean<T> {
    T getObject() throws Exception; // 빈 오브젝트를 생성해서 돌려준다.
    Class<? extends T> getObjectTyoe(); // 생성되는 오브젝트 타입을 알려준다.
    boolean inSingleton(); // getObject()가 돌려주는 오브젝트가 항상 같은 싱글톤 오브젝트인지 알려준다.
}
```
- FactoryBean 인터페이스를 구현한 클래스를 스프링의 빈으로 등록하면 팩토리 빈으로 동작한다.
- 팩토리 빈의 동작 원리를 확인할 수 있도록 만들어진 학습 테스트를 살펴보자.
- 리스트 6-30의 Message 클래스는 생성자를 통해 오브젝트를 만들 수 없다. 오브젝트를 만들려면 반드시 스태틱 메서드를 사용해야 한다. 즉 다음과 같은 방식으로 사용하면 안된다.
  ```xml
  <bean id="m" class="springbook.learningtest.spring.factorybean.Message">
  ...
  ```

```java
public class Message {
    String text;

    // 생성자가 private로 선언되어 있어 외부에서 생성자를 통해 오브젝트를 만들 수 없다.
    private Message(String text) {
        this.text = text;
    }

    public String getText() {
        return text;
    }

    // 생성자 대신 사용할 수 있는 스태틱 팩토리 메서드를 제공한다.
    public static Message newMessage(String text) {
        return new Message(text);
    }
}
```

- 생성자를 private로 만들었다는 것은 스태틱 메서드를 통해 오브젝트가 만들어져야 하는 중요한 이유가 있기 때문이므로 이를 무시하고 오브젝트를 강제로 생성하면 위험하다.
- Message 클래스의 오브젝트를 생성해주는 팩토리 빈 클래스를 만들어보자.
```java
// 6-31. Message의 팩토리 빈 클래스
public class MessageFactoryBean implements FactoryBean<Message> {
    // 오브젝트를 생성할 때 필요한 정보를 팩토리 빈의 프로퍼티로 설정해서 대신 DI 받을 수 있게 한다. 주입된 정보는 오브젝트 생성 중에 사용된다.
    String text;

    public void setText(String text) {
        this.text = text;
    }

    // 실제 빈으로 사용될 오브젝트를 직접 생성한다. 코드를 이용하기 때문에 복잡한 방식의 오브젝트 생성과 초기화 작업도 가능하다.
    public Message getObject() throws Exception {
        return Message.newMessage(this.text);
    }


    public Class<? extends Message> getObjectType() {
        return Message.class;
    }

    // 이 팩토리 빈은 매번 요청할 때마다 새로운 오브젝트를 만들므로 false로 설정한다.
    // 이것은 팩토리 빈의 동작 방식에 관한 설정이고 만들어진 빈 오브젝트는 싱글톤으로 스프링이 관리해줄 수 있다.
    public boolean isSingleton() {
        return false;
    }
}
```

#### 팩토리 빈의 설정 방법

``` xml
// 6-32. 팩토리 빈 설정
// FactoryBeanText-context.xml이라는 이름으로 저장
<bean id="message" class="springbook.learningtest.spring.factorybean.MessageFactoryBean">
    <property name="text" value="Factory Bean" />
</bean>
```
- 여타 빈 설정과 다른 점은 message 빈 오브젝트의 타입이 class 애트리뷰트에 정의된 MessageFactoryBean이 아니라 Message 타입이라는 것이다.
- Message 빈의 타입은 MessageFactoryBean의 getObjectType() 메서드가 돌려주는 타입으로 결정된다.
- 또, getObject() 메서드가 생성해주는 오브젝트가 message 빈의 오브젝트가 된다.

```java
// 6-33. 팩토리 빈 테스트
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration // 설정 파일 이름을 지정하지 않으면 클래스 이름 + "-context.xml"이 디폴트로 사용된다.
public class FactoryBeanTest {
    @Autowired
    ApplicationContext context;

    @Test
    public void getMessageFromFactoryBean() {
        Object message = context.getBean("message");
        asserThat(message, is(Message.class)); // 타입 확인
        asserThat((Message)message.getText(), is("Factory Bean")); // 설정과 기능 확인
    }
    ...
}

- 드물지만 팩토리 빈이 만들어주는 빈 오브젝트가 아니라 팩토리 빈 자체를 가져오고 싶은 경우도 있다.
- 이럴 때는 &을 빈 이름 앞에 붙여주면 된다.
```java
// 6-34. 팩토리 빈을 가져오는 기능 테스트
@Test
public void getFactoryBean() throws Exception {
    Object factory = context.getBean("&message");
    asserThat(factory, is(MessageFactoryBean.class)); 
}
```

#### 다이내믹 프록시를 만들어주는 팩토리 빈
- Proxy의 newProxyInstance() 메서드를 통해서만 생성이 가능한 다이내믹 프록시 오브젝트는 일반적인 방법으로는 스프링의 빈으로 등록할 수 없다.
- 대신 팩토리 빈을 사용하면 다이내믹 프록시 오브젝트를 스프링의 빈으로 만들어줄 수 있다.
- 팩토리 빈의 getObject() 메서드에 다이내믹 프록시 오브젝트를 만들어주는 코드를 넣으면 되기 때문이다.

<img width="542" alt="스크린샷 2024-07-07 오후 7 05 52" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/6692f754-b916-46b5-845e-bc7816cc5098">

- 스프링 빈에는 팩토리 빈과 UserServiceImpl만 빈으로 등록한다.
- 팩토리 빈은 다이내믹 프록시가 위임할 타깃 오브젝트인 UserServiceImpl에 대한 레퍼런스를 프로퍼티를 통해 DI 받아둬야 한다.
- 다이내믹 프록시를 직접 만들어서 UserService에 적용해봤던 upgradeAllOrNothing() 테스트의 코드를 팩토리 빈을 만들어서 getObject() 안에 넣어주기만 하면 된다.

#### 트랜잭션 프록시 팩토리 빈
```java
// 6-35. 트랜잭션 프록시 팩토리 빈
...
public class TxProxyFactoryBean implements FactoryBean<Object> {
    Object target;
    PlatformTransactionManager transactionManager;
    String pattern;
    Class?> serviceInterface; // 다이내믹 프록시를 생성할 때 필요하다. UserService 외의 인터페이스를 가진 타깃에도 적용할 수 있다.

    public void setTarget(Object target) {
        this.target = target;
    }

    public void setTransactionManager (PlatformTransationManager transactionManaer) {
            this.transactionManager = transactionManager;
    }

    public void setPattern(String pattern) {
        ...
    }

    public void setServiceInterface(Class?> serviceInterface) {
        ...
    }

    // FactoryBean 인터페이스 구현 메서드
    public Object getObject() throws Exception { // DI 받은 정보를 이용해서 TransactionHandler를 사용하는 다이내믹 프록시를 생성한다.
        TransactionHandler txHandler = new TransactionHander();
        txHandler.setTarget(testUserService);
        txHandler.setTransactionManager(transactionManager);
        txHandler.setPattern("upgradeLevels");

        return Proxy.newProxyInstance(
            getClass().getClassLoader(), new Class[] { serviceInterface }, txHandler);
    }
    
    public Class<?> getObjectType() {
        return serviceInterface; // 팩토리 빈이 생성하는 오브젝트의 타입은 DI 받은 인터페이스 타입에 따라 달라진다. 따라서 다양한 타입의 프록시 오브젝트 생성에 재사용할 수 있다.
    }

    public booelean isSingleton() {
        return false; // 싱글톤 빈이 아니라는 뜻이 아니라 getObject()가 매번 같은 오브젝트를 리턴하지 않는다는 의미다.
    }
}
```

- 아래와 같이 UserServiceTx 빈 설정을 대신해서 userService라는 이름으로 TxProxyFactoryBean 팩토리 빈을 등록한다. UserServiceTx 클래스는 이제 더 이상 필요 없으니 제거해도 상관 없다.
```xml
// 6-36. UserService에 대한 트랜잭션 프록시 팩토리 빈
<bean id="userService" class="springbook.user.service.TxProxyBean">
    <property name="target" ref="userServiceImpl" />
    <property name="transactionManager" ref="transactionManager" />
    <property name="pattern" value="upgradeLevels" />
    <property name="serviceInterface" value="springbook.user.service.UserService" />
</bean>
```

- serviceInterface는 Class 타입이다. Class 타입은 value를 사용해 클래스 또는 인터페이스의 이름을 넣어주면 된다.

#### 트랜잭션 프록시 빈 테스트
```java
// 6-27. 트랜잭션 프록시 팩토리 빈을 적용한 테스트
public class UserServiceTest {
    ...
    @Autowired ApplicationContext context; // 팩토리 빈을 가져오려면 애플리케이션 컨텍스트가 필요하다. 
    @Test
    @DirtiesContext // 다이내믹 프록시 팩토리 빈을 직접 만들어 사용할 때는 없앴다가 다시 등장한 컨텍스트 무효화 애너테이션
    public void upgradeAllOrNothing() throws Exception {
        TestUserService testUserService = new TestUserService(users.get(3).getId());
        testUserService.setUserDao(userDao);
        testUserService.setMailSender(mailSender);

        // 팩토리 빈 자체를 가져와야 하므로 빈 이름에 & 를 반드시 넣어야 한다.
        TxProxyFactoryBean txProxyFactoryBean = 
            context.getBean("&userService", TxProxyFactoryBean.class); // 테스트용 타깃 주입
        txProxyFactoryBean.setTarget(testUserService);
        UserService txUserService = (UserService) txProxyFactoryBean.getObject(); // 변경된 타깃 설정을 이용해서 트랜잭션 다이내믹 프록시 오브젝트를 다시 생성한다.
                 
        userDao.deleteAll();			  
        for(User user : users) userDao.add(user);
        
        try {
            txUserService.upgradeLevels();   
            fail("TestUserServiceException expected"); 
        }
        catch(TestUserServiceException e) { 
        }
        
        checkLevelUpgraded(users.get(1), false);
    }
}
```

### 6.3.5 프록시 팩토리 빈 방식의 장점과 한계
- 한 번 팩토리 빈을 만들어두면 타깃의 타입과 상관없이 재사용 가능
#### 프록시 팩토리 빈의 재사용
- 타깃 오브젝트에 맞는 프로퍼티 정보를 설정해서 빈으로 등록해주기만 하면 코드 수정 없이 다양한 클래스에 팩토리 빈 적용 가능
``xml
// 6-38. 트랜잭션 서비스 빈 설정
