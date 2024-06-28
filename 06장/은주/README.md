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

### DI 적용을 이용한 트랜잭션 분리
- DI의 기본 아이디어는 **실제 사용할 오브젝트의 클래스 정체는 감춘채 인터페이스를 통해 간접으로 접근** 하는 것
    - 보통 이 방법을 사용하는 이유는 **일반적으로 구현 클래스를 바꿔가면서 사용하기 위해서** 이다

![alt text](image.png)
- UserServiceTx 는 UserSeviceImpl 을 대신하기 위해 만든게 아니라, 단지 트랜잭션 경계설정이라는 책임을 맡고 있는 것이다
- 스스로는 비즈니스 로직을 담고 있지 않기 때문에, 또다른 비즈니스 로직을 담고 있는 **UserService의 구현 클래스에 실제적인 로직 처리 작업은 위임하는 것**이다
