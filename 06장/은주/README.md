# 6장. AOP
- AOP 는 IoC/DI, 서비스 추상화와 더불어 스프링의 3대 기반 기술의 하나이다
- 스프링에 적용된 가장 인기있는 AOP 적용대상은 바로 `선언적 트랜잭션` 기능이다

## 6.1. 트랜잭션 코드의 분리
### 6.1.1. 메소드 분리
```java
// 트랜잭션 경계설정과 비즈니스 로직이 공존하는 메서드
public void upgradeLevels() throws Exception {
    TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
    
    try {
        // 비즈니스 로직
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
- 비즈니스 로직 담당 코드를 메소드로 추출하여 비즈니스 로직과 트랜잭션 경계설정을 분리해보자.
```java
public void upgradeLevels() throws Exception {
    TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
    
    try {
        upgradeLevelsInternal();
    
        this.transactionManager.commit(status);
    } catch (Exception e) {
        this.transactionManager.rollback(status);
        throw e;
    }
}

private void upgradeLevelsInternal() {
    List<User> users = userDao.getAll();
        for (User user : users) {
            if (canUpgradeLevel(user)) {
                upgradeLevel(user);
            }
        }
}  
```

### 6.1.2. DI 를 이용한 클래스의 분리
- 어차피 서로 직접적으로 정보를 주고받지 않으면, 아예 트랜잭션 코드가 존재하지 않는 것처럼 사라지게 할 수는 없을까?

#### DI 적용을 이용한 트랜잭션 분리
- DI의 기본 아이디어는 **실제 사용할 오브젝트의 클래스 정체는 감춘채 인터페이스를 통해 간접으로 접근** 하는 것
    - 보통 이 방법을 사용하는 이유는 **일반적으로 구현 클래스를 바꿔가면서 사용하기 위해서** 이다

![alt text](image.png)
- UserServiceTx 는 UserSeviceImpl 을 대신하기 위해 만든게 아니라, 단지 트랜잭션 경계설정이라는 책임을 맡고 있는 것이다
- 스스로는 비즈니스 로직을 담고 있지 않기 때문에, 또다른 비즈니스 로직을 담고 있는 **UserService의 구현 클래스에 실제적인 로직 처리 작업은 위임하는 것**이다

#### UserService 인터페이스 도입
- 클라이언트가 사용할 핵심 로직만 담은 메소드만 UserService 인터페이스로 만들고 UserServiceImpl 이 구현하도록 만든다
```java
public class UserServicelmpl implements UserService { 
    UserDao userDao;
    MailSender mailSender;
    public void upgradeLevels() {
        List<User> users = userDao.getAll(); 
        for (User user : users) {
            if (canUpgradeLevel(user)) { 
                upgradeLevel(user);
            } 
        }
    }
}
```

#### 분리된 트랜잭션 기능
- UserServiceTx 는 기본적으로 UserService 를 구현하지만, **같은 인터페이스를 구현한 다른 오브젝트에게 작업을 위임하게 만들면 된다**
  - 이를 위해 UserService 오브젝트를 DI 받을 수 있도록 만든다
```java
public class UserServiceTx implements UserService { 
    // UserService 를 구현한 다른 오브젝트를 DI 받는다.
    UserService userService;

    public void setUserService(UserService userService) { 
        this.userService = userService;
    }

    // DI 받은 UserService 오브젝트에 모든 기능을 위임한다.
    public void add(User user) { 
        this.userService.add(user);
    }

    public void upgradeLevels() {
        userService.upgradeLevels();
    }
}
```
- 이 UserServiceTx 에 트랜잭션 경계설정이라는 부가적인 작업을 부여해보자.
```java
public class UserServiceTx implements UserService { 
    UserService userService; PlatformTransactionManager transactionManager;

    public void setTransactionManager(PlatformTransactionManager transactionManager) {
        this.transactionManager = transactionManager; 
    }

    public void setUserService(UserService userService) { 
        this.userService = userService;
    }

    public void add(User user) { 
        this.userService.add(user);
    }

    public void upgradeLevels() {
        Transactionstatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());

