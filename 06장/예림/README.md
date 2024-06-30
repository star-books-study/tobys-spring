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
- 다음 절에서 좀 더 자세히 알아보자.



## 6.2 고립된 단위 테스트
