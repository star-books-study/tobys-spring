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