        try {
            userService.upgradeLevels();
            this.transactionManager.commit(status); 
        } catch (RuntimeException e) {
            this.transactionManager.rollback(status);
            throw e; 
        }
    }
}
```

#### 트랜잭션 적용을 위한 DI 설정
- UserService 인터페이스를 통해 사용자 관리 로직을 이용하려 할 때, **트랜잭션을 담당하는 오브젝트가 사용돼서 트랜잭션 작업을 진행** 하고, 실제 **사용자 관리 로직을 담은 오브젝트는 이후에 호출돼서 비즈니스 로직 처리를 수행**하도록 한다.

#### 트랜잭션 경계설정 코드 분리의 장점
- 비즈니스 로직 담당하는 UserServiceImpl 코드 작성시에는 트랜잭션에 신경쓰지 않아도 된다
  - 트랜잭션은 **DI 를 이용해서 UserServiceTx 와 같은 트랜잭션 기능을 가진 오브젝트가 먼저 실행되도록** 만들기만 하면 된다
- 비즈니스 로직에 대한 테스트를 손쉽게 만들 수 있다

## 6.2. 고립된 단위 테스트
### 6.2.1. 복잡한 의존관계 속의 테스트
![](image-1.png)
- UserServiceTest 가 테스트 하고자 하는 대상인 UserService 는 사용자 정보를 관리하는 비즈니스 로직의 구현 코드이다.
  - 따라서 UserService 의 코드가 바르게 작성되어 있으면 성공, 아니면 실패이다.
  - 따라서 **테스트의 단위는 UserService 클래스여야 한다**
- UserService 는 UserDao, TransactionManager, MailSender 라는 3가지 의존관계를 가지고 있다
  - 따라서 UserService 를 테스트하는 것처럼 보이지만 사실은 그 뒤의 의존관계를 따라 등장하는 오브젝트, 서비스, 환경 등이 모두 합쳐져 테스트 대상이 되는 것이다

### 6.2.2. 테스트 대상 오브젝트 고립시키기
- 테스트 대상이 환경, 외부 서버, 다른 클래스 코드에 종속되고 영향받지 않도록 고립시킬 필요가 있다
  - 테스트를 의존 대상으로부터 분리해서 고립시키는 방법은 **테스트를 위한 대역을 사용하는 것** 이다

#### 테스트를 위한 UserServiceImpl 고립
![alt text](image-2.png)
- 고립된 테스트가 가능하려면 UserServiceImpl 가 2개의 목 오브젝트에만 의존하도록 하면 완벽하게 고립된 테스트 대상으로 만들 수 있다.
- UserDao 는 **테스트 대상의 코드가 정상 수행되도록 돕기만 하는 스텁** 이 아니라, **부가적인 검증기능까지 가진 목 오브젝트** 로 만든다

#### 고립된 단위 테스트 활용
```java
private void upgradeLevels() {
    List<User> users = userDao.getAll(); // 업그레이드 후보 사용자 목록 조회
    for (User user : users) {
        if (canUpgradeLevel(user)) {
            upgradeLevel(user);
        }
    }
}  

protected void upgradeLevel(User user) {
    user.upgradeLevel();
    userDao.update(user); // 수정된 사용자 정보를 DB에 반영한다.
    sendUpgradeEmail(user);
}
```
- userDao.getAll() 은 레벨 업그레이드 후보가 될 사용자 목록을 받아오기에, 테스트용 UserDao 에는 DB에서 읽어온 것처럼 미리 준비된 사용자 목록을 제공해야 한다
- update() 메소드는 upgradeLevels() 의 핵심 로직인 '전체 사용자 중에서 업그레이드 대상자는 레벨을 변경해준다' 에서 **'변경' 에 해당하는 부분을 검증할 수 있는 중요한 기능이기도 하다**
  - 업그레이드를 통해 레벨이 변경된 사용자는 DB에 반영되도록 userDao의 update()에 전달돼야 하기 때문
- getAll() 에 대해서는 `스텁` 으로서, update() 에 대해서는 `목 오브젝트` 로서 동작하는 UserDao 타입의 테스트 대역이 필요하다

```java
public class MockUserDao implements UserDAO {
    // 레벨 업그레이드 후보 User 오브젝트 목록
    private List<User> users;

    // 업그레이드 대상 오브젝트를 저장해둘 목록
    private List<User> updated = new ArrayList<User>();

    private MockUserDao(List<User> users){
        this.users = users;
    }

    public List<User> getUpdated() {
        return this.updated;
    }

    // 스텁 기능 제공
    public List<User> getAll() {
        return this.users;
    }

    // 목 오브젝트 기능 제공
    public void update(User user){
        updated.add(user);
    }
    ...
}
```
- updated 리스트는 update() 메소드를 실행하면서 넘겨준 업그레이드 대상 User 오브젝트를 **저장했다가 검증을 위해 돌려주기 위한 것**

### 6.2.3. 단위 테스트와 통합 테스트
- 단위 테스트의 단위는 정하기 나름이다
  - 사용자 관리 기능 전체일 수도, 하나의 클래스나 하나의 메소드를 단위로 볼 수도 있다
  - 중요한 것은 **하나의 단위에 초점을 맞춘** 테스트라는 점이다
- `단위 테스트` : 테스트 대상 클래스를 목 오브젝트 등의 **테스트 대역을 이용해 의존 오브젝트나 외부 리소스를 사용하지 않도록 고립시켜 테스트 하는 것**
- `통합 테스트` : 2개 이상의, **성격이나 계층이 다른 오브젝트가 연동하도록 만들어 테스트**하거나 또는 외부의 DB나 파일, 서비스 등의 리소스가 참여하는 테스트
- 단위 테스트와 통합테스트 관련 가이드 라인
  - 외부 리소스를 사용해야만 가능한 테스트는 통합 테스트로 만든다
  - DAO 테스트는 DB 라는 외부 리소스를 사용하기에 통합 테스트로 분류되지만 하나의 기능 단위를 테스트하는 것이기도 하다
    - DAO 를 테스트를 통해 검증해두면 DAO 를 이용하는 코드는 DAO 역할을 스텁이나 목 오브젝트로 대체해서 테스트할 수 있다
  - 단위테스트를 만들기 복잡하다고 판단되면 처음부터 통합 테스트를 고려해본다
    - 통합테스트 참여하는 코드 중 가능한 한 많은 부분을 미리 단위 테스트로 검증해두는 게 유리하다

### 6.2.4. 목 프레임워크
#### Mockito 프레임워크
- mockUserDao 의 update() 메소드가 2번 호출됐는지 확인하고 싶을 때
```java
verify(mockUserDao, times(2)).update(any(User.class));
```
-  목 오브젝트가 제공하는 검증 기능을 통해서 **어떤 메소드가 몇 번 호출 됐는지, 파라미터는 무엇인지** 확인할 수 있다.
```java
    verify(mockUserDao, times(2)).update(any(User.class));
    verify(mockUserDao).update(users.get(1));
    assertThat(users.get(1).getLevel(), is(Level.SILVER));        verify(mockUserDao).update(users.get(3));
    assertThat(users.get(3).getLevel(), is(Level.GOLD));
```

## 6.3. 다이내믹 프록시와 팩토리 빈
### 6.3.1. 프록시와 프록시 패턴, 데코레이터 패턴
- 분리된 부가기능을 담은 클래스는 부가기능 외 나머지 모든 기능은 **원래 핵심기능을 가진 클래스로 위임** 해줘야 한다.
  - 핵심기능은 부가기능을 가진 클래스 존재 자체를 모른다. 즉, 부가기능이 핵심기능을 사용하는 구조
- 부가기능은 마치 자신이 핵심기능인 것처럼 꾸며서 클라이언트가 자신을 거쳐 핵심기능을 사용하도록 해야 한다
  - 그러려면 클라이언트는 **인터페이스** 를 통해서만 핵심기능을 사용하게 하고,
  - 부가 기능도 동일한 인터페이스를 구현한뒤 그 사이에 끼어들어야 한다
    ![alt text](image-3.png)
- 부가기능 코드에서는 핵심 기능으로 요청을 위임해주는 과정에서 **자신이 가진 부가 기능을 적용**해줄 수 있다.    
  - ex) 비즈니스 로직 코드에 트랜잭션 기능을 부여해주는 것
- 자신이 클라이언트가 사용하려고 하는 실제 대상인 것처럼 위장해서 클라이언트의 요청을 받아주는 것을 대리자 같은 역할을 한다고 해서 `프록시` 라고 한다
  - 프록시를 통해 최종적으로 요청을 위임받아 처리하는 실제 오브젝트를 `타깃` 이라고 부른다
- 프록시의 특징 : **타깃과 같은 인터페이스를 구현** 했다는 것과 프록시가 타깃을 제어할 수 있는 위치에 있다는 것
- 프록시는 사용 목적에 따라 2가지로 구분가능하다.
  - 클라이언트가 타깃에 접근하는 방법을 제어하기 위해서 -> `프록시 패턴`
  - 타깃에 부가적인 기능을 부여해주기 위해서 -> `데코레이터 패턴`
  - 2가지 모두 대리 오브젝트 개념의 프록시를 두고 사용하는 것은 동일하지만, **목적에 따라 디자인 패턴에서는 다른 패턴으로 구분** 한다

#### 데코레이터 패턴
- 타깃에 부가적인 기능을 **런타임 시 다이내믹하게 부여해주기 위해** `프록시를 사용` 하는 패턴
  - 다이내믹하게 기능 부가 == 컴파일 시점, 즉 코드상에서는 어떤 방법과 순서로 프록시와 타깃이 연결되어 사용되는지 정해져 있지 않다
- 데코레이터 패턴에서는 프록시가 꼭 1개로 제한되지 않는다
  - 같은 인터페이스를 구현한 타겟과 여러 개의 프록시를 사용할 수 있다
  - 프록시로서 동작하는 각 데코레이터는 **위임하는 대상에도 인터페이스로 접근하기 때문에** 자신이 최종 타깃으로 위임하는지, 다음 단계 데코레이터 프록시로 위임하는지 알지 못한다.
    - 데코레이터 다음 위임 대상은 인터페이스로 선언하고, 생성자/수정자 메소드를 통해 **위임 대상을 외부에서 런타임으로 주입받을 수 있게** 만들어야 한다
- 데코레이터 패턴은 **타깃의 코드를 손대지 않고, 클라이언트의 호출 방법도 변경하지 않고** 새로운 기능을 추가할 때 유용한 방법이다

#### 프록시 패턴
- 일반적으로 사용하는 프록시와 디자인 패턴에서 말하는 프록시 패턴은 구분할 필요가 있다
- 전자 : 클라이언트와 사용 대상 사이에 **대리 역할**을 맡은 오브젝트 총칭
- 후자 : 프록시를 사용하는 방법 중 **타깃에 대한 접근 방법을 제어하려는 목적** 을 가진 경우
- 프록시 패턴의 프록시는 **타깃의 기능을 확장하거나 추가하지 않는다**
- 타깃 오브젝트가 생성하기 복잡하거나, **당장 필요하지 않은 경우**, 꼭 필요한 시점까지 오브젝트를 생성하지 않고 프록시를 넘겨주는 것이다.
  - 프록시의 메소드를 통해 타깃을 사용하려고 시도할 때 **프록시가 타깃 오브젝트를 생성하고 요청을 위임** 하는 식이다.
- 특별한 상황에서 타깃에 대한 `접근 권한을 제어`하기 위해 사용할 수도 있다
  - ex) 특정 레이어로 넘어가서는 읽기전용으로만 동작하게 강제하려면 오브젝트의 프록시를 만들어서 사용한다. Collections 의 unmodifiableCollection()
- 구조적으로는 프록시와 데코레이터는 유사하다.
  - 다만 프록시는 코드에서 **자신이 만들거나 접근할 타깃 클래스 정보를 알고 있는 경우가 많다**

![alt text](image-4.png)

### 6.3.2. 다이내믹 프록시 
#### 프록시의 구성과 프록시 작성의 문제점
- 프록시의 기능
  - 타깃과 같은 메소드를 구현하고 있다가, 메소드 호출 시 **타깃 오브젝트로 위임**한다
  - 지정된 요청에 대해서는 **부가 기능을 수행**한다
- 프록시 작성의 문제점
  - 부가기능이 필요없는 메소드도 구현해서 타깃으로 위임하는 코드를 일일이 만들어줘야 한다
  - 부가기능 코드가 여러 메소드에 중복될 가능성이 많다

#### 리플렉션
- 다이내믹 프록시는 리플렉션 기능을 이용해서 프록시를 만든다
- 리플렉션 : 자바 코드 자체를 추상화해서 접근하도록 만든 것
- 자바의 모든 클래스는 그 **클래스 자체의 구성정보를 담은 Class 타입의 오브젝트를 하나씩** 갖고 있다
  - 클래스 오브젝트를 이용하면, 클래스 코드에 대한 메타정보를 가져오거나 오브젝트를 조작할 수 있다
 
#### 다이내믹 프록시 적용
- 다이내믹 프록시 : 프록시 팩토리에 의해 런타임시 다이내믹하게 만들어지는 오브젝트
  - 타깃의 인터페이스와 같은 타입으로 만들어진다
  - 클라이언트는 다이내믹 프록시 오브젝트를 **타깃 인터페이스를 통해 사용할 수 있다**
  - **프록시 팩토리에게 인터페이스 정보만 제공해주면 해당 인터페이스를 구현한 클래스의 오브젝트를 자동으로 만들어준다**
![alt text](image-5.png)
- 부가 기능은 프록시 오브젝트와 독립적으로 InvocationHandler 를 구현한 오브젝트에 담는다
